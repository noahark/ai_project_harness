# Stage Design — Stage A: authorized_exception + 分任务指纹覆盖（RC4）

依据：`funding_hedging/.../82-stage-a-design-rulings-fable5.md`（D1/D2/D3 裁决）、
`81-...-review-fable5.md`（R1、Q4）、`80-harness-design-rootcause.md`（RC4）。
仅改模板仓 `validate-stage.py` + schema + `_template` + 说明书段落。小 diff、聚焦证据
完整性主干。

## 0. 现状锚点（改动落点）

- 红门在 `validate_acceptance()`（`scripts/validate-stage.py:791-806`），关键行：
  `review.diff_fingerprint != status.diff_fingerprint` → error。
- 任务级脚手架已在 `validate_tasks()`（:749-787）：已逐任务校验 base/head + 指纹重算。
- 指纹公式 `compute_diff_fingerprint()`（:160-176）**一字不改**（Q4/R1 约束）。
- 身份分离 `:717-718` 无 override，**属 negative list，不可豁免**。

## 1. D1 — 例外白名单（源码枚举，fail-closed）

**白名单 = validator 源码里的 `assertion_id` 枚举,不是 status/schema 数据。** stage 数据
只能**援引**例外类,不能**定义**例外类。扩类 = 模板仓代码变更 + 强评审。

### v1 白名单（只收一类）
- `review_fingerprint_trails_status` (**class-1**)：某道 review 的 `diff_fingerprint`
  落后于 `status.diff_fingerprint`,且存在用户 review-waiver 授权记录。唯一有真实实例的
  合法例外类（bookticker Task C 后加；docs-truth-sync round≥4 review-1 豁免）。

### v1 明确不收
- **class-2（review-2 形式 REWORK 但内容干净）不收。** 豁免 `verdict==ACCEPT` 是对终审门
  最重的削弱。原则:白名单按已证明的需要生长,绝不按推测生长。
  - 〔evolve-by-note 订正,bookkeeper 2026-07-17〕:82 文件称 class-2"零实例"不精确——
    docs-truth-sync round-6 即一例(F13/F14 已解、仅剩已修的 F16/F17 账本项)。但该 stage
    已由**用户接受覆盖**合并,不再跑 pre-accept,故 v1 不收 class-2 的结论不变;仅"历史
    转绿"表述需精确(见 §5)。若将来 class-2 需求复现,按 bookticker 先例记一次性红门
    例外 + 走模板仓扩类评审。

### Negative list（永不可豁免,写死在设计与代码注释）
1. `status.diff_fingerprint` 与重算一致（内容封印本身）
2. clean worktree
3. reviewer 身份分离（`:717-718`,本已 no-override）
4. 例外记录自身的 `evidence_file` 存在性
5. 例外记录自身的结构完整性（字段齐、`authorizer=="user"`、`assertion_id` 命中枚举、
   `applies_to_fingerprint` 命中当前指纹）
→ **豁免机制不能豁免自己。** 任一 negative-list 断言即便挂了例外记录也照常 fail。

## 2. D2 — `authorized_exceptions` 字段 + 两处硬化

```json
"authorized_exceptions": [{
  "assertion_id": "review_fingerprint_trails_status",   // 必命中 validator 枚举
  "scope": "review_1",                                   // 断言实例(哪道 review / 哪个 task)
  "applies_to_fingerprint": "<head_sha>:<sha256>",       // 硬化1:钉死到具体指纹
  "reason": "…",                                         // 非空(rework-1 校验)
  "authorizer": "user",                                  // 字面量,仅此值合法
  "evidence_file": "reports/agent-runs/<stage>/…",       // 仓内相对、已提交(rework-1)
  "evidence_sha256": "<hex>",                            // 提交 blob 的 sha256(rework-1 新增)
  "at": "ISO-8601"                                        // 可解析 ISO-8601(rework-1 校验)
}]
```

- **硬化1（钉指纹）**：例外只对 `applies_to_fingerprint` 那次 diff 生效。再来一轮 fix、
  指纹一变,例外**自动失效、门重新变红**,必须重新授权。防止"一次豁免静默盖住后续所有
  轮次"——否则把 RC4 的永久假红换成更糟的永久假绿。
