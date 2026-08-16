# 插件扩展点参考

本文供已经阅读 [AGENTS.md](AGENTS.md) 的 AI 按需求查找扩展点。使用事件或服务前，打开当前仓库对应 package README、子系统文档和源码验证签名。

> 文中行为使用以下证据等级：
> - `[公共契约]`：有类型声明和稳定测试或集成边界支持，可直接依赖。
> - `[仓库惯例]`：项目推荐做法，偏离时必须说明原因。
> - `[当前实现]`：已在源码基线 `3bbc502cd2b122fe7d1c4a4562ae8cd51ca1c1a7` 中确认，升级版本时需要重新核对。
> - `[待验证]`：当前没有足够证据，不得写成无条件保证。


## 功能到机制的映射

| 目标 | 扩展点 | 可做什么 | 不保证什么 |
|---|---|---|---|
| 增加模型工具 | `ctx.tools.register(defineTool(...))` | 结构化参数、规范结果和取消 | 不自动提供权限隔离或 deadline |
| 修改工具调用前决策 | `tools/pre-execute` | `allow`/`deny`/`ask` | 不能伪造调用身份或把隐藏当授权 |
| 包装工具执行 | `tools/execute` | timeout/retry/metrics/dispatch wrapper | 不能绕过取消和资源清理 |
| 修改工具结果 | `tools/post-execute` | accept/block/replace/additionalContexts | 不能任意把失败改为成功 |
| 观察最终结果 | `tools/result` | 只读观察、指标、日志 | 不能修改已提交结果 |
| 限制可见工具 | 在目标 Agent 的 `agent.ctx` 上调用 `ctx.tools.restrict()` | 同步展示、查找和执行集合 | 不是权限边界 |
| 最终不可撤销拒绝 | `ctx.tools.guard()` | deny 或 abstain | 不能被 UI 隐藏替代 |
| 静态系统指导 | `ctx.systemPrompt.section()` | 唯一名称、稳定顺序 | 不负责动态事实回放 |
| 动态 Prompt 上下文 | `ctx.systemPrompt.context()` 或持久注入 | 计算模型可见上下文 | 不天然持久化或可回放 |
| 整体 Prompt 改写 | `system-prompt/assemble` | waterfall 改写 | 不得破坏工具、Code Mode、结构化输出边界 |
| 每步消息改写/拒绝 | `agent/pre-step` | 在协议允许范围内变换 Step | 不得破坏身份、工具配对和持久化语义 |
| 修改 LLM 配置 | `agent/request` | 调整兼容的模型参数 | 必须检查 Provider 能力和 Schema |
| 请求失败恢复 | `agent/request-error` | retry/压缩等明确恢复 | 不能把未知错误伪装成正常结果 |
| 轮次结束前续跑 | `agent/turn-stopping` + `agent.steer()` | serial checkpoint 后续跑 | 不等于任意消息因果保证 |
| UI/协议输出 | `session/event` | 可回放事实源 | 不替代任意私有存储 |
| 新模型厂商 | `ctx.llm.registerAdapter()` | 标准 StreamChunk 和取消协议 | 不自动适配 Provider-specific 字段 |
| 新文件/进程/搜索后端 | Service Definition + Provider | 隔离外部协议和实现 | Consumer 不应依赖具体 Provider |
| 后台任务 | `ctx.jobs.start()` | 由 Job 生命周期拥有取消 | 外层 `exec.signal` 不自动终止已发布 Job |
| Web 业务节点 | `ConversationNodeDefinition` + keyed renderer | Host/Client 编译面分离 | Client-safe 不等于安全边界 |

## System Prompt 扩展

### 静态 Section

```ts
export const inject = ['systemPrompt']

export function apply(ctx: Context): void {
  ctx.systemPrompt.section({
    name: 'acme:guidance',
    order: 150,
    text: 'Use Acme terminology in Acme tasks.',
  })
}
```

[当前实现] 同一 layer/scope 内名称必须唯一；重复注册失败。scoped Section 可以遮蔽 global 同名 Section，遮蔽只对该 scope 生效。Section 按有限数值 `order` 升序排列；`order` 只决定排序，不解决冲突。`complete: true` 代表完整系统 Prompt；多个有效 complete Section 会使 assembly 失败。

### 动态 Context

动态 Context 每次 assembly 重新计算。它不是天然持久化数据，也不是天然可回放数据。如果内容必须在历史重放中一致，必须将产生它所需的输入、版本或 Session Event 以可序列化形式保存；不要把运行时对象、凭据、文件句柄或未验证外部结果直接写入 Prompt。

