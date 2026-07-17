# Handoff

This file is the stage's active handoff context. A new terminal session reads
it together with `status.json` and only the needed workflow section.

## Recovery Header

- Active phase: `implementing`（rework-1;review-2 Codex REWORK,等定 fix 范围后派 claude_glm）
- Repo: **模板仓** `ai_project_harness`（不是 funding_hedging）。validate-stage.py +
  schemas 是 harness_owned，在此开发；接受后手动 cp 到 funding_hedging。
- head_sha `941416345c92bf3c42556e0d9178f28e1384b4e6`；diff_fingerprint `9414163…:5b28d88c…`。
- review-1(Kimi/moonshot)=ACCEPT;**review-2(Codex/openai/gpt-5)=REWORK**,rework_count=1/3。
  50-review-2.md 已由 bookkeeper 从 codex read-only 会话逐字落盘;verdict JSON 合 schema、
  含 fix_start_prompt。
- **4 条 P1(bookkeeper 已逐条核代码,全部成立)**:①task 例外静默放行(违反 D2 非静默)
  ②任意非空串冒充 task own-review(D3 假绿)③授权来源未认证/未钉 head_sha,文档"模型不能
  自授"表述不成立 ④task scope 歧义(task:<id> 与裸 <id>)+id 不唯一。P2:reason/at 未校验、
  模板 tasks[] 注解键。
- **finding-3 已 Fable5 DIRECTION-SET(56 文件)**:A 档务实硬化——①仓内相对+防逃逸(删 :819
  绝对分支)②git-tracked 读提交 blob ③记录加 `evidence_sha256`、validator 重算 blob 比对
  (**不用 blob@head_sha,会与 post-head 授权死锁**)④task 按 assertion_id 过滤。Q2 文档必改
  (防静默+强制人核两层,人核是流程义务)。不做 B(receipt 依赖 DRAFT 基建)。10-design §2/§2.1/§7
  已更。拆分:user_authorizations 结构化 + runner receipt → 后续 stage。
- Next action: **操作者派 claude_glm 执行 `35-dispatch-fix-1-claude-glm.md`**(fix 作者
  zhipu_glm,provider 隔离 OK);产出 40-fix-report.md + 追加 60-test-output.txt(别删原证据);
  bookkeeper R4 对账→提交→重算指纹→pre-review→**重进 review-1(Kimi)→ review-2(Codex)**。
  fix 作者禁为 Codex。
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
本地北京时间: 2026-07-17 13:30:00 CST
下一步模型: claude_glm（rework-1 fix,经操作者派发）
下一步任务: 执行 35-dispatch-fix-1-claude-glm.md 修 4 P1+2 P2(finding-3 按 Fable5 A 档),写 40-fix-report + 追加 60-test-output
