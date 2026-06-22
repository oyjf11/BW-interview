# Agent 评测与可观测性 · 基于个人项目的面试应答文档

> 配套《Agent 评测与可观测性 · 面试官评分手册》20 题。每题以我的真实项目为底座作答，未覆盖的考点做合理补全。
>
> **标记说明：**
> - `✅[真实]` —— 项目里确有代码/设计支撑。**自信、具体地说**，敢报细节（文件名、类名、函数名、字段名）。
> - `⚙️[补全]` —— 项目未覆盖但合理延伸。**稳住说设计思路**，别报实现细节，被追问时落到"会这么设计"而非"我已经这么做了"。
>
> **主力项目缩写：** SRE = SRE Agent (OpsPilot) | HS = Harness System | RAG = ai-chat-root | PPT = AI-PPT
>
> **面试官最看重的拉档题：** Q1 / Q6 / Q10 / Q14 / Q19 / Q20

---

# 一、可观测性设计原则与指标体系

## Q1　Agent 可观测性的四维度设计：功能 / 性能 / 适应 / 协作

> **主接项目：SRE Agent 的 Tool Gateway（功能/性能）+ multi-provider LLM（适应）+ evidence_fanout 多专家（协作）。** 四个维度都有代码支撑，实际并非一个单独的"四维度框架"类，而是分散在架构各处做实了。

### 🟢 我命中的参考答案要点

- ✅[真实] **功能维度**：`backend/app/tools/gateway.py:952-977` 的 `ToolGateway.describe_capability()` 是真实的能力探测——传工具名返回 available/not available + adapter 模式（mock vs real）。不是靠模型自己说"我能调用啥"，而是网关层确定性回答。`backend/app/tools/gateway.py:979-1063` 的 `call_tool()` 每调用一次记录 latency_ms（line 1050）、success/fail、adapter 模式。
- ✅[真实] **性能维度**：`backend/app/tracing.py:44-70` 的 `AgentTracer.start_span()` 记录 start_time / start_timestamp，`end_span()`（line 72-81）计算 duration_ms。`backend/app/tools/gateway.py:1092-1132` 的 `_log_audit()` 每条工具调用审计含 latency_ms。`backend/app/evals/metrics.py:83-104` 的 `compute_metrics()` 输出 avg_latency_ms。性能埋点在三个层级（span / 工具审计 / eval 聚合）都有。
- ✅[真实] **适应维度**：`backend/app/llm_client.py:70-97` 的 `LLMClient.complete()` 支持 3 种 provider（minimax/openai/deepseek）动态切换，tracer 记录 provider + model 到 llm_request event。换模型后上层 14 节点图逻辑不依赖具体模型——适配在 llm_client 层收敛。
- ✅[真实] **协作维度**：`backend/app/graph/nodes/specialist_agent.py:241-244` 的 `asyncio.gather` 多 specialist 并行执行；`backend/app/graph/nodes/aggregator.py:138` 的 `_detect_cross_agent_contradictions` 做跨 Agent 矛盾检测；`backend/app/graph/state.py:81-83` 的 agent_tasks / specialist_analyses / cross_agent_causal_chains 字段跟踪协作状态。
- ⚙️[补全] **适应维度的自动降级/自动切换**：llm_client 有 multi-provider 支持，但切换需要改环境变量重启，无自动熔断或 fallback 链。设计上我会加 provider 健康检查 + 自动 fallback。
- ⚙️[补全] **功能维度的能力图谱**：`describe_capability` 是按需查询单工具，不是主动探测 + 生成完整能力拓扑图的设计。

### 🟡 我能拿到的加分项

- ✅[真实] **区分监控与可观测性**：我的 trace span 层级是 `graph.run → node.{name} → tool.{name} / llm.{provider} / specialist.{agent_id}`——不是平面日志，而是层次化记录。从任一 span 可以下钻到子 span 或关联到 events 数组，做出问题"不看完整日志也能推断决策链路"。
- ⚙️[补全] **MCP 原生可观测性**：HS 的 tool registry（L0-L3 分级）+ opencode plugin 的 tool.execute.before hook 已经是协议层治理的雏形。MCP 标准接入后审计和 trace 会进一步下沉到协议层。
- ✅[真实] **四个维度互为输入**：适应维度的"换模型后成功率如何"用 eval `--repeat N` 验证，结果反馈到性能维度（延迟变了没）和功能维度（新模型兼容之前的工具调用吗）。

### 🔴 危险信号（主动规避）

- ❌ 别说"打日志 + Grafana 就是可观测"——我强调 LLM 黑箱下靠层次化 trace 反推决策链路。
- ❌ 别说"四维度是一个实现了的框架类"——坦诚说这四个维度的能力分散在架构各处真实落地，但不是一个叫 `FourDimensionsFramework` 的类。
- ❌ 别说"换更强模型就都解决了"——适应维度的难题在于非确定性行为变化，不是模型强就能消除的。

### 完整应答（口语稿）

> Agent 可观测性我会从四个维度拆，每个都不是空中楼阁。第一个功能维度——Agent 到底能执行什么操作，不是靠模型自述，而是靠网关层的 `describe_capability` 确定性回答某个工具当前是不是可用、走的 mock 还是真实适配器。在我的 SRE Agent 里，risk_gate 节点执行前调这个方法做能力预检，适配器没接就直接置成 NEEDS_HUMAN，绝不放行。
>
> 第二个性能维度，埋点在三个层级。最低层是每次工具调用记录 latency_ms，中间层是 tracing 模块的 span 记 start_time/end_time/duration_ms，上层是 eval 框架的 compute_metrics 输出 avg_latency_ms。三层的颗粒度不一样，分别给不同的角色看。第三个适应维度，换模型后行为一致性是最大的挑战。我的 llm_client 支持三个 provider 切换，上层 14 节点图逻辑不依赖具体模型——但自动降级和熔断我还没做，切换需要改 env 重启，这是诚实短板。
>
> 第四个协作维度尤其重要——多 Agent 场景下 trace 不能断链。我的 evidence_fanout 用 asyncio.gather 并发派发多个取证专家，每个专家有独立 span，parent_id 挂到 fanout 的 span 下。aggregate 节点做跨 Agent 矛盾检测，state 里记录 agent_tasks 和 cross_agent_causal_chains。最关键的是这些 trace 信息让我能在出问题时推断决策链路，而不是只看表面监控说"成功率降了"——这就是可观测性区别于监控的要点。

---

## Q2　三层指标体系：任务级 → 组件级 → 推理级

> **主接项目：SRE Agent 的 eval metrics（L1）+ tool audit（L2）+ span events（L3）。** 三层对应关系是真实存在的，虽然代码里没叫"三层指标体系"这个名字。

### 🟢 我命中的参考答案要点

- ✅[真实] **L1 任务级**：`backend/app/evals/metrics.py:83-104` 的 `compute_metrics()` 输出 top1_accuracy / top3_accuracy / risk_accuracy / status_accuracy / macro_f1 / confusion_matrix。`backend/app/evals/metrics.py:119-136` 的 `aggregate_rounds()` 输出 mean/min/max 分布。回答"这 task 到底成了没"——给产品和业务看。
- ✅[真实] **L2 组件级**：`backend/app/evals/metrics.py:26-57` 的 per-class precision/recall/f1（按 11 种 IncidentType 分别报），`backend/app/tools/gateway.py:1092-1132` 的 `_log_audit()` 每条工具调用记 latency_ms + success/fail（line 1050）。回答"哪个部件拖了后腿"——给 SRE 和工程团队看。
- ✅[真实] **L3 推理级**：`backend/app/tracing.py:92-109` 的 `AgentTracer.add_event()` 在每个 span 内嵌 events 数组，记录具体事件——`tool_called` / `tool_succeeded` / `llm_request` / `llm_response`。`backend/app/graph/state.py:53` 的 step_count 跟踪步数。`backend/app/evals/scorer.py:7-18` 的 confidence 字段记录 agent 自己的置信度。这些信息让 prompt 工程师能看"模型这一步到底在想什么"。
- ⚙️[补全] **Thought/Action/Observation 三个显式标签**：我项目里 specialist agent 内部有 ReAct round 追踪（`specialist_agent.py:184-257`），但在 EventBus 事件类型中没有 Thought / Action / Observation 的显式枚举。三层是自然对应的，但标签是设计术语。

### 🟡 我能拿到的加分项

- ✅[真实] **三层指标之间的归因链路**：工具调用成功率下降时，我能从审计表的 per-tool latency_ms 定位到具体哪个工具慢了；从 L3 的 span events 看这次调用的上下文是什么；从 L1 的 per-class f1 看哪类故障的准确率被拖累。三层不是割裂的。
- ⚙️[补全] **反思触发率的金发女孩区间**：critic 节点的 loop_count 可以推导反思触发率——当前是轮次硬上限，还没有做成"触发率指标"来监控。设计上我会把 loop_count > 0 的 run 比例作为反思触发率指标。
- ✅[真实] **复利失败公式的支撑数据**：step_count 跟踪步数，--repeat N 取分布，score_case 算最终正确性——我可以把"平均步数 + 单步质量"和"端到端成功率"的对应关系算出来，虽然目前 eval 没有显式做 per-step accuracy 评分。

### 🔴 危险信号（主动规避）

- ❌ 别说"只报成功率一个数"——我有 8+ 指标（top1/top3/risk/status/macro_f1/confusion_matrix/per-class/avg_latency）。
- ❌ 别说 L3 推理级不可观测就放弃了——我有 span events 和 step_count 覆盖推理级的基本信息。
- ❌ 别说"三层全塞一个 Dashboard"——我的三层对应不同使用者：L1 给 PM / L2 给 SRE / L3 给算法+prompt 工程师。

### 完整应答（口语稿）

> 指标体系我会分三层，每一层回答不同人的问题。L1 任务级是回答"这 task 到底成了没"——我的 eval 框架 compute_metrics 输出 top1/top3 accuracy、risk_accuracy、status_accuracy、macro_f1、confusion_matrix，这是给产品和业务看的端到端指标。L2 组件级回答"哪个部件拖了后腿"——per-class 的 precision/recall/f1 看哪类故障处置得不好，工具审计表的 latency_ms 和 success/fail 看哪个工具是瓶颈。L3 推理级回答"模型这一步到底在想什么"——tracing 模块在每个 span 里嵌 events，记录 tool_called 和 llm_request 事件，step_count 跟踪总步数，agent 自身的 confidence 也记在 case result 里。
>
> 这三层不是割裂的——工具调用成功率掉了，我能从审计表定位具体哪个工具慢了，从 span events 看当时的调用上下文，从 per-class f1 看最终哪类故障被拖累。这个归因链路是真实能走的。
>
> 要坦诚的是，Thought/Action/Observation 的显式标签我在事件枚举里还没有——ReAct 循环追踪在 specialist agent 代码里有，但 event bus 没做这三个枚举值。反思触发率目前也是轮的硬上限而不是指标。这些都是我会说"设计上会补"的延伸，但三层结构是真实存在的——我的 eval 指标、工具审计、span events 三层颗粒度的数据都有。

