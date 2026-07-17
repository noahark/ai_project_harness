# Handoff

This file is the stage's active handoff context. A new terminal session reads
it together with `status.json` and only the needed workflow section.

## Recovery Header

- Active phase: `review_1`（rework-2 fix 已交付+提交,等 pre-review + 派 round-3 review-1 Kimi）
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
- rework-2 fix 已交付(claude_glm)+ bookkeeper R4:全局身份主门 4 函数逐字未动、两个 round-2
  probe 独测均 fail-closed、合法跨 provider 不误杀、看门狗排除、diff --check 干净。
- Next action: **操作者派 Kimi 执行 round-3 review-1**(被审新 head `8b650ad`;复算指纹须 ==
  `8b650ad…:29127111…`;重点复核 #1 provider 权威隔离 + #2 tree/symlink/submodule 拒 + 看门狗
  确已不在 diff + 零回归)。产出 30-review-1.md(覆盖)+ 写回 status.review_1。之后**绑新 head
  的 round-3 review-2(Codex,另出专用包)**。**rework_count=2/3,最后额度**;fix 作者禁为 Codex。
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
- Status: `planned`
- Branch: `stage/2026-07-harness-authorized-exception-v1`（模板仓，base `00e25b4`）
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
本地北京时间: 2026-07-17 16:45:00 CST
下一步模型: kimi（round-3 review-1,经操作者派发）
下一步任务: 复审新 head 8b650ad(#1 provider 隔离 + #2 blob 类型 + 看门狗排除 + 零回归),复算指纹须 == 8b650ad…:29127111…,写 30-review-1
