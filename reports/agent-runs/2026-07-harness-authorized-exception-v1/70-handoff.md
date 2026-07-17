# Handoff

This file is the stage's active handoff context. A new terminal session reads
it together with `status.json` and only the needed workflow section.

## Recovery Header

- Active phase: `accepted`（终态）。2026-07-17 由直修弧 2026-07-red-gate-greening-v1 T0 收口：
  status 与既有合并事实对齐（合并提交 `9ea36ab` no-ff、记账提交 `cdef1ee`、下行 cp 到
  funding_hedging `e71164c`→`9d28ec4`）。**用户接受原文在已提交记录中查无，证据缺口已
  如实登记于 `status.user_acceptance.verbatim_evidence_gap`，未补造。**
- 双评审 round-3 均 ACCEPT：review-1 Kimi ACCEPT、review-2 Codex ACCEPT(P0/P1/P2=0,P3=1
  非阻塞已处置)。pre-accept 门 PASSED(真绿,无 override/无例外),head 8b650ad。
- 历史记录（合并前快照,保留不改写）:⛔ 未经用户明确接受,禁止合并 main。接受后:合并 main → 手动
  cp harness_owned(validate-stage.py + _template/status.json + AGENTS.md + docs/harness-design.md)
  到 funding_hedging → 下个 stage 实测转绿两历史红门(bookticker class-1、docs-truth-sync
  维持 user-override)。〔进展注记:合并与 cp 已完成;bookticker 转绿由直修弧 T1+T3 承接,
  docs-truth-sync 维持 known_red(class-2, D-i pending)。〕
- **新 head `8b650adfc4747ce6efe36ba7b78806d9b36023bf`；指纹 `8b650ad…:29127111…`**(rework-2 后
  重算;看门狗已排除、diff --check 干净)。round-2 被审 head 是 c82fc2b,见 review_rounds[1]。
- **rework_count=2/3**。round-2 review-2(Codex/gpt-5)=REWORK。4 P1:①task own-review 隔离信
  reviewer 标签先于 provider(假绿自审)②git tree 冒充 evidence(非 blob 绕过)③(bookkeeper 错)
  无关的 adapter-watchdog-runner.md(469行)混进 base..head 且不在 changed_files ④(bookkeeper 错)
  26-dispatch 仍绑 round-1 旧 head。P3:AGENTS.md EOF 空行。均已 bookkeeper 核实成立。
  Codex 确认 round-1 修复本身成立、evidence_sha256 方案可接受(不再要 blob@head_sha)。
- 分工:①②→claude_glm 代码(37-dispatch-fix-2);③④→bookkeeper 已办:③看门狗迁 main
  `d53abc5` + 从 stage git-rm(base..head 已无净 diff);④26-dispatch 已加 SUPERSEDED 横幅,
  round-3 另出绑新 head 的专用包。
