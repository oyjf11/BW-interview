# Atlas Router RAG Production Retrospective Design

## Goal

Rewrite `面试讲法.md` as a production-grade retrospective for the RAG-based Router layer inside the Atlas application-generation agent, instead of presenting it as an isolated RAG demo or a full end-to-end DAG system.

## Source Context

- The current `面试讲法.md` describes a `Router -> Planner -> Executor` flow and focuses on how to speak about the project in interviews.
- `应用生成智能体.md` establishes that Atlas is a larger production system whose main chain is `natural language request -> orchestration -> agent planning/execution -> preview/deploy`.
- The RAG work in `面试讲法.md` should therefore be reframed as the Router layer of Atlas, responsible for intent normalization, pattern routing, and structured handoff to downstream planning.

## Target Positioning

The revised document should present the system as:

`Atlas application-generation agent's Router layer: a production RAG implementation that stabilizes noisy natural-language requests before planning and execution.`

It should explicitly communicate:

- This RAG is not a standalone QA bot.
- This RAG is not the entire Atlas system.
- This RAG is the production Router layer that narrows user requests onto a small set of application patterns and emits structured routing results for downstream Planner and Executor components.

## Audience and Use

- Primary audience: interviewers probing system design, RAG architecture, and production AI engineering.
- Secondary audience: the user rehearsing a concise but defensible production story.
- The document remains a speaking script, but it should sound like a production retrospective rather than a demo cheat sheet.

## Design Principles

### 1. Production-first framing

Every major section should answer three questions:

- What production problem existed in Atlas?
- What did the Router layer do to solve it?
- How was the solution made stable, measurable, and operable online?

### 2. System-boundary clarity

The text must keep Router, Planner, and Executor clearly separated:

- Router handles intent understanding, retrieval, reranking, route narrowing, and structured output.
- Planner consumes Router output and generates executable task graphs.
- Executor consumes the task graph and handles execution-state semantics.

The revised document should not blur these responsibilities.

### 3. Strong production narrative without brittle fake specifics

The writing may describe the system as production-run and online-validated, but should avoid unverifiable precision such as exact online percentages, specific traffic numbers, or hard claims about thresholds unless the text also explains the calibration logic in general terms.

### 4. Interview defensibility

The revised script should sound senior and production-aware under follow-up:

- no direct claim that margin alone defines an online threshold
- no claim that retrieval metrics alone prove end-to-end success
- no overclaim that fallback guarantees semantic correctness

## Proposed Document Structure

## 1. Title and One-line Positioning

Replace the current demo-style title with a title that anchors the work inside Atlas.

Recommended title:

`Atlas 应用生成智能体中的 Router 层：生产级 RAG 实现复盘`

Recommended one-line positioning:

`这套 RAG 不是独立问答系统，而是 Atlas 应用生成智能体里的 Router 层，负责把高噪声自然语言需求稳定路由到应用范式，再把结构化结果交给后续 Planner 和 Executor。`

## 2. Mainline Narrative

Replace the current “user request to executable DAG” framing with a layered Atlas flow:

`用户自然语言需求`
`-> Router：query rewrite、范式检索、rerank、路由判定、结构化结果输出`
`-> Planner：基于命中的应用范式和用户约束生成任务 DAG`
`-> Executor：按依赖执行、校验、隔离失败`

Key emphasis:

- Router is an entry-point convergence layer.
- Its job is to reduce route ambiguity before planning starts.
- Wrong routing at the front amplifies downstream errors; therefore Router quality materially affects the whole generation chain.

## 3. Question 1 Rewrite: Why Router Uses RAG

Recommended section title:

`母题 1：Atlas 的 Router 层为什么要做成 RAG，而不是直接让 Planner 硬吃用户需求？`

Required structure:

- Problem:
  Atlas users describe needs in colloquial, noisy, unstable language. If Planner starts directly from raw input, first-pass planning often drifts or overfits to surface wording.
- Solution:
  Use small-to-big parent/child chunking, query rewrite, two-stage rerank, and metadata filtering to map requests onto a small set of high-quality application patterns.
- Value:
  Router does not aim to answer the user. It narrows Planner’s search space, injects stable pattern priors, and reduces misplanning.

Required message shift:

- From “I built a retrieval system with a few tricks”
- To “I separated a production routing concern from downstream planning to improve stability”

