---
title: REA项目研究与Hermes接入评估
created: 2026-10-13
updated: 2026-10-13
type: research
tags: [research, agent, mcp, tooling, security, architecture]
status: draft
sources: ["https://github.com/morluto/rea", "https://registry.npmjs.org/rea-agents/latest"]
---

# REA：逆向分析工具箱，而不是另一个通用 Agent

## 结论

**值得列入专项工具箱，不建议现在全量接入生产 Hermes。** 对老王最有价值的方向，是客户授权的旧 .NET 组件梳理、Electron 应用行为追踪和迁移前依赖调查。没有明确待分析样本时，安装整套原生逆向引擎的收益不高。这是本次选型判断，不是运行效果承诺。

REA 全称 Reverse Engineer Anything，提供共享工作流的 CLI 和 MCP server，把原生二进制、JavaScript/Electron、.NET、网站、Android 等分析能力交给外部 Agent 调用。模型负责提出问题、解释结果和编写实现；REA 负责分析工具调用与证据返回，不能把两者合写成“REA 自动恢复源码”。[4]

调研对象按近期 AI 日报中的 `morluto/rea` 确定。关联：[[40-调研报告/AI技术与产品/README|AI技术与产品]]、[[50-日报与动态/AI日报/2026-10-09|最初关注记录]]。

## 一、版本与证据边界

本轮采集始于北京时间 2026-10-13 01:09。采用三个独立版本坐标，避免把 main 上的功能写成 npm 已发布能力。

- GitHub latest release：`rea-agents-6.4.0`，发布于 `2026-10-11T03:58:03Z`；不是 prerelease。[2]
- npm latest：`6.4.0`，`gitHead=a56f7bb668ab23fec8d33f7af353ac7ed2fe1895`；本地读取 release tag 指向同一 commit。[3]
- 静态架构审查固定 main：`d96550f2f61647ffa150597de73363fa90c4e169`。本文涉及 main 的新增功能单独标注。

GitHub API 当次返回 143,911 stars、33,251 forks，仓库创建于 2026-04-14；这是热度快照，不是质量背书，也不是独立用户量。[1] 网页抽取层同时返回更早的 81.3k stars，与 API 和 clone 不一致，因此不采用抽取页面作为当前版本与热度依据。

一个具体文档陷阱：6.4.0 tag 内的 installation.md 仍保留“检查过的 npm 版本为 6.3.0、不含 Hermes”的历史段落。6.4.0 release notes 明确加入 Hermes，tag 内 `SupportedClients.ts` 也包含 `hermes` 和对应配置/skill 路径；npm 的 gitHead 与该 tag 一致。因此判断为 **6.4.0 已包含 Hermes 注册实现**，而不是沿用旧说明说不支持；本轮没有实际执行注册。[2][5][9]

## 二、工作链路与真正价值

```text
用户提出问题
  → Hermes / Claude Code / 其他 MCP 客户端
  → REA CLI 或本地 stdio MCP
  → 共享分析工作流
  → 静态解析器或 Ghidra / Hopper / IDA 等 provider
  → Evidence：结果、来源、覆盖限制、未知项
  → Agent 追问、解释、重建实现并另行测试
```

CLI 与 MCP 共享分析工作流；原生分析 provider 按可用性与目标支持选取。多个候选时要求明确选择，已选 provider 失败不会悄悄切换，因此“支持多引擎”不等于自动交叉验证。[4][7]

其工程价值不只是给反编译器加一个聊天入口，而是把证据串起来：源码模型包含目标身份、provider、参数、raw/normalized result、位置、限制，以及 `observed / derived / inferred` 三类结论性质。[12] 这比把一大段反编译结果直接交给模型，更利于追问“结论来自哪里、哪些内容没看见”。

但 Evidence 的内容摘要与一致性验证不是数字签名，也不是分析真实性证明；记录自洽不代表模型推断正确，更不代表恢复了原始源码。[12] 重建验证 contract 也明确限定为声明范围内的验证，不证明全局源码等价。[10]

## 三、能做什么，不能顺带承诺什么

### 1. JavaScript / Electron：最容易起步

能够读取目录或 ASAR，分析模块、imports、source maps、路由、IPC 和原生扩展关系；静态分析不执行目标应用，不要求先安装 Ghidra/Hopper。[4]

