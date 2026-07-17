> Bookkeeper 说明:本文件由 codex read-only 会话逐字抽取保存(codex supports_write:false,无法自写)。来源 session 019f6e4c-3f2d-7713-ad86-26ede6c00456,末条 assistant 消息,内容未改。

# Review-2 结论：REWORK

复算指纹一致：

`941416345c92bf3c42556e0d9178f28e1384b4e6:5b28d88ce253d6d41f9c5841cfe51265a66e132c8fe7556d1177da4404b186f0`

P0=0，P1=4，P2=2。需要返修的核心问题：

1. task 级例外会被静默使用。`validate_task_coverage()` 消费 task 例外却只返回 errors；最终输出只统计 review 级例外。可以依赖 task 例外 PASS，但完全不打印 PASS-with-exception。
2. 任意非空字符串都能冒充 task 自有 review。代码没有要求 task review 的 `verdict=ACCEPT`、`json_schema_valid=true` 或指纹等于 task 指纹，存在假绿。
3. 自授风险需要硬化（P1）。`authorizer=="user"` 加任意非空文件不能证明授权来自用户，且文件可位于仓外、未绑定 `head_sha`。当前文档“模型不能自授”的表述不成立。
4. task scope 不唯一：同时接受 `task:<id>` 和裸 `<id>`，task ID 又不要求唯一。一个例外可能跨 review/task scope 或覆盖多个同名 task。

确认通过的部分：

- malformed exception 的全局 fail-closed 路径成立。
- verdict、schema、tests 和 reviewer identity 没有被 review 例外降级。
- 指纹公式和身份隔离逻辑未改。
- 链式与前缀指纹计算本身未发现 diff 代数漏洞；bool、越界整数会被拒绝。
- 指纹钉定会在 head/fingerprint 改变后失效，但授权证据内容本身尚未被钉定。
- `git diff --check` 通过，未触碰 `funding_hedging`。

本次遵守 reviewer read-only 约束，没有修改 `50-review-2.md` 或 `status.json`。初始读取、git 检查和指纹复算成功；随后 macOS 对目标目录返回 `Operation not permitted`，因此未把计划中的补充对抗用例声称为已执行。上述 P1 均由确定可达的源码路径直接得出。

当前 Session ID: unavailable（运行环境未暴露 provider-native Session ID）
Session ID 来源: unavailable
原始输出路径: unavailable（请 bookkeeper 将本回复原样保存至 `reports/agent-runs/2026-07-harness-authorized-exception-v1/50-review-2.md`）
本地北京时间: 2026-07-17 12:23:39 CST
下一步模型: claude_glm（由 human operator 派发）
下一步任务: 按 verdict 中的 fix_start_prompt 修复四条 P1，补对抗测试后重新进入 review-1

