# Review-1 — Stage A: authorized_exception + 分任务指纹覆盖（RC4）

- reviewer: kimi（provider moonshot），与实现者 claude_glm（zhipu_glm）provider 级隔离 ✓
- 被审范围：`00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..941416345c92bf3c42556e0d9178f28e1384b4e6`
- 被审文件：`scripts/validate-stage.py`、`AGENTS.md`、`docs/harness-design.md`、`reports/agent-runs/_template/status.json`
- 结论：**ACCEPT**（P0=0，P1=0，P2=3；均不阻塞）

## 0. 独立复算指纹（非转述 status）

用改后 `validate-stage.py` 的 `compute_diff_fingerprint()` 在本仓现场复算：

```
941416345c92bf3c42556e0d9178f28e1384b4e6:5b28d88ce253d6d41f9c5841cfe51265a66e132c8fe7556d1177da4404b186f0
```

与 `status.diff_fingerprint` **逐字符一致**。另现场跑
`python3 scripts/validate-stage.py 2026-07-harness-authorized-exception-v1 --phase pre-review`
→ `STAGE VALIDATION PASSED`，exit=0，打印同一指纹。

## 1. 测试证据独立复跑（proof by execution，非只读归档）

- 从 `60-test-output.txt` D 段提取构造脚本源码（176–416 行）至 `/tmp/rc4_review1_check.py`，
  `py_compile` 通过后现场执行：**10/10 用例 ALL CASES AS EXPECTED**，输出与归档一致。
- 自写 4 条对抗用例（`/tmp/rc4_review1_adversarial.py`，复用归档脚手架）补实现者未隔离的
  绕过路径，全部符合预期：
  - **A. class-2 尝试**：合规例外（scope=review_1）+ `review_1.verdict="REWORK"` →
    FAIL，唯一 error 为 `review_1.verdict must be ACCEPT`；指纹 mismatch **未出现**
    （例外确实只降级了指纹断言）——双向证明降级是外科手术式的。
  - **B. 跨 scope**：只有 review_1 例外、review_2 同样 trailing →
    `review_2.diff_fingerprint must match status.diff_fingerprint` 照红，review_1 被放行。
  - **C. negative-list #3**：reviewer 与实现者同 provider（zhipu_glm）+ 合规例外 →
    `review_1 provider identity must differ …` 照红（身份分离不可豁免）。
  - **D. task-scope 不降 review**：仅 `task:A` 例外 + review_1 trailing →
    review_1 mismatch 照红（task 例外进不了 review 降级路径）。

## 2. 严核清单逐条结论

### 1) negative-list 五项确实不可豁免（代码结构，非注释）— 通过

| # | 断言 | 生效位置 | 结构性证据 |
|---|---|---|---|
| ① | status fp 重算 | `validate_common` :695-701，main :1119 对 pre-accept 无条件先跑 | 降级逻辑只存在于 `validate_acceptance` :1080-1091 的 review 指纹比较分支，够不到 validate_common；用例 5（篡改 fp + 自洽例外 → 唯一 error 为 validate_common 的 mismatch）实证 |
| ② | clean worktree | `require_clean_worktree` :146-152，main :1114-1118 最先执行 | 无任何例外路径接触 |
| ③ | 身份分离 | `validate_review_identity` :723-768 | 不在任何 diff hunk 内、零改动、无 override；对抗用例 C 实证挂例外仍 FAIL |
| ④⑤ | 例外记录自身 evidence/结构 | `validate_authorized_exceptions` :847-927 | error 直进 errors 且 :925-926 `if errors: return errors, []` fail-closed；用例 2/3/4 实证 |

`validate_acceptance` 的降级分支**只**作用于 `review.diff_fingerprint != status` 一条：
verdict（:1074）、json_schema_valid（:1077）、tests.status（:1059-1060）均为无条件
append error，无例外查询调用。✓

### 2) class-2 确实未被放宽 — 通过

白名单 `AUTHORIZED_EXCEPTION_ASSERTION_IDS = {"review_fingerprint_trails_status"}`（:84）
仅含 class-1；`assertion_id` 不在枚举即 error（:876-882）且 fail-closed。对抗用例 A 实证
verdict 断言在合规例外存在时照红。

### 3) 钉指纹真能"下一轮自动失效" — 通过

