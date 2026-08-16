# 插件打包与安装指令

本文供 AI 在打包、安装、升级或卸载 DeepSeek Harness 插件时使用。开发行为规则见 [AGENTS.md](AGENTS.md)。

> 文中行为使用以下证据等级：
> - `[公共契约]`：有类型声明和稳定测试或集成边界支持，可直接依赖。
> - `[仓库惯例]`：项目推荐做法，偏离时必须说明原因。
> - `[当前实现]`：已在源码基线 `3bbc502cd2b122fe7d1c4a4562ae8cd51ca1c1a7` 中确认，升级版本时需要重新核对。
> - `[待验证]`：当前没有足够证据，不得写成无条件保证。


## 概念

- **Plugin**：一个 Cordis JS/TS 模块，导出 `apply` 或 Service 类。
- **Bundle**：一个 npm 包，通过 `package.json` 的 `dsh.bundle.patch` 声明它贡献的 Patch。
- **Profile**：`$DSH_HOME/profiles/<name>` 下的可启动组合，记录依赖、Bundle 顺序和用户 Patch。
- **Patch**：配置层；可插入、禁用或按 `id` 覆盖 Cordis Entry。

### 名称边界

- **Package name**：`package.json#name`，用于包管理器和依赖记录。
- **Bundle name/identity**：由 `dsh.bundle.patch` 和 Bundle 清单识别。
- **Profile name**：例如 `demo`、`web`，用于选择 Profile 目录和组合树。
- **Application name**：Profile 中实际挂载的应用/Entry 标识。

这些名称可以相同，但不能默认相同。验证时分别检查 Profile 目录、依赖包名、Bundle Patch 和最终 Entry。

[仓库惯例] 本文的 Plugin、Package、Bundle、Profile 和 Patch 默认指普通插件/安装语境。若任务涉及 `cordis_define`、动态 Host/Client half、`cordis_run` 或浏览器审批，改读 [DYNAMIC-PLUGINS.md](DYNAMIC-PLUGINS.md)，不要把动态 Package 当作可安装 npm Package。


[当前实现] 在目标 CLI 支持该安装来源、Package 正确声明 Bundle 元数据、Profile 状态允许更新且安装命令成功完成的条件下，CLI 会将识别到的 Bundle 纳入目标 Profile 的组合结果。

“纳入 Profile”只表示配置或依赖组合发生变化，不等同于：

- App 已经启动；
- Plugin Fiber 已经 ACTIVE；
- 所有资源已经创建；
- 旧版本运行时已经停止；
- 卸载后不会因其他配置来源再次出现。

正式外部安装路径是：Plugin 构建成 npm Package → Package 声明 Bundle → `dsh plugin --profile <name> add <spec>` 安装 → 通过 `--dump-config` 和一次实际启动分别验证配置层和运行时层。

## 命令环境与前置条件

本文中的 `<name>`、`<spec>`、`<path>`、`<package>` 和其他尖括号内容都是占位符，必须替换为实际值，不能原样执行。示例中的 Profile 名、包名和路径仅用于说明。

源码启动命令默认相对于以下 Harness 源码根目录执行：

```text
D:\tools\deepseek-harness-master\deepseek-harness-master
```

[当前实现] 当前源码基线要求：

- Node.js：`^22.19.0 || >=24.0.0`
- pnpm：`11.7.0`（根 `package.json` 的 `packageManager`）
- Windows：优先使用仓库提供的 `.cmd` shim 或 `pnpm.cmd`，并确认 `pnpm` 位于 PATH。

先确认命令实际来自预期环境：

```powershell
node --version
pnpm --version
dsh --version
```

源码开发启动方式是：

```powershell
Set-Location 'D:\tools\deepseek-harness-master\deepseek-harness-master'
node --import tsx/esm apps/cli/src/bin.ts --profile demo
```

