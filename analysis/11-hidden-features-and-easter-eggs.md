# 第十章：隐藏命令、Feature Flags 与彩蛋

[返回总目录](../README.md)

[上一章：总结结论](./09-final-summary.md)

## 1. 本章导读

前面几章更多在讲主干架构、memory、组件体系和平台能力。本章专门回答另一个问题：

这个项目里，除了已经显式露出的功能之外，还藏着哪些“未公开能力、内部版差异、实验开关、品牌彩蛋”？

我的结论是：这套代码不是简单地“写了一些以后可能会用到的 TODO”。它明确存在：

- 一层内部版与外部版的编译分流。
- 一层大量依赖 `feature(...)` 的编译期开关体系。
- 一层隐藏但已接入主程序的命令。
- 一层纯 `stub` 占位命令。
- 一层产品化得很完整的品牌人格化能力，例如 `buddy`、`Clawd`、`stickers`、`passes`。

## 2. 先看整体机制：它是如何把“隐藏能力”藏起来的

最核心的机制有三个：

1. `process.env.USER_TYPE === 'ant'` 这种内部构建身份判断。
2. `feature('...')` 这种 `bun:bundle` 编译期开关。
3. 命令对象里的 `isHidden` / `isEnabled`。

证据分别在：

- [`src/commands.ts`](../src/commands.ts)
- [`src/types/command.ts`](../src/types/command.ts)
- [`src/utils/undercover.ts`](../src/utils/undercover.ts)

可以用纯文本图概括：

```text
源码中的能力
    |
    +-- 编译期分流
    |      |
    |      +-- USER_TYPE === ant
    |      |      -> 注入 internal only commands
    |      |
    |      +-- feature('...')
    |             -> 决定是否把命令/模块编进产物
    |
    +-- 运行期分流
           |
           +-- isEnabled()
           |      -> 依赖账号类型、平台、实验 gate、环境变量
           |
           +-- isHidden
                  -> 即使存在，也不出现在 help / typeahead
```

命令系统本身就明说了 `isHidden` 的语义是“是否应从 typeahead/help 中隐藏”，见 [`src/types/command.ts`](../src/types/command.ts)。

## 3. 隐藏命令分成三层

### 3.1 内部版专属命令

