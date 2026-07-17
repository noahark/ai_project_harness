> Bookkeeper 说明:round-2 review-2,由 codex read-only 会话逐字抽取覆盖保存(round-1 的 50-review-2.md 见 git @3ede818)。来源 session 019f6e4c-3f2d-7713-ad86-26ede6c00456 末条 assistant 消息,内容未改。

# Round-2 Review-2：REWORK

权威审查范围：

`00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..c82fc2b05299e1edc1ad972b12e12d8597cfb394`

双路复算指纹一致：

`c82fc2b05299e1edc1ad972b12e12d8597cfb394:f3d3fb338fd5983b864c858826ea20748ddb49076b69ae3061b3a6467f7ce9d2`

`py_compile`、`pre-review`、`checkpoint` 均通过，但仍有 4 条 P1：

1. task own-review 的 provider 隔离可绕过。代码优先信任 `reviewer` 标签，而不是必填的 `provider`。实测 owner=`claude_glm`、provider=`zhipu_glm`，只要 reviewer 写成任意未注册字符串，own-review 即返回 `(True, None)`。
2. evidence 可以用 Git tree 冒充文件。实测 `evidence_file="reports"`，对 `git show HEAD:reports` 输出计算匹配 digest 后，`_validate_exception_evidence()` 返回无错误；对象类型实际为 `tree`。
3. stage diff 混入 469 行无关的 `docs/planning/adapter-watchdog-runner.md`，且未列入 `status.changed_files`。该文件在实现前已是未跟踪用户改动，随后被 bookkeeper 提交进本 stage，违反文件边界和“模板/Runner 同步只落 main”的硬门。
4. review-2 dispatch 仍绑定 round-1 的旧 head `9414163` 和旧指纹，违反“review prompt 必须使用 status 记录范围”的硬门。本次仅因直接读取 status/handoff 才纠正到新范围。

已确认 round-1 的静默 task 例外、裸 scope/重复 ID、reason/at、模板键等修复本身成立；Fable5 批准的 `evidence_sha256 + 验证时 HEAD` 方案也可接受，不重新要求 blob@status.head_sha。

Reviewer 只读执行，未修改仓库。

当前 Session ID: unavailable（运行环境未暴露 provider-native Session ID）
Session ID 来源: unavailable
原始输出路径: unavailable（请 bookkeeper 将本回复原样覆盖保存为 `reports/agent-runs/2026-07-harness-authorized-exception-v1/50-review-2.md`）
本地北京时间: 2026-07-17 16:31:36 CST
下一步模型: claude_glm 修复代码；bookkeeper 修复阶段边界与 dispatch
下一步任务: 完成四条 P1 后重新提交、复算指纹、执行 round-3 review-1，再准备绑定新指纹的 review-2 dispatch

