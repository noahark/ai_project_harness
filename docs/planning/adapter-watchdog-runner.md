# Adapter Watchdog Runner - Draft Proposal

状态：DRAFT-2，吸收 Fable5 round-1 review（REWORK，方向 ACCEPT）。
日期：2026-07-06。
适用范围：Harness 模型适配器调用、嵌入预审、正式 review-1、review-2。

## 1. 背景

当前 Harness 已经把模型命令集中记录在 `docs/model-adapters.md` 和
`agents/registry.yaml`，但实际 stage dispatch 仍会把裸命令写入任务书。
这导致两个问题：

1. 外层执行器可能使用自己的默认 timeout。`2026-07-private-account-v1`
   中，Kimi 按 R10 调用 Claude-GLM 预审时被 300 秒杀掉，而 registry 中
   `claude_glm.timeout_seconds` 已是 3000。
2. 模型 CLI 有时并非真正推理，而是卡在会话恢复、权限确认、交互输入、
   认证提示或无输出等待。单纯把 timeout 拉到 60-120 分钟，会重演
   "codex 一次性 review 等了一晚上" 的问题。

DRAFT-1 的缺陷是只把裸命令包进 runner，但没有解决触发事故的真正机制：
如果 runner 仍作为调用方 agent-harness Bash 工具的前台子进程运行，它会被
同一个外层 300 秒超时杀掉。DRAFT-2 因此把 **detach watchdog** 作为核心
能力，而不是可选增强。

## 2. 目标

- 统一模型调用入口，不让任务书手写裸 `claude-glm` / `kimi` / `codex`
  长命令。
- 从 registry 读取 adapter 命令、timeout、shell 包装要求、provider
  identity、output style。
- 支持 detach 模式：从短超时 agent shell 启动后立即脱离，返回 dispatch
  id，后台 watchdog 继续运行。
- 区分 "模型还在运行" 与 "CLI 卡死/等待交互"。
- 每隔固定间隔写 probe 日志，保留可审计证据。
- 在明确卡死或达到 hard timeout 时终止整个进程组并封存证据。
- 不引入第二套 diff fingerprint；runner 只管进程与输出证据。

## 3. 非目标

- 不改变 review verdict schema。
- 不改变 Hard Gates 的 committed-state review 要求。
- 不把 embedded pre-review 升级为正式 review gate。
- 不负责判断代码是否通过；它只负责可靠运行、终止、落档和分类失败。
- 不自动选择 fallback 模型；fallback / bookkeeper decision 仍由 workflow
  和 status.json 决定。
- 不要求当前已发出的 immutable stage dispatch 中途改写；当前 stage 的
  失败预审可由人工普通终端补跑，runner 从下一次采用新 packet 的 stage
  开始试运行。

## 4. 建议 CLI

新增脚本：

```bash
python3 scripts/run-model-adapter.py \
  --adapter claude_glm \
  --mode read_only_review \
  --prompt reports/agent-runs/<stage-id>/pre-review-task-b-by-glm.prompt.md \
  --output reports/agent-runs/<stage-id>/embedded-review-b-round1.raw-output.md \
  --probe-log reports/agent-runs/<stage-id>/embedded-review-b-round1.probe.jsonl \
  --dispatch-log reports/agent-runs/<stage-id>/embedded-review-b-round1.dispatch.md \
  --stage-id <stage-id> \
  --detach
```

`--mode` 建议枚举：

- `noninteractive`
- `read_only_review`
- `development`
- `schema_review`
- `yolo`

`--detach` 是被模型/agent 终端调用时的默认要求。runner 必须：

1. 生成 dispatch id。
2. 写入 `dispatch.md` 初始状态：`status=running`、pid、pgid、started_at、
   prompt/output/probe 路径、redacted command。
3. 以独立进程组启动 watchdog worker（例如 `setsid` 或 Python
   `subprocess.Popen(..., start_new_session=True)`）。
4. 立即返回 dispatch id 与文件路径，确保调用方不会被外层 300 秒 timeout
   拖死。

前台等待模式只允许人工终端或显式 runner 调试：

```bash
python3 scripts/run-model-adapter.py ... --foreground
```

`--foreground` 不得出现在 R10 实现任务 dispatch 的默认命令里。

## 5. Detach 后的职责边界

R10 的 "executor:self" 在 detach 模式下含义调整为：实现终端必须成功启动
对侧预审 dispatch，并落档 dispatch id；它不要求实现终端一直等待到预审完成。

运行结果由两种方式收口：

