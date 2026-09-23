---
title: 知识库操作日志
created: 2026-04-21
updated: 2026-09-09
type: meta
---

# 📜 知识库操作日志

> Append-only。所有wiki治理操作必须记录。
> 格式：`## [YYYY-MM-DD] action | subject`
> Actions: `init` `ingest` `update` `lint` `archive` `delete` `restructure`
> 当本文件超过500条，rotate为 `log-YYYY.md`，新建空log。

---

## [2026-06-09] weekly maintenance | Phase 1-3 — WARN全清零

**审计成果对比**:
| 指标 | Before | After | 改善 |
|------|--------|-------|------|
| CRITICAL | 0 | 0 | ✅ 持平 |
| WARN | 13 | 0 | ✅ -13 (100%清零) |
| INFO | 23 | 17 | ✅ -6 |

**核心动作**:
1. ✅ **野生tag清理** — 修复3个文件的野生tag:
   - `40-调研报告/绘画风格/2026-06-主流绘画风与涂鸦风设计token.md` — 7野生tag → `[research, multimodal]`
   - `40-调研报告/agent/2026-06-01-LangGraph实操入门.md` — 移除 `#tutorial`
   
2. ✅ **Frontmatter规范化**:
   - 补齐 `created/updated/type` 字段
   - 修正非法 `status: done` → `stable`
   - 修正非法 `type: architecture` → `research`

3. ✅ **index.md补全** — 添加5个缺失条目:
   - `20-项目/海马体/决策记录.md`
   - `40-调研报告/agent/2026-05-多Agent跨设备互联方案调研.md`
   - `40-调研报告/agent/2026-06-AI-Agent优质信息源盘点.md`
   - `40-调研报告/绘画风格/2026-06-主流绘画风与涂鸦风设计token.md`
   - `40-调研报告/迁移方案/PnP-PowerShell知识图谱与实战.md`

**治理效果**: 达成全绿状态 — 0 CRITICAL, 0 WARN, 仅剩17个INFO级超长页提醒(均为PRD/调研报告,属结构性长文不拆分)

**下周重点**: 继续内容丰富,优先补充核心方法论空骨架文件

---

## [2026-06-02] weekly maintenance | Phase 1-3 — 内容丰富 + INFO优化

**审计成果对比**:
| 指标 | Before | After | 改善 |
|------|--------|-------|------|
| CRITICAL | 0 | 0 | ✅ 持平 |
| WARN | 0 | 0 | ✅ 持平 |
| INFO | 46 | 9 | ✅ -37 (80%消减) |

**核心动作**:
1. ✅ **Phase 2: 内容丰富** — 补充3个空骨架方法论文件 28.4KB:
   - `03-平台化框架(DevUI_OTel_多语言)` 8.2KB — 平台化三大支柱、2026框架对比、落地清单
   - `07-AI Dev产品化(CLI_GUI_Cloud)` 10.0KB — CLI/GUI/Cloud三形态、工具排名、12-Factor CLI
   - `08-Observability与Evals(可观测_评测)` 10.3KB — 观测vs评测、2026平台、埋点标准、CI门禁
2. ✅ **Phase 3: oversized豁免** — 5个经典方法论深度文章加`oversized_ok: true`:
   - 02-异步消息与事件驱动 (251行)
   - 03-平台化框架 (232行)
   - 07-AI Dev产品化 (260行)
   - 08-Observability与Evals (316行)
   - 10-经典方法论 (347行)

**治理效果**:
- 知识库总页数: 203 (稳定)
- Frontmatter覆盖: 100% (持平)
- INFO级超长页: 46 → 9 (-37, 80%消减)
- 剩余9个oversized全部为海马体PRD/设计/测试文档(结构性长文,下次批量豁免)

**下周重点**:
- 补充剩余2个空骨架文件: `05-Handoff与Triage`、`09-HITL与Guardrails`
- 海马体项目9个PRD/设计文档批量加oversized豁免
- 继续内容丰富: AI Wiki 6大章节更新2026最新进展

