---
title: Copilot Studio 网页读取与自定义解析选型
created: 2026-09-22
updated: 2026-09-22
type: comparison
tags: [architecture, tooling, mcp, security]
status: draft
---

# Copilot Studio 网页读取与自定义解析选型

## 结论

如果目标是读取某个网页并按自己的规则解析，不一定需要自建 MCP 或 Azure Functions：轻量固定格式页面可以用 Custom Connector 的托管 C# code；需要第三方 HTML 解析库、浏览器渲染或复杂鉴权时，再把代码放到外部服务。HTTP、Flow、Custom Connector、MCP 是接入/编排选择，代码运行位置才决定依赖、联网和浏览器能力。[2][4][5]

本报告讨论 **Copilot Studio 的 agent tools/topics**，不是普通 Microsoft 365 Copilot custom federated connector。因此，不应把前一个飞书 MCP 项目中的普通 Connector 只读定位直接套到 Studio 工具上。这里是官方文档核验与工程建议，未在用户 Studio 环境做租户实测。

## 可选路径

### HTTP Request 节点 + Power Fx：最轻量

链路：Topic → Send HTTP request → 取响应变量 → Power Fx 字符串/数据处理 → 返回字段。

Studio Topic 中可直接配置 GET/POST/PATCH/PUT/DELETE、Headers、Body 和响应类型；可按样例 JSON 生成结构，HTTP 默认请求超时为 30 秒，可配置错误处理与超时。[1]

适用：网站已有 JSON API，或静态响应中只有少量稳定字段。请求 HTML 时要按实际 Content-Type/响应类型处理，不把 HTML 当 JSON。HTTP 请求本身不执行网页 JavaScript；字符串处理也不是完整 DOM 解析器。复杂、嵌套或不规范 HTML 不建议堆正则规则。

这条路径可以在 Studio 内完成，但“能写 Power Fx”不等于 Topic 里有一个任意 Python/Node.js 脚本节点。

### Custom Connector 的 Code：平台内托管 C#

链路：Studio Tool → Power Platform Custom Connector → C# `Script.ExecuteAsync()` → 获取响应、加工 → 返回 JSON。

这是经常被忽略的选项：Custom Connector 并非只能转发请求，也允许写 C# 自定义代码。类名为 `Script`，继承 `ScriptBase`，在 `ExecuteAsync()` 内执行；官方建议通过 `Context.SendAsync` 发请求，可读取并转换后端响应。[2]

适用：固定站点的简单标题、meta、文本字段提取，JSON 重组，轻量正则或严格 XML/XHTML 处理。不需要为了这种小处理单独部署 Functions。

边界：[2]

- 当前文档列出 .NET Standard 2.0 和有限 namespace；可使用 Regex、XML、Newtonsoft.Json 等列明能力。
- 不能假设能任意安装 HtmlAgilityPack、AngleSharp 或运行 Chromium；官方支持 namespace 清单不是通用 NuGet 托管环境。
- 每个 Connector 一个脚本文件；脚本不超过 1 MB，执行不超过 2 分钟。旧 Connector 需更新才能应用文档所述新超时。
- 不支持将这类 custom code 与 on-premises data gateway 结合使用。
- 在关联 VNet 的环境中，`Context.SendAsync` 仍走公网端点，不能因此认为脚本能访问 VNet private endpoint。
- 下游调用方可能有更短的等待预算；脚本 2 分钟上限不保证 agent 能同步等待满 2 分钟。

不要用正则承诺任意 HTML 的正确结构解析。若选择 C# 内联路线，应固定站点、字段和测试样本，拒绝异常响应。

### Agent Flow / Power Automate：编排抓取和解析

链路：Studio → Agent Flow → HTTP/Connector → 转换与字段处理 → Respond to the agent。

适用：一次需访问多个地址、有分页/条件/重试、缓存、写入 SharePoint/Dataverse，或希望低代码人员维护处理链。Agent Flow 执行预定义动作序列，也可调用 Custom Connector 或外部 API；它并不自动等同一个能安装任意 Python/npm 依赖的代码宿主。[4]

作为 agent tool 的 Flow 必须满足官方触发/响应要求，同步响应应在 100 秒 action limit 内返回；长任务应设计任务提交、job ID 和结果查询，而非让会话一直等。[3]

### 外部 REST API：最适合正式的可测试解析逻辑

链路：Studio 的 HTTP / REST API tool / Custom Connector → 自己的解析 API → 目标网页。

