---
title: Copilot Studio与Dataverse业务Lab设计
created: 2026-10-18
updated: 2026-10-18
type: methodology
tags: [agent, workflow, tooling, security]
status: draft
sources:
  - https://learn.microsoft.com/en-us/microsoft-copilot-studio/knowledge-add-dataverse
  - https://learn.microsoft.com/en-us/microsoft-copilot-studio/advanced-connectors
  - https://learn.microsoft.com/en-us/connectors/commondataserviceforapps/
  - https://learn.microsoft.com/en-us/microsoft-copilot-studio/configure-enduser-authentication
  - https://learn.microsoft.com/en-us/power-platform/admin/database-security
  - https://learn.microsoft.com/en-us/power-platform/admin/manage-dataverse-auditing
  - https://learn.microsoft.com/en-us/microsoft-copilot-studio/requirements-licensing-subscriptions
---

# Copilot Studio与Dataverse业务Lab设计

> 面向已熟悉 Workflow、Agent、知识库和 Connector 的业务人员。本文是供客户讨论的场景与实验设计，已核官方能力文档，未进入客户租户搭建或验证。行业、课时、许可和培训环境尚待确认。

## 1. 推荐结论

推荐场景：**客户问题受理与跨团队跟进**。Agent 名称：**客户问题处理助手**。

业务人员在对话中查客户历史、登记问题、补充处理进展、查询办理状态；不同角色围绕同一条 Dataverse 业务记录协作。Agent 负责理解和交互，Dataverse 负责保存关系明确、可授权、可追踪的业务事实。

本课的学习增量不是再做一条审批流，而是把“能回答的 Agent”做成“能围绕业务数据持续办事的 Agent”。

### 候选场景取舍

- **客户问题处理（推荐）**：客户—产品—工单—跟进天然有关联，适合演示多表查询、对话写入、跨会话状态和多角色协作；业务门槛低。自建培训表，不依赖 Dynamics 365 Customer Service 的 Case 等受限表或应用。
- **采购申请与供应商协同**：适合已有采购背景的客户，供应商—申请—明细关系清晰；但容易把课程变成已熟悉的审批培训。
- **设备借用与归还**：最容易理解，但若涉及多人抢同一设备，就需要并发与占用控制；简化过度又容易退化为单表增删改查。

行业未定时先用第一个。确定行业后可换名称和素材，不必重做教学主线。

## 2. 面向客户的场景介绍

员工收到客户问题后，先让助手查询该客户的历史记录，判断是否有仍在处理的同类问题；再通过对话登记新工单。处理人员补充进展，负责人确认结果并结案。之后任何有权限的同事都能在新的对话中查询最新状态、处理依据和历史记录。

典型任务（所有名称与编号均为虚构教学素材）：

> “星河公司反馈 A 型设备再次掉线。先查一下之前的处理记录；如果没有相同的未结工单，帮我登记一个，预计明天下午回访。提交前让我确认。”

“相同问题”由 Agent 给出候选和依据，再由人判断，不以语义相似度自动认定重复。“明天下午”应追问或确认具体时间及时区，而非自行猜一个时间写入。

## 3. Dataverse 承担什么

### 最小数据模型

全部使用带培训发布者前缀的自定义表，避免把培训绑定到 Dynamics 365 应用许可。

- **客户表**：客户编号、名称、等级、联系人。客户编号用于消歧，不只靠名称。
- **产品表**：产品编号、名称、型号。
- **工单表**：自动编号、客户 Lookup、产品 Lookup、问题摘要、问题描述、业务状态 Choice、负责人、预计回访时间、处理结果、Owner、提交请求标识。
- **跟进表**：工单 Lookup、跟进内容、创建时间、创建人、Owner。

关系：一个客户对应多张工单，一个产品对应多张工单，一张工单对应多条跟进。新增跟进是增加子记录，不是覆盖工单描述。

讲师预建模型骨架；学员亲手增加“预计回访时间”、配置状态 Choice、建立一处 Lookup 并验证关联。让业务人员体验建模，但不把整节课耗在建表点击上。

### 与普通知识库的区别

- 知识库回答“这种问题通常怎么处理”，偏知识与经验。
- Dataverse 记录“这位客户的这张单现在由谁处理、进行到哪一步”，是当前业务事实。
- Dataverse 表也可以加入 Agent 的知识源，但**添加知识源不等于获得增删改能力**。
- 精确状态、指定工单、完整待办清单通过结构化查询工具读取；不能把知识检索召回结果当作完整清单或财务统计。

### 与 Excel / SharePoint List 的区别

简单单表登记仍可用 Excel 或 SharePoint List。两者并非完全没有关系、权限或版本能力；本场景选 Dataverse，是为了让关系模型、安全角色、结构化写入、变更审计及后续 Power Apps 共用数据集中在一个平台中，而不是声称其他产品做不到。

## 4. Lab 主线：一个 Agent，三个可验收阶段

建议按半天主线设计；若客户需要亲手配置权限、审计及 Power Apps 后台，可扩为一天。具体时长需按学员数量和环境预检后锁定。每阶段提供可导入的前置 Solution 与教学数据，避免上一阶段失败卡住后续练习。

