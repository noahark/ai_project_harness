# Stage A rework-1 Fix Report — executor: claude_glm

- **仓库 / 分支**: `ai_project_harness` / `stage/2026-07-harness-authorized-exception-v1`（rework_count=1/3）
- **修复者**: `claude_glm`（provider=`zhipu_glm`，provider 隔离满足：≠ Codex/openai ≠ Kimi/moonshot）
- **不 commit**（由 bookkeeper 提交、计算指纹、跑 validator、分发 review）。
- **对齐基线**: review-2 diff_fingerprint
  `941416345c92bf3c42556e0d9178f28e1384b4e6:5b28d88ce253d6d41f9c5841cfe51265a66e132c8fe7556d1177da4404b186f0`。
  rework-1 未触碰被 review 的 4 文件之外的任何被 review 语义，checkpoint 仍 PASS 于同一指纹（见 §4）。

---

## 0. 概述

review-2（Codex）REWORK 的 **4 P1 + 2 P2** 全部修复，另完成 Q2 文档措辞改写。执行证据：23 个构造用例全部符合预期（原 10 用例零回归 + 13 个 rework-1 对抗用例），class-2 / 身份 negative-list 守住，py_compile OK，本 stage 自身 checkpoint PASS（bootstrapping）。

改动文件（仅允许范围内 4 个 + 本报告 + 60 追加）：
- `scripts/validate-stage.py`（核心）
- `reports/agent-runs/_template/status.json`
- `AGENTS.md`
- `docs/harness-design.md`
- `40-fix-report.md`（本报告，新增）
- `60-test-output.txt`（追加 E 段，原 A–D 证据保留）

下方逐条映射 review-2 finding → 改了哪些函数/行 → 如何满足 → 证据用例。

---

## 1. finding → 修复映射

### finding-1（P1）—— task 例外静默放行（违反 D2「放行绝不静默」）

**问题**：`validate_task_coverage` 只返回 errors；靠 task 例外 PASS 时 `applied_exceptions` 为空 → 不打印例外行 → 静默放行。

**修法**：
- `validate_task_coverage`（:1114）签名从 `-> list[str]` 改为 `-> tuple[list[str], list[dict[str, str]]]`，**返回它实际依赖的 task 例外**（`applied`）。
- 新增 `_exception_covering_task(valid_exceptions, task_id)`（:1097）：仅匹配规范 `scope == f"task:{task_id}"` 且 `assertion_id == "review_fingerprint_trails_status"`，把命中的 task 例外加入 `applied`。
- 新增 `_merge_applied(*streams)`（:1273）：按 `(assertion_id, scope)` 去重保序，合并 `validate_acceptance` 返回的 review 例外与 `validate_task_coverage` 返回的 task 例外。
- `main` pre-accept 块：`applied_exceptions = _merge_applied(acc_applied, cov_applied)`，PASS 横幅打印**全部** applied `assertion_id@scope`（含 task scope）。

**证据**（60 E 段）：F1 仅靠 task 例外 PASS → 横幅 `PASS (1 authorized exceptions applied: ...@task:B)`；6a → 横幅 N 从 v1 的 1 升为 2，含 `@task:C`。

### finding-2（P1）—— 任意非空串冒充 task own-review

**问题**：`has_own_review` 只查 `diff_fingerprint` 非空串即可放行。

**修法**：新增 `_task_own_review_covers(root, stage_dir, task) -> tuple[bool, str | None]`（:1053），返回 `(covered, error)`：
- 无声明 own-review（非 dict / 无 `diff_fingerprint`）→ `(False, None)`，回退到 authorized_exception 路径。
- 声明的 own-review **用于覆盖当且仅当完整合规**：具名 `reviewer` / `provider` / `model`（非空字符串）、`verdict == "ACCEPT"`、`json_schema_valid` truthy、与 task owner/implementer **跨 provider 隔离**（:1081）、且 `review.diff_fingerprint == 重算的 task 指纹`（`compute_diff_fingerprint(task.base, task.head)`）。
- 不完整 → `error` 被设置 → **FAIL，authorized_exception 无法救**。
- `validate_task_coverage` 对覆盖范围之外的每个 task 调用本函数。

**证据**：F2 `{"review_1":{"diff_fingerprint":"x"}}` → FAIL（`task B review_1.reviewer must be a non-empty string`）；F1 的 task A 提供完整合规跨 provider own review（codex vs claude_glm）→ 覆盖 PASS。

### finding-3（P1）—— 授权来源硬化（Fable5 A 档 + C 边界，§2.1）

