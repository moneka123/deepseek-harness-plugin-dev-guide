# DeepSeek Harness 插件开发技术规范与实现参考

> 面向**使用 AI Agent 开发 DeepSeek Harness 插件的开发者**。本文档集为人类开发者提供任务约束、源码核对路径与验收标准，也为 AI 编程 Agent 提供可执行的工程上下文，帮助双方以可验证、可维护的方式扩展 Harness。

## 简介

本仓库目录提供一套用于**人类开发者借助 AI 编程 Agent 开发 DeepSeek Harness 插件**的技术规范与实现参考。它不是对 API 的静态摘录，而是一份可直接提供给 Agent 作为任务上下文的工程化工作指南；源码、类型、实际导出和测试始终是最终事实来源。

文档覆盖以下核心问题：

- 如何在 `tools`、`systemPrompt`、`agent`、`llm` 与 `jobs` 等扩展点之间作出合适选择；
- 如何基于 Cordis `Fiber`、`ctx.effect()` 与 disposer 建立可取消、可卸载、可热重载的资源生命周期；
- 如何使用动态 Cordis 运行时完成 `define` / `run` / `stop` / `undefine`，并理解 Host / Client 双端沙箱与浏览器审批的编排边界；
- 如何通过 Bundle、Patch、`cordis.patch.yml` 与 Profile 管理配置覆盖、打包安装、升级和卸载；
- 如何识别 `node:vm` 非安全沙箱、工具隐藏不等于授权、凭据最小化等安全限制；
- 如何用 P0～P2 分级测试、HMR 卸载验证和 `--dump-config` 审计完成交付验收。

适用对象：

- 使用 AI 编程 Agent 设计、实现、审查或维护 DeepSeek Harness 插件的开发者；
- 需要在真实仓库中执行明确任务、核对源码并完成验证的 AI 编程 Agent；
- 需要理解 Cordis、Bundle、Patch 与 Profile 装配关系，并对最终交付负责的集成维护者。

---

## 文档导航与推荐阅读顺序

本目录包含四份相互关联的核心文档：

| 文档 | 用途 | 何时阅读 |
| --- | --- | --- |
| [AGENTS.md](AGENTS.md) | 插件开发总则：扩展策略、生命周期、配置、安全、测试与验收要求。 | **开始任何插件任务前必读。** |
| [REFERENCE.md](REFERENCE.md) | 扩展点与运行时参考：`tools`、`systemPrompt`、`agent`、`llm`、事件、服务、MCP、Session 等。 | 选择扩展机制、核对调用约定或设计接口时阅读。 |
| [INSTALLATION.md](INSTALLATION.md) | Bundle / Package / Profile / Patch 的关系，以及打包、安装、升级、卸载和配置审计流程。 | 交付、部署、升级或排查配置装配问题时阅读。 |
| [DYNAMIC-PLUGINS.md](DYNAMIC-PLUGINS.md) | 动态 Cordis Package 指南：`define`、`run`、`stop`、`undefine`、Host / Client Runner 和审批流程。 | 使用 `ctx.dynamicCordisRunner`、`@deepseek-ai/dsh-tool-cordis` 或动态 Host / Client 代码时阅读。 |

### 推荐路径

1. **先读 [AGENTS.md](AGENTS.md)**：确定边界、证据等级、生命周期与安全要求。
2. **按任务分流**：
   - 设计扩展点与 API：读 [REFERENCE.md](REFERENCE.md)；
   - 打包、安装、升级：读 [INSTALLATION.md](INSTALLATION.md)；
   - 动态运行时或浏览器端插件：读 [DYNAMIC-PLUGINS.md](DYNAMIC-PLUGINS.md)。
3. **回到目标 Harness 源码核对**：类型、实现、测试、包 README 和实际导出优先于本文档中的概括说明。
4. **完成后执行验证**：审计最终配置、验证生命周期清理，并记录使用的源码 revision 与命令结果。

---