{
  "schema_version": 1,
  "stage_id": "2026-07-harness-authorized-exception-v1",
  "role": "final_reviewer",
  "model": "gpt-5",
  "verdict": "REWORK",
  "diff_fingerprint": "c82fc2b05299e1edc1ad972b12e12d8597cfb394:f3d3fb338fd5983b864c858826ea20748ddb49076b69ae3061b3a6467f7ce9d2",
  "reviewer_prior_involvement": "none",
  "reviewed_artifacts": [
    "AGENTS.md",
    "workflows/templates/stage-delivery.yaml review-2 section",
    "schemas/review-verdict.schema.json",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/status.json",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/10-design.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/11-adr.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/12-development-breakdown.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/26-dispatch-review-2-codex.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/30-review-1.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/40-fix-report.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/50-review-2.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/56-direction-ruling-finding3-fable5.md",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/60-test-output.txt",
    "reports/agent-runs/2026-07-harness-authorized-exception-v1/70-handoff.md",
    "scripts/validate-stage.py",
    "reports/agent-runs/_template/status.json",
    "docs/harness-design.md",
    "git diff 00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..c82fc2b05299e1edc1ad972b12e12d8597cfb394",
    "independent Python and raw-git diff-fingerprint recomputation",
    "independent provider-label bypass and Git-tree evidence adversarial executions"
  ],
  "findings": [
    {
      "severity": "P1",
      "title": "Task own-review provider separation trusts the reviewer label before the declared provider",
      "file": "scripts/validate-stage.py",
      "line": 1086,
      "evidence": "review_provider_identity iterates reviewer before provider. An executed adversarial task with owner=claude_glm, review.provider=zhipu_glm, and review.reviewer=innocent-unregistered-label produced owner_identity=zhipu_glm, declared_provider_identity=zhipu_glm, chosen_review_identity=innocent-unregistered-label, and _task_own_review_covers returned (True, None). validate_tasks also returned no errors.",
      "impact": "A task can be implemented and reviewed by the same provider while satisfying D3 coverage, producing a false-green provider-isolation result.",
      "recommendation": "Use the explicit provider field as the authoritative provider identity. Prefer provider/selected_provider/primary_provider before reviewer globally, and reject inconsistencies when a recognized reviewer alias conflicts with the declared provider."
    },
    {
      "severity": "P1",
      "title": "A committed Git tree is accepted as non-empty authorization evidence",
      "file": "scripts/validate-stage.py",
      "line": 861,
      "evidence": "_evidence_committed_blob uses git show without checking object type or mode. Executing it with evidence_file=reports returned a non-empty rendered tree listing; a matching evidence_sha256 caused _validate_exception_evidence to return an empty error list, while git cat-file -t HEAD:reports reported tree.",
      "impact": "An exception can pass the mechanical evidence gate without any authorization-text file, defeating the committed-file and mandatory-human-verbatim assumptions.",
      "recommendation": "Require the referenced Git object to be a regular-file blob. Check object type and ls-tree mode, rejecting trees, symlinks, and submodules before hashing exact blob bytes. Add directory and symlink adversarial tests."
    },
    {
      "severity": "P1",
      "title": "The stage delivery diff contains an unrelated 469-line Adapter Watchdog Runner draft",
      "file": "docs/planning/adapter-watchdog-runner.md",
      "line": 1,
      "evidence": "The file entered at commit 972fd06 and contributes 469 added lines to base..c82fc2b. The original implementation-time test log already showed it as an unrelated untracked file. It is absent from status.changed_files and outside this stage's allowed RC4 files.",
      "impact": "The stage mixes unrelated user work into the reviewed fingerprint, violates declared file boundaries and the main-only Harness/Runner synchronization rule, and causes reviewers relying on changed_files or the four-file dispatch to miss part of the delivery.",
      "recommendation": "Have the bookkeeper preserve commit 972fd06 separately, then remove this file from the stage's net base..head diff without losing the user's work. Recompute changed_files and the standard fingerprint."
    },
    {
      "severity": "P1",
      "title": "The review-2 dispatch packet still targets the round-1 head and fingerprint",
      "file": "reports/agent-runs/2026-07-harness-authorized-exception-v1/26-dispatch-review-2-codex.md",
      "line": 9,
      "evidence": "The packet names head_sha 941416345c92bf3c42556e0d9178f28e1384b4e6 and its round-1 fingerprint, while status and the recovery header name c82fc2b05299e1edc1ad972b12e12d8597cfb394 and f3d3fb33... for round-2.",
      "impact": "A human executing the prepared packet would review the already-rejected round-1 diff and could record a verdict against the wrong committed state, violating a hard review gate.",
      "recommendation": "After the next fix and review-1, prepare a fresh round-specific review-2 packet containing the exact status-recorded base, head, fingerprint, current review-1 artifact, prior findings, and Fable5 evidence policy."
    },
    {
      "severity": "P3",
      "title": "The committed diff fails git diff --check due to a trailing blank line",
      "file": "AGENTS.md",
      "line": 483,
      "evidence": "git diff --check 00e25b4b..c82fc2b reports 'new blank line at EOF' and exits 2.",
      "impact": "Cosmetic whitespace noise only; no validator behavior changes.",
      "recommendation": "Remove the extra EOF blank line during the bounded fix."
    }
  ],
  "required_fixes": [
    "Make the declared review.provider authoritative for task own-review provider isolation and reject recognized reviewer/provider inconsistencies.",
    "Require authorized-exception evidence to be a regular Git file blob; reject trees, symlinks, submodules, and other non-file objects before digest validation.",
    "Remove docs/planning/adapter-watchdog-runner.md from the stage's net delivery diff while preserving the unrelated user work separately, then correct status.changed_files.",
    "Prepare the next review-2 dispatch only after the new fix head and round-3 review-1 are committed, using that exact status-recorded fingerprint."
  ],
  "residual_risks": [
    "Human-origin authenticity remains an explicit workflow obligation under the approved Fable5 direction; evidence_sha256 prevents silent content substitution but is not proof of human authorship.",
    "The validator may print duplicate task errors from multiple call sites; this remains non-blocking output noise.",
    "The current execution corrected the stale dispatch from status.json and the recovery header, but the stored packet itself remains invalid until replaced."
  ],
  "fix_start_prompt": "Stage: 2026-07-harness-authorized-exception-v1\nRound-2 reviewed fingerprint: c82fc2b05299e1edc1ad972b12e12d8597cfb394:f3d3fb338fd5983b864c858826ea20748ddb49076b69ae3061b3a6467f7ce9d2\nRaw review artifact and final verdict JSON: reports/agent-runs/2026-07-harness-authorized-exception-v1/50-review-2.md\n\nThis is rework-2. Claude-GLM fixes only the validator/code finding scope; the bookkeeper owns unrelated-work preservation, stage bookkeeping, commits, fingerprints, and dispatch packets.\n\nCode fixes:\n1. In task own-review identity validation, use review.provider as the authoritative vendor identity instead of allowing an arbitrary reviewer label to override it. Prefer provider/selected_provider/primary_provider before reviewer in review_provider_identity, or use a task-specific strict helper. If a reviewer value is a recognized provider alias and conflicts with provider, fail closed.\n2. Harden _evidence_committed_blob/_validate_exception_evidence so evidence must be a regular committed file. Verify the Git object type and ls-tree mode before reading bytes; reject tree objects, symlinks, submodules, and non-regular modes. Preserve the Fable5-approved validation-time HEAD plus evidence_sha256 design; do not bind evidence to status.head_sha.\n\nBookkeeper fixes:\n3. Preserve the unrelated user work from commit 972fd06 separately, then ensure docs/planning/adapter-watchdog-runner.md has no net diff in the RC4 stage range. Do not silently discard it or mix it into this stage. Correct status.changed_files to equal the actual bounded delivery.\n4. After code fixes are committed, recompute head_sha and the standard diff fingerprint, run pre-review, complete a fresh cross-provider round-3 review-1, and only then create/update a round-specific review-2 dispatch packet containing that exact new head and fingerprint. The packet must include the Fable5 evidence_sha256 ruling and the round-2 findings.\n\nAllowed code files: scripts/validate-stage.py, focused validator tests/evidence, AGENTS.md or docs/harness-design.md only if behavior wording changes, and the stage fix/test reports. Forbidden: compute_diff_fingerprint changes, class-2 exceptions, identity weakening, RC1/RC2/RC3 work, funding_hedging changes, unrelated refactors, merge, push, or silent deletion of the watchdog draft.\n\nRequired adversarial evidence:\n- owner=claude_glm plus review.provider=zhipu_glm must FAIL even when review.reviewer is an arbitrary unregistered label;\n- a valid cross-provider provider field must PASS;\n- recognized reviewer/provider disagreement must FAIL;\n- evidence_file pointing to a directory such as reports with a matching digest must FAIL;\n- committed symlink and submodule evidence must FAIL;\n- a regular committed non-empty blob with a matching digest must PASS;\n- digest mismatch, absolute path, escape path, untracked file, empty blob, malformed reason/at, class-2 verdict, and identity negative-list tests must remain FAIL;\n- prior task-banner, canonical-scope, unique-ID, own-review fingerprint, template-default, and zero-regression cases must remain green.\n\nRun: python3 -m py_compile scripts/validate-stage.py; the archived construction/adversarial suite plus the new cases; python3 scripts/validate-stage.py 2026-07-harness-authorized-exception-v1 --phase checkpoint. After the bookkeeper creates the new committed head, run python3 scripts/validate-stage.py 2026-07-harness-authorized-exception-v1 --phase pre-review, git diff --check <base>..<new-head>, independently recompute the fingerprint, verify git diff --quiet <base>..<new-head> -- docs/planning/adapter-watchdog-runner.md, and confirm the fresh review-2 dispatch contains the new head/fingerprint. Record finding-to-fix mapping in 40-fix-report.md and full raw output in 60-test-output.txt.",
  "next_action": "fix"
}
