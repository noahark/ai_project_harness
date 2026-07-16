# Stage Intake And Complexity

## User Discussion Summary

这是 Fable5 裁决里的 **Stage A**（`81-harness-design-rootcause-review-fable5.md` Q4）。
用户在 `funding_hedging` 的 `2026-07-docs-truth-sync-v1` stage 里，亲历了 6 轮 review
螺旋，其中最后合并时 `validate-stage.py --phase pre-accept` 红在两条**无法真实变绿**的
断言上：
1. `review_1.diff_fingerprint` 与 head 对不上——因为 review-1 被用户授权豁免（waive），
   指纹停在旧版（RC4「假红门」）；
2. `review_2.verdict` 是 REWORK——但内容已确认干净，只剩 bookkeeper 账本项。

validator 目前**没有任何字段能表达"这条断言被用户授权跳过、证据在此"**，也**没有分任务
指纹**的概念，所以只能靠人工「知情覆盖」。Stage A 就是给 validator 补上这个能力。

决定分两个 stage（用户 2026-07-17 拍板）：**A（本 stage）= 独立、小 diff、聚焦证据完整性**；
**B+ = 文档门 + 索引生成 + Harness 说明书文字**，B+ 复用 A 建的例外槽。

## 为什么在模板仓做

`scripts/validate-stage.py` 与 `schemas/` 是 `harness_owned`（模板仓拥有，向下 cp 到各
项目）。所以 Stage A 在**本模板仓**开发、评审、接受；再按两仓纪律**手动 cp** 到
`funding_hedging`，在其下一个 stage 实测。

## Classification

- Complexity: `MEDIUM`
- Direction panel required: `false`
- Existing synthesis covers this work: `true`（Fable5 裁决 `81` 已定方向）
- User approved lightweight route: `true`
- Lightweight skip allowed: `true`

## Rationale

- 方向已由 Fable5 独立裁决覆盖（RC4 修法 (a) 分任务指纹 +(b) 授权例外槽，复用
  `validate-stage.py:735-737` 现有 evidence-file 模式；R1 铁律：任何新 pre-accept 断言
  必须自带例外槽）。等价既有 synthesis，跳过 direction panel。
- **但触及"证据完整性主干"（指纹/validator 核心），故 review-2 必须严审、小 diff 聚焦**。
  Fable5 明确要求本 stage 独立、聚焦。

## Human Gates

- Gate: 触及 fingerprint 契约核心，review-2（终审）必须严格；不得放松。
- Gate: 接受后**手动 cp 到 funding_hedging** 需用户执行/确认（两仓纪律）。
- Gate: 本 stage 自身仍由**现行（改动前）validator** 把关（见 `00-task.md` Bootstrapping）。

## Routing Decision

- Next node: `stage-design`（先落 `10-design.md` 把设计决策定死，再进实现）

## Bookkeeper

- Provider/model/session: Anthropic / Claude Opus 4.8 / Claude Code
- Independent from implementers: `true`（实现拟派 Claude-GLM，未开始）
- If not independent, disclosure: n/a

## Parallel Mode

- Uses `docs/parallel-development-mode.md`: `false`（单实现者）
- R10 dispatch tail required: `false`
- R4 diff reconciliation required: `false`

## Evaluator

- Provider: Anthropic
- Model: Claude Opus 4.8（bookkeeper 直接分类，MEDIUM lightweight）
- Skill: complexity_evaluator

当前 Session ID: unavailable (Claude Code session id 未暴露)
Session ID 来源: unavailable
原始输出路径: reports/agent-runs/2026-07-harness-authorized-exception-v1/00-intake.md
本地北京时间: 2026-07-17 01:02:53 CST
下一步模型: human（审阅设计决策）→ 之后派 Claude-GLM 实现
下一步任务: 审阅 `00-task.md` 的 3 个设计决策；据此定 `10-design.md`
