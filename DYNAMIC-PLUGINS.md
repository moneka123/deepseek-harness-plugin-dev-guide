# 动态插件参考

仅当任务涉及 `ctx.dynamicCordisRunner`、`@deepseek-ai/dsh-tool-cordis`、`@deepseek-ai/dsh-cordis-host-runner`、`@deepseek-ai/dsh-cordis-client-runner`，或模型即时定义的 Host/Client 动态包时读取本文。普通 npm Package、Bundle、Profile Patch 的开发与安装不使用此机制，继续遵守 [AGENTS.md](AGENTS.md)、[REFERENCE.md](REFERENCE.md) 和 [INSTALLATION.md](INSTALLATION.md)。

[当前实现] `@deepseek-ai/dsh-tool-cordis` 是显式 opt-in 能力；只有当前 Profile 装配了该工具并提供 `ctx.dynamicCordisRunner` 时，动态工具才会激活。调用前先确认 Profile 的 Bundle/Patch 已装配所需依赖；不要把动态工具当作所有 Profile 的默认能力。

> 本文是当前目标源码版本的动态插件操作参考，不是脱离源码版本的永久 API 承诺。若本文与目标版本的类型、实现或测试冲突，先记录冲突，再以目标版本源码为准并更新本文。
>
> 文中行为使用以下证据等级：
>
> - `[公共契约]`：有类型声明以及稳定测试或集成边界支持，可直接依赖。
> - `[仓库惯例]`：项目推荐做法，偏离时必须说明原因。
> - `[当前实现]`：已在当前源码版本中确认，可能随版本变化。
> - `[待验证]`：当前证据不足，使用前必须核对目标版本源码和测试。

## 适用边界

[公共契约] 动态插件是在运行时定义的 Plugin/Package，不是可发布的外部 npm 包。它适合当前进程中的临时能力、实验、检查、运行和停止。需要长期安装、跨进程恢复、版本发布、团队复用或 Profile 默认装配时，应开发普通 Package 和 Bundle，并遵守 [INSTALLATION.md](INSTALLATION.md)。

[当前实现] 动态插件不会创建 Plugin 文件，不会安装 npm Package，不会修改 `cordis.yml`、项目配置或个人配置，也不会自动晋升为普通 Package。需要保留实验时，应让 Agent 通过普通插件开发流程实现并测试可发布版本。

## 术语与对象层级

[公共契约] 动态插件相关对象按以下层级理解：

- **Plugin**：稳定的动态插件身份；一个 Plugin 可以拥有多个不可变 Package。
- **Package**：一次 `define` 创建的不可变版本，包含名称、用途、Host half 和可选的 Client half。
- **Run**：某个 Package 的一次激活尝试，具有唯一的 `pluginRunId`。
- **Host half**：在 Harness 宿主进程中执行的动态代码。
- **Client half**：在浏览器页面中加载和执行的动态代码。
- **Fiber**：Host half 返回的 Cordis Plugin 被挂载后的生命周期对象。
- **Session**：拥有 Plugin 定义及其生命周期操作权限的会话。

不要把 Plugin、Package 和 Run 当作同一对象：删除 Package 版本、停止当前 Run、删除 Plugin 身份是三个不同操作。

## 工具入口与本文语义的对应关系

[当前实现] 本文解释以下动态 Cordis 操作的语义。实际参数字段和返回结构必须以目标版本的工具 Schema 为准：

| 操作语义 | 工具入口 | 本文对应内容 |
|---|---|---|
| 列出当前 Host 已知的 Inspect Provider | `cordis_inspect_list` | Inspect 与能力发现 |
| 查询 Provider 暴露的 Service、Event、Builtin、Tool Schema、Slot 或 Theme 信息 | `cordis_inspect_query` | Inspect 与能力发现 |
| 查询当前 Session 拥有的动态 Plugin、Package、版本指针、源码和诊断 | `cordis_inspect_self` | 会话、进程与持久化 |
| 预检并登记 Package | `cordis_define` | `define` |
| 启动或更新一次 Run | `cordis_run` | `run` 与审批 |
| 停止 Run 或撤回请求 | `cordis_stop` | `stop` |
| 删除 Plugin 及其 Package 集合 | `cordis_undefine` | `undefine` |