**Git记录**:
- `a6511b0` Phase 2内容丰富 (2026-06-02 19:50)
- `b9e8190` Phase 3 oversized豁免 (2026-06-02 19:52)

---

## [2026-05-05] weekly maintenance | Automated by cron — COMPLETE

**审计结果**: CRITICAL:0, WARN:10 → 0, INFO:53 → 49 ✅ **全绿达成**

**修复动作**:
- 房产调研文件缺字段：补齐 created/updated/type 字段，野生tag归并为规范tag  
- 孤儿页面收编：AI风口调研 → 项目/Hermes分类，英语学习方案 → 新增家庭教育分类
- index.md同步更新：新增"👪家庭教育"分类，收编2个孤儿页面
- 审计脚本更新：补齐缺失的 family/realestate 标签到 VALID_TAGS

**治理效果**: 
- ✅ 10个WARN问题全部修复
- ✅ 2个孤儿页面成功入库  
- ✅ 7个野生标签全部规范化
- ✅ 知识库达到全绿状态 (仅49个INFO级超长页提醒)
- ✅ 143个页面，100% frontmatter覆盖

**下周重点**: 内容质量提升 — 重点检查🔴标记的空洞文件

---

## [2026-04-29] weekly maintenance | Automated by cron

**审计结果**: CRITICAL:0 → 0, WARN:4 → 0, INFO:63 → 40

**问题修复**:
- 🔧 **野生标签清理**: 修复 `40-调研报告/2026高考志愿全行业评估.md` 的4个野生标签 (`#高考`, `#志愿填报`, `#行业分析`, `#张雪峰视角`) → 规范化为 `[research]`
- 📄 **索引更新**: 在 `index.md` 补充14个缺失条目:
  - Pi-Rover项目: 12个架构模式+硬件+软件栈文档
  - Dreaming项目: 1个研究报告
  - 调研报告: 1个高考志愿评估报告
- ✅ **全绿状态**: WARN级问题从4个清零，CRITICAL级问题保持0

**剩余事项**: 40个INFO级超长页（>200行），建议考虑添加 `oversized_ok: true` 豁免标记，因为多数为技术深度文档和调研报告，拆分会影响阅读体验。

---

## [2026-04-21] init | 知识库治理体系建立

按 Karpathy LLM Wiki 模式为 obsidian-vault 建立治理骨架：

**新增文件**：
- `SCHEMA.md` — 治理宪法，定义tag taxonomy、frontmatter规范、页面阈值规则
- `index.md` — 75页全分类索引，每页一句话摘要
- `log.md` — 本文件，操作日志

**治理范围**：
- ✅ 受Schema约束：`10-知识库/`、`20-项目/`、`40-调研报告/`
- ⏭️ 自由区：`00-收件箱/`、`50-日报与动态/`、`90-治理/`

**当前健康度**：
- 总页数：75
- Frontmatter覆盖：62/75 (82%)
- 超长页(>200行)：16个，其中海马体PRD达2085行（待拆）
- Tag体系：基本空白，需要按SCHEMA重新打tag

**下一步（第2波待执行）**：
1. 拆分海马体PRD（2085行 → 5-8个子页）
2. 给13个无frontmatter页面补元数据
3. 建立第一批跨页 `[[wikilinks]]`

**第3波**：升级 kb-maintenance / research-to-kb / dreaming skill 强制走Schema。

---

## [2026-04-21] restructure | 第2波：拆PRD + 补frontmatter + 注入wikilinks

### 拆分海马体PRD（2085行 → 13个分卷）
原 `20-项目/海马体/项目需求文档(PRD).md` 改写为索引页（46行）。
新建 `20-项目/海马体/PRD分卷/` 目录，13个分卷文件：
- prd-01-目的与痛点 (23行)
- prd-02-用户与开源形态 (31行)
- prd-03-部署形式 (19行)
- prd-04-1-核心记忆操作 (67行) — F1
- prd-04-2-协议与接入 (136行) — F6-F8
- prd-04-3-生命周期管理 (104行) — F9-F10
- prd-04-4-隔离与共享 (303行) — F11-F16 🔴
- prd-04-5-安全与智能 (240行) — F17-F20 🔴
- prd-04-6-运维与集成 (310行) — F21-F25 🔴
- prd-04-7-Dogfood迁移 (110行) — F26-F27
- prd-05-操作流程与架构 (93行)
- prd-06-环境与里程碑 (163行)
- prd-07-附录 (474行) 🔴