- **硬化2（防自授,rework-1 按 Fable5 finding-3 裁决重写；见 §2.1）**：把"防自授"从
  **不可能性声称**降级为**防静默 + 强制人核**两层。`authorizer=="user"` 字面量、钉指纹、
  **仓内已提交且 digest 封存**的 evidence,共同保证自授**无法静默发生**(伪造豁免必须把
  伪造文本提交进受审 git 历史、并在 PASS-with-exception 横幅亮明,留下可追溯痕迹);但代码
  **不能证明** evidence 文本源自人类(模型能写任何字节)。「不能自授」的**最终保证**是
  pre-accept 放行任何例外前**操作者人工核对 evidence 原文**(横幅即人核触发点)。人核是
  **流程义务**(workflow checklist + handoff),validator 无法强制人读——与 80 报告 RC 方法论
  一致,不装成机械门。〔订正:旧文"授权原文只能来自用户消息这一结构,不靠自觉"是把惯例说成
  代码强制,出自 82/D2 的过度声称,Fable5 已认账修正。〕
- **放行绝不静默**：validator stdout 打印
  `PASS (N authorized exceptions applied: <assertion_id@scope> …)`;exit 0,但 pre-accept
  证据(62-)必须含该行;例外同时列进 70-handoff Recovery Header。
- 无 expiry 字段:钉指纹天然一次性。

## 2.1 finding-3 授权来源机械硬化（rework-1，Fable5 DIRECTION-SET，见 56 文件）

review-2 Codex 判 finding-3(P1):实现的 evidence 校验只查存在+非空,未兑现 D2「evidence
含真实授权」的承诺。Fable5 裁 **A 档（务实硬化）+ C 级边界**,本 rework 只做四件机械可靠事,
**不做 B**(结构化 user_authorizations 链 / runner receipt 依赖仍 DRAFT 的看门狗基建,在未建
基建上立门是已判反模式):

1. **仓内相对路径 + 逃逸防护**：`evidence_file` 拒绝绝对路径;resolve 后必须仍在 repo root
   内(防 `../` 逃逸)。删 `_evidence_file_nonempty` `:819` 的 absolute-candidate 分支。
   **不改共享的 `evidence_path_exists`(:363)**——RC4 committed-blob 逻辑自包含,不影响别的
   caller。
2. **必须已提交(git-tracked)**：validator 读**提交内容**(`git show <验证时 HEAD>:<path>`),
   不是工作树文件;杜绝 untracked 文件充当证据、堵非 clean 阶段口子。
3. **digest 封存(对 Codex fix 第 3 条的必要修正)**：例外记录新增 `evidence_sha256`,
   validator 重算提交 blob 的 sha256 比对。**不采用** Codex 原文的 blob@`status.head_sha`
   形态——它与真实流程**死锁**:class-1 授权须引用新指纹,新指纹只在 head commit 之后存在,
   故授权证据必然 **post-head** 提交(bookticker 事后补录、docs-truth-sync 的 waiver 落在 fix
   head 之后的簿记提交,两个真实实例在 blob@head_sha 下都无法合法转绿)。digest-in-record +
   git-tracked 达成同等防篡改,无顺序死锁。
4. **task 消费端按 `assertion_id == review_fingerprint_trails_status` 显式过滤**(与 finding-4
   同批)。