## 4. Question 2 Rewrite: How Router Was Validated

Recommended section title:

`母题 2：你怎么验证 Router 层在线上确实改善了主链路，而不是只把检索分数做漂亮？`

Required structure:

- Offline validation:
  Describe A/B/C ablation on bare vector retrieval, rewrite-enhanced retrieval, and rerank-enhanced retrieval.
- What offline results mean:
  Margin improvements indicate stronger route discriminability and better confidence signaling.
- What offline results do not prove:
  Retrieval scores alone do not prove end-to-end planning or execution quality.
- Online validation:
  Tie Router quality to production chain outcomes such as first-pass planning stability, misrouting rate, invalid generation rate, manual takeover rate, and end-to-end convergence rounds.

Required replacement:

Replace any direct “margin increase implies fixed threshold X” statement with:

`margin 提升说明 rerank 增强了路由判别性，在线上我会把它作为路由置信度信号之一，再结合误路由样本和人工接管结果做阈值校准。`

## 5. Question 3 Rewrite: Router/Planner Contract

Recommended section title:

`母题 3：Router 层是怎么和 Planner 解耦的，如何保证下游可消费？`

The section should focus on interface contracts, not only model choice.

Required content:

- Router output is a structured routing payload, not free-form prose.
- Suggested fields:
  `pattern_id`, `domain`, `complexity`, `need_auth`, `multi_user`, `confidence`, `retrieved_pitfalls`, and normalized intent details.
- Planner consumes this payload as a constrained prior for DAG generation.
- Low-confidence cases should be described as either multi-candidate handoff or clarification-triggering, not as a forced single route.

This section should present the user as someone who understands service boundaries and downstream contracts.

## 6. Question 4 Rewrite: Stability, Fallback, and Observability

Recommended section title:

`母题 4：这套 Router 层在线上怎么做稳定性、回退和可观测？`

Required content:

- Rerank failure falls back to cosine retrieval without breaking the chain.
- Low-confidence routing does not force a brittle single-path decision.
- Logging/tracing should include:
  rewritten query, pre-rewrite query, top-k recall set, rerank results, final pattern choice, and confidence signals.
- Misroutes should be collected back into the evaluation set.
- Parent/child pattern knowledge should be versioned so that regressions can be traced to knowledge changes.

This section is the main production upgrade over the current document.

## 7. Opening Script Rewrite

The current opening should be replaced with a Router-inside-Atlas production pitch.

Recommended 30-second opening:

`我在应用生成智能体 Atlas 里负责过 Router 层的 RAG 能力，它不是一个独立问答系统，而是整条生成链路的入口收敛层。核心作用是先把用户高噪声、口语化的需求路由到少数高质量应用范式上，再把结构化路由结果交给后面的 Planner 和 Executor。这样做的原因是，应用生成场景里如果入口理解错了，后面的规划和执行只会把错误放大，所以我重点优化的是路由召回、判别性、回退策略和线上观测闭环。`

## 8. Deep-dive Script Rewrite

The current “亮点三选一” format should become a structured 90-second explanation covering:

- Why Router was separated from Planner
- How parent/child chunking + rewrite + rerank work together
- How effectiveness was validated offline and online
- How the Router layer was made operable under failures and ambiguity

## 9. Language Rules for the Rewrite

The revised `面试讲法.md` should use these tone rules:

- Speak as a production engineer doing a system retrospective.
- Prefer “Router improved route stability for Planner” over “RAG retrieved correctly.”
- Prefer “confidence signal + calibration” over “fixed threshold.”
- Prefer “observability and fallback” over “works fine when dependencies fail.”
- Avoid framing Executor mechanics as if they belong to Router.

## 10. Out-of-Scope

The rewrite should not:

- turn into a full Atlas architecture doc
- re-explain all of Planner or Executor internals in depth
- fabricate exact online business metrics
- claim that the Router alone proves full-system success

## Acceptance Criteria

The design is successful if the rewritten `面试讲法.md`:

- clearly positions the RAG work as Atlas’s Router layer
- reads as a production retrospective rather than a demo cheat sheet
- strengthens the interview story under follow-up on evaluation, thresholds, contracts, and observability
- keeps Router, Planner, and Executor boundaries clear
- avoids brittle or statistically weak claims