### 补frontmatter（9个无元数据页面）
- 20-项目/Hermes/memory-system-upgrade.md
- 20-项目/海马体/{开发进度,访问凭证,架构审查v0.2,审查日志-v0.3-2026-04-20,竞品调研与商业计划书,F5-Dream设计-v0.1}.md
- 40-调研报告/{AI-Agent-Memory架构借鉴分析,Hermes上下文管理优化方案}.md

### 注入跨页wikilinks（6处保守注入）
- 调研↔项目互链：AI-Agent-Memory架构借鉴分析、Hermes上下文管理优化方案、竞品调研、F5-Dream设计、开发进度、架构审查v0.2

### 健康度变化
| 指标 | 第1波后 | 第2波后 |
|---|---|---|
| 总页数 | 75 | **88** (+13 PRD分卷) |
| Frontmatter覆盖 | 82% | **100%** ✅ |
| 超长页(>200行) | 16 | 待重新审计 |

### 重写 index.md
反映新的88页结构与100% FM覆盖。

---

## [2026-04-21] tool-release | wiki_lint.py 上线 | 治理体系第3波完成
- 新工具：`~/.hermes/skills/note-taking/kb-maintenance/scripts/wiki_lint.py`（~330行Python，零依赖）
- 检查项：frontmatter完整性 / tag taxonomy合规 / 坏链 / 孤儿页 / 超长页 / stale / index对账
- 退出码：0=clean / 1=warnings / 2=errors（可串CI/cron）
- Skill升级：kb-maintenance + research-to-kb 均已注入Schema约束段落
  - kb-maintenance：审计流程改为一键跑lint，标签体系更新为17-tag taxonomy
  - research-to-kb：入库子Agent context 强制要求frontmatter规范+更新index+追加log+lint自检
- 首跑基线：88页 / 100% frontmatter覆盖 / 0 CRITICAL / 463 WARN（66坏链、205野生tag、183缺字段、其余孤儿页/超长页）
- 这些 WARN 是真实存量债务，留待后续按主题分批清理

## [2026-04-21] mass-cleanup | 第4波债务清理 | 451 WARN已修复
**工具**: 新增 `~/.hermes/skills/note-taking/kb-maintenance/scripts/wiki_fix.py`
**模式**: dry-run → apply → re-lint → 幂等验证

### 治理数据 BEFORE → AFTER
| 指标 | BEFORE | AFTER | Δ |
|---|---|---|---|
| CRITICAL | 0 | 0 | — |
| WARN     | 463 | **7** | -451 (-97.4%) |
| 坏链      | 66  | **0** | 全清 |
| 野生tag   | 205 | **0** | 全清 |
| 缺字段    | 183 | **0** | 全清 |
| 非法status | 2  | **0** | 全清 |
| 超长页    | 7   | 7  | 留待拆文波 |

### 4A 坏链修复（66处）
- 13× [[SQL Server]] → 删除（旧笔记目标已不在治理区）
- 7× [[Model_Context_Protocol 规范解析]] → [[10-知识库/AI模型与Agent/03-工具调用与上下文协议/3.2-Model_Context_Protocol规范解析.md]]
- 7× [[AI_Agent 核心心智模型]] → [[10-知识库/AI模型与Agent/04-智能体架构与工作流设计/4.1-AI_Agent核心心智模型.md]]
- 7× [[工作流编排模式]] → [[10-知识库/AI模型与Agent/04-智能体架构与工作流设计/4.2-工作流编排模式.md]]
- 4× [[1.1-LLM基础与模型选型]] → [[10-知识库/AI模型与Agent/01-基础架构与模型底座/1.1-大模型演进与主流架构体系.md]]
- 其余按映射表批量替换

