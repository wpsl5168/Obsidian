---
title: 飞书文档 MCP 与 Copilot Connector 实现选型
created: 2026-09-22
updated: 2026-09-22
type: research
tags: [mcp, architecture, security, tooling]
status: draft
---

# 飞书文档 MCP 与 Copilot Connector 实现选型

## 结论

建议独立开发面向 Microsoft 365 Copilot custom federated connector 的飞书文档 MCP，直接封装飞书正式 REST OpenAPI，不依赖飞书官方远程 MCP，不通过 CLI 子进程转发。复用 azsql-mcp-demo 的 MCP 接入、Entra 验证和安全设计，替换 SQL 数据访问层；不是照搬数据库业务代码。

本轮仅完成仓库读取、官方文档核验和设计建议；未创建服务、未发布 Connector、未访问或修改用户飞书文档。

## “官方废弃 MCP”核验

官方个人接入页明确声明：“MCP Token（个人托管链路）后续将逐步下线”，推荐飞书 CLI。该声明针对个人托管链路，页面未给出完整下线时间表，不能扩大成所有开发者 MCP 接口已经关闭，更不等于 REST OpenAPI 下线。[12]

开发者接入页仍列出远程 MCP 服务、UAT/TAT 鉴权与云文档工具；同时提醒工具入参和出参可能调整，不应依赖其结构定义。对于需要可测试、可版本化工具合同的自建 Connector，这是额外依赖风险。[11]

因此，选直接 REST 不是因为断言所有官方 MCP 均已不可用，而是要消除托管服务生命周期、上游工具 schema 漂移和双层 MCP 转发依赖。

## 参考项目已核事实

参考版本：`wpsl5168/azsql-mcp-demo`，SHA `bc5a5a3d983d898a2232725fda499a7b2b75b7fc`。本地工作树干净，HEAD 与本轮 GitHub 返回的最新提交一致。[1]

README、EXPERIMENT、DEPLOYMENT-STATUS 记录：2026-09-18，普通 Copilot Chat 经 custom federated connector 对隔离 SQL 演示数据完成真实 INSERT、UPDATE、DELETE，并有独立数据库读取与关联审计验证；不能再沿用早期“仅发现工具、未完成 CRUD”的中间结论。[1]

但上述实验把写工具声明为 `readOnlyHint=true`。源码默认开关关闭，实验成功不是微软正式支持写入的证明；微软当前 custom federated connector 文档仍要求只读工具。[1][2]

### 复用与替换边界

- 复用设计：Python/FastMCP、HTTPS MCP 接入、Entra JWT 校验、读写 scope 分离、测试用户限制、错误脱敏、correlation ID、独立读回验收。
- 替换业务层：SQL 实体、SQL guard、ODBC、数据库权限脚本，改为飞书 REST client、文档服务与身份凭据管理。
- 独立配置：新服务的应用登记、Connector、域名和密钥，不直接复用 SQL 服务的生产标识或改动现有部署。
- 重新设计审计：飞书远程写入和本地审计无法沿用 SQL 的同事务提交保证；使用操作状态记录、结果核验和待对账状态，不能声称跨系统原子提交。

## 候选方案比较

### A. 自建 MCP → 飞书 REST OpenAPI（推荐）

自己维护稳定的工具名称、参数、分页、内容转换和写入安全边界。可按用户需求只开放文档操作，不把全部开放 API 暴露给模型。代价是需要实现文档块处理、OAuth 刷新、限流和失败恢复。

### B. 自建 MCP → 飞书官方远程 MCP

开发快；官方有搜索、读取、创建、更新等文档工具，开发者端点为 `https://mcp.feishu.cn/mcp`。[4][11]

但增加一层工具转发，受上游 schema 与生命周期影响；开发者路径要求 `X-Lark-MCP-UAT` 或 `X-Lark-MCP-TAT`，并显式提供工具白名单。不能把这些头与普通 Bearer OAuth 接入视为自然等价。[11]

### C. 自建 MCP → 飞书 CLI

适合个人自动化和本地 Agent，不作为本服务推荐架构。多用户请求需要另外隔离 CLI 会话、登录缓存和进程；REST client 更适合在服务层明确管理用户身份及请求生命周期。这是工程判断，不是 CLI 不具备读写能力。

## 推荐链路

```text
普通 Microsoft 365 Copilot Chat
  → Custom federated connector
  → 自建飞书文档 MCP（HTTPS / Streamable HTTP）
      → 入口认证、用户绑定、权限、审计
      → 文档业务工具、分页与内容转换
      → 飞书 REST OpenAPI
          → Docx / Wiki
```