## 快速开始

### 1. 将本目录作为 Agent 的任务上下文

在要求 AI Agent 开发或修改插件前，先向它提供本目录及目标 Harness 源码路径，并明确任务目标、允许修改的范围、运行环境与验收命令。Agent 应先阅读 [AGENTS.md](AGENTS.md)，再按任务查阅其余专题文档；不要只依据自然语言描述推断运行时行为。

### 1. 确认源码与文档基线

本目录的 `[当前实现]` 标注绑定到指定 Harness revision。开始工作前，应先在目标源码仓库确认当前 `HEAD`：

```powershell
git rev-parse HEAD
```

若结果与本文档的基线不一致，请将相关结论视为待复核项，并以当前版本的源代码、类型声明、测试和命令输出为准。

### 3. 根据任务选择文档

- **新增普通插件、Tool 或服务**：从 [AGENTS.md](AGENTS.md) 开始，再查 [REFERENCE.md](REFERENCE.md)。
- **制作 Bundle、Package、Patch 或 Profile**：阅读 [INSTALLATION.md](INSTALLATION.md)。
- **定义并运行动态 Cordis Package，或涉及 Host / Client 双端代码**：阅读 [DYNAMIC-PLUGINS.md](DYNAMIC-PLUGINS.md)。

### 4. 优先选择最小侵入式扩展方式

优先复用现有配置、Profile、Home/命令层或 Patch；仅在这些手段不能满足需求时，才引入独立插件、独立 Bundle、新 Package 或核心运行时修改。避免为了一个局部需求直接修改 `apps/`、核心服务或 Agent Loop。

### 5. 实现时绑定资源生命周期

所有订阅、定时器、后台任务、文件句柄、Socket、Client 注入和临时状态都应拥有明确的 disposer。使用 Cordis 的 `ctx.effect()`、`Fiber` 和取消信号时，应验证：启动、取消、卸载、热重载和异常退出后资源都能被正确清理。

### 6. 验收配置与运行结果

至少使用 `--dump-config` 或相应 CLI 输出检查最终 Entry、Profile、Patch 和覆盖结果；再根据变更风险执行 P0～P2 对应测试。涉及动态插件或 HMR 时，必须额外验证停止与卸载后没有残留服务、工具、事件监听器或浏览器端注入。

---

## 技术亮点

1. **面向真实扩展点的决策框架**  
   覆盖 `tools`、`systemPrompt`、`agent`、`llm`、`jobs`、事件与服务，并强调按目标能力选择最小、最稳定的接入点。

2. **以 Fiber 为中心的资源生命周期**  
   将 `ctx.effect()`、disposer、取消传播与 Cordis `Fiber` 联系起来，避免插件在卸载、重载或失败后留下后台任务与监听器。

3. **动态 Cordis 双端编排**  
   系统说明 `define` / `run` / `stop` / `undefine` 的状态变化，以及 Host Runner、Client Runner、浏览器审批与双端沙箱的职责边界。

4. **配置装配可审计**  
   明确 Bundle、Package、Profile、Patch 的层级关系，解释 `cordis.patch.yml` 覆盖规则与 Profile 装配顺序，并将 `--dump-config` 作为最终配置事实检查手段。

5. **安全结论不被“便利性”掩盖**  
   明确 `node:vm` 不是安全沙箱；隐藏 Tool 不等同于权限控制；外部输入、动态代码与凭据必须采用最小权限与显式验证策略。

6. **测试按风险分级**  
   用 P0～P2 区分阻断性核心路径、重要集成路径和补充质量检查；将真实 Profile 启动、卸载清理、HMR 与配置审计纳入验收。

7. **证据等级与版本漂移控制**  
   使用 `[公共契约]`、`[仓库惯例]`、`[当前实现]`、`[待验证]` 区分陈述强度，避免将某个 revision 的实现细节误写为永久承诺。

---

## 文档基线版本

本文档集基于以下 Harness 源码 revision 编写：

