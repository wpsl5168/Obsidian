---
title: Azure SQL MCP代码审查与M365 Copilot接入评估
created: 2026-09-18
updated: 2026-09-18
type: research
tags: [mcp, security, architecture, evaluation]
status: draft
---

# Azure SQL MCP代码审查与M365 Copilot接入评估

## 结论

**当前版本不建议原样部署。架构可以保留，但尚未达到“在 M365 Copilot 的 Connector 中安全调用”的验收条件。**

- 用户目标可以使用 **custom federated connector** 实现，无须为了读取 Azure SQL 改走 Copilot Studio。微软当前官方文档支持 MCP 实时读取，列出的工具示例包括 search、fetch、query。[1][2]
- 此入口的官方支持边界是只读数据源，不能推断MCP协议或当前所有租户运行时都技术性禁止写入。[1][7] 第三方Troy Taylor于2026-06-19称把写工具错标readOnlyHint=true后，在custom federated connector中完成CRUD；这是其测试观察，本轮未在用户租户复现。[6] 不将伪造只读标记作为受支持交付方案，也不在核实用户写入需求前擅自删除原写工具。
- 原依赖在本次隔离安装中解析到 MCP SDK 2.2.0，`import server` 直接失败；限定到 SDK 1.x 后，本地 MCP 握手与工具枚举可通过，说明项目有可保留的基础。
- 已复现 SQL 白名单绕过、返回行数限制失效、只读模式仍枚举写工具，以及冒烟测试误报通过。未创建或修改 Azure 资源，未连接真实数据库，未登录客户租户。

## 本轮验证范围

源材料：用户上传的 `azsql-mcp-demo.zip` 和独立部署文档。压缩包有35个文件；复核原文件与解压副本逐文件字节一致，未改项目源码。

证据工作区：`/home/wpsl5168/reviews/azsql-mcp-20260918/`。

- 原依赖环境：本地 Python 3.11.15，按原 requirements 安装，解析清单见 `dependencies-original-resolved.txt`。Dockerfile 使用 Python 3.12；本轮没有完整构建镜像，因此不把本地测试写成容器验收。
- 对照环境：不修改原 requirements，另建 `mcp<2` 环境，解析到 `mcp==1.30.0`。仅为本地测试解包 unixODBC 动态库到工作区，未全局安装驱动或连接 SQL。
- 原自测：pytest 的两个测试函数通过，其内部共14个 SQL 样例全部通过。
- 补充验证：真实执行 SQL 校验器、SQLGlot T-SQL 语法解析、本地 ASGI Streamable HTTP initialize/tools/list，以及带显式 DB stub 的工具调用。业务控制流测试与 SQLite 等价 JOIN 验证均为隔离测试，不冒充 Azure SQL 实测。
- 官方资料：直接读取 Microsoft Learn 完整 HTML，保留正文和最终 URL；最初抽取的 custom connector 页面不完整，已用原站完整页面补齐。

## 正确接入链路

推荐链路：

M365 Copilot Chat → custom federated connector → Entra SSO → APIM → Container Apps 只读 MCP → Azure SQL 专用身份。

- 管理端：Microsoft 365 admin center → Copilot → Connectors → Gallery → Created by your org → Create a new connector → Connect to MCP server。[1]
- 认证：custom connector 设置页列出 Microsoft Entra SSO、OAuth 2.0、No Auth；没有项目当前使用的 APIM Subscription Key 接法。内部数据库建议使用 Entra SSO，而非匿名公开。[1]
- SSO：需要 API 的 Entra app registration、Teams Developer Portal 的 SSO 注册、实际生成的 Application ID URI/audience、scope 与客户端授权配置。[1][5]
- 使用：先针对测试用户或组 staged rollout，再由用户在 Copilot 的 Sources 中连接；工具由 Copilot 按问题动态选择，不等于必须手工输入工具名。[1][2]
- 授权：源系统必须执行用户权限。M365 的登录和连接器可见范围，不会自动给共享数据库服务身份增加行级权限过滤。[2]
- 发布规范要求工具带可读标题和 `readOnlyHint`。本项目应给真正只读的工具补齐；不能给写工具伪装成只读标记。[3]

### 候选方案与取舍

- **保留 APIM + Container Apps：**对原项目改动最小，可集中做 JWT 验证、速率限制和后端密钥注入。已有 APIM 时优先；新建实例前先核 SKU、区域和实际费用。
- **Container Apps 直接提供受 Entra 保护的 MCP：**少一层网关，适合小范围演示；需要把令牌验证、访问范围、限流和审计完整落实到运行时，不能只是去掉订阅密钥。
- **Synced connector：**把内容索引进 Microsoft 365，更适合稳定知识内容，不适合本次强调的实时 SQL 聚合；不是首选。微软区分 synced 的索引模式与 federated 的实时读取模式。[2]