### 4B 标签归一（205处 → 17-tag taxonomy）
- 大小写归一: #AI/#LLM → #llm | #BrickHub → #brickhub | #Agent → #agent
- 分类归并: #Cursor/#Copilot/#Devin/#OpenHands/#GeminiCLI/#Windsurf → #vibe-coding
- 厂商映射: #OpenAI/#Google/#Meta/#xAI/#DeepSeek/#Qwen/#Kimi → #llm
- 语义归并: #MemGPT/#长期记忆/#向量数据库 → #memory | #ReAct/#TDD → #methodology
- 章节中文tag: #基础架构与模型底座 等 → 删除（章节本身就是目录）
- 杂项: #daily/#qa/#dreaming/#sql/#database → 删除
- Schema扩展: 新增 #meta（README/index/说明文档专用）

### 4C frontmatter补字段（62文件 / 183字段）
- created: 62处（git log --diff-filter=A 取首次提交日期）
- updated: 58处（git log -1 取最近提交日期）
- type:    62处（按目录推断: 10-/AI模型→concept, 30-/→research, 20-/→entity）
- status:  60处（30-调研→stable, 其余→draft）
- tags:    1处

### 验证闭环
- ✅ wiki_fix.py 幂等性测试通过（第二次执行0变更）
- ✅ wiki_lint.py 复跑确认 451 WARN 已清
- ✅ 抽查 1.1-大模型演进 / Memory架构借鉴分析 frontmatter 符合规范

### 留待清理
- 7处 oversized 长文需拆分（独立的"拆长文波"，工作量大）
- 55处 orphan + 17处 not_in_index 是 INFO 级，不影响主体质量


## 2026-04-21 第5波：拆长文 + lint修缺陷 + 索引补全

**拆长文（7个超长页）**
- 3.2-MCP规范解析 (520→376) → 拆出 3.2.1 / 3.2.2
- 4.2-工作流编排模式 (551→367) → 拆出 4.2.1 / 4.2.2
- 6.3-SWE-Agent (484→272) → 拆出 6.3.1 生产化与CICD
- DL.AI学习路径 (641→114) → 拆出 按主题索引 / 全量目录
- prd-07-附录 (500→43) → 拆出 附录A/B/C
- 测试方案与用例 (433→144) → 拆出 D1-D5 / D6-D8 / 报告与缺陷追踪
- 2.3-结构化输出 (425→133) → 拆出 厂商对比 / 工程实战
- 共生成 **15 个子页**，每页带回链 + 父页留 stub

**lint工具修复2个隐性bug**
- 修 `Path("2.3.1-x").stem == "2.3"` 误判 → 加 `link_stem()` 工具函数（误报97%孤儿页）
- 入链来源补全：根级 `index.md/log.md/README.md` 也算入链来源

**lint增强**
- 加 `oversized_ok: true` frontmatter豁免（用于Schema/参考文档结构性长文）

**索引/孤儿补全**
- index.md 新增"📑 拆分子页索引"块（15子页）
- 创建 20-项目/Hermes/README.md，挂入 memory-system-upgrade

**最终状态**
- CRITICAL=0, WARN=0, INFO=29 (均为 200-400 行"略长但合理"档)
- vault总页数：88 → 103
- 100% frontmatter / 100% tag合规 / 0坏链 / 0孤儿


## 2026-04-21 调研产出
- 40-调研报告/AI-Agent个人盈利赛道扫描-2026Q2.md (新增)
- 9个赛道完整评分 + 三层组合策略

## [2026-04-22] research-ingest | McKinsey-2026-AI报告与5岁AI启蒙 | 整合 McKinsey 2026 三份核心报告（State of Organizations/State of AI/MGI Agents-Robots-Us）+ 儿童AI教育研究，输出AI现状+5岁娃三层启蒙路径+12月节奏，~12KB。

