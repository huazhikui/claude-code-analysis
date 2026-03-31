# 第四章补充：Tool Call 机制实现细节

[返回总目录](../README.md)

## 1. 本章导读

第三部分如果只讲 memory，还不够完整。这个项目的另一个核心能力，是把“模型发起 tool_use”变成一条工程上可控、可并发、可审计、可回流到下一轮对话的执行链。

这一章重点解释：

1. tool 在代码里是怎么定义的
2. 工具池是怎么组装出来的
3. tool_use 到底怎么执行
4. 权限、Hook、并发调度、tool_result 是怎么衔接的
5. query 主循环如何把工具结果回流到下一轮模型调用

主要依据这些实现：

- [`src/Tool.ts`](../src/Tool.ts)
- [`src/tools.ts`](../src/tools.ts)
- [`src/services/tools/toolOrchestration.ts`](../src/services/tools/toolOrchestration.ts)
- [`src/services/tools/StreamingToolExecutor.ts`](../src/services/tools/StreamingToolExecutor.ts)
- [`src/services/tools/toolExecution.ts`](../src/services/tools/toolExecution.ts)
- [`src/query.ts`](../src/query.ts)
- [`src/tools/FileEditTool/FileEditTool.ts`](../src/tools/FileEditTool/FileEditTool.ts)
- [`src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx`](../src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx)

## 2. 总链路：从 `tool_use` 到 `tool_result`

先看完整链路：

```text
模型输出 assistant message
  -> 含一个或多个 tool_use blocks
  -> query.ts 收集这些 tool_use
  -> 选择 streaming executor 或普通 runTools()
  -> toolOrchestration.ts 按并发安全性分批
  -> toolExecution.ts 对每个 tool_use 逐个执行
       1. schema 校验
       2. validateInput
       3. pre-tool hooks
       4. permission / ask / deny
       5. tool.call()
       6. 生成 tool_result / attachment / progress
  -> query.ts 把结果规范化为 user-side tool_result messages
  -> 下一轮 API 调用时把这些结果带回去
```

这里最关键的一点是：

tool call 不是“模型直接调函数”，而是被拆成了多层 runtime pipeline。

## 3. `Tool` 抽象：工具不是散落函数，而是统一协议

相关实现：

- [`src/Tool.ts`](../src/Tool.ts)

### 3.1 `Tool` 接口包含什么

在 [`src/Tool.ts`](../src/Tool.ts) 中，`Tool` 接口远不止 `call()` 一个函数。

它至少覆盖这些方面：

- 能力描述
  - `name`
  - `description()`
  - `prompt()`
  - `searchHint`
- 输入输出
  - `inputSchema`
  - `outputSchema`
  - `mapToolResultToToolResultBlockParam()`
- 安全属性
  - `isConcurrencySafe()`
  - `isReadOnly()`
  - `isDestructive()`
  - `checkPermissions()`
  - `preparePermissionMatcher()`
- 语义校验
  - `validateInput()`
- UI 表现
  - `renderToolUseMessage()`
  - `renderToolResultMessage()`
  - `renderToolUseRejectedMessage()`
  - `renderToolUseErrorMessage()`
- 运行控制
  - `interruptBehavior()`
  - `requiresUserInteraction()`
  - `backfillObservableInput()`

这说明这里的 tool 不是“一个函数名 + 参数”，而是一种完整的运行时协议对象。

### 3.2 `buildTool()` 的默认值策略

所有工具都通过 `buildTool()` 构造。

`buildTool()` 在 [`src/Tool.ts`](../src/Tool.ts) 里统一补了默认值：

- `isEnabled -> true`
- `isConcurrencySafe -> false`
- `isReadOnly -> false`
- `isDestructive -> false`
- `checkPermissions -> allow`
- `toAutoClassifierInput -> ''`
- `userFacingName -> name`

这个默认值策略很关键，因为它体现了一个保守原则：

- 并发默认不安全
- 写操作默认存在风险
- 安全相关能力必须显式声明

也就是说，作者在 tool 抽象层默认选择了 fail-closed，而不是 fail-open。

### 3.3 `ToolUseContext` 是执行时上下文总线

`ToolUseContext` 里挂了很多运行时状态：

- 当前工具池
- permission context
- app state
- MCP clients / resources
- readFileState
- abortController
- 当前消息列表
- subagent / agentType 信息
- tool decisions
- content replacement state

这意味着工具执行不是纯函数调用，而是依赖完整的会话运行时。

## 4. 工具池是怎么组装的

相关实现：

- [`src/tools.ts`](../src/tools.ts)

### 4.1 `getAllBaseTools()` 是内建工具总表

`getAllBaseTools()` 返回当前环境下理论可用的所有内建工具。

这个列表里可以看到几类来源：

- 始终存在的基础工具
  - `BashTool`
  - `FileReadTool`
  - `FileEditTool`
  - `FileWriteTool`
  - `WebFetchTool`
  - `AgentTool`
- feature flag 条件启用工具
  - `WebBrowserTool`
  - `WorkflowTool`
  - `MonitorTool`
  - `SleepTool`