不另建 Agent，不改成 Copilot Studio，也不把文档先同步入 Microsoft Graph 当作写回方案。

## 第一版工具建议

对外提供业务级工具，不机械一 API 对应一工具：

- `search`：按关键词查找有权访问的文档，返回标题、引用 URL、标识和摘要。
- `fetch`：读取文档，包含分页/截断标记；Wiki 链接先解析真实对象类型，不把 Wiki token 当 Docx ID。
- `create_document`：在指定位置创建新文档，返回准确 URL，并读回验证。
- `append_document`：追加正文；处理分块和重复提交。
- `update_document`：限定段落或块更新，展示变更范围；不得默认整篇重建。

默认仅承诺新版 Docx 与指向 Docx 的 Wiki 节点。旧版 Doc、Sheets、Base、附件和复杂嵌入资源分别识别，未实现的类型返回明确限制。第一版不开放删除文档、批量改权限或全库修改。

REST 层需处理 Docx 内容块；官方批量更新 API 已提供结构化内容编辑。其完整参数、版本语义及 Markdown 转换方式应在实现阶段按接口原文建立测试，不将 revision 参数存在直接当作原子乐观锁。[8]

## 身份与权限

推荐团队使用时，每位使用者绑定自己的飞书账号，以其 `user_access_token` 调用 API；如果业务只处理固定共享库，可采用明确隔离的应用身份方案。不能让所有 Copilot 用户静默共用管理员凭据。

需要先确定身份模式，再落地 OAuth：

- 入口可延续参考项目的 Entra 认证；Entra `tid + oid` 与飞书身份需通过显式授权绑定，不能按邮箱猜测或信任模型传入的用户 ID。
- Entra token 不直接作为飞书 token 使用；两个授权域独立。
- 飞书现行用户 token 接口为 `https://accounts.feishu.cn/oauth/v3/token`；开放平台自建应用属于 Confidential Client，需提供 client secret。OAuth scope 按需申请，按实际返回 scope 判断权限。[9]
- refresh token 单次使用；要串行刷新、原子保存轮换结果，失败后不得自动降级成应用管理员身份。[9]
- API scope、用户授权、目标文档 ACL 是不同层次，均需满足。

## 写入安全与验收

- 写操作必须经过服务端写权限与目标范围校验；工具 annotations 不是授权系统。
- 追加/创建记录操作标识与请求摘要；超时后先查询是否已生效，不能盲目重试造成重复内容。
- 内容替换前核验目标和旧内容；没有上游原子条件写保证时，承认并发窗口，不声称检查一次 hash 就消除了竞争。
- 多步写入返回实际完成部分及可恢复状态，不能把部分成功包装成整体成功。
- 日志保存调用者标识、目标、操作、时间、结果和 correlation ID；默认不记录全文及 token。
- 仅在明确授权、隔离测试文档与限定用户的实验中考虑参考项目的 annotation 对照。默认写工具仍为 `readOnlyHint=false`。
- 验收分层：直接 MCP 调用成功；真实 Copilot 调用成功；独立飞书读回一致。三项不能混为一项。
- 最终演示使用临时文档，验证读取、创建、追加、局部修改、无权限访问失败及重复请求处理。测试文档清理另按授权执行。

## 待确定事项

唯一先决产品问题：按每个使用者自己的飞书身份读写，还是用统一应用身份限定在共享知识库内？该选择决定授权链路和数据隔离设计，不应由默认缓存决定。

## 关联与证据

- [[40-调研报告/企业IT与数字化/README]]
- [[40-调研报告/企业IT与数字化/2026-09-17-官方MCP服务与M365-Connector选型]]
- 本轮证据目录：`~/reviews/feishu-copilot-mcp-20260922/`。
- 原文获取注意：飞书 HTML/提取器可能缺表格与警告；本轮以对应 `.md` 原文补核个人链路下线说明及开发者接口合同。

## Sources

[1] https://github.com/wpsl5168/azsql-mcp-demo/tree/bc5a5a3d983d898a2232725fda499a7b2b75b7fc
[2] https://learn.microsoft.com/en-us/microsoft-365/copilot/connectors/set-up-custom-federated-connectors
[4] https://open.feishu.cn/document/mcp_open_tools/supported-tools
[8] https://open.feishu.cn/document/ukTMukTMukTM/uUDN04SN0QjL1QDN/document-docx/docx-v1/document-block/batch_update
[9] https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/authentication-management/access-token/get-user-access-token-v3
[11] https://open.feishu.cn/document/mcp_open_tools/developers-call-remote-mcp-server
[12] https://open.feishu.cn/document/mcp_open_tools/end-user-call-remote-mcp-server