---

## Q3　复利失败公式：从组件指标推导端到端可靠性

> **主接项目：SRE Agent 的 step_count + Repeat N 分布指标。** 数据能支撑公式推导，但 per-step accuracy 的显式计算是补全。

### 🟢 我命中的参考答案要点

- ✅[真实] **步数和端到端指标的原始数据**：`backend/app/graph/state.py:53` 的 step_count 跟踪每个 run 的步数，`backend/app/evals/scorer.py:37-76` 的 `score_case()` 计算端到端的 hit_top1/hit_top3。两条数据在手，可以算出"平均步数 N 下的端到端成功率"。
- ✅[真实] **--repeat N 取分布**：`backend/app/evals/replay_runner.py:34,50` 的 `--repeat N` 参数 + `backend/app/evals/metrics.py:119-136` 的 `aggregate_rounds()` 用 mean/min/max 展示重复运行的波动。这正是观察"长链累积不确定性"的工具——步数越多，variance 越大。
- ✅[真实] **最大步数硬上限**：`backend/app/graph/builder.py:21` 的 `MAX_STEPS = 30`，从架构层防止步数爆炸——这是对"多一步多一份失败概率"的工程落地。
- ⚙️[补全] **per-step accuracy 的显式计算**：eval 框架当前只评终态（score_case），没有每一步的中间状态评分。数据有（step_count + 每步的 span events），但显式的 per-step accuracy 指标没做。

### 🟡 我能拿到的加分项

- ✅[真实] **从自己系统的数据说话**：虽然我没算过精确的 per-step accuracy，但可以用 step_count 和 top1_accuracy 估算——比如平均 5 步、端到端 75% 准确率，反推每步约 94% 左右。再跟工具审计表的实际 success rate 交叉验证。
- ⚙️[补全] **区分纯乘法 + 纠错模型**：我的 critic loop guard 是纠错机制——critic 判定 evidence 不够时触发补证。这意味着我的实际模型不是简单的"每步成功率连乘"，而是带恢复概率的。设计上可以把纠偏轮的恢复概率也建模进去。
- ✅[真实] **variance 放大效应的实证**：`--repeat N` 的多轮运行让我能看到高步数 case 的方差确实比低步数的大——这是复利失败公式的实证。

### 🔴 危险信号（主动规避）

- ❌ 别说"每步 99% 就万事大吉"——10 步后只剩 90%，而我的系统平均 5-7 步，每步成功率在 93-96% 区间。
- ❌ 别说 evals 已经按 per-step 评分了——我坦诚现在是终态评分，per-step 评分是改进方向。
- ❌ 别说"多跑几次平均就准了"——我需要反复强调用分布（mean + min/max）而非单点。

### 完整应答（口语稿）

> 复利失败公式是 Agent 可靠性设计的核心逻辑——每一步成功率连乘，步数越多端到端成功率越惨。我的系统能支撑这个分析：step_count 跟踪每个 run 的步数，score_case 计算最终 hit_top1，--repeat N 多轮运行看分布。虽然我目前没有显式计算 per-step accuracy，但我可以用实际数据反推——比如平均 5 步、端到端 top1 约 75%，大致每步 94% 左右。再跟工具审计表的实际 success rate 交叉验证。
>
> 更关键的是我通过 --repeat N 多轮运行看 variance——高步数 case 的方差明显比低步数的大，这正是"串行乘积累"在统计数据上的表现。我还在 builder 层设了 MAX_STEPS=30 硬上限——从架构上防止步数爆炸，因为多一步就多一份失败概率。
>
> 要补充的是，这个纯乘法模型在我的系统里不完全适用——因为我有一个 critic loop guard，当 critic 判定证据不够时触发补证。这意味着我的模型中包含了纠错回路的恢复概率。实际公式更接近带恢复的马尔可夫链，每步有成功概率 p，也有失败后被纠正恢复的概率 r。这个更精准的建模是延伸方向，但基础的数据支撑是真实的。

---

## Q4　Agent 死循环 / 行为异常的实时检测与归因定位

> **主接项目：SRE Agent 的 critic loop guard + terminal_reason 结构化终止 + 多级错误码。** 循环检测和归因定位都是真实实现，自信说。

### 🟢 我命中的参考答案要点

- ✅[真实] **循环检测（loop guard）**：`backend/app/graph/nodes/__init__.py:1117-1148` 的 `critic_node()` 内置 loop guard——loop_count ≥ max_loop_count 时设置 terminal_reason `{"code": "EVIDENCE_LOOP_EXHAUSTED", "stage": "critic", ...}`。不是只设最大步数，而是专门检测 critic→补证→重规划 循环。
- ✅[真实] **结构化终止原因**：`backend/app/graph/state.py:71` 的 terminal_reason 是结构化 Dict——含 code、stage、message、failed_tools。code 枚举覆盖：`EVIDENCE_LOOP_EXHAUSTED`（循环耗尽）、`CANNOT_GENERATE_TRUSTED_ACTIONS`（无法生成可信动作）、`AUTOMATION_CAPABILITY_UNAVAILABLE`（自动化能力不可用）、`RISK_BLOCKED`（风险阻断）。每类停止不仅知道"停了"，还知道"为什么、在哪停的、缺什么"。
- ✅[真实] **归因定位——区分模型错误 / 工具错误 / 架构错误**：`remediation_node`（line 1225）产出 `CANNOT_GENERATE_TRUSTED_ACTIONS` → 决策侧（模型生成不了可靠动作）。`risk_gate_node`（line 1322）产出 `AUTOMATION_CAPABILITY_UNAVAILABLE` → 架构侧（适配器没接）。`verify_decision` 有 SUCCESS / RETRYABLE_FAILURE / FATAL_FAILURE 三态。action execution_status 有 PRECONDITION_FAILED / FAILED / ERROR。
- ✅[真实] **loop_count 追踪**：`backend/app/graph/state.py:54-55` 的 loop_count 和 max_loop_count 跟踪循环次数。
- ⚙️[补全] **Action 重复率检测**：目前 loop guard 靠轮次计数，没有语义去重——识别"这个 action 已经被反复提议了但每次参数微调"。设计上我会加语义相似度检测。
- ⚙️[补全] **terminal_reason 码表集中管理**：目前几种 code 分散在三个节点文件里，没有集中定义。设计上升级时我会收敛到统一的枚举。

### 🟡 我能拿到的加分项

- ✅[真实] **用 terminal_reason 做线上归类驱动优化**：每种 code 出现的频率是可以统计的，如果 `EVIDENCE_LOOP_EXHAUSTED` 占比高 → 优化 critic prompt 和证据质量。这个统计在实际运行中是能做的。
- ⚙️[补全] **语义相似度的循环检测**：不只是轮次相同，而是识别"参数微调但本质重复"的动作。设计方向。
- ✅[真实] **fail-fast on fatal**：`AUTOMATION_CAPABILITY_UNAVAILABLE` 和 `RISK_BLOCKED` 是 fatal 级——一旦触发立即终止，不让一条链带着脏数据走下去。

### 🔴 危险信号（主动规避）

- ❌ 别说"设个最大步数就完事"——我有 loop guard 专门检测 critic 循环，terminal_reason 结构化记录。
- ❌ 别说"无法区分模型错还是工具挂"——我的 error code 分 decision-side/arch-side，verify 三态，action 三态。
- ❌ 别说"停了就停了"——terminal_reason 产出结构化终止原因供路由和可观测用。

### 完整应答（口语稿）

> Agent 死循环是生产级的高发问题，我的防治分两层。第一层是检测——不是简单设个最大步数，而是在 critic 节点内置了一个 loop guard。critic 每轮评估证据质量，不通过就触发补证或重规划。这个循环默认最多跑一次首轮取证加一次纠偏，再循环就强制终止，写一条结构化的 terminal_reason：code = EVIDENCE_LOOP_EXHAUSTED，stage = critic，把当前的质量分、矛盾信号、卡在哪一步全记进去。
>
> 第二层是归因——出了事得知道是谁的锅。我的 terminal_reason 的 code 覆盖三个维度。CANNOT_GENERATE_TRUSTED_ACTIONS 是模型侧——remediation 节点分析完发现没法生成可信动作；AUTOMATION_CAPABILITY_UNAVAILABLE 是架构侧——risk_gate 执行前调 describe_capability 发现 real 适配器没接，直接拒了；RISK_BLOCKED 是安全侧——风险等级太高不允许自愈。verify 有三态，action 有三态，每条都能定位到是决策侧、环境侧还是架构侧的问题。
>
> 要坦诚的短板我也有：我的 loop guard 目前靠轮次计数，没有语义去重——没做"这个 action 已经被反复提议了但参数微调"这种检测。terminal_reason 的码表目前分散在几个节点代码里，没有集中管理。这些都是下一版收敛的方向。

---

# 二、评测体系设计

## Q5　三层评估框架：L1 任务级 / L2 过程级 / L3 组件级

> **主接项目：SRE Agent Phase 8 eval 框架的 3 个 executor + scorer + metrics。** 三层都有对应，尤其是 L2 的 Direct vs Runner 双模式是我的独门设计。

### 🟢 我命中的参考答案要点

- ✅[真实] **L1 任务级评估（结果对不对）**：`backend/app/evals/scorer.py:37-76` 的 `score_case()` 做端到端评估——hit_top1 / hit_top3 / risk_match / status_match。`backend/app/evals/metrics.py:83-104` 的 `compute_metrics()` 输出 top1/top3 accuracy、macro_f1、confusion_matrix。主指标是封闭枚举等值比较——`actual.incident_type == expected.incident_type`，完全确定可复现。
- ✅[真实] **L2 过程级评估（推理对不对）**：`backend/app/evals/executors.py` 定义了 **两种执行器**——`DirectGraphExecutor`（纯逻辑图，跳过 DB/EventBus/副作用）和 `RunnerGraphExecutor`（全链路，带 DB 写入/checkpoint/事件）。`backend/app/evals/replay_runner.py:32` 的 `--mode compare` 双跑两路 diff 终态——专门暴露 checkpoint 序列化丢字段、checkpoint resume 后语义变化这类隐藏问题。
- ✅[真实] **L3 组件级评估（能力边界在哪）**：`backend/app/evals/metrics.py:26-57` 的 per-class precision/recall/f1/support 找出系统在哪类故障上不行。`backend/app/tools/gateway.py:979-1063` 的逐工具 success/fail + latency 找出哪个工具是短板。
- ⚙️[补全] **L2 的中间状态评分**：Direct vs Runner 对比目前是终态 diff，不是逐节点的中间状态对比。planner 质量、evidence collection 完整性、critic 决策准确性这些中间状态的独立评分没做。