不默认转 Copilot Studio，不假定现有 Azure 登录账号就是目标客户租户。

## 应用层阻塞与修复清单

### A01｜阻塞：原依赖无法启动

位置：`src/requirements.txt:1`，`src/server.py:26`，`src/Dockerfile:25–26`。

原配置 `mcp[cli]>=1.9.0` 没有主版本上界。本次安装得到2.2.0，FastMCP旧导入路径失败，进程退出码1。实际错误：

```text
ModuleNotFoundError: No module named 'mcp.server.fastmcp'. This is mcp 2.x, where FastMCP was renamed to MCPServer ...
```

修复：近期先锁定并测试 SDK 1.x 的确切版本及依赖锁文件；或单独做2.x迁移，不能只改 import 就宣称迁移完成。对照环境1.30.0已完成本地握手，但并非完整生产兼容认证。

### A02｜阻塞：Connector认证与读写范围不匹配

位置：`docs/02-部署步骤.md:200–209`，`infra/apim-mcp-policy.xml:35–49`，`src/server.py:96–320`，`src/config.py:39–45`，`sql/03-security.sql:39–43`。

部署说明只给 VS Code / Copilot Studio API Key 接法。APIM中的 JWT 验证仍在注释内，项目没有完成用户级 Entra 认证链路。仅添加一层网关不等于支持 M365 SSO。

本地 `tools/list` 返回六个工具，`title` 与 `annotations` 均为空；将应用 `allow_writes` 和 `allow_delete` 关闭后，六个工具仍被列出。关闭调用与移除工具发现是两件事；数据库绑定脚本仍把同一身份加入 reader 和 writer。

修复：提供专用只读入口，只注册 search/fetch/受控统计工具，写工具不出现在 tools/list；数据库去掉业务写权限，单独处理必要审计写权限。完成 SSO、租户/受众/调用客户端及 delegated scope 验证，再调整该 API 的 subscription requirement；不能先关闭密钥保护把接口公开。

### A03｜高风险：SQL对象白名单可绕过，审计表也在数据库可读范围

位置：`src/sql_guard.py:28–43,65–74`；`sql/03-security.sql:15`。

真实校验器接受：

```sql
SELECT a.Payload FROM app.Orders o, mcp.AuditLog a
```

其检测对象只有 `app.orders`，漏掉逗号后的 `mcp.AuditLog`。SQLGlot解析能识别两个表。数据库脚本同时授予 `SELECT ON SCHEMA::mcp`，因此审计表不是由数据库权限明确挡住的对象；日志中的写入值、查询参数和删除快照可能被读取。

另一个被校验器放行的语句：

```sql
SELECT * INTO [dbo].[ReviewProbe] FROM app.Orders
```

方括号绕过 `INTO` 正则。**这里只证明应用防护失效；没有执行真实数据库写入，也不能断言现有运行身份具有 CREATE TABLE 权限。**

修复优先级：演示版优先把任意 SQL 改为预定义参数化统计工具；若保留 query，必须使用 T-SQL AST严格检查整棵语法树、对象和表达式，配合独立数据库最小权限。将 `mcp` schema级 SELECT 改为逐视图授予，禁止读取 AuditLog。

### A04｜高风险：所谓强制行数上限并未生效

位置：`src/sql_guard.py:76–81`；`src/db.py:108–110`；`src/server.py:201–221`。

实际放行且没有有效整体限额的输入包括：

- `SELECT TOP (999999) * FROM app.Orders`：现有TOP不下调。
- `WITH c AS (SELECT * FROM app.Orders) SELECT * FROM c`：CTE不加TOP。
- `SELECT OrderId FROM app.Orders UNION ALL SELECT OrderId FROM app.Orders`：只限制第一分支。
- `SELECT TOP (100) PERCENT * FROM app.Orders`：百分比不是返回条数。

DB层使用 `fetchall()`；mock返回25行、请求上限10时，query仍返回25行，仅把 `truncated` 标成true。

修复：查询级整体限额与驱动层 `fetchmany(limit+1)` 双层限制；同步限制执行时间、返回字节数和可用查询形态。TOP本身不能保证聚合/笛卡尔积计算成本受控。

### A05｜高风险：没有真实用户到业务数据的授权链路

位置：`src/server.py:82–92,116–138,152–178,213–215`；`infra/apim-mcp-policy.xml:71–73`。

调用方只用于日志，查询没有按验证后的用户或组做过滤。APIM把订阅名称传为 `X-Caller-Principal`，不是当前M365用户身份。统一托管标识意味着数据库看到服务身份，而不是自动看到每个用户。

