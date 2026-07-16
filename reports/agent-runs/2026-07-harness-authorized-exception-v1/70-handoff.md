# Handoff

This file is the stage's active handoff context. A new terminal session reads
it together with `status.json` and only the needed workflow section.

## Recovery Header

- Active phase: `intake`（Stage A intake 已落档，等用户拍板 3 个设计决策）
- Repo: **模板仓** `ai_project_harness`（不是 funding_hedging）。validate-stage.py +
  schemas 是 harness_owned，在此开发；接受后手动 cp 到 funding_hedging。
- Next action: **用户审阅 `00-task.md` 的 D1/D2/D3 三个设计决策并拍板**；据此 bookkeeper
  出 `10-design.md`，再派 Claude-GLM 实现 → Kimi review-1 → Codex review-2。
- 方向：Fable5 裁决已覆盖（RC4 (a)分任务指纹 +(b)授权例外槽，复用 :735-737 模式，R1）。
- 范围：只修 validator 的例外/指纹能力（RC4）；RC1/RC2/RC3 属 Stage B+。
- Bootstrapping：本 stage 改 validator，自身仍由改前 validator 把关；验收含"用改后脚本
  对本 stage 自己跑仍 PASS + 零回归"。
- 顺带目标（接受+cp 后做，不在本 stage）：转绿 bookticker / docs-truth-sync 两红门。
- Read-set: = `status.current_inputs`
- Open blockers: None（等用户定 D1/D2/D3）
- Do-not-read: `reports/agent-runs/**/history/**`, other stages

## Current State

- Stage: `2026-07-harness-authorized-exception-v1`（Fable5 的 Stage A）
- Status: `planned`
- Branch: `stage/2026-07-harness-authorized-exception-v1`（模板仓，base `00e25b4`）
- Complexity: MEDIUM，lightweight route（方向 Fable5 覆盖）

## Open Decisions（用户）

- **D1**：例外槽白名单 vs 任意断言（倾向白名单）。
- **D2**：`authorized_exceptions` 字段形状 + 仅 human 授权 + evidence_file 必须存在。
- **D3**：分任务指纹做多深（倾向轻量：覆盖并集校验）。

## Next Steps

1. 用户拍板 D1/D2/D3。
2. bookkeeper 出 `10-design.md`（把三决策定死）+ `11-adr.md`。
3. 派 Claude-GLM 实现（validator + schema + _template + 说明书段落）。
4. review-1 Kimi → review-2 Codex（终审严审证据完整性主干）。
5. 接受 → 手动 cp 到 funding_hedging → 下个 stage 实测 → 转绿两历史红门。
6. 之后开 **Stage B+**（文档门 + 索引生成 + Harness 说明书文字，复用例外槽）。

当前 Session ID: unavailable (Claude Code session id 未暴露)
Session ID 来源: unavailable
原始输出路径: reports/agent-runs/2026-07-harness-authorized-exception-v1/70-handoff.md
本地北京时间: 2026-07-17 01:02:53 CST
下一步模型: human（拍板 D1/D2/D3）
下一步任务: 定三个设计决策；据此出 10-design