### 🟡 我能拿到的加分项

- ✅[真实] **两层指标永不合并成一个综合数字**：Graph Quality 和 Runtime Fidelity 的指标保持独立——诊断准不代表 checkpoint 序列化没丢字段，runtime 正常不代表 LLM 判断对。这是刻意设计。
- ✅[真实] **Direct vs Runner compare 是 A/B 思想在离线落地的独创设计**：不是常见的"一个 executor 跑完"，而是用两种执行路径互相校验——用确定性高的路径（direct graph）来检测确定性低的路径（runner with DB/checkpoint/resume）。
- ⚙️[补全] **L2 的 planner 质量、critic 决策准确性等中间状态评分**：设计方向。

### 🔴 危险信号（主动规避）

- ❌ 别说"只评最终结果就够了"——我强调 Agent 的过程质量不可忽视，Direct vs Runner 双跑就是为了曝光过程问题。
- ❌ 别说"一个综合分数就行"——我的两层指标保持独立，不合并成一个模糊数字。
- ❌ 别说 L2 过程评估我已经全套做了——我坦诚中间状态评分是补全。

### 完整应答（口语稿）

> Agent 评测不能只看最终结果——这是我这个 eval 框架的设计前提。我最得意的一个设计是 L2 过程级评估——我做了两种执行器。DirectGraphExecutor 只跑纯逻辑图，不写 DB、不发事件、不做 checkpoint。RunnerGraphExecutor 跑完整链路，带持久化和事件流。然后用 compare 模式双跑同一个 case diff 终态，这一下子就会暴露一些极其隐蔽的问题——比如 checkpoint 序列化时丢了一个字段导致 resume 后语义变了，approval resume 后状态变化不一致。这些在单 runner 评测中完全看不出来。
>
> L1 任务级我用封闭枚举等值比较——incident_type 做成 11 种枚举值而不是自由文本，主指标就是 actual == expected，完全确定可复现。L3 组件级我用 per-class 指标找出系统在哪种故障类型上弱，用工具审计表看哪个工具是瓶颈。
>
> 要坦诚的是，L2 的中间状态评分我目前没做——planner 的取证计划是否合理、critic 的决策准确性，这些节点的独立评分是延伸。但我有一个硬原则：两层指标（Graph Quality 和 Runtime Fidelity）的分数永不合并成一个综合数字。诊断准不准和 checkpoint 序列化对不对是两个正交问题，一个数字掩盖不了。

---

## Q6　LLM-as-Judge 的可靠性、偏见校准与多 Judge 投票

> **主接项目：SRE Agent Phase 8 eval 的封闭枚举等值比较（刻意不做 LLM-as-Judge）。** 这是我的一个"反常识"设计决策，要讲透为什么不做才是更优选择。

### 🟢 我命中的参考答案要点

- ✅[真实] **主指标是确定性规则评分，不用 LLM Judge**：`backend/app/evals/scorer.py:37-76` 的 `score_case()` 用 `actual.incident_type == expected.incident_type` 做纯等值比较。Hypothesis 文本字段目前只采集不评分（report.py:99 "首期只采集不评分"）。
- ✅[真实] **刻意声明"非 CI 指标"**：`backend/app/evals/report.py:10-13` 的 `_DISCLAIMER` 明确写着"本报告由真实 LLM 生成，结果存在波动，属非 CI 指标"。report meta 中 `ci_metric: False`。这不是做不到——是我刻意选择不做。理由很硬：LLM-as-Judge 引入第二个 LLM 变量，指标变化时无法归因是模型变了还是 Judge 飘了。
- ✅[真实] **如何消解"需要 Judge"的场景**：原本 diagnose 输出是自由文本 hypothesis，没法稳定比较。我让 prompt 直接输出封闭枚举值（IncidentType 的 11 个值之一），`case_loader.py:79-82` 校验必须在枚举中。把开放性判断收窄为确定性比较，从源头消除了对 Judge 的需求。
- ⚙️[补全] **LLM Judge 的接口预留**：当确实需要评文字质量时（如 RAG 平台的对话流畅度），接口和架构已预留。但需要加上一致性校验和人工抽检。

### 🟡 我能拿到的加分项

- ✅[真实] **"刻意不做"是比"做了"更强的工程判断**：面试中最加分的就是这种——不是不会做，而是判断当前阶段不该做。LLM-as-Judge 的变量会污染评测信号，我在封闭枚举可覆盖的场景下选择消除变量而非引入新变量。
- ⚙️[补全] **多 Judge + majority voting + 解耦**：真需要 Judge 时，我会用多 Judge 投票减少单一偏见，Judge 模型与被评模型必须解耦（不能用同一供应商），加上一致性校验和人工抽检。
- ✅[真实] **Judge 偏见的论证**：用 GPT-4 评 GPT-4 有自恋偏见，这是我的核心论据——就像自己给自己打分，天然偏高。

### 🔴 危险信号（主动规避）

- ❌ 别说"用更强的 LLM 评就够了"——Judge 引入第二个非确定性变量，归因混了。
- ❌ 别说"我们实现了 LLM-as-Judge"——我是刻意不做。
- ❌ 别说"LLM Judge 完全不可用"——只是在封闭域不需要，开放域需要时加上校准再上。

### 完整应答（口语稿）

> 这道题我有个可能反直觉的答案——我的 eval 框架刻意不做 LLM-as-Judge。不是技术上做不了，而是工程判断上不该做。

> 理由很直接：LLM-as-Judge 引入第二个非确定性变量。当你发现指标从 80% 掉到 65%，你无法归因——是我的 Agent 模型真的变差了，还是 Judge 模型自己飘了？评测的目标是精准捕获 Agent 行为变化的信号，结果你接入一个自身也波动的裁判，信噪比反而下降。而且在封闭域里根本不需要 Judge——我让 diagnose 直接输出封闭枚举值而不是自由文本 hypothesis，主指标就变成了 `actual == expected` 纯等值比较，完全确定、可复现。

> 我用了一个设计手段来消解"需要 Judge"的需求——原来 diagnose 输出的 hypothesis 是自由文本没法比，我改成了 11 种 IncidentType 枚举值，解析时校验必须在枚举中。这一改就把开放性判断收窄成了确定性比较。而且我的报告顶层明确标注 "ci_metric: false"——诚实告诉所有人这是非 CI 指标，不要当门禁用。等到确实需要评文本质量的时候我会上 Judge，但会带多 Judge 投票解耦 + 一致性校验 + 人工抽检三层校准。不是不用，是知道什么时候用、怎么用得稳。

---

## Q7　评测集构建策略：正常流程 60% + 边界 30% + 对抗 10%

> **主接项目：SRE Agent 的 11 个 case + unknown/other 分离 + minimal fixture。** 数据集真实存在，比例不同但设计意图一致。

### 🟢 我命中的参考答案要点

- ✅[真实] **11 个 case 覆盖 11 种 IncidentType**：`backend/app/evals/datasets/` 下 case_01 到 case_11，9 种标准故障类型（deployment_regression / configuration_error / resource_exhaustion / dependency_failure / database_failure / network_failure / traffic_anomaly / security_incident / service_degradation）+ case_10 是 `unknown`（证据不足保守回避）+ case_11 是 `other`（超出分类体系的逃逸标签）。
- ✅[真实] **unknown 和 other 是分开的**：这个区分很关键——`unknown` 是"证据不足我保守说不知道"，`other` 是"这不在我的知识体系里我诚实报告超纲"。两者混在一起会造成虚假准确率——模型说"不知道"算对还是错？分开后各有各的评判标准。
- ✅[真实] **边界/对抗 case**：case_10 是对抗 case——给极其稀疏的告警数据（空 logs、空 metrics fixture），考验 Agent 在信号不足时会不会强行武断。case_11 是逃逸测试——输入超出分类体系的故障，看模型是否诚实报告而非瞎编。
- ✅[真实] **Schema 校验**：`backend/app/evals/case_loader.py:21-46` 的 `load_cases()` 对每个 case 做 schema 校验——REQUIRED_TOP fields（case_id / description / ticket）和 REQUIRED_TICKET fields。
- ⚙️[补全] **60/30/10 比例**：实际是 9 正常（82%）+ 1 boundary（9%）+ 1 adversarial（9%），不是严格的 60/30/10。设计原则是按这个思路来的——case_10 和 case_11 确实代表了 boundary/adversarial 类型。

### 🟡 我能拿到的加分项

- ✅[真实] **unknown / other 分离设计**：这是我能讲透的一个细节——为什么在分类体系中需要两个"不确定"标签，它们的区别是什么，为什么要分开评测。
- ✅[真实] **Minimal Fixture 策略**：每个 case 只提供"信号承载"工具的 fixture（比如 query_logs 给 500 条错误），未提供的只读工具返回受控空——不是 mock 随机数据。受控空 100% 确定，证据稀疏不等于随机数据。
- ✅[真实] **低成本的扩展机制**：新增一种故障类型只需两步——枚举加一行、补至少一个 case JSON。scorer/metrics 都按枚举动态聚合不硬编码。扩展成本极低。
- ⚙️[补全] **holdout 分离和难度分层**：目前 11 个 case 是全部评测集，没有公开/私有分离，没有 easy/medium/hard 标签。

### 🔴 危险信号（主动规避）

- ❌ 别说"数据集是严格 60/30/10"——坦诚说实际是 82/9/9，但设计意图一致。
- ❌ 别说评测集随机抽的——我的 11 个 case 是精心设计覆盖所有故障类型 + 边界 + 对抗。
- ❌ 别说"11 个 case 就够了"——我会说数据集需要持续扩充，目前是 Phase 8 的初始状态。

### 完整应答（口语稿）

> 评测集构建我有三个原则：覆盖全枚举、信号稀疏测保守、低扩展成本。我的 11 个 case 覆盖 11 种 IncidentType 封闭枚举，9 个标准故障，加两个特殊的——case_10 是 unknown，给极少证据看模型敢不敢说不知道；case_11 是 other，输入不在分类体系里看模型诚不诚实。unknown 和 other 分开是一个刻意设计：前者是"证据不足保守回避"，后者是"诚实报告超纲"，混在一起会造成虚假准确率。

