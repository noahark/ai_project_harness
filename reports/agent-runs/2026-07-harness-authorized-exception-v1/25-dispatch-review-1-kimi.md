# Dispatch Packet — Stage A review-1（reviewer: kimi）

⚠️ **模板仓 `ai_project_harness`**，分支 `stage/2026-07-harness-authorized-exception-v1`。
被审对象是 harness_owned 的证据完整性主干（`validate-stage.py`）。**你只做 review-1,
不修代码**（修复由后续 fix 作者做）。产出 `30-review-1.md` + 写回 `status.review_1`。

- base_sha `00e25b4bb6e492ffbd974d1d7793ef10246fa6ad`
- head_sha `941416345c92bf3c42556e0d9178f28e1384b4e6`
- status.diff_fingerprint
  `941416345c92bf3c42556e0d9178f28e1384b4e6:5b28d88ce253d6d41f9c5841cfe51265a66e132c8fe7556d1177da4404b186f0`
- implementer `claude_glm`（provider zhipu_glm）；你 provider=moonshot,身份分离满足。

---

## PROMPT BODY

你是 Stage A 的 review-1（`kimi`）。这是 RC4 证据完整性主干改动——**写宽一行就毁掉信任
地基,严审**。你只审、不改。

先读（规格 → 实现 → 证据）：
- `reports/agent-runs/2026-07-harness-authorized-exception-v1/10-design.md`（机制全文、
  negative list、D1/D2/D3）
- `.../11-adr.md`、`.../12-development-breakdown.md`（逐条 §A–§E + Review Focus + 风险点）
- `.../20-implementation.md`（实现者自述:函数/行号、negative-list 物理隔离论证、§5 规格冲突处置）
- `.../60-test-output.txt`（构造样例 + bootstrapping + 脚本源码）

看代码 diff：
```
git -C . diff 00e25b4bb6e492ffbd974d1d7793ef10246fa6ad..941416345c92bf3c42556e0d9178f28e1384b4e6 -- scripts/validate-stage.py AGENTS.md docs/harness-design.md reports/agent-runs/_template/status.json
```

**独立复算指纹**（不要只信 status 里的值）：
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

**review-1 严核清单（12-breakdown Review Focus 的 review-1 部分,逐条给结论）：**
1. **negative-list 五项确实不可豁免**——不是靠注释,而是靠代码结构。逐条追:
   ①status.diff_fingerprint 重算(validate_common,降级路径够不到)②clean worktree
   ③reviewer 身份分离(:717-718 等价,`validate_review_identity` 无 override)④例外记录自身
   evidence_file 存在/非空 ⑤例外记录结构完整。确认 `validate_acceptance` 的降级分支**只**
   作用于 `review.diff_fingerprint != status` 这一条。
2. **class-2 确实未被放宽**——`verdict!=ACCEPT`/`json_schema_valid`/`tests.status` 无任何
   降级分支;白名单枚举 `AUTHORIZED_EXCEPTION_ASSERTION_IDS` 只含 class-1。
3. **钉指纹真能"下一轮自动失效"**——`applies_to_fingerprint==status.diff_fingerprint`
   是唯一保证;构造"下一轮指纹变了"能否让旧例外失效(测试用例 3 是否充分)。
4. **fail-closed 真成立**——`validate_authorized_exceptions` 任一坏记录 → `return errors,[]`
   → `_exception_covering_review` 必返回 None;确认 `validate_task_coverage` 第二次调用
   拿到的 valid 集也受同一 fail-closed 约束。
5. **退化零改变**——`tasks` 缺省/单任务(len<=1)→ return [];现有断言文案与行为逐字不变
   (对照测试用例 7a/7b/REG 与改前)。
6. **边界零越界**——`compute_diff_fingerprint` 公式一字未动;身份分离逻辑未放宽;未碰
   funding_hedging;未做 RC1/RC2/RC3;AGENTS.md 两仓 carve-out 行未覆盖。
7. **§5 规格冲突处置是否可接受**——`covers_through_task` 落在 review 上(逻辑规格)、
   `_template` 的 tasks[] 上是说明键;判断这是否会误导或产生歧义。

**测试证据审查**：60-test-output 的 10 个用例是否真覆盖上述断言;用例 5(negative-list #1)
是否真证明"降级接触不到 status 重算";有无遗漏的绕过路径(尤其自授、跨 scope 误降级)。

产出 `30-review-1.md`：结构化结论（ACCEPT / REWORK）、P0/P1 清单（每条给 file:line +
失败场景）、`json_schema_valid` 判定、你复算的 `diff_fingerprint`。
写回 `status.review_1`：`reviewer=kimi, provider=moonshot, model=<你的模型>, verdict,
json_schema_valid, diff_fingerprint=<复算值>, open_p0_p1, reviewer_prior_involvement="none",
invalid_json_attempts`。**不改被审代码。** 报告用统一 footer + provider-native Session ID
（或 unavailable+原因）。

## END PROMPT BODY
