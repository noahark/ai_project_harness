# Development Breakdown — Stage A 实现

Owner：`claude_glm`（validator/schema/backend 域）。Breakdown 作者：bookkeeper
（Claude Opus 4.8）——作为 review-2 design-involvement 披露。单实现者、非并行。

## 逐文件规格（模板仓内）

| # | 文件 | 改什么 | 依据 |
|---|---|---|---|
| 1 | `scripts/validate-stage.py` | 见下"validator 改动"逐条 | 10-design §4 |
| 2 | `reports/agent-runs/_template/status.json` | 加 `"authorized_exceptions": []`;`tasks[]` 示例项加 `"covers_through_task": null` 说明键 | D2/D3 |
| 3 | `AGENTS.md` | Hard Gates 段增补:pre-accept 例外机制(白名单源码枚举 + negative list + 钉指纹 + 非静默),不改两仓 carve-out 行 | 说明书同步 |
| 4 | `docs/harness-design.md` | pre-accept 段增补同上机制描述 | 说明书同步 |
| 5 | 新增测试(若仓内有 validator 测试目录则加;否则在 60-test-output 用构造样例证明) | 见 Acceptance | bootstrapping |

## validator 改动（逐条,小 diff）

### A. 常量
- 加 `AUTHORIZED_EXCEPTION_ASSERTION_IDS = {"review_fingerprint_trails_status"}`（v1 白名单,
  源码枚举）。

### B. 新增 `validate_authorized_exceptions(root, stage_dir, status_doc) -> list[str]`
对 `status_doc.get("authorized_exceptions", [])` 每条校验(任一不满足即 append error,
**这些 error 属 negative-list #4/#5,不可被豁免**)：
- `assertion_id ∈ AUTHORIZED_EXCEPTION_ASSERTION_IDS`（不在枚举 → error）
- `authorizer == "user"`（字面量,其它值 → error）
- `applies_to_fingerprint == status_doc["diff_fingerprint"]`（不命中当前指纹 → error;
  这实现"钉指纹/自动失效"）
- `evidence_file` 存在(用现有 `evidence_path_exists(root, stage_dir, evidence_file)`);
  且文件内容非空(读文件,含用户授权原文的最低证据——至少非空;不做 NLP 校验)
- `scope` 为非空字符串(如 `review_1` / `task:<id>`)
返回该条是否"合规可用"给 C 使用（建议返回 `(errors, valid_exceptions_by_assertion)`）。

### C. 改 `validate_acceptance()`（:791-806）
- **仅** `review.diff_fingerprint != status.diff_fingerprint` 这条断言可被 class-1 例外
  降级:当且仅当存在一条**合规**(经 B 校验通过) `assertion_id=="review_fingerprint_trails_status"`、
  `scope` 命中该 review 的例外时,把该断言从 error 降级为"applied exception"记录,不计入
  errors。
- **不动** `verdict != ACCEPT`、`json_schema_valid`、`tests.status`、身份分离——这些即便
  有例外记录也照常 fail（class-2 不收 + negative list）。
- 先调用 B（`validate_authorized_exceptions`）；B 有 error 则例外全部无效(fail-closed)。
- 收集 applied exceptions,供 main() 打印。

### D. 新增 `validate_task_coverage(root, stage_dir, status_doc) -> list[str]`（D3）
- 若 `status_doc.get("tasks")` 为空/缺省 → 直接 return []（退化到现行,零改变）。
- 否则:
  - 链式:`tasks[0].base_sha == status.base_sha`;`tasks[i+1].base_sha == tasks[i].head_sha`;
    `tasks[-1].head_sha == status.head_sha`。任一断裂 → error。
  - 前缀:对每道有 `covers_through_task=j` 的 review,校验
    `review.diff_fingerprint == compute_diff_fingerprint(root, stage_dir, status.base_sha, tasks[j].head_sha)`。
    不等 → error。
  - j 之后每个任务:必须有自身 review 记录 **或** 一条命中该 task scope 的合规 class-1 例外;
    否则 → error（"uncovered task <id>"）。
- 在 pre-accept 阶段调用（main() 里 `phase=="pre-accept"` 分支,`validate_acceptance` 之后）。

### E. main() stdout（非静默）
- pre-accept PASS 且有 applied exceptions 时,打印:
  `PASS (N authorized exceptions applied: <assertion_id>@<scope>, …)`;exit 仍 0。
- 无例外时输出与现行一致。

## 禁改（硬边界）
- `compute_diff_fingerprint()` 指纹公式**一字不改**。
- 身份分离 `:717-718` 逻辑不放宽。
- 不重构 validator 无关部分;不做 RC1/RC2/RC3(Stage B+)。
- 任何 `funding_hedging` 仓文件（cp 是接受后独立动作）。
- AGENTS.md 两仓 carve-out 行不得覆盖。

## Review Focus
- review-1(Kimi)：逐条核对 negative-list 五项确实不可豁免;class-2 确实未被放宽;钉指纹
  真能"下一轮自动失效";退化(单任务)行为零改变;边界零越界。
- review-2(Codex,证据完整性主干严审)：例外机制会不会被绕过自授;`validate_authorized_
  exceptions` 的 error 是否真的 fail-closed（B 挂了 C 不能放行）；链式+前缀是否等价覆盖
  base..head 且无 diff 代数漏洞。

## 风险点
- **最危险**:降级逻辑写宽了,把 negative-list 断言也降级 → 直接毁掉信任地基。C 必须只对
  单一断言、且经 B 全合规后才降级。
- 钉指纹漏了 → 变永久假绿。B 里 `applies_to_fingerprint==status.diff_fingerprint` 是这条
  的唯一保证,必须有测试。

当前 Session ID: unavailable
原始输出路径: reports/agent-runs/2026-07-harness-authorized-exception-v1/12-development-breakdown.md
本地北京时间: 2026-07-17 01:15:00 CST
下一步模型: claude_glm（经操作者派发）
下一步任务: 实现 validator 改动 + _template + 说明书;写 20-implementation + 60-test-output