演示可明确约定“所有试点用户均可访问同一套虚构数据”；换成客户数据时需要验证令牌并实现授权策略/受控视图或RLS，服务端决定用户范围，不能让模型自由传用户ID或依赖prompt限制。[2]

### A06｜正确性：产品销售视图把已取消订单也计入销售额

位置：`sql/01-schema.sql:158–172`。

`Orders`上的 LEFT JOIN 条件过滤取消订单，但 SUM 的来源仍是前一个 JOIN 的 `oi.LineTotal/Quantity`；取消订单的明细没有被排除。

SQLite等价JOIN验证：一条正常明细金额100、一条取消明细金额200，当前关系表达式汇总300，非取消口径应为100。此为关系语义复现，不是SQL Server执行记录。

修复：先筛非取消订单再聚合明细，或在聚合表达式明确判定匹配的有效订单。随后核产品销售、客户收入、区域收入的业务口径一致性。

### A07｜正确性：明细移单与金额重算不完整，事务被拆开

位置：`src/server.py:259–263,286–307,340–350`；`src/db.py:123–142`；`src/schema_registry.py:74–77`。

`OrderId`允许更新；移动明细后只读更新后的OrderId，只重算新订单，旧订单总额不重算。mock调用轨迹证实只执行新订单5002的重算。

明细写入、订单重算、审计分别独立连接/提交。重算失败时明细已提交，但调用可能报错，随后重试存在重复写入或状态不一致风险。

修复：读取旧OrderId，与新OrderId一起重算；主变更与金额维护纳入同一事务。增加并发更新防护、幂等策略和失败后状态核验。

### A08｜安全与可观测性：审计承诺过强，删除参数不等于人工确认

位置：`src/db.py:146–161`；`src/server.py:236–245,279–288,322–334`。

- `write_audit`吞掉异常，mock注入审计故障后主调用不抛错。业务写入先提交后写日志，不能宣称所有写操作必有审计。
- 关闭写权限、缺必填字段、列校验失败等分支未统一写失败审计。
- `confirm=true`可由调用方第一次直接传入，没有绑定前一次预演、人类确认、版本或一次性令牌；只是API两阶段使用约定，不是强制人工审批。

只读Connector应直接移除删除能力；若另保留管理入口，再独立设计审批和可靠审计，避免在本次只读接入中扩范围。

### A09｜验收缺陷：冒烟脚本能把工具错误打印成全部通过

位置：`tests/smoke-test-client.py:17–25,41–77`。

脚本主要显示工具结果，未统一断言 `isError`、业务success、更新/删除后状态。测试中让search/fetch/query/Update/delete都返回 `isError=true`，只让New返回id，脚本最终仍打印“6个工具全部验证通过”。这是针对测试器的显式fake-session复现，不是云端测得六次失败。

修复：按协议错误、工具错误、业务错误、数据状态四层断言；只读和写入分别测试；使用唯一测试键，并在finally中核对清理状态。HTTP 200不是MCP工具成功。

### A10｜数据返回与运维边界

位置：`src/server.py:53,132,166,378–398`；`src/db.py:164–170`。

- 返回链接默认 `azsql://contoso-retail/...`，没有可在浏览器打开的业务记录页；应映射真实受权HTTPS页面，知识文章可按已验证的SourceUrl返回。未证明当前链接会被M365硬性拒绝，但无法作为可点击来源交付。
- `/healthz`公开返回数据库名、服务主体；失败时返回原始异常。建议外部存活探针仅给最小状态，详细数据库诊断放受保护入口。
- 本地测试确认配置网关共享密钥时，无密钥的`/mcp`返回403，正确密钥返回200。这一层有效，但不是最终用户认证，也不是数据库ACL。

## 部署层审查

完整部署分支报告及逐项证据在工作区 `infra-review.md`；主审已回读编译输出与关键源码，并独立重跑本地编译/合成验证。以下与Connector能否写入无关，应先修：