```text
3bbc502cd2b122fe7d1c4a4562ae8cd51ca1c1a7
```

该 revision 是所有 `[当前实现]` 标注的验证基线，**不自动等同于你的工作树 `HEAD`**。当 Harness、Cordis、Loader、Patch 引擎、CLI、Node.js 或 pnpm 版本更新时，应重新核对四份核心文档中的：

- 工具名称与 Schema；
- Bundle / Patch / Profile 装配与覆盖语义；
- Host / Client Runner 和动态 Package 生命周期；
- 扩展点类型、导出与测试路径；
- Node.js、pnpm、CLI 参数和源码路径引用。

建议在每次升级后重新执行仓库已有的文档、工具目录、链接与配置验证命令，并将验证的源码 revision 记录在变更说明中。

---

## 许可证与免责声明

### 许可证

本目录是 DeepSeek Harness 插件开发的参考文档。其所依赖或引用的 DeepSeek Harness 上游源码采用 [MIT License](../deepseek-harness-master/LICENSE)；使用、复制、修改或再分发相关源码时，请遵守上游仓库中的完整许可证与版权声明。

除非本目录另行提供独立 `LICENSE` 文件，本文档本身不应被理解为对上游许可证、第三方组件许可证或产品授权范围的替代说明。

### 免责声明

- 本文档旨在提供工程实践与实现参考，不构成 DeepSeek 或任何第三方的官方支持、服务承诺或安全保证。
- 文档中的 `[当前实现]` 只对“文档基线版本”有效；上游实现、公开 API、CLI、配置格式和运行时行为均可能发生变化。
- 动态代码、浏览器端代码、外部输入、凭据和网络访问具有安全风险。`node:vm` 不应被视为隔离不可信代码的安全边界；Tool 的可见性也不等同于授权机制。
- 使用者应在自身环境中完成权限控制、秘密管理、测试、审计、备份与发布审批，并对生产环境的变更负责。

---

## 贡献指南

欢迎通过 Issue、讨论或 Pull Request 改进本目录的技术准确性、可读性与可验证性。

### 提交问题前

请尽可能提供：

- 使用的 Harness revision、Node.js 和 pnpm 版本；
- 目标操作系统、执行命令与完整错误输出；
- 相关配置片段（请删除 Token、密钥、用户数据及其他敏感信息）；
- 最小复现步骤、预期行为与实际行为；
- 关联的文档位置、源码路径、类型定义或测试证据。

### 改进文档时

1. 将事实陈述与证据等级匹配：公共 API、仓库惯例、当前实现和待验证事项应明确区分。
2. 对涉及运行时行为的修改，优先引用或核对目标 revision 中的类型、实现、测试、包 README 与实际导出。
3. 修改动态 Cordis、Bundle/Patch/Profile、CLI、生命周期或工具 Schema 后，同步检查受影响的其他核心文档。
4. 保持中英文术语与源码命名一致；关键标识符、命令、配置键与路径使用原始英文名称。
5. 不要在文档中承诺未经验证的原子性、回滚、自动清理、权限隔离或兼容性行为。
6. 提交前检查相对 Markdown 链接、代码块、命令示例与版本基线，并运行项目中适用的验证脚本。

### 建议的变更说明格式

```text
范围：AGENTS / REFERENCE / INSTALLATION / DYNAMIC-PLUGINS
基线：<Harness revision>
问题：<现有描述与源码/测试不一致之处>
证据：<源码路径、类型、测试、CLI 输出或生成目录>
修改：<拟调整的文字或示例>
验证：<执行的命令及结果>
```

---

## 使用原则


> 开发者负责定义目标、约束权限并审阅最终改动；AI Agent 负责先读规范、核对源码、执行最小必要改动并报告验证结果。
> 以目标 Harness 源码、实际导出、类型声明和测试结果为最终事实来源；以最小侵入方式实现扩展；以可取消、可卸载、可验证作为交付底线。