1. 如果预审很快完成，执行者可以读取 raw output，按 PASS/BLOCKER 分支处理。
2. 如果预审仍在后台运行，执行者输出 dispatch id 和路径后停下；bookkeeper
   轮询 `dispatch.md` / `probe.jsonl`，并在完成后决定 PASS、scope 内 fix、
   reviewer reassignment 或 escalation。

这不改变 embedded pre-review 的性质：它仍是 checkpoint，不是 formal
review-1 gate。

## 6. 默认时间预算

registry 中每个 adapter 应明确四类时间参数；**per-adapter registry 值是
权威**，下表只是缺省回退：

```yaml
timeout_seconds: 7200
idle_timeout_seconds: 1800
probe_interval_seconds: 180
startup_grace_seconds: 300
output_style: streaming
```

建议缺省值：

| 场景 | probe interval | idle timeout | hard timeout |
|---|---:|---:|---:|
| embedded pre-review | 180s | 1800s | 7200s |
| formal review-1 | 180s | 1800s | 7200s |
| review-2 | 180s | 2700s | 10800s |
| implementation | 300s | 3600s | 14400s |

说明：

- `hard timeout` 只作为兜底，不是正常终止判断。
- `idle timeout` 只对适用的 adapter output style 生效。
- `startup_grace_seconds` 允许 CLI 初始化、加载长上下文、恢复 session，但
  若 grace 后仍只有交互提示或无正文输出，应按卡住处理。

## 7. Output Style

远程模型推理时，本地 CLI 可能只是阻塞在 socket 上等待响应。此时 CPU 接近
0、输出无增长、无子进程活动并不等于卡死。runner 必须按 adapter 的
`output_style` 判断 idle：

```yaml
output_style: streaming | buffered
```

- `streaming`：输出增长可作为活跃信号；长时间无输出可参与 idle 判定。
- `buffered`：一次性输出型 CLI 在推理期间可能没有任何输出。除启动期交互
  卡点外，不得仅因无输出增长或 CPU 低而 idle kill；可选检查存活 socket，
  否则只按 hard timeout 兜底。

CPU 有活动、输出有增长、socket 仍存活都应重置或延后 idle 判定。CPU 无活动
本身不是卡死证据。

## 8. 探针判定

每个 probe interval，runner 记录：

- 当前时间。
- elapsed seconds。
- pid / pgid / child pids。
- 输出文件字节数。
- output bytes delta。
- stdout/stderr 最近 N 行摘要。
- 进程状态。
- CPU 近似使用率（可用时）。
- socket 存活情况（可用时）。
- 判定：`continue` / `terminate` / `escalate_candidate`。

继续等待条件：

- 输出文件仍在增长；或
- CPU / 子进程 / socket 活动显示仍在工作；或
- 最近输出是模型正文、测试进度、工具调用日志；或
- 尚未超过 startup grace；或
- adapter 是 `buffered` 且未达到 hard timeout，也没有明确交互等待证据。

主动终止条件：

- 到达 hard timeout。
- `streaming` adapter 超过 idle timeout 且输出无增长、无活动。
- 输出末尾显示等待交互，并在 startup grace 后无模型正文、无输出增长。
- adapter 进程已退出但没有成功状态或有效输出。

## 9. 交互等待识别

runner 可扫描最近输出末尾若干行，识别常见卡点：

```text
Do you want to continue
Approve
Press enter
permission
continue?
login
authentication
Restored session
connectors are disabled
Cannot combine --prompt
model not found
quota
rate limit
```

这些字符串是弱信号，不是立即失败条件。必须同时满足：

1. 命中输出末尾，而不是历史全文。
2. startup grace 后仍无输出增长或实质模型正文。
3. 不是 prompt/评审对象文档被模型复述。

单独命中模式字符串永不终止。

## 10. 终止与证据封存

runner 必须用独立进程组启动 adapter，并在主动终止时杀整个进程组：

- Python: `start_new_session=True` + `os.killpg(pgid, signal.SIGTERM)`。
- 等待短 grace 后仍未退出，再 `SIGKILL`。

只杀父进程不合格，因为 `zsh -lic '...'` 会产生子进程树，孤儿 CLI 可能继续
写 raw output。

终止时必须在 `dispatch.md` 记录：

- termination_signal。
- output_bytes_at_termination。
- terminated_at。
- process_group_killed: true/false。

终止后 runner 应重新 stat raw output。若文件继续增长，记录：

```text
evidence_contaminated_after_termination: true
```

该状态必须进入 bookkeeper-decision，不能作为正常 review 证据。

## 11. 输出文件

每次 runner 调用至少落档三类文件：

```text
<run>.raw-output.md
<run>.probe.jsonl
<run>.dispatch.md
```

`dispatch.md` 建议包含：