该命令依赖源码布局和 `tsx/esm` hook。发布 Bundle 不应默认指向仓库内 `.ts` 绝对路径；发布包应指向已构建且已纳入 `files`/`exports` 的 JavaScript 入口。`pnpm dsh ...` 与直接命令只有在同一仓库根目录、依赖已安装、脚本参数转发未改变、Profile 和环境变量相同的条件下才可视为等价；验证时比较 `--help`、Profile 解析、`--patch` 顺序、`--dump-config` 和实际 Fiber/Entry。

## 配置层顺序

[当前实现] CLI 的通用组合顺序为：

1. Bundle patches：按 `dsh.profile.bundles` 顺序；
2. Profile 自身 `cordis.patch.yml`；
3. Home patch：`$DSH_HOME/cordis.patch.yml`；
4. CLI `--patch` overlays：按 argv 出现顺序；
5. 当前 CLI 的内置修正（例如特定 Profile 的 agent presets roots 或 telemetry switch）。

第 5 项不是面向插件作者的通用 Patch API，而是当前 CLI 实现细节。Bundle 顺序会影响最终组合结果，不能仅按 YAML 文件中的行顺序推断加载顺序。后层按 identity 覆盖前层时，目标 `config` 整体替换，不做深度合并；覆盖时重述要保留的全部字段。

`--patch <path>` 是可重复的单值参数，多个参数按 argv 顺序收集：

```powershell
dsh --profile demo --patch .\a.yml --patch .\b.yml
```

`--dump-config` 输出包含用户层和 overlays 的组合树；`--dump-default-config` 只输出 Bundle 层，不接受 `--patch`。两者互斥，dump 模式不启动 App，也不接受 App 参数。执行前先运行 `dsh --help`，不要把未在当前 CLI 参数解析器中存在的命令写入完成标准。

源码依据：`apps/cli/src/profile-boot.ts:121-170`、`apps/cli/src/args.ts:58-100`、`132-165`。

## 开发期本地加载

推荐结构：

```text
my-dsh-plugin/
├── package.json
├── tsconfig.json
├── src/
│   └── index.ts
├── tests/
├── dev.patch.yml
└── cordis.patch.yml
```

源码开发可以在明确启用 `tsx/esm` 的 launcher 下使用 TypeScript 入口；这不是发布包的通用加载契约。`dev.patch.yml` 可以使用插件源码绝对路径：

```yaml
- insert:
    - id: acme:lookup
      name: 'D:/work/my-dsh-plugin/src/index.ts'
      config:
        endpoint: 'http://127.0.0.1:9000'
```

从 Harness 源码运行：

```powershell
pnpm dsh web --patch D:/work/my-dsh-plugin/dev.patch.yml
```

发布前必须确认：`package.json#exports` 指向存在的构建产物；tarball 的 `files` 包含入口、Patch 和静态资源；Bundle Patch 不依赖调用者机器上的绝对路径；clean profile 可以从安装后的包启动。Windows YAML 路径使用引号和正斜杠。

检查最终配置：

```powershell
pnpm dsh --profile web --dump-config --patch D:/work/my-dsh-plugin/dev.patch.yml
```

## Bundle Package 模板

### `package.json`

下面是结构模板，版本范围和构建工具必须按当前宿主与插件实际情况填写：

```json
{
  "name": "@acme/dsh-lookup-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "./lib/index.js",
  "types": "./lib/types/index.d.ts",
  "exports": {
    ".": {
      "types": "./lib/types/index.d.ts",
      "default": "./lib/index.js"
    }
  },
  "files": [
    "lib/index.js",
    "lib/types/**/*.d.ts",
    "cordis.patch.yml"
  ],
  "dsh": {
    "bundle": {
      "patch": "./cordis.patch.yml"
    }
  },
  "peerDependencies": {
    "@deepseek-ai/cordis": "<verified-compatible-range>",
    "@deepseek-ai/dsh-tools": "<verified-compatible-range>"
  },
  "devDependencies": {
    "@deepseek-ai/cordis": "<verified-compatible-range>",
    "@deepseek-ai/dsh-tools": "<verified-compatible-range>",
    "typescript": "<chosen-version>"
  },
  "scripts": {
    "build": "<build-command>"
  }
}
```

