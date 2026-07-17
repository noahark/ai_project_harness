# Review-1(round-2)— Stage A rework-1 复审：RC4 review-2 四条 P1 + 两条 P2 修复

- reviewer: kimi（provider moonshot，model kimi-for-coding），与实现/修复者 claude_glm
  （zhipu_glm）provider 级隔离 ✓；round-1 同身份 ACCEPT 已由本文件覆盖（历史见
  `30-review-1.md@972fd06`，git 历史保留）
- 被审范围：`00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..c82fc2b05299e1edc1ad972b12e12d8597cfb394`
- 被审交付文件：`scripts/validate-stage.py`、`AGENTS.md`、`docs/harness-design.md`、
  `reports/agent-runs/_template/status.json`（rework commit c82fc2b 另含本 stage 的
  40-fix-report.md / 60-test-output.txt 证据文件）
- 结论：**ACCEPT**（P0=0，P1=0，P2=0；P3=1 非阻塞观察项）

## 0. 自证落点 + 独立复算指纹（非转述 status）

- `git rev-parse --show-toplevel` = `/Users/ark/Desktop/ai code/ai_project_harness`（以
  `/ai_project_harness` 结尾 ✓；dispatch 误从 funding_hedging 启动，已切到正确仓）；
  `git branch --show-current` = `stage/2026-07-harness-authorized-exception-v1` ✓；
  工作树 clean。
- 复算（python 导入 `vs.compute_diff_fingerprint(root, sd, base, head)`）：
  `c82fc2b05299e1edc1ad972b12e12d8597cfb394:f3d3fb338fd5983b864c858826ea20748ddb49076b69ae3061b3a6467f7ce9d2`
  与 `status.diff_fingerprint` **逐字符一致**。
- 第二路独立复算（shell 直管，不经 Python helper）：
  `git diff --binary 00e25b4..c82fc2b -- . ":(exclude)reports/agent-runs/2026-07-harness-authorized-exception-v1/status.json" | shasum -a 256`
  → `f3d3fb338fd5983b864c858826ea20748ddb49076b69ae3061b3a6467f7ce9d2`，与记录一致。
  两路互证，helper 未被做手脚。
- 现场跑 `python3 scripts/validate-stage.py 2026-07-harness-authorized-exception-v1 --phase pre-review`
  → `STAGE VALIDATION PASSED`，exit=0，打印同一指纹（bootstrapping 自指门过）。

## 1. 逐条复核结论（均经真实 validator 复跑/对抗构造验证）

### 1) finding-1：task 例外不再静默 — 通过

- `validate_task_coverage`(:1121) 签名改为 `-> (errors, applied)`，实际依赖的 task 例外
  在 :1223-1229 以 `{assertion_id, scope}` 追加进 applied。
- `_merge_applied`(:1280) 按 `(assertion_id, scope)` 集合去重、保首次出现序；上游
  `_task_id_errors` 已保证 task id 唯一、review scope 互异，去重是正确兜底，无误删。
- main(:1330-1334) 合并 acc_applied+cov_applied，:1352-1358 横幅打印全部 applied。
- 复跑：F1（仅靠 `task:B` 例外 PASS）→ 横幅
  `PASS (1 authorized exceptions applied: review_fingerprint_trails_status@task:B)`；
  6a → `PASS (2 ... review_fingerprint_trails_status@review_1, ...@task:C)`，N=2 含 task scope。

### 2) finding-2：task own-review 完整性 — 通过

`_task_own_review_covers`(:1060) 逐项要求：具名 `reviewer`/`provider`/`model` 非空、
`verdict=="ACCEPT"`、`json_schema_valid` truthy、与 task owner/implementer 跨 provider、
`diff_fingerprint == 重算的 task 指纹`。复跑 + 自构对抗：

- F2：伪造 `{"review_1":{"diff_fingerprint":"x"}}` → FAIL
  （`task B review_1.reviewer must be a non-empty string`）。
- ADV-rescue（自构）：伪造 review **同时配一条合法 `task:B` 例外** → 仍 FAIL
  （:1210-1214 own_err 直进 errors 且 `continue`，例外够不到）——不完整对象不能被
  task 例外救回 ✓。
