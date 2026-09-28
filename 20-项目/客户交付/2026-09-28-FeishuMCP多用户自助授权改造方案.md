---
title: FeishuMCP多用户自助授权改造完整方案
created: 2026-09-28
updated: 2026-09-28
type: entity
tags: [mcp, architecture, security, tooling]
status: draft
sources: ["https://github.com/wpsl5168/feishu-mcp", "https://learn.microsoft.com/en-us/microsoft-365/copilot/connectors/set-up-custom-federated-connectors"]
---

# FeishuMCP 多用户自助授权改造完整方案

> 方案版本：v1.1 · 基线核验：2026-09-28 · 状态：待审批实施方案。新 OAuth 基础模块已完成部分离线测试，但完整授权路由、新服务部署和真实 Copilot 验收尚未完成。本文不是“已上线”报告。

## 01｜结论与目标

采用“一套 MCP、多用户独立授权”的架构。管理员配置应用与 Connector；每名用户在 Microsoft 365 Copilot Chat 的 FeishuMCP 连接入口完成微软身份确认、飞书官方授权和账号关联确认。后端自动保存授权，不再要求管理员导入个人 token，也不依赖 enrollment CLI。

用户日常体验：首次连接时授权，后续正常访问由后端维护令牌；授权撤销、刷新失效或安全策略要求重认证时，通过重新连接恢复。不承诺永久无需登录，也不承诺任意工具报错会自动弹出授权窗口。

**范围是普通 Copilot Chat 的自定义 Federated Connector。** 不改成专用 Declarative Agent，不依赖 Copilot Studio，不假定动态客户端注册（DCR）。微软该入口文档支持静态 OAuth 注册、授权/令牌/刷新端点及可选 PKCE，用户从 Sources 的 Connect 入口完成认证。[1][2]

只开放搜索和读取，不新增写入工具，不扩大已批准的使用人群。搜索、读取仅覆盖现有适配器能力：关键词文档搜索、Docx 元信息与正文块、Wiki 指向 Docx 的解析；不是全量文档同步器，也不把“关键词搜索前几条”冒充“全库最新文档”。

## 02｜当前已核实的基线

以下来自真实 Git、systemd、数据库只读检查和已登录管理后台，不是历史推测。

- GitHub：`wpsl5168/feishu-mcp`；线上对应提交 `276cd194c5447a89e67102961d86d07c44238733`，运行中的 Python 源码与该次本地基线一致。
- 生产服务：`feishu-mcp.service`；本机监听 `127.0.0.1:18766`；公网地址 `https://mcp.brickhub.cc/feishu/mcp`；写入关闭。
- 生产凭据库：已有一个 `ready` 用户绑定。历史 `EnrollmentRequired` 已解决，不作为本轮仍在故障的依据。
- 真实管理后台：**FeishuMCP 为 Ready、On**，端点正确；**CustomFeishuMCP 为 Disabled**，是另一个对象。
- FeishuMCP 目前管理端显示对所有人可见，但 MCP 后端仅允许明确列出的微软用户。展示范围和实际访问权限不是一回事；本轮不擅自改变原有范围。
- 精确 Connector 对象：`T_bd1d6d99-d84c-dd7a-cb12-994b8403d6f9`。其详情接口没有返回 OAuth reference ID，不能据此认定没有 OAuth，也不能编造绑定关系。灰度新登记必须单独读回并留证。
- 原链路采用 Entra 入站 JWT，再按已验证的 `tid:oid` 查飞书授权；保留原密钥、状态目录、绑定和服务。
- 改造前完整测试：187 项测试与 53 个子测试通过。新增基础模块后，本机再次运行完整测试得到 372 项测试与 53 个子测试通过；这仅证明该本地代码快照的离线测试状态。

配置和数据库已经分别备份，SQLite 快照完整性检查通过。**备份不是允许把旧刷新令牌恢复到在线服务继续使用**；回滚优先退回代码与路由，保留现用凭据状态。

## 03｜架构选择与取舍

### 推荐：OAuth 代理 + 真实 Entra 身份校验 + 飞书用户授权