规则：

- 生产入口是普通 Node 可加载的 ESM JavaScript，不只发布 TypeScript。
- Bundle Patch 和所有 export 子路径文件进入 `files`。
- 普通运行时第三方依赖放 `dependencies`。
- Cordis 和宿主 Service Definition 使用兼容 peer，避免安装第二份框架或服务类型。
- Git 安装若依赖构建后的 `lib`，提供自包含 `prepare`；它不能依赖旁边存在 Harness monorepo。
- npm/tarball 发布应预先构建，尽量避免让使用者执行安装脚本。
- 不照抄占位版本范围；从当前 dsh 安装的 package manifest 验证。

### `cordis.patch.yml`

```yaml
- insert:
    - id: acme:lookup
      name: '@acme/dsh-lookup-plugin'
      config:
        endpoint: !!js process.env.ACME_SEARCH_URL
        timeoutMs: 30000
```

Entry `id` 使用稳定、命名空间化名称。Patch 中引用子路径时，确保 `package.json#exports` 暴露它：

```yaml
- insert:
    - id: acme:startup
      name: '@acme/dsh-lookup-plugin/startup'
### `!!js` 执行边界

[当前实现] `!!js`/`!js` 节点会由 Loader 在配置插值时求值；配置 reload、reparse 或重复加载可能导致表达式再次执行。表达式必须是纯的、确定的、快速的值计算。禁止在其中执行网络请求、Shell/子进程、写文件或修改 Profile、读取并输出凭据，或注册不可回收的 listener、timer 和资源。需要一次性初始化或资源清理时，使用插件生命周期和 `ctx.effect()`，不要隐藏在 Patch 表达式中。

```

## Profile 文件

默认 Harness Home 是 `~/.dsh`；设置 `DSH_HOME` 后使用该目录。Windows 通常为 `%USERPROFILE%\.dsh`。

```text
$DSH_HOME/
├── cordis.patch.yml
└── profiles/
    ├── node_modules/              # dsh 管理的宿主模块 fallback
    └── demo/
        ├── package.json
        ├── pnpm-workspace.yaml
        ├── pnpm-lock.yaml
        ├── cordis.yml
        ├── cordis.patch.yml
        └── node_modules/
```

规则：

- 用户修改 Profile 的 `cordis.patch.yml`。
- 不编辑 Profile 的 `cordis.yml`；启动器会把它维护为空根并把它用作模块解析锚点。
- 不手工复制包到 `$DSH_HOME/profiles/node_modules`；该目录由 dsh 维护 symlink/junction。
- Profile 的 `pnpm-workspace.yaml` 使用 hoisted linker 和 `autoInstallPeers: false`，用于让外部插件共享宿主 Cordis。
- 不随意改 linker 或 peer 安装策略。

默认 Bundle：

```text
web      → dsh-base + dsh-web-app
headless → dsh-base + dsh-headless
其他新 Profile → dsh-base
```

## 安装来源
[当前实现/仓库惯例] 安装来源的语义和验收要求：

| 来源 | 主要语义 | 适用场景 | 验收要求 |
|---|---|---|---|
| `file:` | 本地包引用，是否复制/链接由包管理器和配置决定 | 可复现本地验证 | 检查 lockfile 和实际解析路径 |
| `link:` | 工作区/本地链接语义 | 开发和 HMR | 不作为发布验收唯一证据 |
| 普通相对路径 | CLI 按调用目录锚定后传给 pnpm | 快速本地操作 | 验证调用目录和最终依赖记录 |
| npm 固定版本 | registry 包 | 发布安装 | 核对 resolved version/integrity |
| Git commit SHA | 固定源码版本 | 源码依赖 | 核对 commit、prepare/build 和 lockfile |
| tarball | 预构建发布物 | 发布验收 | 解包检查 manifest、exports、Patch、JS 入口 |