- ADV-fp（自构）：reviewer/provider/model/ACCEPT/schema 全合规但指纹乱写 → FAIL，报错
  明确给出 `recorded=x, expected=<重算值>`——指纹钉定真实生效 ✓。
- ADV-prov（自构，二次修正后）：reviewer 用已注册别名 `glm52`(→zhipu_glm) 与 owner
  `claude_glm`(→zhipu_glm) 同 provider → FAIL（validate_tasks 两处 + own-review 一处
  三报错）——跨 provider 要求真实生效 ✓。
- F1 task A：完整合规跨 provider own review（codex vs claude_glm，指纹==重算）→ 覆盖
  PASS，证明合法路径未被误伤 ✓。

### 3) finding-3：授权来源五条机械硬化 — 通过

`_validate_exception_evidence`(:882) 五条逐一对账 + 复跑：

| 条 | 实现 | 对抗复跑 |
|---|---|---|
| 拒绝对路径 | :887-889 | F3-abs `/etc/hosts` → FAIL |
| 逃逸防护 | :890-894 resolve 后须在 root 内 | F3-esc `../../../../etc/passwd` → FAIL |
| 必须 git-tracked（读提交 blob，不读工作树） | `_evidence_committed_blob`(:861) `git show <验证时HEAD>:<path>` | 用例 2 MISSING.md → `not committed/tracked`；ADV-untracked（自构：工作树存在但未 tracked，直调函数绕过 clean-worktree 门）→ 同样 FAIL，证明是 blob 读取本身拦截 |
| blob 非空 | :901 | F3-empty（空 waiver 已提交）→ FAIL |
| digest 封存 | :904-912 重算 sha256 比对 `evidence_sha256` | F3-dig（digest 全 0）→ FAIL，报 expected 真实值 |

- **无死锁**：:960-963 取的是**验证时 HEAD**（`git rev-parse HEAD`，try/except→`""`
  fail-closed，空 head→blob None→FAIL），**不是** `status.head_sha`。合法路径
  （用例 1/6a/F1）的 evidence 均提交在 status.head_sha **之后**（post-head）、tracked、
  digest 匹配 → PASS，确认未重新引入 Fable5 指出的 blob@head_sha 死锁 ✓。
- **共享 `evidence_path_exists`(:394) 未动**：全范围 diff 对该函数零改动行，8 个既有
  caller（:434/:460/:479/:482/:493/:636/:790 等）原样服务 ✓。

### 4) finding-4：task scope 规范与唯一 — 通过

- 裸 `<id>` scope 被拒：`_exception_covering_task`(:1104) 只认 `scope == f"task:{task_id}"`
  且显式过滤 `assertion_id == "review_fingerprint_trails_status"`。F4-raw（scope `"C"`）
  → `uncovered task C` FAIL ✓。
- task id 非空唯一（>=2 时）：`_task_id_errors`(:256) 由 validate_tasks /
  validate_task_coverage / validate_dispatch_ready 显式调用（不进 normalize_tasks，
  checkpoint 经 validate_parallel_mode 不触发，bootstrapping 保住）。F4-dup → FAIL；
  F4-miss（2 任务缺 id）→ FAIL ✓。
- 跨 scope 碰撞（自构 ADV-xscope）：task 恰好命名 `review_1` + 一条 scope `"review_1"`
  的 review 级例外 → task 侧查的是 `task:review_1`，不匹配 → `uncovered task review_1`
  FAIL ✓（Codex 指出的 review scope 溢盖 task 已封死）。
- 一例外不盖两 task（自构 ADV-1exc2t）：3 任务链仅配 `task:B` 例外 → `uncovered task C`
  FAIL ✓；重复 id 路径由 id 唯一性先行 FAIL（F4-dup）✓。

### 5) P2：reason/at 结构校验 + 模板键 — 通过

