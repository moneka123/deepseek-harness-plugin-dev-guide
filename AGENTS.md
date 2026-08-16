# DeepSeek Harness 插件开发指令

本目录用于指导 AI 在 `D:\tools\deepseek-harness-master\deepseek-harness-master` 基础上设计、开发、测试、打包和安装插件。开始任何插件任务前，先读完本文件；涉及扩展点选择时读取 [REFERENCE.md](REFERENCE.md)，涉及打包或安装时读取 [INSTALLATION.md](INSTALLATION.md)，涉及 `ctx.dynamicCordisRunner`、`@deepseek-ai/dsh-tool-cordis`、模型即时定义插件或带浏览器审批的动态包时读取 [DYNAMIC-PLUGINS.md](DYNAMIC-PLUGINS.md)。本指南是决策导航，不是冻结 API 合约：以当前源码、目标包 README 和实际导出为最终事实来源；本指南与源码冲突时，先记录冲突，再按目标源码基线的类型、实现和测试更新方案。

> 文中行为使用以下证据等级：
> - `[公共契约]`：有类型声明和稳定测试或集成边界支持，可直接依赖。
> - `[仓库惯例]`：项目推荐做法，偏离时必须说明原因。
> - `[当前实现]`：已在源码基线 `3bbc502cd2b122fe7d1c4a4562ae8cd51ca1c1a7` 中确认，升级版本时需要重新核对。
> - `[待验证]`：当前没有足够证据，不得写成无条件保证。


## 目标

以最小改动扩展 Harness，并在宿主支持且插件实现了完整清理路径的前提下，保持插件可卸载、可热重载、可测试、可回放或可升级；未提供这些能力的插件必须明确记录限制。优先级固定为：

1. 用户设置或现有配置字段。
2. Profile/Home/命令行 Patch。
3. 独立函数插件。
4. 独立 Bundle。
5. 新增仓库 Package。
6. 修改核心服务或 Agent Loop。

能由前一级完成时，不进入后一级。普通个性化插件不修改 `apps/`、`packages/core/`、`packages/bundle/`、`vendor/`、`native/` 或 CI 配置。

## 核心术语快速区分

| 术语 | 含义 | 不等同于 |
|---|---|---|
| Plugin | 插件逻辑或动态插件身份 | Package、Fiber |
| Package | 可安装或可运行的版本载荷 | 当前运行实例 |
| Bundle | 通过 Patch 参与 Profile 装配的发布单元 | 单个 Fiber |
| Profile | 一组配置、依赖和 Bundle 的启动组合 | Session |
| Patch | 配置层变更 | 运行时停止或资源释放 |
| Entry | Cordis 配置树中的一个装配项 | Plugin 本身 |
| Fiber | 插件或服务的运行时生命周期实例 | Package |
| Session | Agent 会话及其事件边界 | 动态插件所有权本身 |
| Run | 动态 Package 的一次运行实例 | Plugin 定义本身 |

[仓库惯例] 在普通插件/安装语境和动态插件语境中，以下术语不要交叉解释：

| 术语 | 普通插件/安装语境 | 动态插件语境 |
|---|---|---|
| Plugin | npm/TypeScript 模块或 Cordis 插件逻辑 | 运行时稳定的动态 Plugin 身份 |
| Package | 可安装、构建和发布的 npm Package | `cordis_define` 创建的内存态不可变版本 |
| Bundle | 参与 Profile 装配的发布单元 | 不适用；动态 Package 不通过 Bundle 装配 |
| Run | 没有统一的普通插件专用 Run 对象 | 某个动态 Package 的一次激活尝试 |

其他指南使用这些术语时，优先引用本表；专题文档只补充本领域的增量含义。

## 确定性规则

1. 本文件面向 AI Agent，所有操作必须能通过源码、类型、测试或可复制命令验证。
2. 未确认的运行时行为不得使用“始终”“必然”“自动”“原子”“无损”“安全”等绝对词。
3. 涉及当前实现的描述必须注明源码 revision 或对应源码文件。
4. “配置移除”“包卸载”“Fiber 停止”“资源释放”是不同动作，不得互相替代。
5. “模型不可见”“工具不可调用”“权限拒绝”“客户端不加载”是不同边界，必须分别说明。
6. 指南与源码冲突时，先记录冲突，再以目标 revision 的类型、实现和测试为准。


