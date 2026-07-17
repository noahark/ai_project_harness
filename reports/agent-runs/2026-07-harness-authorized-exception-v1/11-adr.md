# ADR — Stage A: authorized_exception + task-fingerprint coverage

## 决策

给 `validate-stage.py` 增加两个机制,修 RC4：

1. **authorized_exception(源码白名单 + 数据援引)**：pre-accept 断言可被一条"用户授权 +
   证据文件 + 钉死当前指纹"的例外记录降级为 PASS-with-exception(仅白名单内的 assertion,
   v1 只含 class-1 `review_fingerprint_trails_status`)。白名单枚举在 validator 源码,
   stage 数据只能援引不能定义。
2. **task-level fingerprint coverage(链式 + 前缀)**：tasks[] 首尾相接 ⇒ 覆盖 base..head;
   review 指纹按前缀比对到 `covers_through_task`;之后任务需自审或 class-1 例外。退化到
   单任务 = 现行行为。

## 理由

- RC4 的病是"validator 不会表达合法评审例外/增量覆盖",与指纹格式无关;指纹公式一字不改。
- 复用现有 `:735-737` override-evidence 模式与 `:749-787` 任务级脚手架,小 diff、低新颖度。
- fail-closed 关键:数据援引例外类、源码定义例外类;negative list 保证豁免机制不能豁免
  自己(指纹封印、clean worktree、身份分离、例外记录自身完整性)。
- 硬化"钉指纹"避免把永久假红换成永久假绿。

## 备选与放弃

- 纯 commit id 指纹:放弃——丢 status.json 排除 + 内容封印,且**治不了 RC4**(评审例外/
  覆盖问题与指纹格式正交)。
- class-2(豁免 verdict==ACCEPT):v1 不收——终审门最重的削弱,按已证明需要生长。
- 完整任务级评审矩阵/DAG:放弃——重、超范围;链式+前缀已表达 bookticker。
- 取消开发者 commit 禁令 + 自动跑 dev→review1:独立议题(=已退役的 auto-review-pipeline
  方向),不在本 stage;若重开需读退役证据 + Fable5 约束。

## 影响

- 模板仓 `validate-stage.py` + `_template/status.json` + `AGENTS.md`/`harness-design.md`
  段落;接受后手动 cp 到 funding_hedging。
- 落地后:bookticker 可真转绿(class-1);docs-truth-sync 维持 user-acceptance-override。

当前 Session ID: unavailable
原始输出路径: reports/agent-runs/2026-07-harness-authorized-exception-v1/11-adr.md
本地北京时间: 2026-07-17 01:02:53 CST
下一步模型: human → Claude-GLM
下一步任务: 认可设计,派实现