- round-3 review-1(Kimi/moonshot)=ACCEPT(P0/P1/P2=0,P3=1;双路指纹一致;#1/#2 闭合、主门零外溢)。
- Next action: **操作者派 Codex 执行 round-3 review-2**(终审,专用包 `27-dispatch-review-2-round3-codex.md`
  绑 head `8b650ad`)。ACCEPT → bookkeeper pre-accept 门认证 → 用户接受 → 合并 main → cp 到
  funding_hedging → 下个 stage 实测转绿两历史红门。**rework_count=2/3:再 REWORK 触顶 max_rework
  → 人工升级**;fix 作者禁为 Codex。
- Repo: **模板仓** `ai_project_harness`（不是 funding_hedging）。validate-stage.py +
  schemas 是 harness_owned，在此开发；接受后手动 cp 到 funding_hedging。
- **新 head `c82fc2b05299e1edc1ad972b12e12d8597cfb394`；diff_fingerprint `c82fc2b…:f3d3fb33…`**
  (rework-1 fix + R4 修正后重算;旧 9414163 是 round-1 被审指纹,见 review_rounds[0])。
- review-1(Kimi/moonshot)=ACCEPT;**review-2(Codex/openai/gpt-5)=REWORK**,rework_count=1/3。
  50-review-2.md 已由 bookkeeper 从 codex read-only 会话逐字落盘;verdict JSON 合 schema、
  含 fix_start_prompt。
- **4 条 P1(bookkeeper 已逐条核代码,全部成立)**:①task 例外静默放行(违反 D2 非静默)
  ②任意非空串冒充 task own-review(D3 假绿)③授权来源未认证/未钉 head_sha,文档"模型不能
  自授"表述不成立 ④task scope 歧义(task:<id> 与裸 <id>)+id 不唯一。P2:reason/at 未校验、
  模板 tasks[] 注解键。
- **finding-3 已 Fable5 DIRECTION-SET(56 文件)**:A 档务实硬化——①仓内相对+防逃逸(删 :819
  绝对分支)②git-tracked 读提交 blob ③记录加 `evidence_sha256`、validator 重算 blob 比对
  (**不用 blob@head_sha,会与 post-head 授权死锁**)④task 按 assertion_id 过滤。Q2 文档必改
  (防静默+强制人核两层,人核是流程义务)。不做 B(receipt 依赖 DRAFT 基建)。10-design §2/§2.1/§7
  已更。拆分:user_authorizations 结构化 + runner receipt → 后续 stage。
- round-2 review-1(Kimi/moonshot)=ACCEPT(P0/P1/P2=0,P3=1 非阻塞;双路复算指纹一致)。
- Next action: **操作者派 Codex 执行 round-2 review-2**(终审,被审新 head `c82fc2b`)。
  **⚠️ 必带前置说明:finding-3 采用 `evidence_sha256`-in-record + 读提交 blob(Fable5 56-裁决),
  不采用 Codex 原文的 blob@`status.head_sha`——后者与 post-head 授权死锁。该偏离是方向权威批准的,
  Codex 应评估 evidence_sha256 是否达成同等防篡改,不得再要求 blob@head_sha。** 产出 50-review-2.md
  (覆盖 round-1)+ 写回 status.review_2。ACCEPT → pre-accept 门 → 用户接受。fix 作者禁为 Codex。
- 设计已定(Fable5 D1/D2/D3,见 `10-design.md`)：D1 白名单源码枚举、v1 只收 class-1、
  negative list 五项不可豁免;D2 钉指纹自动失效 + authorizer=="user" 硬化 + 非静默;
  D3 链式+前缀、单任务退化。
- ⚠️ 最危险点:降级逻辑只能对 `review.diff_fingerprint` 单条、且经例外全合规后降级;
  绝不降级 verdict/身份/指纹重算(negative list)。
- Bootstrapping：改后脚本对本 stage 自己跑仍需正常;现有行为零回归。
- 顺带目标（接受+cp 后做,不在本 stage）：bookticker 真转绿(class-1);docs-truth-sync
  维持 user-acceptance-override(其 review_2 是 class-2,不在 v1)。
- Read-set: = `status.current_inputs`
- Open blockers: None
- Do-not-read: `reports/agent-runs/**/history/**`, other stages

## Current State

- Stage: `2026-07-harness-authorized-exception-v1`（Fable5 的 Stage A）
- Status: `accepted`（已合并模板仓 main `9ea36ab`，记账 `cdef1ee`；T0 收口 2026-07-17）
- Branch: `stage/2026-07-harness-authorized-exception-v1`（已合并，保留）
- Complexity: MEDIUM，lightweight route（方向 Fable5 覆盖）

## Open Decisions（用户）

- **D1**：例外槽白名单 vs 任意断言（倾向白名单）。
- **D2**：`authorized_exceptions` 字段形状 + 仅 human 授权 + evidence_file 必须存在。
- **D3**：分任务指纹做多深（倾向轻量：覆盖并集校验）。

## Next Steps

1. 用户拍板 D1/D2/D3。
2. bookkeeper 出 `10-design.md`（把三决策定死）+ `11-adr.md`。
3. 派 Claude-GLM 实现（validator + schema + _template + 说明书段落）。
4. review-1 Kimi → review-2 Codex（终审严审证据完整性主干）。
5. 接受 → 手动 cp 到 funding_hedging → 下个 stage 实测 → 转绿两历史红门。
6. 之后开 **Stage B+**（文档门 + 索引生成 + Harness 说明书文字，复用例外槽）。

当前 Session ID: unavailable (Claude Code session id 未暴露)
Session ID 来源: unavailable
原始输出路径: reports/agent-runs/2026-07-harness-authorized-exception-v1/70-handoff.md
本地北京时间: 2026-07-17 23:40:00 CST
下一步模型: 无（stage 终态;账本由直修弧 red-gate-greening-v1 T0 收口）
下一步任务: 无后续;本 stage 已 accepted + merged,后续转绿工作在 red-gate-greening-v1