## 开始前的源码勘察

每次开发前完成以下检查：

1. 阅读仓库根 `AGENTS.md`、`packages/AGENTS.md` 和目标子树的 `AGENTS.md`。
2. 阅读 `docs/architecture.zh.md`，确认需求属于哪项服务、事件或 capability seam。
3. 在 `packages/bundle/base/cordis.patch.yml`、`packages/bundle/web-app/cordis.patch.yml`、`packages/bundle/headless/cordis.patch.yml` 中搜索已有装配。
4. 搜索同类生产插件及测试，不凭空发明接口。
5. 阅读被调用服务的 package README 和源码导出，不根据旧文档猜签名。
6. 确认修改属于 Host、Client 还是外部独立插件；
   [当前实现] 普通 Package 只进入一个 TypeScript aggregate。
7. 明确模型可见、用户可见、持久化、协议和安全影响，并在编码前确定对应测试层。

完成标准：能明确指出要复用的服务或事件、插件所在编译面、配置入口、生命周期所有者和验证路径。

## 选择插件形态

### 函数插件

[当前实现] 默认选择。函数插件必须导出 Loader 实际读取的顶层字段；示例使用命名导出，不提供整体 `default export`：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'example-plugin'
export const inject = ['tools']

export function apply(ctx: Context): void {
  // Register contributions through ctx.
}
```

### 导出约束

函数插件必须导出 Loader 实际读取的顶层字段：`name`、`apply`，以及需要时的 `inject`、`Config` 等。除非已在目标 Loader 版本中确认同级命名导出不会丢失，否则不要把插件定义整体包在 `default export` 中。发布前必须用实际 Loader 启动一次，不能只通过 TypeScript 编译。

Service Definition 可以使用框架要求的导出形式，但 Provider 和 Bundle 必须分别验证：Definition 能否被 Consumer 和 Provider 共同导入；Provider 能否被 Loader 发现；`inject`、`name`、`Config` 和 `apply` 是否仍位于 Loader 读取的位置。

### Service Definition 与 Provider

只有插件需要向其他插件提供具名能力时使用 `Service`。定义包负责接口和类型，不是可单独运行的 Provider：

```ts
// Definition package: imported by the Provider and Consumer.
import { Service, type Context } from '@deepseek-ai/cordis'

declare module '@deepseek-ai/cordis' {
  interface Context {
    acmeSearch: AcmeSearchService
  }
}

export interface SearchRequest {
  query: string
}

export interface SearchResult {
  title: string
  url: string
}

export abstract class AcmeSearchService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'acmeSearch')
  }

  abstract search(request: SearchRequest, signal: AbortSignal): Promise<SearchResult[]>
}

export default AcmeSearchService
```

实际装载的 Provider 必须继承 Definition 并实现方法：

```ts
import type { Context } from '@deepseek-ai/cordis'
import AcmeSearchService, { type SearchRequest, type SearchResult } from '@acme/dsh-search'

class AcmeSearchHttpService extends AcmeSearchService {
  async search(request: SearchRequest, signal: AbortSignal): Promise<SearchResult[]> {
    // Call the configured provider and normalize its result.
    return []
  }
}

export const name = 'acme-search-http'
export const inject = []