工具实际字段和返回结构按以下顺序核对：`docs/tool-catalog.md`、目标版本生成的工具 Schema、`packages/extensions/tool-cordis/`，以及 Host Runner/Client Runner 的类型和测试。本文不复制完整工具 Schema，以避免 Schema 更新后产生第二份不一致定义。

[当前实现] `packages/extensions/tool-cordis/README.md` 中的 `cordis_inspect` 是对 Inspect 能力的高层聚合描述；当前实际注册并供模型调用的 Inspect 工具以 `docs/tool-catalog.md` 和 `packages/extensions/tool-cordis/src/index.ts` 为准，分别是 `cordis_inspect_list`、`cordis_inspect_query` 和 `cordis_inspect_self`。当 README、工具目录和源码的命名层级不一致时，不要把聚合名称当作可调用工具名。

## 定义、运行与撤销

### `define`：只定义，不激活

[当前实现] `define` 会：

1. 清理并校验 `name` 和 `purpose`；
2. 要求至少提供 `code.host` 或 `code.client`；
3. 对 Host/Client 代码执行语法预检；
4. 为新的 Plugin 生成稳定的 Plugin ID；
5. 为每次定义生成不可变的 Package ID；
6. 将 Package 记录到创建它的 Session 所拥有的 Plugin 下。

`define` 的语法预检不会执行 Host 或 Client 代码，也不会创建 Host Fiber、注册 handler 或加载浏览器 UI。语法预检失败时，不得把该 Package 当作可运行版本。

### `run`：创建激活尝试

[当前实现] `run` 的参数包括 Plugin、Package 和 `run`/`update` 模式。它创建一次具有唯一 `pluginRunId` 的激活尝试，并根据 Package 是否包含 Client half 进入不同路径：

- Host-only Package：在宿主进程中评估 Host half，并在 `cordis-dynamic` 组 Fiber 下挂载返回的 Cordis Plugin。
- 包含 Client half 的 Package：创建或继续一个 Host/Client 激活流程，由 Client Runner 和浏览器页面完成 Client 侧加载及结果回传。

[当前实现] 对包含 Client half 的 Package，Host Runner 的 `run` 先登记运行请求并返回 `awaiting-approval`、`starting` 等中间状态；不要把 Host Runner 的 `run()` 简化为必然同步阻塞到页面完成。模型可见的最终工具结果还取决于 `@deepseek-ai/dsh-tool-cordis` 和 Client Runner 的编排路径。

[公共契约] 带 Client half 的运行需要可用的浏览器页面参与 Client 激活。是否需要显式用户批准由运行请求的 `requiresApproval` 布尔值决定：

- `requiresApproval === true`：必须等待用户批准或拒绝；
- `requiresApproval === false`：本次运行不得额外要求用户批准；
- 用户批准只代表允许进入 Client 激活流程，不代表 Host half 已成功、Client 模块已成功加载、Client Plugin 已成功激活或 UI 已成功渲染。

### Host/Client 激活顺序

[公共契约] Client Runner 的正常顺序是：

1. 调用 Host Runner 的 `runHostHalf`；
2. Host half 成功后调用 `getClientCode` 获取精确的 Package/Run 代码；
3. 在浏览器页面中求值、导入并激活 Client half；
4. 将成功或失败结果通过 `settleUserRun` 或 `resolveRequestRun` 回传 Host Runner。

Host half 失败时，Client half 不应继续加载。已有 Host Run 的页面附着如果只在本页加载 Client half 失败，不得错误停止其他页面正在使用的 Host Run；应依据 `startedHere` 判断当前页面是否创建了该 Host Run。

