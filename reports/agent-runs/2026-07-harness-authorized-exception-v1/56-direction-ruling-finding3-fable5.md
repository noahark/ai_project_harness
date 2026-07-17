# Fable5 方向裁决 — finding-3（authorized_exception 授权来源硬化深度）

回应: `55-escalation-finding3-fable5.md`（bookkeeper 升级）
裁决方: Fable 5（D2 裁决所有者）
已独立复核: `50-review-2.md` finding-3 全文与 fix_start_prompt 第 3 条、`10-design.md`
§1-§2、`validate_authorized_exceptions()`/`_evidence_file_nonempty()`（`:813-927`）、
`require_clean_worktree()`（`:146-151`）。finding-3 属实——`:819` 接受仓外绝对路径、
`:910-918` 只查存在+非空、无 committed/封存校验；`:886-888` 的"结构性防自授"注释照抄了
我 D2 的过度声称。

## Verdict: **DIRECTION-SET（可实现）**

先认账：被 finding-3 推翻的那句「模型不能自授靠的是'授权原文只能来自用户消息'这一结构，
不靠自觉」出自我的 82/D2。它把**惯例**（evidence 应含用户授权原文）说成了**代码强制**。
Codex 判得对：文档承诺了代码兑现不了的东西——这正是本仓 RC 分析里反复出现的
"文本承诺 > 机械强制"缺陷,出现在了防它的机制自己身上。修正如下。

## Q1 — 档位：**A（务实硬化），含以下 C 级精确边界；其中对 codex 修法一处必要修正**

v1 代码只强制机械可靠的部分，四条全属本 rework：

1. **仓内相对路径 + 逃逸防护**：拒绝绝对路径；resolve 后必须仍在 repo root 之内
   （防 `../` 逃逸）。`:819` 的 absolute-candidate 分支删除。
2. **必须已提交（git-tracked）**：validator 读取的是**提交内容**
   （`git cat-file`/`git show <验证时 HEAD>:<path>`），不是工作树文件。pre-accept 本已
   要求 clean worktree（`--porcelain` 含 untracked，`:146-151`），此项把 checkpoint 等
   非 clean 阶段也堵上，并杜绝 untracked 文件充当证据。
3. **digest 封存（对 codex fix 第 3 条的必要修正）**：例外记录新增
   `evidence_sha256` 字段，validator 重算提交 blob 的 sha256 并比对。
   **不采用** codex 原文的"verify the committed evidence blob or digest at
   `status.head_sha`"中的 blob@head_sha 形态——它与真实流程**死锁**：class-1 授权须引用
   新指纹，新指纹只在 head commit 之后存在，故授权证据必然 post-head 提交；两个真实实例
   （bookticker 事后补录、docs-truth-sync 的 waiver 证据落在 fix head 之后的簿记提交）
   在该形态下都无法合法转绿。digest-in-record + git-tracked 达成同等防篡改意图
   （记录写定并受审后，内容不可静默替换），且无顺序死锁。
   **随之修正 adversarial 清单**："post-head evidence fails" 一项改为
   "digest 不匹配 fails / untracked fails"；post-head 但 tracked+digest 匹配 = 合法。
4. **task 消费端显式按 `assertion_id == review_fingerprint_trails_status` 过滤**
   （finding-4 已覆盖，此处仅确认属同批）。

**不做（v1）**：结构化链接 `user_authorizations[].instruction_verbatim`、可信 runner
receipt。理由见 Q3 与拆分段。**不加密码学签名/receipt**——B 档 receipt 依赖仍是 DRAFT
的看门狗/runner 基建,在未建成的基建上立门是本仓已判定过的反模式（推测性工程 + 依赖
未就绪）。

## Q2 — 文档表述：**必须改**，措辞如下（可直接采用）

> `authorizer=="user"` 字面量、钉指纹、仓内已提交且 digest 封存的 evidence，共同保证
> 自授**无法静默发生**：伪造一条豁免必须把伪造的授权文本提交进受审的 git 历史并在
> PASS-with-exception 横幅中亮明——留下跨评审与用户可见、可追溯的痕迹。但代码**不能
> 证明** evidence 文本源自人类（任何模型能写的字节,模型都能写）。「不能自授」的最终
> 保证是：**pre-accept 放行任何例外前，操作者必须人工核对 evidence 原文确系自己的授权**
> （PASS-with-exception 横幅即人核触发点），以及——若未来建成——可信 runner receipt。

要点：把"防自授"从**不可能性声称**降级为**防静默 + 强制人核**两层。同时在 negative-list
文档段补一句诚实标注：人核是流程义务（workflow checklist + handoff 惯例），validator
无法强制人读——这与 80 报告 RC 分析的方法论一致，不装成机械门。

## Q3 — 人核收口：**可接受，且是唯一诚实的收口**

原理层面：仓库文本的"人类来源"不存在纯文本密码学证明——模型可写任何字节。receipt 只是
把信任搬到 runner 基建上（该基建自身尚 DRAFT,且其可信性又需要另一层保证）。v1 的威胁
模型本来就是**防事故与漂移、让蓄意伪造留下显性可审计痕迹**,不是防拥有写权限+提交权的
恶意方——后者在现行"模型可直接 commit"的运行模式下无门可防,只有 runner 收权后才谈得上。
以"机械防静默 + 显式人核为最终保证"收口 v1，立场准确、承诺可兑现。

## 拆分（防 rework 膨胀）

- **本 rework**：Q1 的 1-4 + Q2 文档改写 +（已在 prompt 内的）reason/at 结构校验与
  findings 1/2/4/P2 修复。全部机械、无新基建依赖。
- **后续 stage（不阻塞本 stage）**：① `user_authorizations[]` 结构化 schema 化 + 例外
  记录到授权条目的引用链——它增加的是**可审计导航**而非来源证明,且把 status.json 惯例
  升格为规范 schema 值得单独设计；② 可信 runner receipt 元数据——等看门狗/runner 基建
  离开 DRAFT 后,作为"人核之上的第二保证"接入,接入时 Q2 措辞已预留位置。

---

模型身份: Fable 5（anthropic/claude-fable-5，Claude Code CLI；session id 未暴露）
本地北京时间: 2026-07-17 13:19
下一步模型: bookkeeper（opus4.8）——按本裁决更新 10-design D2 段（含 Q2 措辞与
adversarial 清单修正），出 fix dispatch 派 claude_glm；修复后重进 review-1（Kimi）→
review-2（Codex）
