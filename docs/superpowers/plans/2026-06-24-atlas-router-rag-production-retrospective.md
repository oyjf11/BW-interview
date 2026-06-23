# Atlas Router RAG Production Retrospective Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite `面试讲法.md` into a production-grade retrospective for the Atlas Router layer, positioning the RAG work as the online routing layer inside the Atlas application-generation agent.

**Architecture:** Keep the work scoped to one interview script file. Replace the current demo-style framing with a Router-inside-Atlas production narrative, then reshape the four core “母题” sections so they emphasize routing responsibility, evaluation discipline, service contracts, and online operability instead of isolated RAG tricks.

**Tech Stack:** Markdown, `rg`, `sed`, `git`

---

## File Map

- Modify: `/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md`
  Responsibility: the interview speaking script to be rewritten.
- Reference only: `/Users/ouyangjinfeng/Documents/projects1/interview/应用生成智能体.md`
  Responsibility: source of Atlas system context and Router-layer placement.
- Reference only: `/Users/ouyangjinfeng/Documents/projects1/interview/docs/superpowers/specs/2026-06-24-atlas-router-rag-production-retrospective-design.md`
  Responsibility: approved design spec that defines the target framing and constraints.

### Task 1: Reframe the Title, Positioning, and Mainline

**Files:**
- Modify: `/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md`
- Reference: `/Users/ouyangjinfeng/Documents/projects1/interview/docs/superpowers/specs/2026-06-24-atlas-router-rag-production-retrospective-design.md`

- [ ] **Step 1: Verify the current file is still demo-framed**

Run:

```bash
rg -n 'cheat-sheet|一句话定位|可执行任务图|DAG' '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
```

Expected:

```text
Matches for demo-style framing such as "cheat-sheet", "可执行任务图(DAG)", and the current one-line定位.
```

- [ ] **Step 2: Replace the title and opening with Atlas Router positioning**

Update the top section of `面试讲法.md` so it starts with content in this shape:

```md
# Atlas 应用生成智能体中的 Router 层：生产级 RAG 实现复盘

> 用法：先盖住答案，看「母题」自己复述；卡壳了再翻答案。本文件讲的是
> Atlas 应用生成智能体里 Router 层的生产实现复盘，重点不是独立问答，
> 而是如何把自然语言需求稳定路由到应用范式，再交给后续 Planner 和 Executor。
```

- [ ] **Step 3: Rewrite the mainline data flow to show Atlas layering**

Replace the current “用户大白话需求 -> 可执行任务图(DAG)” section with content in this shape:

````md
## 主线（先背这条，其它都往上挂）

**一句话定位**：这套 RAG 不是独立问答系统，而是 Atlas 应用生成智能体里的
**Router 层**。它负责把高噪声自然语言需求收敛到少数高质量应用范式，再把
**结构化路由结果**交给后续 **Planner** 和 **Executor**。

**数据流全景（白板能画）**
```
用户自然语言需求
  ↓  Router   : ① query 改写 ② 范式检索 ③ rerank 判别 ④ 输出结构化路由结果
命中应用范式父块（完整 plan + pitfalls + 路由标签）
  ↓  Planner  : 基于范式和用户约束生成任务 DAG
产出任务 DAG（任务 + 依赖）
  ↓  Executor : 按依赖执行、验证、失败隔离

旁路：离线消融 + 线上观测 = 持续优化 Router 的召回、判别性与误路由率
```
````

- [ ] **Step 4: Verify the new framing is present**

Run:

```bash
rg -n 'Atlas 应用生成智能体中的 Router 层|结构化路由结果|误路由率' '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
```

Expected:

```text
Three matches showing the new title, Router-layer framing, and online-optimization wording.
```

- [ ] **Step 5: Commit**

```bash
git add '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
git commit -m "docs: reframe interview script as Atlas router retrospective"
```

### Task 2: Rewrite Mother Question 1 and 2 Around Router Value and Validation

**Files:**
- Modify: `/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md`
- Reference: `/Users/ouyangjinfeng/Documents/projects1/interview/应用生成智能体.md`
- Reference: `/Users/ouyangjinfeng/Documents/projects1/interview/docs/superpowers/specs/2026-06-24-atlas-router-rag-production-retrospective-design.md`

- [ ] **Step 1: Verify the current Question 1/2 wording is still trick-centric**

Run:

```bash
rg -n '母题 1|small-to-big|反直觉结论|margin 从 0.21→0.51|0.4 设' '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
```

Expected:

```text
Matches showing the current retrieval-tricks framing and the fixed-threshold style margin wording.
```

- [ ] **Step 2: Rewrite Question 1 so it explains why Router exists**

Replace the Question 1 section with content in this shape:

