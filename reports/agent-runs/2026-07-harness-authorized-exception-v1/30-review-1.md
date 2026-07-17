# Review-1(round-3)— Stage A rework-2 复审：provider 隔离权威化 + evidence 常规 blob 门

- reviewer: kimi（provider moonshot，model kimi-for-coding），与实现/修复者 claude_glm
  （zhipu_glm）provider 级隔离 ✓；本文件覆盖 round-2 版（round-1 版见 git @972fd06，
  round-2 版见 @636e92d 前树）
- 被审范围：`00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..8b650adfc4747ce6efe36ba7b78806d9b36023bf`
- rework-2 内容：review-2 round-2（Codex）4 P1 + 1 P3 的修复——代码 2 条（#1 task
  own-review provider 权威化、#2 evidence 常规 blob 门，claude_glm）、簿记 2 条
  （#3 watchdog 移出 stage diff、#4 旧 dispatch 作废，bookkeeper）、P3（AGENTS.md EOF 空行）
- 结论：**ACCEPT**（P0=0，P1=0，P2=0；P3=1 非阻塞观察项）

## 0. 自证落点 + 独立复算指纹（非转述 status）

- `git rev-parse --show-toplevel` = `/Users/ark/Desktop/ai code/ai_project_harness` ✓；
  `git branch --show-current` = `stage/2026-07-harness-authorized-exception-v1` ✓；工作树 clean。
- 复算（python 导入 `vs.compute_diff_fingerprint`）：
  `8b650adfc4747ce6efe36ba7b78806d9b36023bf:29127111d6f38bbe9aedc5b956c13cef632c97da5148cb90859ddce741c2e5f0`
  与 `status.diff_fingerprint` **逐字符一致**。
- 第二路（shell 直管）：`git diff --binary 00e25b4..8b650ad -- . ":(exclude)…/status.json" | shasum -a 256`
  → `29127111d6f38bbe9aedc5b956c13cef632c97da5148cb90859ddce741c2e5f0`，一致。
- 现场 `python3 scripts/validate-stage.py 2026-07-harness-authorized-exception-v1 --phase pre-review`
  → PASSED，exit=0，同一指纹（bootstrapping 门过）。

## 1. 逐条复核（全部经真实 validator 复跑/对抗构造验证）

### #1（P1）task own-review provider 权威化 — 通过

- 代码（`_task_own_review_covers` :1127 起）：reviewer 身份改取
  `provider_identity(review.get("provider"))`（provider 字段权威，不再 reviewer 标签优先）；
  task 无可解析 owner/implementer → FAIL（不再静默跳过）；`review.provider` 缺失/不可解析
  → FAIL；`reviewer` 为**已注册别名**且与 declared provider 冲突 → fail-closed；
  owner==declared → FAIL。
- 复跑（60 F 段 4 例 + 自构 2 例）：
  - G1：owner=claude_glm + review.provider=zhipu_glm + reviewer=任意未注册串 → FAIL
    （Codex 实测的假绿路径闭合；这正是 round-2 我降级为观察项的洞，Codex 判 P1 正确）。
  - G3：reviewer='kimi'(→moonshot_kimi) 冲突 declared 'codex'(→openai) → FAIL（报文明示
    冲突双方）。
  - G4：own-review 无 owner/implementer → FAIL。G5：仅 reviewer 标签、provider 空 → FAIL。
  - ADV2-alias-ok（自构）：reviewer='gpt' 与 provider='codex' 同厂一致 + 跨 provider owner
    + 合法指纹 → **PASS**——一致别名不误伤，冲突判定不是粗暴"有别名就杀"。
  - F1（回归）：合法跨 provider own-review 仍 PASS；ID（主门 review_2 同 provider）仍 FAIL。
- **主门零外溢**：全范围 diff 对 `review_provider_identity` / `validate_review_identity` /
  `PROVIDER_IDENTITIES` **零改动行**（grep 实证）——task 专用严格解析没有泄漏进主身份门，
  符合 fix spec 的"task-specific strict helper"路线。

### #2（P1）evidence 必须常规 blob — 通过

- 代码：新增 `_evidence_object_mode_and_type()`（:882，`git ls-tree <验证时HEAD> -- <path>`
  取 (mode, type)），`_validate_exception_evidence` 在读 blob **前**设门：不可 ls-tree →
  `not committed/tracked`；`mode ∉ {100644,100755}` 或 `type != blob` → FAIL 并打印实际
  mode/type。symlink 与常规文件同为 git type blob，用 mode 区分——实现与注释一致。