**拆分(不阻塞本 stage)**:① `user_authorizations[]` 结构化 schema + 例外记录引用链(增加
可审计导航,非来源证明,单独设计);② runner receipt(看门狗基建离 DRAFT 后作为"人核之上
第二保证",Q2 措辞已预留接口)。

## 3. D3 — 分任务指纹覆盖（链式 + 前缀,轻量）

复用 `validate_tasks()` 现成的逐任务指纹重算。新增覆盖语义(仅 sha 相等 + 前缀比对,零
diff 代数):

1. **链式覆盖**：`tasks[]` 首尾相接——
   `task[0].base_sha == status.base_sha`;`task[i+1].base_sha == task[i].head_sha`;
   末段 `task[-1].head_sha == status.head_sha`。链成立 ⇒ 并集覆盖 `base..head`。
2. **前缀规则**：`review_k.diff_fingerprint` 必须等于重算的
   `status.base_sha .. task[j].head_sha` **前缀指纹**（`j = review_k.covers_through_task`）。
   j 之后的每个任务,必须**要么有自己的 review 记录,要么一条 class-1 `authorized_exception`**。
3. **退化（写进设计）**：单任务 stage（绝大多数）j=末段、链长 1,行为与现行**完全一致**,
   零迁移成本;`tasks[]` 缺省时维持现行单指纹断言不变。
4. bookticker 表达：review-1 覆盖到 Task B（前缀 j=B）;Task C 要么补审、要么按用户已有
   授权记 class-1 例外——D-A 两条转绿路径都有表达。

> 〔evolve-by-note 订正,2026-07-17,直修弧 red-gate-greening-v1〕**本节 v1 链式模型已被
> D3-v2 路标模型取代**（裁决：funding_hedging `reports/agent-runs/2026-07-red-gate-greening-v1/
> 06-direction-ruling-d3-fable5.md`；落地：同弧 T1）。v1 的两处假设被真实数据证伪/判险：
> ①链式要求 `task[i+1].base==task[i].head` 在真实簿记下结构性不可满足（任务起点坐标 +
> 任务间必然插入 bookkeeper 证据提交）——bookticker 与 funding-annualized-history 的
> 真实记录均证伪；②task own-review 按任务本地 dev 坐标重算担保,存在 dev-diff 冒充集成
> 段的口子,且 0/23 stage 生产使用。v2 不变量改为：**路标分段 + 逐段担保**（快照差天然
> 无缝,唯一问题是每段是否被担保）——担保仅两条:top-level review 前缀命中（全量命中
> j=n 即全段担保,无需 waypoints）与 class-1 例外（scope=review_k 盖该 review 命中前缀
> 之后的尾随段,无命中则不担保任何东西;scope=task:<id> 盖 task.head 所在段,task.head
> 须为已声明路标）。`coverage_waypoints[]` 可选、缺省 `[base,head]`;`covers_through_task`
> 退役（0/23 使用,忽略保留兼容）;task.base/head 降级为簿记元数据。现行规范以
> `docs/harness-design.md` D3 段与 `scripts/validate-stage.py:validate_task_coverage`
> docstring 为准。

## 4. Validator 改动清单（小 diff）

| 函数 | 改动 |
|---|---|
| `validate_acceptance()` :791-806 | `review.diff_fingerprint != status` 断言前,插入:若存在命中当前指纹的合规 class-1 例外则降级 PASS-with-exception;否则维持 fail。verdict==ACCEPT / json_schema_valid / tests 断言**不变**（class-2 不收）。 |
| 新增 `validate_authorized_exceptions()` | 校验每条例外:`assertion_id` 命中枚举、`authorizer=="user"`、`evidence_file` 存在、`applies_to_fingerprint`==status.diff_fingerprint;违反即 fail（negative-list #4/#5）。 |
| 新增 `validate_task_coverage()` | 链式 + 前缀校验(D3);`tasks[]` 缺省则跳过（退化）。 |
| `ALLOWED` 枚举 / 常量 | 加 `AUTHORIZED_EXCEPTION_ASSERTION_IDS = {"review_fingerprint_trails_status"}`。 |
| stdout | pre-accept 打印例外应用行。 |

- schema：`review-verdict.schema.json` 不动（例外不在 verdict 里）;`_template/status.json`
  加 `authorized_exceptions: []` 与 `tasks[].covers_through_task` 示例。若有 status schema
  文件则同步；无则仅模板。
- 说明书：`AGENTS.md` Hard Gates + `docs/harness-design.md` pre-accept 段增补例外机制与
  negative list（必要文档同步,属本 stage）。

## 5. 历史红门转绿（接受 + cp 后做,不在本 stage diff）—— 精确版

- **bookticker-open-columns-v1**：✅ Stage A v1 可**真转绿**——其病是 Task C 后加指纹
  （class-1 例外 + D3 前缀,授权证据已在账）。
- **docs-truth-sync-v1**：review-1 豁免部分是 class-1,可清;但 `review_2==REWORK` 是
  class-2（v1 不收）→ 其 pre-accept 这条**仍红**。**该 stage 已由用户接受覆盖合并,不再
  跑 pre-accept**,故维持"user-acceptance-override"记录即可,**不追求 pre-accept 全绿**。
  （订正 82 文件"两个都转绿"的表述。）

> 〔evolve-by-note 订正,2026-07-17,直修弧 red-gate-greening-v1〕bookticker 的"真转绿"
> 最终**不是由本 stage 的 D3-v1 达成**：D3-v1 链式检查使 bookticker 在 pre-accept 出现
> 不可豁免红（`task chain broken` ×3）,转绿实际由直修弧 T1(D3-v2 路标模型,链式删除)
> + T3(追加 class-1 例外 scope=review_1)完成——与本节设想的"class-1 例外 + D3 前缀"
> 不同在:前缀担保实际经由 review-2 全量命中（j=n）达成,例外只降级 review-1 尾随断言。
> docs-truth-sync 结论不变：维持 user-acceptance-override,在 fixture 台账登记为
> known_red(class-2, pending user decision D-i)。

## 6. Bootstrapping（自指)

本 stage 改 validator,自身仍由改前逻辑把关。验收含:**用改后脚本对本 stage 自己跑
pre-review/pre-accept 仍符合预期 + 现有行为零回归**。对 funding_hedging 的效果在 cp 后的
下一个 stage 实测。

## 7. Acceptance Criteria（可机械复核）

1. 构造带合规 class-1 例外(evidence_file 存在、指纹命中)的样例:pre-accept 从 FAIL →
   PASS,且 stdout 显式打印例外应用行;去掉 evidence_file / 改 applies_to_fingerprint 不
   命中 / authorizer≠user → 仍 FAIL。
2. Negative list:给 fingerprint-recompute / clean-worktree / 身份分离 挂例外记录 → 仍 FAIL。
3. D3 链式:review-1 覆盖 A/B(前缀 j=B)、Task C 有 class-1 例外 → PASS;链断裂(base/head
   接不上)→ FAIL;单任务/tasks 缺省 → 行为同现行。
4. 现有 validator 测试零回归;改后脚本对本 stage 自身 pre-accept 符合预期。
5. `AGENTS.md`/`harness-design.md` 已同步描述例外机制 + negative list。
6. **rework-1 finding-3 对抗清单**(§2.1):evidence_file 绝对路径 / `../` 逃逸 / untracked /
   `evidence_sha256` 不匹配 → 仍 FAIL;**post-head 但 tracked+digest 匹配 = 合法 PASS**
   (取代旧"post-head fails")。缺 `reason` / `at` 非 ISO-8601 → FAIL。
7. **rework-1 finding-1/2/4 对抗清单**:仅靠 task 例外 PASS 时横幅**打印全部** applied
   `assertion_id@scope`(含 task scope);伪造 `{"review_1":{"diff_fingerprint":"x"}}` 冒充
   task own-review → FAIL(要求完整合规 ACCEPT + 跨 provider + 指纹==task 指纹);裸 `<id>`
   scope / 重复或缺失 task id / 一个例外覆盖两 task → FAIL。
8. **Q2 文档**:`AGENTS.md`/`harness-design.md`/本设计 §2 的"防自授"表述已改为防静默+强制
   人核两层,并标注人核是流程义务(validator 不强制)。

## 8. Routing

实现 `claude_glm`（validator/schema/backend 域）→ review-1 Kimi → review-2 Codex
（证据完整性主干,严审）。单实现者。

当前 Session ID: unavailable
Session ID 来源: unavailable
原始输出路径: reports/agent-runs/2026-07-harness-authorized-exception-v1/10-design.md
本地北京时间: 2026-07-17 01:02:53 CST
下一步模型: human（认可设计）→ Claude-GLM 实现
下一步任务: 认可 10-design；据此派实现 dispatch