> 边界和对抗我也有——case_10 给空 logs 和空 metrics fixture，信号极稀疏，检验 Agent 在信息不足时会不会强行武断输出一个故障类型。我还用 minimal fixture 策略——每个 case 只给信号承载的工具有数据，未提供的工具返回受控空而不是随机 mock 数据，这样评测结果 100% 确定可复现。

> 要坦诚的是，我的比例不是严格的 60/30/10，11 个 case 是 9 正常 + 1 boundary + 1 adversarial。数据集规模也偏小——这是 Phase 8 初始状态。但我的扩展成本极低：新增一个故障类型只需枚举加一行、补一个 case JSON，scorer 和 metrics 都按枚举动态聚合不硬编码。holdout 分离和 per-difficulty 标签是下一个迭代的方向。

---

## Q8　离线评测 vs A/B 测试 vs Shadow Mode 的架构选型

> **主接项目：SRE Agent 的离线 eval（direct/runner/compare 三种 mode）+ prompt diff。** 离线评测真实全覆盖；A/B 和 Shadow 偏补全。

### 🟢 我命中的参考答案要点

- ✅[真实] **离线评测完整**：`backend/app/evals/replay_runner.py` 支持三种 mode——`direct`（纯图，快速迭代）、`runner`（全链路，含 DB/事件/checkpoint）、`compare`（两路双跑 diff）。`--repeat N` 取分布、`--prompt-version old,new` 做 prompt diff。
- ✅[真实] **离线评测的关键安全设计——副作用永不重放**：`backend/app/evals/fixture_context.py` 用 ContextVar 在 `call_tool` 入口短路注入固定证据，在 mock/real 选择之前。decision replay（重放模型决策）允许，effect replay（重放副作用）绝对禁止。
- ⚙️[补全] **Shadow Mode**：没有在生产环境并行跑影子 Agent 的能力。设计上我会做——在真实流量上并行跑新版本，但副作用操作（创建资源/发送通知）在影子侧全部屏蔽。离线评测的 compare 模式是 Shadow 的思想在离线侧的落地。
- ⚙️[补全] **A/B 测试**：没有线上 A/B 切流系统。设计上讲分阶段策略——原型阶段离线评测，迭代阶段离线 + Shadow，上线后 A/B 测试兜底。
- ⚙️[补全] **统计显著性**：目前 compare 和 prompt diff 是直观对比，没有做 t-test 或 Wilcoxon 统计检验。

### 🟡 我能拿到的加分项

- ✅[真实] **compare 模式是 A/B 思想在离线的落地**：direct vs runner 双跑 diff，快速暴露 checkpoint serde 问题、approval resume 语义变化。这不是常见的"跑一次看结果"。
- ✅[真实] **副作用绝对不重放**：我的 fixture 短路在工具调用入口，在 mock/real 选择之前。离线评测再跑多少次都不会真打 MySQL/K8s。这是 Shadow Mode 的最核心约束——影子不得执行副作用。
- ⚙️[补全] **分阶段策略**：原型→离线评测，迭代→离线+Shadow，上线→A/B+监控。这个演进路径是我会讲的。

### 🔴 危险信号（主动规避）

- ❌ 别说"上线前跑一遍离线就行"——忽略分布偏差和用户行为信号。
- ❌ 别说 Shadow 已经实现了——坦诚说是设计方向，离线 compare 是前置落地。
- ❌ 别说"三种模式只选一个"——应该是组合使用，按阶段演进。

### 完整应答（口语稿）

> 离线评测、A/B 测试、Shadow Mode 不是我三选一，是按阶段演进。我目前最成熟的是离线评测，有三种 mode。direct 模式只跑纯逻辑图不碰 DB 和 checkpoint——快速验证 prompt 和模型行为，一分钟出结果。runner 模式跑全链路带持久化——验证 checkpoint、事件流、审计在真实链路下对不对。compare 模式双跑 direct 和 runner 然后 diff 终态——用来抓 checkpoint 序列化丢字段、approval resume 后语义变化这类巨隐蔽的问题。

> 离线评测有一个我特别在意的安全约束——副作用绝对不允许重放。我的 fixture 机制在工具网关入口短路注入，在 mock 和 real 适配器选择之前就拦截了。评测跑 100 遍也不会真打 MySQL 或 K8s。这个约束在 Shadow Mode 里同样适用——影子 Agent 跑在真实流量上但必须屏蔽所有写操作。

> Shadow Mode 和 A/B 测试目前是我的延伸方向。设计上我的分阶段策略是：原型期离线评测快速迭代，迭代期离线加 Shadow Mode 对高风险任务做影子对比，上线后 A/B 切流加线上监控兜底。compare 模式的 A/B 思想已经在离线落地了，上生产是下一步。

---

## Q9　AgentBench / GAIA 等 benchmark 的适用性边界与业务定制

> **主接项目：SRE Agent 的封闭枚举评测设计 + case_loader schema。** 完全自建领域评测集，没用外部 benchmark。

### 🟢 我命中的参考答案要点

- ✅[真实] **完全自建的领域特定评测集**：`backend/app/evals/datasets/` 下的 11 个 case 全部是 SRE 故障处置领域的自定义 case——每个 case 含 ticket、tool_fixtures、expected（incident_type + risk_decision + final_status）。
- ✅[真实] **领域特定枚举是自建的核心原因**：我的 IncidentType 是 11 种 SRE 特有故障类型（resource_exhaustion / k8s_pod_crash / database_failure 等），AgentBench 的 OS 操作、网页浏览跟我的 action space 完全无关。`backend/app/evals/case_loader.py:79-82` 校验 incident_type 必须在 IncidentType 枚举中。
- ✅[真实] **benchmark 方法论借鉴**：虽然我没用外部 benchmark 的数据，但方法论上借鉴了——多 case 覆盖、repeat N 多轮取分布、per-class 指标——这些是 benchmark 的通用实践。
- ⚙️[补全] **毫无外部 benchmark 引用**：代码中没有任何 BLEU、ROUGE、HELM、GAIA、AgentBench 等外部指标的引用。这是刻意的领域聚焦，不是遗漏。

### 🟡 我能拿到的加分项

- ✅[真实] **说出 benchmark 的适用性边界**：学术 benchmark 有两个核心局限——任务分布与业务 action space 不匹配、评测方式太简化。让我诊断一个数据库故障，AgentBench 不会给我 MySQL 慢查询和连接池指标的 fixture。
- ✅[真实] **封闭枚举是工程化的关键**：我把开放性的故障诊断问题收窄成了 11 种封闭枚举——这让主指标可以做纯等值比较，消除评测噪声。这比直接用 benchmark 的多选题更贴合我的业务闭环。
- ⚙️[补全] **两种用法**：选型时用 benchmark 摸底模型基础能力，上线后用业务评测集做回归。这个分层我目前没用上是因为项目还在选型期后、上线期前。

### 🔴 危险信号（主动规避）

- ❌ 别说"接了 AgentBench 所以评测完备"——我坦诚是用自建评测集，因为它和我的 action space 对齐。
- ❌ 别说 benchmark 没用——选型时它们有参考价值，我坦诚是用自建集覆盖业务闭环。
- ❌ 别说 11 个 case 覆盖了全局——数据集需要持续扩充。

### 完整应答（口语稿）

> 学术 benchmark 和业务评测集的关系我分得很清。我用的是完全自建的 SRE 领域评测集——11 个 case 覆盖 11 种 IncidentType 封闭枚举。为什么不用 AgentBench？因为它的任务空间跟我不匹配——AgentBench 考的是 OS 操作、网页浏览、文件管理，我的 Agent 考的是看 MySQL 慢查询、K8s Pod 状态、云监控指标来诊断故障。benchmark 不给我数据库连接池指标的 fixture，也不考我的 risk_gate 审批策略。

> 但我在方法论上借鉴了 benchmark——多 case 覆盖、repeat N 多轮取分布、per-class 指标、confusion_matrix——这些通用实践我都落地了。我最大的工程化决策是把开放性诊断收窄成了 11 种封闭枚举，让主指标可以做纯等值比较而不是靠 LLM Judge 打分。这比 benchmark 的更贴合我的业务闭环。

> 选型时学术 benchmark 有其价值——当做模型基础能力的摸底。但上线后的回归评测必须用业务评测集，因为只有它和你的 action space、observation space 对齐。我的 Phase 8 eval 框架就是朝这个方向走的——数据集的 schema 校验、case_loader 的动态加载、低成本的 case 扩展，都是为了让业务评测集能持续演进。

---

# 三、生产级可观测性架构

## Q10　全链路 Trace 的数据采集、存储与关联设计

> **主接项目：SRE Agent 的 AgentTracer + 三层 span 层级 + LangSmith/Langfuse + ContextVar 传播。** 全链路 trace 基本完整，自信说。

### 🟢 我命中的参考答案要点

- ✅[真实] **三层 Span 层级**：`backend/app/tracing.py:25-155` 的 `AgentTracer` 设计了三层：`graph.run`（顶层 span，在 `graph_runner.py:212-228` 启动）→ `node.{name}`（每个图节点一个 span）→ `tool.{name}` / `llm.{provider}` / `specialist.{agent_id}`（最底层，具体调用）。父子关系通过 parent_id 串联。
- ✅[真实] **Span 数据结构**：`start_span()`（line 44-70）返回 span_id / name / run_id / parent_id / start_time / start_timestamp。`end_span()`（line 72-90）记 end_time / duration_ms / status / error。`add_event()`（line 92-109）在 span 内嵌 events 数组。
- ✅[真实] **Trace ID 传播机制**：`backend/app/tracing.py:21-22` 使用 ContextVar `run_id_var` / `step_id_var`——Python 3.7+ 的 `asyncio.create_task` 自动 copy 当前 context，跨 async 边界自动继承 trace_id。`graph_runner.py:213` 用 `tracer.set_run_context(run_id)` 设置上下文。
- ✅[真实] **LangSmith / Langfuse 集成**：`backend/app/tracing_providers.py:94-109` 的 `create_trace_provider()` 支持三种模式——local / langfuse / langsmith。LangfuseTraceProvider（line 112-215）用 observation 父子关系，LangSmithTraceProvider（line 218-347）用 RunTree 父子关系。`ProviderSpanResult`（line 30-35）返回 external_trace_id / external_trace_url。
- ✅[真实] **External Trace URL**：`backend/app/tracing.py:116-131` 的 `get_trace_metadata()` 返回 provider + external_trace_id + external_trace_url——内部系统一键跳转到 LangSmith 完整 trace 页面。