- Copilot 只持有本服务签发的 MCP 专用令牌。
- 代理通过真实 Entra 登录取得并验证微软唯一身份，保留原租户和用户范围。
- 飞书官方页面完成登录、扫码（以官方页面实际能力为准）和权限确认；代理保存飞书授权。
- OAuth 代理与 MCP 可使用同一代码包、同一灰度服务进程，不另建重型身份平台。
- 代价：微软浏览器会话失效时，可能多一次微软登录；账号关联确认页也是一次显式操作。不能承诺一定静默跳过。

### 可选：只以飞书身份作为代理用户主体

可以用飞书企业标识和应用内用户 ID 做主体，并配置严格飞书用户白名单。流程更短，但它**改变了原来的微软用户级访问边界**，不能从 OAuth client ID 或 URL 参数推导微软用户。需独立批准身份模型变更后再考虑，不作为本轮默认实现。

### 不采用：把飞书令牌直接当成 MCP 入站令牌

这会混淆“访问飞书 API”和“访问本 MCP”的授权。MCP 官方安全文档明确禁止接受不是专门签发给 MCP 的令牌并向下游透传。飞书 Token 能调用 user_info，并不能证明它的目标资源就是本 MCP。[7]

## 04｜三方分别负责什么

**Microsoft 365 Copilot**：显示标准连接入口；保存自己获得的 MCP 授权；调用 MCP 工具。它不应获得后台飞书 Access Token 或 Refresh Token。

**我们的服务**：提供 Web/OAuth 入口、两段授权事务、可信账号关联、MCP 专用令牌签发与校验、飞书凭据加密与刷新，以及 search/fetch 工具。

**飞书官方**：验证飞书登录身份，展示授权页面，签发和刷新飞书用户令牌，并在每次文档 API 调用时执行文档访问控制。

这里的“授权入口由 MCP 做”，准确说是**与 MCP 共部署的 OAuth/Web 模块负责转接和签发自己的通行证**，不是 MCP 工具收集或验证飞书密码。

## 05｜首次连接的完整过程

1. 用户用自己的微软账号打开 Copilot，选择灰度 FeishuMCP Connector 并点击 Connect。
2. Copilot 按静态 OAuth 登记进入本服务 `/authorize`，携带客户端、精确回调、state、scope，以及启用时的 PKCE challenge。
3. 服务校验请求是否合法，创建短期授权事务，并通过浏览器安全 Cookie 绑定本次操作。未知客户端、错误回调、重复参数、错误 resource 或越权 scope 在此拒绝。
4. 跳转当前租户的 Entra 官方登录。后端通过授权码流程验证 ID Token 的签名、issuer、audience、nonce、时间字段、tenant 和用户 Object ID；再检查允许使用的人群。[8]
5. 通过后创建另一套独立的飞书 state 和 PKCE，跳转飞书官方授权页。请求最小搜索/读取及刷新权限，不让用户把密码或验证码交给 MCP。[3][4]
6. 飞书回调本服务。服务先验证 state、浏览器会话、时效和一次性状态，再向飞书 v3 令牌端点兑换；不自动重试可能已经消费的授权码。[3][4]
7. 用新获得的飞书令牌读取官方 user_info，取得 `tenant_key`、`open_id` 和显示名称。邮箱、手机号、同名都不用于自动跨系统匹配。[6]
8. 显示关联确认页：已验证的微软账号 → 已授权的飞书账号，并写清“允许此微软账号通过 Copilot 读取该飞书账号可访问的文档”。名称只用于展示，真正关联使用唯一 ID。
9. 用户确认后，在用户级锁下提交关联、加密凭据和新的连接版本号。随后向微软注册的精确回调返回**本服务的一次性授权码**，不是飞书授权码。[1]
10. Copilot 的令牌服务拿该码及客户端认证、PKCE verifier 换取 MCP 专用 Access Token 和独立 Refresh Token，回到 Chat 后调用 search/fetch。

用户取消或流程失败时：不建立半成品连接、不覆盖原来可用的绑定；已取得但尚未确认的飞书凭据仅在短期加密事务中暂存，到期清理。默认不允许静默改绑到另一个飞书账号。

