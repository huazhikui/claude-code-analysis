# MCP 技术实现细节与运行机制

[返回总目录](../README.md)

MCP (Model Context Protocol) 是一种开放标准，目的在于连接 AI 模型与各种外部的数据源代码和工具箱。在 Claude Code 中，由于本地系统权限局限或特定场景需要接入外部能力，系统高度集成了 MCP 协议层的对接实现。

根据静态代码分析，主要负责对接网络与通讯的核心实现在 `src/services/mcp/client.ts` 之中，它是承载这套协议并转化为大模型能力的核心层。

## 1. 核心架构与底层 Client 构建

Claude Code 的底层完全利用了官方开源的 `@modelcontextprotocol/sdk`。但它并不是简单封装调用，而是根据不同的宿主连接层级与传输网络做了大量的二次包装和安全处理：

在 `connectToServer` 的设计中，系统支持多种维度的 MCP 连接类型（`serverRef.type`）：
- **Stdio (`stdio`)：** 默认与最为常见的标准输入输出挂载调用方式。适合调用本地进程作为插件提供端。
- **SSE (`sse` / `sse-ide`)：** 以 Server-Sent Events 为载体，借助长链接事件流来与远程服务器做桥接。
- **WebSocket (`ws` / `ws-ide`)：** 原生的流式跨端连接，专门为长久运行的通道（如通过 IDE 提供上下文）所设计。
- **HTTP / Streamable Proxy：** 这是其中较为复杂的扩展。除了基本的 HTTP/Proxy 请求支持外，系统内部特殊设计了 `createClaudeAiProxyFetch` 以防在访问 `claude.ai` 等远程私有节点时遭遇 OAuth 过期等拦截（它会在底层主动触发 Token Refresh 和轮转）。

## 2. API 的转化与映射

当 MCP Server 被成功连接并返回功能列表后：
1. **Tool 列表转换：** 系统将 MCP 返回的方法转化为内部通用的 `MCPTool` 实例类或 `Command`。这是极其重要的隔离抽象层。这意味着无论是本地 `BashTool` 还是外部一个未知的 `FetchTool`，在大模型的眼中，都只是普通的统一 Schema 的描述能力。
2. **描述长度防膨胀拦截：** OpenAPI 衍生出的某些 MCP Tool 会包含成千上万字的冗长字段描述。为了避免大模型的上下文（Context Window）被毫无意义地污染甚至超限（OOM），定义中有一个强硬拦截常量 `MAX_MCP_DESCRIPTION_LENGTH = 2048`，强制修剪超过 2K 的 Payload 返回体指令。

## 3. 请求控制面与异常处理

1. **资源并发控制（Batch 控制）：**
   默认系统对常规 MCP Server 连接数量定义了分层策略。普通服务控制一定并发加载率，而针对外部远端网络资源的批处理上限（`MCP_REMOTE_SERVER_CONNECTION_BATCH_SIZE`）更高，以此避免在初始化应用时因太多 MCP 连接而导致启动卡死。

2. **Session 过期与 Auth 高速缓存控制 (Auth Cache Control)：**
   连接远程 MCP 最危险的地方在于“连锁认证风暴”。
   假如一个 Token 无效，上百个工具子调用并发会造成极其可怕的 4xx 无效雪崩重试。在 `client.ts` 中有着严密的闭环：使用 `McpAuthCacheData` （写入到本地如 `mcp-needs-auth-cache.json`）并配合着约 15 分钟的 `MCP_AUTH_CACHE_TTL_MS`，保证一个 ServerId 倘若认证失败，后续所有的对应 MCP 会立马阻断而直接报告 `needs-auth`，绝不滥发导致浪费 Token 并拖垮响应进度。

3. **HTTP 代理打通 (Proxy Mount)：**
   部分组织网络具备防火墙限制，系统在针对 `fetch` 和甚至 WebSocket（利用原生 `agent` 和底层打通）级别全面支持读取进程周边的 HTTP Proxy 注入环境变量以便于内部网关下的 MCP 内省和使用。

## 4. MCP 到应用的“双向”绑定

不止是 Claude 去调用 MCP，在此项目中，IDE 等设施也可以借由 `ws-ide` 及对应 Token 创建起向 Claude 发送或展示消息/控制事件的双向隧道。通过特权认证口，双方可以共享诊断信息（如 `mcp__ide__getDiagnostics`）甚至是让其具备底层代执行的能力。

## 总结

MCP 在此套大代码库里的价值不仅仅是为了多支持一种格式，更是为了实现业务解耦下的“平台化”（PaaS）。
借助于底层的安全验证隔离、状态机维护、工具链截断及各种网络协议封装，大模型被赋能上了可以安全且跨地域调取远程能力并以极低开发门槛向当前代码上下文中注入私有知识（如企业自身内部 Wiki 等）的前期基础建设。