### Lab 1：接到问题，先查清客户背景

**业务任务**：“星河公司之前有没有报过 A 型设备掉线？现在哪些工单还没结束？”

学员操作：

1. 查看客户、产品、工单关系，完成一处字段或 Lookup 配置。
2. 在 Copilot Studio 的 Tools 中添加 Microsoft Dataverse Connector 查询工具。
3. 固定可访问表与字段，通过 List rows / Get a row by ID 查询目标客户及工单。
4. 让 Agent 展示历史记录、当前未结工单及各自编号；同名客户必须先消歧。

学习增量：Lookup 不只是文本列；自然语言请求应落到明确业务实体和查询条件；关联记录可组成业务上下文。

验收：返回的编号、状态、客户和产品能与 Dataverse 逐条对上；没有记录时明确说明；若结果被截断或分页未取完，不声称已经列全。

可选对照：将预置工单的适用短文本字段配置为 Dataverse 知识源，补同义词与术语，比较“找相似案例”和“查准确状态”的差异。该能力不作为核心主线的单点依赖。

### Lab 2：用对话登记一张真实工单

**业务任务**：“给星河公司新建一张 A 型设备掉线工单，安排回访。”

学员操作：

1. 通过 Topic 收集缺失信息，查询并选定客户、产品对应的真实记录 ID。
2. 展示待提交摘要，要求用户明确确认。
3. 调用限定字段的 Add a new row，默认业务状态由流程明确设为“新建”。
4. 用返回的 ID 再调用 Get a row by ID，成功读回后才回复工单编号和状态。
5. 新开对话，用工单编号查询，证明业务事实不依赖上一轮聊天上下文。

学习增量：Agent 不再只生成一段文本，而是将用户意图转成有类型、有关系的持久化数据。

验收：工单真实存在；客户和产品 Lookup 正确；取消确认不写入；缺失必要数据不提交；重复点击不生成第二张相同请求的工单。

防重复设计：确认阶段生成并保存请求标识，重试沿用同一标识；对该字段设置 Dataverse Alternate Key，且确认索引已生效。唯一键冲突应查询已有记录并返回，不再生成新标识盲目重试。“同一提交重试”与“业务内容相似”是两类问题，不能混用。

### Lab 3：换个人接手，继续办完同一张单

**业务任务**：“为这张工单追加今天的排查进展；客户确认恢复后结案。”

学员操作：

1. 处理人员用工单编号读取最新记录，追加跟进子记录。
2. 使用受控 Topic 与 Connector 工具更新状态；课堂状态可限定为“新建→处理中→待客户确认→已结案”。
3. 结案前检查处理结果、客户确认等条件并让用户确认；缺条件时拒绝写入。
4. 换一个有权限的账号、新开对话查询同一编号，看到新的状态及跟进历史。
5. 讲师演示另一个团队账号无权读取或更新该工单，并展示已启用的字段变更审计。

学习增量：业务状态是共享数据，而非 Agent 的临时记忆；访问权限由 Dataverse 和工具执行身份决定，不由提示词决定；跟进记录与系统审计用途不同。

验收：新旧会话查询一致；跟进新增而非覆盖；越权读写失败；未满足结案条件时状态不变；审计能看到启用范围内的字段前后值和实际操作身份。

教学边界：课堂的顺序状态检查不等于生产级并发状态机。正式上线需补并发冲突控制、绕过 Agent 的写入口保护及错误恢复。不要把“先读再改”的 Topic 包装成原子事务。

## 5. 技术链路与落地顺序

业务用户 → Copilot Studio（理解、追问、确认）→ 受控 Topic / 工具 → Dataverse（查询、写入、关系、权限、状态）→ 读回结果 → 用户。

### 主线优先直接 Connector，不强塞 Flow

1. 在 Power Apps 中创建自定义表、字段、关系及教学数据，放入培训 Solution。
2. 在 Copilot Studio 中创建 Agent，配置 Microsoft 身份验证。
3. 在 Tools → Add a tool → Connector 中选择 Microsoft Dataverse，分别配置查询、新增、更新操作。
4. 将表名、允许返回和修改的字段、状态值限制在课程范围；不给模型一个可任意选择表、过滤表达式和写入字段的通用数据库工具。
5. 写操作放在明确的 Topic 路径中，加入校验、确认、执行和读回，不仅在系统提示词写“操作前请确认”。
6. 多步校验、防重复、事务组合复杂时，再封装 Agent flow；使用 Flow 后重新核其连接身份，不能假定自动沿用对话用户。

Power Apps 的简易后台由讲师预置、作为可选演示：业务人员可用表单查看或处理同一记录。它不是本 Lab 的必修开发任务，使用权另外核验。

### 权限演示的正确做法