## 06｜确认页如何实现

确认页由本服务提供，不是管理员操作页。

- 微软身份来自真实验证的 OIDC 结果；飞书身份来自官方 user_info，不信任网页传来的 `user_id`。**独立 OIDC 登录只证明本次向代理授权的微软身份，并不自动证明它与 Copilot 界面当前登录账号相同。** 页面应明确显示“本次授权的微软账号”，不能写“已核对当前 Copilot 账号”。运营流程要求用户确认使用预期账号；若业务必须机器强制两者相同，需额外取得 Connector 可验证身份绑定机制，否则该要求必须列为阻塞，不能靠提示文字假装实现。
- 待确认的绑定保存在服务端加密事务中；浏览器只拿高熵、短期、一次性的事务句柄，不携带 token 或可修改的账号绑定对象。
- 提交必须通过 Cookie、CSRF/事务校验，并再次检查身份范围及预期连接版本。
- 展示文本全部 HTML 转义；页面不加载第三方脚本、统计或图片；设置 no-store、no-referrer 和受限 CSP。
- 采用唯一性约束：默认一个微软用户关联一个飞书账号，一个飞书账号不被静默关联给多个微软用户。如需共享/多账号切换，另行设计和审批。
- 重连原账号无需管理员参与；变更为另一个账号需显式解除与重新确认，不能利用一次过期扫码覆盖当前账号。

这一步保留了明确的跨账号授权确认。若以后要减少一次点击，只能在有可核验的既有同一账号关联、风险评审通过后优化，不能靠猜测邮箱或省略同意。

## 07｜日常文档查询与十用户场景

```text
张三的 Copilot 请求
  → 校验张三的 MCP 令牌
  → 查张三的连接版本与飞书授权
  → 用张三的飞书用户令牌查询
  → 飞书按张三权限返回文档

李四的 Copilot 请求
  → 独立验证李四身份
  → 仅取李四的飞书授权
  → 飞书按李四权限返回文档
```

同一个服务可以支持十名或更多获准用户，不需要每人部署一套 MCP。管理员把应用开放给十人，并不等于这十人已经完成 OAuth，也不等于给他们新增了所有文档权限。每人须用自己的微软账号和飞书账号授权；共享一个 Copilot 账号无法可靠区分实际操作者。

MCP 不使用“统一管理员 Token 查询后自行过滤”的方式。文档权限取决于飞书用户现有 ACL 与本次应用 scope 的交集；scope 足够不等于有某篇文档的阅读权限。[2][4]

当前搜索 API 有结果窗口和分页限制；Docx 按块分页读取。必须遵守 `has_more/page_token`，不能将一页结果称为全量，不顺带扩展 Sheets、Base 或任意 API 代理。

## 08｜多个 Session 并发为什么不会串台

**用户身份是隔离主键，Session 不是“当前用户”的全局变量。** MCP 请求验签后才能得到 `tid:oid`，工具参数、HTTP 任意身份头和 query 不能覆盖它。

- 同一用户多个 Copilot 对话可能共用连接令牌，也可能存在多条 OAuth session/family；不假定一段对话对应一个 family。无论哪种方式，请求都按验签后的授权主体隔离。
- 不同用户并发：按用户主键分别查凭据、分别加锁；不存在默认用户、最近登录用户或全局 UAT。
- 多个扫码流程：每次独立 state、PKCE、阶段和时限；浏览器会话绑定与服务端事务对应，不把 A 的回调写入 B 的记录。
- 同用户并发刷新：按用户使用跨进程文件锁；一个请求刷新，其他请求等待并重新读取新凭据。不同用户不共用同一把业务锁。
- 同用户同时重连：确认事务记录预期 generation，使用比较后更新；较早的回调不能覆盖已经完成的新连接。
- 重连与刷新竞态：统一按“用户凭据锁 → 短数据库事务”的顺序提交，禁止在数据库写锁内等待外部网络；禁止重复获取同一用户的独立 flock 导致死锁。
- 下游 JWT 包含并校验用户、客户端、audience、scope、session ID、连接版本。旧版本会话在重新授权后失效。