适合调查桌面应用的剪贴板、搜索、文件打开、renderer → preload → main 交互链。输出是代码与关系证据，不保证恢复压缩前变量名、原始工程结构或服务端实现。

### 2. .NET：与企业 IT 工作关联最强

提供 PE/CLI 元数据、类型/方法签名、CIL、声明的原生调用以及跨版本成员比较；直接读取文件，不加载或执行程序集。[6]

适用实例：客户有授权但源代码遗失的旧程序集，先梳理它调用哪些 DLL、暴露哪些类型、两个版本哪些方法发生变化，再判断迁移或重写范围。**这只是静态技术调查，不能据此证明组件在新平台兼容、许可证允许重写或业务行为完全等价。**

边界也明确：普通 managed 路线不自动拆解 single-file host、不解析 IL2CPP 元数据；NativeAOT 方法体走原生分析，不能按普通 CIL 反编译成原始 C#。NativeAOT 扩展还有特定引擎、运行时、架构和宿主限制。[6]

### 3. 原生程序：能力更深，依赖更重

Hopper、Ghidra 或 IDA 提供伪代码、汇编、字符串、符号、调用及引用；各 provider 的宿主、格式与操作范围不同，不能视为可互换的等价实现。[4][7]

Ghidra 是已有安装接入，需匹配 Ghidra 与 full JDK；Hopper 是另行授权的商业软件，其 demo 有自身限制。REA 的 MIT 不覆盖这些外部工具的收费和许可。[5]

### 4. 网站、捕获、Android 与固件

6.4.0 文档列出浏览器页面/脚本/网络观察、已有 HAR/mitmproxy 捕获、JADX APK、ADB 设备操作、Apktool 资源、JEB 项目以及 Binwalk/Unblob 固件分析；依赖需按目标单独提供。[4] 网站分析只能看到客户端可观察信息，不能凭浏览器结果恢复远端服务的私有源码。

6.4.0 的新增重点包括 ADB、Apktool、JEB、保留证据的摘要/分页、大型 JavaScript 分析进程隔离，以及 Hermes 等客户端注册；release 也列出 breaking changes，不能按小版本号假设所有旧 Evidence 都无缝兼容。[2]

**main-only 提醒：**本次审查的 main 包含 Frida attach/spawn 和注入脚本相关 contracts；相同 contract 文件不存在于 6.4.0 tag，不能据此宣称 npm 6.4.0 已提供这套功能。[13]

## 四、对 Hermes 的增量与选型

Hermes 官方支持外部本地 MCP server 和工具 include/exclude；REA 已有 Hermes 注册实现，接入不要求修改 Hermes 源码。[8][9] 但“协议可接”不代表当前 Copilot 模型、工具 schema、超时和权限均已经验收。

候选路径：

- **先用 CLI，按任务调用——推荐首轮。** 不把整个逆向工具目录常驻暴露给模型；选择一个授权样本，固定 npm 版本，只跑静态分析，结果保存为文件后取摘要。缺点是跨次 CLI 调用要显式保存和传递证据。[15]
- **专用隔离环境 + Hermes MCP 白名单——有复用需求后采用。** 适合连续追踪函数、模块与版本变化；需要管理 provider、上下文体积、超时、数据出境和工具权限。不是直接运行自动 setup 就完成安全接入。
- **继续现有浏览器/源码检索或单一解析器——简单任务更省事。** 有源代码的普通项目、仅需看网页交互时，不应为了使用 REA 再增加整套逆向依赖。这是按业务增量做的判断。

不建议把它放在记忆系统、RAG 或企业业务 Agent 平台的位置；它补的是“理解没有完整源代码的软件”的工具能力，而不是长期记忆、权限目录或业务流程管理。

## 五、风险与部署约束

### 不是安全沙箱

本次审查的 `capture_process_scenario` 明确会在当前用户权限下运行所选命令；环境构造复制宿主环境变量再叠加场景值，源码直接声明不是 security sandbox。[10][11] 临时目录、进程树清理和资源上限解决的是运行管理，不是样本降权。

因此不能让未知程序在持有 Azure、GitHub、邮件等凭据的主机用户下执行。建议把可执行样本放在无个人凭据的可销毁 VM；普通静态分析也需控制样本目录与第三方解析器的风险。此为部署建议，不宣称当前发现了可利用漏洞。

### 本地分析不等于数据不出机

