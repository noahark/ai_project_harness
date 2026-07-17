# Dispatch Packet — Stage A round-3 review-2（reviewer: codex，终审）

⚠️ **模板仓 `ai_project_harness`**,分支 `stage/2026-07-harness-authorized-exception-v1`。
这是**绑定当轮 head 的专用包**(取代已作废的 26-,即 review-2 round-2 finding-4 的正解)。
终审 rework-2 后的新状态。只审、不改。产出 `50-review-2.md`(覆盖)+ 写回 `status.review_2`。

- **base_sha** `00e25b4bb6e492ffbd974d1d7793ef10246fa6ad`
- **head_sha** `8b650adfc4747ce6efe36ba7b78806d9b36023bf`
- **status.diff_fingerprint**
  `8b650adfc4747ce6efe36ba7b78806d9b36023bf:29127111d6f38bbe9aedc5b956c13cef632c97da5148cb90859ddce741c2e5f0`
- 身份:你 openai;实现/修复者 claude_glm=zhipu_glm;round-3 review-1 kimi=moonshot。三方隔离。
- **rework_count=2/3(最后额度)**:ACCEPT 即收官;若再 REWORK 触顶 max_rework → 人工升级。

## 你 round-2 的 4 P1 + P3 处置(逐条复核是否真闭合)

- **#1(代码)** task own-review 隔离信 reviewer 标签 → 已改 **provider 权威**(claude_glm 修)。
- **#2(代码)** git tree/symlink 冒充 evidence → 已加 **常规 blob 门**(ls-tree mode)(claude_glm 修)。
- **#3(bookkeeper)** 无关看门狗混入 diff → 已迁 main `d53abc5` + 从 stage git-rm;
  `git diff --quiet 00e25b4..8b650ad -- docs/planning/adapter-watchdog-runner.md` 干净。
- **#4(bookkeeper)** 陈旧 dispatch → 26- 已标 SUPERSEDED;**本包(27-)绑当轮 head**。
- **P3** AGENTS.md EOF 空行 → 已删;`git diff --check` 干净。

你上轮已确认 round-1 修复成立、Fable5 的 `evidence_sha256`+验证时 HEAD 方案可接受
(**不要再要求 blob@status.head_sha**)。

---

## PROMPT BODY

你是 Stage A 的 round-3 review-2(codex,证据完整性主干终审)。rework-2 后复审。只审、不改。
产出 `50-review-2.md`(覆盖)+ 写回 `status.review_2`。

第0步(自证落点,不对就停):
  git rev-parse --show-toplevel   # 必须以 /ai_project_harness 结尾
  git branch --show-current        # stage/2026-07-harness-authorized-exception-v1

被审范围 base..head:
  00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..8b650adfc4747ce6efe36ba7b78806d9b36023bf
须复算指纹 ==:
  8b650adfc4747ce6efe36ba7b78806d9b36023bf:29127111d6f38bbe9aedc5b956c13cef632c97da5148cb90859ddce741c2e5f0

先读:reports/agent-runs/2026-07-harness-authorized-exception-v1/ 下 50-review-2.md(你 round-2
的 4 P1+P3 原文)、40-fix-report.md(rework-1+2 修法映射)、30-review-1.md(Kimi round-3 ACCEPT)、
56-direction-ruling-finding3-fable5.md、60-test-output.txt(全段)、10-design.md §2/§2.1。

看 diff + 独立复算指纹(python vs.compute_diff_fingerprint,须 == 上面的值)。

终审焦点(逐条裁断你自己 round-2 提的问题是否真闭合;可构造对抗复跑真实 validator):
  1. #1 provider 权威隔离:owner=claude_glm + review.provider=zhipu_glm,**reviewer 任意未注册串**
     → FAIL;缺 owner/implementer provider 或缺 review.provider → FAIL;reviewer 已注册别名与
     provider 冲突 → FAIL;**一致别名(如 reviewer=gpt/provider=codex)+ 跨 provider owner → 不误杀**;
     合法跨 provider → PASS。确认**全局 review_provider_identity / validate_review_identity /
     身份分离 :717-718 未被动**(task-local,不外溢主门)。
  2. #2 evidence 常规 blob 门:目录(tree)/ symlink(mode 120000)/ submodule(160000)→ FAIL;
     常规 blob(100644/100755)+ 匹配 digest → PASS。确认保留 evidence_sha256 + 验证时 HEAD、
     未动共享 evidence_path_exists、未改回 blob@status.head_sha。
  3. #3 边界:看门狗确不在 base..head;git diff --check 干净。
  4. 全量回归:round-1/round-2 已闭合的所有 P1/P2(task 例外横幅、own-review 指纹绑定、canonical
     scope、唯一 id、evidence 五条、reason/at、钉指纹、fail-closed、退化、class-2/身份 negative-list、
     模板默认不破 pre-accept)仍成立。
  5. 边界:compute_diff_fingerprint 一字未动;无 RC1-3;未碰 funding_hedging;AGENTS carve-out 未动。
  6. **有没有再新的相邻边界洞**(这是最后额度,请一次性挖到底,别留到下一轮)。

bootstrapping:改后脚本对本 stage --phase pre-review 仍 PASS、零回归。

产出 50-review-2.md:结构化结论、P0/P1/P2(file:line+失败场景)、复算指纹。末尾附符合
schemas/review-verdict.schema.json 的 JSON verdict(带 --output-schema 则自动约束,否则结束前
python jsonschema 自校验;必填 11 字段;REWORK 时带 fix_start_prompt)。写回 status.review_2:
reviewer=codex, provider=openai, model=<你的模型>, verdict, json_schema_valid,
diff_fingerprint=<复算值>, primary_provider=codex, reviewer_prior_involvement="none", open_p0_p1。
不改被审代码。footer + provider-native Session ID(或 unavailable+原因)。

## END PROMPT BODY