```md
## 母题 1：Atlas 的 Router 层为什么要做成 RAG，而不是直接让 Planner 硬吃用户需求？

**一句话**：应用生成场景里，用户输入口语化、噪声高、业务术语不稳定，如果
直接让 Planner 从原始输入起步，首轮规划很容易跑偏，所以我先用 Router 层
做范式识别和路由收敛。

**① 为什么要先做 Router**
- Router 的目标不是回答用户，而是先把需求归到少数高质量应用范式上。
- 这样能给 Planner 一个稳定的模式先验，缩小搜索空间，降低误规划率。

**② 检索怎么做**
- 父子分块的 small-to-big：子块贴近用户口语做召回，父块保留完整 plan_steps 和 pitfalls 供下游消费。
- query rewrite：补 vocabulary gap，把口语说法改写成更接近知识库范式的术语。
- 两阶段 rerank：先保证召回，再拉开候选之间的判别性。
- metadata 过滤：把 domain、complexity、need_auth、multi_user 这类标签前置到路由阶段。

**③ 这层的生产价值**
- 不是“检索到几个 chunk”，而是降低首轮误路由，减少后面 Planner 和 Executor 在错误起点上的无效放大。
```

- [ ] **Step 3: Rewrite Question 2 around offline plus online validation**

Replace the Question 2 section with content in this shape:

```md
## 母题 2：你怎么验证 Router 层在线上确实改善了主链路，而不是只把检索分数做漂亮？

**一句话**：我把验证拆成两层，离线看路由召回和判别性，线上看它是否真的改善了
主链路的规划稳定性和收敛效率。

**① 离线消融看什么**
- A：裸向量召回
- B：向量召回 + query rewrite
- C：向量召回 + query rewrite + rerank
- 重点看 hit、top1/top2 margin、翻盘样本和翻车样本

**② 这些离线结果说明什么**
- rerank 的价值不一定体现在命中率绝对值，而更体现在候选间判别性是否被拉开。
- margin 提升说明 Router 更敢区分“最像哪个范式”和“第二像哪个范式”。

**③ 这些离线结果不能说明什么**
- 检索分数本身不能直接证明端到端生成质量。
- 我不会把某个 margin 直接当成固定线上阈值，而是把它作为路由置信度信号之一。

**④ 线上最终看什么**
- 首轮规划稳定性
- 误路由率
- 无效生成率
- 人工接管率
- 端到端收敛轮次
```

- [ ] **Step 4: Add the calibrated-threshold wording**

Ensure the section contains this sentence exactly once:

```md
margin 提升说明 rerank 增强了路由判别性，在线上我会把它作为路由置信度信号之一，再结合误路由样本和人工接管结果做阈值校准。
```

- [ ] **Step 5: Verify the old brittle threshold wording is gone and the new validation framing exists**

Run:

```bash
rg -n '0.4 设|误路由率|人工接管率|首轮规划稳定性' '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
```

Expected:

```text
No match for "0.4 设".
Matches for "误路由率", "人工接管率", and "首轮规划稳定性".
```

- [ ] **Step 6: Commit**

```bash
git add '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
git commit -m "docs: rewrite router rationale and validation sections"
```

### Task 3: Rewrite Mother Question 3 and 4 Around Contracts and Operability

**Files:**
- Modify: `/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md`
- Reference: `/Users/ouyangjinfeng/Documents/projects1/interview/docs/superpowers/specs/2026-06-24-atlas-router-rag-production-retrospective-design.md`

- [ ] **Step 1: Verify the current Question 3/4 still centers Planner/Executor internals**

Run:

```bash
rg -n '怎么保证 LLM 输出能用|拓扑排序|Kahn|DAG 怎么被消费|pending → running' '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
```

Expected:

```text
Matches showing the current focus is on Planner JSON handling and Executor state-machine details.
```

- [ ] **Step 2: Rewrite Question 3 around the Router/Planner contract**

Replace the Question 3 section with content in this shape:

```md
## 母题 3：Router 层是怎么和 Planner 解耦的，如何保证下游可消费？

**一句话**：Router 不直接输出大段自然语言，而是输出可供 Planner 消费的结构化
路由结果，把“路由判断”与“任务生成”清晰拆开。

**① Router 输出什么**
- `pattern_id`
- `domain`
- `complexity`
- `need_auth`
- `multi_user`
- `confidence`
- `retrieved_pitfalls`
- 归一化后的用户意图

**② Planner 怎么消费**
- 把命中的应用范式当成受约束的模式先验，而不是从零开始自由发散。
- 结合用户原始需求和 Router 输出，生成后续任务 DAG。

**③ 低置信度怎么处理**
- 不强推单一路由。
- 可以给 Planner 多候选，也可以触发进一步澄清。
```

- [ ] **Step 3: Rewrite Question 4 around stability, fallback, and observability**

Replace the Question 4 section with content in this shape:

```md
## 母题 4：这套 Router 层在线上怎么做稳定性、回退和可观测？

**一句话**：Router 真正的生产难点不是把 rerank 接上，而是依赖波动、低置信度样本
和误路由回灌都要可控。

**① 回退策略**
- rerank 失败时回退到 cosine，不阻断主链路。
- 低置信度时不强推单一路由，而是多候选或澄清。

**② 可观测**
- 记录 rewrite 前后 query
- 记录 recall topk
- 记录 rerank 结果和最终命中范式
- 记录 confidence 信号和最终路由选择

**③ 持续优化闭环**
- 把误路由样本沉淀回评测集
- 把父块/子块知识版本化
- 在知识变更后追踪回归
```