- 作者账号与业务测试账号分开。拥有 System Customizer / 管理权限的作者账号，不适合证明普通用户权限隔离。
- 主数据按教学需要开放读取；工单和跟进采用用户/团队所有权，按两个测试团队配置安全角色与实际记录 Owner。
- 工单与跟进子表均配置权限，不能因为建立了 Lookup 就假定所有权和可见范围自动一致。
- 关联写入还要核查 Append / Append To 等所需权限。
- 优先使用 End-user credentials 的 Connector 工具，并在目标发布渠道实测。Agent 已登录，不代表所有下游工具也用该用户执行。
- 若使用高权限 Maker-provided credentials 或 Flow 服务连接，Dataverse 按该连接身份授权；此时不能声称普通用户权限自动透传。
- 业务“只能谁结案”的限制不能仅靠隐藏按钮或提示词；生产需明确服务端校验或字段/写入口控制方案。

## 6. 课前准备与验收清单

讲师准备：

- 独立测试环境和完整 Dataverse 数据库，不把 Dataverse for Teams 当等价环境。
- 培训 Solution、字段字典、脱敏/虚构教学记录、分阶段起点包、验收脚本。
- 同名客户、无历史、已有未结同类问题、正常新建、重复提交、缺结案信息、跨团队越权等测试样本。
- 作者账号、普通业务账号、另一团队业务账号；必要的测试团队与角色由管理员提前创建。
- 固定课堂状态和时间字段行为，确认环境/用户时区。

环境与许可：

- 确认客户是 Global 还是中国区租户，以及实际开放的生成式能力与发布渠道；不能仅凭某个连接器“所有区域”就推定整条链路可用。
- 核查 Copilot Studio 作者与运行权益、Copilot Credits/容量、Dataverse 数据库与日志容量。
- Microsoft Dataverse Connector 官方列为 Premium；不能只凭 Premium 标签断定每位用户必买哪种许可证，也不能凭 M365 Copilot 已授权就推定全套覆盖。
- 独立 Power Automate 流程、Power Apps 后台等按实际使用方式另核许可。最终以当期 Licensing Guide 和客户合同为准。
- Copilot Studio trial 文档允许创建和测试，但不允许发布。若只有 trial，不能承诺 Teams 发布和跨账号端到端验收已经可做。
- DLP/数据策略允许相关 Connector；作者具有建表权限，业务用户具有需要的数据权限。Environment Maker 不是数据访问权限的替代。
- Dataverse 知识源需开启 Dataverse search、具备表权限，并设置 Authenticate with Microsoft；提前完成索引配置和预热。
- 当前所核标准体验文档将 Multiline Text / File 的非结构化检索列为 preview，核心 Lab 不依赖它。知识源/术语更新不承诺立刻生效。
- 审计须在环境、表、字段层启用，消耗日志容量，展示可能延迟；业务跟进表不是原生审计的替代，原生变更审计也不自动覆盖所有读取/导出。

## 7. 可选扩展，不进入第一轮主线

- 超期跟进提醒：用 Dataverse 行变更触发或定时 Flow，明确运行身份和防重复，不能指望 Agent 在聊天结束后自己持续盯单。
- 费用减免审批：异步提交、返回待审批状态，审批后更新同一业务记录；不让对话同步等待人工完成。
- 管理视图：查询未结、待回访、超期工单。精确数量使用结构化查询/聚合，处理分页，不让模型对知识检索片段估算。
- Dataverse MCP：连接器文档中列有 preview 项，适合后续专题，不作为业务学员首个 Lab 的必需组件。

## 8. 与客户确认的事项

先确认“客户问题处理”是否贴近其日常业务；若不贴近，替换成采购申请或内部服务请求，但保留多表关系、受控写入、跨会话共享状态、权限验证的教学结构。

排课前确认：半天还是一天；学员是否亲自建表；能否提供完整 Dataverse 测试环境和发布权限；是否有独立普通用户账号用于权限验收。

## 官方依据

以下为 2026-10-18 实际读取的官方页面，不代表已验证客户租户：

- [Dataverse 作为知识源](https://learn.microsoft.com/en-us/microsoft-copilot-studio/knowledge-add-dataverse)：身份、搜索、同义词与预览边界。
- [在 Copilot Studio 使用 Connector 工具](https://learn.microsoft.com/en-us/microsoft-copilot-studio/advanced-connectors)：直接添加工具、Topic 调用与默认用户凭据。
- [Microsoft Dataverse Connector](https://learn.microsoft.com/en-us/connectors/commondataserviceforapps/)：Premium 分类、List/Get/Add/Update、关系及行变更触发器。
- [工具用户身份验证](https://learn.microsoft.com/en-us/microsoft-copilot-studio/configure-enduser-authentication)：连接身份与渠道支持限制。
- [Dataverse 安全角色](https://learn.microsoft.com/en-us/power-platform/admin/database-security)：数据权限、作者角色与管理角色的区别。
- [Dataverse 审计](https://learn.microsoft.com/en-us/power-platform/admin/manage-dataverse-auditing)：启用层级、容量、延迟与审计范围。
- [Copilot Studio 访问与许可](https://learn.microsoft.com/en-us/microsoft-copilot-studio/requirements-licensing-subscriptions)：trial、独立订阅及 Dataverse for Teams 差异。

相关：[[40-调研报告/企业IT与数字化/README]] · [[40-调研报告/企业IT与数字化/2026-09-22-Copilot-Studio网页读取与自定义解析选型]]。