:895-901 `applies_to_fingerprint != status.diff_fingerprint` → error → fail-closed →
`_exception_covering_review` 必返回 None → review 指纹断言复红。下一轮 fix 改 head_sha
必然改 status fp，旧记录自动失效。用例 3（钉错值即 FAIL）走同一相等性判断，充分。
无 expiry 字段与设计一致。

### 4) fail-closed 真成立 — 通过

`validate_authorized_exceptions` 任一坏记录 → `return errors, []`（:925-926）→
`validate_acceptance`（:1066-1068）valid 集为空 → 降级不可触发。
`validate_task_coverage` :1000 第二次调用同一函数取 valid 子集，受同一 fail-closed 约束；
其丢弃的 errors 已由 `validate_acceptance` 先 surfaced（main :1130-1132 顺序保证），无丢失。

### 5) 退化零改变 — 通过

`validate_task_coverage` :988-989 `len(tasks) <= 1 → return []`（0/单任务完全走旧路径）。
对照 `git show 00e25b4:scripts/validate-stage.py`：四条既有断言文案
（tests.status / verdict / json_schema_valid / diff_fingerprint must match）逐字不变。
用例 7a/7b/REG 实证。

### 6) 边界零越界 — 通过

- `compute_diff_fingerprint` :182-198 不在任何 diff hunk 内，公式一字未动（我的复算值
  与 status 一致亦佐证）。
- 身份分离逻辑未动（见 1③）。
- 全量 diff 仅 4 个声明文件 + 本 stage 证据目录；无 funding_hedging 仓文件（异仓）。
- 无 RC1/RC2/RC3 内容。
- AGENTS.md 为纯新增（+20/−0），两仓 carve-out 等既有行零触碰。

### 7) §5 规格冲突处置（covers_through_task 落点）— 可接受，记 P2

生效字段在 `review_1`/`review_2.covers_through_task`（逻辑规格，validator 实际读取），
`_template` tasks[] 示例项上的同名键是说明性注解、validator 不读。JSON 无注释，存在
误导后续 operator 的余地；但失败方向安全（填错位置 → `uncovered task` 红，不会产生
假绿）。20-implementation §5 已如实披露。判 P2，不阻塞。

## 3. Findings

### P2-1 模板 tasks[] 的 covers_through_task 注解键有误导余地
- file: `reports/agent-runs/_template/status.json:47`
- evidence: tasks[] 示例项含 `"covers_through_task": null`，validator 只读 review 侧
  （`validate_task_coverage` :1003-1007）；JSON 无法注释说明。
- impact: 后续 stage operator 可能把覆盖声明填到 task 上而误以为生效；结果为红（uncovered
  task），不致假绿，但浪费一轮排错。
- recommendation: 删除 tasks[] 示例项上的该键，或改名为 `_note` 类明显非字段键；可随
  下一次模板修订顺带处理。

### P2-2 evidence_file 内容真实性靠评审过程约束（设计明示范围）
- file: `scripts/validate-stage.py:906-913`
- evidence: validator 校验 evidence_file 存在+非空；`authorizer=="user"` 为字面量检查。
  "授权原文只能来自用户消息"不由代码强制（12-breakdown §B 明示"至少非空；不做 NLP 校验"）。
- impact: 恶意/失职的 status 写入方可伪造非空 evidence 文本通过结构校验；最终防线是
  review-1/2 与人核对 evidence 原文。
- recommendation: 不阻塞（符合已批准规格）；提请 review-2 按其既定 focus
  （"例外机制会不会被绕过自授"）重点确认该残余风险可接受。

### P2-3 evidence_file 接受绝对路径可指向仓外
- file: `scripts/validate-stage.py:363-378`（`evidence_path_exists` 既有模式）
- evidence: 绝对路径直接采纳；沿用 :735-737 override-evidence 既有模式，非本次新增面。
- impact: evidence 可落在仓外、不进 git 证据链。
- recommendation: 后续模板修订可考虑限制 evidence_file 必须解析到仓内；本次不动。

## 4. json_schema_valid 判定

本报告末尾 verdict JSON 已按 `schemas/review-verdict.schema.json` 用 python `jsonschema`
校验通过（draft 2020-12，additionalProperties=false 下无多余键；findings 三项均含
severity/title/evidence/impact/recommendation）。判定：**true**。

## 5. verdict 依据汇总

