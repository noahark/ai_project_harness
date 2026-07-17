# Implementation Report — Stage A: authorized_exception + 分任务指纹覆盖(RC4)

实现者:`claude_glm`(validator/schema/backend 域)。仅改模板仓 4 个文件,落实
`12-development-breakdown.md` §A–§E。`compute_diff_fingerprint()` 公式与
身份分离(`:717-718` 等价逻辑)未动。**不 commit**(bookkeeper 提交)。

## 1. 改动文件清单(仅这 4 个,与 dispatch 一致)

| 文件 | 性质 |
|---|---|
| `scripts/validate-stage.py` | 代码:常量 + 4 个新函数 + 改写 `validate_acceptance` + 改 `main` |
| `reports/agent-runs/_template/status.json` | 模板:`authorized_exceptions: []` + `covers_through_task` |
| `AGENTS.md` | 说明书:Hard Gates 增补 2 条(例外机制 + negative list) |
| `docs/harness-design.md` | 说明书:新增 "Authorized Exceptions (RC4)" 段 |

`git diff --stat`:4 文件,+368/−4。

绝对未触碰:`compute_diff_fingerprint()` 公式、身份分离逻辑、validator 无关重构、
RC1/RC2/RC3(Stage B+)、任何 `funding_hedging` 仓文件、AGENTS.md 两仓 carve-out 行。

## 2. 逐条改动(函数 → breakdown 条目,行号为改后 `validate-stage.py`)

### §A 常量 — `:84`
```python
AUTHORIZED_EXCEPTION_ASSERTION_IDS = {"review_fingerprint_trails_status"}
```
源码枚举的 v1 白名单(class-1 唯一类)。常量上方注释写明 negative list 五项
(镜像到 AGENTS.md / harness-design.md)。stage 数据只能**援引**例外类,不能**定义**。

### §B `validate_authorized_exceptions()` — `:847`(+helper `:813`/`:829`)
- 签名 `(root, stage_dir, status_doc) -> tuple[list[str], list[dict]]`。
- 逐条校验:`assertion_id ∈ AUTHORIZED_EXCEPTION_ASSERTION_IDS`、
  `authorizer == "user"`(字面量)、`applies_to_fingerprint == status.diff_fingerprint`(钉指纹)、
  `evidence_file` 存在且**非空**(新增 `_evidence_file_nonempty()` `:813`,补齐
  `evidence_path_exists` 只查存在、不查内容的缺口)、`scope` 为非空字符串。
- 任一不满足 → append error;**fail-closed**:`if errors: return errors, []`,
  一条坏记录使**全部**例外作废。
- `_resolve_task()` `:829`:把 `covers_through_task` 解析为 task(支持 int 索引 / str id)。

### §C 改写 `validate_acceptance()` — `:1050`(+helper `:930`)
- 签名改为 `(root, stage_dir, status_doc) -> tuple[list[str], list[dict]]`(errors, applied_exceptions)。
- 先调 §B 拿 `valid_exceptions`(B 有 error 则为空集)。
- 对 `review_1`/`review_2`:**只有** `review.diff_fingerprint != status.diff_fingerprint`
  这一条断言可降级——当 `_exception_covering_review()` `:930` 命中一条合规例外
  (`assertion_id == "review_fingerprint_trails_status"` 且 `scope == 该 review`)时,
  不计 error、记进 `applied_exceptions`。
- `verdict != ACCEPT` / `json_schema_valid` / `tests.status` / 身份分离
  (`validate_review_identity`) / task 指纹(`validate_tasks`)**一律无降级分支**——
  代码上根本不存在例外路径(negative list + class-2 不收)。

### §D `validate_task_coverage()` — `:948`(D3)
- `tasks` 缺省或单任务(`len(tasks) <= 1`)→ 直接 `return []`(退化,现行行为零改变)。
- 多任务:**链式**(`tasks[0].base == status.base`、`tasks[i+1].base == tasks[i].head`、
  末段 `head == status.head`);**前缀**(每道 `review.covers_through_task == j` 的
  `review.diff_fingerprint == 重算 base..tasks[j].head`);j 之后每个 task 需自身
  `review_1` 或一条 scope 命中(`task:<id>` / `<id>`)的合规 class-1 例外,否则
  `uncovered task <id>`。