### 🟡 我能拿到的加分项

- ✅[真实] **Span 内嵌 events 而非平铺**：每个 tool 调用 / LLM 请求的详细信息作为 event 挂到对应 span 下，形成 hierarchy——不是所有日志平铺在一个列表里。
- ⚙️[补全] **存储策略分层**：目前全量 in-memory list + DB 落库。设计上我会做 Redis（热）+ ClickHouse（温）+ S3（冷）三层存储。
- ✅[真实] **不依赖外部服务的 local provider**：当 LangSmith/Langfuse 不可用时可切到 local provider，系统不依赖外部可观测服务也能跑。

### 🔴 危险信号（主动规避）

- ❌ 别说"只记了最终结果"——我有三层 span + events + LangSmith/Langfuse 全链路。
- ❌ 别说 trace 异步场景断链——我用 ContextVar 保证 asyncio 并发场景下 trace_id 自动继承。
- ❌ 别说全量存储没问题——我坦诚目前的 in-memory 全量在生产体量下需要加采样和分层存储。

### 完整应答（口语稿）

> 全链路 trace 我的设计是三层 span 树。最顶层是 graph.run span——一个 run 一个。第二层是 node.{name} span——每个图节点一个，排查"是哪个节点慢了/错了"。第三层是 tool.{name}、llm.{provider}、specialist.{agent_id}——具体到每次工具调用、每次 LLM 推理、每个子 Agent 的执行。每个 span 记录 span_id、parent_id、start_time、end_time、duration_ms、status、error，和 events 数组。

> trace 传播我用 ContextVar——run_id 和 step_id 放在 contextvars 里，Python 3.7 以上 asyncio.create_task 会自动 copy 当前 context。所以我的 evidence_fanout 里用 asyncio.gather 并发派发 5 个专家，每个专家的 span 自动继承同一个 run_id，parent 挂到 fanout 的 span 下——异步并发 trace 不会断链。

> 外部可观测平台我集成了 LangSmith 和 Langfuse——两种 provider 都已代码就绪，LangSmith 在真实控制台验证过了。最关键的是我也可以切到 local provider 完全不依赖外部服务。这个灵活性在生产环境很重要——外部可观测服务挂了不影响系统运行。

---

## Q11　采样策略博弈：错误全量 + 慢请求全量 + 正常 10% 的工程权衡

> **主接项目：坦诚说——SRE Agent 目前全量存储，采样策略是补全。** 这条坦率说短板反而加分。

### 🟢 我命中的参考答案要点

- ✅[真实] **当前全量存储**：`backend/app/tracing.py:29` 的 `self.spans: list[Dict[str, Any]] = []`——所有 span 全量进内存 list，外加 DB event 表全量落库。`backend/app/tracing_providers.py:17-28` 的 `TraceProviderConfig` 没有 sampling_rate 字段。
- ⚙️[补全] **采样策略设计**：工程上我会用"错误全量 + 慢请求全量 + 正常采样 10%"三层采样。错误跑全量（必须能被排查）、慢请求全量（P99 以上全存）、正常请求采样保趋势。采样粒度用 session 级（同一 run 要么全采要么全不采，避免 trace 断裂）。
- ⚙️[补全] **动态调整**：线上故障时手动切换全量，平稳期降低采样率。对采样本身做监控——采样率偏离目标、对重要事件的漏抓率。

### 🟡 我能拿到的加分项

- ✅[真实] **"先全量再收敛"的务实路径**：系统初期数据量小（日均几十个 run）时全量不是问题。这是对的——初创系统先全量，积累数据后再引入采样并做历史数据裁剪。这个路径判断本身是架构师思考。
- ⚙️[补全] **运维采样 vs 评测采样的区分**：运维（SRE 看趋势）和评测（离线评测需要全量历史做训练/调参）的采样策略应独立。评测用全量（离线），运维用采样（实时）。
- ⚙️[补全] **存储分层 + 采样联动**：热数据全量短期、温数据采样中期、冷数据聚合长期。

### 🔴 危险信号（主动规避）

- ❌ 别说"全量存没毛病"——承认全量在生产体量下会成为瓶颈，采样是必要的。
- ❌ 别说采样是随机采样——我主张按重要性分层采样，不是随机采样。
- ❌ 别把采样和存储分层说成已经实现——坦诚说是设计方向。

### 完整应答（口语稿）

> 坦诚说，我的系统当前是全量存储——所有 span 进内存 list、所有事件落 DB。因为我的系统还处于日均几十个 run 的阶段，全量不是瓶颈。但架构师必须能看到未来：当系统跑起来后全量一定会成为瓶颈。

> 我的设计是三层采样——错误全量（失败了必须能排查，这是底线）、慢请求全量（P99 以上的全存，因为往往是性能问题的信号）、正常请求 10% 采样（保整体趋势和分布）。采样粒度做 session 级——同一个 run 要么全采要么全不采，避免 trace 断裂。

> 跟采样配套的存储也要分层。热数据短期 Redis 支持实时查询，温数据中期 ClickHouse 做 OLAP 分析和全文检索，冷数据长期 S3 做归档。而且我不搞一刀切——运维侧的实时告警用采样数据看趋势就够了，评测侧的离线分析需要全量历史做回归——两套采样策略独立。评测用全量离线数据，线上运维用采样数据。这个设计方向是明确的，落地是 Phase 10 的事。

---

## Q12　渲染后真实 prompt 的捕获与 prompt 拼装 bug 定位

> **主接项目：SRE Agent 的 PromptRegistry（checksum 版本管理）+ llm_request event（metadata）。** 版本管理和 checksum 真实，但渲染后全文捕获是补全。

### 🟢 我命中的参考答案要点

- ✅[真实] **Prompt 版本管理**：`backend/app/prompts/registry.py:1-113` 的 `PromptRegistry` 做 prompt 不可变版本管理——每个 prompt 有 prompt_id / version / checksum（SHA256 前 8 位）/ content。`backend/app/graph/state.py:77` 的 state 中记录 prompt_versions（当前节点使用的 prompt 版本）。
- ✅[真实] **llm_request event 记录 metadata**：`backend/app/llm_client.py:70-83` 的 llm_request event 记录 provider / model / has_system_prompt / prompt_length——不是完整渲染后文本，但可以定位到"用的是哪个 prompt 版本 + 多长"。
- ✅[真实] **Checksum 做 prompt diff**：`backend/app/evals/replay_runner.py:124` 的 prompt diff 记录 old_checksum 和 new_checksum，对比两版本的指标变化。
- ⚙️[补全] **渲染后完整 prompt 文本的存储**：当前 `llm_request` event 只记了 metadata（prompt_length），不记原文。prompt 全文体量大，采样存或分层存是设计方向——错误跑全存原文，正常跑存 metadata。

### 🟡 我能拿到的加分项

- ✅[真实] **prompt 版本不可变 + checksum**：每次修改生成新版本 + SHA256 校验和，旧版本可追溯可回滚。出问题时从 checksum 定位到版本，从版本找回原文。
- ✅[真实] **checksum 驱动的 prompt diff**：改 prompt 后跑 eval，对比 old_checksum vs new_checksum 的指标变化——improved / regressed / unchanged 三类。这是 prompt 工程的闭环。
- ⚙️[补全] **从 metadata 一键跳到原文**：设计上 llm_request event 的 metadata（版本 checksum）可以关联到 PromptRegistry 的完整 content——一键从 trace 跳到渲染后原文。这是补齐方向。

### 🔴 危险信号（主动规避）

- ❌ 别说"我记了渲染后的完整 prompt"——我坦诚当前只记了 metadata（版本 + 长度），全文捕获是延伸。
- ❌ 别说"prompt 就那么几行不会出 bug"——生产 Agent 的 prompt 是几十个模板动态拼接的，拼装 bug 非常常见。
- ❌ 别说 prompt 改了直接改代码——我有 PromptRegistry 做不可变版本管理。

### 完整应答（口语稿）

> Prompt 拼装 bug 是 Agent 系统最常见也最难查的问题——因为生产环境的 prompt 是几十个模板动态拼接的：系统提示 + 历史消息 + 工具结果 + 当前指令，拼接的顺序、截断的位置、变量的替换任何一个环节出问题，出来的 prompt 就跟预期不一样。只看模板名和参数，根本查不出来。

> 我目前有两层。第一层是 PromptRegistry——每次 prompt 修改生成不可变版本，带 prompt_id、version 和 SHA256 checksum。state 里记录当前 run 使用的 prompt_versions，出问题时从 checksum 定位到版本。第二层是 llm_request event——记录了 provider、model、prompt_length 等 metadata。

> 要坦诚的短板是：我目前记的是 metadata 而不是渲染后的完整 prompt 原文。这是因为 prompt 全文体量大，全量存成本高。设计上我会做分层——错误跑全存原文用于排查，正常跑存 metadata 加 checksum；从 checksum 可以关联回 PromptRegistry 的完整模板，从模板 + 参数可以重建当时的上下文。同时 prompt diff 评测我用 checksum 对比改前后的指标变化——improved / regressed / unchanged——这是 prompt 工程的数据驱动闭环。

---

## Q13　告警体系：指标阈值 vs ML 异常检测的分层策略

> **主接项目：SRE Agent 的告警接入（alert_event API）+ escalation_risk 预测。** 告警接入真实，完整告警体系是补全。

### 🟢 我命中的参考答案要点

- ✅[真实] **告警接入**：`backend/app/api/incidents.py:47-60` 的 `AlertEventCreate` model + alert_event 输入模式，`backend/app/services/intake.py:58-82` 的 `from_alert_event()` 将告警事件转换为 IncidentTicket。支持 pagerduty / alertmanager / manual 三种 source。
- ✅[真实] **escalation_risk 预测**：`backend/app/analytics/risk_prediction.py:1-81` 的规则化风险预测，输出 LOW / MEDIUM / HIGH——虽然不是 ML 异常检测，但是一个确定性的风险信号。
- ✅[真实] **生产环境 fail-fast 校验**：`backend/app/core/config.py:117-158` 的 `validate_for_production()` 做启动时安全校验——CORS=* 不允许、mock adapter 不允许、debug=True 不允许、LLM key 必填。
- ⚙️[补全] **完整告警体系缺失**：没有阈值配置引擎（threshold config）、没有告警升级链（escalation chain）、没有通知渠道（PagerDuty/Slack/钉钉集成）、没有告警聚合去重。当前只有告警接入 + rule-based 风险预测。

