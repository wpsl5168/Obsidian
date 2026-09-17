---
title: 飞书之外的官方MCP服务与M365-Connector选型
created: 2026-09-17
updated: 2026-09-17
type: comparison
tags: [mcp, tooling, comparison, security]
status: draft
---

# 飞书之外的官方 MCP 服务与 M365 Connector 选型

> 核验日期：2026-09-17。范围是厂商正式文档中的业务 MCP，不把社区同名项目、仅检索开发文档的 MCP、普通 OpenAPI 混为一谈。本次仅公开资料核验，未登录或授权任何业务账号，也没有实测 M365 租户端到端接入。

## 结论

有多家厂商直接运营远程 MCP，无须自己部署业务 Server。GitHub、Notion、Linear 都有官方托管入口。[1][5][10]

Slack、Atlassian、Asana 也有官方远程接入地址。[3][4][6]

HubSpot 另外提供面向 CRM 的远程 MCP。[9]

**对老王当前的 M365 后台 Connector 场景，第一筛选条件不是“有没有写工具”，而是“能否安全地只暴露读取工具”。** Microsoft 的 custom federated connector 要求只读 MCP；支持通用 MCP、支持 OAuth，不等于已通过该 Connector 的联调。[12]

我的优先级建议：先用已有业务数据验证，别为了 MCP 迁移办公平台。技术试点优先 GitHub 或 Linear 的专门只读入口；文档协作再看 Notion / Atlassian。下文“推荐验证”是选型判断，不是已接通的结论。[5][13]

## 官方托管服务清单

### GitHub：代码、Issue、PR、工作流

- 形态：GitHub 官方托管，另提供官方本地 Server；不是同名社区封装。[10]
- 主入口：`https://api.githubcopilot.com/mcp/`；只读入口：`https://api.githubcopilot.com/mcp/readonly`。可以进一步按工具集细分。[2][13]
- 鉴权：支持 OAuth；也支持 PAT，但自定义 MCP host 的 OAuth 仍需配置对应 GitHub App / OAuth App，不能把 VS Code 的一键登录体验直接等同于 M365 Connector。[2][10]
- 能力：仓库、代码、Issue、PR、Actions 等读取和操作；只读模式跳过写工具。[10][13]
- 判断：适合已有 GitHub 数据的低成本试点。通过只读 URL 降低工具暴露风险后，仍需核 M365 的 OAuth client、回调和工具标注兼容。

### Linear：项目、任务与研发协作

- 形态：官方集中托管的远程 MCP。[5]
- 主入口：`https://mcp.linear.app/mcp`；专用只读入口：`https://mcp.linear.app/mcp/readonly`。[5]
- 鉴权：交互授权为 OAuth 2.1 + DCR，也支持 OAuth bearer / API key。只申请 `read` scope 时，底层令牌也不能调用写 API。[5]
- 能力：查找、创建和更新 Issue、项目、评论；只读入口只暴露读取工具。[5]
- 判断：只读机制最清楚的候选之一；M365 的静态客户端注册与 DCR 是不同路径，需核选定入口支持哪种，不把 DCR 自动视为静态 client secret 可直接填。

### Notion：知识库、文档与数据库

- 形态：官方托管、持续维护的 MCP；旧开源本地 Server 已不再积极维护，不建议新项目默认选旧包。[1]
- 入口：`https://mcp.notion.com/mcp`，推荐 Streamable HTTP；另有 SSE fallback。[1]
- 鉴权：OAuth 交互授权；官方当前不支持完全无交互授权。[1]
- 能力：搜索、读取、创建和更新页面、数据库相关内容，并支持文件上传相关工具。部分搜索与高级查询能力受套餐、Notion AI 和工作区权限限制；工具列出不等于账号可调用。[1][11]
- 判断：适合已有 Notion 内容。默认含写工具；接 M365 只读 Connector 前，要核工作区工具选择与只读暴露方式，不只在 prompt 里写“不要修改”。[11][12]

### Atlassian：Jira、Confluence 等

- 形态：官方远程 Atlassian MCP。[4]
- 当前官方接入页给出的入口：`https://mcp.atlassian.com/v2/mcp`。网关如果需要分页平铺全部工具，文档另列 `?tools=all`；不要照旧文章直接固定 v1 或 SSE 地址。[4]
- 鉴权：OAuth 2.1，另有可选 API token 方式。[4]
- 能力：搜索和读取业务上下文，也可创建、更新工作项和页面；操作继承用户现有权限，受产品与管理策略限制。[4]
- 判断：企业已有 Jira / Confluence 时价值高。需要核 OAuth client/domain 限制，并确保向 Connector 暴露的只有读取工具。

### Slack：聊天、文件、Canvas 与 Lists

