---
title: Centrus 与 Palantir：三亿美元节省与公众号技术细节核查
date: 2026-09-12
tags: [事实核查, Palantir, Ontology]
---

# 后续证据更新

同日进一步取得 Centrus 官方 LinkedIn 视频页 HTML 中 `application/ld+json` 的完整 `transcript`。Patrick Brown 在 AIPCon 9 确实公开了：项目进度有时滞后8周、连接站点/离心机/任务、缺陷根因分析、人员重新分配、人工批准后更新排班及薪资相关系统，以及轴承质量失败后换供应商、展示例子中货运时间从3周缩到1周。人工审批和动作追溯因此有直接公开依据，不能笼统归为作者推演。

这仍不支持公众号的全部字段/函数清单、四家监管报表、30%经理工时、90%数据精度、跨密级零延迟等说法，也没有提供3亿美元逐项测算或净收益。详见 [[2026-09-12-Palantir企业转型的实际做法与收益核算]]。下文保留首次核查的材料范围和结论，遇到重叠以本更新为准。

# 首次核查结论

合作和“识别近3亿美元潜在节省”有官方依据；不能理解为已经省下3亿美元。公众号将真实新闻扩写成详细实施复盘，其中大量量化指标、实现机制、决策过程在本次找到的官方材料中没有依据，不宜作为客户案例或架构设计证据。

## 核查清单

- **合作真实**：Centrus 于2026年3月12日公告与 Palantir 合作，使用 Foundry 与 AIP，覆盖工程、制造执行、供应链、项目控制和监管合规，并提及整合涉密与非涉密环境的系统、构建统一 Ontology。
- **近3亿美元是潜在节省**：原文为“identified nearly $300 million in potential cost savings and efficiencies”。是合作双方披露的识别结果，不等于经独立审计的实际节省。公告没有提供计算明细、实施成本或净收益核算。
- **60天不是公告精确口径**：公告称合作于1月下旬开始，3月12日发布成果；没有声称准确60天。应保留原始时间表述。
- **30亿美元未获此次原始公告支持**：公告仅写“multi-billion-dollar expansion”，未给出精确30亿美元。不可据此反推金额。
- **“把扩建交给 Palantir”容易误导**：Palantir 提供软件支持，公告明确另有 Fluor 的 EPC 合作，不能表述为 Palantir 承接全部工程建设。
- **30%经理时间、18个月前置期、60—90天合规周期、90%+数据精度**：本次找到的官方公告与检索结果未提供这些指标的出处。未找到依据不等于已证明造假，但不能当已验证事实。
- **4家机构自动派生4套报表、6类动作、具体函数和字段、所有建议都有可审计推理链、无物理复制、跨密级无延迟**：本次材料未提供这些 Centrus 专属实现细节；更像作者架构推演，不应标成已发生的项目事实。

## 技术风险

公众号将“密级作为 Property”“Ontology 打通”与物理隔离对立，并称跨密级决策必选前者，这种指导不可靠。

Palantir 官方 CBAC 文档将分级控制定义为强制访问控制，说明其不是默认启用、配置需要 Palantir 参与，且可能关联平台外的安全许可流程。不能将其等同于添加一个普通业务属性。

美国空军研究实验室的跨域方案说明强调网络与数据流隔离、强制访问控制、传输过滤和授权流程。语义模型整合不等于取消安全域边界。本文不能证明 Centrus 具体使用了哪种跨域方案，也不能从新闻公告反推出部署拓扑。

## 可安全引用的版本

> Centrus 与 Palantir 于2026年3月12日宣布合作。双方称，自1月下旬启动以来，已识别近3亿美元的潜在成本节省与效率改善机会。项目使用 Foundry、AIP 和统一 Ontology 支持铀浓缩产能扩建，但官方公告未公布节省的实际兑现情况及详细实施架构。

## 信源与边界

1. [待核公众号原文](https://mp.weixin.qq.com/s/ALhju3Yngg9xnyvzFmcRrw)：已抓取正文；图片未逐张核验。
2. [Centrus 官方联合公告，2026-03-12](https://www.centrusenergy.com/news/centrus-partners-with-palantir-to-drive-cost-savings-and-unlock-operational-efficiencies-in-major-expansion-of-u-s-uranium-enrichment-capacity/)：本次事实核查主要原始依据。合作方自述不是独立绩效审计。
3. [Palantir：Classification-based Access Controls](https://palantir.com/docs/foundry/security/classification-based-access-controls/)：用于核查产品权限机制，不代表 Centrus 的实际配置。
4. [AFRL：Cross-Domain Solutions 101](https://afresearchlab.com/cross-domain-solutions-101/)：用于解释跨域安全边界，不代表 Centrus 部署情况。

本次未完整核验 AIPCon 9 视频，也未获得项目内部设计文件、节省测算明细或独立审计报告。“无依据”限于本次已取得材料，不声称穷尽全部公开披露。