```markdown
# Adapter Dispatch

- stage_id:
- dispatch_id:
- adapter:
- mode:
- output_style:
- command_template:
- command_redacted:
- prompt_path:
- output_path:
- probe_log_path:
- started_at:
- completed_at:
- pid:
- pgid:
- exit_code:
- termination_reason: completed | hard_timeout | idle_timeout | external_signal | command_error
- failure_class:
- timeout_seconds:
- idle_timeout_seconds:
- probe_interval_seconds:
- startup_grace_seconds:
- output_bytes_at_termination:
- evidence_contaminated_after_termination:
- last_output_excerpt:
```

`last_output_excerpt` 必须经过与 raw output 相同的脱敏规则。

`probe.jsonl` 每行示例：

```json
{"ts":"2026-07-06T10:00:00+08:00","elapsed":180,"pid":12345,"pgid":12345,"children":[12346],"output_bytes":8120,"delta_bytes":1200,"cpu_percent":18.4,"socket_alive":true,"state":"running","decision":"continue"}
```

## 12. Failure Class

建议标准化 runner 失败分类：

- `success`: 进程正常结束；`termination_reason=completed` 必填。
- `hard_timeout`: 到达 hard timeout。
- `idle_timeout`: 对允许 idle 判定的 adapter，超过 idle timeout。
- `killed_by_external`: runner 或 adapter 被外层信号杀死。
- `requires_interactive_input`: 明显等待人工输入。
- `adapter_missing`: 命令不可解析。
- `model_unavailable`: 模型不存在或不可选择。
- `auth_failure`: 认证失败。
- `quota_exhausted`: 额度/rate limit。
- `service_unavailable`: provider 或网络故障。
- `invalid_invocation`: 参数组合错误，例如 Kimi `--plan -p`。
- `command_error`: 非上述类别的非零退出。
- `evidence_contaminated`: 终止后 raw output 继续变化。

映射回 workflow：

- embedded checkpoint：进入 `bookkeeper-decision`，不直接 terminal。
- review-1：按 workflow retry / reassign / human escalation。
- review-2：仅 quota/auth/service/timeout/合法 anti-self-review 等 workflow
  允许原因可触发 fallback；`requires_interactive_input` 和
  `invalid_invocation` 不得伪装成 `model_unavailable`，应先修命令后重试。
- 两个 decision models 都因合法不可用原因失败，才进入
  `decision_models_exhausted`。

## 13. Yolo / Bypass 保护

`--mode yolo` 必须显式要求：

```bash
--user-authorized
```

没有该参数时 runner 拒跑。该参数只证明用户或 stage 明确授权 yolo，不改变
文件边界、证据落档和 Hard Gates。

## 14. 与 R10 的关系

`docs/parallel-development-mode.md` R10 当前要求任务书写死真实路径和 adapter
命令。引入 runner 后，下一次采用新 packet 的 stage 中，R10 应改为：

1. 实现任务 prompt 仍写死真实路径。
2. 对侧预审命令统一写为 `scripts/run-model-adapter.py --detach` 调用。
3. runner 命令必须包含 raw output、probe log、dispatch 路径。
4. 裸 `claude-glm ... | tee ...`、`kimi ... | tee ...` 不再满足 R10。

示例：

```bash
python3 scripts/run-model-adapter.py \
  --adapter claude_glm \
  --mode read_only_review \
  --prompt reports/agent-runs/2026-07-private-account-v1/pre-review-task-b-by-glm.prompt.md \
  --output reports/agent-runs/2026-07-private-account-v1/embedded-review-b-round1.raw-output.md \
  --probe-log reports/agent-runs/2026-07-private-account-v1/embedded-review-b-round1.probe.jsonl \
  --dispatch-log reports/agent-runs/2026-07-private-account-v1/embedded-review-b-round1.dispatch.md \
  --stage-id 2026-07-private-account-v1 \
  --detach
```

当前已经发出的 immutable dispatch 不回改；当前 stage 的 Claude-GLM 预审若
需要补跑，应由人工普通终端直接运行原 R10 命令或临时手工 watchdog，而不是
把 DRAFT-2 规则倒灌进旧 packet。

## 15. Registry 变更建议

`agents/registry.yaml` 中每个 adapter 增加：

```yaml
timeout_seconds: 7200
idle_timeout_seconds: 1800
probe_interval_seconds: 180
startup_grace_seconds: 300
output_style: streaming
requires_login_shell: false
```

需要注意：

- `claude_glm.requires_login_shell: true`，因为本地 `claude-glm` 是 zsh alias。
- `claude_glm.output_style` 需要实测。若 `claude -p` 类输出是 buffered，
  禁用 idle kill，只保留 startup 交互检测 + hard timeout。