### 整体 Assembly Waterfall

`system-prompt/assemble` 是 waterfall。监听器可以包装或替换 Assembly，但必须保持以下边界：

1. 不重复贡献同一 Section、Context、Variable 或工具 Schema；
2. 不丢失工具调用所需的 Schema 和工具顺序约束；
3. 不破坏 Code Mode/structured output 的协议片段；
4. 不伪造消息身份、来源或持久化事件；
5. 需要完整替换时，明确声明 `complete: true`，并确保只有一个有效 complete Section。

仅过滤工具时，在目标 Agent 的 scoped `agent.ctx` 上使用 `ctx.tools.restrict()`；Restriction 只控制该 Agent 可见和可调用的工具集合，不是安全权限边界。不可撤销的执行拒绝使用 `ctx.tools.guard()` 或真正执行路径上的权限策略。

## 工具执行扩展点

### `tools/pre-execute`

这是可重排的 allow/deny/ask 策略点。签名中的 `ToolExecution` 是本次调用的身份和取消边界。插件可以根据工具名、参数、Agent、Session 和权限上下文返回决策，但不得修改或伪造 `callId`、Agent 身份、原始 signal 或已记录的参数。需要不可被后续监听器重新允许的策略时使用 `ctx.tools.guard()`。

### `tools/execute`

这是 around-dispatch waterfall。调用 `next()` 才继续内层执行；适合超时、重试、指标和统一包装。重试必须遵守幂等性、取消信号和副作用规则。

```ts
ctx.on('tools/execute', async (exec, next) => {
  const upstream = exec.signal
  const controller = new AbortController()
  exec.signal = AbortSignal.any([upstream, controller.signal])
  try {
    return await next()
  } finally {
    exec.signal = upstream
  }
})
```

该片段只展示临时替换和恢复 signal，不是完整超时实现；必须设置和清理 timer、识别自己触发的超时并等待下游停稳。

### `tools/post-execute`

它可以接受结果、阻断结果，或按类型替换 `value`/`content`，并可附加 `additionalContexts`。不能同时替换 `value` 和 `content`；不能把失败结果的 value 任意替换为成功值。观察型插件先 `await next()` 再处理下游最终结果；有意接管决策时才短路。

### `tools/result`

这是只读观察点。通知前的结果已经完成规范化、快照和冻结；监听器不得修改它，异常按观察点契约记录并隔离，不得破坏工具执行。

源码依据：`packages/core/tools/src/index.ts:152-197`、`376-387`、`1663-1666`、`1731-1777`。

### 参数 Schema 与 Code Mode

文中的 `parameters` 示例只有在明确标记为项目内部 Schema DSL 时，才能使用项目专用字段。若示例声称是 JSON Schema，必须符合当前 Harness 支持的 JSON Schema 子集，并通过运行时 Schema 校验。

修改工具参数 Schema 时，必须同步检查工具定义和运行时 validator、Code Mode SDK/catalog、Prompt 中的工具描述、Client/Host wire 类型、相关测试和生成文件。

### 注册名称和作用域

注册名称的冲突策略按扩展点分别说明。默认规则是同一 scope/layer 内重复注册失败；scoped provider 可以按扩展点规则遮蔽 global provider；不得假设所有注册表都会自动覆盖、合并或去重。需要 per-agent 变体时，通过对应的 `agent.ctx` 注册。

## Agent 与 Session 扩展

### `agent/pre-step`

它决定本 Step 进入模型的消息。修改消息前先确认 Step 的身份、来源、工具配对和持久化字段。监听器可以委派后改写 `enter.messages`，或有意拒绝；若替换导致来源、tool-call/tool-result 配对或回放关系变化，必须记录新的派生关系。新增消息必须具有准确来源，并通过正常 Session 事件记录。

### `agent/request`

只调整提供方、模型、温度、推理强度等请求配置。修改前验证目标 Adapter/Provider 支持该参数、枚举值和组合；不要把任意 Provider-specific 字段复制到所有模型请求中。历史消息由 Session 日志派生。

### `agent/request-error`

用于明确拥有的恢复策略，例如重试或压缩。接管恢复时不调用 `next()` 并返回约定的 retry 决策；不拥有该错误时调用 `next()`。插件卸载时取消并 drain 活跃恢复工作。

### `agent/turn-stopping`

这是轮次将结束时的 serial checkpoint。插件需要继续时通过公共 Agent API steering，而不是直接调用 Loop 内部实现。

### Agent 输入 API