现有基础模块离线测试覆盖了任务和进程并发、重放及身份隔离。最终仍须两名真实用户在 Copilot 中并发查各自私有文档，不能用单用户或 mock 冒充通过。

## 09｜两段 OAuth 与令牌生命周期

### A 段：Copilot ↔ 本服务

- 本服务是 OAuth Authorization Server，预注册固定 confidential client；不开放 DCR。
- 向 Copilot 签发自己签名的 RS256 JWT；issuer 和 audience 固定为灰度服务与灰度 MCP resource，只授权 `Mcp.Read`。
- MCP 严格验签、issuer、audience、时间、scope、客户端、人群、session 状态和连接版本；不接受生产 Entra token 或飞书 token 的宽松回退。
- 下游 Refresh Token 为独立随机字符串，库内只存哈希，绑定 client、user、resource、scope、family 和期限；原子轮换，匹配客户端重放已使用令牌时吊销该 family。
- 不把下游刷新和飞书刷新混为同一事务；Copilot 的重试/轮换持久化行为需真实验证。

### B 段：本服务 ↔ 飞书

- 官方授权页：`https://accounts.feishu.cn/open-apis/authen/v1/authorize`；授权码兑换与刷新：`https://accounts.feishu.cn/oauth/v3/token`。[3][4][5]
- `response_type=code`；固定回调；独立 S256 PKCE；confidential client 以 form body 传 client_id/client_secret，不与 HTTP Basic 混用。[3][4]
- 请求最小 scope：`search:docs:read`、`docx:document:readonly`、`wiki:node:read`、`offline_access`；其中 Wiki scope 用于保留现有 Wiki→Docx 能力。
- 在授权码兑换及每次刷新显式传缩权 scope，并检查实际返回值。飞书存在累计授权行为，不能只看授权 URL 就宣称 token 只有只读权限。[3][4][5]
- 飞书 Refresh Token 单次轮换，期限按响应保存；不硬编码示例有效期，也不承诺无限续期。官方注明授权达到相应长期边界后需重新授权。[5]

### 生命周期策略——设计目标，不是已交付参数

- MCP Access Token：建议 5 分钟，缩短泄露窗口。
- 授权事务：建议 10 分钟；本服务授权码：建议 60 秒且单次使用。
- MCP Refresh 会话：上线前实现可配置的闲置到期与绝对到期，建议先评估“30 天闲置、最长不超过 365 天且受上游有效性约束”；具体值须由安全策略与实测批准。活动会话不应无理由每天要求重新扫码。
- **当前基础模块仍是绝对固定 Refresh TTL，默认 1 天、配置上限 7 天；不满足上述长期体验目标，必须在正式上线前调整并补测。** 不把代码默认值包装成最终方案。
- 上游刷新在接近 Access Token 过期时触发；已撤销授权可能要等 API 拒绝或刷新时才被发现，不能把本地 ready 状态当作实时撤销检测。

## 10｜刷新、撤销与重新连接

飞书刷新顺序：取得用户锁 → 读取最新凭据 → 确认需要刷新 → **先持久化 refresh_pending 并移除旧令牌的可重放状态** → 单次调用飞书 → 校验身份/scope/新令牌/有效期 → 加密提交 ready。

- 网络超时、进程中断、响应不完整或成功后落盘失败：保留不确定状态，不恢复旧 RT，不盲目重发；必须 fresh authorization。
- 可信用户映射独立于会被 tombstone 清掉的凭据内容，因此凭据失效后仍能正确识别要恢复哪一条连接。
- 新授权完整确认后，更新连接版本并使旧下游会话失效。不能通过回滚数据库让旧 family 重新有效。
- 用户主动断开 Copilot，不等于已经撤销飞书全部授权；本地 session 吊销、Copilot token-store 清理、飞书撤销是不同动作，分别验证。
- 上游授权失效后，工具返回安全的重新连接说明；后续入站检查/下游刷新拒绝已失效连接。**不承诺 `EnrollmentRequired` 或任意 tool error 自动弹窗。**

