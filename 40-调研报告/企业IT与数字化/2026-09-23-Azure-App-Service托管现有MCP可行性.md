---
title: Azure App Service托管现有MCP可行性
created: 2026-09-23
updated: 2026-09-23
type: research
tags: [mcp, architecture, research]
status: draft
---

# Azure App Service 托管现有 MCP 可行性

## 结论与范围

本次将“Azure app”按 Azure App Service / Web App 理解，不是手机 Azure 管理 App。用户未指定 MCP 名称，因此同时检查 SQL MCP 和最近的飞书文档 MCP。

**App Service 官方支持托管自建 MCP。SQL 版可复用容器迁移；飞书版可部署，但当前代码不是直接上传即可可靠运行。** 微软明确区分“Bring your own MCP server”和“built-in MCP”：前者托管现有 SDK 实现，后者从 REST/OpenAPI 生成 MCP。[1]

本轮只读查看项目代码与官方文档；未创建 Azure 资源、未修改项目代码、未做 App Service 云端部署或重新验证 Copilot。既有测试结果只是项目历史证据。

## 推荐链路

M365 Copilot → custom federated connector → HTTPS App Service `/mcp` → 项目内 Entra JWT 鉴权 → SQL / 飞书 REST。

App Service 是代码宿主，不是替代 MCP Server，也不会替你完成 Microsoft 365 Connector 登记、飞书用户授权和身份绑定。APIM 是否保留是单独的网关治理决策，不是使用 MCP 协议的必要条件；若去掉网关，要显式重新审核项目的网关依赖，不能只删除一项部署资源。

## SQL MCP：适合先迁移验证

本地证据：`~/projects/azsql-mcp-demo/`。

- `src/server.py:44` 使用 `stateless_http=True,json_response=True`；不是只有本地 stdio 的程序。
- `src/server.py:315–316` 用 Uvicorn 运行 Streamable HTTP ASGI app。
- `src/Dockerfile:20,34–49` 已安装 Microsoft ODBC Driver 18，监听 `0.0.0.0:8080`，提供 `/healthz`，因此优先复用镜像，避免用源码部署重新处理系统 ODBC 依赖。
- 该 Dockerfile 默认 `MCP_REQUIRE_GATEWAY=true`；`src/config.py:91` 和 `src/server.py:292–305` 检查网关 secret/header。App Service 前端不会天然提供项目自定义网关凭据。需要保留受控网关，或经审批采用不要求该网关的入口配置，同时保持 Entra 校验、最小权限和访问限制。
- `README.md:25` 表明仓库原 Azure IaC 分支针对 Container Apps/APIM，不是 App Service；需要另写 Web App 的部署配置，不应直接运行旧脚本。
- MCP 托管不包含数据库迁移。现有实验使用 VM 内 SQL Server Developer；改为 Azure SQL 要单独核连接、网络、身份和数据库兼容性；保留原库则需设计私网访问，不能为了演示裸露 SQL 端口。

App Service 传统自定义容器配置需将目标 HTTP 端口对应到容器端口；官方文档给出 `WEBSITES_PORT`。采用新版 sidecar 部署模式时须再核该模式的端口字段，不盲用传统设置。[3]

## 飞书 MCP：两个部署阻塞点

本地证据：`~/projects/feishu-docs-mcp/`。

### 1. 监听地址

`feishu_docs_mcp/server.py:43` 同样使用无状态 HTTP/JSON。但 `config.py:57–58` 明确只允许 `127.0.0.1`/`::1`；简单把 `MCP_HOST` 改成 `0.0.0.0` 会触发异常。

建议二选一：经安全评审加入容器监听模式并保留应用层 Entra 验证；或在同容器内由反向代理监听外部端口，再代理到 loopback。此处只是设计建议，尚未修改或验证。

### 2. 凭据和操作台账不是无状态

`README.md:40` 明确仅支持本地文件系统，不支持 distributed replicas/NFS。`tokens.py` 使用 SQLite 与 `fcntl.flock` 协调单次 refresh token；项目还持久化加密凭据和写入幂等台账。