- [ ] **Step 4: Verify Router ownership is now clear**

Run:

```bash
rg -n 'pattern_id|retrieved_pitfalls|rewrite 前后 query|误路由样本|Kahn|pending → running' '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
```

Expected:

```text
Matches for "pattern_id", "retrieved_pitfalls", "rewrite 前后 query", and "误路由样本".
No matches for "Kahn" or "pending → running" in the main mother-question sections.
```

- [ ] **Step 5: Commit**

```bash
git add '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
git commit -m "docs: rewrite router contract and operability sections"
```

### Task 4: Rewrite the Opening Script, Deep-dive Script, and Closing Cues

**Files:**
- Modify: `/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md`
- Reference: `/Users/ouyangjinfeng/Documents/projects1/interview/docs/superpowers/specs/2026-06-24-atlas-router-rag-production-retrospective-design.md`

- [ ] **Step 1: Verify the current opening still frames the work as a generic demo**

Run:

```bash
rg -n '电梯版|介绍个项目|我做了个把大白话需求转成可执行任务图的 demo|深挖版' '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
```

Expected:

```text
Matches showing the current opening talks about a generic DAG demo.
```

- [ ] **Step 2: Replace the opening with the approved 30-second Router pitch**

Replace the current opening section with:

```md
## 开场白模板

**30 秒版**
> "我在应用生成智能体 Atlas 里负责过 Router 层的 RAG 能力，它不是一个独立问答系统，
> 而是整条生成链路的入口收敛层。核心作用是先把用户高噪声、口语化的需求路由到少数
> 高质量应用范式上，再把结构化路由结果交给后面的 Planner 和 Executor。这样做的原因是，
> 应用生成场景里如果入口理解错了，后面的规划和执行只会把错误放大，所以我重点优化的是
> 路由召回、判别性、回退策略和线上观测闭环。"
```

- [ ] **Step 3: Replace the deep-dive section with a four-part 90-second script**

Use content in this shape:

```md
**90 秒深挖版**
- 为什么把 Router 从 Planner 前面单独拆出来
- 父子分块、query rewrite、rerank 怎么一起工作
- 我怎么同时看离线消融和线上主链路指标
- 这套 Router 层怎么处理低置信度、依赖失败和误路由回灌
```

- [ ] **Step 4: Rewrite the parameter summary so it supports the new production story**

Adjust the “关键参数速记” section so it keeps only supportive facts such as:

```md
| 项 | 值 |
|----|----|
| Router 职责 | query rewrite + 范式检索 + rerank + 结构化路由输出 |
| 父子知识组织 | 8 父块 × 3 场景子块 = 24 子块 |
| rerank | 粗排 `recall_k=8` → 精排 |
| 评测 | 离线 A/B/C 消融 + 线上误路由/人工接管/收敛轮次观测 |
| 下游接口 | `pattern_id` / `confidence` / `retrieved_pitfalls` 等结构化字段 |
```

- [ ] **Step 5: Run a final consistency pass**

Run:

```bash
rg -n 'demo|可执行任务图|固定线上阈值|Kahn|pending → running' '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
sed -n '1,260p' '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
```

Expected:

```text
No stale demo-centric phrasing that contradicts the new Atlas Router framing.
The full document reads as one consistent production retrospective.
```

- [ ] **Step 6: Commit**

```bash
git add '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
git commit -m "docs: finalize Atlas router interview retrospective"
```

### Task 5: Final Verification and Delivery

**Files:**
- Modify: `/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md`

- [ ] **Step 1: Check the working tree before final review**

Run:

```bash
git status --short
```

Expected:

```text
Only the intended interview-script change remains, along with any unrelated pre-existing user changes.
```

- [ ] **Step 2: Read the final document once top-to-bottom**

Run:

```bash
sed -n '1,260p' '/Users/ouyangjinfeng/Documents/projects1/interview/面试讲法.md'
```

Expected:

```text
The file consistently presents the RAG work as Atlas's Router layer, keeps Router/Planner/Executor boundaries clear, and avoids brittle claims.
```

- [ ] **Step 3: Prepare the delivery summary**

Use a summary in this shape:

```md
- Repositioned the script from a generic RAG/DAG demo to an Atlas Router production retrospective.
- Rewrote the four mother questions around routing value, validation discipline, Router/Planner contracts, and online operability.
- Updated the opening and deep-dive scripts so the interview narrative is production-first and easier to defend under follow-up.
```

- [ ] **Step 4: Confirm there is no unintended extra work left**

```bash
git status --short
```

Expected:

```text
No unexpected modified files from the rewrite flow. If the previous task commits were completed, the interview script should already be committed.
```