export function apply(ctx: Context): void {
  ctx.plugin(AcmeSearchHttpService)
}
```

仓库内 Service Definition 包按项目约定 default-export 服务类；函数 Provider 插件仍保持命名导出，不添加 default export。不要把抽象 Definition 直接写入 Bundle 并期待它提供能力。

缺失硬依赖会让 Fiber 处于 `PENDING`。这是 Cordis 的合法生命周期状态，但完整 DSH Profile 在启动审计结束后仍存在的 `PENDING` 通常表示装配错误，并会被启动器报告为未激活 Entry。只有明确预期稍后挂载 Provider 的动态组合窗口，才允许暂时保持该状态。服务名位于扁平命名空间，使用组织或能力前缀，避免占用 `tools`、`llm`、`agents` 等内置名称。

### Capability seam

只有 Definition、Provider、Consumer 需要独立替换或演进时才拆包：

```text
Service Definition ← Provider
Service Definition ← Consumer/Tool
```

Provider 和 Consumer 互不依赖。Definition 拥有请求、结果、错误和取消约定；Provider 拥有外部协议；Consumer 拥有模型 Schema、展示和用户体验。简单工具保持单包，不做预防性拆分。

## 依赖注入和 PENDING

- [公共契约] 对启动即必须存在的 Provider，使用硬依赖 `inject`；依赖缺失时应让启动失败或进入仓库定义的失败状态。
- [公共契约] 对可选能力，使用 `ctx.get('serviceName')` 或等价的运行时探测，并明确降级行为。
- [当前实现] `PENDING` 只表示当前 Fiber/Provider 尚未完成可用状态；它不是通用的错误判定。
- [仓库惯例] 在 Profile 中判断故障时，同时检查 Entry 是否被加载、Provider 是否注册、Fiber 状态、依赖是否满足、是否存在初始化异常。
- [仓库惯例] 插件不得把“服务暂时未出现”自动解释为“安装失败”，也不得把“长期 PENDING”自动解释为“正常等待”。

硬依赖示例：

```ts
export const inject = ['tools', 'acmeSearch']
```

可选服务示例：

```ts
const jobs = ctx.get('jobs')
if (jobs !== undefined) {
  // Optional integration.
}
```

`ctx.<name>` 只用于已声明注入的服务。配置行顺序不是通用的加载顺序保证；最终顺序取决于 Patch 层、Entry identity 和 Loader 组合。

## 配置规则

所有部署可调值使用 Schemastery `Config`，包括 URL、模型、超时、重试、路径、上限和功能开关。协议常量和安全不变量保留为代码常量。

```ts
import Schema from '@deepseek-ai/schemastery'

export interface Config {
  endpoint: string
  timeoutMs: number
}

export const Config: Schema<Config> = Schema.object({
  endpoint: Schema.string().required(),
  timeoutMs: Schema.number().default(30_000),
})