- `kimi` 需要补齐 timeout 字段，避免继续使用外层默认值。
- `codex` review 也应走 runner，防止等待交互或模型选择失败时挂一晚上。
- `claude` probing 仍必须遵守 Anthropic/GLM auth reroute hygiene。

## 16. STARTED / HEARTBEAT 约定

prompt 可加入辅助约定：

```text
启动后先输出 STARTED <model> <time>。
长任务每 5 分钟尽量输出 HEARTBEAT。
```

这只作辅助信号：

- 出现 `STARTED` 可帮助区分启动期交互卡点。
- streaming CLI 出现 `HEARTBEAT` 可重置 idle。
- buffered CLI 推理中无法输出 heartbeat；缺失 `HEARTBEAT` 不得作为终止
  或失败依据。

runner 不能依赖模型配合看门狗。

## 17. Validator 关系

短期不要求 `validate-stage.py` 校验 probe 日志，否则会阻塞现有 stage。

建议分两步：

1. DRAFT/试运行阶段：runner 落档 probe log，但 validator 不强制。
2. ADOPTED 后：parallel mode 的 embedded review 要求存在：
   - raw output
   - dispatch log
   - probe log
   - failure_class（失败时）

## 18. 安全与凭证

runner 必须：

- 记录 redacted command，不记录扩展后的 alias 环境。
- 不落完整 env dump。
- 对 known secret env key 做输出脱敏。
- 保留 stderr/stdout 片段时，按 secret pattern 做最小脱敏。
- 不把 private key、API key、cookie、token 写入 dispatch/probe。
- 对 `last_output_excerpt` 使用与 raw output 相同的脱敏规则。

## 19. 试运行计划

1. Fable5 review DRAFT-2。
2. 若 ACCEPT，作为 Harness 小阶段进入 stage design。
3. 在模板仓 `ai_project_harness` 先实现：
   - `scripts/run-model-adapter.py`
   - registry timeout/output_style 字段补齐
   - `docs/model-adapters.md` runner 用法
   - `docs/parallel-development-mode.md` R10 命令改写
   - `workflows/templates/stage-delivery.yaml` adapter watchdog 约束
4. sync 回 `funding_hedging`：等当前业务 stage 合回 main 后执行，不在当前
   stage 分支中途同步 harness。
5. 用下一次 Kimi -> Claude-GLM 或 Claude-GLM -> Kimi 预审做实测。

## 20. Fable5 Round-1 Findings 吸收索引

- P1-1：已新增 detach 模式、独立进程组、dispatch id、立即返回。
- P1-2：已新增 `output_style`，buffered adapter 禁止用无输出/低 CPU
  直接 idle kill。
- P1-3：已新增 kill process group、termination byte seal、污染检测。
- P2 交互等待：已改为输出末尾 + 无增长 + grace 后三条件。
- P2 failure class：已新增 `killed_by_external`，success 必填
  `termination_reason=completed`。
- P2 Q6：`requires_interactive_input` 单独进入 bookkeeper-decision，不伪装
  `model_unavailable`。
- P2 Q7：`STARTED` / `HEARTBEAT` 只作为辅助信号。
- P3：`--mode yolo` 要求 `--user-authorized`，excerpt 同步脱敏。

## 21. 给 Fable5 的 DRAFT-2 review 启动文案

```text
请只读 review docs/planning/adapter-watchdog-runner.md（DRAFT-2）。

背景：DRAFT-1 因未处理外层 agent-harness 300 秒超时、buffered CLI 无输出误杀、
进程组终止与证据封存不足，被你判定 REWORK。DRAFT-2 已加入 detach watchdog、
output_style、killpg/termination byte seal、failure_class 修订、R10 下一阶段
硬化路径。

请重点核对：
1. P1-1/P1-2/P1-3 是否完整吸收；
2. detach 后 R10 executor:self 的职责边界是否合理；
3. buffered adapter 的 idle 策略是否足够保守；
4. failure class 与 review-2 fallback/override 是否避免虚假路由；
5. 当前 stage 不回改 immutable dispatch、下一 stage 起试运行的时序是否正确；
6. 是否可以进入 Harness 小阶段实现。

输出 verdict：ACCEPT / REWORK / BLOCKED。
若 REWORK，请列 P1/P2/P3 findings 和具体修法。
末尾请注明模型身份、本地北京时间、建议下一步调用模型。
```

本地北京时间: 2026-07-06 10:02:33 CST
记录: Codex/GPT 起草 DRAFT-1；DRAFT-2 吸收 Fable5 round-1 review