## [2026-04-22] weekly-maintenance | 知识库维护 | 修复4个CRITICAL问题（缺frontmatter）+ 1个WARN（invalid type）+ 6个新文档加入index.md。丰富AI知识库内容：大模型演进补充Claude 4.5/GPT-5/Gemini 3.0，MCP更新2026生态数据，终端IDE新增Continue/Claude Code，Agent架构/评测基准/推理策略补充最新进展。621行新增，107行删除。

## [2026-04-23] lint-fix | 拆长文+加豁免 | 处理wiki_lint 5个oversized WARN：海马体两文档加 oversized_ok: true 豁免（架构方案-v0.4 437行 / F5-软删除-v0.3 424行，结构性长文不拆）；三篇知识库综述按H2拆分：2.2-高阶推理策略 534→329行（剥离§10-13到 2.2.1-推理模型工程化进阶 237行）、3.2-MCP规范 424→173行（剥离协议架构+传输层到 3.2.3-MCP协议架构与传输层 274行）、4.1-Agent心智模型 453→253行（剥离2026新进展+主流框架到 4.1.1-Agent认知架构与主流框架2026 232行）。同步更新index.md添加3个新子页索引。WARN结构性oversized 5→0。

## [2026-05-05] research-ingest | 怡海花园真实成交价调研 | 房天下网签数据，3 个分园近期成交单价区间 3.5~4.6 万/㎡，链家参考价偏高 10~20%。放 00-收件箱/。

## [2026-05-05] schema-update | 新增 #realestate / #family tags + 40-调研报告/房产/ 子分类，迁入怡海花园调研。
## [2026-05-28] index | full rebuild — 341 pages across 28 sections
[2026-05-28] structure | rename 10-个人/ → 05-个人/ (resolve numbering collision with 10-知识库)

## [2026-06-02] weekly maintenance | Phase 2+3 内容丰富 + 问题修复

**审计结果**: CRITICAL:0, WARN:1 → 0, INFO:11 → 10

**Phase 2: 内容丰富 (3个核心方法论文件)**:
- `01-工作流编排（Graphs & Workflows）.md` (114字 → 7.2KB) — 补充DAG/状态机/事件驱动三种编排模式对比、LangGraph/CrewAI/Claude SDK 2026框架排名、工程落地清单（状态管理/失败恢复/成本控制/可观测性）
- `02-异步消息与事件驱动（Event-driven Messaging）.md` (122字 → 9.8KB) — 补充EDA核心概念、同步vs异步决策表、Kafka/RabbitMQ框架对比、工程落地清单（幂等性/DLQ/背压/可观测性）、银行反欺诈案例
- `10-经典方法论（ReAct_Reflexion_ToT_MetaGPT）.md` (123字 → 12.8KB) — 补充四大方法论（ReAct/Reflexion/ToT/MetaGPT）核心概念、论文链接、代码示例、HumanEval实际提升数据、工程落地清单（选型/成本控制/监控）

**Phase 3: 问题修复**:
- WARN: `2026-05-10-deeplearningai-update.md` (554行) 加 `oversized_ok: true` 豁免（结构性长文）
- INFO: `20-项目/BrickHub/dreaming/2026-05-28-research.md` 补充到 index.md，消除孤儿+未登记问题

**治理效果**:
- ✅ 3个空洞文件丰富为实质内容（共29.8KB新增）
- ✅ 1个WARN问题修复（oversized豁免）
- ✅ 1个孤儿页面入库

**Git记录**: commit c2e8b48

---

## 2026-06-09 ingest — Karpathy Sequoia Ascent 2026

**触发**: 老王要求分析 https://karpathy.bearblog.dev/sequoia-ascent-2026 并入库

**操作**:
- 新建 `40-调研报告/AI洞察/2026-04-Karpathy-Sequoia-Ascent-Software3.0.md`（research/stable）— 12 节论点链 + 稀缺性转移大图景 + 对老王 KB/skill/班子体系映射
- 新建 `40-调研报告/AI洞察/2026-盈利线索-可验证RL环境wedge.md`（research/draft）— 第⑥节衍生盈利线索，三筛选标准 + 候选领域
- index.md 新增「💡 调研 · AI洞察」区块（2 条）
- 修正：初稿误放 `30-调研报告/`（企业IT迁移专区），已迁至 `40-调研报告/`（AI调研主区）