- 仅 pre-accept 调用,在 `validate_acceptance` 之后(`main` 内)。

### §E `main()` stdout — `:1096` 区域
- pre-accept PASS 且 `applied_exceptions` 非空时追加一行:
  `PASS (N authorized exceptions applied: <id>@<scope>, …)`;exit 仍 0。
- 无例外时输出与现行完全一致(不追加该行)。

## 3. Negative list 如何保证不可豁免(代码结构,不靠自觉)

"豁免机制不能豁免自己"由**降级路径的物理隔离**保证:

| # | 断言 | 生效位置 | 为何不可豁免 |
|---|---|---|---|
| 1 | `status.diff_fingerprint` 重算一致 | `validate_common` | 该 error 在 `validate_common` 产生;`validate_acceptance` 的降级逻辑只作用于 `review.diff_fingerprint` 断言,接触不到它。测试用例 5 证明:即便例外 `applies_to` 钉的是篡改值、记录本身自洽,仍只剩这一条 mismatch。 |
| 2 | clean worktree | `require_clean_worktree`(`main` 最早执行) | 在所有 acceptance 逻辑之前;无降级路径。 |
| 3 | 身份分离(`:717-718` 等价) | `validate_review_identity` | 例外机制从不调用它;它照常 append error。 |
| 4 | 例外记录自身 `evidence_file` 存在性 | `validate_authorized_exceptions` | error 直接进 errors;且 fail-closed 使 `valid_exceptions` 清空。 |
| 5 | 例外记录自身结构完整 | `validate_authorized_exceptions` | 同上。 |

关键不变式:**降级只发生在 `validate_acceptance` 内、只对单一断言、且只在 §B 全合规
(`valid_exceptions` 非空)之后**。§B 任一 error → `valid_exceptions == []` →
`_exception_covering_review` 必返回 `None` → 该 review 照常报 mismatch。fail-closed。

## 4. 钉指纹如何实现自动失效

`applies_to_fingerprint == status.diff_fingerprint`(§B 校验)。一旦进入下一轮 fix,
`head_sha` 变 → `status.diff_fingerprint` 变 → 旧记录的 `applies_to_fingerprint` 不再相等 →
§B 报 `applies_to_fingerprint must equal current status.diff_fingerprint` → fail-closed →
例外失效 → `review.diff_fingerprint` 断言重新变红,必须重新授权。无 `expiry` 字段:
钉指纹天然一次性。测试用例 3 证明 `applies_to` 不命中即 FAIL。这把 RC4 的"永久假红"
换成"一次豁免",而不是更糟的"永久假绿"。

## 5. 规格冲突说明(`covers_through_task` 落点)

dispatch 第二步字面写"tasks[] 示例加 `covers_through_task: null`",而 `10-design` §3.2
与 `12-breakdown` §D 的**逻辑规格**为 `review_k.covers_through_task`(字段在 review 上,
值是 task id/索引,用于前缀比对)。本实现以逻辑规格为准:

- **validator 实际读 `review_1`/`review_2.covers_through_task`**(`validate_task_coverage`)。
- `_template/status.json` 在 `review_1`、`review_2` 各加 `covers_through_task: null`(生效字段);
  同时按 dispatch 字面在 `tasks[]` 示例项也加 `covers_through_task: null`(说明键,
  validator 不读,仅人类可读反向标注)。后者若 review 认为冗余可删,不影响逻辑。

## 6. Bootstrapping 与回归

详见 `60-test-output.txt`。改后脚本:对本 stage 自身 `pre-review` 不崩(FAIL 仅因
clean worktree + `status=implementing` + 缺文件,均为现行行为);`checkpoint` PASS;
7 条构造样例 + 零回归 + 退化全部符合预期。

---
本地北京时间: 2026-07-17 11:09:19 CST
下一步模型: bookkeeper(Claude Opus 4.8)
下一步任务: 提交本 stage 4 文件改动、重算 `diff_fingerprint`、跑 validator `pre-accept`、派 review-1(Kimi)