- `followup(message)`：持久入队并唤醒。
- `steer(message)`：向进行中的 Agent 提供控制输入。
- `inject(message)`：增加下一次模型请求可见的持久上下文，但不会单独唤醒空闲 Agent。实时输入与 Session Event 的回放行为是两个边界；运行时注入不自动等于持久化。
- `AgentHandle.dispose()`：停止并等待 Agent 完全退出。

自动化桥若需要“请求—结果”语义，应明确拥有一个执行区间，并把区间内输出称为该区间输出，不能声称某条消息与某条 assistant 输出具有核心 API 不保证的逐消息因果关系。

## LLM Adapter

参考 `packages/llm/llm-deepseek` 和 `docs/cookbook/adding-an-llm-adapter.zh.md`。基本注册：

```ts
export const name = 'llm-acme'
export const inject = ['llm']

export function apply(ctx: Context, config: Config): void {
  ctx.llm.registerAdapter(['acme'], new AcmeAdapter(config))
}
```

适配器义务：

- [公共契约] `stream()` 返回标准 `AsyncIterable<StreamChunk>`，并遵守请求中的 `options.signal`。
- `usage` 在 `finish` 前产生，`finish` 后不再产生内容；工具参数流保持原始 JSON 字符串和增量。
- 每个块的 index 按首次出现顺序分配并保持稳定。
- 能够安全表达为 Provider-neutral 事实的异常，应通过 `normalizeLlmFailure()` 转换为稳定错误字段，例如 code/status/requestId；无法安全转换的传输或协议异常，应抛出稳定 code 的 Harness 错误。
- 不要把所有异常伪装成正常文本。仅在错误协议明确要求时产生 error finish chunk；否则让调用方区分正常完成、取消、Provider 错误和协议错误。
- 不支持的请求字段明确报 `UNSUPPORTED`，不静默忽略。
- [仓库惯例] 需要原生响应 ID 或签名以续接历史时，仅保存续接历史所需的最小、可序列化 `replayState`，并在恢复前验证版本和字段完整性。
- 密钥由 Cordis 配置和 Credentials 注入，不自行读取约定外密钥文件。
- 协议类型、请求序列化、传输解析、Chunk 转换和 Adapter 类分离职责。

当前源码依据：`packages/llm/llm/src/index.ts:229`、`868-883`、`911-935`；`packages/llm/llm/src/adapter-failure.ts:16-103`。

### Stream Assembler 边界

[当前实现] Stream assembler 对重复 close 采用“首次关闭生效、后续异常片段忽略”的行为；未知 block 类型在未正确闭合时应视为 malformed stream。Adapter 不得依赖未声明的隐式 block 修复。
源码：`packages/llm/llm/src/assembler.ts:63-77`、`132-148`。

## UI 与 Host/Client

Client 插件运行在浏览器：

- 不导入 `node:fs`、`node:path`、`child_process` 或仅 Node 可用模块。
- 通过 Connection/Remote API 请求 Host 数据。
- 从 `session/event` 渲染可回放状态。
- 工具卡片投影是纯函数。
- Client Package 扩展 `tsconfig.base.client.json`，声明 `dsh.client` 和相应 client export，并使用共享 Client 构建 preset。

Host 插件运行在 Node，可提供 Remote Service。Host 和 Client 对 Cordis `Context` 的声明分属不同 TypeScript aggregate；普通 Package 只能加入其中一个。需要双端功能时拆成 Host 包、Client 包和共享纯类型/协议包。

[仓库惯例] Host/Client 扩展同时检查运行时、协议、aggregate、Node-only 依赖和 bundle purity。Client-safe 只表示可以进入 Client wire/bundle 边界，不表示自动获得权限或 HTML/URL 安全性。


## 持久化状态

`ctx.storage` 是后端与已挂载 data form 的 hub，不是假设存在 `get()`/`set()` 的通用 KV API。插件需要跨重启保存状态时，先读取目标 storage form、domain 或 Provider 的 README、类型和实现，确认数据模型、JSON 限制、并发、事务、取消和清理语义。Session Event 仅承载模型可见且可回放的事实，不替代任意插件私有存储；内存状态必须能在 HMR、dispose、依赖替换和进程重启后重建。

[当前实现] Projection cache 只能作为 replay 加速层。缓存命中必须核对 session identity、revision 和必要的 log tail；缓存不一致时回退到事实源。Storage 只保证其定义的 opaque JSON 和版本检查。


## 工作区与文件能力