**信源**: bearblog 反爬，走 jina reader 取全文（含 Karpathy 用 Codex 5.5 生成的 summary+transcript）

**待决策**: 可验证 RL 环境 wedge 是否做"验证器可行性 spike"（企业IT迁移校验为优先候选）

---

## [2026-08-25] weekly-maintenance | MCP 2026-07-28 与 Coding Agent CI/CD 生产化刷新

- `3.2-Model_Context_Protocol规范解析.md`：升级到 MCP 2026-07-28；补充无状态请求、Streamable HTTP、MRTR、订阅、OAuth resource binding、step-up scope、token passthrough 与 SSRF 防护；删除无法稳定复核的生态硬数字。
- `6.3.1-SWE-Agent生产化与CICD.md`：按生成/验证/发布分权重写；补充无特权 PR CI、`pull_request_target` 风险、OIDC 部署、patch 策略门禁、企业评测指标与审计清单；删除未验证榜单和固定模型推荐。
- 修复 `50-日报与动态/AI日报/2026-04-09` 至 `2026-04-13` 的 10 处过时 WikiLink，指向现有 Claude Code 与 MCP 页面。

---

## [2026-09-01] weekly-maintenance | LangGraph/MCP 时效刷新与审计器纠偏

- `4.2.1-LangGraph深度实操.md`：纠正 HITL 为 `interrupt()` + `Command(resume=...)`；补重放幂等纪律与 subgraph `None/True/False` 持久化模型。
- `3.2.3-MCP协议架构与传输层.md`：对齐 MCP `2026-07-28`，补 per-request metadata、双时代版本回退、Streamable HTTP 网关/幂等/安全基线。
- `90-治理/audit_kb.py`：忽略 fenced/inline code 和 append-only 历史映射，分列全库与 Schema 治理区结果。
- 复验：全库 686 篇；治理区 281 篇 Frontmatter 缺口 0；死链 0；真实骨架 0。README 无新增、重命名、归档页面，本周无需更新索引。

---

## [2026-09-07] ingest | Attention Is All You Need 深度解读

- 新增 `1.1.1-Attention-Is-All-You-Need深度解读.md`：基于 arXiv v7 原文、独立 Evidence Auditor 与 Red Team，拆解 QKV、自注意力并行优势、GPT decoder-only 继承关系、二次复杂度和解释性边界。
- 引用链通过 strict evidence gate；分析门禁 92/100，3 个承重 claim，原文摘要占比 10.17%。
- 同步更新 AI模型与Agent README 与全库 index。

## [2026-09-07] audit | 全机体检与目录治理

- 新增 [[90-治理/2026-09-07-全机体检与整理.md]]：服务器资源正常；主站HTTP500、海马体异地备份停滞、cron错误和旧域名待处理。
- 已做在线本地热备及副本完整性验证；workspace新增README/PROJECTS台账，现有生产路径和未提交代码保留。
- 修5个skill YAML描述，正文不变；修订运维安全流程，未删除业务文件、改DNS或重跑cron。
- 完整本地证据归档：`~/work/system-audit/2026-09-07/`。

## [2026-09-08] ingest | 十只持仓综合研究

- 新增 [[40-调研报告/投资与量化/账户研究/2026-09-08-十只持仓综合研究.md]]：核四股2026半年报、六ETF中报股票表、历史量价和条件化建议，status=draft保留待确认的风险预算与买入逻辑。
- 独立核验6份ETF共481条股票记录、370个不同代码；历史10只持仓加沪深300共6469根日线、33组收益窗口与45个持仓相关性配对。历史行情截止9月7日，未混入盘中K。
- 引用账本、逐字证据、CSV和复算脚本在 `~/work/portfolio-research-20260908/`；同步README与index，不提交或推送Git，不修改交易规则。

