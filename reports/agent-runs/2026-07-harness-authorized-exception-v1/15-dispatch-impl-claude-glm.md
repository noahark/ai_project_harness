# Dispatch Packet — Stage A 实现（executor: human operator → claude_glm）

⚠️ **在模板仓 `ai_project_harness` 执行**（不是 funding_hedging）。改的是 harness_owned
的 `validate-stage.py` + schema/template + 说明书。当前分支
`stage/2026-07-harness-authorized-exception-v1`（base `00e25b4`）。

操作者在 Claude-GLM 终端执行 PROMPT BODY。实现者产出后写
`reports/agent-runs/2026-07-harness-authorized-exception-v1/20-implementation.md` +
`60-test-output.txt`，**不 commit**（bookkeeper 提交、算指纹、跑 validator、派 review）。

`claude-glm -p "$(cat reports/agent-runs/2026-07-harness-authorized-exception-v1/15-dispatch-impl-claude-glm.md)"`

---

## PROMPT BODY

你是 Stage A 的实现者（`claude_glm`，validator/schema 域）。**当前工作目录必须是模板仓
`ai_project_harness`。** 给 `scripts/validate-stage.py` 增加 RC4 修法：authorized_exception
+ 分任务指纹覆盖。**这是证据完整性主干,写宽一行就可能毁掉整个信任基础——严格照规格,不
自由发挥。** 不 commit。

先读（规格与约束,逐条落实）：
- `reports/agent-runs/2026-07-harness-authorized-exception-v1/10-design.md`（机制全文）
- `reports/agent-runs/2026-07-harness-authorized-exception-v1/12-development-breakdown.md`
  （validator 逐条改动 A–E、禁改边界、风险点）
- `reports/agent-runs/2026-07-harness-authorized-exception-v1/11-adr.md`
- 现有代码锚点：`scripts/validate-stage.py` 的 `validate_acceptance()`(~:791-806)、
  `validate_tasks()`(~:749-787)、`compute_diff_fingerprint()`(~:160-176)、
  `evidence_path_exists()`、`validate_review_identity()`(:701-746)。

改这些文件（仅这些）：
1. `scripts/validate-stage.py`：按 breakdown §A–§E 实现——
   - §A 常量 `AUTHORIZED_EXCEPTION_ASSERTION_IDS = {"review_fingerprint_trails_status"}`。
   - §B 新增 `validate_authorized_exceptions()`：校验每条例外(assertion_id 命中枚举、
     authorizer=="user" 字面量、applies_to_fingerprint==status.diff_fingerprint【钉指纹】、
     evidence_file 存在且非空、scope 非空)。任一不满足 → error（**fail-closed,这些 error
     不可被任何例外降级**）。
   - §C 改 `validate_acceptance()`：**只有** `review.diff_fingerprint != status.diff_
     fingerprint` 这一条,且在 §B 全部合规、存在命中该 review scope 的 class-1 例外时,
     才降级为"applied exception"不计 error。**`verdict!=ACCEPT` / `json_schema_valid` /
     `tests.status` / 身份分离 一律不降级**（class-2 不收 + negative list）。
   - §D 新增 `validate_task_coverage()`：tasks 缺省→return []（退化,零改变）；否则做链式
     (base 首尾相接、末段==status.head)+ 前缀(review 指纹==重算 base..task[j].head)；j 之后
     任务需自审或命中 task scope 的 class-1 例外。pre-accept 阶段调用。
   - §E main() pre-accept 打印 `PASS (N authorized exceptions applied: <id>@<scope>, …)`,
     exit 仍 0；无例外时输出同现行。
2. `reports/agent-runs/_template/status.json`：加 `"authorized_exceptions": []`;`tasks[]`
   示例加 `"covers_through_task": null`。
3. `AGENTS.md` Hard Gates：增补 pre-accept 例外机制段（白名单在 validator **源码枚举**、
   negative list 五项永不可豁免、钉指纹自动失效、放行非静默、数据只能援引不能定义例外类）。
   **不改两仓 carve-out 行。**
4. `docs/harness-design.md` pre-accept 段：同步同上描述。

**Negative list（永不可豁免,写进代码注释 + 说明书）**：status.diff_fingerprint 重算一致、
clean worktree、reviewer 身份分离(:717-718)、例外记录自身 evidence_file 存在性、例外记录
自身结构完整性。**豁免机制不能豁免自己。**

绝对禁改：`compute_diff_fingerprint()` 公式；`:717-718` 身份逻辑；validator 无关重构；
RC1/RC2/RC3 相关（Stage B+）；任何 funding_hedging 仓文件；AGENTS.md 两仓 carve-out 行。

完成后：
- 写 `20-implementation.md`（改了哪些函数/行、每处对应 breakdown 哪条、negative-list 如何
  保证不可豁免、钉指纹如何实现自动失效）。
- 写 `60-test-output.txt`，用**构造样例**证明这 7 条(可写成临时 pytest 或 python 脚本,
  跑完贴输出;临时脚本别留在交付里)：
  1. 合规 class-1 例外(evidence 存在+指纹命中) → 原本 fingerprint-mismatch 的 pre-accept
     从 FAIL → PASS,且 stdout 打印 exception-applied 行。
  2. 去掉 evidence_file → 仍 FAIL。
  3. 改 applies_to_fingerprint 不命中当前指纹 → 仍 FAIL（钉指纹生效）。
  4. authorizer 非 "user" → 仍 FAIL。
  5. 给 negative-list 断言(如伪造 status.diff_fingerprint 使重算不一致)挂例外记录 → 仍 FAIL。
  6. D3 链式:review-1 覆盖 task A/B(前缀)、task C 挂 class-1 例外 → PASS;链断裂 → FAIL。
  7. 退化:tasks 缺省 / 单任务 → 行为与现行完全一致（现有断言不变）。
- 另跑一遍现有 validator 自检/测试(若仓内有)确认零回归;并**用改后的脚本对本 stage 自己**
  跑 `python3 scripts/validate-stage.py 2026-07-harness-authorized-exception-v1 --phase
  pre-review` 确认仍正常（bootstrapping）。
- 报告用统一 footer + provider-native Session ID（或 unavailable+原因）。**不要 commit。**

## END PROMPT BODY