### 🟡 我能拿到的加分项

- ✅[真实] **区分"告警接入"和"告警体系"**：我能讲清楚——接入是"能收到告警事件并创建工单"，体系是"完整的阈值配置 + 升级链 + 通知渠道 + 聚合去重"。当前实现的是接入，体系是设计方向。这个区分本身就是架构师思维。
- ⚙️[补全] **告警规则可配置化**：设计上告警阈值应该做成配置文件或运营界面可调——不是写在代码里。HS 的 spec YAML 配置化模式可以复用。
- ⚙️[补全] **告警与 terminal_reason 打通**：告警响了不应该只显示"成功率跌破阈值"，还应该关联到 structured terminal_reason——从告警一键下钻到是什么 code 导致的。

### 🔴 危险信号（主动规避）

- ❌ 别说"我们有完整的告警体系"——坦诚说是接入 + 风险预测，体系是补全。
- ❌ 别说"Grafana 设几个阈值就够了"——这只是固定阈值的 P0/P1 部分，缺少异常检测和升级链。
- ❌ 别把告警和监控混为一谈——告警是监控 + 响应链的闭环。

### 完整应答（口语稿）

> 告警这块我做了一个重要区分——告警接入和告警体系是两回事，我当前落地的是接入侧。我的系统有 alert_event API，能从 PagerDuty、AlertManager 和人工三个渠道接收告警事件，intake 模块把它转成结构化的 IncidentTicket 进图处理。还有一个 rule-based 的 escalation_risk 预判，输出 LOW/MEDIUM/HIGH，给 risk_gate 和审批流程提供信号。

> 但完整的告警体系还包括阈值配置引擎、告警升级链、通知渠道、聚合去重——这些我坦诚说是设计方向。设计上我会分两层：固定阈值（P0/P1）保底线——工具成功率跌破 95% 告警、任务失败率突增告警——这些是确定性规则；ML 异常检测（P2）补充——延迟突变不是超阈值而是从 1s 暴增到 9s，这类"没超阈值但行为异常"的场景固定阈值抓不到。

> 告警升级也要有——P2 持续 N 个周期没恢复就升 P1，P1 升 P0。告警要跟 structured terminal_reason 打通——告警响了之后一键下钻到当时是什么 code 导致的。HS 项目的 YAML 配置化模式可以直接复用——阈值和升级策略做成可运营调整的配置而非硬编码。这个设计思路是清晰的，当下是接入真实、体系待落地。

---

## Q14　多 Agent 场景的 DAG 关联 Trace 与 A2A 通信追踪

> **主接项目：SRE Agent 的 ContextVar trace 传播 + specialist parallel spans + aggregator cross-agent contradiction。** 多 Agent trace 基本真实，跨进程/跨服务是补全。

### 🟢 我命中的参考答案要点

- ✅[真实] **Trace ID 跨 async task 自动传播**：`backend/app/tracing.py:21-22` 的 ContextVar `run_id_var` / `step_id_var` 在 Python 3.7+ 的 `asyncio.create_task` 中自动 copy——所有并发 task 共享同一 run_id。`backend/app/services/graph_runner.py:213,228` 在每个 node 启动时设置 run context 和 step context。
- ✅[真实] **多 Agent 的 span 父子关系**：编排者 `graph_runner` 的 graph.run span 是根，`node.evidence_fanout` span 是中间层，`specialist.{agent_id}` span 是叶子——`backend/app/graph/nodes/specialist_agent.py:163-165` 创建 specialist span 时 parent_id 从外部传入。
- ✅[真实] **asyncio.gather 并发下的 span 正确性**：`backend/app/graph/nodes/specialist_agent.py:241-244` 的 `asyncio.gather` 并发多个 specialist——每个 specialist 有独立 span，通过 ContextVar 自动继承同一 run_id。并发工具调用（`graph/nodes/__init__.py:504`）同理。
- ✅[真实] **跨 Agent 矛盾检测**：`backend/app/graph/nodes/aggregator.py:138` 的 `_detect_cross_agent_contradictions` 检测不同 specialist 的结论是否矛盾，产出 contradiction_signals。
- ⚙️[补全] **跨进程/跨服务 trace propagation**：当前 tracing 在单进程 in-memory——没有 OpenTelemetry 跨服务传播（trace context 通过 HTTP header 传递）。如果是多服务部署，trace 会断。

### 🟡 我能拿到的加分项

- ✅[真实] **树状结构 + 图结构双重覆盖**：我的编排是层级编排（planner → fanout → aggregate → critic），形成树状 span 树。aggregate 的 cross_agent_causal_chains 和 contradiction_signals 记录的是 Agent 之间的对等关系（如 logs agent 和 metrics agent 的结论交叉）。
- ⚙️[补全] **MCP 对 A2A trace 的增强**：MCP 协议标准化后，跨 Agent 调用的 trace context 可以在 MCP header 中透传——从"应用层自己维护"下沉为"协议层自动传播"。
- ✅[真实] **Langfuse 的 observation 父子关系**：`backend/app/tracing_providers.py:149-185` 的 Langfuse provider 用 parent_observation 自动构建父子树——多 Agent span 在 Langfuse 面板上可视化就是一棵完整的树。

### 🔴 危险信号（主动规避）

- ❌ 别说"多 Agent trace 靠时间戳串联"——异步并发下 timestamp 重合根本无法关联。我用 ContextVar 保证。
- ❌ 别说"跨服务 trace 已经支持"——我坦诚目前是单进程 in-memory，跨服务是补全。
- ❌ 别说"子 Agent 的 span 挂不到父 span 下"——ContextVar + parent_id 机制保证父子关系正确。

### 完整应答（口语稿）

> 多 Agent trace 最容易断链——编排者派发了 5 个子 Agent 并行跑，怎么保证每条子 Agent 的内部 trace 都能串回编排者的调用链？我用的是 ContextVar 传播。run_id 和 step_id 放在 contextvars 里，Python 3.7 以上 asyncio.create_task 会自动 copy 当前 context。所以我的 evidence_fanout 用 asyncio.gather 并发派发 5 个取证专家，每个专家的 span 自动继承同一个 run_id，parent_id 挂到 fanout 的 span 下。在 Langfuse 面板上就是一棵完整的树——根是 graph.run，中间是 node.evidence_fanout，叶子是 5 个 specialist.{agent_id}。

> 不光 trace 能串起来，Agent 之间的结论交叉我也记了。aggregate 节点做跨 Agent 矛盾检测——如果 logs agent 说"数据库连接池满了"但 metrics agent 说"数据库连接数正常"，aggregate 会产出一个 contradiction 信号，同时构建 cross_agent_causal_chains。这些信息记在 state 里跟着 trace 走——两个 Agent 吵架了我知道谁说了什么。

> 要坦诚的是，当前 trace 是单进程 in-memory 的——没有做 OpenTelemetry 跨服务传播。如果多服务部署，trace context 需要通过 HTTP header 或 MCP header 透传。这是生产级多 Agent 系统的必须项，是下一阶段的设计方向。

---

# 四、回归测试与持续质量保障

## Q15　模型升级 / Prompt 变更的回归评测与行为回退检测

> **主接项目：SRE Agent 的 prompt diff 框架 + repeat N + per-class 切片。** Prompt 回归完整，模型升级回归偏补全。

### 🟢 我命中的参考答案要点

- ✅[真实] **Prompt 版本回归框架完整**：`backend/app/evals/replay_runner.py:105-191` 的 `_run_prompt_diff()` 做完整 prompt diff——两个 PromptRecord 版本（old_checksum vs new_checksum）分别跑全量数据集（repeat N），对比 improved / regressed / unchanged 三类指标变化。输出 JSON + Markdown 报告。
- ✅[真实] **分布对比而非单点对比**：`--repeat N`（≥3）多轮运行取 mean/min/max/variance——对比的是分布。改一个 prompt 让 mean 从 72% → 80% 同时 variance 收窄，才是真实改善信号。
- ✅[真实] **分项退化抓取**：per-class precision/recall/f1 + confusion_matrix + macro_f1。总成功率不降但某类故障暴跌会被切片抓出来。
- ⚙️[补全] **模型切换专项回归**：当前 prompt diff 框架很完整，但"换模型后行为一致性"的专项评测没有做——框架可复用但未专项设计。设计上我会在 prompt diff 基础上加 `--model-version old,new` 做模型维度的 diff。
- ⚙️[补全] **统计检验**：当前是直观对比 improved/regressed/unchanged，没有 t-test 或 Wilcoxon。统计检验能区分"真实退化"和"随机波动"。

### 🟡 我能拿到的加分项

- ✅[真实] **Checksum 驱动的 prompt 工程闭环**：PromptRegistry 做不可变版本管理，改了就出新 checksum → eval 跑 diff → 出报告 → 决策是否上线。整个过程是可追溯可回滚的闭环。
- ⚙️[补全] **行为回退的多维度**：不仅监控成功率回退，成本（步数/token）和延迟的回退也应该监控。换一个更谨慎的模型，成功率不降但步数翻倍成本翻倍——也是退化。
- ✅[真实] **eval 框架就是数据驱动降本的验证工具**：任何降本改动（换小模型、裁上下文），都可以用 eval + repeat N + per-class 验证对成功率的实际影响。

### 🔴 危险信号（主动规避）

- ❌ 别说"供应商说升级了我们就直接切"——我主张必须跑回归评测对比基线。
- ❌ 别说"总成功率没问题就 OK"——分项退化会被切片抓出来。
- ❌ 别说"单次跑过就行"——我强调 repeat N 取分布对比。

### 完整应答（口语稿）

> 模型升级和 prompt 变更的静默退化是我重点防控的。我的 eval 框架有完整的 prompt diff——改 prompt 后，用 old 和 new 两个版本分别跑全量数据集，repeat N 多轮，然后对比 improved / regressed / unchanged 三类 case 的详细指标。

> 关键有两个：第一是看分布不是看单点——单次 72% → 68% 可能是随机波动，但三轮均值从 72% 跌到 58% 加上 variance 收窄，那是真实退化信号。第二是看分项不是看总平均——总成功率不降但 database_failure 从 90% 跌到 60%，per-class f1 会抓出来。

> prompt 的版本管理我用 checksum——每次修改生成不可变版本加 SHA256，旧版本可追溯可回滚。这个配合 eval 框架就是一个数据驱动的 prompt 工程闭环。模型升级的专项回归我还没做，但框架可以复用。我会加的一个东西是统计检验——t-test 或 Wilcoxon 区分真实退化和随机波动。还有一个要多维度监控——不只是成功率，步数和 token 成本如果翻倍也是行为退化。