## [2026-09-08] ingest | 股票交易接入技术方案

- 新增 [[40-调研报告/投资与量化/账户研究/2026-09-08-股票交易接入技术方案.md]]，比较券商条件单、认可API和RPA；推荐研究草案与人审、风控、执行分权。
- 核中信CATS平台说明、普通交易条件单条款、迅投官方API及上交所程序化细则正文；个人账户准入与深交所适用细则留待券商确认，不编资金门槛。
- 附单文件手机HTML；证据在 `~/work/trading-integration-20260908/`。仅技术探索，无账户连接、交易软件安装、实盘委托或Git推送。

## [2026-09-08] ingest | 全账户条件单与仓位整理草案

- 新增 [[40-调研报告/投资与量化/账户研究/2026-09-08-全账户条件单与仓位整理草案.md]]，按用户纠正的“截图只是功能展示、需要整体方案”覆盖全部10只持仓。
- 10:22公开行情及已确认份额计算建议额度与约80%权益仓位；现金沿用截图，明确无交易/资金变化假设，不当作实时券商资产。
- 未设置、提交或启用条件单，未修改持仓/监控。原始方案测算在 `~/work/portfolio-plan-20260908/`。

## [2026-09-08] ingest | 赣东北五日亲子交互行程

- 新增 [[05-个人/旅行/2026-赣东北/2026-09-22-赣东北五日亲子行程.md]] 及单文件HTML：地图、五日卡片、每日住宿餐饮、雨天/疲劳替代、儿童票、租还车与返程缓冲。
- Chrome离线验证五日主/雨线、地图、键盘和清单通过；390px无页面溢出，运行异常0、外部运行请求0；96条引文存在性检查通过。
- 原携程行程只读；未嵌协作令牌、未预订或公开部署。完整源码与证据：`~/work/jiangxi-family-trip-20260922/`。

## [2026-09-08] lint/update | 每周知识库维护：MCP 文档纠错

- 只读扫描 288 篇治理页：缺 Frontmatter 与 linter 报告坏链均为 0；INFO 共 305 条、涉及 275 个去重路径。全库非隐藏 Markdown 709 篇，按短文加占位规则未发现空洞候选；自由区无头文件不自动套用治理 Schema。
- 更新 [[10-知识库/AI模型与Agent/03-工具调用与上下文协议/3.2.1-MCP-Server生态系统.md]] 与 [[10-知识库/AI模型与Agent/03-工具调用与上下文协议/3.2.2-MCP-SDK开发实战.md]]：官方来源核验归档/包名/SDK v2，移除无来源统计与危险或不可运行示例，补选型、权限和两个业务场景。
- 实测 Python/TS SDK 的 19 项检查通过；文内代码与实测文件一致，小测试另行运行通过；未连接外部业务系统。Phase 2 复检 CRITICAL/WARN 为 0。
- 审计快照、原文件备份与测试证据：`~/.hermes/workspaces/kb-weekly-2026-09-08/`。原有未提交内容保留，Git 仅选择本轮文档和本条追加日志。

## [2026-09-08] update | 每周知识库维护：索引与锚点修复

- [[10-知识库/AI模型与Agent/README.md]] 补齐 11 个现存章节入口，覆盖 33 篇文章；明确旧目录名映射，没有新建骨架。
- [[10-知识库/README.md]] 修复两处 Tag Taxonomy 标题锚点；[[40-调研报告/商业与行业/Palantir-FDE/07-移植到个人咨询.md]] 修复一处缺点号的章节锚点，保留 intentional stub。
- 复验治理页 576 处非代码 WikiLink，按同目录/vault 路径、stem、alias、heading 解析后无未解析候选；未声称全库外链可用。核心 linter 仍为 CRITICAL/WARN=0，INFO=301。
- 独立按 SCHEMA 解析 YAML 发现核心 linter 漏检：59 页空/非列表 tags、1 页缺 status、5 次未注册 tag 使用（合计 65 条、62 个去重文件）。未批量改元数据或 taxonomy；这些是残留治理债务，不以脚本零 WARN 当健康证明。
- Phase 2 已推送并核 SHA：`c3e65173f26284b541f585757fceeccdb68d3b66`。本阶段仅提交上述索引/锚点文件和本条日志；审计基线不变。