[当前实现] `runHostHalf` 对已经运行的同一 Plugin/Package 具有附着语义，而不是无条件重新评估代码；并发启动由 Host Runner 协调。Client 代码不会通过运行广播事件传输，页面必须通过 `getClientCode` 获取与 `pluginRunId` 对应的精确版本。

### `stop`：停止 Run，不删除 Package

[公共契约] `stop` 停止当前激活，不删除 Plugin 或其 Package。停止流程包括：

- 撤销待审批的运行请求（如果存在）；
- 删除当前 Host Run 的 handler；
- 等待 Host half Fiber 完成 disposal 并达到停稳状态；
- 广播撤回当前动态 Package 的运行状态；
- 保留定义和 Package，以便之后显式再次运行。

[当前实现] Client 页面收到撤回通知后还需要完成自己的页面侧清理。Host Runner 的 `stop` 等待 Host Fiber 的清理，但不要把它写成“所有浏览器页面已经完成 UI 清理”的统一同步确认，除非目标版本提供了相应确认契约。

### `undefine`：删除 Plugin 及全部 Package

[公共契约] `undefine` 先取消待处理请求、停止活动 Run，再删除 Plugin 记录及其所有 Package。删除后，原 Plugin ID 和 Package ID 不应继续用于运行、停止、检查或 Client 回传。

不要把“删除 Package/Plugin”与“停止运行时”混为一谈：`stop` 保留定义；`undefine` 删除定义及其版本记录。

## 版本更新与失败处理

[公共契约] 一个 Plugin 可以拥有多个不可变 Package。新增版本时创建新的 Package，并通过 `run(..., mode: 'update')` 请求切换当前运行版本；不要原地修改已经创建的 Package 或正在运行的代码。

[当前实现] `currentPackageId`、`nextPackageId`、`activeRun` 和 `latestRun` 是不同的生命周期信息：

- `currentPackageId`：最近一次成功完成激活的 Package；
- `nextPackageId`：失败或进行中的版本转换目标；
- `activeRun`：当前激活中的 Package/Run；停止后该字段可能不存在；
- `latestRun`：最近一次激活尝试，包括待审批状态和诊断。

旧 Package 记录保留，不等于旧 Host Fiber、旧 handler 或旧 Run 自动保留。更新失败后，必须重新检查上述指针和状态；如果需要恢复旧版本，应显式运行已经验证过的旧 Package。不要把动态版本更新写成事务性回滚、无中断升级或失败后必然保留旧实例。

## 运行状态与失败阶段

[公共契约] 当前类型定义了以下运行状态：

| 状态 | 含义 |
|---|---|
| `awaiting-approval` | 等待用户决定是否允许 Client 激活 |
| `starting-host` | 正在启动 Host half |
| `client-pending` | Host 已启动，等待 Client half 完成 |
| `running` | 当前激活已经建立 |
| `waiting` | Fiber 已创建，但仍等待声明的服务或其他就绪条件 |
| `rejected` | 用户拒绝了本次 Client 激活 |
| `failed` | 当前激活尝试在某个阶段失败 |
| `cancelled` | 待处理请求在完成前被取消 |
| `stopped` | Run 已停止，但 Plugin/Package 定义仍保留 |

`failed`、`cancelled` 和 `stopped` 不等价：前者表示错误，第二个表示待处理流程被撤销，第三个表示显式停止后的状态。

[仓库惯例] 按以下流程处理 `cordis_run` 的结果，不要只依据返回消息宣称成功：