- 复跑（60 F 段 3 例 + 自构 2 例）：
  - E1：evidence=目录（mode 040000 type tree）+ digest 匹配 → FAIL（Codex 实测路径闭合）。
  - E2：committed symlink（120000/blob，仓内目标，隔离逃逸门）→ FAIL。
  - E3：committed submodule/gitlink（160000/commit）→ FAIL。
  - ADV2-exec（自构）：**可执行常规文件（100755）** + digest 匹配 → PASS——模式门不误伤
    合法常规文件。
  - ADV2-link-esc（自构）：symlink 指向仓外 /etc/passwd → FAIL（逃逸/模式双门之一即拦）。
  - 回归：F3-abs/F3-esc/F3-dig/F3-empty/用例 2/ADV-untracked 全仍 FAIL；合法 post-head
    tracked+digest 匹配（1/6a/F1）全仍 PASS。
- Fable5 方案保留：head 仍取**验证时** `git rev-parse HEAD`（:960），未改回
  blob@status.head_sha；digest 封存逻辑未动 ✓。

### #3（P1）watchdog 469 行移出 stage diff — 通过

- `git diff --name-only 00e25b4..8b650ad | grep watchdog` → 0；49141dd 精确删除 469 行
  （972fd06 的逆）。
- 用户工作**未丢**：`git log --all` 显示 d53abc5（同标题）在 **main** 上
  （`git branch --contains d53abc5` → main）——符合"模板/Runner 同步只落 main"。
- `status.changed_files` 已更正为 7 项（4 交付 + 3 stage 证据），net diff 中的非 stage
  簿记文件恰好=这 4 交付文件，无 watchdog ✓。

### #4（P1）旧 review-2 dispatch 绑定 round-1 head — 通过

- `26-dispatch-review-2-codex.md` 顶部已加 **SUPERSEDED/作废** 横幅（明示绑定的是 round-1
  旧 head `9414163`/旧指纹，不得据此审查；round-3 review-2 将另出绑定新 head 的专用包）。
- 新 review-2 包按 hard gate 须在 round-3 review-1 落账后由 bookkeeper 出——当前不存在
  是正确状态，非缺项。

### P3（AGENTS.md EOF 空行）— 通过

- 空行已删；`git diff --check 00e25b4..8b650ad` exit 0 ✓。

## 2. 测试证据独立复跑（proof by execution）

- 重建套件：E/E.1 归档 25 例 + round-2 自构 6 例 + rework-2 新增 7 例（G1/G3/G4/G5/E1/E2/E3，
  源码取自 60 F 段，含 prep 回调构造 in-repo symlink / file:// submodule）+ round-3 自构
  4 例（ADV2-alias-ok / ADV2-exec / ADV2-link-esc / ADV2-unreg-prov）。
- `py_compile` COMPILE_OK（Python 3.9.6）；**42/42 ALL CASES AS EXPECTED**，零回归。
- negative-list 守住：CL2（class-2 + 例外仍 FAIL）、ID（主门同 provider 仍 FAIL）；
  退化（单任务/tasks 缺省/模板默认）不变。

## 3. 禁改边界

- `compute_diff_fingerprint` / 主身份门 / 共享 `evidence_path_exists`：零改动行（grep 实证）✓。
- class-2 未收；RC1-3 未触；`funding_hedging` 0 文件；AGENTS.md 两仓 carve-out 未动 ✓。
- rework-2 commit（8b650ad）仅触 4 文件（validate-stage.py / AGENTS.md / 40-fix-report /
  60-test-output），在 37-dispatch 允许边界内 ✓。

## 4. 非阻塞观察项（不影响 ACCEPT）

- **P3（registry-hygiene 残留，pre-existing 语义）**：`review.provider` 写**未注册字符串**
  时按原样成为独立身份（ADV2-unreg-prov 探针实测 PASS：owner=claude_glm +
  provider="zhipu-clone-unregistered" 可通过隔离）。这与主身份门对未注册别名的语义一致
  （`PROVIDER_IDENTITIES.get(text, text)`），不在 rework-2 fix spec（"recognized alias
  conflict fail-closed"）范围内，但意味着 provider 字段的防伪最终靠 registry 治理 +
  评审人核。另：G4/G5 报错文案说 "recognized, non-empty provider"，实际接受未注册串——
  文案与语义的小错位。建议未来硬化时考虑「provider 须命中注册表」并同步文案，非本 stage 阻塞。
- 延续观察（round-2 P3，Codex 亦列为 residual）：同一 id/身份错误从多调用点重复打印
  （输出噪音，FAIL 方向正确，不进 applied）。