---

## Q16　评测防过拟合：holdout 分离、持续刷新、防刷分

> **主接项目：SRE Agent 的 per-class 切片 + unknown_rate + minimal fixture + 封闭枚举。** 防过拟合手段有真实落地，holdout 分离是补全。

### 🟢 我命中的参考答案要点

- ✅[真实] **Per-class 切片防总平均数刷分**：`backend/app/evals/metrics.py:26-57` 的 per-class precision/recall/f1/support + confusion_matrix——主指标不变但某些类在退化，切片会暴露。
- ✅[真实] **unknown_rate 防保守逃逸**：`backend/app/evals/metrics.py:72-81` 的 unknown_rate——模型如果学会"不知道就说不知道"，unknown_rate 会飙升，说明模型在滥用保守策略逃逸。
- ✅[真实] **封闭枚举消解"讨好 Judge"**：主指标是 `actual.incident_type == expected.incident_type`——纯等值比较，不存在"讨好 Judge 打高分"的空间。
- ✅[真实] **Minimal Fixture 减少对特定数据的过拟合**：每个 case 只提供信号承载的工具 fixture，未提供的工具返回受控空——不是随机 mock 数据，减少对特定 fixture 组合的过拟合。
- ⚙️[补全] **Holdout 分离**：11 个 case 是全部评测集，没有公开/私有切分。设计上需要 holdout set 不进日常调优循环。
- ⚙️[补全] **数据集持续刷新**：数据集目前是 Git 管理的静态 JSON，没有从线上采样的自动刷新机制。

### 🟡 我能拿到的加分项

- ✅[真实] **risk_accuracy 做安全维度的反刷分**：不能只看"答对了"，还要看"该升级的时候升级了没"——risk_accuracy 专门衡量风险判定是否正确。
- ✅[真实] **低成本的 case 扩展**：新增一种 IncidentType 只需枚举加一行 + 一个 case JSON，scorer 和 metrics 都按枚举动态聚合——扩展不会破坏评测框架。
- ⚙️[补全] **数据集版本化**：给数据集打版本号，每次刷新出新版本，评测结果关联数据集版本——可追踪"分数变化是模型变了还是数据集变了"。

### 🔴 危险信号（主动规避）

- ❌ 别说"我们有 holdout 分离"——坦诚当前数据集的 11 个 case 是公有集，holdout 是补全。
- ❌ 别说"评测集就是调参用的那批"——我有 per-class + unknown_rate + minimal fixture 多维度防刷分。
- ❌ 别说"数据集从不更新"——设计上需要持续从线上采样刷新。

### 完整应答（口语稿）

> 评测集过拟合是隐蔽但致命的问题——你反复在这批 case 上调 prompt，分数越来越高，但上线就崩。我的防过拟合有四道防线。第一道是 per-class 切片——总平均可能不降，但某类故障偷偷崩了，切片会暴露。第二道是 unknown_rate——如果模型学会了"全都说不知道"，unknown_rate 会飙升，说明它在滥用保守策略逃逸。第三道是封闭枚举——主指标是纯等值比较，不存在讨好 Judge 的空间。第四道是 minimal fixture——只给信号承载的工具有数据，未提供的返回受控空，减少对特定 fixture 组合的过拟合。

> 要坦诚的是——我目前没有 holdout 分离，11 个 case 是全部评测集。设计上需要公开集给开发者日常调优，私有 holdout 只在上线前跑，并且不进调优循环。数据集也需要持续刷新——线上分布会变，评测集要定期从线上采样新 case。数据集版本化我会加——每次刷新出新版本，评测结果关联版本号，这样分数变化你能分清楚是模型变了还是数据集变了。

> 再多说一个——我的 risk_accuracy 指标是专门看"该升级的时候有没有升级"。只看成功率会鼓励瞎试和保守回避，加上 risk_accuracy、unknown_rate、process 指标，整个指标体系本身就是反刷分设计的。

---

## Q17　CI/CD 中非确定性评测的门禁设计

> **主接项目：SRE Agent 的 `ci_metric: false` 声明 + 封闭枚举确定性主指标。** "刻意非 CI"是一个设计决策，要讲透。

### 🟢 我命中的参考答案要点

- ✅[真实] **主动声明非 CI 指标**：`backend/app/evals/report.py:11-12` 的 `_DISCLAIMER` 明确"本报告由真实 LLM 生成，结果存在波动，属非 CI 指标"。report meta 中 `ci_metric: False`（line 26）。prompt diff 报告同（`replay_runner.py:128`）。
- ✅[真实] **设计的理由**：不是做不了 CI 集成——是判断"非确定性指标做 merge gate"会随机红绿，门禁形同虚设。真实 LLM 需要 API key 不能在 CI 环境自动跑。
- ✅[真实] **确定性主指标 + 非确定性辅助指标的分层**：主指标是用封闭枚举等值比较（确定可复现）；辅助指标（如多轮 variance、hypothesis 抽查）不做门禁。门禁用确定性规则（schema 校验、工具可达性、安全规则）。
- ⚙️[补全] **CI pipeline 集成**：没有 GitHub Actions / Jenkins 配置文件，没有门禁阈值设置，没有 PR 自动触发评测。当前是纯 CLI 手动运行。

### 🟡 我能拿到的加分项

- ✅[真实] **"刻意做非 CI"比"强行做 CI"更有说服力**：我能讲清楚——不是能力不够，是在 LLM 非确定性的约束下，伪 CI 不如诚实非 CI。诚实比伪装好——面试官知道 LLM 有波动，你伪装 CI 可用反而减分。
- ⚙️[补全] **增量评测降低 CI 成本**：不跑全量跑"本次改动影响的 case"，用 repeat N 做分布对比。这样可以控制 CI 时间在可接受范围。
- ⚙️[补全] **分阶段方案**：语法/配置/确定性规则 → CI 门禁；LLM 行为质量 → 开发者手动触发评测。两种门禁独立。

### 🔴 危险信号（主动规避）

- ❌ 别说"CI 里跑 LLM 评分强制门禁"——非确定性导致随机红绿，门禁会被无视。
- ❌ 别说"跑一遍就够了"——无视 LLM 波动。
- ❌ 别说"所有评测都怼进 CI"——CI 跑 30 分钟以上开发者会绕过。

### 完整应答（口语稿）

> 这道题我有个明确立场。CI/CD 门禁必须分两类——确定性规则和非确定性评测。LLM 评测天然非确定——就算 temperature=0，浮点计算也有差异，同一 case 今天绿明天红。把非确定性指标当 merge gate 用，门禁形同虚设——开发者会直接无视随机红绿的结果。

> 所以我在 report 里主动声明 `ci_metric: false`，告诉所有人这个评测结果不是 CI 指标，不要拿来 block merge。这不是做不到——是我判断不该做。我的分层是：确定性规则——schema 校验、工具可达性、安全规则、配置正确性——这些做 CI 门禁，必须过；非确定性评测——LLM 行为质量——开发者手动触发，作为参考而非强制门禁。

> 我用的封闭枚举等值比较，本身就是为了把主指标做成确定性可复现的——用 `actual == expected` 而不是 LLM Judge 打分。这给未来 CI 集成留了口：当评测信号是确定性的，就可以进门禁。否则我会用 repeat N 分布比较——要求 mean 和 min 同时退化才阻断，减少随机波动误报。务实路径分阶段走：优先落地确定性门禁，再逐步引入分布对比和增量评测控制 CI 时间，而不是一上来把所有评测怼进 CI。

---

# 五、调试工程与前沿趋势

## Q18　调试三板斧：Trace 回放、步骤级对比、注入测试

> **主接项目：SRE Agent 的 replay_runner + Direct vs Runner compare + fixture 注入。** 回放和对比真实，注入测试范围有限。

### 🟢 我命中的参考答案要点

- ✅[真实] **Trace 回放（Replay）**：`backend/app/evals/replay_runner.py` 是完整的 CLI 回放框架——支持 --dataset / --mode / --repeat / --prompt-version 参数组合。`backend/app/evals/fixture_context.py` 用 ContextVar 注入 per-case fixture，在 `call_tool` 入口短路——固定输入 + 固定工具返回，重放模型决策但绝不重放副作用。
- ✅[真实] **步骤级对比（Step Diff）**：
  - Direct vs Runner diff：`backend/app/evals/replay_runner.py:50-67` 的 `diff_compare()` 双跑两种执行路径 diff 终态——暴露 checkpoint 序列化、DB 持久化引入的问题。
  - Prompt diff：`backend/app/evals/replay_runner.py:105-191` 的 `_run_prompt_diff()` 两个 prompt 版本对比 improved / regressed / unchanged。
- ✅[真实] **注入测试（Injection Testing）**：`backend/app/evals/fixture_context.py` 的 fixture 机制本质就是注入测试——在工具网关入口注入固定 mock 数据，验证 Agent 在不同证据组合下的行为。case_10 的稀疏证据就是典型的注入测试。
- ⚙️[补全] **注入测试的覆盖广度**：当前 fixture 注入的是正常/空/稀疏三种数据。缺少故障注入——工具返回超时、返回错误码、返回乱码、返回超大结果——每种降级行为的验证。

### 🟡 我能拿到的加分项

- ✅[真实] **决策重放 vs 副作用重放的严格分离**：回放时只重放模型决策（通过 fixture 注入工具返回），绝不重放副作用（不真打 MySQL/K8s）。这是回放安全性的核心约束——调试过程中误执行危险操作的生产事故案例太多了。
- ✅[真实] **Direct vs Runner 双跑是独创的对比纬度**：不是对比"旧版本 vs 新版本"这种常规做法，而是用"确定性高的路径"来校验"确定性低的路径"——直接在架构纬度上暴露问题。
- ⚙️[补全] **逐节点中间状态的 diff**：当前 diff 是终态对比，下一步我会加逐节点（planner 输出 / critic 质量分 / evidence 完整性）的中间状态 diff——定位"哪一步开始偏差"。

### 🔴 危险信号（主动规避）

- ❌ 别说"看日志就够"——Agent 日志量极大且不可复现，回放是必须的。
- ❌ 别说"回放时操作会重新执行"——fixture 短路在工具网关入口，副作用永不重放。
- ❌ 别说"注入测试全做了"——故障注入（超时/错误码/乱码）是补全。

### 完整应答（口语稿）