### 本地目录

在插件目录的父目录执行：

```sh
dsh plugin --profile demo add ./my-dsh-plugin
```

CLI 会把相对路径锚定到调用目录，再在 Profile 目录运行 pnpm。也可显式使用：

```sh
dsh plugin --profile demo add file:./my-dsh-plugin
dsh plugin --profile demo add link:./my-dsh-plugin
```

`link:` 指向可变工作副本，适合本地迭代和 HMR 验证，不作为发布验证、部署记录或可复现安装证据。可复现交付使用固定 npm 版本、固定 Git commit SHA 或预构建 tarball；`file:` 同样需要通过 lockfile 与实际包内容确认来源。

Windows PowerShell：

```powershell
dsh plugin --profile demo add .\my-dsh-plugin
```

### npm

固定明确版本：

```sh
dsh plugin --profile demo add @acme/dsh-lookup-plugin@0.1.0
```

安装前检查包名、发布者、源码、provenance、生命周期脚本和 tarball 内容。

### Git

固定 commit SHA：

```sh
dsh plugin --profile demo add github:owner/repository#<commit-sha>
```

实际 Git spec 以 pnpm 当前支持格式为准。Git 安装通常获取源码；若 Package 入口需要 `lib`，必须有 `prepare`。pnpm 10+ 可能阻止构建脚本，并要求在 Profile 的 `pnpm-workspace.yaml` 中添加精确 `allowBuilds` 键。授权前审查脚本；该代码在宿主用户权限下运行，不在 Agent 沙箱中。

### Tarball

作者：

```sh
pnpm run build
pnpm pack --dry-run
pnpm pack
```

使用者：

```sh
dsh plugin --profile demo add D:/packages/acme-dsh-lookup-plugin-0.1.0.tgz
```

检查 tarball 包含 manifest、Patch、构建 JS、声明文件、所有 exports 和静态资源。预构建 tarball 仍可能携带生命周期脚本和恶意运行时代码；校验哈希并使用可信渠道。

## 安装后验证

按顺序执行：

```powershell
dsh plugin --profile demo why @acme/dsh-lookup-plugin
dsh plugin --profile demo list
dsh --profile demo --dump-config
dsh --profile demo
```

| 检查 | 命令/方法 | 通过标准 |
|---|---|---|
| 包存在 | `dsh plugin --profile demo list` | 目标包和版本存在 |
| Bundle 可识别 | 检查 `dsh.bundle.patch` | Bundle 被识别而不是普通依赖 |
| 组合树 | `dsh --profile demo --dump-config` | 目标 Entry 出现在最终树中 |
| 默认树 | `dsh --profile demo --dump-default-config` | 只验证 Bundle 层，不误把用户层当默认层 |
| 实际启动 | `dsh --profile demo` | App/Fiber 达到预期状态 |
| 功能验证 | 最小调用或测试 | 工具/Provider/Renderer 可工作 |
| 停止验证 | 退出、重载或明确 stop 流程 | 资源释放，无重复 listener/子进程 |

每次报告必须注明命令是否真正执行；不能把“命令存在”写成“命令通过”。安装成功但没有 `dsh.bundle` 的包只是普通依赖，不会自动成为配置层。

## 更新与删除

固定版本更新必须同时记录目标版本和实际解析版本：

```powershell
dsh plugin --profile demo update @acme/dsh-lookup-plugin
pnpm --dir "$env:DSH_HOME\profiles\demo" list @acme/dsh-lookup-plugin
dsh --profile demo --dump-config
dsh --profile demo
```

验收至少包括：依赖版本、lockfile、Bundle Patch、最终 Entry、Fiber 状态和关键功能。不能只根据 `pnpm update` 的退出码判断插件已经激活。目标版本应使用明确版本、lockfile 或 commit SHA，并写出允许升级范围。

删除时使用安装后 manifest 中的真实包名：

```powershell
dsh plugin --profile demo remove @acme/dsh-lookup-plugin
```

