# Stage Task

## Stage ID

`2026-07-harness-authorized-exception-v1`（Fable5 裁决的 **Stage A**）

## Goal

给 `scripts/validate-stage.py` 补两个能力，使 validator 能表达"合法的评审例外"，从而
消除 RC4「假红门」：

1. **RC4(a) 分任务指纹覆盖（task-level fingerprint coverage）**：让 pre-accept 能按
   `tasks[]` 分块校验评审覆盖的并集，而不是只做单条全量 `review.diff_fingerprint ==
   status.diff_fingerprint` 比对。
2. **RC4(b) 授权例外槽（authorized_exception）**：新增一个正式、可审计的字段，记录
   "某条 pre-accept 断言被用户授权跳过 + 理由 + 证据文件"；validator 见到合规记录即
   放行，而非红灯。复用现有 `validate-stage.py:735-737` 的 evidence-file 放行模式
   （已验证同型先例，风险低）。

**顺带目标**：机制落地后，两个历史「假红门」可合法转绿——
- `funding_hedging` 的 `bookticker-open-columns-v1`（Task C 后加、review-1 指纹不覆盖）；
- `funding_hedging` 的 `docs-truth-sync-v1`（review-1 被豁免 + review-2 内容干净但形式
  REWORK）。
（转绿动作在 cp 回本仓后做，不在本 stage。）

## Direction Basis（等价 synthesis）

- Fable5 裁决：`funding_hedging/reports/agent-runs/2026-07-docs-truth-sync-v1/
  81-harness-design-rootcause-review-fable5.md`（Q4 + F3 + R1）。
- 根因报告 RC4：同目录 `80-harness-design-rootcause.md`。

## 要在 10-design 定死的 3 个设计决策（本 stage 的真正难点）

### D1 — 例外槽能豁免哪些断言？（白名单 vs 任意）
- 选项 A（推荐）：**白名单**，只允许豁免两类已知合法例外——
  `review_1_fingerprint_mismatch_under_waiver`、`review_2_content_clean_rework`。
- 选项 B：允许豁免任意 pre-accept 断言。
- 取舍：白名单安全（绝不能让人豁免掉"指纹重算一致"这种信任地基本身）；任意更灵活但
  危险。**倾向 A**。

### D2 — 授权例外槽的字段形状 + 谁能授权
- 字段（拟）：`authorized_exceptions: [{ assertion_id, reason, authorizer:"user",
  evidence_file, at }]`。
- validator 逻辑：某条断言失败时，若存在匹配 `assertion_id` 的合规记录（含存在的
  `evidence_file`），则降级为 PASS-with-exception 并在输出里显式标注，不静默。
- 谁能授权：仅 human/user（模型不能自授）。证据文件必须存在（照 735-737 模式）。

### D3 — 分任务指纹覆盖做到多深？
- 选项 A（轻）：pre-accept 校验"review-1/2 各自 `diff_fingerprint` 属于 `tasks[]` 里
  某个已记录的任务指纹，且这些任务的并集覆盖 `base..head`"。
- 选项 B（重）：完整实现按任务分裂 diff、逐任务重算指纹。
- 取舍：schema 已有 `tasks[].review_1.diff_fingerprint` 脚手架；**倾向 A**（够用、小
  diff、贴合"聚焦证据完整性"）。若 A 表达不了 bookticker 那种"后加任务"，再考虑 B。

## File Boundaries

- Allowed（模板仓内）：
  - `scripts/validate-stage.py`（核心逻辑）
  - `schemas/review-verdict.schema.json` 和/或 stage `status.json` 的 schema（若新增
    `authorized_exceptions` 字段需 schema 支持）
  - `reports/agent-runs/_template/status.json`（补新字段模板）
  - `docs/harness-design.md` / `AGENTS.md` 中描述 pre-accept 与例外机制的段落（说明书
    同步，属本 stage 的必要文档）
  - 本 stage 证据文件
- Forbidden：
  - 任何 `funding_hedging` 仓的文件（cp 是接受后的独立动作，不在本 stage diff）
  - 与例外/指纹无关的 validator 重构、别的 RC 修法（RC1/RC2/RC3 属 Stage B+）

## Bootstrapping（重要，避免自指悖论）

本 stage **改的就是 validator**，但它自身的 pre-review/pre-accept 仍由**改动前的
validator** 把关（分支上跑的是被改的脚本——所以要用"改完后的脚本对本 stage 自己跑一遍
必须仍 PASS"作为验收之一，同时确认现有测试不回归）。新能力对 `funding_hedging` 的效果，
在 cp 回本仓后的**下一个 stage** 实测。

## Non-Goals

- 不做 RC1/RC2/RC3/RC7（文档门、索引生成、manifest 分类）——那是 Stage B+。
- 不在本 stage 内 cp 到 funding_hedging，也不转绿历史红门（接受后单独做）。
- 不重构 validator 的无关部分。

## Acceptance Criteria（可机械复核）

1. 新增 `authorized_exceptions` 机制：给一个带合规例外记录（含存在的 evidence_file）的
   构造样例，pre-accept 从 FAIL 变为 PASS-with-exception 且**显式标注**该例外；去掉
   evidence_file 或记录不合规则仍 FAIL。
2. 白名单（若采 D1-A）：尝试豁免"指纹重算一致"这类地基断言必须被拒绝。
3. 分任务指纹（D3）：构造 review-1 覆盖 A/B、review-2 覆盖 A/B/C 的样例，pre-accept 能
   正确判定覆盖并集。
4. 现有 validator 测试/行为**零回归**；用改后脚本对本 stage 自己跑 pre-review/pre-accept
   仍符合预期。
5. `docs/harness-design.md`/`AGENTS.md` 相应段落已同步描述新机制。

## Routing（拟）

- 实现：`claude_glm`（backend/schema/validator 属其域）。
- review-1：Kimi（跨厂商隔离）。
- review-2：Codex/GPT（终审，证据完整性主干必须严审）。
- 单实现者，非并行。

## Linked / Next

- 接受后：手动 cp 到 `funding_hedging` → 下个 stage 实测 → 转绿 bookticker /
  docs-truth-sync 两红门。
- 之后：**Stage B+**（RC1/RC2/RC3/RC7 文档门 + 索引生成 + Harness 说明书文字），复用
  本 stage 的例外槽。

当前 Session ID: unavailable
Session ID 来源: unavailable
原始输出路径: reports/agent-runs/2026-07-harness-authorized-exception-v1/00-task.md
本地北京时间: 2026-07-17 01:02:53 CST
下一步模型: human（定 D1/D2/D3）→ Claude-GLM 实现
下一步任务: 审阅并拍板 D1/D2/D3；据此出 10-design.md
