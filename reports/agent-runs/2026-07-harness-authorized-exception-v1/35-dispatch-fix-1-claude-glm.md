# Dispatch Packet — Stage A rework-1 fix（executor: claude_glm）

⚠️ **模板仓 `ai_project_harness`**,分支 `stage/2026-07-harness-authorized-exception-v1`
(rework_count=1/3)。修 review-2(Codex)REWORK 的 4 P1 + 2 P2,finding-3 按 Fable5
DIRECTION-SET 裁决。**你是 fix 作者(claude_glm/zhipu_glm),不 commit**(bookkeeper 提交)。
provider 隔离:fix 作者 ≠ Codex(openai)≠ Kimi(moonshot),满足;修完**重进 review-1(Kimi)
→ review-2(Codex)**。

---

## PROMPT BODY

你是 Stage A 的 rework-1 fix 作者(`claude_glm`)。**当前目录必须是模板仓
`ai_project_harness`。** 修 review-2 的 6 条 finding。**这是证据完整性主干,写宽一行毁掉信任
地基——严格照规格,不自由发挥。** 不 commit。

先读(逐条规格):
- `reports/agent-runs/2026-07-harness-authorized-exception-v1/50-review-2.md`
  (Codex 6 finding 全文 + `fix_start_prompt`)
- `.../56-direction-ruling-finding3-fable5.md`(finding-3 的 Fable5 裁决,**权威**)
- `.../10-design.md` §2、**§2.1**(finding-3 机械硬化四条)、§7(对抗清单)
- 代码:`scripts/validate-stage.py` 的 `validate_authorized_exceptions()`、
  `_evidence_file_nonempty()`(:813)、`evidence_path_exists()`(:363,**共享,别改**)、
  `validate_task_coverage()`、`validate_acceptance()`、`main()`。

改这些文件(仅这些):`scripts/validate-stage.py`、`reports/agent-runs/_template/status.json`、
`AGENTS.md`、`docs/harness-design.md`。

### finding-1(P1)—— task 例外静默放行
`validate_task_coverage` 现只返回 errors;靠 task 例外 PASS 时 `applied_exceptions` 为空 →
不打印例外行 → 违反 D2「放行绝不静默」。修:让 `validate_task_coverage` **返回它实际依赖的
task 例外**;在 `main` 里与 `validate_acceptance` 的 review 例外**合并去重**,`main` 打印
**全部** applied `assertion_id@scope`(含 task scope)。**仅靠 task 例外 PASS 也必须打印
PASS-with-exception 横幅。**

### finding-2(P1)—— 任意非空串冒充 task own-review
`has_own_review` 现只查 `diff_fingerprint` 非空串。修:task own-review 用于覆盖**当且仅当**
它完整合规——有具名 reviewer/provider/model、`verdict=="ACCEPT"`、`json_schema_valid==true`、
与 task owner/implementer **跨 provider 隔离**、且 `review.diff_fingerprint == 重算的
task.diff_fingerprint`。任意非空串 / 不完整 review 对象 → FAIL。

### finding-3(P1)—— 授权来源硬化（Fable5 A 档 + C 边界,§2.1）
在 `validate_authorized_exceptions` 内(**不动共享 `evidence_path_exists`:363**):
1. **仓内相对 + 逃逸防护**:`evidence_file` 拒绝绝对路径;`(root/evidence_file).resolve()`
   必须仍在 `root.resolve()` 之内(防 `../` 逃逸)。删 `_evidence_file_nonempty` `:819` 的
   absolute-candidate 分支。
2. **必须 git-tracked + 读提交内容**:用 `git show <验证时 HEAD>:<evidence_file>` 读
   **提交 blob**(不是工作树)。未 tracked / `git show` 失败 → FAIL。仍要求 blob 非空。
   (验证时 HEAD:pre-accept 已要求 clean worktree,取当前 `git rev-parse HEAD`。)