[当前实现] `dsh plugin` 先在 Profile 目录执行 pnpm；只有 pnpm 成功退出后，CLI 才执行 Bundle reconcile。当前实现没有足够证据证明“pnpm 成功 + Bundle reconcile + Patch 更新”是完整事务，不要把安装失败写成必然自动回滚。Bundle 通过 `dsh.bundle.patch` 识别，并按真实包名去重。删除依赖后，如果 Bundle 仍在组合清单中，下一次安装/更新调和可能把它重新加入。

删除 Package 不等于移除 Bundle 声明，也不等于停止当前运行时。完整删除至少分为：

1. 从 Profile dependencies 移除包；
2. 从 `dsh.profile.bundles` 移除 Bundle；
3. 停止或重载当前 Profile；
4. 确认 Fiber、listener、任务、子进程和连接已经释放；
5. 重新 dump/启动验证不再出现该 Entry。

源码依据：`apps/cli/src/plugin.ts:48-67`、`115-145`。

## 模块解析

Bundle 名称按两个锚点解析：

1. 当前 dsh 安装。
2. Profile 目录。

内置 Bundle 优先使用当前 dsh 安装版本。Patch 中普通插件包从 Profile `node_modules` 解析，再沿父目录找到 dsh 维护的共享 fallback。外部插件把 Cordis/Service Definition 声明为 peer，才能复用宿主实例。

重复 Cordis 的典型症状：

- Context 类型或服务实例不一致。
- 注册存在但消费方看不到。
- `instanceof` 或模块单例状态异常。
- Loader Fiber 长期 PENDING。

发现此问题时检查 `dependencies`/`peerDependencies`、Profile linker、lockfile 和 `pnpm why`。

## 常见安装故障
### 构建可重复性与环境变量

发布验证必须记录源码 revision、Node/pnpm 版本、lockfile 是否干净、构建命令和工作目录、tarball 文件列表和大小、package `exports`、Bundle Patch 路径、`--dump-config` 与实际启动结果。构建成功不等于构建可复现；至少在 clean profile 或干净安装目录重新安装并启动一次。

需要影响一次启动组合的环境变量，应在 Profile 组合前读取并记录快照。不要假设运行中修改环境变量会重新组合已经加载的 Patch。对 `DSH_HOME`、`DSH_TELEMETRY_DISABLED` 等变量，记录读取时机、原始值、是否产生 Patch、是否在 reload 中重新读取。


### pnpm 不在 PATH

```text
dsh: pnpm not found on PATH
```

确认：

```sh
pnpm --version
```

Windows 可运行：

```powershell
Get-Command pnpm
```

### 安装成功但未激活

检查：

- Package 是否声明 `dsh.bundle.patch`。
- Patch 文件是否进入 tarball。
- Patch 顶层是否为 YAML 数组。
- Bundle 名是否可解析。
- `--dump-config` 是否显示 Entry。

### 空 Patch 解析失败

空文件或只有注释不是合法空列表。禁用一层时写：

```yaml
[]
```

### Patch 覆盖导致字段丢失

原因是 `config` 整体替换。读取被覆盖 Bundle 原行，复制全部保留字段后再修改目标字段。

### Git 安装后缺少 `lib/index.js`

检查：

- 是否提供 `prepare`。
- `prepare` 是否自包含。
- pnpm 是否因 `allowBuilds` 拒绝执行。
- `files`/`exports` 是否指向真实产物。

更稳妥的解决方式是发布预构建 npm 包或 tarball。

### 插件长期 PENDING

`PENDING` 不是通用错误判定。检查 Entry 是否加载、Provider 是否注册、Fiber 状态、依赖是否满足、是否存在初始化异常，以及 Host/Client 面是否匹配；不要把短暂等待自动解释为安装失败，也不要把长期 PENDING 自动解释为正常。

### Windows junction 冲突

