---
title: BrickHub Dream B 整合收口 — 2026-07-03
created: '2026-07-03'
updated: '2026-09-22'
type: entity
tags: [brickhub, workflow]
status: archived
created_basis: 正文日期；历史整合记录，本轮未重新核验项目状态
maintenance_note: 2026-09-22仅补Frontmatter，正文保持原样；updated不代表事实或外部来源已重新核验。
---

# BrickHub Dream B 整合收口 — 2026-07-03

## 背景
7-2 老王授权方案 A(整合而非逐条合)。整合分支 `dreamB/consolidated-20260702` 本地就绪(2 commits,Visual Check 过)但**卡在"等你一句合"未落地**。本次(7-3)老王「全部处理」→ 核实真实状态后执行收尾。

## 守卫结果(合并前,全绿才动手)
| 守卫 | 结果 | 说明 |
|---|---|---|
| lint | ✔ No ESLint warnings or errors | — |
| test | 106 passed / 12 failed | **零回归**:切 main 基线对比同为 12 failed(freeBuildTray 10 + officialLdraw 2),均为既有环境性失败(styled-jsx babel 编译 + official LDraw 素材缺失),非本次引入 |
| build | ✓ Compiled successfully, exit 0 | **无 WASM fallback 警告**,native swc(14.2.33)生效 — 验证 7-1 止血成果 |

## 合并落地
- 方式:`git merge --ff-only`(保留 2 语义 commit,健壮性 vs 3D 分离便于回溯)
- `origin/main`:`8fc5a742` → **`fe0ae5c3`**(远程 fetch 二次确认 SHA 一致)
- 2 commits:
  - `ad44ca00` fix(robustness):整合 12 条 PR 真实健壮性修复(去重)
  - `fe0ae5c3` fix(3D/BrickRenderer):SandboxAnimator unmount 清理 userData + camera 简化

### 整合覆盖的 7 文件真实修复(非抹奶油)
- `dslCompiler.js`:extractDSL 类型防护 / NaN 校验重排(登记 brickStateMap 前拦截,防 relative_to 引用 NaN 污染)/ color 非法值回退 15
- `pipeline/llm.js`:reader.releaseLock 移入 finally,防流未关资源泄漏
- `pipeline/index.js`:error.msg 空值防护(String 包裹)/ themes 空数组防护
- `pipeline/utils.js`:deduplicateLDraw/extractLDraw 的 `code ?? ''` 空值防护
- `knowledgeBase.js`:删重复 `'house'` keyword / category_path 空数组防护
- `BrickRenderer.js`:SandboxAnimator unmount 清理 userData(防内存泄漏)/ camera 三元冗余简化 / cleanLDrawCode 冗余变量消除
- `FreeBuildTray.js`:`onSelectDef?.()` → `onSelectDef()`(核实 sandbox.js 必传该 prop,去可选链安全)

## PR 队列处置
起点 19 OPEN → **8 OPEN**(close 11)。

### 已 CLOSE(11 条,core 修复已等价进 main + 死代码部分本就该丢)
`#12 #13 #14 #15 #16 #17 #18 #34 #38 #40 #41`
带 comment 指向 ad44ca00/fe0ae5c3;这些 PR 额外触碰的 `components/home/*` 为死代码(不再拉起)。

### 剩余 8 条 — 待老王拍板
| PR | 状态 | 规模 | 性质 | 建议 |
|---|---|---|---|---|
| **#42** | DIRTY | PartsTray | **真 bug**:`scrollbarWidth:'thick'`→`'none'`。当前 main 是 `'auto'`(仍不对,应配 scrollbar-hide 用 none) | rebase 后合 |
| **#21** | DIRTY | PartsTray | **真 bug**:`border-3`(Tailwind 非标准刻度,渲染失效) | rebase 后合 |
| #37 | DIRTY | +25/-9, 2文件 | pages a11y(model/[id].js + index.js 触控/aria) | rebase 后合 |
| #28 | DIRTY | +31/-8, 2文件 | pages/index.js a11y | 与 #37 同域,可合并处理 |
| #29 | DIRTY | +36/-12, 3文件 | pipeline/utils(已覆盖)+ pages/index.js + 死代码 | utils 已进 main,仅取 pages 部分或 close |
| #20 | DIRTY | +4/-2, 1文件 | model/[id].js 小改 | 与 #37 重叠,建议 close |
| #19 | DIRTY | +46/-18, 6文件 | sandbox.js a11y + core(core 已覆盖) | 仅取 sandbox 部分或 close |
| **#5** | DIRTY | +674/-105, 1文件 | 首页像素级复刻(独立大 PR,copilot 分支) | 需独立评估,非 Dream B 产物 |