`commands.ts` 里有一个非常直接的名单：`INTERNAL_ONLY_COMMANDS`。注释写得也很直白，叫“Commands that get eliminated from the external build”，见 [`src/commands.ts#L224`](../src/commands.ts#L224)。

这份名单里包括：

- `backfillSessions`
- `breakCache`
- `bughunter`
- `commit`
- `commitPushPr`
- `ctx_viz`
- `goodClaude`
- `issue`
- `initVerifiers`
- `mockLimits`
- `bridgeKick`
- `version`
- `resetLimits`
- `onboarding`
- `share`
- `summary`
- `teleport`
- `antTrace`
- `perfIssue`
- `env`
- `oauthRefresh`
- `debugToolCall`
- `agentsPlatform`
- `autofixPr`

其中又有一批依赖 `feature(...)` 或内部身份才会被装入，比如：

- `ultraplan`
- `subscribePr`
- `agentsPlatform`

对应代码见 [`src/commands.ts#L48`](../src/commands.ts#L48)、[`src/commands.ts#L62`](../src/commands.ts#L62)、[`src/commands.ts#L225`](../src/commands.ts#L225)。

这说明项目并不是“代码里碰巧残留一些内部脚本”，而是把内部命令作为一等公民维护，只是在外部构建中裁掉。

### 3.2 隐藏但仍然是真命令

还有一类不是 `stub`，而是正式命令，只是故意不在帮助里展示。

典型例子：

- [`heapdump`](../src/commands/heapdump/index.ts)：描述就是 “Dump the JS heap to ~/Desktop”，但 `isHidden: true`，见 [`src/commands/heapdump/index.ts#L3`](../src/commands/heapdump/index.ts#L3)
- [`rate-limit-options`](../src/commands/rate-limit-options/index.ts)：注释直接写了 “Hidden from help - only used internally”，见 [`src/commands/rate-limit-options/index.ts#L15`](../src/commands/rate-limit-options/index.ts#L15)
- [`output-style`](../src/commands/output-style/index.ts)：已废弃，但仍保留隐藏入口，见 [`src/commands/output-style/index.ts#L3`](../src/commands/output-style/index.ts#L3)
- [`thinkback-play`](../src/commands/thinkback-play/index.ts)：注释写明“Hidden command that just plays the animation”，由 thinkback skill 在生成结束后调用，见 [`src/commands/thinkback-play/index.ts#L4`](../src/commands/thinkback-play/index.ts#L4)

这类命令更像“内部工作流节点”或“调试/过渡兼容入口”，不是完全未实现功能。

### 3.3 纯 stub 占位命令

另一类就真的是占位。很多命令目录只有一行：

```js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

典型文件包括：

- [`src/commands/good-claude/index.js`](../src/commands/good-claude/index.js)
- [`src/commands/share/index.js`](../src/commands/share/index.js)
- [`src/commands/summary/index.js`](../src/commands/summary/index.js)
- [`src/commands/onboarding/index.js`](../src/commands/onboarding/index.js)
- [`src/commands/bughunter/index.js`](../src/commands/bughunter/index.js)
- [`src/commands/oauth-refresh/index.js`](../src/commands/oauth-refresh/index.js)
- [`src/commands/ctx_viz/index.js`](../src/commands/ctx_viz/index.js)
- [`src/commands/teleport/index.js`](../src/commands/teleport/index.js)
- [`src/commands/break-cache/index.js`](../src/commands/break-cache/index.js)
- [`src/commands/ant-trace/index.js`](../src/commands/ant-trace/index.js)
- [`src/commands/backfill-sessions/index.js`](../src/commands/backfill-sessions/index.js)
- [`src/commands/issue/index.js`](../src/commands/issue/index.js)
- [`src/commands/autofix-pr/index.js`](../src/commands/autofix-pr/index.js)
- [`src/commands/perf-issue/index.js`](../src/commands/perf-issue/index.js)

这一组的含义通常是：

- 外部源码快照里不提供真实实现。
- 但命令名和接线位保留了下来。
- 真正的实现要么在内部仓库，要么被构建流程替换，要么已经下线但接口位还在。

因此像 `good-claude` 这种名字，确实带一点彩蛋气质，但从当前快照看，它更像“内部接口残影”，而不是完整功能。

## 4. `feature(...)` 说明项目有一整套实验/裁剪矩阵

我在 `src/` 全仓扫描到了 **89 个不同的 `feature(...)` 开关**。这不是零星几处 if 判断，而是一整套编译期开关体系。

代表性开关包括：

- 交互与产品能力：`VOICE_MODE`、`BUDDY`、`BRIDGE_MODE`、`TERMINAL_PANEL`、`QUICK_SEARCH`
- agent / 协作：`FORK_SUBAGENT`、`COORDINATOR_MODE`、`TEAMMEM`、`AGENT_MEMORY_SNAPSHOT`
- memory / compact：`EXTRACT_MEMORIES`、`REACTIVE_COMPACT`、`CACHED_MICROCOMPACT`
- 平台扩展：`WORKFLOW_SCRIPTS`、`MCP_RICH_OUTPUT`、`MCP_SKILLS`、`WEB_BROWSER_TOOL`
- 内部产品线：`KAIROS`、`KAIROS_BRIEF`、`KAIROS_DREAM`、`KAIROS_GITHUB_WEBHOOKS`
- 实验与观测：`PERFETTO_TRACING`、`ENHANCED_TELEMETRY_BETA`、`SLOW_OPERATION_LOGGING`
- 更激进或更内部的能力名：`ULTRAPLAN`、`TORCH`、`LODESTONE`、`CHICAGO_MCP`

这套矩阵在命令层尤其明显。比如：

- `VOICE_MODE` 才会引入 `/voice`，见 [`src/commands.ts#L80`](../src/commands.ts#L80)
- `ULTRAPLAN` 才会引入 `ultraplan`，见 [`src/commands.ts#L104`](../src/commands.ts#L104)
- `BUDDY` 才会引入 `buddy`，见 [`src/commands.ts#L118`](../src/commands.ts#L118)
- `BRIDGE_MODE` 才会引入 bridge 能力，见 [`src/commands.ts#L73`](../src/commands.ts#L73)

这一层很适合继续做一张“开关矩阵表”，因为它能回答三个问题：

1. 哪些能力在源码里已经存在。
2. 哪些能力只是内部版或实验版开启。
3. 哪些能力和公开文档里的功能边界并不完全一致。

## 5. beta headers 也是重要线索

另一个很值钱的线索不是命令，而是 beta header。  
[`src/constants/betas.ts`](../src/constants/betas.ts) 里直接列出了一批能力名和日期：

- `context-1m-2025-08-07`
- `web-search-2025-03-05`
- `fast-mode-2026-02-01`
- `token-efficient-tools-2026-03-28`
- `advisor-tool-2026-03-01`
- `afk-mode-2026-01-31`
- `cli-internal-2026-02-09`

见 [`src/constants/betas.ts#L3`](../src/constants/betas.ts#L3)。

这些字符串至少说明两点：

- 项目和上游模型能力之间是通过显式 beta 协议头协商的。
- 一些今天在产品表面不一定高可见的能力，其实已经在代码里占了稳定接口位。

这里要做一个谨慎推断：

- “代码里有 beta header” 不等于 “你现在就能在公开版里用到这项能力”。
- 但它至少说明，这些能力不是纯概念，而是已经进入工程接线阶段。

## 6. 真正像“彩蛋”的，是 buddy / Clawd / stickers 这一层

### 6.1 buddy 不是一句玩笑，而是一套完整人格化子系统

`buddy` 相关代码并不薄。最关键的地方在 [`src/buddy/prompt.ts`](../src/buddy/prompt.ts)：

- 它明确告诉主模型：输入框旁边坐着一个小生物，会偶尔在气泡里评论。
- 当用户直接叫这个生物名字时，主模型要“退开”，只用一行以内回复，别替它说话。

见 [`src/buddy/prompt.ts#L7`](../src/buddy/prompt.ts#L7)。

这说明它不是普通欢迎文案，而是已经进入对话协议层的一种“第二角色”设计。

### 6.2 companion 还是按用户身份确定性孵化的

[`src/buddy/companion.ts`](../src/buddy/companion.ts) 更有意思：

- 用用户 ID 加盐 `friend-2026-401` 做哈希，见 [`src/buddy/companion.ts#L84`](../src/buddy/companion.ts#L84)
- 决定 rarity、species、eye、hat、shiny、stats，见 [`src/buddy/companion.ts#L91`](../src/buddy/companion.ts#L91)
- species、稀有度、帽子、属性都不是随手写着玩的装饰

对应类型定义在 [`src/buddy/types.ts`](../src/buddy/types.ts)：

- 稀有度：`common` 到 `legendary`
- 物种：`duck`、`goose`、`blob`、`cat`、`dragon`、`octopus`、`penguin`、`capybara` 等
- 属性：`DEBUGGING`、`PATIENCE`、`CHAOS`、`WISDOM`、`SNARK`

见 [`src/buddy/types.ts#L1`](../src/buddy/types.ts#L1)。

其中还有一个很有意思的注释：物种名里有一个会撞上“模型代号 canary”，所以代码故意用 `String.fromCharCode` 动态构造，避免字面量进入 bundle，见 [`src/buddy/types.ts#L10`](../src/buddy/types.ts#L10)。这已经不是普通 UI 彩蛋，而是“彩蛋与内部安全约束共存”。

### 6.3 Clawd 是可以点击互动的

`LogoV2` 里的 `Clawd` 不是静态 logo。  
[`src/components/LogoV2/AnimatedClawd.tsx`](../src/components/LogoV2/AnimatedClawd.tsx) 里明确做了：

- 点击触发动画
- 动作包括 crouch-jump wave
- 或者左右张望 look-around

见 [`src/components/LogoV2/AnimatedClawd.tsx#L27`](../src/components/LogoV2/AnimatedClawd.tsx#L27) 和 [`src/components/LogoV2/AnimatedClawd.tsx#L49`](../src/components/LogoV2/AnimatedClawd.tsx#L49)。

这类细节对主能力没有任何必要，但对产品气质很重要，所以它很典型地属于“工程化彩蛋”。

### 6.4 `/stickers` 是直接通向周边页面的

`/stickers` 也不是玩笑命令。  
它会直接打开 `https://www.stickermule.com/claudecode`，见 [`src/commands/stickers/stickers.ts#L4`](../src/commands/stickers/stickers.ts#L4)。

这说明团队把实体周边都当成产品体系的一部分接入了命令界面。

### 6.5 `passes` / guest passes / referral upsell 也带明显“产品彩蛋感”

[`src/components/LogoV2/GuestPassesUpsell.tsx`](../src/components/LogoV2/GuestPassesUpsell.tsx) 里会：

- 检查 guest passes 资格缓存
- 控制 upsell 展示次数
- 展示 “3 guest passes at /passes”
- 或者展示 “Share Claude Code and earn ... extra usage”

见 [`src/components/LogoV2/GuestPassesUpsell.tsx#L22`](../src/components/LogoV2/GuestPassesUpsell.tsx#L22) 和 [`src/components/LogoV2/GuestPassesUpsell.tsx#L57`](../src/components/LogoV2/GuestPassesUpsell.tsx#L57)。

这部分不算“隐藏命令”，但非常像产品层面的半隐藏运营入口。

## 7. 还有一条很值得单独写：undercover 模式

如果说前面几节更像彩蛋或实验功能，那么 `undercover` 则体现了很强的内部产品文化。

[`src/utils/undercover.ts`](../src/utils/undercover.ts) 的逻辑是：

- 在公开 / 开源仓库里，内部版会默认进入 undercover 模式
- 自动加安全说明，避免 commit message 或 PR 泄露内部代号、版本号、项目名、Slack 频道、短链
- 甚至明确禁止写 “Claude Code” 或暴露自己是 AI

见 [`src/utils/undercover.ts#L1`](../src/utils/undercover.ts#L1)。

这说明项目在“公开协作场景下如何伪装内部 agent 身份”这件事上，是有明确设计的。  
它不是彩蛋，但它比彩蛋更能揭示这个项目背后的组织形态和真实使用场景。

## 8. 本章结论

关于“还有什么能分析”的结论可以收敛成四点：

1. 最值得继续深入的不是零散彩蛋，而是“隐藏命令 + feature flags + beta headers”组成的隐藏能力矩阵。
2. 真正意义上的彩蛋，主要集中在 `buddy`、`Clawd`、`stickers`、`passes` 这一层。
3. `undercover`、`INTERNAL_ONLY_COMMANDS`、大量 `stub` 命令说明这个项目公开版与内部版之间存在明确而系统的分层。
4. 这套代码库最有研究价值的“隐藏信息”，不是单个玩笑命令，而是它如何工程化地同时维护：
   - 对外可发布版本
   - 对内实验版本
   - 品牌人格化体验
   - 安全与组织边界

## 9. 如果继续往下挖，最适合的三个后续专题

1. `feature(...)` 全量开关矩阵：把 89 个开关按子系统整理成表。
2. 隐藏命令与内部命令索引：逐个判断是真实现、半实现还是纯 `stub`。
3. buddy / Clawd / passes 专题：专门写“人格化设计与产品彩蛋系统”。