可执行恢复路径：Copilot Settings → Sources（或租户实际界面的 Add/manage sources）→ 选中该测试源 → Disconnect/Sign out → Connect → 完成官方授权。最终交付必须以实际界面记录精确名称、步骤及是否保留旧 token；官方文档的通用入口说明不能替代本租户实测。[2]

## 11｜错误分类和用户提示

- **未完成授权**：提示“请从此 Connector 的连接入口完成授权”，不自动套用别人的授权。
- **授权撤销或刷新不确定**：提示“连接已失效，请断开后重新连接”；标记需要重连，失效会话不能继续发新 MCP 令牌。
- **缺少应用 scope**：说明需要重新同意相应读取权限；如果是飞书后台没开通，提示管理员修复应用权限，不让用户无限扫码。
- **没有某篇文档权限**：提示申请该文档访问权限；不误导为“重新登录必然解决”。
- **限流或飞书服务异常**：提示稍后重试。只读请求可以按限定策略退避；已经发送的刷新请求仍不得自动重放。
- **客户端、PKCE、state、回调或签名错误**：安全失败，返回稳定错误分类和随机 correlation ID，不暴露上游完整错误体。

API 错误码按当前官方文档分类；不猜未出现过的错误码，也不将 HTTP 200 等同工具成功。[4][5]

## 12｜数据模型与密钥管理

建议逻辑对象如下，均在单机私有状态目录内：

- `connections`：已验证的 Entra subject、飞书 app/tenant/open_id、generation、active/status；双向唯一性约束。
- `credentials`：绑定 subject 的加密飞书 Access/Refresh Token、实际 scope、两个过期时间和刷新状态；不依靠 ciphertext 中的旧身份信息恢复已擦除凭据。
- `authorization_transactions`：短期阶段、浏览器绑定哈希、两段 PKCE/nonce、下游请求绑定；敏感 payload 加密。
- `authorization_codes`：本服务授权码哈希、加密授权对象、期限；消费后不能再次交换。
- `sessions` 与 `refresh_families`：客户端、subject、连接版本、scope/resource、期限、撤销标记、Refresh Token 哈希及已使用记录。
- `audit_events`：事件类型、结果、版本、随机关联 ID及必要的最小用户标识；不保存 token、code、secret、正文或原始请求体。

密钥分开：飞书 App Secret、Entra OIDC Client Secret、下游 OAuth 客户端 Secret、JWT 签名私钥、凭据加密密钥。它们不是一把万能钥匙。本服务用于验证下游 OAuth 客户端的 Secret 只存校验哈希；飞书与 Entra 上游 Client Secret 则必须以可供兑换使用的受保护形式保存，不能只存哈希。签名密钥和加密密钥重启后持久化，不能临时生成替换。

文件目录 0700、密钥及数据库 0600；保护 symlink/hardlink；密钥与状态备份分离。密钥丢失或损坏时拒绝使用，不自动生成新密钥“修复”旧库。多机 HA 不在本轮范围：现有 flock 只协调同机进程，不适用于复制数据库后的多节点并发刷新。

清理任务必须有 TTL 和容量上限：过期待授权事务/授权码清理，过期 session/family 按保留策略清理，永久绑定只按获批注销处理；不能让未认证 `/authorize` 无限增长数据库。

## 13｜公网端点与 Connector 配置

以下均为**拟议灰度地址，尚未部署**：

- Base URL / issuer：`https://mcp.brickhub.cc/feishu-oauth`
- MCP endpoint：`https://mcp.brickhub.cc/feishu-oauth/mcp`
- Authorization endpoint：`https://mcp.brickhub.cc/feishu-oauth/authorize`
- Token 与 Refresh endpoint：`https://mcp.brickhub.cc/feishu-oauth/token`
- 本服务撤销端点：`https://mcp.brickhub.cc/feishu-oauth/revoke`
- Entra 回调：`https://mcp.brickhub.cc/feishu-oauth/entra/callback`
- 飞书回调：`https://mcp.brickhub.cc/feishu-oauth/feishu/callback`
- 确认页提交：`https://mcp.brickhub.cc/feishu-oauth/confirm`
- JWKS：`https://mcp.brickhub.cc/feishu-oauth/jwks`
- AS metadata：`https://mcp.brickhub.cc/.well-known/oauth-authorization-server/feishu-oauth`
- Protected Resource metadata：`https://mcp.brickhub.cc/.well-known/oauth-protected-resource/feishu-oauth/mcp`