**推荐路径**:#42/#21(PartsTray 真bug)+ #37/#28(pages a11y)= 4 条真价值,rebase main 后逐条合;#20/#29/#19 与已合内容重叠,建议 close;#5 单独评估。

---

## 二次收尾(2026-07-03 续,老王「持续推荐推进」授权)

延续方案 A:PR 都 DIRTY(基于旧 main,行号已漂),不 rebase,改为**读意图→当前 main 等价 apply→守卫→close**。

### 等价 apply(main 32d838b2)
| 来源 PR | apply 内容 | 备注 |
|---|---|---|
| #42 | PartsTray `scrollbarWidth: 'auto'→'none'` | main 现状是 auto,真 bug,已修 |
| #37 | model/[id]: 关闭钮 w-6→w-11、上下步 w-9→w-11(触控44px红线)、返回主页 min-h-11 | aria 部分 main 早已合,只补触控尺寸 |
| #28 | index: AI工坊 button type/aria-label、搜索框 type=search、难度条内联三元→IIFE(等价健壮) | 搜索框 aria-label 早已合;KnowledgePreviewPanel a11y 早已在 main |
| **bug类** | model/[id] 的 `border-3`→`border-2` 共 **6 处**(Tailwind 无此刻度,渲染失效) | #37 原 diff 只改 2 处,顺手修完整 bug 类 |

### 已在 main、直接 close(无需 apply)
- #21:PartsTray border-3→border-2 + 删冗余 font-bold — main 第56行早已是正确态

### 守卫(合并前)
- lint ✔ No ESLint warnings or errors
- build ✓ Compiled successfully, exit 0, **无 WASM fallback**(native swc 生效)
- 未碰 3D,纯 CSS/a11y,无需 Visual Check

### 落地
- commit `32d838b2`(3 files, +34/-14)→ push origin/main(远程 fetch 二次确认)

### close 处置(7 条,带精确 comment)
- 已 apply:#42 #37 #28
- 已在 main:#21
- 重叠/被覆盖:#20(触控被覆盖)、#29(utils 已进 main + pages 被覆盖 + 死代码)、#19(core 已进 main + sandbox 重叠 + 死代码 BrickScene)

### 队列终态
**19 → 1 OPEN**(今天累计 close 18 条)。**唯一剩 #5**。

### #5 首页像素级复刻 — 留待老王拍板(不替合)
- 作者 `app/copilot-swe-agent`(非 Dream B),2026-04-27 创建已 2+ 月
- +674/-105,**整体重写 pages/index.js**(内联 9 个 SVG 图标组件 + 全新 header/hero/LegoBar 布局 + BrickPatternSvg 背景)
- **触老王铁律**:「测试页先调满意→再迁移主页,禁未确认就改正式页」+「HermesPet 改码规矩同款」
- 且与本次刚合的 index.js(button/search/难度条)**全面冲突**,UNKNOWN
- **建议**:老王先本地起 dev 看 #5 视觉效果,满意再决定"整体替换 index.js"还是"提取局部元素";在此之前不合。要不要我起本地 dev 截图给你看?

### 本次二段总结
- 又清 7 条,队列 19→1,只剩需你亲自看效果的 #5
- **关键方法**:DIRTY PR 不 rebase(会带回已合重复改动),改为 main 等价 apply;apply 前必逐处核 main 现状(发现 #21/aria/KnowledgePreviewPanel 多处早已合,避免重复劳动)
- 顺手修完 border-3 bug 类(6处),符合「修整个 bug 类」原则


## 本地清理
- 删整合分支 `dreamB/consolidated-20260702`(已并入 main)
- 删 11 条已 close PR 对应本地分支
- 保留 7 条 OPEN PR 活分支 + 2 无主老分支(20260503/20260523,无害)
- `dream_b_rotation.json` 已重置(04-24→07-01 轮完整一圈,全核心文件覆盖)

## 关键决策沉淀
- **"整合而非逐条合"是对的**:12 条 PR 每天基于旧 main 生成,重复携带同一 knowledgeBase 修复,逐条合会疯狂冲突;去重整合成 2 commit 干净落地。
- **零回归判定必须切 main 基线对比**,不能只看整合分支的 X failed 数字——12 failed 是既有环境失败,非回归。