官方明确：分析在本地，但 Agent 收到工具结果，其模型提供商有独立的数据政策。[4] 代码、业务响应、路径和部分原始证据可能进入模型上下文；客户授权分析与客户授权上传是两件事。客户文件必须先确认允许的模型与数据处理边界。

### 工具数量、输出和资源不能忽略

`tools/list` 可以包含当前不可用的工具，实际可用性需检查 `binary_session`；不能只按工具名出现就承诺可执行。[7] 完整应用 Evidence 可能很大，官方 CLI 文档举出数百 MB 文件取约 10 KB 摘要的例子，这是项目方案例，不是本机测量。[15] 首轮应只取摘要与选定模块，保留完整证据文件而不是塞回聊天。

本机只读预检：Node `v22.22.2`、npm `10.9.7`；Node 满足 npm 包声明的引擎范围。[3] 当时内存 available 约 3.7 GB、无 swap；这是环境快照，不是 REA 峰值内存评估。没有做负载测试，不建议据此承诺大型 Electron 应用或 Ghidra 能与现有服务稳定共存。

### 热度与测试状态分开

审查 commit 的 CI 当次仍为 `in_progress`，同 commit 的 Release workflow 已 success；发布流水线成功不能替代 CI、真实 provider 验证或本机测试。[14] 本轮未运行 REA，也没有复现官方展示案例，不能报“测试通过”。

## 六、建议试验与验收标准（尚未执行）

**我的建议：先不常驻部署。出现一份明确的客户授权 .NET DLL 或自己的小型 Electron 样本后，再做一次限范围 CLI 试验。**

首轮只验证这些事项：

1. 固定 6.4.0 和样本摘要；不跟随 `@latest` 自动漂移，不修改 default Hermes。
2. 仅静态解析，不启用进程执行、浏览器登录、设备修改或注入。
3. 回答预先约定的问题，例如模块调用链、声明的外部依赖或版本差异；每个结论能回到文件/方法/证据位置。
4. 明确输出 observed、inferred 与 unknown，不把缺失内容写成“不存在”。
5. 记录耗时、峰值内存、证据体积及模型接收量；与人工/现有工具完成同一个问题比较。
6. 核对返回内容无不允许出机的数据，再决定是否接 MCP；结束后核查残留进程与文件。

待决策项只有一个：是否有值得分析、且允许使用所选模型处理的真实样本。无样本则维持观察，不为追热点安装。

## 核验记录

本轮完成 GitHub release/API、npm metadata、release tag 源码、固定 main 的重点路径静态审查，以及 Hermes 官方 MCP 文档核对。没有安装依赖、启动 server/provider、修改生产配置、申请云资源、执行未知样本或提交/推送知识库 Git。

原始证据和安全分稿：`/home/wpsl5168/work/rea-research-20261013/`。本稿 status=draft 表示采用前仍缺真实样本验收，不把文档完成等同于部署完成。

## Sources

[1] https://api.github.com/repos/morluto/rea
[2] https://github.com/morluto/rea/releases/tag/rea-agents-6.4.0
[3] https://registry.npmjs.org/rea-agents/latest
[4] https://github.com/morluto/rea/blob/rea-agents-6.4.0/README.md
[5] https://github.com/morluto/rea/blob/rea-agents-6.4.0/docs/installation.md
[6] https://github.com/morluto/rea/blob/rea-agents-6.4.0/docs/managed-code-analysis.md
[7] https://github.com/morluto/rea/blob/rea-agents-6.4.0/docs/mcp-contracts.md
[8] https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp
[9] https://github.com/morluto/rea/blob/rea-agents-6.4.0/src/application/SupportedClients.ts
[10] https://github.com/morluto/rea/blob/d96550f2f61647ffa150597de73363fa90c4e169/src/contracts/sessionToolContracts.ts
[11] https://github.com/morluto/rea/blob/d96550f2f61647ffa150597de73363fa90c4e169/src/process/capture/ProcessCaptureEnvironment.ts
[12] https://github.com/morluto/rea/blob/d96550f2f61647ffa150597de73363fa90c4e169/src/domain/evidence.ts
[13] https://github.com/morluto/rea/blob/d96550f2f61647ffa150597de73363fa90c4e169/src/contracts/fridaToolContracts.ts
[14] https://github.com/morluto/rea/actions/runs/38281580054
[15] https://github.com/morluto/rea/blob/d96550f2f61647ffa150597de73363fa90c4e169/docs/cli.md
