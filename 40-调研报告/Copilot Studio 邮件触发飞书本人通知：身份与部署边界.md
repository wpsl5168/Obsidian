---
tags: [Copilot-Studio, Power-Automate, 飞书, 身份集成]
status: 官方文档核验完成，客户环境未实测
---

# Copilot Studio 邮件触发飞书本人通知：身份与部署边界

## 结论

将“谁的邮箱收到邮件”“谁有权限调用飞书”“通知发给谁”分别建模。推荐由飞书应用机器人统一发送，接收者由可信的邮箱所属人身份确定。不需要为了给本人发通知，让每位员工使用自己的飞书身份发送消息。

共享一个 Copilot Studio Agent 给员工，并不自动为每位员工建立个人邮箱事件订阅。微软事件触发器文档明确：触发器使用 maker 的连接凭据；自主运行中需要认证的动作也应具备可用的 maker authentication，不能依赖现场用户登录。

## 推荐链路

邮箱到信 → 可信触发器/订阅确定 mailbox owner → 身份映射 → 可选 Agent 分类或摘要 → 固定接收者的发送步骤 → 飞书应用机器人私聊本人。

- 交互会话可以使用认证用户上下文，但邮箱后台事件不可假设存在正在聊天的用户，也不能把会话 User.ID 当作飞书 open_id。
- mailbox owner 必须取自连接绑定、受控配置或 Graph subscription 映射，不从邮件正文推断，不直接取 To 的第一个地址。抄送、密送、邮件组、别名、共享邮箱都会使 To 与实际邮箱所属人不一致。
- 若使用 Office 365 Users 的 Get my profile，必须验证它与 Outlook 触发器连接的是同一个员工；它只代表该动作自身的连接账号。

## 接收者映射

### 快速验证：真实邮箱一致

飞书发送消息支持 receive_id_type=email。若 M365 邮箱地址与飞书可识别的用户真实邮箱一致，可直接使用邮箱，不必先查询 open_id。

示意请求（未在客户环境发送）：

```http
POST https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=email
Authorization: Bearer <tenant_access_token>
Content-Type: application/json; charset=utf-8
```

```json
{
  "receive_id": "employee@example.com",
  "msg_type": "text",
  "content": "{\"text\":\"你有一封新邮件\"}"
}
```

receive_id 在正式流程中必须绑定到可信的 mailbox owner 变量，不能固定成开发者邮箱，也不能开放给模型自由生成。content 是序列化 JSON 字符串，不是嵌套对象。

### 正式方案：稳定 ID 映射

受控映射记录：Entra tenant ID + object ID → 主邮箱/别名 → 飞书 tenant + app ID + open_id → 启用状态。

可由管理员导入，或由用户经双边认证完成绑定。映射表不能允许员工任意修改成其他人的接收者。open_id 与应用关联，换飞书应用不能直接复用。

注意：飞书 batch_get_id 支持通过用户邮箱或手机号查询 ID，但官方明确不支持企业邮箱字段 enterprise_email。不能因两个系统都显示“公司邮箱”就认定可匹配；需要抽样验证实际字段。映射失败、重复、离职时停止发送并记录告警，禁止猜测收件人。

## 部署候选

1. 每人一份 Power Automate 自动化 Flow，各自授权 Outlook；共享发送能力/飞书应用。最适合 PoC、少量试点和低代码优先，但全员连接授权、模板升级、凭据失效需要管理。共享 Flow 本身不会为所有使用者自动创建邮箱监听。
2. Microsoft Graph 邮箱订阅 + Azure Function/受控服务统一处理。适合公司级集中部署，可维护一个 Agent，不要求人人复制 Agent；代价是应用授权、邮箱访问范围控制、订阅续期、webhook 验证、重试与监控。Graph 支持 delegated 与 application 权限；集中订阅使用适当 application 权限并限制目标邮箱范围。仅提醒与读取正文所需权限不同，依实际载荷取最小权限。
3. 各人复制 Agent 并绑定各自触发器连接。可以按连接所属人建立路由，但存在多份 Agent 的配置和版本漂移，不推荐公司级推广。

若只是新邮件通知，无需 Agent 推理；仅在分类、判断重要性或摘要时加入 Agent。身份决定与发送授权留在确定性流程/服务端，避免邮件 prompt injection 改变收件人。

## Connector 封装建议

底层飞书通用发送接口需要 receive_id_type、receive_id、msg_type、content。对 Agent 暴露的业务动作应更窄，例如发送当前邮箱通知，由受信任上下文在后端解析收件人；不要给模型任意接收者的自由发送能力。

应用密钥、tenant_access_token 的获取/刷新/缓存由受控 Flow 或后端维护，不交给模型、不写入普通参数或提示词。共享机器人发消息权限不等于每位员工都应拿到机器人密钥。

## 上线检查与验收

- 飞书为开发者后台创建的应用机器人，不是只能发到固定群的自定义 webhook 机器人。
- 已开启机器人能力、发布应用，授予适当应用发消息权限，目标员工在可用范围内。
- Power Platform DLP 允许 Outlook 与自定义 Connector 组合；环境 maker credentials 策略与自主触发兼容。
- 核实 Connector / Flow / Agent 的实际许可证及消费计费，不将 Outlook Standard 推导为整条链路免费。
- 默认推送最小必要信息，敏感邮件不转正文；M365 链接仍需源端权限。
- A、B 两人分别收信，只通知各自；开发者不应收到两人的通知。
- 测试 To/CC/BCC、别名、群组、共享邮箱，以及映射缺失、离职、token 过期、重试和重复通知。
- 收件人不能被邮件正文或用户输入替换。

## 待确认

客户是在一个共享 Agent 上添加 When a new email arrives 触发器，还是计划每人复制并重新绑定连接？邮箱是否为 Exchange Online，飞书实际用户邮箱是否与其一致？这决定触发侧的具体配置。

## 官方信源

- [Copilot Studio 事件触发器：maker credentials 与自主运行限制](https://learn.microsoft.com/en-us/microsoft-copilot-studio/authoring-triggers-about)
- [控制 maker/end-user credentials](https://learn.microsoft.com/en-us/microsoft-copilot-studio/configure-no-maker-authentication)
- [Copilot Studio 用户认证与 User.ID](https://learn.microsoft.com/en-us/microsoft-copilot-studio/configuration-end-user-authentication)
- [Office 365 Outlook Connector](https://learn.microsoft.com/en-us/connectors/office365/)
- [Graph Outlook change notifications：权限与邮箱订阅](https://learn.microsoft.com/en-us/graph/outlook-change-notifications-overview)
- [飞书发送消息：email/open_id、token、可用范围](https://open.feishu.cn/document/server-docs/im-v1/message/create?lang=zh-CN)
- [飞书按邮箱/手机号查询 ID：enterprise_email 不支持](https://open.feishu.cn/document/server-docs/contact-v3/user/batch_get_id)