- 严核清单 7/7 通过；实现者 10 用例独立复跑全过；自写 4 对抗用例全过。
- P0=0，P1=0，P2=3（均不阻塞，2 条转后续模板修订、1 条转 review-2 focus）。
- 指纹复算值与 status 一致；pre-review 门现场 PASS。

当前 Session ID: unavailable（Kimi CLI 会话内模型无法自省 provider-native Session ID；操作者可经 runner 补录 status.session_receipts）
Session ID 来源: unavailable（会话内无 runtime_env/hook_payload 可查）
原始输出路径: reports/agent-runs/2026-07-harness-authorized-exception-v1/30-review-1.md
本地北京时间: 2026-07-17 12:02:23 CST
下一步模型: bookkeeper（Claude Opus 4.8）→ 提交本报告与 status 写回，派 review-2（codex）
下一步任务: 记录 review-1 ACCEPT 证据，准备 review-2 dispatch（残余风险 P2-2 提请列入 review-2 审查重点）

```json
{
  "schema_version": 1,
  "stage_id": "2026-07-harness-authorized-exception-v1",
  "role": "first_reviewer",
  "model": "kimi-for-coding",
  "verdict": "ACCEPT",
  "diff_fingerprint": "941416345c92bf3c42556e0d9178f28e1384b4e6:5b28d88ce253d6d41f9c5841cfe51265a66e132c8fe7556d1177da4404b186f0",
  "reviewer_prior_involvement": "none",
  "reviewed_artifacts": [
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/10-design.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/11-adr.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/12-development-breakdown.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/20-implementation.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/60-test-output.txt",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/status.json",
    "git diff 00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..941416345c92bf3c42556e0d9178f28e1384b4e6 (scripts/validate-stage.py, AGENTS.md, docs/harness-design.md, reports/agent-runs/_template/status.json)",
    "independent re-execution: 10/10 archived construction cases + 4/4 reviewer adversarial cases (class-2 attempt, cross-scope, identity negative-list #3, task-scope isolation)"
  ],
  "findings": [
    {
      "severity": "P2",
      "title": "Template tasks[] carries a covers_through_task annotation key the validator never reads",
      "file": "reports/agent-runs/_template/status.json",
      "line": 47,
      "evidence": "validate_task_coverage reads review_1/review_2.covers_through_task only (scripts/validate-stage.py:1003-1007); the tasks[] example key is documentation-only and JSON cannot carry a clarifying comment.",
      "impact": "A future stage operator may declare coverage on the task side and believe it is effective; failure direction is safe (uncovered-task red, never false-green) but costs a debugging round.",
      "recommendation": "Remove the tasks[] annotation key or rename it to an obviously non-field key in a future template revision; non-blocking."
    },
    {
      "severity": "P2",
      "title": "evidence_file content authenticity is process-enforced, not code-enforced (within approved spec)",
      "file": "scripts/validate-stage.py",
      "line": 906,
      "evidence": "Validator checks evidence_file existence and non-emptiness plus the authorizer=='user' literal; it does not verify the file text is a genuine user message (12-development-breakdown.md section B explicitly scopes this to 'at least non-empty; no NLP check').",
      "impact": "A malicious or negligent status writer could fabricate non-empty evidence text and pass structural checks; the final defense is reviewers and the human operator inspecting the evidence content.",
      "recommendation": "Non-blocking per approved spec; flag to review-2 under its assigned focus (self-authorization bypass) to confirm the residual risk is acceptable."
    },
    {
      "severity": "P2",
      "title": "evidence_file accepts absolute paths outside the repository",
      "file": "scripts/validate-stage.py",
      "line": 363,
      "evidence": "evidence_path_exists (reused pre-existing :735-737 pattern, not new in this diff) accepts absolute paths directly.",
      "impact": "Exception evidence can live outside the repo and escape the git evidence chain.",
      "recommendation": "Consider constraining evidence_file to resolve inside the repository in a future template revision; non-blocking."
    }
  ],
  "required_fixes": [],
  "residual_risks": [
    "P2-2: exception evidence authenticity relies on reviewer/human inspection of the evidence file content (no NLP verification by design); assigned to review-2 focus.",
    "P2-1/P2-3: template annotation-key clarity and absolute-path evidence are deferred to future template revisions."
  ],
  "next_action": "continue"
}
```