文件、Shell、PTY、LSP 与 workspace 的边界由当前 Profile 装配的 `ctx.fs`、`ctx.sandbox`、`ctx.shell` 和 workspace 服务决定。使用前确认实际 Provider 的路径解析、符号链接、访问范围、取消和错误语义；不通过 `process.cwd()`、Node 原生 `fs` 或自行拼接绝对路径绕过这些能力。模型参数、外部输入、配置字段与持久化路径均为路径校验边界。

## MCP 信任边界

每个 MCP Server 使用独立插件接入。本地 stdio MCP 命令在 Agent 工作区沙箱之外以宿主用户权限启动，安装和配置前必须审查命令、参数、工作目录、环境变量与生命周期脚本。HTTP MCP Server 是外部协议端点，不因使用 MCP 而成为宿主可信代码。两种传输中的工具 Schema、结果、错误、URL、命令和凭据引用都属于边界数据，消费前按用途校验；不要把远程返回内容作为可执行配置或 Shell 参数直接使用。

## Capability Provider

[仓库惯例] 可选 Provider/Service 定义三种状态：未挂载、已挂载但未 ready、ready。Consumer 必须明确每种状态下是等待、降级还是失败；动态出现的服务不应被写成必然同步可用。


新 Provider 只实现 Definition 的公开约定：

- Provider 不把厂商字段泄漏进通用 Request/Result，除非所有当前 Consumer 都需要。
- 外部协议、认证、重试、分页和响应归一化属于 Provider。
- Consumer 只依赖 Definition，不导入 Provider。
- 默认值在 Provider 的显式 `resolve(request): Spec` 阶段确定。
- 取消、超时和资源清理属于 Provider 的运行约定。
- 如果同一执行世界中 FS、Shell、PTY、LSP 需要一致，使用共享 Provider/沙箱组合，不创建彼此矛盾的局部实现。

## 后台工作

工具支持后台执行时，先由 producer 配置决定 `run_in_background`，再使用 `ctx.jobs.start()`。发布 Job ID 后，任务由 Job 生命周期拥有：外层 `exec.signal` 只取消等待本次调用，不能终止已经发布的后台任务；终止由 Job kill、Owner dispose 或 Service teardown 管理。前台任务继续绑定 `exec.signal`。

## 生产源码参考

## 保留、弃用和多版本治理

[仓库惯例] 外部结果的保留期限、字段来源、弃用周期和多版本共存策略属于具体功能或项目治理决策，不是 Harness 统一公共契约。插件若需要这些保证，必须在自己的存储 Schema、迁移脚本和运行时检查中实现，并在文档中标注为插件契约。


开发前按需求阅读：

- Cordis 生命周期：`vendor/cordis/src/fiber.ts`
- Cordis Context：`vendor/cordis/src/context.ts`
- Cordis Service：`vendor/cordis/src/service.ts`
- Cordis 事件：`vendor/cordis/src/events.ts`
- Prompt：`packages/core/system-prompt/src/index.ts`
- Tools：`packages/core/tools/src/index.ts`
- Session：`packages/core/session/src/index.ts`
- Agent 类型：`packages/core/agent/src/runtime-types.ts`
- 时间上下文：`packages/context/time-context/src/index.ts`
- LLM 重试：`packages/llm/llm-retry/src/index.ts`
- 工具超时：`packages/guard/timeout-policy/src/index.ts`
- 工具结果 spill：`packages/spill/spill-policy/src/index.ts`
- 默认装配：`packages/bundle/base/cordis.patch.yml`

## 文档与源码基线

本文的 `[当前实现]` 段落按以下文档验证基线核对：

- Harness revision：`3bbc502cd2b122fe7d1c4a4562ae8cd51ca1c1a7`
- Node：`^22.19.0 || >=24.0.0`
- pnpm：`11.7.0`

该 revision 是验证基线，不自动等同于使用者当前工作树的 HEAD。若目标源码 revision 不同，先按 [AGENTS.md](AGENTS.md#文档与源码基线) 的规则重新核对本文所有 `[当前实现]` 段落。Prompt、Tool、Session、MCP 或 LLM 的扩展点变更还必须同步复核 [AGENTS.md](AGENTS.md)、[INSTALLATION.md](INSTALLATION.md) 和 [DYNAMIC-PLUGINS.md](DYNAMIC-PLUGINS.md) 中受影响的交叉引用。

升级后重新核对所有 `[当前实现]` 段落。

## AI Agent 执行模板

- [ ] 确认扩展点、scope、identity、signal 和 Session 回放边界。
- [ ] 先检查类型、源码、运行时 validator 和生成的 SDK/catalog。
- [ ] 不把可见性当授权，不把观察点当修改点，不把实时注入当持久化。
- [ ] 记录实际命令、输入、输出、源码版本和未执行项目。