- 环境变量条件启用工具
  - `LSPTool`
  - `REPLTool`
- ant / 内部用户专属工具
  - `ConfigTool`
  - `TungstenTool`

所以“有哪些工具”并不是静态常量，而是 feature flags、环境变量、用户类型共同决定的。

### 4.2 `getTools()` 还会继续筛一层

`getTools(permissionContext)` 还会再做几层过滤：

1. `CLAUDE_CODE_SIMPLE` 时只给极简工具集
2. deny rules 直接过滤整类工具
3. REPL 模式下隐藏 primitive tools
4. `tool.isEnabled()` 再跑一遍动态判断

所以模型最终看到的工具池，不等于代码里声明的全量工具池。

### 4.3 `assembleToolPool()` 把内建工具和 MCP 工具合并

最终真正运行时用的是 `assembleToolPool(permissionContext, mcpTools)`：

```text
built-in tools
  + 过滤后的 MCP tools
  -> 按名字排序
  -> 去重
  -> built-in 优先
```

这一步说明 MCP 并不是旁路能力，而是被合并进统一 tool pool。

## 5. 调度层：并发不是默认开启，而是按工具属性分批

相关实现：

- [`src/services/tools/toolOrchestration.ts`](../src/services/tools/toolOrchestration.ts)

### 5.1 `partitionToolCalls()` 如何分组

`runTools()` 不会把 assistant 返回的一串 tool_use 全部并发跑掉，而是先按 `isConcurrencySafe()` 分批。

规则是：

- 连续的 concurrency-safe 工具可以组成一个并发批次
- 非 concurrency-safe 工具单独成批，串行执行

大致像这样：

```text
tool_use 序列:
  A(read-only safe)
  B(read-only safe)
  C(edit unsafe)
  D(read-only safe)

分批结果:
  [A, B] 并发
  [C] 串行
  [D] 并发批次，但实际上只有 1 个
```

### 5.2 为什么并发后还要延迟应用 `contextModifier`

对于并发批次，工具返回的 `contextModifier` 不会立刻生效，而是先排队，等批次结束后再按顺序应用。

原因也很直接：

- 并发工具如果边跑边改共享上下文，容易产生竞态
- 先收集结果，再顺序提交修改，可以保持上下文一致性

这说明作者并没有把“可并发”理解成“上下文也能随便同时写”。

## 6. Streaming Tool Executor：边流式接收，边执行工具

相关实现：

- [`src/services/tools/StreamingToolExecutor.ts`](../src/services/tools/StreamingToolExecutor.ts)

在一些路径下，工具不是等 assistant 消息完整结束后统一执行，而是边接收流式 `tool_use`，边开始执行。

这个执行器维护了每个工具的状态：

- `queued`
- `executing`
- `completed`
- `yielded`

它解决了几个问题：

1. 并发安全工具可以尽早启动
2. 非并发工具仍然保持独占
3. 结果按原始顺序产出
4. 某个并行工具报错时，可以取消兄弟工具
5. streaming fallback 时，可以丢弃已排队结果并生成 synthetic error

这比“全部等模型说完再跑”更快，也更复杂。

## 7. `runToolUse()`：真正的执行主干

相关实现：

- [`src/services/tools/toolExecution.ts`](../src/services/tools/toolExecution.ts)

这一层是整个 tool call 机制的核心。

### 7.1 第一步：Zod schema 校验

所有 tool input 先走 `tool.inputSchema.safeParse(input)`。

如果失败，直接返回带 `tool_result` 的错误消息：

- 这是对模型生成参数的第一层硬约束
- 错误会被包装回 transcript，而不是只在本地抛异常

这意味着模型能从失败结果里继续修正下一轮调用。

### 7.2 第二步：工具自己的 `validateInput()`

schema 通过后，还要跑工具自定义语义校验：

- 文件是否存在
- old_string / new_string 是否相同
- 是否命中 deny 目录
- 是否超大文件
- HTML preview 是否有效

这一层检查的是“参数结构正确，但语义是否合理”。

### 7.3 第三步：预处理与 backfill

执行前还会做一些防御性处理，例如：

- Bash 的 `_simulatedSedEdit` 会被剥离，防止模型伪造内部字段
- `backfillObservableInput()` 会在浅拷贝上补全派生字段

后者很重要。因为 hooks 和权限系统往往需要看到规范化路径，但真正传给 `tool.call()` 的原始字段又要尽量保持稳定，以免影响 transcript 序列化和缓存命中。

### 7.4 第四步：PreToolUse hooks

`runPreToolUseHooks(...)` 可以在执行前插入多种结果：

- 额外消息
- 更新后的输入
- stop reason
- permission hook 决策
- additional context

也就是说，tool call 并不是直线执行，而是允许 hook 中途改写输入、补充上下文，甚至阻断执行。

### 7.5 第五步：权限决策

权限决策通过 `resolveHookPermissionDecision(...)` 汇总得出，来源可能包括：

- hook 给出的决策
- tool 自己的 `checkPermissions()`
- 通用权限系统
- classifier
- 用户交互

这里最后会得到一个 `PermissionDecision`，核心分三类：

- `allow`
- `deny`
- `ask`