## 5. json_schema_valid 判定

本报告末尾 verdict JSON 已按 `schemas/review-verdict.schema.json` 用 python jsonschema
4.25.1（draft 2020-12）校验通过：required 全含、additionalProperties=false 无多键、
finding（P3）五要素齐、ACCEPT 无需 fix_start_prompt。判定：**true**。

## 6. verdict 依据汇总

- round-2 review-2 的 4 P1 + 1 P3 全部验证修复（代码 2 条复跑闭合，簿记 2 条 git 实证，
  P3 diff --check 干净）。
- 42/42 用例复跑全过（含 25 归档零回归 + 17 对抗）；pre-review 门现场 PASS；双路指纹
  复算与 status 逐字符一致。
- P0=0，P1=0，P2=0；P3=1（registry-hygiene 残留 + 文案错位，非阻塞）。

当前 Session ID: unavailable（Kimi CLI 会话内模型无法自省 provider-native Session ID；操作者可经 runner 补录 status.session_receipts）
Session ID 来源: unavailable（会话内无 runtime_env/hook_payload/cli_output 可查）
原始输出路径: reports/agent-runs/2026-07-harness-authorized-exception-v1/30-review-1.md
本地北京时间: 2026-07-17 17:52:24 CST
下一步模型: bookkeeper（Claude Opus 4.8）→ 提交本报告与 status 写回，出绑定新 head 的 round-3 review-2 dispatch 包
下一步任务: 记录 round-3 review-1 ACCEPT 证据，准备 round-3 review-2 dispatch（须绑定 8b650ad + 指纹 29127111…，含 Fable5 evidence 方案与 round-2 findings 上下文）

```json
{
  "schema_version": 1,
  "stage_id": "2026-07-harness-authorized-exception-v1",
  "role": "first_reviewer",
  "model": "kimi-for-coding",
  "verdict": "ACCEPT",
  "diff_fingerprint": "8b650adfc4747ce6efe36ba7b78806d9b36023bf:29127111d6f38bbe9aedc5b956c13cef632c97da5148cb90859ddce741c2e5f0",
  "reviewer_prior_involvement": "none",
  "reviewed_artifacts": [
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/50-review-2.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/37-dispatch-fix-2-claude-glm.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/40-fix-report.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/60-test-output.txt",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/26-dispatch-review-2-codex.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/status.json",
    "scripts/validate-stage.py",
    "AGENTS.md",
    "git diff 00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..8b650adfc4747ce6efe36ba7b78806d9b36023bf",
    "git log --all -- docs/planning/adapter-watchdog-runner.md (preserved on main as d53abc5)",
    "independent re-execution: 42/42 cases (25 archived E/E.1 + 6 round-2 ADV + 7 rework-2 G/E + 4 round-3 ADV2)",
    "dual-path diff-fingerprint recomputation (python compute_diff_fingerprint + raw git diff | shasum)",
    "git diff --check 00e25b4..8b650ad (exit 0)"
  ],
  "findings": [
    {
      "severity": "P3",
      "title": "Task own-review provider field accepts unregistered strings as self-declared identities (pre-existing registry-model semantics)",
      "file": "scripts/validate-stage.py",
      "line": 1137,
      "evidence": "provider_identity() maps unregistered text to itself, so review.provider='zhipu-clone-unregistered' with owner=claude_glm passes cross-provider isolation (ADV2-unreg-prov probe executed, PASS). The G4/G5 error text says 'recognized, non-empty provider' but unregistered strings are accepted. Same semantics as the global identity gate and outside the rework-2 fix spec (recognized-alias conflict fail-closed works: G3).",
      "impact": "A self-declared unregistered provider string can mask a same-vendor task review; the final defenses are registry hygiene and reviewer/human inspection, identical to the pre-existing main identity gate.",
      "recommendation": "Non-blocking; in a future hardening consider requiring provider values to resolve through the registry (or aligning the error wording), applied consistently to both the task own-review path and the global identity gate."
    }
  ],
  "required_fixes": [],
  "residual_risks": [
    "Human-origin authenticity of exception evidence remains a workflow obligation per the Fable5 Q2 ruling; evidence_sha256 prevents silent substitution but is not proof of human authorship.",
    "Unregistered provider/reviewer strings self-declare identity under the registry-based model (see P3); registry governance is the mitigation.",
    "Duplicate task error lines from multiple call sites remain non-blocking output noise (also noted by round-2 review-2 as residual)."
  ],
  "next_action": "continue"
}
```