- 形态：Slack 官方远程 MCP，入口 `https://mcp.slack.com/mcp`。[3]
- 鉴权：confidential OAuth，需要 Slack app 的 client ID / secret；官方提供两个 `.well-known` 元数据地址。文档明确目前不支持 SSE 和 DCR，支持 PKCE。[3]
- 能力：搜索消息、读频道/线程、发送消息、管理 Canvas / Lists、上传文件等；各工具对应不同的用户 scopes。[3]
- 判断：静态 OAuth 配置方向较接近 M365 表单，但这不是兼容认证。应只授权所需读取 scopes，并核服务是否随之仅暴露只读工具；不能因无写权限就默认工具清单满足 Connector 要求。[3][12]

### Asana：任务、项目与工作图谱

- 形态：官方远程 MCP V2，入口 `https://mcp.asana.com/v2/mcp`。[6]
- 鉴权：OAuth + Streamable HTTP；自定义 client 可能需要按 Asana 集成指南生成 client ID / secret。[6]
- 能力：读取和分析项目任务，也能创建、管理任务和项目。企业管理员可通过应用管理限制 client。[6]
- 判断：适合已有 Asana 的团队；默认含写能力，需额外处理只读 Connector 边界。优先 V2，不复制已经标为 deprecated 的旧 SSE 配置。[6][12]

### HubSpot：CRM 与营销内容

- 形态：官方远程 CRM MCP，与本地 developer MCP 是不同产品。入口 `https://mcp.hubspot.com/`。[9]
- 鉴权：在 HubSpot 创建 MCP connector，获取其 OAuth client ID / secret；PKCE 为必需。[9]
- 能力：读取 CRM 记录、活动、组织资料、部分营销数据，也能创建/更新 CRM 记录、活动和内容。实际工具受套餐、权限和账号配置限制。[9]
- 边界：启用 Sensitive Data 的账号会限制 MCP 对活动及 conversation 数据的访问；不是普通 CRM API 权限的简单复制。[9]
- 文档取舍：优先采用具体集成指南。概览 FAQ 仍混有旧的“未来支持 OAuth 2.1”和只读描述，不据此否定现行集成页明确列出的 PKCE、读写功能。[9]
- 判断：不是用来替代个人邮箱；这里的 email activity / marketing draft 不能泛化成任意 Gmail/Outlook 收发。

### Google Workspace：Gmail 与办公套件

- 形态：Google 官方托管远程 MCP；Gmail 精确入口为 `https://gmailmcp.googleapis.com/mcp/v1`，使用 Streamable HTTP 与 OAuth 2.0。[14]
- 状态：仍为 Developer Preview，需 Workspace Developer Preview Program。官方申请表明确不接受 Gmail addresses、Service Accounts 或 Google Groups，因此**个人 `@gmail.com` 当前不能按这一公开预览申请入口接入**；不能推广为 Gmail API 永久不支持个人账号。[14][16]
- 能力：检索/读取邮件、列草稿/标签、创建草稿、修改标签。当前配置页未列直接发送工具，示例是生成草稿后让用户在 Gmail 发送；不要把“当前工具没列发送”理解为 compose scope 是只读。[14]
- 其他官方服务分开部署：Drive、Docs、Sheets、Slides、Calendar、Chat、People 各有自己的 MCP；不是把 Gmail 地址换成一个所谓万能 Workspace endpoint。[15]
- **更值得关注的只读候选**：`https://workspacemcp.googleapis.com/mcp/v1` 的 Universal Search MCP，可按授权范围搜索 Gmail、Drive、Calendar、Chat，只授部分读取 scopes 也可用；同样受预览资格限制。[17]
- 判断：企业 Workspace 用户可以先核预览资格；若只是个人 Gmail，不应据这份官方预览文档承诺“立即可用”。未实测其与 M365 custom federated connector 的 OAuth/工具契约兼容性。

### Microsoft Work IQ：M365 数据与 Outlook Mail

- 形态：微软官方远程 MCP 确实存在，不只有本地 CLI。统一入口：`https://workiq.svc.cloud.microsoft/mcp`，官方接入示例使用 Entra ID 认证和用户权限。[18]
- 能力与权限：访问范围由用户权限、应用 OAuth 权限、租户策略共同决定；统一 Work IQ 的写入默认受租户策略阻断，不能据此当作“服务只有只读工具”。[18][20]
- 另有面向租户的 Work IQ Mail / Outlook Mail 工具服务，官方参考列出查邮件、草稿、回复、发送、删除等操作，且标为 Preview；不能把这些写工具直接塞进只读 Connector。[19][12]
- 许可：Work IQ 新版概览采用 usage-based billing，接入前另核租户启用、管理员同意与计费；不把旧版 CLI 许可要求套到所有远程入口，也不把 M365 租户支持推广到个人 Outlook.com。[18][21]
- 判断：若目标只是让 M365 Copilot 使用同租户 Outlook/SharePoint/Teams，先评估现有原生能力是否已满足，避免绕出再绕回。外部 Agent 要调用 M365 数据时，这类 MCP 更值得评估。