3. **digest 封存**:例外记录**新增 `evidence_sha256`** 字段;validator 重算提交 blob 的
   sha256 与之比对,不等 → FAIL。**不要**用 blob@`status.head_sha`(与 post-head 授权死锁,
   见 §2.1)。**adversarial 改**:旧"post-head evidence fails" → "digest 不匹配 fails /
   untracked fails";**post-head 但 tracked + digest 匹配 = 合法**。
4. task 消费端已在 finding-4 按 `assertion_id` 过滤。
这些 error 属 negative-list #4/#5(fail-closed,不可被任何例外降级)。

### finding-4(P1)—— task scope 歧义/不唯一
`normalize_tasks`/`validate_tasks`:要求 task `id` **非空且唯一**(重复/缺失 → FAIL)。
task 例外 scope **只认规范 `task:<id>`**(拒绝裸 `<id>` 别名);task 消费端**显式**要求
`assertion_id == "review_fingerprint_trails_status"`。拒绝歧义/重复/跨 scope/无法解析的 scope。

### P2-1 —— reason/at 结构校验
`validate_authorized_exceptions` 增:`reason` 非空字符串;`at` 可解析 ISO-8601。违反 → FAIL
(同 negative-list #5 fail-closed)。

### P2-2 —— 模板注解键
删 `reports/agent-runs/_template/status.json` 里 `tasks[]` 示例项上的 `covers_through_task`
键(validator 只读 review 侧);`authorized_exceptions` 示例项**加 `evidence_sha256`**。

### Q2 文档改写（必做,措辞见 56 文件 / 10-design §2）
`AGENTS.md`、`docs/harness-design.md`、以及 `validate-stage.py` `:886-888` 的 negative-list
注释:把"防自授"从**不可能性声称**改为**防静默 + 强制人核**两层,并标注**人核是流程义务、
validator 无法强制人读**。删除"授权原文只能来自用户消息这一结构,不靠自觉"这类过度声称。

## 禁改(硬边界)
`compute_diff_fingerprint()` 公式;身份分离 `:717-718`;**共享 `evidence_path_exists`:363
的行为(别的 caller 在用)**;validator 无关重构;class-2;RC1/RC2/RC3;任何 funding_hedging
文件;AGENTS.md 两仓 carve-out 行;不 commit/merge/push。

## 完成后
- 写 `40-fix-report.md`:每条 finding → 改了哪些函数/行、如何满足;**把每条 review-2 finding
  映射到其修法**;§2.1 的死锁修正(evidence_sha256 而非 blob@head_sha)要说明。
- 更新 `60-test-output.txt`(**追加 rework-1 段,别删原证据**):构造样例证明——
  1. finding-1:仅 task 例外 PASS → 横幅打印全部 applied(含 task scope)。
  2. finding-2:`{"review_1":{"diff_fingerprint":"x"}}` → FAIL;完整合规跨 provider + 指纹==
     task 指纹的 task review → PASS。
  3. finding-3:绝对路径 / `../` 逃逸 / untracked / evidence_sha256 不匹配 → FAIL;
     **post-head 但 tracked + digest 匹配 → PASS**;空 blob → FAIL。
  4. finding-4:裸 `<id>` scope / 重复 task id / 缺 id / 一例外覆盖两 task → FAIL。
  5. P2:缺 reason / at 非 ISO-8601 → FAIL。
  6. 零回归:原 10 构造用例 + class-2/身份 negative-list 仍全过;单任务/tasks 缺省退化不变。
  跑 `python3 -m py_compile scripts/validate-stage.py`、原构造脚手架、新对抗用例、
  `python3 scripts/validate-stage.py 2026-07-harness-authorized-exception-v1 --phase checkpoint`;
  命令与输出全贴。临时脚本不留在交付树。
- 报告用统一 footer + provider-native Session ID(或 unavailable+原因)。**不要 commit。**

## END PROMPT BODY