- `reason` 非空(:1014)、`at` 经 `_valid_iso8601`(:847，`Z`→`+00:00` 后
  `datetime.fromisoformat`，Python 3.9.6 实测）可解析(:1017)；P2-r/P2-a 复跑均 FAIL ✓。
- 模板 `tasks[]` 的 `covers_through_task` 已删（diff 实证），review 侧
  `review_1/review_2.covers_through_task` 保留 ✓。

### 6) R4 两处（bookkeeper 补）— 通过

- R-A：模板 `authorized_exceptions` 现为空数组 `[]`（rework commit 对模板净改动仅删
  covers_through_task 一行；全-null 占位已回退）。
- R-B：`_task_id_errors` 开头 `if len(tasks) < 2: return []`；>=2 仍强制（F4-miss FAIL
  实证 finding-4 未削弱）。
- TPL 用例复跑：直读模板默认值调 `validate_tasks` / `validate_authorized_exceptions`
  → 双双 `errors = []`；F4-sgl（单 null-id 任务）→ PASS。模板默认不再破 pre-accept ✓。

### 7) Q2 文档措辞 — 通过

- `AGENTS.md` RC4 段：`authorizer=="user"` 字面量 + 钉指纹 + 仓内已提交 digest 封存
  evidence =「自授无法静默发生」；明确「code cannot prove the evidence text came from a
  human」「final guarantee is mandatory human verification…a workflow obligation the
  validator cannot mechanically enforce」；negative-list #4/#5 同步更新 ✓。
- `docs/harness-design.md`：同两层表述 + committed-blob 读取 + post-head 合法性 +
  「Anti-self-grant is two layers, not an impossibility claim」专段 ✓。
- `validate-stage.py` :982-990 注释：同样两层 + 人核是流程义务 ✓。
- 旧的过度声称（「授权原文只能来自用户消息这一结构」）在三处均已删除 ✓。

### 8) 禁改边界 — 通过

- `compute_diff_fingerprint`(:183-199)、`validate_review_identity`(:755)、validate_tasks
  内既有 task 身份检查(:826-830)：全范围 diff **零改动行**（逐行 grep 实证）✓。
- class-2 未收：白名单仍单元素；CL2 复跑（`review_2.verdict=REWORK` + 合规例外）→ FAIL ✓。
- 身份 negative-list：ID 复跑（review_2 与实现者同 provider）→ FAIL ✓。
- 无 RC1-3 语义改动：rework commit c82fc2b 仅触 6 文件（4 交付 + 40/60 证据）✓。
- `funding_hedging`：全范围 diff 0 文件 ✓；AGENTS.md 两仓 carve-out 行不在任何 hunk 内 ✓。

## 2. 测试证据独立复跑（proof by execution）

- 从 `60-test-output.txt` E 段源码（:628-968）+ E.1 段 R4 三例替换（:1074-1119）逐字重建
  `/tmp/rc4_round2_review.py`，另加 6 条自构对抗（ADV-rescue / ADV-fp / ADV-prov /
  ADV-xscope / ADV-1exc2t / ADV-untracked）。
- `python3 -m py_compile scripts/validate-stage.py` → COMPILE_OK（Python 3.9.6）。
- 执行结果：**31/31 ALL CASES AS EXPECTED**——E/E.1 归档 25 用例全过（输出与归档一致，
  含横幅文案、错误消息、TPL 双零 error），零回归；6 条自构对抗全过。

## 3. round-1 遗留 P2 关闭核对

round-1（`30-review-1.md@972fd06`）P2×3 全部被本 rework 覆盖：模板 covers_through_task
键（已删）、evidence 真实性靠流程（finding-3 硬化 + Q2 措辞落地）、evidence 绝对路径
（已拒 + 逃逸防护）。无遗留。

## 4. 非阻塞观察项（不影响 ACCEPT）

- **P3**：同一 id/身份错误会从多个调用点重复打印（如 F4-dup/F4-miss 同条报错出现 3 次：
  validate_tasks 经 main 与 validate_acceptance 两路 + validate_task_coverage 一路）。
  仅输出噪音，FAIL 方向正确、无重复计数进 applied；建议后续去重或单点调用，非本 stage 阻塞项。
- 噪音级：AGENTS.md 末尾新增一个多余空行（cosmetic）。
- 提示（pre-existing，非本 diff 引入）：身份模型对**未注册别名**按字符串原样当身份
  （`PROVIDER_IDENTITIES.get(text, text)`），与顶层身份门同一语义；ADV-prov 首跑用未注册
  别名即未触发同 provider 报错，改注册别名 `glm52` 后正常拦截。属注册表治理话题，与
  既有身份门一致，不在本 rework 范围。

## 5. json_schema_valid 判定

本报告末尾 verdict JSON 已按 `schemas/review-verdict.schema.json` 用 python jsonschema
4.25.1（draft 2020-12）校验通过：required 全含、additionalProperties=false 无多键、
finding（P3）五要素齐、ACCEPT 无需 fix_start_prompt。判定：**true**。

## 6. verdict 依据汇总

- 复核重点 8/8 通过；归档 25 用例独立复跑全过；自构 6 对抗用例全过；pre-review 门现场
  PASS；双路指纹复算与 status 逐字符一致。
- P0=0，P1=0，P2=0；P3=1（重复报错噪音，非阻塞）。round-1 P2×3 已全关。

当前 Session ID: unavailable（Kimi CLI 会话内模型无法自省 provider-native Session ID；操作者可经 runner 补录 status.session_receipts）
Session ID 来源: unavailable（会话内无 runtime_env/hook_payload/cli_output 可查）
原始输出路径: reports/agent-runs/2026-07-harness-authorized-exception-v1/30-review-1.md
本地北京时间: 2026-07-17 15:56:00 CST
下一步模型: bookkeeper（Claude Opus 4.8）→ 提交本报告与 status 写回，派 round-2 review-2（codex）
下一步任务: 记录 round-2 review-1 ACCEPT 证据，准备 review-2 dispatch（终审重点：finding-3 五条硬化的对抗面与 R4 模板默认值路径）

```json
{
  "schema_version": 1,
  "stage_id": "2026-07-harness-authorized-exception-v1",
  "role": "first_reviewer",
  "model": "kimi-for-coding",
  "verdict": "ACCEPT",
  "diff_fingerprint": "c82fc2b05299e1edc1ad972b12e12d8597cfb394:f3d3fb338fd5983b864c858826ea20748ddb49076b69ae3061b3a6467f7ce9d2",
  "reviewer_prior_involvement": "none",
  "reviewed_artifacts": [
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/50-review-2.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/56-direction-ruling-finding3-fable5.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/40-fix-report.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/10-design.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/60-test-output.txt",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/status.json",
    "scripts/validate-stage.py",
    "AGENTS.md",
    "docs/harness-design.md",
    "reports/agent-runs/_template/status.json",
    "git diff 00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..c82fc2b05299e1edc1ad972b12e12d8597cfb394",
    "independent re-execution: 25/25 archived construction+adversarial cases (E/E.1) + 6/6 reviewer-added adversarial cases (rescue, wrong-fp, same-provider, cross-scope, one-exc-two-tasks, untracked-worktree-evidence)",
    "dual-path diff-fingerprint recomputation (python compute_diff_fingerprint + raw git diff | shasum)"
  ],
  "findings": [
    {
      "severity": "P3",
      "title": "Identical task-id/identity errors are printed once per call site (up to 3 duplicates)",
      "file": "scripts/validate-stage.py",
      "line": 256,
      "evidence": "_task_id_errors is invoked from validate_tasks (reached via both main:1326 and validate_acceptance:1276) and validate_task_coverage:1137; F4-dup/F4-miss/ADV-prov reruns each show the same error line three times.",
      "impact": "Output noise only; fail direction is correct, no exception is double-counted into applied, and the banner is unaffected.",
      "recommendation": "Non-blocking; deduplicate identical error strings or call _task_id_errors once per phase in a future validator revision."
    }
  ],
  "required_fixes": [],
  "residual_risks": [
    "Human-origin authenticity of exception evidence remains a workflow obligation (mandatory human verification of the evidence verbatim at the PASS-with-exception banner), honestly documented as non-mechanical per the Fable5 Q2 ruling.",
    "Pre-existing identity model treats unregistered alias strings as their own provider identity (same semantics as the top-level identity gate); registry hygiene is the mitigation, unchanged by this diff.",
    "AGENTS.md gained a stray trailing blank line at EOF (cosmetic)."
  ],
  "next_action": "continue"
}
```
