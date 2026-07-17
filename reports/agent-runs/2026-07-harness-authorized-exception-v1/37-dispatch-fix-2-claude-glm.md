# Dispatch Packet — Stage A rework-2 fix（executor: claude_glm）

⚠️ **模板仓 `ai_project_harness`**,分支 `stage/2026-07-harness-authorized-exception-v1`
(**rework_count=2/3,最后一次返修额度**)。修 review-2 round-2(Codex)的 **2 条代码 P1 +
P3**。#3/#4 是 bookkeeper 的活(看门狗已迁 main、dispatch 已标作废),**不归你**。你是 fix
作者(claude_glm/zhipu_glm),**不 commit**。修完重进 review-1(Kimi)→ review-2(Codex)。

Codex 已确认:round-1 的修复本身成立、Fable5 的 `evidence_sha256`+验证时 HEAD 方案可接受
(**不要改回 blob@status.head_sha**)。本轮只堵两个相邻边界洞。

---

## PROMPT BODY

你是 Stage A 的 rework-2 fix 作者(`claude_glm`)。**目录必须是模板仓 `ai_project_harness`。**
修 review-2 round-2 的 2 条代码 P1 + 1 条 P3。**证据完整性主干,严格照规格。** 不 commit。

先读:
- `reports/agent-runs/2026-07-harness-authorized-exception-v1/50-review-2.md`(Codex round-2
  4 finding 全文 + `fix_start_prompt`;#1/#2 是你的范围,#3/#4 是 bookkeeper 的)
- `.../56-direction-ruling-finding3-fable5.md`(evidence_sha256 方案权威,别改回 blob@head_sha)
- `.../40-fix-report.md`、`.../10-design.md` §2.1
- 代码:`scripts/validate-stage.py` 的 `_task_own_review_covers()`、`review_provider_identity()`、
  `provider_identity()`、`_evidence_committed_blob()`、`_validate_exception_evidence()`。

改这些文件(仅这些):`scripts/validate-stage.py`、`AGENTS.md`(仅 P3 空行 + 若措辞需同步)。

### finding-1（P1）—— task own-review 隔离信 reviewer 标签先于 provider
现状:`_task_own_review_covers` 用 `review_provider_identity(review)` 作 reviewer 身份,而该
helper **先取 reviewer 再取 provider**;于是把 `reviewer` 写成任意未注册串就能盖过真实
`provider`,让 owner=claude_glm / provider=zhipu_glm 的同 provider 自审通过(假绿)。且当 task
无 owner/implementer 时隔离分支被整体跳过。修:
- 在 **task own-review 上下文**用 **`provider` 字段为权威** vendor 身份:
  `reviewer_identity = provider_identity(review.get("provider"))`(不再用 reviewer 标签)。
  **要求 `review.provider` 存在且可解析**,否则该 own-review 不成立 → FAIL。
- **要求 task 有可解析的 owner/implementer provider**;缺失 → 该 own-review 不能建立隔离 →
  FAIL(不再静默跳过)。
- owner provider == reviewer provider → FAIL(跨 provider 隔离)。
- 若 `review.reviewer` 本身是**已注册 provider 别名**且与 `review.provider` **冲突** → fail-closed。
- **禁止**改全局 `review_provider_identity()` 或 `validate_review_identity()`/身份分离 `:717-718`
  的行为(那是禁改主干)。用 **task 专用的严格解析**,不外溢到主 review 身份门。若你判断必须动
  全局 helper,停下并在报告里 surface,不要自行放宽主门。

### finding-2（P1）—— git tree/symlink 冒充 evidence
现状:`_evidence_committed_blob` 用 `git show` 读内容,**不校验对象类型/模式**;evidence_file
指向目录时 `git show HEAD:dir` 返回非空 tree 列表,digest 匹配即过。修:
- 读 blob 前,校验 git 对象是**常规文件 blob**:`git cat-file -t <HEAD>:<path>` == `"blob"`,
  且 `git ls-tree <HEAD> -- <path>` 的 mode 为常规文件(`100644`/`100755`),**拒绝** tree、
  symlink(`120000`)、submodule(`160000`)、其它非常规模式。非常规 → FAIL(negative-list #4)。
- 保留 Fable5 的 `evidence_sha256` + 验证时 HEAD 设计;**不要**绑定 `status.head_sha`。
- 常规 blob 通过后再按现有逻辑算 sha256 比对。

### P3 —— AGENTS.md EOF 空行
删 `AGENTS.md` 结尾多余空行(`git diff --check` 报 "new blank line at EOF")。

## 禁改（硬边界）
`compute_diff_fingerprint()`;身份分离 `:717-718` / 全局 `review_provider_identity` 主门语义;
共享 `evidence_path_exists`;class-2;RC1/RC2/RC3;funding_hedging;AGENTS.md 两仓 carve-out;
Fable5 的 evidence_sha256 方案(别改回 blob@head_sha);不 commit/merge/push。

## 完成后
- 追加 `40-fix-report.md`(rework-2 段):#1/#2/P3 逐条 → 函数/行/修法映射;说明为何不动全局
  身份主门。
- 追加 `60-test-output.txt`(rework-2 段,别删已有 A–E.1),用构造样例证明 Codex 的对抗清单:
  - owner=claude_glm + review.provider=zhipu_glm,**即使 reviewer 是任意未注册串** → FAIL;
  - 合法跨 provider(review.provider 与 owner 不同)→ PASS;
  - reviewer 是已注册别名且与 provider 冲突 → FAIL;
  - task 缺 owner/implementer 的 own-review → FAIL;
  - evidence_file 指向目录(tree)+ 匹配 digest → FAIL;committed symlink / submodule → FAIL;
  - 常规非空 blob + 匹配 digest → PASS;
  - digest 不匹配 / 绝对路径 / 逃逸 / untracked / 空 blob / reason&at 非法 / class-2 /
    身份 negative-list → 仍 FAIL;
  - 原 task 横幅、canonical scope、唯一 id、own-review 指纹、模板默认、零回归 → 仍绿。
  跑 `python3 -m py_compile scripts/validate-stage.py`、原套件 + 新用例、`--phase checkpoint`。
  临时脚本不留在交付。
- 报告用统一 footer + provider-native Session ID(或 unavailable+原因)。**不要 commit。**

## END PROMPT BODY