App Service Linux 自定义容器默认未开启持久化；普通容器目录不能保证重启后保留。开启 `/home` 持久化后，该目录可以跨实例共享，也不能据此认为它等价于当前代码已验证的本地磁盘语义。[3]

因此：

- 不应把 SQLite 移到 `/home` 就宣布生产完成；锁、并发、重启与刷新令牌不重放必须验证。
- 即使只开一个实例，也没有消除重启丢失、部署替换、共享存储锁语义等问题。
- 面向托管平台的长期方案是外置事务型状态存储，并实现原子幂等 claim、凭据更新协调与刷新结果未知的 fail-closed 状态；不是只换 SQLite connection string。
- Key Vault 可用于应用密钥管理，但不能替代业务幂等账本与并发控制。
- 在状态层未改造前，保留 VM 本地盘部署是改动更少的路线；换 Container Apps 也不会自动解决本地状态问题。

另外，`README.md:94` 说明目前仍由管理员绑定 Entra 身份与飞书用户凭据，自助 OAuth enrollment/callback 尚未实现。App Service 不会自动补齐此功能。

## 三个候选

- **App Service + 自定义容器**：适合要摆脱 VM 维护、已有稳定 HTTP 服务的场景。SQL 版优先；飞书版先处理监听和状态。保留现有 MCP 的工具定义与安全逻辑。[1]
- **Azure Container Apps**：SQL 仓库已有对应 IaC，但不是已部署成功的云端产物。若目标是沿用现有部署设计，可继续评估；不因它支持容器就默认优于 Web App。
- **现有 VM**：平台运维仍由自己负责，但最贴合飞书版当前本地文件系统设计。仅验证业务时不必为了 MCP 特意迁移。

本轮未查询订阅库存、地区/SKU报价或预算，不给未经核验的月费，也不声称某方案一定更便宜。

## 不建议为本项目改走 built-in MCP

App Service 内置 MCP 仍标注 Preview，通过 OpenAPI 3.0.x 把 REST operations 转为工具，当前不支持 OpenAPI 3.1.x；官方专页要求 Basic 或以上专用层。[2]

咱们已有 MCP SDK 实现、Entra 校验、用户隔离、幂等及审计逻辑。直接托管现有程序更符合目标，没必要先改成 REST 再由平台包装一次。上述 Preview 和套餐要求属于 built-in MCP 功能，不能扩大成“所有 App Service 自建 MCP 都是预览”。

## 验收清单

1. 明确迁移 SQL 版还是飞书版，确认 App Service 而非 Container Apps。
2. 本地验证发布镜像、监听端口、健康检查和无凭据 fail-closed 启动。
3. 云端验证 initialize → tools/list → 只读 tools/call；未认证/错误 audience/越权用户均拒绝。
4. 重启、部署切换、并发请求后核凭据与幂等状态，不丢数据、不重复刷新/写入。
5. 校验实际 HTTPS URL、OAuth 配置、Connector endpoint 与限定测试用户。
6. 普通 Copilot 实际调用与后端读回单独验收，不用直接 MCP 成功替代。

项目 SQL README 已记录此前普通 Copilot CRUD 实验成功，但依赖刻意不准确的 `readOnlyHint=true` 元数据，不是受支持生产写回合同。换宿主不会改变此边界。飞书版默认保持真实写工具声明，尚未完成真实 Entra/Copilot 全链路联调。

## 相关

- [[40-调研报告/企业IT与数字化/2026-09-22-飞书文档MCP与Copilot-Connector实现选型]]
- [[40-调研报告/企业IT与数字化/2026-09-18-Azure-SQL-MCP代码审查与M365接入评估]]

## Sources

[1] https://learn.microsoft.com/en-us/azure/app-service/scenario-ai-model-context-protocol-server
[2] https://learn.microsoft.com/en-us/azure/app-service/configure-mcp-built-in
[3] https://learn.microsoft.com/en-us/azure/app-service/configure-custom-container