微软侧新建静态 OAuth registration，填写本服务的客户端 ID/Secret、上述端点、`Mcp.Read` 与 `offline_access`；开启 PKCE，优先 S256。客户端认证方式明确选定并验证，建议先用 Portal 的 Request body parameters；不能从“支持 OAuth”推断具体请求格式已实测。[1]

新 registration 设为当前组织可用，按共享登记系统和本租户界面选择合适应用范围；随后将返回的 **OAuth registration reference ID** 填入新测试 Connector。它不是 client ID，更不是飞书 App ID。

本服务允许的微软第三方 OAuth 回调严格限定为：
`https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect`。[1]

`oAuthConsentRedirect` 属于另一种 Entra SSO 配置，不能无依据混入本服务第三方 OAuth 回调列表。Entra 登录和飞书登录分别使用本服务各自的回调，不把微软 callback 直接配置成飞书→代理回调。

**关于“URL 不泄露授权码”**：标准授权码协议必须向预注册回调短暂传送 code，飞书官方当前采用 GET query code/state。这不是把令牌公开到任意 URL；不能承诺完全没有 code 出现在协议回调 URL。[3] 实施要求是：只发精确 HTTPS callback、PKCE与单次短效、禁 query/Location 日志、no-referrer、无第三方资源，不把 code 放进业务链接、截图、聊天、审计或仓库。若要求连精确协议回调 URL 也绝对不能包含 code，须先确认上游与微软 callback 都支持 form_post；目前未证实，属于需要另议的产品约束。

## 14｜部署、灰度与保护原服务

### 灰度资源

- 复用现有主机、Caddy 和证书；新增精确 `/feishu-oauth/*` 及对应 well-known 路由，不重写旧 `/feishu/*`。
- 独立 systemd 服务，拟名 `feishu-mcp-oauth-pilot.service`，loopback 端口拟用 18768；部署时重查占用。
- 若 Caddy 在容器网络中，沿用已验证的主机桥接暴露模式，只给新端口加对应受限桥接代理，不把应用绑定公网，不修改旧端口。
- 独立配置文件、状态目录、签名/加密密钥和版本目录；禁止复制生产 Refresh Token 到灰度数据库。
- 独立 Entra OIDC 测试应用、独立下游 OAuth 客户端/registration、独立测试 Connector，拟名 **FeishuMCP-OAuth-Pilot**。
- 独立飞书测试应用是 AC4 撤销测试的前置条件，开启最小读取与刷新权限。无法提供独立应用则阻塞该项验收，不能静默降级为在生产应用/同一用户上撤销授权。

### 实施顺序与每步产物

1. **基线与备份**：已完成；保留实际 Connector 对象、代码 SHA、配置定位、数据库完整性和原路由基线。
2. **代码集成**：完善 Web 路由、两段授权、确认页、错误反馈、JWT验证、刷新协调、TTL/容量和日志脱敏；保留旧启动入口。
3. **离线门禁**：完整测试、构建、秘密扫描、独立安全评审通过；生成提交 SHA 和测试证据。评审超时不算通过。
4. **变更确认**：列出全部注册、回调、scope、用户名单、服务和路由变更，取得明确批准。
5. **灰度部署**：配置先备份、Caddy 验证再 reload、systemd 新服务就绪检查、公网TLS/metadata/401/路由及旧服务回归。此时只称“灰度已部署”。
6. **真实 Copilot 验收**：执行后述七项；需要本人扫码或管理员同意时暂停交接，不索取密码/token。
7. **正式切换**：验收完成后单独批准。保留旧 FeishuMCP，优先按明确用户名单迁移新源；如果必须保留同名入口，先验证管理界面是否支持原地更新，不能假定可改或擅自删建。重新授权是合法迁移方式，不复制旧单次刷新令牌。

生产现状虽然“所有人可见”，新灰度必须显式设置 Specific users/groups，仅包含批准的测试用户。默认先不增加现有后端允许用户；新增测试用户须单独明确 ID 和使用范围。