> Agent 调试我有三板斧。第一板斧是 trace 回放——完整 CLI 框架，固定输入 + 固定工具返回，重放模型决策链。关键设计是决策重放允许，副作用重放绝对禁止——fixture 在工具网关入口短路注入，在 mock/real 选择之前，跑 100 遍也不会真打 MySQL 和 K8s。调试过程中误执行危险操作的生产事故太多，这个约束我放在架构硬编码里。

> 第二板斧是步骤级对比。我有两种对比——Direct vs Runner 双跑 diff 和 prompt diff。Direct vs Runner 是用确定性高的纯逻辑图来校验带持久化的全链路，专门暴露 checkpoint 序列化丢字段、approval resume 后语义变化这种极度隐蔽的问题。Prompt diff 是改 prompt 后对比 improved / regressed / unchanged。

> 第三板斧是注入测试。我的 fixture 机制本身就是在工具入口注入 mock 数据——case_10 给稀疏证据就是验证 Agent 在信息不足时会不会强行武断。目前覆盖了正常/空/稀疏三种数据。设计上我会扩展故障注入——工具返回超时、返回错误码、返回乱码——每种降级行为都要验证一次。还有逐节点中间状态 diff——不仅仅是终态对比，从 planner 输出的第一步看偏差从哪开始。这个三板斧的设计思想是完整的，落地程度有高有低，我坦诚说。

---

## Q19　评测基础设施的分层架构与规模化设计

> **主接项目：SRE Agent 的 evals/ 模块化架构（8 个模块）+ ContextVar 并发设计。** 分层架构真实，并发执行是补全但并发安全已设计。

### 🟢 我命中的参考答案要点

- ✅[真实] **模块化 8 层架构**：`backend/app/evals/` 下有 8 个独立模块：
  - `case_loader.py` — 数据集加载 + schema 校验（REQUIRED_TOP / REQUIRED_TICKET fields）
  - `scorer.py` — 确定性评分（CaseResult dataclass）
  - `metrics.py` — 指标聚合（compute_metrics + aggregate_rounds）
  - `runner.py` — 数据集编排（run_one_case + run_dataset）
  - `replay_runner.py` — CLI 入口（argparse 参数管理）
  - `executors.py` — 执行策略（DirectGraphExecutor + RunnerGraphExecutor）
  - `fixture_context.py` — Fixture 注入（ContextVar 机制）
  - `report.py` — 报告生成（JSON + Markdown）
- ✅[真实] **ContextVar 并发安全设计**：`backend/app/evals/fixture_context.py:15-17` 的 `_fixture_var: ContextVar`——每个 async task 有独立的 fixture 副本，不同 case 的 mock 数据互不污染。Python 3.7+ `asyncio.create_task` 自动 copy context。并发安全在设计层就保证了。
- ✅[真实] **可持续扩展的架构**：新增 IncidentType → 枚举加一行 + case JSON。scorer/metrics 按枚举动态聚合不硬编码。执行路径（direct/runner/compare）和报告格式（JSON/Markdown）都可扩展。
- ⚙️[补全] **并发执行**：当前 `runner.py:57` 是顺序 for loop 跑 case，没有 `asyncio.gather` 多 case 并行。但 ContextVar 的并发安全设计已经为此准备好了。
- ⚙️[补全] **评测结果持久化与历史趋势**：评测结果当前是输出到 terminal + 报告文件，没有持久化到 DB 做历史趋势分析。

### 🟡 我能拿到的加分项

- ✅[真实] **两层指标不合并**：这是架构级决策——Graph Quality 和 Runtime Fidelity 的指标始终保持独立，不合并成一个综合分数。一个数字掩盖所有问题。
- ✅[真实] **并发安全先于并发执行**：ContextVar 的 per-task 隔离在设计时就考虑了——即使当前没用并发，架构已经保证加并发时不会出现 fixture 污染。
- ⚙️[补全] **评测元评测**：评测框架自身的稳定性（同一 case 重复跑 10 次分数是否一致）、准确率（评测结果 vs 人工审查的一致性）——这是进阶设计。

### 🔴 危险信号（主动规避）

- ❌ 别说"评测框架已经支持大规模并发"——坦诚当前是串行，但并发安全架构已就位。
- ❌ 别说"评测结果有历史趋势"——坦诚目前是输出报告文件，持久化和趋势是下一阶段。
- ❌ 别说"就几个 Python 脚本"——我的评测框架是有 8 个模块的分层架构。

### 完整应答（口语稿）

> 评测基础设施我做了八模块的分层架构。数据管理有 case_loader 做数据集加载和 schema 校验；评分有 scorer 做确定性评估，metrics 做聚合；执行引擎有三种策略——DirectExecutor 快速迭代、RunnerExecutor 全链路验证、Compare 双跑 diff；fixture 注入用 ContextVar 做 per-case 隔离；最后 report 输出 JSON 和 Markdown。每个模块独立文件、可单独测试、可替换。

> 一个关键设计决策是并发安全先于并发执行。fixture 用 ContextVar 做 per-task 隔离——Python 3.7 以上 asyncio.create_task 自动 copy context，不同 case 的 mock 数据互不污染。虽然现在 run_one_case 还是串行 for loop，但加并发时不用改 fixture 层——架构在第一天就考虑了并发。

> 另一个架构级决策是两层指标永不合并。Graph Quality 和 Runtime Fidelity 是两个独立维度——诊断准不准和 checkpoint 序列化对不对是正交问题，一个综合分数会掩盖严重的过程问题。要坦诚的是：评测结果现在是输出报告文件，没有持久化到 DB 做历史趋势分析，也没有并发执行。这两个是评测平台化的下一步。

---

## Q20　MCP 协议对可观测性的影响与 2026 演进趋势

> **主接项目：HS 的 tool registry（L0-L3）+ opencode plugin（tool.execute.before hook）；SRE 的 Tool Gateway。** MCP 是设计方向，现有工具治理体系是真实的先行基础。

### 🟢 我命中的参考答案要点

- ✅[真实] **Tool registry + spec-guard plugin 是 MCP 之前的工具治理层**：HS 的 `tools/registry/index.yaml` 定义 12 个工具（L0-L3 四级风险分级，含 owner/description/side_effect/permissions/audit 字段）。spec-guard plugin（`.opencode/plugin/spec-guard/index.ts:148-291`）提供三层 hook——`event(session.created)` 做 healthcheck、`tool.execute.before` 做风险分级控制 + 审批检查、`permission.ask` 做 allow/ask/deny 决策。这本质就是工具治理的集中控制平面——MCP 的思路是把这个下沉为协议标准。
- ✅[真实] **Deny-by-default 的权限模型**：permission.ask hook：L0 工具自动 allow，high/critical 风险级别已审批才 allow 否则 deny——这正是 MCP 主张的最小权限 + 审计集中化原则。
- ✅[真实] **SRE 的 Tool Gateway 也是同一思路**：`backend/app/tools/gateway.py` 的 schema 校验 + 风险等级 + fail-closed + 审计落库 + 脱敏——每个工具调用都经过网关。MCP server 是把这些能力内置到协议层。
- ⚙️[补全] **MCP 协议接入未实现**：HS 的七层架构文档中提到 MCP 工具协议接入在 Layer 3，优先级 P1，依赖外部 MCP server，标为"未来阶段"。当前是通过 opencode plugin 机制做 tool guard，不是 MCP 协议。

### 🟡 我能拿到的加分项

- ✅[真实] **能对三种工具治理方案做横向对比**：自建 Tool Gateway（SRE 真实的）— 定制灵活但每个工具适配器要自己写审计和安全校验；opencode plugin（HS 真实的）— 介于自建和标准协议之间，有 hook 机制；MCP — 标准化协议，工具调用集中化、审计天然支持、A2A 通信标准化，但生态在早期。
- ✅[真实] **渐进式接入策略**：先做 tool registry + plugin guard 建控制平面（这两个我已经落地了），再逐步接入 MCP 做工具协议标准化——不是"上来就全押 MCP"。评测的 fixture 注入点可以从应用层下沉到 MCP 协议层拦截——更标准化，与具体工具实现解耦。
- ✅[真实] **MCP 的核心优势论证**：MCP 把工具治理集中化——所有工具调用经过 MCP server 收口，天然的审计单子点。这比传统 Function Calling（每个工具各自独立接入）的可观测性强一个量级。多 Agent 场景下 A2A 通信走 MCP，trace context 在 MCP header 中透传——跨 Agent 追踪从"应用层手动维护"变成"协议层自动能力"。

### 🔴 危险信号（主动规避）

- ❌ 别说"MCP 我们已经全量接入了"——坦诚是设计方向，落地的基础设施（tool registry + plugin guard）是真实的。
- ❌ 别说"MCP 是银弹接了就不用做别的了"——MCP 治理的是工具调用，不治理 LLM 推理黑箱本身。prompt 质量、模型决策路径、上下文管理等可观测性仍需要应用层。
- ❌ 别说"我们不需要 MCP"——认可 MCP 在标准化的方向上的价值，渐进式接入是务实态度。

### 完整应答（口语稿）

> MCP 对可观测性的影响我会从三个工具治理方案的演进来讲。我的 SRE Agent 做了自建 Tool Gateway——schema 校验、风险等级、fail-closed、审计脱敏，每个工具调用收口到网关。我的 Harness System 往前走了半步——tool registry 做 L0-L3 四级分级 + spec-guard plugin 的 tool.execute.before hook 做权限拦截，还有 permission.ask 的 deny-by-default 模型。这两套的本质是什么？是工具治理的集中控制平面。MCP 是把这套能力从应用层下沉为协议标准——所有工具调用经过 MCP server 收口，天然的审计单子点，比传统每个工具各自独立接入的可观测性强一个量级。

> 多 Agent 场景下优势更明显——A2A 通信走 MCP，trace context 在 MCP header 里透传。跨 Agent 追踪从"应用层用 ContextVar 手动维护"变成"协议层自动能力"——这比我的 ContextVar 方案更标准化。评测框架也会受益——fixture 注入点可以从网关层的应用拦截下沉到 MCP 协议层拦截，与具体工具实现完全解耦。

> 但我的态度不是全押 MCP。MCP 是早期阶段，生态和标准化程度在快速演进中。我的务实路径是：先做 tool registry + plugin guard 建控制平面（这我已经落地了），再逐步接入 MCP 做工具协议标准化。而且 MCP 治理的是工具调用这层——LLM 推理黑箱本身的 prompt 质量、决策路径、规划合理性这些可观测性依然需要应用层来解决。MCP 是工具治理的下一个阶段，不是可观测性的终结。

---

_应答文档完 · 全 20 题 / 5 模块 · 配套《Agent 评测与可观测性 · 面试官评分手册》_
