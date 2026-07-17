> ⚠️ **SUPERSEDED / 作废（review-2 round-2 finding-4）**：本包绑定的是 round-1 旧 head
> `9414163`/旧指纹,**不得据此审查**。round-2 起改用 status.json 记录的当轮 head/指纹;
> round-3 的 review-2 将另出绑定新 head 的专用包。保留此文件仅作历史。

# Dispatch Packet — Stage A review-2（reviewer: codex，终审）

⚠️ **模板仓 `ai_project_harness`**，分支 `stage/2026-07-harness-authorized-exception-v1`。
你是**终审门(review-2)**,审的是 Harness **证据完整性主干**(`validate-stage.py`)——
写宽一行毁掉整个信任地基,**严审**。你只审、不改。产出 `50-review-2.md` + 写回
`status.review_2`。

- base_sha `00e25b4bb6e492ffbd974d1d7793ef10246fa6ad`
- head_sha `941416345c92bf3c42556e0d9178f28e1384b4e6`
- status.diff_fingerprint
  `941416345c92bf3c42556e0d9178f28e1384b4e6:5b28d88ce253d6d41f9c5841cfe51265a66e132c8fe7556d1177da4404b186f0`
- 身份隔离:你 provider=openai;实现者 claude_glm=zhipu_glm;review-1 kimi=moonshot ——
  三方 provider 互不相同,满足。
- **设计披露**:`10-design.md` 与 `12-development-breakdown.md` 由 bookkeeper(anthropic,
  claude-opus-4-8)撰写(design involvement);与你(openai)无 provider 重叠,不触发
  designer-overlap override。据此严审设计本身是否有漏洞,不因"设计已定"而放松。

---

## PROMPT BODY

你是 Stage A 的 review-2(`codex`,证据完整性主干终审)。RC4 给 `validate-stage.py`
加了 authorized_exception(源码白名单 class-1 + 钉指纹 + fail-closed)与 D3 分任务指纹
覆盖(链式+前缀)。**你只审、不改代码。**

先读:
- `reports/agent-runs/2026-07-harness-authorized-exception-v1/10-design.md`（机制全文、
  negative list、D1/D2/D3;bookkeeper 撰写）
- `.../11-adr.md`、`.../12-development-breakdown.md`（逐条 §A–§E + Review Focus + 风险点）
- `.../20-implementation.md`（实现者自述:函数/行号、negative-list 物理隔离论证）
- `.../60-test-output.txt`（10 构造用例 + bootstrapping + 脚本源码）
- `.../30-review-1.md`（**review-1 已 ACCEPT**;含 Kimi 自写的 4 条对抗用例结果与 3 条 P2;
  你需独立复核,不得直接采信）

看代码 diff：
```
git -C . diff 00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..941416345c92bf3c42556e0d9178f28e1384b4e6 -- scripts/validate-stage.py AGENTS.md docs/harness-design.md reports/agent-runs/_template/status.json
```

**独立复算指纹**(不要只信 status/review-1):
```
python3 - <<'PY'
import importlib.util; from pathlib import Path
spec=importlib.util.spec_from_file_location('vs','scripts/validate-stage.py')
vs=importlib.util.module_from_spec(spec); spec.loader.exec_module(vs)
root=Path('.').resolve(); sd=root/'reports/agent-runs/2026-07-harness-authorized-exception-v1'
print(vs.compute_diff_fingerprint(root, sd,
  '00e25b4bb6e492ffbd974d1d7793ef10246fa6ad',
  '941416345c92bf3c42556e0d9178f28e1384b4e6'))
PY
```
必须等于上面的 status.diff_fingerprint。

**review-2 终审焦点(逐条给结论,允许构造对抗用例复跑真实 validator):**
1. **自授绕过**——`authorizer=="user"` 只是字面量检查,evidence_file 只查存在+非空(不做
   NLP)。追问:模型/失职写入方能否伪造非空 evidence 文本 + 字面量 "user" 通过结构校验从而
   自授?这条残余风险(**review-1 P2-2 已标,明确转交你**)在"授权原文只能来自用户消息"这一
   进程约束下是否可接受?还是需要代码层硬化(如 evidence 必须援引 status.user_authorizations
   的 instruction_verbatim)?给明确裁断:**可接受 / 需硬化(P 级)**。
2. **fail-closed 真成立**——`validate_authorized_exceptions` 任一坏记录 → `return errors, []`;
   确认 `validate_acceptance` 与 `validate_task_coverage` 两处消费 valid 集时都无法在 B 挂了
   的情况下放行;确认 `validate_task_coverage` 丢弃 B 的 errors 不会导致某类坏记录被静默吞掉
   (main 调用顺序是否保证 errors 已由 acceptance 先 surfaced)。
3. **降级外科手术性**——降级是否**只**作用于 `review.diff_fingerprint != status` 单条;
   verdict/json_schema_valid/tests/身份分离/task 指纹是否都无例外路径(negative list + class-2
   不收)。尝试构造跨 assertion / 跨 scope / task-scope 误降 review 的绕过。
4. **链式+前缀等价覆盖 base..head 且无 diff 代数漏洞**——D3 用 sha 相等 + 前缀指纹比对表达
   覆盖。追问:链成立(首尾相接)是否**充分**保证并集覆盖 base..head 无缝无叠?前缀指纹
   `base..task[j].head` 与 review 指纹比对能否被构造出"看似覆盖实则漏改"的反例?
   `covers_through_task` 解析(int/str、越界、bool 排除)有无注入/绕过面。
5. **钉指纹自动失效**——`applies_to_fingerprint==status.diff_fingerprint` 是否真能保证下一轮
   fix(head 变)后旧例外失效;有无绕过(如构造 head 不变但内容变——指纹公式含 head_sha,
   评估可行性)。
6. **边界零越界**——`compute_diff_fingerprint` 公式一字未动、身份分离 :717-718 未放宽、未碰
   funding_hedging、无 RC1/RC2/RC3、AGENTS.md 两仓 carve-out 未覆盖。

**bootstrapping**:改后脚本对本 stage 自己 `--phase pre-review` 仍 PASS、现有断言零回归。

产出 `50-review-2.md`：结构化结论、P0/P1/P2 清单(每条 file:line + 失败场景)、对焦点 1 的
明确裁断、你复算的指纹。**报告末尾附一个符合 `schemas/review-verdict.schema.json` 的 JSON
verdict**(你本次调用已带 `--output-schema`)。写回 `status.review_2`:
`reviewer=codex, provider=openai, model=<你的模型>, verdict, json_schema_valid,
diff_fingerprint=<复算值>, primary_provider=codex, reviewer_prior_involvement="none",
open_p0_p1`。**不改被审代码。** 若 REWORK,列 required_fixes 与具体修法。报告用统一 footer +
provider-native Session ID(或 unavailable+原因)。

## END PROMPT BODY
