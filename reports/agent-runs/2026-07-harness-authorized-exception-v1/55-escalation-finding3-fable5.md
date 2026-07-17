# Direction Escalation — Stage A finding-3（authorized_exception 授权来源硬化深度）

**升级方:bookkeeper（Claude Opus 4.8）。裁决方:Fable5（方向权威,拥有 D2 裁决）。**
仅需方向裁决,不需实现。落点:模板仓 `ai_project_harness`,stage
`2026-07-harness-authorized-exception-v1`(rework_count=1/3)。

## 为什么升级你

review-2(Codex,证据完整性主干终审)判 REWORK,其中 **finding-3(P1)** 直接改动了**你在
D2 的裁决**,故不由 bookkeeper/实现者自行决定硬化深度,请你出方向。findings 1/2/4 + 两条
P2 是无争议代码假绿,已照修,不在本升级范围。

## 你原来的 D2 裁决（原文,来自 82-stage-a-design-rulings-fable5.md）

> D2: authorized_exceptions:[{assertion_id, scope, applies_to_fingerprint (harden 1:
> pinned to exact fingerprint, auto-expires on next fix), reason, **authorizer=='user'
> literal (harden 2: evidence_file must contain verbatim user authorization)**,
> evidence_file (must exist, :735-737 pattern), at}]; PASS-with-exception is explicitly
> printed, never silent; no expiry field.

10-design §2 硬化2 进一步表述为:**「模型不能自授靠的是"授权原文只能来自用户消息"这一
结构,不靠自觉。」**

## Codex finding-3 说这个表述不成立（P1）

实现落地的 `validate_authorized_exceptions()` 对 evidence_file 只做:**存在 + 非空**
(`_evidence_file_nonempty`),`authorizer` 只做**字面量 `"user"`** 检查。它**没有**:
- 校验文件内容确实是用户授权原文(12-breakdown §B 明示「至少非空;不做 NLP 校验」);
- 把 evidence 限制为**仓内相对路径**(现接受仓外绝对路径,Kimi P2-3 亦标);
- 把 evidence 按 **head_sha 封存**(committed blob/digest);后续改动 evidence 不被发现;
- 链接到 `status.user_authorizations[].instruction_verbatim` 这类结构化来源。

**后果**:一个有写权限的写入方可以**同时**造出例外记录**和**那份"非空 evidence 文件",
自授一个 waiver 并拿到 validator PASS。所以 harden-2 的「evidence 含真实用户授权」在代码层
**没有兑现**,而文档「模型不能自授」是**过度声称**。

（补背景:review-1 Kimi 曾把这条标为 P2-2「靠人核、非代码强制」,判非阻塞;Codex 升到 P1,
理由是这是证据完整性主干、且文档做了代码兑现不了的承诺。）

## 需要你裁决的问题

**Q1（核心）：v1 代码硬化到多深?** 三档:
- **A 务实硬化**:代码只强制可靠的机械部分 —— evidence_file 必须①仓内相对路径(拒绝
  绝对/仓外)②已 committed 且③按 head_sha 封存 blob/digest;task 例外显式按 assertion_id
  过滤。**放弃**「代码证明人类来源」,把文档改为「结构检查 + **强制人核**为最终保证」。
  不建加密 receipt。
- **B 最大硬化**:A + 结构化链接 `user_authorizations[].instruction_verbatim` + 可信
  runner receipt 元数据。**注意**:receipt 依赖尚是 DRAFT 的看门狗/runner
  (`docs/planning/adapter-watchdog-runner.md`),v1 落这条有过度工程 + 依赖未建基建的风险。
- **C 其它**:你给的更精确边界。

**Q2：文档表述。** 确认「模型不能自授」这句**必须改**为诚实版(代码做结构检查、人核是最终
保证)?还是你有更准的措辞?

**Q3：人核作为 v1 最终保证是否可接受?** 即:承认「仓库文本本身不是人类来源的密码学证明,
终极保证是可信 runner receipt 或显式人工核对 evidence 原文」,v1 以此立场收口是否 OK?

## 请先读（本仓内)

- `reports/agent-runs/2026-07-harness-authorized-exception-v1/50-review-2.md`
  （finding-3 全文 + `required_fixes` + `fix_start_prompt` 第 3 条）
- `.../10-design.md` §1（negative list）、§2（D2 两处硬化）
- `.../20-implementation.md` §3–§4（negative-list 物理隔离 + 钉指纹）
- 代码锚点:`scripts/validate-stage.py` 的 `validate_authorized_exceptions()`、
  `_evidence_file_nonempty()`、`evidence_path_exists()`。

## 请给

- Q1 档位裁断（A/B/C,含边界）、Q2 文档措辞立场、Q3 人核收口是否可接受;
- 若选超出 A 的范围,请指明哪些属**本 rework**、哪些应拆到后续 stage(避免 rework 膨胀,
  且 receipt 依赖未就绪)；
- verdict：DIRECTION-SET（可实现）/ NEEDS-MORE。
- 末尾:模型身份、本地北京时间、下一步模型。

裁决后:bookkeeper 据此更新 10-design 的 D2 段 + 出 fix dispatch(findings 1/2/4 + P2 +
finding-3 按你的档位)派 claude_glm;修复后重进 review-1(Kimi)→ review-2(Codex)。