### Box：企业文件与知识检索

- 形态：Box 官方托管，入口 `https://mcp.box.com`。[22]
- 鉴权：管理员创建 Integration Credentials，取得 client ID / secret，并登记 client redirect URI 和 scopes；OAuth 授权与 token endpoint 均有正式文档。[22]
- 能力：文件搜索、内容与元数据读取、Box AI，也有复制、上传、创建、移动、共享等写工具；部分写工具默认开启。管理员可管理工具开关。[23]
- 判断：适合企业文件知识库，但不是默认只读。接入前核可关闭的写工具与最小 scope；文档中的 `docgen.readwrite` 还要求 Enterprise Advanced，不能当作免费基础能力。[22][23]

### Dropbox：文件搜索与读取

- 形态：官方远程 MCP，open beta，入口 `https://mcp.dropbox.com/mcp`。[24]
- 鉴权：Dropbox OAuth；DCR 只对其列出的可信客户端开放，其他客户端需要自行创建 Dropbox app、登记回调并取凭据。[24]
- 能力：搜索、读取文件、列目录，另有创建、移动、删除、共享等写工具。[24]
- 判断：有条件候选。不能因为支持 Claude/ChatGPT 的 DCR 就推断 M365 免注册可用；也不能直接照抄包含写 scopes 的示例用于只读 Connector。

### 钉钉：官方 MCP 市场

- 形态：`https://mcp.dingtalk.com/` 市场可见由“钉钉（中国）信息技术有限公司”发布的 Remote 文档、AI 表格、通讯录、日历、日志等业务 MCP，不只是开发文档搜索。[25]
- 实际入口：登录市场并选择企业、服务后复制专属 URL；阿里云指南里的示例 URL 含 key，**不是可直接共享的公共 endpoint**，也不应放进日志、公开报告或 Git。[26]
- 能力：既有搜索/读取，也有创建文档、表格操作、写日志等。[25]
- 判断：国内协作平台的真实官方候选，但本次未核到可直接给 M365 注册的标准 OAuth client 流程；仍需按用户绑定、只读配置和具体认证方式复核。

### 企业微信：包含邮件能力的官方 MCP

- 官方 AI 开放页明确提供 CLI 和 MCP 方式，覆盖邮件、文档等业务；邮件能力包括检索、读取正文、发送或回复。[27]
- 接入路径：已有 API 模式智能机器人 → 编辑 → 可使用权限 → 授权 → 权限详情，复制 MCP URL 或 MCP JSON Config。不是仅把第三方 CLI 改名叫 MCP。[28]
- 权限：搜索与获取邮件、文档、会议、微盘等内容可能需要企业管理员审批；实际 MCP URL、认证参数与租户许可需在对应企业中核验。[28]
- 判断：**如果老王优先考虑国内邮箱/文档，这是值得进一步看的一项。** 本次没有取得实际租户 endpoint，也没有证明标准 OAuth 或 M365 直连；更不能据此宣称独立腾讯企业邮 exmail 的所有账号都自动可用。

### 腾讯文档：官方团队提供的远程文档 MCP

- 证据：腾讯云 MCP 页标注“By 腾讯文档团队”，给出业务入口 `https://docs.qq.com/openapi/mcp`，并提供文档创建、查询、编辑等工具说明。[29]
- 鉴权：在腾讯文档的 MCP 授权页获取个人 Token，示例通过 `Authorization` header 传原 Token；不擅自改成未经文档说明的 Bearer 前缀。[29]
- 能力：搜索空间文件、读取内容、查节点，以及文档/智能表格/智能文档的增删改；存在 VIP 权限不足错误，实际工具以 `tools/list` 为准。[29]
- 判断：不需要自建业务 MCP，但本次未核实标准 OAuth discovery/DCR。**有 URL + Token 不等于能直接填进 M365 的 OAuth Connector。** 个人 Token 必须视为凭据，不放进 URL 或共享文档。

## M365 Connector 接入前的检查单

1. **确认产品路径。** 本报告对应微软管理后台的 custom federated connector，不是 Copilot Studio 或泛 Agent 插件；此入口要求只读工具。[12]
2. **确认 Server 形态。** 官方托管 MCP 才能省去业务 Server 运维；只有 stdio 的包不能直接填入云端 Connector URL。
3. **确认 OAuth 能否注册你的 client。** 要核 client ID / secret、精确微软回调、PKCE、scope 和 refresh 机制。厂商一键支持 Claude/VS Code 不等于已经支持 M365。[2][3][12]
4. **确认只读既在工具层，也在权限层。** 优先专用只读 endpoint，再配最小读取权限；不要把写工具伪装成 `readOnlyHint=true`，也不要只依赖模型承诺不写。[5][12][13]
5. **确认宿主能力。** 除 `initialize` / `tools/list`，还需实际做一次用户授权、一次只读查询及刷新；本次没有把这些标记为通过。
6. **核数据位置与商业边界。** 套餐、管理员策略、网络可达性、数据授权范围分别检查；不为了“免费 MCP”把企业数据授权给不明第三方。