{
  "schema_version": 1,
  "stage_id": "2026-07-harness-authorized-exception-v1",
  "role": "final_reviewer",
  "model": "gpt-5",
  "verdict": "REWORK",
  "diff_fingerprint": "941416345c92bf3c42556e0d9178f28e1384b4e6:5b28d88ce253d6d41f9c5841cfe51265a66e132c8fe7556d1177da4404b186f0",
  "reviewer_prior_involvement": "none",
  "reviewed_artifacts": [
    "AGENTS.md",
    "workflows/templates/stage-delivery.yaml review-2 section",
    "schemas/review-verdict.schema.json",
    "agents/skills/reality-checker.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/00-task.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/10-design.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/11-adr.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/12-development-breakdown.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/20-implementation.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/30-review-1.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/60-test-output.txt",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/status.json",
    "scripts/validate-stage.py",
    "reports/agent-runs/_template/status.json",
    "docs/harness-design.md",
    "git diff 00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..941416345c92bf3c42556e0d9178f28e1384b4e6",
    "independent diff-fingerprint recomputation"
  ],
  "findings": [
    {
      "severity": "P1",
      "title": "Task-scoped authorized exceptions can affect acceptance without being reported",
      "file": "scripts/validate-stage.py",
      "line": 1000,
      "evidence": "validate_task_coverage consumes valid_exceptions at lines 1000 and 1036-1040 but returns only errors. main prints only applied_exceptions returned by validate_acceptance at lines 1130 and 1150-1156. With full-fingerprint top-level reviews, task exceptions can satisfy every uncovered task while applied_exceptions remains empty, yielding an unannotated PASS.",
      "impact": "The validator can release a stage using authorized exceptions without the mandatory PASS-with-exception disclosure, directly violating the non-silent hard gate and hiding waiver use from operators.",
      "recommendation": "Return and aggregate the exact task exceptions relied upon, deduplicate them with review exceptions, and print every applied assertion_id@scope whenever any exception contributes to PASS."
    },
    {
      "severity": "P1",
      "title": "An arbitrary non-empty string is accepted as a task's own review",
      "file": "scripts/validate-stage.py",
      "line": 1032,
      "evidence": "has_own_review checks only that task.review_1.diff_fingerprint is a non-empty string. validate_tasks verifies the task boundary fingerprint and optional provider separation, but never requires the nested review verdict, schema validity, reviewer identity, evidence, or equality between the review fingerprint and the task fingerprint.",
      "impact": "A trailing task can be marked covered by a fabricated object such as {\"review_1\":{\"diff_fingerprint\":\"x\"}}, allowing D3 to produce a false-green without a valid task review.",
      "recommendation": "When a task review is used for coverage, require a complete schema-valid ACCEPT review, an identified cross-provider reviewer, and review.diff_fingerprint equal to the recomputed task.diff_fingerprint; reject incomplete review objects."
    },
    {
      "severity": "P1",
      "title": "User authorization provenance is neither authenticated nor sealed to the reviewed commit",
      "file": "scripts/validate-stage.py",
      "line": 886,
      "evidence": "The validator accepts the literal authorizer value \"user\" plus any currently existing non-empty file. It does not link the record to a structured user_authorizations entry or trusted runtime receipt, does not require the evidence file to be repository-relative, and does not verify the evidence blob at status.head_sha. A write-capable model can create both the record and purported evidence.",
      "impact": "The party seeking the waiver can self-grant it and still receive a validator PASS. Later committed or external evidence can also escape the recorded diff seal. This defeats the central human-only authorization property.",
      "recommendation": "Harden before acceptance: require canonical structured user-authorization linkage and trusted receipt metadata, restrict evidence to repository-relative committed files, verify the evidence blob or digest at head_sha, and retain mandatory human inspection. Update documentation so it does not claim code proves human origin where it cannot."
    },
    {
      "severity": "P1",
      "title": "Ambiguous and non-unique task scopes allow one exception to cover unintended assertions",
      "file": "scripts/validate-stage.py",
      "line": 1036,
      "evidence": "Task matching accepts both task:<id> and the raw id, while normalize_tasks and validate_tasks do not require non-empty unique IDs. Thus scope review_1 also covers a task whose id is review_1, and one task:X exception covers every duplicate task named X. The match also does not explicitly re-check assertion_id.",
      "impact": "A waiver authorized for one review or task can silently widen to another task, defeating assertion-instance scoping and causing false-green coverage.",
      "recommendation": "Require unique non-empty task IDs, accept only canonical task:<id> task scopes, explicitly require the class-1 assertion_id at the task consumer, and reject ambiguous or duplicate scope targets."
    },
    {
      "severity": "P2",
      "title": "Authorized-exception reason and timestamp are not structurally validated",
      "file": "scripts/validate-stage.py",
      "line": 871,
      "evidence": "The approved D2 record shape includes reason and at, and the negative list promises field completeness, but validate_authorized_exceptions never checks either field.",
      "impact": "Waiver records can lack their audit rationale and time metadata while being treated as structurally valid.",
      "recommendation": "Require a non-empty reason and a parseable ISO-8601 at value as part of fail-closed record validation."
    },
    {
      "severity": "P2",
      "title": "Template places covers_through_task on a task object even though the validator reads it only from reviews",
      "file": "reports/agent-runs/_template/status.json",
      "line": 47,
      "evidence": "The template task example contains covers_through_task, but validate_task_coverage reads only review_1/review_2.covers_through_task.",
      "impact": "Operators may populate the wrong location and incur misleading uncovered-task failures.",
      "recommendation": "Remove the task-side key or replace it with unambiguous documentation outside the JSON data shape."
    }
  ],
  "required_fixes": [
    "Aggregate and print every review- and task-scoped exception actually used to obtain PASS.",
    "Require complete, valid, fingerprint-bound task review records before treating a task as reviewed.",
    "Replace arbitrary evidence-file trust with structured, repository-bound, commit-sealed user-authorization evidence and correct the human-origin documentation claim.",
    "Enforce unique task IDs and canonical assertion-specific scopes so one exception cannot cross reviews or cover multiple tasks."
  ],
  "residual_risks": [
    "Even after structured evidence hardening, human-origin authenticity ultimately requires a trusted runner receipt or explicit human inspection; repository text alone is not cryptographic proof.",
    "The supplemental adversarial execution planned by review-2 was not run after macOS denied further reads of the target directory; no unexecuted test is represented as evidence.",
    "The P2 template-key and reason/timestamp issues should be corrected during the same bounded fix if practical."
  ],
  "fix_start_prompt": "Stage: 2026-07-harness-authorized-exception-v1\nReviewed fingerprint: 941416345c92bf3c42556e0d9178f28e1384b4e6:5b28d88ce253d6d41f9c5841cfe51265a66e132c8fe7556d1177da4404b186f0\nRaw review artifact: reports/agent-runs/2026-07-harness-authorized-exception-v1/50-review-2.md\nRaw verdict JSON: final JSON object in that same file.\n\nImplementer: claude_glm. Make a bounded repair only; Codex must not implement it.\n\nRequired fixes, in order:\n1. Change validate_task_coverage so it reports the exact task-scoped exceptions it actually relies on. Aggregate and deduplicate those with validate_acceptance's review exceptions, and make main print all applied assertion_id@scope values. A PASS dependent only on task exceptions must still print PASS-with-exception.\n2. A task own-review may satisfy coverage only when it is complete and valid: require identified reviewer/provider/model, verdict ACCEPT, json_schema_valid true, cross-provider separation from the task owner/implementer, and diff_fingerprint exactly equal to the recomputed task fingerprint. An arbitrary non-empty string must fail.\n3. Harden authorization provenance. Replace arbitrary current-filesystem evidence trust with a canonical structured user-authorization reference and trusted receipt metadata; require a repository-relative evidence path contained inside the repository, reject absolute/outside paths, and verify the committed evidence blob or digest at status.head_sha. Keep mandatory human verification explicit and remove the inaccurate claim that the literal alone prevents model self-authorization.\n4. Require unique non-empty task IDs. Accept only canonical task:<id> task scopes, explicitly filter task exceptions by assertion_id == review_fingerprint_trails_status, and reject raw-ID aliases, duplicate targets, cross-scope collisions, and unresolved scopes.\n5. Validate non-empty reason and parseable ISO-8601 at fields. Remove the misleading task-side covers_through_task key from reports/agent-runs/_template/status.json.\n\nAllowed delivery files: scripts/validate-stage.py, reports/agent-runs/_template/status.json, AGENTS.md, docs/harness-design.md, and this stage's implementation/fix/test evidence files as routed by the bookkeeper. Forbidden: changing compute_diff_fingerprint, weakening reviewer identity checks, implementing class-2 exceptions, RC1/RC2/RC3 work, modifying funding_hedging, merging, pushing, or broad refactoring.\n\nRequired adversarial evidence:\n- task-only exceptions produce PASS-with-exception listing every relied-upon task scope;\n- arbitrary or stale task review fingerprints fail;\n- task review REWORK or json_schema_valid=false fails;\n- a complete fingerprint-matching cross-provider task review passes;\n- missing/duplicate task IDs fail;\n- raw scope aliases and review/task collisions fail;\n- one task exception cannot cover two tasks;\n- absolute, outside-repository, post-head, empty, or unlinked evidence fails;\n- missing reason/at fails;\n- one malformed exception invalidates all exceptions;\n- prior 10 construction cases and class-2/identity negative-list cases remain unchanged.\n\nRun at minimum: python3 -m py_compile scripts/validate-stage.py; rerun the archived construction harness from reports/agent-runs/2026-07-harness-authorized-exception-v1/60-test-output.txt; run the new adversarial cases against the real validator; python3 scripts/validate-stage.py 2026-07-harness-authorized-exception-v1 --phase checkpoint. After the bookkeeper commits and recomputes the standard fingerprint, run python3 scripts/validate-stage.py 2026-07-harness-authorized-exception-v1 --phase pre-review. Record full commands and output in 60-test-output.txt and map every review-2 finding to its fix in 40-fix-report.md. Success requires all four P1 paths to fail closed, all applied exceptions to be visible, no fingerprint/identity regression, a clean committed stage branch, and re-entry through cross-provider review-1 before review-2.",
  "next_action": "fix"
}