**修法**（全部在 `validate_authorized_exceptions` 内，**不动共享 `evidence_path_exists`:387**）：新增 `_validate_exception_evidence(root, evidence_file, record, head_sha, prefix)`（:875），五条机械硬化：
1. **仓内相对**：拒绝绝对路径（:888 `must be repo-relative, got absolute path`）。
2. **逃逸防护**：`(root / evidence_file).resolve()` 必须仍在 `root.resolve()` 之内（:893 `escapes repository root`）。
3. **必须 git-tracked + 读提交内容**：`_evidence_committed_blob`（:854）用 `git show <验证时 HEAD>:<evidence_file>` 读**提交 blob**（不是工作树）；`git show` 失败/未 tracked → FAIL（:898 `not committed/tracked`）。
4. **blob 非空**（:901 `committed blob must be non-empty`）。
5. **digest 封存**：例外记录**新增 `evidence_sha256`**；validator 重算提交 blob 的 sha256 与之比对，不等 → FAIL（:909 `evidence_sha256 mismatch`）。**不绑定 `status.head_sha`**（死锁修正见 §2）。
- `head_sha` 在 `validate_authorized_exceptions`（:932）内取 `git rev-parse HEAD`（pre-accept 已要求 clean worktree）；try/except → `""`（fail-closed）。
- 删 `_evidence_file_nonempty` 旧的工作树 absolute-candidate 分支；RC4 committed-blob 逻辑自包含在 `_evidence_committed_blob` / `_validate_exception_evidence`，**不触碰共享 `evidence_path_exists`:387**（它仍服务其它 caller）。

**证据**：F3-abs（绝对路径）/ F3-esc（`../` 逃逸）/ F3-dig（digest 不匹配）/ F3-empty（空 blob）→ FAIL；合法路径（用例 1 / 6a / F1）的例外均为 **post-head 提交、tracked、digest 匹配** → PASS；用例 2（MISSING.md）`git show` 失败 → `not committed/tracked`（证明 validator 读提交 blob、不读工作树）。

### finding-4（P1）—— task scope 歧义 / 不唯一

**修法**：
- 新增 `_task_id_errors(tasks)`（:256）：task `id` **非空且唯一**（重复 → `duplicate task id`；缺失 → `tasks[i].id must be a non-empty string`）。**显式**由 `validate_tasks` / `validate_task_coverage` / `validate_dispatch_ready` 调用，**不**放进 `normalize_tasks`（:236 保持 `return raw, []`）→ checkpoint 经 `validate_parallel_mode` 运行 `normalize_tasks` 时不在继承的模板占位 id=null 任务上触发，**本 stage 自身 checkpoint 仍 PASS**（bootstrapping，design §6）。
- task 例外 scope **只认规范 `task:<id>`**（`_exception_covering_task`:1097 拒绝裸 `<id>` 别名）；消费端**显式**要求 `assertion_id == "review_fingerprint_trails_status"`。

**证据**：F4-raw（裸 scope `"C"` 而非 `"task:C"`）→ `uncovered task C` FAIL；F4-dup（重复 id A）→ FAIL；F4-miss（缺 id）→ FAIL。「一例外覆盖两 task」由 id 唯一性杜绝（重复 id 先 FAIL）。

### P2-1 —— reason / at 结构校验

**修法**：`validate_authorized_exceptions`（:932）增：`reason` 非空字符串（:1008）；`at` 可解析 ISO-8601（经新增 `_valid_iso8601`:840，`'Z'→'+00:00'` 后 `datetime.fromisoformat`，Python 3.9 兼容）（:1011）。违反 → FAIL（同 negative-list #5 fail-closed，不可被例外降级）。

**证据**：P2-r（空 reason）/ P2-a（非 ISO `at`）→ FAIL。

### P2-2 —— 模板注解键

**修法**（`reports/agent-runs/_template/status.json`）：删 `tasks[]` 示例项上的 `covers_through_task`（validator 只读 review 侧）；`authorized_exceptions` 示例改为含**全部字段**的对象（`assertion_id` / `scope` / `applies_to_fingerprint` / `reason` / `authorizer` / `evidence_file` / **`evidence_sha256`** / `at`，全为 `null`）。

### Q2 —— 防自授措辞改写（必做）