## 15｜回滚方案

**灰度回滚**：禁用新增测试 Connector → 停止新灰度服务/桥接代理 → 删除或关闭仅新增的精确反代路由并验证 → 保留旧 FeishuMCP 和旧服务原样。新客户端或新应用是否撤销由批准范围决定，不清理共享资源。

**代码回滚**：切回记录的上一版 release symlink/启动目标，仅操作灰度服务；新旧版本对状态 schema 的兼容必须先测试。无法降级时，停止灰度而不是破坏性重建数据。

**凭据回滚**：默认不恢复旧数据库快照。Refresh Token 可能已经轮换，恢复快照会重放失效令牌；保留当前状态，确需恢复时对受影响用户重新授权。

**原服务验证**：检查原 PID/启动目标、健康、metadata、既有用户 search/fetch、原路由及其他站点基线。单纯“systemd active”不等于业务未受影响。

## 16｜真实 Copilot 验收清单

每项记录时间、部署 SHA、具体测试源、脱敏用户标识、服务 correlation ID、结果与证据路径。截图不含密码、token、code 或私有正文。

### AC1：首次自助连接
使用从未绑定新灰度服务的明确测试用户，在实际 Copilot Sources 点 Connect；完成 Entra校验→飞书官方授权→确认→返回。禁止预先运行 enrollment CLI。要声明该用户此前是否使用过旧生产源，不能把“新数据库”偷换成“全新身份”。

### AC2：真实 search/fetch
由 Copilot 实际调用两类工具；结果与该用户在飞书直接可见的受控测试文档一致，服务日志可关联。curl/Inspector/操作员直连仅作辅助，不替代。

### AC3：新会话与服务重启
新建 Copilot 对话继续查询；重启仅灰度服务后重复查询，不重新扫码；验证持久化key/state及session状态。另验证成功刷新延长闲置期限但不延长绝对期限、过期重新认证、移出本服务允许名单后旧令牌和刷新均拒绝。长效Broker令牌不等同实时执行Entra条件访问/CAE；如要求持续执行，需另加正式支持的机制，不据首次登录推定持续满足。重启前需确认没有另一用户进行授权交易。

### AC4：两段刷新、撤销与重连
分别证明 Copilot→代理的 MCP令牌刷新和代理→飞书的用户令牌刷新确实发生，而不是一次初始授权。灰度可用受控短寿命MCP令牌；飞书令牌真实轮换用专用测试凭据进行受控验证，不篡改生产库或伪造上游响应。撤销仅测试飞书应用授权，确认旧连接拒绝，再从实际 Sources 重连恢复。

### AC5：两用户隔离及并发
准备两名获批微软用户、对应飞书账号，以及 A-only、B-only、shared 测试文档。A读A-only成功、读B-only失败；B反向同理；shared均可。并发多会话和刷新重复测试，包括多个对话共用同一 Refresh family 的竞争、重试及重连后旧会话失效。另测“Copilot A／浏览器 Entra B”情形：如均在白名单且 B 显式完成授权，后端看到的是 B 的授权主体，不得宣称识别成 A 或已证明两端同人；严格同人要求未获协议支持时阻塞上线。文档内容由测试方提供或另获授权创建，本轮不擅自写客户文档。

### AC6：授权失败安全性
用户取消、错/过期state、错误浏览器会话、授权码重放、PKCE错误、callback/client/resource不匹配、重复参数、非法JWT均安全失败。可系统化用离线与本地HTTP测试覆盖攻击矩阵；真实浏览器至少验证取消/返回/正常恢复，明确证据来源，不把模拟攻击说成Copilot真实触发。

### AC7：旧链路及回滚
灰度期间原FeishuMCP与原绑定仍可用；进行一次获批的灰度停用/路由回退演练，再恢复灰度，原服务保持正常。最终交付可执行回滚命令及目标，不依赖历史口头记忆。

七项不能只通过一部分就宣布整体完成。无法取得第二测试用户、必要授权或租户功能时，列明确阻塞、负责人和剩余验证，不编造通过结果。