官方集成指南支持 custom APIs，并列出 Azure Functions 等 pro-code 扩展方式。代码也可以运行在合适的自有服务或容器中；不要求一定使用 Azure Functions。[4]

工程推荐：

- 静态 HTML：Python `httpx + BeautifulSoup/lxml`，Node.js `fetch + Cheerio`，或 C# `HttpClient + AngleSharp`。这些是外部服务技术候选，不是声称 Connector 内联 C# 支持这些依赖。
- 动态页面：在适合安装浏览器的容器/主机运行 Playwright，等待明确的页面完成条件后读取 DOM。
- 对 Studio 暴露窄接口，如 `ExtractArticle(url)`、`ExtractPrice(url)`，不要直接开放任意请求头、Cookie、脚本和代理能力。
- 返回精简、可验证的 JSON：源 URL、标题、正文/字段、抓取时间、解析器版本、错误/截断状态；不要把整页脚本和无关 HTML 交给模型。

这条路径最容易做依赖锁定、单元测试、站点回归样本、缓存、可观测性和限流，但要承担托管、安全和运维。

### 自建 MCP：同一解析服务的另一种接入方式

链路：Studio → MCP tool → 自己的解析代码 → 目标网页。

Studio 可连接外部 MCP，并发现服务暴露的工具/资源；当前文档要求启用 generative orchestration。[5]

适用：同一套网页提取能力还需要给其他 Agent 使用，或要统一多个业务工具。MCP 并不自带网页读取、浏览器或解析能力，仍需服务端实现。只有一个固定网页接口时，REST API / Custom Connector 往往更直接；多客户端工具复用时 MCP 更有价值。这是接入协议取舍，不是必须多建一套解析业务逻辑。

### 浏览器自动化：动态页面、登录和交互流程

可选两种微软原生路线：

- Power Automate Desktop：浏览器动作可按元素提取单值、列表、行、表格，也有 `Run JavaScript function on web page`，可在已打开页面上运行 JavaScript 并返回结果。接入 Studio 时通过适当的 Flow 和桌面运行环境编排。[6]
- Copilot Studio computer use：用自然语言驱动 Windows 浏览器/桌面 UI，适合无 API 的页面操作；这不是直接托管开发者代码。当前官方页面已列出 GA 模型选项，不能统一沿用早期“全部仅预览”的结论；可用性仍需核目标环境。[7]

适用：必须登录后打开、点击、翻页，或内容需要前端渲染。代价是运行机器、凭据管理、并发、页面变化及交互延迟。对固定结构的高频抽取任务，我更偏向可回归测试的 DOM/选择器方案；只在确需界面操作时使用视觉 computer use。

登录/MFA/验证码应按站点允许的授权流程处理，不把浏览器自动化理解成绕过访问控制。

## Code Interpreter 是否可用

Studio 已支持在 prompt 中生成并执行 Python，用于数据/文档加工。官方 FAQ 明确列出外部网络访问限制，因此不能因为“支持 Python”就承诺能直接 `requests.get(任意URL)` 或把它当成稳定公网爬虫宿主。[8][9]

可把已经通过受控 HTTP/API 获取的数据交给 code interpreter 做后续处理，但固定解析规则、可重复的单元测试与依赖控制，仍优先放进自有代码服务。

资料冲突提示：较早 FAQ 还写“不支持从 topic 直接调用 prompt tool”，较新的专门配置页已经给出在 topic 添加 prompt 的步骤。这里只采用 FAQ 的外部网络限制，不把其旧 topic 限制当成当前统一能力边界。[8][9]

## 选型建议：优先比较这三个候选

- **固定公开静态页、少数字段**：Custom Connector 内联 C#。优点是少一套服务；缺点是依赖和联网边界受限。
- **通用正文/表格解析、多个站点、需要第三方库**：外部 REST 解析服务，Studio 用 Custom Connector 调用。优点是测试和运维可控；代价是托管。
- **必须执行 JS、登录、翻页**：浏览器运行环境。自有代码优先考虑 Playwright 容器；微软原生自动化优先考虑 Power Automate Desktop，复杂视觉流程再考虑 computer use。

仅拿网站内容回答问题时，网站知识源可能足够；但它不是“按指定 URL 执行自定义解析并返回固定 schema”的等价实现。

## Azure Functions 部署位置与跨租户调用

Azure Functions 通常作为独立 Function App，创建在指定 Azure subscription 的 resource group 内，并选择运行 region 和 hosting plan；它不是部署到 Copilot Studio 的 Topic 或 Power Platform environment 中。Azure 资源的部署、运维和费用归属应与 Studio 分开管理。[10]