## 2026-09-09 · 全库目录重组

按用户批准方案迁移433个文件，重建主题与章节入口；仅修复内链/已有updated，不删除正文，不Git提交/推送。详见 [[90-治理/迁移记录/2026-09-09-目录重组/README]]。

## [2026-09-17] ingest | 官方MCP服务与M365 Connector选型

- 新增 [[40-调研报告/企业IT与数字化/2026-09-17-官方MCP服务与M365-Connector选型]]，按14组服务比较官方托管、OAuth、业务读写与账号边界。
- 官方原文与引文验证通过；Google个人Gmail预览申请限制、国内MCP认证差异、M365联邦Connector只读边界均单独说明。证据保存在 `~/work/mcp-provider-survey-20260917/`。
- 仅新增报告、更新最近专题README与本日志；未安装/授权业务服务、未重试暂停部署、未提交或推送Git。


## [2026-09-18] ingest | Azure SQL MCP审查报告

- 新增 [[40-调研报告/企业IT与数字化/2026-09-18-Azure-SQL-MCP代码审查与M365接入评估]]，更新最近专题README。
- 汇总用户项目静态审查、本地编译/协议/安全测试与微软官方接入核验；明确测试替身与真实云端未验收边界。
- 原项目源码未改，未创建Azure资源、未创建Connector；未commit/push。详细原始证据保留任务工作区。

- 2026-09-21：新增 WeKnora 官方资料调研，区分 RAG/Agent/Wiki、IM 与长期记忆边界及 main/release MCP 差异；更新 AI技术与产品专题索引。未安装、未迁移、未提交推送。

- 2026-09-21：WeKnora报告补充RAGFlow、FastGPT、Dify、AnythingLLM官方定位对标及许可注意事项；未部署、未提交推送。

- 2026-09-21：新增RAGFlow能力与本地部署评估，核v0.27.2部署组件、本地模型、硬件与内网边界，更新专题索引；只读调研，未部署、未提交推送。

- 2026-09-22：新增《飞书文档MCP与Copilot-Connector实现选型》，核对 azsql-mcp-demo 最终实验与飞书个人托管MCP下线范围，建议自建MCP直接封装REST；更新企业IT专题索引。仅调研，未部署或调用飞书业务写入，未提交Git。

- 2026-09-22：新增《Copilot Studio网页读取与自定义解析选型》，核验HTTP、Custom Connector内联C#、Flow、外部REST/MCP、浏览器自动化及Code Interpreter网络边界；更新企业IT索引，未部署或修改用户租户。

- 2026-09-22：补充Copilot Studio网页解析报告中的Azure Functions部署归属、跨租户鉴权/网络/DLP边界与客户交付建议；未修改Azure资源。


## 2026-09-22 — 每周知识库定向维护

- 全库枚举807个Markdown，受治理271页；AI专题35篇正文均非空洞，本轮不扩写、不制造待办。
- 为6篇既有稿补Frontmatter，保留正文；历史状态与待核验边界在metadata中标明。英语时态速查表创建日期未知、当日未提交新稿均暂留。
- 40-调研报告/README补4篇既有根目录报告入口；不移动正文，不修改首页或taxonomy。
- 审计与备份：`~/.hermes/workspaces/kb-weekly-2026-09-22/`；残留元数据问题见工作区报告，不据此宣称全库健康。


## 2026-09-23 — Azure App Service MCP 托管核查

新增《2026-09-23-Azure-App-Service托管现有MCP可行性》，核官方托管文档与SQL/飞书项目；区分协议无状态与SQLite/flock业务状态，更新企业IT专题索引。仅调研归档，未改代码、未创建Azure资源、未部署或提交Git。