1. 返回 `awaiting-approval`：等待用户决定；不要重试或声称已经运行。
2. 返回 `starting`，或状态进入 `starting-host` / `client-pending`：流程仍在异步激活；不要宣称最终成功。
3. 使用 `cordis_inspect_self` 检查 `latestRun`、`activeRun`、`currentPackageId` 和 `nextPackageId`；需要读取源码或诊断时同时提供 `pluginId` 与 `packageId`。
4. 状态为 `running`：才可说明激活已建立；仍应分别验证 Host、Client 和 UI 功能。
5. 状态为 `waiting`：按 `waitingFor` 或对应诊断确认缺失服务；不要把等待自动改写成失败。
6. 状态为 `failed`：读取诊断，修复同一 Plugin 下的新 Package 后再重试；不要默默创建替代 Plugin。
7. 状态为 `rejected`：记录用户拒绝，不再次请求同一审批。
8. 状态为 `cancelled`：记录取消；只有在需求仍然存在时才显式重新发起运行。
9. 状态为 `stopped`：确认 Plugin 定义仍存在，再显式调用 `cordis_run` 重新激活。

[公共契约] 失败诊断至少应按以下阶段区分：

- `approval`：审批或请求阶段失败；
- `host-load`：Host 代码加载或求值失败；
- `host-apply`：Host Plugin 创建或激活失败；
- `client-load`：Client 代码求值或模块导入失败；
- `client-apply`：Client Plugin 激活或 Guard 检查失败；
- `client-render`：Client 已加载并激活，但 UI 渲染阶段失败。

[当前实现] `client-render` 是后置诊断：它发生在 Client 代码已经加载并完成激活之后，不应被描述为 `resolveRequestRun` 的同步返回错误。Host Runner 会记录该 Run 的渲染诊断，并可通过独立路径将信息反馈给模型或检查工具；它不等价于初始 Client 加载失败。

## 会话、进程与持久化

[公共契约] 定义、追加 Package、检查详情、运行、停止和删除等拥有权操作必须由拥有该 Plugin 的 Session 执行。对当前 Session 的列表和详情查询只返回该 Session 拥有的 Plugin；其他 Session 对需要拥有权的对象表现为不存在，而不是收到可用于越权的详细错误。

[公共契约] 进程级 `inventory` 是例外：它可以列出进程内所有动态 Plugin 的无源码摘要，并标出各 Plugin 的拥有 Session。`inventory` 是列举，不等于授权；运行、停止、删除和读取源码等操作仍需通过拥有权检查。运行广播、页面状态和部分运行事件也可能是进程级或页面级事实，不应把 Session 操作隔离误写成所有运行事实都完全不可见。

[当前实现] Registry 是进程内存中的事实源。动态定义不会写入磁盘，Session 日志只保存定义调用的元数据，不保存可用于恢复执行的 Host/Client 源码。进程重启后没有动态定义可自动恢复；浏览器页面重载后也必须重新运行以重新绑定 Host Run 并重新获取 Client half。

不要假定动态插件可以跨进程恢复、从 Session 日志重建源码，或因为页面仍然打开就自动恢复 Client UI。

## 等待、取消与超时

[当前实现] `AbortSignal` 目前只能可靠表示“运行请求创建前已经取消”。目标版本的 Host Runner `run` 会检查信号是否已经 aborted，但没有把该信号自动注册为请求创建后的通用撤销机制。

因此，请求登记后不得假定 `AbortSignal` 会自动：

- 撤销待审批请求；
- 停止 Host half；
- 撤回 Client half；
- 清理 handler、Fiber 或页面资源。

请求创建后的撤销应使用明确的生命周期操作，例如 `stop` 或 `undefine`，然后重新检查 `latestRun.status`、`activeRun`、待处理请求和资源状态。只有目标版本明确提供持续监听取消信号的实现时，才可以扩展上述语义。

[当前实现] Host Runner 的 `vmTimeoutMs` 默认值为 `5000` 毫秒。它限制 Host half 在 `node:vm` 中执行的同步评估部分，不等价于动态插件整个生命周期、网络请求、Fiber 等待、浏览器往返或后台异步任务的总 deadline。