- **I01｜APIM模板编译失败。** `infra/apim.bicep:57,61,72,87`把三元表达式变量`apimRef`作为parent；本地Bicep 0.34.44报BCP120/BCP240。Container Apps模板编译通过。修法是拆创建/配置模块或使用合法的直接资源符号；部署前先编译全部模板，避免资源收费后才发现尾段失败。
- **I02｜PowerShell冒烟运行错误。** `scripts/07-Test-McpEndpoint.ps1:64`的`Json = (try {...} catch {...})`在PowerShell 7.4.6合成运行中报`try`命令不存在；10个文件AST都通过也不能发现此运行错误。先单独赋值再组装对象。
- **I03｜重跑会破坏部署状态。** `scripts/01-Deploy-AzSql.ps1:42–64,123–135`每次生成新管理员密码，已存在服务器却不更新密码，再覆盖整份connection.json，可能丢掉原凭据、镜像及网关字段。修成读取/合并/原子保存状态，口令轮换单独执行。
- **I04｜默认重新发布会清空演示业务表。** `sql/02-seed-data.sql:10–20`无条件DELETE五张表并重置identity；`03`默认运行它。只能用于专用演示库，初始化应显式opt-in并检查空库/演示标记，不能在客户已有库上直接跑Deploy-All。
- **I05｜SQL登录开关没有贯通。** `03 -UseSqlLogin`创建SQL用户，但`05`和Container Apps模板不注入SQL_USER/SQL_PASSWORD，运行时仍走MI；全新SQL登录分支不能据文档保证连库。应明确authMode并分别测试，或只保留目标MI路线。
- **I06｜清理预演没有保护全部删除。** `99-Remove-All.ps1:26–41`只有资源组删除受ShouldProcess控制，APIM purge HTTP DELETE在外面。未执行清理脚本；静态控制流显示WhatIf不能保证无外部删除，应先补保护和HTTP mock测试。

另有CLI/Az上下文不一致、健康检查失败仍继续、JSON空白绕过删除专项限流、持续SQL健康探针破坏自动暂停预期、机密明文状态文件、缺.dockerignore等问题，详见工作区部署报告。默认`/mcp/mcp`是API前缀与operation路径组合，不能仅因重复名称判为错误；本轮没有为美化路径修改源码。

**读写范围仍待确认，不把“删除写功能”作为既定决定。** 原源码和六个工具均保留。只读专用入口是遵循现行官方Connector边界的一种方案；若目标包含写入，须分开验证产品入口、真实工具声明、用户授权和数据库写入验收，不把改hint即能调用等同正式支持。

## 建议验收顺序

1. 修运行依赖并生成锁文件；在实际Python3.12容器中构建和启动。
2. 提供只读工具入口、修SQL防护及销售口径；本地回归与SQL集成测试通过。
3. 确定目标租户、订阅、区域、预算，确认是否复用APIM；不沿用本机默认订阅创建客户资源。
4. 完成Entra SSO、数据库身份与授权；未授权用户/错误audience/scope访问必须失败。
5. 验证APIM入口、流式协议、初始化/工具列表/每个工具及数据权限；未完成这些不进入租户发布。
6. 在M365管理中心创建custom federated connector，先投放测试组。[1]
7. 用真实M365 Copilot问题验收，例如“查询APAC区域非取消订单收入”“查找Gateway相关文章并给出处”；核结果与SQL一致、来源可打开、权限不越界。**客户端握手成功不等于M365已联调成功。**

## 本轮未做

未修改业务源码；未执行任何部署或清理脚本；未创建Azure资源、Entra应用、数据库账号或Connector；未验证真实Azure SQL权限、APIM策略上传、完整镜像构建、目标租户许可证与实际Copilot回答。README的费用和部署时长未核为本次报价。

## 证据索引

- `source-integrity.json`：压缩包哈希、35个原文件一致性。
- `baseline-results.json`：原依赖导入失败与原SQL自测结果。
- `dependencies-original-resolved.txt`：未限主版本的实际解析清单。
- `review_probes.py`、`probe-results.json`、`probe-run.log`：校验器绕过、行数、工具列表、控制流与JOIN语义验证。
- `protocol_extra.py`、`protocol-extra-results.json`：本地协议、网关拦截与冒烟误报验证。
- `m365-doc.html/.txt`、`source-2`至`source-5`的HTML/TXT、`official-fetch-results.json`、`sources.json`：官方文档与引文账本。

相关背景：[[40-调研报告/企业IT与数字化/2026-09-17-官方MCP服务与M365-Connector选型]]。

## Sources

[1] https://learn.microsoft.com/en-us/microsoft-365/copilot/connectors/set-up-custom-federated-connectors
    > "A custom federated connector starts with a Model Context Protocol (MCP) server that exposes read-only tools to safely surface your data."
[2] https://learn.microsoft.com/en-us/microsoft-365/copilot/connectors/federated-connectors-overview
    > "All permissions are enforced by the source system."
[3] https://learn.microsoft.com/en-us/microsoft-365/copilot/connectors/submit-federated-connector
    > "A reachable public HTTPS endpoint for your MCP server, with every tool carrying a human-readable title and the readOnlyHint annotation."
[5] https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/plugin-authentication-entra-sso
    > "Your MCP server or API must be secured by an Entra app registration."
[6] https://troystaylor.com/power%20platform/mcp/2026-06-19-federated-connectors-crud-experiment.html
    > "All four CRUD operations succeeded in Copilot Chat on 2026-06-19:"
[7] https://modelcontextprotocol.io/specification/latest/schema
    > "NOTE: all properties in ToolAnnotations are hints ."