如果不是 `allow`，系统不会直接崩，而是会生成带 `tool_result` 的拒绝或错误消息写回 transcript。

### 7.6 第六步：真正执行 `tool.call()`

只有走到这里，才会真正调用工具本体：

```text
tool.call(
  callInput,
  enrichedToolUseContext,
  canUseTool,
  assistantMessage,
  onProgress
)
```

也就是说，工具本体本身是执行链里的最后一段，不是第一段。

### 7.7 第七步：把结果转回 transcript

执行结果出来后，系统会继续做：

- telemetry
- tool output span
- structured output 附件
- 渲染结果消息
- tool_result block 序列化

最终目标不是“在本地拿到 JS 返回值”，而是“把工具结果变成下一轮模型可消费的 transcript 片段”。

## 8. Query 主循环如何接住工具结果

相关实现：

- [`src/query.ts`](../src/query.ts)

`query.ts` 里的主循环有一段非常关键的代码：

```text
assistant 输出 tool_use
  -> runTools(...) 或 streamingToolExecutor.getRemainingResults()
  -> 每个 update.message 先 yield 给 UI
  -> 再 normalizeMessagesForAPI(...)
  -> 只保留 user 类型的 tool_result 等消息
  -> 塞进 toolResults
  -> 下一轮递归 query 时带上这些结果
```

这说明工具结果回流有两个面向：

1. 面向 UI
   用户立刻看到 progress / result / reject / error
2. 面向模型
   下一轮 API 调用拿到标准化的 `tool_result`

### 8.1 为什么要先 normalizeMessagesForAPI

因为内部消息类型很多：

- attachment
- progress
- meta message
- system-local message

但 API 真正需要的是严格格式的消息序列。  
所以 `normalizeMessagesForAPI()` 在这里扮演“把内部 transcript 投影成 API transcript”的角色。

### 8.2 tool summary 也是异步生成的

工具批次结束后，系统还可能异步生成 `tool use summary`，供后续上下文使用。

这说明 tool call 结果不仅有“原始结果”，还有一层“摘要化再表达”。

## 9. 具体例子：`FileEditTool`

相关实现：

- [`src/tools/FileEditTool/FileEditTool.ts`](../src/tools/FileEditTool/FileEditTool.ts)

`FileEditTool` 是理解整个工具系统最好的例子之一，因为它把 tool 协议几乎都用上了。

它定义了：

- `inputSchema` / `outputSchema`
- `toAutoClassifierInput()`
- `getPath()`
- `backfillObservableInput()`
- `preparePermissionMatcher()`
- `checkPermissions()`
- `validateInput()`
- 各类 UI renderers
- `call()`

### 9.1 它的校验不只是 schema

`validateInput()` 里还有一长串语义约束，例如：

- team memory secret guard
- old_string 和 new_string 相同直接拒绝
- deny 规则目录预检查
- UNC path 安全保护
- 超大文件保护
- 文件不存在时给相似路径提示

这说明“编辑文件”在这个系统里不是粗糙的字符串替换，而是带安全治理、路径提示、用户体验修复的一整套执行器。

## 10. 具体例子：`AskUserQuestionTool`

相关实现：

- [`src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx`](../src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx)

这个工具很能说明：tool call 并不等于 shell 或文件操作。

它的几个关键属性是：

- `shouldDefer = true`
- `requiresUserInteraction() = true`
- `isConcurrencySafe() = true`
- `isReadOnly() = true`
- `checkPermissions()` 直接返回 `behavior: 'ask'`

也就是说，这个工具本质上是一种“模型发起 UI 交互请求”的协议。

它的执行结果不是外部命令输出，而是“用户回答了什么”。

这说明 tool call 框架足够通用，既能承载：

- 文件系统工具
- shell 工具
- MCP 工具
- UI 交互工具

## 11. 这套 Tool Call 机制的设计特点

### 11.1 不是直接执行，而是多层管线

顺序大致是：

```text
tool_use
  -> schema
  -> semantic validation
  -> hook
  -> permission
  -> call
  -> result serialization
  -> transcript feedback
```

这样做的好处是每一层职责都很清晰。

### 11.2 transcript 是一等公民

工具结果最终都要回到 transcript。  
这意味着 tool call 系统不是围绕“本地函数返回值”设计，而是围绕“下一轮模型上下文”设计。

### 11.3 并发是声明式的

不是框架猜某个工具能不能并发，而是由工具自己通过 `isConcurrencySafe()` 声明。

### 11.4 权限系统不是后补的

tool 接口天生就有 `checkPermissions()`、`preparePermissionMatcher()`、`requiresUserInteraction()` 这些能力，说明安全模型从一开始就被放进了协议本身。

## 12. 本章小结

如果只用一句话总结这个项目的 Tool Call 实现：

它不是“模型调一个函数”，而是把工具调用做成了统一协议、分层校验、可并发调度、可权限裁决、可回流 transcript 的执行框架。

把它和第四章主文连起来看，会更容易理解整个系统的核心：

```text
memory 负责长期状态
tool call 负责当前行动
query 主循环负责把两者串成连续协作过程
```