[当前实现] Client-bearing `run` 是否最终进入 `rejected`、`cancelled` 或 `failed`，取决于审批结果、页面是否响应、调用方是否撤销以及 Client 加载/激活结果。没有页面响应时，不要假定会立即失败；应读取当前运行状态和请求结果。

## 安全边界

[公共契约] `node:vm` 只隔离部分全局对象，不是安全边界。动态 Host half 通过获注入的 Cordis 服务仍可能触达真实文件、网络、Shell、子进程或计时器能力，应按宿主 Shell 级别审查。

[当前实现] Host sandbox 会隐藏或重定向部分 Node 全局对象，并通过诸如 `ctx.fs`、`ctx.web`、`ctx.bash` 和计时器服务暴露能力；这不构成运行不可信代码的安全沙箱。Client renderer 的纯函数或页面隔离同样不构成权限、URL、HTML、Schema 或 XSS 安全边界。

遵守 [AGENTS.md](AGENTS.md#安全规则) 的输入校验、凭据最小化和资源清理规则：

- 不把模型输入未经校验地拼接到 Shell、URL、文件路径或命令；
- 不向动态 Host half 注入不必要的凭据；
- 不把动态插件当作第三方不可信代码的隔离层；
- 确保 handler、Fiber、监听器、计时器、网络请求和后台任务拥有明确的清理路径；
- 对 Host/Client 之间传输的数据使用 JSON-safe、Schema 校验的边界。

## AI Agent 执行流程

[仓库惯例] 开始编写动态插件前执行以下步骤：

1. 阅读本文以及 `AGENTS.md` 的安全、生命周期和测试要求。
2. 阅读目标版本的 Host Runner、Client Runner 和 Tool README。
3. 按确定顺序发现和查询能力：
   - 先调用 `cordis_inspect_list`，获取当前 Host 已知的 Provider、平台、只读方法及其输入/输出 Schema；
   - 再使用上一步返回的精确 `platform`、`provider`、`method` 和符合 Schema 的 `input` 调用 `cordis_inspect_query`；不要猜名称，也不要把 Provider 名称当作业务 Service 名称；
   - 查询已有动态 Plugin 时调用 `cordis_inspect_self`：不带 ID 列出当前 Session 的 Plugin 摘要；只带 `pluginId` 读取版本指针、最近 Run 和 Package 摘要；同时带 `pluginId` 与 `packageId` 读取不可变 Package 源码和诊断；
   - 不要把 `cordis_inspect_query` 当作动态 Plugin inventory 工具，也不要把 Inspect 结果当作业务运行时数据。
4. 先决定是否真的需要动态插件；如果需要长期安装、发布或跨进程恢复，改用普通 Package/Bundle。
5. 明确只使用 Host half、只使用 Client half，还是同时使用两者。
6. 为每个 Host handler、Fiber、事件监听器、计时器和后台异步任务设计 disposer 或停止路径。
7. 对 Client half 分别验证加载、激活、Guard 和实际渲染，不以“文件存在”或“构建成功”代替页面验收。
8. 运行后检查 `pluginId`、`packageId`、`pluginRunId`、`currentPackageId`、`nextPackageId`、`activeRun` 和 `latestRun`，不要只依据一条成功消息判断完成。

## 验证清单

在声明动态插件完成前，至少覆盖以下场景：

### 定义与版本

- [ ] `name` 或 `purpose` 为空时定义失败。
- [ ] Host 和 Client 代码都缺失时定义失败。
- [ ] Host/Client 语法预检失败时不会创建可运行的有效 Package。
- [ ] 同一 Plugin 可以拥有多个不可变 Package。
- [ ] `run` 与 `update` 模式选择正确。
- [ ] 更新失败后重新检查 `currentPackageId`、`nextPackageId`、`activeRun` 和 `latestRun`。
- [ ] 不把失败更新写成事务性回滚或无中断升级。

### Host half

- [ ] Host-only Package 可以运行、停止并再次运行。
- [ ] Host half 返回合法 Cordis Plugin 或 `apply(ctx)` 对象。
- [ ] Host half 返回非法值时得到结构化失败。
- [ ] Host Fiber 进入预期的 `running`、`waiting` 或失败状态。
- [ ] 缺失服务时正确区分等待和失败。
- [ ] `stop` 后 handler、Fiber 和相关资源完成清理。
- [ ] `vmTimeoutMs` 只被当作同步 VM 评估限制验证。

### Client half

- [ ] 首次运行的审批流程。
- [ ] 已授权后续版本的运行流程。
- [ ] 用户拒绝。
- [ ] 请求创建前的取消。
- [ ] 请求创建后不把 `AbortSignal` 错误描述成自动撤销，除非目标版本有明确持续监听实现。
- [ ] 无浏览器页面时的待处理状态。
- [ ] Client 代码求值失败。
- [ ] Client 模块导入失败。
- [ ] Client Plugin 激活失败。
- [ ] Client Guard 拒绝。
- [ ] UI 渲染失败及其 `client-render` 诊断。
- [ ] 多个页面竞争回答时只有有效的第一个回答生效。
- [ ] 旧 `pluginRunId`、旧 `packageId` 或 stale 结果被拒绝。
- [ ] 页面附着失败不会错误停止其他页面正在使用的 Host Run。

### Session 与进程

- [ ] 非拥有 Session 不能运行、停止、删除或读取受保护的 Package 源码。
- [ ] 进程级 `inventory` 与 Session-scoped inspect 的行为区分正确。
- [ ] 进程重启后动态定义不存在。
- [ ] 页面重载后必须重新运行以恢复 Client half。
- [ ] 不假定源码可以从 Session 日志恢复。

### 安全与资源

- [ ] 动态 Host half 按宿主 Shell 级别审查。
- [ ] 不把 `node:vm` 作为不可信代码沙箱。
- [ ] 所有 handler、Fiber、定时器、监听器和后台任务均有清理路径。
- [ ] 动态工具输入经过 Schema 和 JSON-safe 边界校验。
- [ ] 安全决定在真正执行操作的位置强制，不能只依赖工具可见性或 UI 隐藏。

## 版本说明

本文的 `[当前实现]` 段落按以下文档验证基线核对：

- Harness revision：`3bbc502cd2b122fe7d1c4a4562ae8cd51ca1c1a7`
- Node：`^22.19.0 || >=24.0.0`
- pnpm：`11.7.0`

该 revision 是验证基线，不自动等同于使用者当前工作树的 HEAD。若目标源码 revision 不同，先按 [AGENTS.md](AGENTS.md#文档与源码基线) 的规则重新核对本文所有 `[当前实现]` 段落。

## 事实来源与验证顺序

动态插件行为按以下优先级核对：

1. 目标版本的类型声明：
   - `packages/extensions/cordis-host-runner/src/types.ts`
2. Host Runner 实现：
   - `packages/extensions/cordis-host-runner/src/index.ts`
   - `packages/extensions/cordis-host-runner/src/lifecycle.ts`
   - `packages/extensions/cordis-host-runner/src/sandbox.ts`
3. Client Runner 实现：
   - `packages/extensions/cordis-client-runner/src/client/orchestrator.ts`
   - `packages/extensions/cordis-client-runner/src/client/runtime.ts`
4. 生命周期、沙箱、版本和 Client 测试：
   - `packages/extensions/cordis-host-runner/tests/`
   - `packages/extensions/cordis-client-runner/tests/`
5. 包 README：
   - `packages/extensions/cordis-host-runner/README.md`
   - `packages/extensions/cordis-client-runner/README.md`
   - `packages/extensions/tool-cordis/README.md`
6. 模型工具实际 Schema：
   - `docs/tool-catalog.md`

源码、类型和测试与本文冲突时，以目标版本源码和测试为准，并在同一修改中更新本文的证据标签和版本说明。