不要求 Function App 与 Studio 位于同一租户、同一订阅或同一区域。典型情况是客户租户 A 的 Studio，通过获准的 HTTPS API 调用服务方租户 B 中的 Function App。资源管理归属、运行时认证和网络可达性是三条独立边界；“不同租户”本身不是 HTTP 无法调用的原因。[4][11][12]

跨租户时需要核定：

- 认证：Function Key 是共享密钥，不要求调用者具有同租户 Entra 身份；通过受保护的连接/凭据配置传入 `x-functions-key`，不用 master key，不在提示词或公开 URL 中散发。生产可选择 App Service Authentication 或 APIM 等更完整的认证控制。若启用 Entra OAuth，要明确 API 接受的 issuer/tenant、audience、scope/role，以及是否需要多租户应用、目标租户 consent；不能以资源在哪个租户直接推导 token 会被接受。[11]
- 网络：公网 HTTPS 仍可能受 Function access restrictions、防火墙/IP规则影响；私有端点需要另外设计受支持的网络路径和 DNS，同租户也不会自动连通。
- 治理：Power Platform tenant isolation 文档针对 Entra 鉴权连接器；启用后有效跨租户凭据也可能被拦，需要管理员按方向配置允许规则。不要把它理解成所有 HTTP/API Key 请求的统一跨租户防火墙。Custom Connector 还受独立 DLP 分类和 Host URL 策略约束，不应通过换认证方式绕过组织政策。[12][13]
- 数据：不同 Azure region/cloud 还涉及延迟、数据驻留和跨境要求；商业云结论不能无条件推广到 21V/GCC 等独立云组合。

工程建议：短期 Demo、只处理获准公开网页，可评估服务方 Azure 订阅中的受控 API；客户专用生产交付优先客户自己的 Azure 订阅/租户，便于权限、账单、审计和交接。多客户共享服务可以放服务方租户，但需按真正多租户 SaaS 设计客户隔离、授权和配额，而不是共用一个无边界密钥。

## 最低验收与风险

- 先确认网页是否已有可授权调用的数据 API；有 API 时通常比抓 HTML 稳定。
- 样本至少覆盖正常页、空页、403、重定向、超时、编码差异、分页与前端尚未渲染完的情况。
- 区分“请求成功”“解析成功”“字段完整”，空字符串不能冒充成功。
- URL 参数做域名/IP/协议与每次重定向校验，防止 SSRF 访问本机、私网和云 metadata；内部业务站点采用独立受控网络方案，不通用放行私网。
- 认证信息走连接、Key Vault 或受控凭据存储，不放提示词、对话和网页返回内容；遵守 DLP、目标站授权和限流。
- 返回的网页内容是数据，不得让网页提示词改变 Agent 权限或触发未授权写操作。
- 云流程、连接器、计算资源和浏览器机器的许可/额度独立核验，不能默认所有路线都免费。

## 关联

- [[40-调研报告/企业IT与数字化/README]]
- [[40-调研报告/企业IT与数字化/2026-09-22-飞书文档MCP与Copilot-Connector实现选型]]
- 本轮原文与引用账本：`~/reviews/copilot-studio-web-parsing-20260922/`。

## Sources

[1] https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-http-node
[2] https://learn.microsoft.com/en-us/connectors/custom-connectors/write-code
[3] https://learn.microsoft.com/en-us/microsoft-copilot-studio/flow-agent
[4] https://learn.microsoft.com/en-us/microsoft-copilot-studio/guidance/integrations
[5] https://learn.microsoft.com/en-us/microsoft-copilot-studio/agent-extend-action-mcp
[6] https://learn.microsoft.com/en-us/power-automate/desktop-flows/actions-reference/webautomation
[7] https://learn.microsoft.com/en-us/microsoft-copilot-studio/computer-use
[8] https://learn.microsoft.com/en-us/microsoft-copilot-studio/faq-code-interpreter
[9] https://learn.microsoft.com/en-us/microsoft-copilot-studio/code-interpreter-for-prompts
[10] https://learn.microsoft.com/en-us/azure/azure-functions/functions-create-function-app-portal
[11] https://learn.microsoft.com/en-us/azure/azure-functions/function-keys-how-to
[12] https://learn.microsoft.com/en-us/power-platform/admin/cross-tenant-restrictions
[13] https://learn.microsoft.com/en-us/power-platform/admin/dlp-custom-connector-parity