export function apply(ctx: Context, config: Config): void {
  // config has been validated by Loader.
}
```

规则：

- 配置错误在加载时或最早可判定点明确失败。
- 不为缺失必需配置提供静默降级。
- Schema 无法表示的非空、正数、跨字段关系在边界显式校验。
- 默认值放在 Schema 或明确的解析步骤中，不藏在执行逻辑的 `?? default` 中。
- API Key 不写进源码、Patch、快照或 Git 管理文件；使用 Harness Credentials、环境变量引用或部署注入。
- Patch 中仅使用 `!!js`，不是 `!js`。

## 生命周期与资源所有权
### 资源清理语义

`ctx.effect()` 将 disposer 绑定到拥有它的 Fiber；Fiber unload 时会执行这些 disposer，并等待异步清理完成。[当前实现]

注册顺序用于确定 disposer 的取出顺序，但不要依赖 disposer 之间严格串行执行。若资源 B 必须在资源 A 之前释放，请在一个 disposer 内显式编排顺序，或让资源 B 自己等待资源 A 完成。

插件必须：

1. 保存并处理 `ctx.effect()` 返回的 disposer；
2. 在 disposer 中停止监听、取消请求、终止子进程并释放外部句柄；
3. 使 disposer 可重复调用或安全处理重复清理；
4. 不把“Package 已移除”当作“当前 Fiber 已停止”。

源码依据：`vendor/cordis/src/fiber.ts:71`、`vendor/cordis/src/fiber.ts:676`、`vendor/cordis/src/fiber.ts:718-751`。


所有贡献必须随插件卸载而撤销。`ctx.on()`、`ctx.plugin()`、Service 注册以及 Harness 注册表的 `register()` 已经是 effect。定时器、连接、Watcher、AbortController、子进程和外部后台工作使用 `ctx.effect()`：

```ts
export function apply(ctx: Context): void {
  ctx.effect(() => {
    const controller = new AbortController()
    const work = runWorker(controller.signal)

    return async () => {
      controller.abort()
      await work
    }
  })
}
```

清理必须达到完全停稳：发出 abort/kill 后等待 Promise、进程或 Worker 真正结束。多个异步 disposer 会并发启动；存在严格拆卸顺序时，把步骤放进同一个 async disposer 并顺序 `await`。只有扩展点契约明确规定为观察型或错误隔离型时，才隔离订阅者异常；策略、转换、Provider、插件加载和生命周期失败应按其契约传播并明确失败，不能用宽泛 `catch` 静默吞掉。

完成标准：卸载插件后，没有注册项、监听器、计时器、进程、连接、后台 Promise 或临时资源残留。

## 事件规则

[当前实现] 事件模式决定监听器行为：

| 模式 | 是否等待 Promise | 顺序 | 结果/错误语义 | 插件适用场景 |
|---|---:|---|---|---|
| `emit` | 否 | 同步逐个调用 | 忽略返回值；异步 listener 的 Promise 不会被等待 | 只做同步广播 |
| `parallel` | 是 | 并发 | 等待所有 listener；拒绝项聚合为 `AggregateError` | 独立异步观察者 |
| `serial` | 是 | 注册顺序 | 依次等待；按事件协议处理 bail | 有顺序依赖的异步处理 |
| `bail` | 按事件返回协议 | 顺序 | 首个有效返回值停止 | 授权、选择器、拦截器 |
| `waterfall` | 按 listener 返回值 | 外层到内层 | listener 必须决定是否调用 `next()`；不调用可以阻止后续链 | 修改、包装、组装 |

`waterfall` 的 `next()` 是继续链的回调。当前实现没有为所有事件提供“只能调用一次”的通用运行时保护，因此插件不得重复调用 `next()`，但不要把该保护写成 Harness 已经保证的公共契约。

只观察或包装的 waterfall 监听器必须调用 `next()`：

```ts
ctx.on('tools/execute', async (exec, next) => {
  const startedAt = performance.now()
  try {
    return await next()
  } finally {
    recordDuration(exec.name, performance.now() - startedAt)
  }
})
```

不调用 `next()` 只用于插件有意拥有最终决定的情况。误漏 `next()` 会吞掉后续插件和默认实现。

新增类型化事件时通过 declaration merging 扩展 `Events`，使用 `namespace/action` 命名，并按仓库规则为事件 JSDoc 标注模式和参数。闭合判别联合使用 `assertNever`；允许外部扩展的联合保留有说明的 default 分支。

## 模型可见、执行权限与持久化边界

[公共契约] 普通模型可见事实必须能够从 Session 日志或项目已有的持久化注入路径重建；但可见性、执行授权和持久化不是同一层。

[当前实现] 动态 Cordis 插件的 Host/Client 源码属于进程内临时能力。当前实现不将可恢复源码写入 Session 日志，因此动态插件源码不能作为 Session 重放后的恢复来源。如果功能要求在进程重启、Session 重放或页面重载后继续存在，应改用普通 Package/Bundle、持久化配置、可迁移状态或显式恢复流程。

| 问题 | 应使用的机制 | 不能替代的机制 |
|---|---|---|
| 模型是否看到工具 | Tool catalog / Prompt tool restriction | 不能当作权限控制 |
| 工具调用是否允许执行 | `ctx.tools.guard()`、`tools/pre-execute` | 不能只靠 UI 隐藏 |
| 结果是否进入会话 | Session event / 持久化策略 | 不能只靠内存变量 |
| Client 是否加载模块 | Client-safe wire layer / bundle purity | 不能等同于安全边界 |

隐藏工具不是授权；客户端纯函数不是安全边界；Session Event 是否可回放不等于任意 JavaScript 对象都可序列化。

- 静态系统指导通过 `ctx.systemPrompt.section()` 注册。
- 动态、每轮变化且模型可见的信息必须进入持久会话事件，或通过项目已经落日志的 `agent.inject()` 路径注入。
- `session/event` 是回放、UI、遥测和持久化的事实源；`agent/*` 用于实时协调。
- 不把只存在内存中的动态数据直接塞入模型请求。
- [当前实现] Harness 只对支持的 JSON-safe 值域进行快照、规范化和校验；不要依赖原型、类实例、函数、`Symbol`、循环引用、`undefined`、`BigInt`、`Date`、`Map`、`Set`、稀疏数组或身份引用在工具结果和事件中的保留。
- `followup()` 只表示消息已持久入队；它不返回该消息对应的模型结果。
- `whenIdle()` 表示整个 Agent 空闲，不能证明某一条消息的因果完成。

新增模型可见行为前，先在 [REFERENCE.md](REFERENCE.md) 中选择正确扩展点。

### 缓存与热重载边界

[当前实现] 缓存是回放加速层，不是唯一事实源。缓存记录必须绑定适用的 session identity、revision、log tail 或等价一致性键；identity 不匹配时丢弃或重建。Storage 只保证其定义的 opaque JSON 和版本检查，不负责业务 Schema、权限策略、保留期限或来源链。

[待验证] 配置热重载失败后的旧实例保留、服务注册、请求路由和资源状态依赖具体 Provider 与当前实现。插件不得默认假设 Harness 提供事务性回滚。若要求无中断更新，必须验证旧 Fiber 是否仍 ACTIVE、旧服务是否可获取、请求是否仍路由到旧实例、资源是否重复或泄漏，以及失败后的恢复路径。


## 工具插件标准

最小生产模板：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'acme-lookup-tool'
export const inject = ['tools', 'acmeSearch']

export function apply(ctx: Context): void {
  ctx.tools.register(defineTool({
    name: 'acme_lookup',
    description: 'Search the Acme knowledge base for one query.',
    parameters: {
      query: {
        type: 'string',
        required: true,
        description: 'Non-empty search query',
      },
    },
    output: {
      schema: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            title: { type: 'string' },
            url: { type: 'string' },
          },
          required: ['title', 'url'],
          additionalProperties: false,
        },
      },
      render: (_args, value) => [{
        type: 'text',
        text: value.map(item => `${item.title}\n${item.url}`).join('\n\n'),
      }],
    },
    async execute(args, exec) {
      if (args.query.trim() === '') throw new Error('query must not be empty')
      return ctx.acmeSearch.search({ query: args.query }, exec.signal)
    },
  }))
}
```

工具规则：

- `parameters` 是模型输入协议；描述使用模型能理解的任务概念，不泄漏 UI、传输或内部实现术语。
- `execute()` 接收已通过 Schema 基础校验的只读参数，仍校验 Schema 无法表达的领域约束。
- `execute()` 返回符合 `output.schema` 的规范 JSON 值，不返回内容块或供程序解析的自然语言。
- `output.render()` 将规范值转换为模型可见内容。
- 长耗时 I/O 传递并遵守 `exec.signal`。
- 工具定义注册后视为只读；热替换通过卸载旧 effect 并重新注册，不原地改 Schema 或回调。
- 结构化 ID、状态和字段直接放进返回值；Code Mode 会从同一 Schema 生成 SDK。
- UI 展示在设计时区分调用与结果意图：`presentCall()` 支持 `generic`、`terminal`、`diff`；`presentResult()` 支持 `generic`、`terminal`、`diff`、`search`、`read`、`web`。`presentCall`、`presentationMeta`、`presentResult` 必须是纯函数，不执行 I/O、不读取时钟、随机数、Session 或文件。
- 权限、审计、超时、重试和结果变换放到相应流水线插件，不复制进每个工具。

工具执行顺序：### 工具执行流水线

[当前实现]

```text
tool/call
→ tools/pre-execute
→ 内置 guards
→ tools/execute
→ tool body
→ tools/post-execute
→ canonical result/finalization
→ JSON-safe materialization
→ tools/result
→ result log/通知
```

| 阶段 | 是否可阻止 | 是否可替换结果 | 是否应修改身份/参数 |
|---|---:|---:|---|
| `tools/pre-execute` | 是：allow/deny/ask | 不直接替换已执行结果 | 不得伪造 `callId`、Agent 或 signal |
| `tools/execute` | 可通过不调用 `next()` 包装/阻断 | 可包装 dispatch 结果 | 保持原执行身份和取消边界 |
| 工具本体 | 由工具实现 | 产生原始结果 | 遵守输入 Schema 和 `exec.signal` |
| `tools/post-execute` | 是：accept/block | 可按协议替换 `value`/`content` | 失败结果不能任意改成成功 value |
| `tools/result` | 否 | 否 | 只读观察；异常按契约隔离 |

当前实现限制 `post-execute` 不能同时替换 `value` 和 `content`，也不能替换失败结果的 value。参数 Schema、运行时校验、Code Mode SDK/catalog 和工具描述必须同步修改。



```text
tool/call 日志
→ tools/pre-execute
→ 单调 guards
→ tools/execute
→ tool execute()
→ tools/post-execute
→ finalizeContent
→ tools/result
→ tool/result 日志
```

需要详情时读取 [REFERENCE.md](REFERENCE.md#工具执行扩展点)。

## 安全规则
[当前实现] 所有取消示例针对当前仓库根 `package.json` 的 Node `^22.19.0 || >=24.0.0`；不要把 `AbortSignal.any()` 示例写成 Node 18 的通用代码。插件收到 `AbortSignal` 后必须把它传递到网络、LLM、工具和后台任务边界，并在取消后停止继续提交结果。取消信号不是 Fiber 生命周期的替代品；资源释放仍需通过 disposer 完成。


插件代码和 Bundle Patch 是宿主进程内的可信代码，不受 Agent 的工作区沙箱保护。安装脚本同样继承当前用户权限。

- 安装第三方插件前审查源码、`package.json` 生命周期脚本、`cordis.patch.yml`、`!!js`、子进程/MCP 命令和发布产物。
- Git 安装锁定 commit SHA；不要使用可移动 branch 或 tag 作为生产固定点。
- pnpm `allowBuilds` 是允许第三方代码在安装期执行，不是普通配置开关。
- 工具参数不能未经验证拼接到 Shell、SQL、URL、文件路径或模板中。
- 外部输入、模型 JSON、配置、持久文件、Worker、进程和网络是校验边界；同进程静态类型边界信任 TypeScript，避免重复防御代码。
- 不向不可信子进程透传凭据环境变量。
- 临时文件使用私有目录、随机名称和仅所有者权限。
- 文件和进程能力优先复用 `ctx.fs`、`ctx.shell`、`ctx.subprocess` 和 `ctx.sandbox`，不要绕过已有策略。
- 诊断使用 `ctx.logger('组织:能力名')`，记录可安全公开的阶段和资源标识；不记录凭据、完整环境变量、用户私有内容或大型内部对象。日志不能替代按契约传播策略、Provider、转换和生命周期错误。

## Host/Client 与客户端安全边界

[当前实现] Host/Client 拆分同时受运行环境、Node-only 依赖、协议层、aggregate 依赖、bundle purity 和数据传输边界约束。共享类型可以位于普通 workspace package，但供 Client 使用的导出必须是 Client-safe wire layer；是否拆成独立 npm 包由依赖图和发布边界决定。

Client renderer 的纯函数性质只说明其计算模型，不构成权限、URL、HTML、Schema 或 XSS 安全边界。所有外部数据仍须经过协议校验和安全输出编码。


## 外部插件目录建议

个性化插件优先放在独立仓库：

```text
my-dsh-plugin/
├── package.json
├── tsconfig.json
├── src/
│   └── index.ts
├── tests/
│   └── index.spec.ts
├── cordis.patch.yml
└── README.md
```

开发期通过绝对源码路径 Patch 加载；发布时构建为 ESM JavaScript，并通过 Bundle 安装。完整步骤见 [INSTALLATION.md](INSTALLATION.md)。

## 仓库内 Package 规则

只有功能要成为内置能力时才在 `packages/<group>/<pkg>/` 新建 Package。必须包含：

```text
package.json
README.md
tsconfig.json
src/index.ts
src/invariant.ts
tests/*.spec.ts
```

并遵守：

- Package 名为 `@deepseek-ai/dsh-<name>`，版本与根一致，`private: true`，ESM。
- Host 包扩展 `tsconfig.base.json` 并只加入 `tsconfig.host.json`；Client 包扩展 `tsconfig.base.client.json` 并只加入 `tsconfig.client.json`。
- 相对源码导入使用显式 `.ts` 后缀；跨包使用包名。
- `src/types.ts` 只放类型，不放运行时代码。
- 测试放在 Package 级 `tests/`，不放 `src/__tests__`。
- 每包拥有 `./invariant`，注册包名并断言本包拥有的运行时关系；没有合理不变量时给出包专属理由。
- README 与 JSDoc 同步记录配置、错误、事件、模型 Token/KV Cache 影响及限制。
- 新行为通常需要 Agent Note；归档 Agent Note 不修改。
- 默认启用才修改相应 Bundle；可选能力保持 opt-in。

详细清单以仓库 `docs/cookbook/adding-a-package.zh.md` 和 `packages/AGENTS.md` 为准。

## 不直接修改的内容

以下内容是来源外的产物或冻结内容：

- `node_modules/`。
- `lib/`、`dist/`、`coverage/`、`*.tsbuildinfo`。
- `pnpm-lock.yaml`：由 pnpm 更新。
- 生成的英文目录和图，例如 `docs/tool-catalog.md`、`docs/config-catalog.md`、`docs/module-graph.md`、`docs/persistence-catalog.md`、`docs/event-producer-consumer.md`、`THIRD_PARTY_NOTICES.md`：修改来源后运行生成器。
- `.agents/notes/archived/`：冻结历史。
- `vendor/`：仅按 vendoring 同步流程修改。
- `native/`：仅在明确修改原生沙箱或平台能力时进入。

普通插件也不直接修改 `packages/core/agent-loop`、`packages/core/tools`、`packages/core/session`、`apps/cli` 或内置 Bundle；先使用文档化扩展点和后置 Patch。

## 测试与完成标准
### 测试分级

- P0：类型检查、构建、最小启动、关键安全/权限路径。
- P1：插件生命周期、取消、错误处理、配置和持久化边界。
- P2：跨平台、并发压力、性能和升级兼容性。

每项检查必须注明命令是否存在、执行目录、输入、预期结果和实际结果。命令不存在时不得把它写进“已通过”清单，应改为“未提供该脚本”并给出替代验证方式。


测试必须匹配影响面：

1. **单元测试**：参数边界、错误、取消、事件顺序、并发和清理。
2. **HMR/dispose 测试**：卸载插件 Fiber，断言工具、事件、服务和资源已移除。
3. **真实组合测试**：产品可见插件通过 Loader 和测试用 `cordis.yml`/进程启动；手工 `ctx.plugin()` 不足以证明发布组合。
4. **快照**：非平凡模型可见、协议可见或用户可见变化必须更新无密钥组装快照。
5. **构建产物测试**：发布入口由普通 Node 加载 `lib`，防止 tsx 掩盖 ESM、exports 或模块解析问题。
6. **真实 API E2E**：新 LLM Provider 或依赖真实提供方行为的功能。
7. **Web 测试**：Client UI 或工具卡片改变时运行浏览器快照，并确认浏览器实际加载预期模块、目标 UI 已渲染且可交互。双端改动分别验证 Host 行为、远程协议、Client 加载和页面渲染；构建成功或 `./client` 文件存在不构成 UI 验收。

外部插件最低验证：

```text
构建成功
→ pnpm pack --dry-run 内容正确
→ 安装到干净 Profile 成功
→ 按 [INSTALLATION.md](INSTALLATION.md#命令环境与前置条件) 选择 `dsh` 或 `pnpm dsh`，并用 `--profile <name> --dump-config` 确认预期行
→ 启动并观察外部结果
→ 卸载插件并确认贡献消失
```

仓库内插件按修改面选择命令，不无差别运行全部套件：

```sh
pnpm run test -- <focused-test>
pnpm run typecheck
pnpm run lint
pnpm run build
pnpm run constraints
pnpm run hygiene
pnpm run doc-sync
```

只报告实际运行的命令和结果。测试断言外部世界或持久日志，不以 Agent 自我报告中的关键词代替真实结果。

## 最终交付检查表

在宣称插件完成前逐项确认：

- [ ] 复用了现有服务或事件，没有无必要修改 Agent Loop。
- [ ] 函数插件没有 default export；Service 导出符合包约定。
- [ ] 硬依赖使用 `inject`，可选依赖使用 `ctx.get()`。
- [ ] 所有部署可调值均有 Config Schema。
- [ ] 所有注册和资源都随插件卸载，异步工作达到停稳。
- [ ] 每个 waterfall 监听器明确选择委派或短路。
- [ ] 普通模型可见动态内容可从 Session 日志或已落日志的持久化注入路径重建；若使用动态 Cordis，已明确其源码不能从 Session 日志恢复。
- [ ] 工具返回结构化 JSON，遵守 `exec.signal`，展示函数保持纯净。
- [ ] 安全决定在真正执行操作的位置强制，不能被替代调用路径绕过。
- [ ] Bundle Patch 使用稳定且命名空间化的 `id`。
- [ ] Patch 覆盖重述完整 `config`，没有误以为会深度合并。
- [ ] Package、Patch、构建入口和 `exports` 全部进入 tarball。
- [ ] 单元、清理、真实组合和必要快照均已覆盖。
- [ ] 安装来源、生命周期脚本、权限和凭据要求已写明。
- [ ] 没有手工修改生成物、锁文件、构建产物或冻结记录。

## 文档与源码基线

[仓库惯例] 本目录四份执行指南必须针对同一个 Harness 文档验证基线、Node.js 范围和 pnpm 版本进行复核。本文当前验证基线为：

- Harness revision：`3bbc502cd2b122fe7d1c4a4562ae8cd51ca1c1a7`
- Node：`^22.19.0 || >=24.0.0`
- pnpm：`11.7.0`

这里的 revision 是本文 `[当前实现]` 段落的验证基线，不自动等同于使用者当前工作树的 HEAD。开始任务时，若目标源码工作树有 Git 元数据，先执行 `git rev-parse HEAD`；若结果不同，必须重新核对所有 `[当前实现]` 段落，不能直接视为当前版本保证。

修改 Harness、Cordis、Loader、Patch 引擎、CLI、Dynamic Runner 或工具 Schema 后，必须同步复核：`AGENTS.md`、`REFERENCE.md`、`INSTALLATION.md` 和 `DYNAMIC-PLUGINS.md`。专题文档的版本说明应与本节保持一致。

## 文档变更影响矩阵

| 修改对象 | 必须复核 |
|---|---|
| 动态 Cordis 工具、审批或 Client Runner | `AGENTS.md`、`REFERENCE.md`、`DYNAMIC-PLUGINS.md` |
| Bundle、Patch、Profile 或安装 CLI | `AGENTS.md`、`INSTALLATION.md` |
| Prompt、Tool、Session、MCP 或 LLM | `AGENTS.md`、`REFERENCE.md` |
| Node、pnpm、Loader 或源码 revision | 四份指南 |
| Fiber、资源生命周期或取消语义 | `AGENTS.md`、`REFERENCE.md`、`DYNAMIC-PLUGINS.md` |

[仓库惯例] 发布或提交文档前，至少执行轻量漂移检查：确认四份指南的验证基线一致、相对 Markdown 链接和引用的源码/README/测试路径存在、DYNAMIC 文档中的工具名存在于 `docs/tool-catalog.md`，以及 INSTALLATION 文档中的 CLI 参数仍出现在 `dsh --help` 中。该检查不能替代运行时测试。

升级 Harness、Node、pnpm、Cordis、Loader 或 Patch 引擎后，必须重新核对所有文档标记为 `[当前实现]` 的内容。

## 不要这样写

以下表述禁止在没有直接证据时使用：

- 卸载包后插件已经停止；
- 配置热重载失败一定保留旧实例；
- 所有错误都会变成正常文本；
- 所有 JSON 对象都能无损保存；
- `emit` 会等待异步监听器；
- `order` 可以解决名称冲突；
- Client 纯函数就是安全边界；
- `--patch` 是事务性的；
- Bundle 删除后不会再次自动加入；
- 所有 API 都有统一 retention/deprecation/multi-version 契约。

## AI Agent 执行模板

### 执行前
- [ ] 普通插件开发读取 `AGENTS.md`；
- [ ] 扩展点选择读取 `REFERENCE.md`；
- [ ] 打包或安装读取 `INSTALLATION.md`；
- [ ] 涉及 `ctx.dynamicCordisRunner`、`@deepseek-ai/dsh-tool-cordis`、Host/Client 动态包、模型即时定义或浏览器审批时读取 `DYNAMIC-PLUGINS.md`。
- [ ] 确认文档验证基线、目标源码 revision、Node、pnpm 和工作目录；若不一致，已重新核对 `[当前实现]` 段落。
- [ ] 标记使用的是公共契约、仓库惯例还是当前实现。

### 执行中
- [ ] 保留 `inject`、identity、signal、Schema 和资源 disposer。
- [ ] 不把包管理动作当作运行时生命周期动作。
- [ ] 不在未验证时假设回滚、原子性、排序或自动去重。
- [ ] 对 `!!js`、外部结果、Client 输出和工具参数执行安全校验。

### 执行后
- [ ] dump 配置并检查最终 Entry。
- [ ] 启动实际 Profile。
- [ ] 验证工具/Provider/LLM/Client 功能。
- [ ] 验证取消、重载、停止和资源释放。
- [ ] 记录命令、输入、输出、源码版本和未执行项目。