## 17｜当前完成度与剩余工作

**已核实/已完成**：生产基线、精确Connector状态、配置和DB备份、微软/飞书官方协议研究；本地 `oauth_config.py`、`oauth_state.py`、`oauth_upstream.py` 及对应基础测试已形成工作区改动，本机完整测试快照为372通过、53子测试通过。

**尚未完成**：完整授权HTTP路由与确认页集成、新MCP入站校验接线、上游失效后的路由/工具处理、长会话策略调整、事务清理和容量门禁、端到端本地路由测试、独立安全评审、新版本提交/CI、新服务部署及全部真实Copilot验收。

**当前生产仍是原版本**，不是新OAuth代理。新增基础模块尚未作为此次改造版本推送；不把本地测试通过当成已部署。之前独立安全架构评审因模型调用超时未完成，必须补做，不能标记已审。

资料与代码合同在 `~/reviews/feishu-oauth-upgrade-20260928/`；代码分支为 `feat/copilot-self-service-oauth`。本方案的生成不授权任何新租户注册、权限或生产切换。

## 18｜待批准的具体变更与输入

批准前不执行以下外部变更：

- 新建专用Entra OIDC应用，登记本服务Entra回调；只请求身份所需的openid/profile，不申请Graph业务权限。
- 新建或由飞书管理员提供独立测试应用，登记本服务飞书回调、开通四项最小scope及刷新设置并发布；禁止改动原应用的授权范围来完成撤销测试。
- 新建静态Broker OAuth客户端与微软OAuth registration；登记本服务三个端点、精确微软callback、PKCE、client认证方式及组织范围。
- 新建FeishuMCP-OAuth-Pilot，只分配给明确测试用户；未确认第二用户前不扩大范围。
- 新增灰度systemd/受限桥接代理/精确Caddy路由，复用证书；不改旧服务、密钥和状态目录。
- 进行灰度服务重启、测试应用授权撤销和灰度回滚演练；正式生产切换另批。切换审批须再次实测旧用户授权仍有效；若旧链路授权已过期或被撤销，应准备原入口重新授权恢复，不能默认旧绑定永远可回用。

还需测试方提供：第二个获准微软测试账号及对应飞书账号、必要Copilot使用资格、A/B/shared受控文档，以及本人扫码/管理员同意的配合窗口。只要账号标识和操作确认，不索取密码、验证码、token或secret明文。

## 19｜最终交付物

1. 可复现代码、锁定依赖、测试、构建产物和Git提交/CI链接。
2. 经核验的新OAuth注册/Connector配置说明，秘密不写入文档。
3. 部署清单、实际版本、状态目录与密钥管理说明。
4. 逐项AC1—AC7证据，区分模拟、本地HTTP、公网操作员验证和真实Copilot。
5. 用户首次连接与重新连接操作说明。
6. 监控/错误分类、凭据轮换边界、迁移与回滚Runbook。
7. 明确未完成事项及权限/产品限制。只有全部验收达到原要求，才将状态从“已实现/已灰度部署”改为“真实Copilot验收完成”。

相关入口：[[20-项目/客户交付/README|客户交付]]。

## Sources

[1] https://learn.microsoft.com/en-us/microsoft-365/copilot/connectors/set-up-custom-federated-connectors — Custom Federated Connector 设置
[2] https://learn.microsoft.com/en-us/microsoft-365/copilot/connectors/federated-connectors-overview — Federated Connectors 总览
[3] https://open.feishu.cn/document/authentication-management/access-token/obtain-oauth-code.md — 飞书获取授权码
[4] https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/authentication-management/access-token/get-user-access-token-v3.md — 飞书用户令牌 v3
[5] https://open.feishu.cn/document/uAjLw4CM/ukTMukTMukTM/authentication-management/access-token/refresh-user-access-token-v3.md — 飞书刷新用户令牌 v3
[6] https://open.feishu.cn/document/server-docs/authentication-management/login-state-management/get.md — 飞书 user_info
[7] https://modelcontextprotocol.io/specification/latest/basic/security_best_practices — MCP Security Best Practices
[8] https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow — Entra OAuth 授权码流程