若 `$DSH_HOME/profiles/node_modules/<package>` 是普通目录而不是 dsh 管理的链接，启动会失败。确认内容和来源后移走冲突目录，让 dsh 重建 fallback；不要递归删除可能指向重要目标的 junction。

## 安装安全检查表

安装前：

- [ ] 来源可信，npm 版本或 Git SHA 已固定。
- [ ] 已审查 `package.json` 的 preinstall/install/postinstall/prepare。
- [ ] 已审查 `cordis.patch.yml`、`!!js` 和所有 Entry 名称。
- [ ] 已审查子进程、MCP、网络、文件和凭据访问。
- [ ] 已审查实际 tarball，而不只看源码仓库。
- [ ] Cordis 与 DSH Service Definition 使用兼容 peer。
- [ ] 需要 `allowBuilds` 时理解其宿主代码执行权限。

安装后：

- [ ] `pnpm-lock.yaml` 记录符合预期。
- [ ] `--dump-config` 没有意外覆盖或禁用。
- [ ] 已按源码、生命周期脚本、Patch 和运行时行为核对插件实际访问；Harness 不提供声明式的逐插件权限隔离，不要把文档中的权限声明当作安全边界。
- [ ] 对不可信插件使用操作系统账户、独立进程、容器或其他真正的隔离边界。
- [ ] 卸载后进程、连接、Watcher 和注册项全部消失。
- [ ] 没有凭据进入日志、错误、快照或模型结果。

## 发布完成标准
[仓库惯例] 数据保留、字段迁移和多版本策略由具体插件、Profile 或发布流程定义。若插件依赖这些策略，必须在自己的 Schema、迁移和验收脚本中明确实现；本指南只规定安装、组合和启动验证边界。


作者必须在干净环境证明：

```text
build
→ unit + disposal tests
→ pnpm pack --dry-run
→ install into clean profile
→ dump config
→ start real profile from built package
→ exercise external behavior
→ remove package
→ verify contributions disappear
```

将兼容的 dsh/Cordis 版本、所需权限、配置字段、凭据来源、安装方式、平台限制和卸载行为写入插件 README。

## 文档与源码基线

本文的 `[当前实现]` 段落按以下文档验证基线核对：

- Harness revision：`3bbc502cd2b122fe7d1c4a4562ae8cd51ca1c1a7`
- Node：`^22.19.0 || >=24.0.0`
- pnpm：`11.7.0`

该 revision 是验证基线，不自动等同于使用者当前工作树的 HEAD。若目标源码 revision 不同，先按 [AGENTS.md](AGENTS.md#文档与源码基线) 的规则重新核对本文所有 `[当前实现]` 段落。四份指南必须保持同一验证基线；修改 CLI、Patch 引擎、Bundle 或 Profile 行为时同步复核 [AGENTS.md](AGENTS.md)、[REFERENCE.md](REFERENCE.md) 和 [DYNAMIC-PLUGINS.md](DYNAMIC-PLUGINS.md)。

升级 Harness、Node、pnpm、Cordis、Loader 或 Patch 引擎后，必须重新核对本文标记为 `[当前实现]` 的内容。

## 不要这样写

以下表述禁止在没有直接证据时使用：卸载包后插件已经停止；配置热重载失败一定保留旧实例；`--patch` 是事务性的；Bundle 删除后不会再次自动加入；所有插件都有统一 retention/deprecation/multi-version 契约。

## AI Agent 执行模板

### 执行前
- [ ] 确认源码 revision、Node、pnpm、Profile 和工作目录。
- [ ] 区分 Package、Bundle、Profile、Patch、Fiber 和运行时资源。

### 执行中
- [ ] 记录最终 Entry、identity、环境变量快照和实际安装来源。
- [ ] 不把包管理动作当作运行时停止，不在 `!!js` 中隐藏副作用。
- [ ] 不在未验证时假设事务回滚、原子性、排序或自动去重。

### 执行后
- [ ] dump 配置、启动实际 Profile、验证功能和资源释放。
- [ ] 记录实际运行的命令、输出、源码版本和未执行项目。
