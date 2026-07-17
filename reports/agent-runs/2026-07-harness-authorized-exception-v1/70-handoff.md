# Handoff

This file is the stage's active handoff context. A new terminal session reads
it together with `status.json` and only the needed workflow section.

## Recovery Header

- Active phase: `review_2`（review-1 Kimi ACCEPT,等派 review-2 Codex）
- Repo: **模板仓** `ai_project_harness`（不是 funding_hedging）。validate-stage.py +
  schemas 是 harness_owned，在此开发；接受后手动 cp 到 funding_hedging。
- head_sha `941416345c92bf3c42556e0d9178f28e1384b4e6`；diff_fingerprint
  `9414163…:5b28d88c…`（bookkeeper 已交付+记账；review-1 复算一致、pre-review PASS）。
- review-1(Kimi/moonshot) = **ACCEPT**，P0=0/P1=0/P2=3(非阻塞)；30-review-1.md 已入库。
- Next action: **操作者在 Codex 终端(模板仓)执行 review-2**（证据完整性主干终审严审）;
  **必带 focus：P2-2 evidence_file 真实性靠人核、非代码强制（Kimi 转交）——确认残余风险
  可接受还是需硬化**；产出 `50-review-2.md` + 写回 `status.review_2`。
- review-1 遗留 P2(非阻塞,后续模板修订):P2-1 tasks[] 注解键误导余地;P2-3 evidence_file
  接受仓外绝对路径。
- 设计已定(Fable5 D1/D2/D3,见 `10-design.md`)：D1 白名单源码枚举、v1 只收 class-1、
  negative list 五项不可豁免;D2 钉指纹自动失效 + authorizer=="user" 硬化 + 非静默;
  D3 链式+前缀、单任务退化。
- ⚠️ 最危险点:降级逻辑只能对 `review.diff_fingerprint` 单条、且经例外全合规后降级;
  绝不降级 verdict/身份/指纹重算(negative list)。
- Bootstrapping：改后脚本对本 stage 自己跑仍需正常;现有行为零回归。
- 顺带目标（接受+cp 后做,不在本 stage）：bookticker 真转绿(class-1);docs-truth-sync
  维持 user-acceptance-override(其 review_2 是 class-2,不在 v1)。
- Read-set: = `status.current_inputs`
- Open blockers: None
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
本地北京时间: 2026-07-17 12:10:00 CST
下一步模型: codex（review-2,证据完整性主干终审）
下一步任务: 严审自授绕过/fail-closed/链式覆盖代数漏洞 + 确认 P2-2 残余风险,写 50-review-2