把「防自授」从**不可能性声称**改为**防静默 + 强制人核**两层，并标注**人核是流程义务、validator 无法强制人读**；删除「授权原文只能来自用户消息这一结构，不靠自觉」这类过度声称。
- `AGENTS.md`（RC4 段）：列 `reason` / `at` + 仓库相对 git-tracked evidence_file 被 `evidence_sha256` 封存；Q2 两层段；negative-list #4 更新为「已提交、digest 封存的仓库相对文件」，#5 含「reason, at」。
- `docs/harness-design.md`：删过度声称；加 committed blob 读取 + `evidence_sha256` 封存 + post-head 合法性说明 + 专段「防自授是两层，而非不可能性声明」+ 更新的负面清单。
- `validate-stage.py` `validate_authorized_exceptions` 内注释：防自授两层（非静默 + 强制人核）。

---

## 2. §2.1 死锁修正：记录内 `evidence_sha256`，而非 `blob@status.head_sha`

**旧设计陷阱**：若用 `git show <status.head_sha>:<evidence_file>` 取证，则 evidence 必须存在于 `status.head_sha` 那个提交的 tree 里。但授权流程是「实现完成 → `status.head_sha` 固定 → user 事后授权 → 授权 evidence 提交在 head_sha **之后**的新提交里（post-head）」。若绑定 `head_sha`，evidence 永远不可能出现在 `head_sha` 的 tree 中 → 合法授权恒 FAIL，与「post-head 授权」死锁。

**修法**：例外记录内存放 **`evidence_sha256`**（封存预期 digest，user 授权时盖戳）；validator 取**验证时 HEAD**（pre-accept clean worktree 后的 `git rev-parse HEAD`，已包含 evidence 提交）的提交 blob，重算 sha256 与记录内 `evidence_sha256` 比对。这样：
- **post-head 提交的 evidence** 只要 tracked（在 HEAD tree 里可 `git show`）+ digest 匹配 = 合法，不绑定 `status.head_sha`，不死锁。
- **digest 封存防的是事后篡改**：记录内 digest 是授权戳，validator 重算当前提交 blob 的 digest 必须等于它，否则 evidence 被换过 → FAIL（用例 F3-dig 证明）。
- 「必须 git-tracked」+「读提交 blob（不读工作树）」+「digest 封存」三层共同把来源硬化到「一个 user 授权过、且未被篡改的已提交文件」；工作树里的未跟踪副本不可能到达此检查（pre-accept 的 `require_clean_worktree` 先拦截，见 60 E 段 finding-3 映射注）。

---

## 3. 禁改边界遵守（硬边界，逐项核验）

| 禁改项 | 状态 |
|---|---|
| `compute_diff_fingerprint()` 公式（:183） | 未改 |
| 身份分离 `review_1`/`review_2` 全局（:757/:765）、acceptance 内 task `review_1`（:823） | 未改；:1081 是 finding-2 **新增**的 task own-review 维度隔离（规格要求），非改原逻辑 |
| 共享 `evidence_path_exists`:387 的行为 | 未改（仍 absolute-candidate + 工作树 `.exists()`，服务其它 caller）；RC4 committed-blob 逻辑自包含于 `_evidence_committed_blob`/`_validate_exception_evidence` |
| validator 无关重构 | 无 |
| class-2 / RC1 / RC2 / RC3 | 未动 |
| 任何 `funding_hedging` 文件 | 未触碰（`git status` 仅 5 文件，全在 `ai_project_harness`） |
| `AGENTS.md` 两仓 carve-out 行 | 未动 |
| commit / merge / push | 未执行 |

`git status --short`（本报告写入前）：
```
 M AGENTS.md
 M docs/harness-design.md
 M reports/agent-runs/2026-07-harness-authorized-exception-v1/60-test-output.txt
 M reports/agent-runs/_template/status.json
 M scripts/validate-stage.py
```
（写入本报告后追加 `?? 40-fix-report.md`。）

---

## 4. 零回归 + bootstrapping

- **py_compile**：`python3 -m py_compile scripts/validate-stage.py` → `COMPILE_OK`（Python 3.9.6）。
- **checkpoint（bootstrapping）**：`python3 scripts/validate-stage.py 2026-07-harness-authorized-exception-v1 --phase checkpoint` → `STAGE VALIDATION PASSED`，`diff_fingerprint=941416345c92bf3c42556e0d9178f28e1384b4e6:5b28d88ce253d6d41f9c5841cfe51265a66e132c8fe7556d1177da4404b186f0`（== review-2 指纹，无回归）。
- **原 10 构造用例**（1/2/3/4/5/6a/6b/7a/7b/REG）行为符合预期；用例 2 的 message 由 v1 的 "does not exist" 精化为 "not committed/tracked"（语义升级，仍 FAIL，非回归）。
- **negative-list 守住**：CL2（`review_2.verdict==REWORK` 即便带例外仍 FAIL，verdict 不降级）；ID（`review_2` 同 provider 仍 FAIL，身份分离不降级）。
- **退化不变**：单任务 / tasks 缺省 → PASS（用例 7a/7b）。