## 两条落地路线

- **已有 SaaS 数据 → 优先官方托管。** 例如 GitHub 仓库问题总结、Linear 逾期任务汇总，先用只读 endpoint 验证 Connector。服务器零自建不代表 OAuth 零配置。[5][13]
- **要自定义只读视图/厂商 OAuth 不兼容 → 再加自己的适配层。** 只负责安全认证与读工具过滤，业务调用仍用官方 MCP；不要默认把全部写入接口搬进 Connector。[12]

## 验证边界与失败记录

- 本轮完成官网/官方仓库原文检索；不代表逐个服务已经认证、可用或在当前微软租户中完成联调。
- Salesforce 的官方 hosted MCP 页面被搜索定位，但本轮未能取得完整部署正文；浏览器工具也启动失败，暂不列为已核完的推荐候选。
- 没查到某厂商官方原文只能写“未核实”，不能推成“该厂商没有 MCP”。
- 本调查与此前暂停的飞书部署相互独立；没有重试被拒绝的 service 安装，也未变更 DNS、Caddy 或现有业务授权。

相关：[[10-知识库/AI模型与Agent/03-工具调用与上下文协议/3.2.1-MCP-Server生态系统]] · [[40-调研报告/企业IT与数字化/README]]。

本轮证据目录：`~/work/mcp-provider-survey-20260917/`。仅本地入库，不提交/推送 Git；未更改任何业务账号或公网服务。

## Sources

[1] https://developers.notion.com/guides/mcp/get-started-with-mcp — Notion official MCP documentation
[2] https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp-in-your-ide/set-up-the-github-mcp-server — GitHub official MCP documentation
[3] https://docs.slack.dev/ai/slack-mcp-server — Slack official MCP documentation
[4] https://support.atlassian.com/atlassian-ai-gateway/docs/get-started-with-the-atlassian-remote-mcp-server — Atlassian official MCP documentation
[5] https://linear.app/docs/mcp — Linear official MCP documentation
[6] https://developers.asana.com/docs/using-asanas-mcp-server — Asana official MCP documentation
[9] https://developers.hubspot.com/docs/apps/developer-platform/build-apps/integrate-with-hubspot-mcp-server — HubSpot-integration official MCP documentation
[10] https://github.com/github/github-mcp-server — GitHub-repository official MCP documentation
[11] https://developers.notion.com/guides/mcp/mcp-supported-tools — Notion-tools official MCP documentation
[12] https://learn.microsoft.com/en-us/microsoft-365/copilot/connectors/set-up-custom-federated-connectors — M365-connector official MCP documentation
[13] https://raw.githubusercontent.com/github/github-mcp-server/main/docs/remote-server.md — GitHub remote server read-only options
[14] https://developers.google.com/workspace/gmail/api/guides/configure-mcp-server — Gmail official source
[15] https://developers.google.com/workspace/guides/configure-mcp-servers — Google-Workspace official source
[16] https://docs.google.com/forms/d/e/1FAIpQLSd7BiMXXHDlUDkF7G0TSY5zfJbQwFNH3m6K_ZYFi3vCHLFbng/viewform?resourcekey=0-1uHeVg8junj3PPTLNcn7WQ — Google-preview-application official source
[17] https://developers.google.com/workspace/guides/universal-search-mcp — Google-Universal-Search official source
[18] https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/mcp/quickstart/github-copilot-cli — Work-IQ-remote official source
[19] https://learn.microsoft.com/en-us/microsoft-copilot-studio/mcp-mail-work-iq — Work-IQ-Mail official source
[20] https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq/mcp/policy-governance-mcp — Work-IQ-policy official source
[21] https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/work-iq — Work-IQ-billing official source
[22] https://developer.box.com/guides/box-mcp/setup.md — Box-setup official source
[23] https://developer.box.com/guides/box-mcp/tools.md — Box-tools official source
[24] https://help.dropbox.com/integrations/connect-dropbox-mcp-server — Dropbox official source
[25] https://mcp.dingtalk.com — DingTalk-market official source
[26] https://help.aliyun.com/zh/jvs/user-guide/dingtalk-ai-form-and-document-access — DingTalk-setup official source
[27] https://work.weixin.qq.com/nl/index/aicli — WeCom-official official source
[28] https://open.work.weixin.qq.com/help2/pc/21676 — WeCom-MCP-setup official source
[29] https://cloud.tencent.com/developer/mcp/server/11803 — Tencent-Docs official source