23 用例完整命令、输出、finding→用例映射、临时脚手架源码见 `60-test-output.txt` E 段（临时脚本 `/tmp/rc4_rework_test.py` 不留在交付树）。

---

## 5. 交付物清单

| 文件 | 状态 | 内容 |
|---|---|---|
| `scripts/validate-stage.py` | M | finding-1~4 + P2-1 + Q2 注释 |
| `reports/agent-runs/_template/status.json` | M | P2-2 模板键（删 covers_through_task）；R4：authorized_exceptions 回 [] |
| `AGENTS.md` | M | RC4 段 + Q2 + 负面清单 |
| `docs/harness-design.md` | M | §2.1 committed-blob + Q2 + 负面清单 |
| `40-fix-report.md` | ?? | 本报告 |
| `60-test-output.txt` | M | 追加 E 段（原 A–D 保留） |

---

## 6. R4 补丁（bookkeeper R4 对账后的两处模板默认值修正）

rework-1 首版（§1–§5）经 bookkeeper R4 对账，发现两处**模板默认值回归**——E 段 23 用例都自造 status.json，未测模板默认路径，漏了。续修两点，不动 §1–§5 已修好的部分：

- **R-A**（细化 §1 P2-2）：`reports/agent-runs/_template/status.json` 的 `authorized_exceptions` 由全-null 占位记录**改回空数组 `[]`**。根因是 35-dispatch「示例项加 evidence_sha256」措辞误导——模板原本是空数组、无示例项；全-null 占位会让任何用模板默认值的 stage 在 pre-accept 报 7 条 `authorized_exceptions[0].*` 错。记录字段形状（含 `evidence_sha256`）已在 10-design §2 JSON 块 + AGENTS.md/harness-design.md 文档化，模板不带活示例（JSON 无注释）。**保留** P2-2 对 `tasks[]` 删 `covers_through_task` 的改动。
- **R-B**（细化 §1 finding-4）：`_task_id_errors`（:256）开头加 `if len(tasks) < 2: return []`（并更新 docstring）。id 只在 task-scope 覆盖/dispatch（>=2 真任务）时有意义；单任务退化/占位（模板与本 stage status.json 都带 null-id 占位）不再误红 pre-accept。>=2 仍强制 id 非空唯一（finding-4 不削弱；`validate_dispatch_ready` 本就要求 >=2 task objects）。

**证据**（`60-test-output.txt` E.1 段，25 用例全过）：套件重建为 25 用例——
- `F4-miss` 由单任务改为**多任务**（2 任务，第二个缺 id）→ FAIL（`tasks[1].id must be a non-empty string`），证明 >=2 时 id 强制不削弱；
- 新增 `F4-sgl`（单任务无 id）→ PASS，证明 R4 <2 豁免；
- 新增 `TPL`（加载模板默认值：null-id 占位 task + `authorized_exceptions=[]`）→ `validate_tasks`/`validate_authorized_exceptions` 双双零 error，证明 R-A + R-B 组合使模板默认不再破 pre-accept。
- 其余 22 用例与 E 段一致，零回归；class-2 / 身份 negative-list 守住。

**禁改边界**：R-A/R-B 仅触碰 `_template/status.json`（authorized_exceptions 回 []）与 `_task_id_errors`（加 <2 短路），未动 `compute_diff_fingerprint`、身份分离、共享 `evidence_path_exists`:387、class-2/RC1-3、AGENTS.md carve-out 行；`funding_hedging` 未触碰；未 commit。py_compile OK；checkpoint PASS（`diff_fingerprint=941416345...:5b28d88...`，与 review-2 一致）。

---

模型身份: claude_glm（zhipu_glm / glm-5.2；Claude Code CLI 桥接；session id 未暴露 — zhipu_glm 运行环境未提供原生 session id API）
本地北京时间: 2026-07-17 14:41:23 CST
下一步模型: bookkeeper（Claude Opus 4.8）—— 提交 rework-1 改动（4 文件 + 本报告 + 60 追加；含 R4 两处模板默认值修正）、重算 diff_fingerprint、跑 validator pre-accept、重进 review-1（Kimi）→ review-2（Codex）
