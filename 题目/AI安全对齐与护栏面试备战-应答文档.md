# AI 安全、对齐与护栏 — 面试应答文档

> 主接项目：sre-agent (OpsPilot) 的多层护栏架构（ToolGateway + risk_gate + HITL + 审计脱敏）
> 辅助项目：harness-system 的工具风险分级治理（L0-L3 + 宪法门控 + 审批链）
> 补充项目：rag 的 RAG 管线安全面（FTS5 注入防御 + 检索结果安全盲区）

---

## 模块一：AI 安全——提示注入与攻击面

### Q1　提示词注入原理、分类与防御体系（★拉档题）

> 主接项目：sre-agent 的 SpecialistAgent 角色分离 + 工具白名单架构。system/user role 硬分离 + 工具层兜底拦截。

#### 🟢 我命中的参考答案要点

- ✅[真实] system/user role 硬分离——系统提示词用 `{"role": "system"}` 独立 message，用户输入用 `{"role": "user"}`，永远不拼接。三个 LLM provider（OpenAI/DeepSeek/MiniMax）统一执行此策略。
  → `backend/app/graph/nodes/specialist_agent.py:174-180`（SpecialistAgent.run 中系统提示和用户提示独立构建）、`backend/app/llm_client.py:257-259,364-366,404-406`（三个 provider 的 system/user 分离）

- ✅[真实] 工具白名单兜底——即使 LLM 被注入要求执行危险操作，工具层通过 `_tool_allowed()` 校验拦截不在白名单的工具调用，注入错误消息让 LLM 自我修正。
  → `backend/app/graph/nodes/specialist_agent.py:297-311`（运行时逐 tool_call 校验白名单）、`:33`（FORBIDDEN_TOOLS = {"execute_action"}）

- ✅[真实] OWASP LLM Top10 前三（提示注入 > 不安全输出 > 训练投毒）在 harness-system 设计文档中有映射——七层架构的安全评测维度覆盖了注入和输出处理。
  → `harness-system/七层架构.md:929`（"能否抵抗 prompt injection"作为 Safety Eval 主题）

- ⚙️[补全] 输入侧语义注入检测——目前项目中没有正则校验或语义安全分类器对用户输入做注入检测，依赖的是结构层面的角色分离和工具层的白名单拦截。如果要加，会在 L1 输入护栏层加一个小模型做语义安全分类。

#### 🟡 我能拿到的加分项

- ✅[真实] 间接注入的防御思路已落地——FORBIDDEN_TOOLS 禁止所有 specialist agent 调用执行类工具，即使 RAG 检索结果中被注入了"请执行 rm -rf"的指令，agent 也没有执行权限。这是一种纵深防御中的"兜底层"设计。
- ✅[真实] 工具白名单 + 环境策略两层拦截——EnvPolicy 在生产环境限制可执行动作为 restart/scale_up/scale_down，禁止 delete/drop/truncate。即使注入绕过了工具白名单，环境策略层仍能拦截。
  → `backend/app/tools/policies/env.py:5-53`（ENV_ALLOWLIST + RESTRICTED_ACTIONS）

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们的系统有注入检测" → 目前只有结构层面的角色分离和工具白名单，没有语义级别的注入检测器。但如果被追问，我会说"下一阶段计划在 L1 输入护栏加小模型做语义安全分类"。
- ❌ 别说"prompt 写好了就不会被注入" → 我知道间接注入（RAG 投毒）是真实威胁，我的防御是分层纵深而非单点防护。

#### 完整应答（口语稿）

> 我在 sre-agent 项目里做了三层提示注入防御。第一层是结构层面的 role 分离——系统提示词和用户输入在 messages 数组里用不同的 role 字段，system prompt 从不和用户输入做字符串拼接。三个 LLM provider——OpenAI、DeepSeek、MiniMax——都在 LLMClient 里统一执行这个分离策略。
>
> 第二层是工具白名单兜底。每个 specialist agent 有自己的 tool_names 白名单，运行时每个 LLM 返回的 tool_call 都要在白名单里校验。如果 LLM 被注入建议执行一个不在白名单的工具——比如说 delete 操作——系统不会执行，而是注入一个错误消息让 LLM 自我修正。而且所有 specialist agent 有一组全局的 FORBIDDEN_TOOLS，里面包含 execute_action，意味着即使 agent 被注入，它也发不出任何写操作指令。
>
> 第三层是环境感知的策略控制。我们的 EnvPolicy 在生产环境限制了可执行动作的 allowlist——只能 restart、scale_up、scale_down——delete、drop、truncate 这些危险操作在生产环境直接被拒绝。这个设计思路是：即使前两层都被绕过，环境策略层仍然是一道硬栅栏。

---

### Q2　间接注入——RAG 知识库投毒攻击与防御

> 主接项目：rag 的 RAG 检索管线。FTS5 查询安全 + 向量阈值过滤，但检索结果安全过滤缺失是核心盲区。

#### 🟢 我命中的参考答案要点

- ✅[真实] FTS5 全文搜索做了查询注入防御——将用户查询中的 SQL FTS5 特殊字符 `'()*:^` 替换为空格，防止搜索语法本身成为注入载体。
  → `rag/server/src/modules/knowledge/retrieval.service.ts:96-101`（`safeQ = query.replace(/['"()*:^]/g, ' ').trim().split(/\s+/).filter(Boolean).join(' OR ')`）

- ✅[真实] 向量搜索有相似度阈值过滤——cos_sim < 0.3 的检索结果直接丢弃，相当于粗粒度的"质量门控"，也可以拦截部分向量碰撞攻击。
  → `rag/server/src/modules/knowledge/retrieval.service.ts:30`（`VEC_THRESHOLD = 0.3`）、`:87`（`.filter((r) => r.score >= this.VEC_THRESHOLD)`）

- ✅[真实] 文件上传有扩展名白名单——只允许 `.md`, `.txt`, `.docx`，结合文件大小限制（10-20MB），是 RAG 知识库入口的第一道防线。
  → `rag/server/src/modules/knowledge/knowledge.controller.ts:88-91` + `file/file.controller.ts:14-18`

- ⚙️[补全] 检索结果安全过滤层——目前检索到的文档 chunk 直接拼接到 LLM system prompt 中，没有经过安全内容审核。这是间接注入的核心盲区。如果要做，会在检索后、注入 LLM 前加一层内容安全分类（用小模型或规则引擎），过滤掉包含指令改写类模式的文档。

#### 🟡 我能拿到的加分项

- ✅[真实] sre-agent 中 FORBIDDEN_TOOLS + EnvPolicy 的设计是间接注入的纵深防御——即使 RAG 检索结果被投毒，agent 也没有执行危险操作的权限。这体现了"假设检索结果不可信"的安全设计原则。
- ✅[真实] harness-system 在设计层面覆盖了 RAG 数据污染防御——七层架构中明确列了"RAG 知识库投毒"作为威胁模型，PRD 中将其列为 P1 任务。
  → `harness-system/docs/superpowers/specs/2026-06-19-harness-prd.md:534`（"安全过滤 | P1 | PII 脱敏、注入检测"）

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们有内容过滤" → 目前检索结果直接拼入 prompt，没有安全过滤层。但我会说"这是已知盲区，设计方案是在检索结果进入 LLM 前加一层安全内容分类器"。
- ❌ 别说"文件上传白名单够了" → 白名单只能挡文件格式，不能挡文件内容——恶意 .md 文件完全可以包含注入指令。

#### 完整应答（口语稿）

> 间接注入是目前最难防的攻击面，因为攻击载荷不在用户输入里，而在检索结果里。我参与的 rag 项目里，这个问题非常明显——检索到的文档 chunk 直接拼接到 LLM 的 system prompt 里，没有经过任何安全内容审核。
>
> 但我们在入口和中间层做了一些防护。入口层是文件上传的扩展名白名单——只允许 md、txt、docx 三种格式，搭配大小限制。中间层是 FTS5 查询的注入防御——把用户查询里的 FTS5 特殊字符做了转义，防止搜索语法本身被利用。向量搜索也有 0.3 的相似度阈值，低质量结果直接滤掉。
>
> 但最关键的盲区——检索结果的内容安全过滤——目前是缺失的。我的设计思路是：在检索结果注入 LLM 之前，加一层小模型做安全分类，识别文档中是否包含"忽略之前的指令"、"你是"这类提示注入模式。同时，sre-agent 的 FORBIDDEN_TOOLS + EnvPolicy 提供了兜底——即使检索结果被投毒，agent 也没有执行权限。这是一种基于"纵深防御"的设计：我不假设检索结果是安全的，我用多层约束来限制被注入后能造成的伤害。

---

### Q3　模型投毒（训练投毒/后门投毒/RAG 污染）与防御

> 主接项目：harness-system 的工具 risk_level 元数据作为"模型行为护栏"。虽然不直接做模型训练，但工具风险分级是防投毒后危害的关键设计。

#### 🟢 我命中的参考答案要点

- ✅[真实] 工具元数据中的 risk_level（LOW/MEDIUM/HIGH/CRITICAL）是一种"模型行为护栏"——即使基座模型被投毒建议危险操作，工具层根据 risk_level 决定是否需要审批或直接拒绝。
  → `harness-system/tools/registry/index.yaml:1-142`（每个工具声明 risk_level、approval_required、runtime_limits、side_effect）

- ✅[真实] sre-agent 也做了类似的工具风险分级——ToolMetadata 包含 risk_level、requires_approval、timeout_ms。
  → `sre-agent/backend/app/tools/schemas/__init__.py:31-38`

- ✅[真实] 文件上传白名单（扩展名 + 大小限制）是 RAG 数据入口的第一道防线，可以拦截部分恶意文档投毒。
  → `rag/server/src/modules/knowledge/knowledge.controller.ts:88-91`

- ⚙️[补全] 向量签名校验——目前没有对文档嵌入做哈希签名验证。如果要防向量碰撞投毒，会设计一个签名机制：为每个文档的 embedding 计算哈希签名，检索时对比当前向量签名与入库时是否一致，不一致则告警。

#### 🟡 我能拿到的加分项

- ✅[真实] harness-system 的 spec validation 可以扩展到训练数据审计——validator 的 DSL 引擎已经支持 `exists`、`non_empty`、`in` 等校验，这套规则引擎可以直接用于训练数据质量审计。
- ✅[真实] EnvPolicy 的生产环境限制——禁止 delete/drop/truncate——是一种基于 "最小权限" 原则的设计，直接降低了模型被投毒后推荐危险操作的危害半径。

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们做了训练数据审计" → 不训练模型，没有训练数据审计流程。
- ❌ 别说"后门检测做了" → 没有推理层后门检测。但我知道检测思路是异常输出触发人工审核。

#### 完整应答（口语稿）

> 我们的项目不直接做模型训练，所以训练投毒和后门投毒不是我们的直接经验。但有一个设计思路很关键：工具层的风险分级本质上是"模型行为护栏"。在 harness-system 里，每个工具都有 risk_level——L0 只读、L1 本地写、L2 外部写、L3 高危——和 approval_required 标记。sre-agent 的 ToolMetadata 也是这样设计的。
>
> 这个设计的价值在于：即使底层 LLM 被投毒推荐了危险操作，工具层会根据 risk_level 拦截。L3 操作必须人工审批，L0-L2 在环境策略允许范围内才放行。生产环境还禁止了 delete、drop、truncate 这些危险操作。
>
> 在 RAG 数据污染这个场景上，我们的文件上传有白名单校验——只允许 md、txt、docx，大小 10-20MB 封顶。但这只是初级防御。要真正防向量投毒，还需要向量签名校验：为每个文档的 embedding 计算哈希，检索时验证签名一致性，防止攻击者构造语义相近但内容恶意的新向量混入库中。

---

### Q4　模型窃取、对抗样本与影子 AI 综合

> 主接项目：sre-agent 的 timeout + retry 机制 + EnvPolicy 环境隔离提供基础的 API 滥用防护。

#### 🟢 我命中的参考答案要点

- ✅[真实] 工具级 timeout 机制——每个工具通过 ToolMetadata 声明 timeout_ms，在 SpecialistAgent 中通过 `asyncio.wait_for()` 强制执行，防止恶意长查询耗尽资源。
  → `sre-agent/backend/app/tools/schemas/__init__.py:36`（timeout_ms 声明）、`specialist_agent.py:328`（asyncio.wait_for 执行）

- ✅[真实] retry 指数退避——工具调用失败时用 tenacity 库做最多 2 次重试，指数退避 1-10s。这不是直接的反窃取，但限制了异常高频调用的放大效应。
  → `sre-agent/backend/app/tools/gateway.py:1065-1079`

- ✅[真实] 环境隔离——EnvPolicy 在生产环境只允许 restart/scale_up/scale_down，限制 delete/drop/truncate——相当于缩小了攻击者可利用的攻击面。
  → `sre-agent/backend/app/tools/policies/env.py:5-53`

- ⚙️[补全] IP 级限流和 API 画像——目前没有 IP 级限流、没有查询语义分析、没有异常调用模式检测。如果要做，会在 API 网关层加限流 + 用户画像异常检测。

#### 🟡 我能拿到的加分项

- ✅[真实] eval fixture 模式——评估场景下用 fixture 数据替代真实 API 调用，减少不必要的 API 消耗，间接降低模型窃取的攻击面。
  → `sre-agent/backend/app/tools/gateway.py:888-941`（_maybe_eval_fixture_response）

- ✅[真实] adapter mode 控制——通过 ADAPTER_MODE 切换 mock/real，生产环境强制要求 real adapter 配置正确，fail-closed 策略防止调试模式泄露到生产。
  → `sre-agent/backend/app/tools/gateway.py:134-197`

#### 🔴 危险信号（主动规避）

- ❌ 别说"有完整的 API 安全防护" → 没有 IP 限流、没有模型水印、没有对抗样本检测。
- ❌ 别说"做了影子 AI 管控" → 没有流量审计、没有 DLP 拦截。

#### 完整应答（口语稿）

> 模型窃取和影子 AI 不是我们项目的主要关注面，但有几层基础防护值得一提。第一层是工具调用的 timeout + 重试限制——每个工具都有配置的 timeout_ms，通过 asyncio.wait_for 强制执行。重试用 tenacity 的指数退避，最多 2 次。这不能直接防模型窃取，但限制了异常高频调用的放大效应。
>
> 第二层是环境感知的策略隔离。我们的 EnvPolicy 在生产环境限制了可执行动作——只能做 restart、scale_up、scale_down——delete、drop、truncate 等危险操作直接被拒绝。这缩小了攻击者可以利用的攻击面。
>
> 第三层是 adapter mode 控制。评估场景下用 fixture 数据替代真实调用，避免不必要的 API 消耗。生产环境通过 config validation 强制检查 CORS 不为 *、debug 关闭、adapter 不为 mock，fail-fast 启动。但要达到生产级的安全防护，还需要加 IP 限流、查询语义分析、异常调用模式检测这些能力。

---

## 模块二：对齐（Alignment）技术

### Q5　RLHF 全链路（SFT → RM → PPO）（★拉档题）

> 主接项目：sre-agent 的多信号质量打分 + 4 级降级策略。虽然不直接做 RLHF，但"自动评价 + 迭代优化"的工程模式是相通的。

#### 🟢 我命中的参考答案要点

- ✅[真实] 自动质量打分——`compute_quality_from_results()` 基于多证据源收集覆盖率计算 quality_score（成功工具数/总工具数），作为后续决策的输入信号。这类似于 RLHF 中的奖励模型——用可量化的信号评价系统输出的质量。
  → `sre-agent/backend/app/graph/evidence_utils.py:167-170`（compute_quality_from_results）

- ✅[真实] 质量门控循环——critic_node 检查 quality_score，低于 0.4 触发重新收集证据，超过 2 轮循环触发 NEEDS_HUMAN。这类似于 PPO 中的迭代优化——不够好就再来一轮，但不能无限循环。
  → `sre-agent/backend/app/graph/nodes/__init__.py:1115-1148`（critic_node + loop_guard）

- ⚙️[补全] RLHF 三阶段流程——我们的项目不训练模型，所以没有 SFT → RM → PPO 的实现。但从工程角度理解：SFT 教会模型"怎么说"，RM 教会模型"什么更好"，PPO 通过策略优化让模型输出高奖励内容同时用 KL 散度约束防止策略跑偏。

#### 🟡 我能拿到的加分项

- ✅[真实] 加权质量聚合——aggregator node 中将多个 specialist agent 的分析按 quality 加权聚合，类似于 PPO 中按奖励信号加权优化策略的思路。这说明我们理解了"质量信号驱动决策"的核心原则。
  → `sre-agent/backend/app/graph/nodes/__init__.py:897`（_compute_weighted_quality）

- ✅[真实] 4 级降级策略——L0 完整分析 → L1/L2 降级（LLM 失败但有规则兜底）→ L3 LLM_FAILED——体现了"迭代失败时有安全退化路径"的设计思维，类似于 RLHF 中 KL 散度约束防止策略跑偏的思想。
  → `sre-agent/backend/app/graph/nodes/specialist_agent.py:1-5,436-468`

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们做了 RLHF" → 没有训练奖励模型，没有 PPO 优化。但我可以说"我们用了自动质量打分来驱动 agent 的行为优化，这在工程上类似于奖励信号的作用"。
- ❌ 别说"RLHF 就是多标点数据" → 我知道 RLHF 的三阶段各自解决不同问题。

#### 完整应答（口语稿）

> 我们的项目不直接做模型训练，所以没有标准 RLHF 流程的实现。但从工程角度看，我们在 agent 系统中落地了几个类似的思想。
>
> 第一个是自动质量打分。我们的 specialist agent 收集证据后，系统会计算一个 quality_score——基于成功工具的执行比例。多个 agent 的结果在 aggregator 里按这个分数加权聚合。这跟 RLHF 里的奖励模型思路很像——用一个可量化的信号来评价输出质量，驱动后续决策。
>
> 第二个是质量门控循环。我们有个 critic node，检查 quality_score 是否低于 0.4。如果太低，触发重新收集证据——这相当于 PPO 中的"不够好就再优化一轮"。但我们有 loop_guard 兜底——最多 2 轮，超过就触发人工介入。这跟 PPO 的 KL 散度约束是同一个思路：你可以迭代优化，但不能无限跑偏。
>
> RLHF 的核心价值——减少有害输出、提升指令遵循、学会拒绝——在我们系统里是通过多层护栏实现的：工具白名单防越权执行，环境策略防高危操作，退化链路保证永远有安全兜底。虽然实现路径不同，但目标是相通的。

---

### Q6　DPO vs RLHF：原理对比与选型（★拉档题）

> 主接项目：sre-agent 的多信号隐式打分 + harness-system 的硬约束策略。在工程层面实际选择了类似"隐式奖励 + 硬约束"的混合模式。

#### 🟢 我命中的参考答案要点

- ⚙️[补全] DPO 和 RLHF 的选型——项目不训练模型，所以没有直接使用 DPO 或 RLHF。但从设计角度理解：DPO 把 RLHF 的奖励模型隐含到偏好对中，用分类损失替代策略梯度，流程从三阶段简化为两阶段。优势是简单稳定，劣势是依赖高质量偏好对且不能在线迭代。

- ✅[真实] 工程层面的"隐式奖励"模式——sre-agent 的证据质量打分不显式训练奖励模型，而是直接从多个信号（工具执行成功率、证据覆盖率）计算质量分数。这类似于 DPO"不需要显式奖励模型"的思想——把评价直接隐含在打分逻辑中。
  → `sre-agent/backend/app/graph/evidence_utils.py:167-170`

- ✅[真实] 工程层面的"硬约束"模式——harness-system 的 spec validation 规则和 tool risk_level 是硬编码在系统中的，不依赖 LLM 自觉。这类似于"护栏 + 对齐"的组合：DPO/RLHF 负责让模型"想做好"，硬约束负责让模型"不能做坏"。

#### 🟡 我能拿到的加分项

- ✅[真实] 多种信号加权聚合 vs 单一奖励模型——我们用 quality_score + evidence_coverage + confidence 多个信号做决策，而不是依赖单一奖励模型。这可以类比为"多目标对齐"——不追求单一指标最优，而是多个维度上的平衡。
- ✅[真实] 规则兜底 vs 纯模型对齐——L0-L3 降级策略中，LLM 失败时回退到规则引擎。这体现了"不对齐时该怎么兜底"的工程思维——当模型输出不可靠时，确定性规则接管。

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们选择了 DPO 而不是 RLHF" → 我们没有训练过模型，选型是理论层面的讨论。
- ❌ 别说"DPO 完全替代 RLHF" → 我知道 DPO 的两个核心限制：依赖高质量偏好对、不能做在线迭代。

#### 完整应答（口语稿）

> 我们的项目不训练模型，所以 DPO vs RLHF 的选型我更多是从技术理解和工程类比角度来谈。
>
> 先讲原理。DPO 的核心创新是把 RLHF 的奖励模型给它"隐式化"了——不需要显式训练一个 RM，直接用偏好对里的 chosen/rejected 关系做分类损失。这样流程就从 SFT → RM → PPO 三段变成 SFT → DPO 两段，训练更稳定，LoRA + DPO 是 2026 新项目标配。但 DPO 有两个限制：一是偏好数据噪声大的时候效果很差，二是它不支持在线迭代——你不能像 RLHF 那样持续收集反馈、更新奖励模型。
>
> 从工程角度看，我们的系统虽然没有 DPO，但做了一个类似于"隐式奖励"的设计。证据质量打分不是训练出来的奖励模型，而是直接从多个信号——工具执行成功率、证据覆盖率——计算出来。这就跟 DPO 的思路有点像：不显式建奖励模型，而是把评价逻辑隐含在系统设计中。
>
> 另外一个关键点是"对齐不能只靠模型自觉"。harness-system 里我们做了两层策略——AGENTS.md 里的行为准则是对齐的软约束，但真正不让你越权执行的是代码层的硬约束：工具风险分级、审批链、环境策略。这跟业界的趋势一致：实际落地都是 DPO + 护栏的组合，用 DPO 让模型"想做对"，用护栏让模型"不能做错"。

---

### Q7　Constitutional AI（宪法 AI）+ RLAIF 与对齐技术演进

> 主接项目：harness-system 的 AGENTS.md Constitution Gates + spec validation rules。嵌入式宪法的工程实践。

#### 🟢 我命中的参考答案要点

- ✅[真实] Constitution Gates——AGENTS.md 中定义了 9 步强制流程，其中 step 2 Planning 阶段有明确的宪法门控清单：需求可验证性、范围澄清、安全考量等 6 项 check。Agent 在规划阶段必须自我审查清单，通过后才进入执行。这是 Constitutional AI 的工程化落地——用 checklist 而非训练的方式让 agent 自我修正。
  → `harness-system/AGENTS.md:24-31`（Constitution Gates checklist）

- ✅[真实] Spec validation rules 作为代码化的宪法——validator-rules.yaml 定义了 13 条规则（block/warn/info 三级），检查 spec 的完整性、风险等级、安全考量。这些规则是固定的、可审计的、不可绕过的——相当于"嵌入在系统中的宪法原则"。
  → `harness-system/specs/validator-rules.yaml:1-135`（13 条规则）、`platform/src/lib/validator.ts:169`（validateSpec 入口）

- ⚙️[补全] RLAIF（用 AI 做奖励模型）——项目中用 quality_score 做自动评价，但没有用 AI 生成训练数据进行对齐。如果要做 RLAIF，可以利用已有的 eval framework + quality scoring 作为 AI 奖励信号的基础设施。

#### 🟡 我能拿到的加分项

- ✅[真实] 双策略载体——AGENTS.md（行为侧软约束）+ opencode.json plugin hooks（系统侧硬约束）。这个设计原则是"策略不能只写在 prompt 里"——模型可能忘记 prompt 里的规则，但插件钩子的拦截是 100% 执行的。
  → `harness-system/docs/superpowers/specs/2026-06-19-harness-prd.md:429-431`

- ✅[真实] 宪法原则的审计闭环——每次 spec 提交都经过 validator 校验，校验失败阻止创建。每次 agent 执行都记录 run_records。如果违反宪法原则，事后可追溯、可定责。这是 CAI 的核心要求——原则不是写写而已，要有执行和审计机制。

#### 🔴 危险信号（主动规避）

- ❌ 别说"Constitutional AI 就是写一套 prompt 规则" → 我知道 CAI 的核心是让模型根据原则自我修正 + 用 AI 反馈替代人类反馈。我们的 Constitution Gates 是固定 checklist，没有 AI 自我修正循环。但我会说"这是 CAI 思想在工程约束条件下的务实落地"。

#### 完整应答（口语稿）

> Constitutional AI 的核心思想是让模型根据一套原则自我修正。我们在 harness-system 里做了一个很有意思的落地——AGENTS.md 里的 Constitution Gates。
>
> 每个 agent 在执行任务前必须走 Planning 步骤，这个步骤里有一个 6 项的宪法门控清单：需求是不是可验证的、范围是不是澄清了、有没有安全考量、有没有明确的完成标准。Agent 在规划阶段必须逐项自查，通过后才进入执行。没通过就暂停、请求人类澄清。这跟 Anthropic 的 CAI 思路一样——不是靠训练让模型对齐，而是靠规则让模型在执行边界内行动。
>
> 更重要的是，这套规则不是只写在 AGENTS.md 里——那只是软约束，模型可能忘。我们还做了代码层的硬约束：validator-rules.yaml 里定义了 13 条可编程规则，每次 spec 提交都会被这些规则校验，不符合的直接 block。这就实现了"双策略载体"——行为侧写宪法原则，系统侧写硬性校验。
>
> 跟 RLAIF 的关系是这样的：RLAIF 用 AI 做奖励模型，可以大规模自动化对齐。我们的 quality_score 自动打分其实就是一个雏形——用多信号自动评价 agent 输出质量。如果要做 RLAIF，这套基础设施可以直接复用。

---

### Q8　红队测试方法论与评估

> 主接项目：sre-agent 的 SQL 注入防御测试 + security incident eval case。有基础的安全测试实践，但缺少系统性红队框架。

#### 🟢 我命中的参考答案要点

- ✅[真实] SQL 注入防御测试——`test_query_logs_from_db_sql_injection_safe` 用例输入 `"; DROP TABLE--"` 验证查询不执行注入，断言 SQL 中不含 DROP TABLE。
  → `sre-agent/backend/app/tests/test_mysql_adapter.py:191-211`

- ✅[真实] 安全事件评估用例——`case_08_security_incident.json` 测试凭证填充场景，验证系统输出 `incident_type: "security_incident"` 和 `risk_decision: "NEEDS_APPROVAL"`。
  → `sre-agent/backend/app/evals/datasets/case_08_security_incident.json`

- ✅[真实] 评估用例 schema 校验——`_validate_case()` 对所有 11 个 eval case 做结构校验，确保测试数据质量。
  → `sre-agent/backend/app/evals/case_loader.py:56`

- ⚙️[补全] 系统性红队框架——目前只有 1 个安全 eval case 和 1 个 SQL 注入测试，没有系统化的红队测试套件、没有对抗性 prompt 数据集、没有自动红队（LLM-as-Attacker）。如果要做，会用 SafetyBench + 自建攻击 prompt 库 + LLM-as-Attacker 自动生成攻击样本。

#### 🟡 我能拿到的加分项

- ✅[真实] harness-system 的 bad_cases 表——记录了失败案例及其 root_cause、severity，可以作为红队测试的"战果库"，每次系统变更后回归跑 bad cases 验证不退化。
  → `harness-system/platform/src/lib/db.ts:114-141`（bad_cases 表）

- ✅[真实] spec_versions 表——记录了每次 spec 变更的 snapshot，可以对比不同版本的 spec 在安全维度上的变化，相当于"安全回归测试"的基础设施。

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们有系统化的红队测试" → 只有 1 个安全 case 和 1 个注入测试。但我会说"我们有基础测试用例和评估框架，下一步计划引入 SafetyBench + 自动红队"。
- ❌ 别说"安全测试就是跑跑功能测试" → 我知道红队测试需要专门的攻击方法论和评估指标（攻击成功率、漏洞严重等级、回归通过率）。

#### 完整应答（口语稿）

> 我们项目里安全测试目前是起步阶段，但有实用的基础。做了两个具体的测试：一个是 SQL 注入防御测试——输入 `"; DROP TABLE--"` 这样的经典注入字符串，验证查询层不会直接拼接执行；另一个是安全事件评估用例——一个凭证填充的场景，验证系统能正确识别为 security_incident 并把风险决策标为 NEEDS_APPROVAL。
>
> 评估框架层面，我们有 eval case loader 对 11 个评估用例做 schema 校验，保证测试数据质量。harness-system 有 bad_cases 表记录所有失败案例和根因分析，这可以作为红队测试的"战果库"，每次系统变更后回归跑历史 bad cases。
>
> 但要达到生产级的红队测试能力，还需要三件事：一是引入 SafetyBench 这样的标准基准做持续评估；二是建立对抗性 prompt 数据集，覆盖直接注入、间接注入、越狱等多种攻击模式；三是用 LLM-as-Attacker 的方式做自动红队——用一个专用模型生成攻击 prompt，批量测试系统的安全边界。这三步是目前我们在设计阶段规划但还没实现的。

---

## 模块三：护栏（Guardrails）工程

### Q9　护栏四层架构设计（L1-L4）（★拉档题）

> 主接项目：sre-agent 的四层护栏实际实现。ToolGateway → risk_gate + loop_guard → 输出脱敏 + audit → HITL 审批。这是三个项目中护栏实现最完整的。

#### 🟢 我命中的参考答案要点

- ✅[真实] **L1 输入护栏**：ToolGateway 的参数校验 + EnvPolicy 环境白名单。
  - `_validate_tool_params()` 校验所有必填参数存在且类型匹配 → `sre-agent/backend/app/tools/gateway.py:862-885`
  - `EnvPolicy.can_execute_action()` 生产环境禁止 delete/drop/truncate → `sre-agent/backend/app/tools/policies/env.py:5-53`
  - `_tool_allowed()` 校验 LLM 请求的工具是否在 agent 白名单中 → `sre-agent/backend/app/graph/nodes/specialist_agent.py:297-311`

- ✅[真实] **L2 推理护栏**：risk_gate_node + loop_guard + quality gating。
  - risk_gate_node 做 4 路决策：LOW_ONLY / NEEDS_APPROVAL / BLOCKED / NEEDS_HUMAN → `sre-agent/backend/app/graph/nodes/__init__.py:1249-1363`
  - loop_guard 限制证据收集最多 2 轮 → `:1115-1148`
  - quality_score < 0.4 触发 critic 重试 → `:1164-1167`

- ✅[真实] **L3 输出护栏**：敏感信息脱敏 + 置信度封顶 + 审计日志脱敏。
  - ConfigMap 敏感值脱敏（password/secret/token/api_key → "***REDACTED***"）→ `sre-agent/backend/app/tools/adapters/k8s_adapter.py:646-658`
  - 置信度强制封顶 ≤ 0.9 → `sre-agent/backend/app/graph/nodes/specialist_agent.py:395`
  - 审计日志递归脱敏（password/secret/token/api_key 等 7 个 key）→ `sre-agent/backend/app/tools/gateway.py:840-856`

- ✅[真实] **L4 业务护栏**：HITL 审批 + 分级响应。
  - approval_interrupt_node 创建 ApprovalRequest（含 risk_level、expected_impact、rollback_plan）→ `sre-agent/backend/app/graph/nodes/__init__.py:1374-1515`
  - 持久化到 DB，通过 `/approvals/{id}/decision` 端点人工审批
  - Module 4 degradation: degrade_on_failure = True → `sre-agent/backend/app/models/planning.py:38-39`

#### 🟡 我能拿到的加分项

- ✅[真实] harness-system 的七层架构设计提供了另一套护栏视角——从 Spec 层开始，每一层都是一个 gate。L0-L3 工具风险分级是护栏策略的数据基础，每个工具声明 risk_level + approval_required + timeout_ms。
  → `harness-system/tools/registry/index.yaml:1-142`

- ✅[真实] 两层脱敏覆盖——L1（工具参数）和 L3（审计日志）各有一套 SENSITIVE_KEYS，覆盖不同阶段的数据生命周期。工具层的脱敏防止敏感数据泄露到 LLM 上下文，审计层的脱敏防止敏感数据写入持久化日志。

- ✅[真实] 熔断机制（Circuit Breaker）——loop_guard 限制最大 2 轮证据收集，超过触发 NEEDS_HUMAN。ControlledExecutor 的 idempotency_key 检查防止重复执行。
  → `sre-agent/backend/app/services/executor.py:62-80`

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们有语义安全分类" → L1 目前用规则+白名单，没有小模型做安全分类。
- ❌ 别说"输出内容过滤做了" → 输出层目前脱敏敏感字段，不做毒性/幻觉内容检测。

#### 完整应答（口语稿）

> 我在 sre-agent 项目里实际落地了四层护栏架构，每一层都有对应的代码实现。
>
> L1 输入护栏有三道检查：第一道是 ToolGateway 的参数校验——所有必填参数的存在性和类型都在 _validate_tool_params 里检查。第二道是工具白名单——每个 LLM 返回的 tool_call 必须在 agent 的 tool_names 里，不在的注入错误消息让 LLM 修正。第三道是环境策略——生产环境禁止 delete、drop、truncate 操作，只允许 restart、scale_up、scale_down。
>
> L2 推理护栏的核心是 risk_gate_node。它根据 action risk_level、环境、告警严重度、诊断置信度做 4 路路由：全低风险直接放行，需要审批的进入审批流程，高危低置信的 BLOCKED 直接拒绝，复杂场景触发 NEEDS_HUMAN。还有 loop_guard 限制证据收集最多 2 轮，超过就熔断。
>
> L3 输出护栏有三层脱敏：工具层的 k8s ConfigMap 会把 password、secret、token 等敏感值替换成 REDACTED；置信度被强制封顶在 0.9——即使 LLM 说 100% 确定也只给 0.9；审计日志里的敏感字段用 _sanitize_for_audit 递归替换。这保证了敏感数据不会泄露到输出里，也不会残留在日志中。
>
> L4 业务护栏是人工审批体系。当风险决策是 NEEDS_APPROVAL 时，系统生成 ApprovalRequest——包含风险等级、预期影响、回滚方案——持久化到数据库，等人工审批通过后才继续执行。整套护栏的告警分三级：P0 数据泄露、P1 注入攻击成功、P2 误杀率突增。

---

### Q10　护栏关键设计决策

> 主接项目：sre-agent 的工程决策——白名单 vs 黑名单、输入校验用规则还是模型、多 Agent 用统一护栏。

#### 🟢 我命中的参考答案要点

- ✅[真实] 工具授权用白名单而非黑名单——agent 的 tool_names 是显式枚举的，不在列表里的工具一律拒绝。这是零信任原则的体现：默认拒绝，显式授权。
  → `sre-agent/backend/app/graph/nodes/specialist_agent.py:297-311`（_tool_allowed）

- ✅[真实] 输入校验用规则（低延迟）+ 环境策略硬编码——ToolGateway 的参数校验是同步规则匹配，毫秒级完成。EnvPolicy 的 allowlist 和 restricted_actions 是代码级别的硬约束，不会被绕过。
  → `sre-agent/backend/app/tools/gateway.py:862-885`、`env.py:5-53`

- ✅[真实] 告警策略用组合：固定阈值 + 行为检测——risk_gate_node 有固定条件（产线 + execute_action + 低置信 → BLOCKED），loop_guard 有行为检测（同一流程中超过 2 轮证据收集 → NEEDS_HUMAN）。
  → `sre-agent/backend/app/graph/nodes/__init__.py:1335-1344,1127`

- ✅[真实] 多 Agent 用统一护栏 + trace_id 串联——所有 agent 通过同一个 graph state 传递数据，trace_id（run_id）贯穿整个调用链，Tracer 记录所有 span，确保 A2A 通信可追踪、可审计。
  → `sre-agent/backend/app/tracing.py:47`（start_span 带 run_id）、`gateway.py:1084-1132`（audit log 带 run_id）

#### 🟡 我能拿到的加分项

- ✅[真实] 工具注册时声明风险元数据——risk_level + requires_approval + timeout_ms 在注册时就声明，护栏运行时直接从元数据读取策略，不需要每次调用都判断。这降低了护栏的推理开销，也保证了策略的一致性。
- ✅[真实] fail-closed 策略——adapter 在真实模式下如果没配置，抛出 RuntimeError 而不是静默降级到 mock。生产环境 check CORS 不为 *、debug 关闭、adapter mode 不为 mock，任何一项不满足直接 ValueError 阻止启动。
  → `sre-agent/backend/app/core/config.py:117-158`（validate_for_production）

#### 🔴 危险信号（主动规避）

- ❌ 别说"用 GPT-4 做安全判断" → 我们的输入护栏是规则驱动的（低延迟），如果加语义判断会用 7B 级别的小模型，不会用 GPT-4。
- ❌ 别说"每个 agent 自己管自己的安全" → 我们是统一护栏 + trace_id 串联，Agent A 的输出是 Agent B 的结构化输入，共享护栏逻辑。

#### 完整应答（口语稿）

> 护栏设计有几个关键的工程决策。第一个是授权模型选白名单还是黑名单——我们选了白名单。每个 agent 的 tool_names 是显式枚举的，不在列表里的一律拒绝。这是零信任原则，默认拒绝、显式授权。而且工具注册时声明了 risk_level、requires_approval、timeout_ms，护栏运行时直接从元数据读取策略。
>
> 第二个是输入校验用什么。我们用规则引擎——ToolGateway 的参数校验是同步的、毫秒级完成；EnvPolicy 的 allowlist 是硬编码的、不会被 prompt 覆盖。如果以后要加语义判断，会用 7B 级别的小模型，不用大模型，因为输入侧的延迟要求很高。
>
> 第三个是告警策略。我们用固定阈值加行为检测的组合。固定条件比如产线 + execute_action + 低置信直接 BLOCKED。行为检测比如 loop_guard 检测同一流程中超过 2 轮证据收集就熔断。不是简单的"异常了就告警"，而是有具体触发条件。
>
> 第四个是多 Agent 护栏。所有 agent 共享同一套护栏逻辑，通过 run_id 串联所有工具调用和 LLM 交互。Agent 之间不共享自然语言上下文，只传结构化数据，降低了 A2A 注入传播的风险。

---

### Q11　幻觉治理全链路

> 主接项目：sre-agent 的多证据源质量门控 + 置信度封顶 + 规则兜底。检索→生成→验证→闭环 四层各有落地。

#### 🟢 我命中的参考答案要点

- ✅[真实] **检索层**：多证据源加权打分——aggregator 将多个 specialist agent 的分析结果按 quality_score 加权聚合，低质量证据直接降权。这是从源头控制幻觉——不采纳不可靠的信息源。
  → `sre-agent/backend/app/graph/nodes/__init__.py:897`（_compute_weighted_quality）、`evidence_utils.py:167-170`（compute_quality_from_results）

- ✅[真实] **生成层**：置信度强制封顶——LLM 返回的 confidence 被 `min(llm_confidence_raw, 0.9)` 封顶，系统层面限制 overconfidence。这是生成侧的一道硬约束——即使模型说"我 100% 确定"，系统也只给 0.9。
  → `sre-agent/backend/app/graph/nodes/specialist_agent.py:395`

- ✅[真实] **验证层**：critic_node 质量门控——quality_score < 0.4 时触发重新收集证据，相当于系统层面的"事实核查"——你不确定就再查一遍。
  → `sre-agent/backend/app/graph/nodes/__init__.py:1164-1167`

- ✅[真实] **闭环**：4 级降级策略——L0 完整分析 → L1/L2 降级（LLM 失败但有规则兜底）→ L3 LLM_FAILED。规则引擎是确定性的，不会产生幻觉。这是闭环的最底层保障。
  → `sre-agent/backend/app/graph/nodes/specialist_agent.py:1-5,436-468`

- ⚙️[补全] 引用归属检查——目前没有验证 LLM 输出的引用是否真的来自检索结果。rag 项目虽然记录了 rag_sources，但没有做引用准确率验证。

#### 🟡 我能拿到的加分项

- ✅[真实] 多证据交叉验证——不同 specialist agent（k8s、db、log、metrics）从不同数据源收集证据，aggregator 只有在多个源一致的结论上才给高置信。相当于"多源事实核查"——单一来源的结论不被信任。
- ✅[真实] rag 项目的 RAG hit rate 分析——通过 SQL 查询统计检索命中率，虽然不直接在生成时验证，但为幻觉治理提供了可量化的指标基础。
  → `rag/server/src/modules/knowledge/knowledge.service.ts:325-338`

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们做了完整的事实核查" → 没有专门的事实核查模型，目前靠多源打分和置信封顶。但我会说"在当前架构下，多源交叉验证 + 置信封顶 + 规则兜底已经覆盖了幻觉治理的核心链路"。

#### 完整应答（口语稿）

> 幻觉治理需要贯穿整个推理链路，我们在四个层级都有对应设计。
>
> 检索层做多证据源加权打分。我们有 k8s、db、log、metrics 四个 specialist agent 各自收集证据。aggregator 会根据每个 agent 的质量分数加权聚合——工具执行成功率高的 agent 权重高，低的直接降权。这从源头减少了幻觉——不采纳不可靠的信息源。而且多源交叉验证——只有多个源一致指向的结论才会被采信。
>
> 生成层做置信度封顶。无论 LLM 返回的 confidence 是多少，系统强制封顶在 0.9。这是系统层面的硬约束，不依赖模型自觉。因为在真实场景中，没有 100% 确定的事。
>
> 验证层用 critic node 做质量门控。如果 quality_score 低于 0.4，触发重新收集证据，相当于系统在说"你给我的证据不够好，再去查一遍"。最多查 2 轮，超过就触发人工介入。
>
> 闭环层是降级策略。LLM 分析失败了怎么办？我们有规则引擎兜底——按 category 从工具执行结果中提取异常信号。k8s 提取 pod 重启和 OOM，db 提取慢查询和连接池耗尽。这些是确定性规则，不会产生幻觉。这是我们幻觉治理的最后一道防线。

---

### Q12　多 Agent 护栏与 Agent 互注入防御

> 主接项目：sre-agent 的 Agent 工具隔离 + trace_id 串联 + Graph state 结构化传递。实际的 Agent 互注入防御工程。

#### 🟢 我命中的参考答案要点

- ✅[真实] Agent 工具隔离——每个 specialist agent 通过 CATEGORY_PREFIX_MAP 限制工具调用范围。k8s agent 只能调 `query_k8s_*`，db agent 只能调 `query_db_*`。泄露一个 agent 不会导致全系统沦陷。
  → `sre-agent/backend/app/graph/nodes/specialist_agent.py:35-41`（CATEGORY_PREFIX_MAP）

- ✅[真实] FORBIDDEN_TOOLS——所有 specialist agent 都禁止调用 execute_action。即使某个 agent 被注入"请删除所有 pod"的指令，它没有执行权限——执行操作只能由专门的 ControlledExecutor 在严格条件下进行。
  → `sre-agent/backend/app/graph/nodes/specialist_agent.py:33`

- ✅[真实] Agent 间通过 graph state 传递结构化数据——agent 的输出是 `SpecialistAnalysis` 结构化对象（含 run_status、confidence、evidence），不传递自然语言文本。这天然降低了 A2A 注入传播面。
  → `sre-agent/backend/app/graph/nodes/specialist_agent.py:174-180`（run 方法构建结构化输出）

- ✅[真实] trace_id 串联 A2A 通信——所有工具调用和 LLM 交互通过 run_id 串联，AgentTracer 记录每个 span，支持跨 agent 追踪和安全审计。
  → `sre-agent/backend/app/tracing.py:47,116-131`（start_span + get_trace_metadata）、`gateway.py:1084-1132`（audit log 带 run_id）

- ⚙️[补全] Agent 认证机制——目前 agent 之间通过共享 graph state 通信，没有 agent 级别的身份认证。如果引入第三方 agent，需要加 agent key/签名验证。

#### 🟡 我能拿到的加分项

- ✅[真实] 最小权限原则在 Agent 层的落地——每个 agent 只授予完成其职责所需的最少工具权限。k8s agent 不能查数据库，db agent 不能操作 k8s。这不是"建议"而是代码级别的硬约束。
- ✅[真实] harness-system 的 HITL 审批作为"最危险的 Agent 互注入防御"——当风险决策为 NEEDS_APPROVAL 时，所有操作暂停，有审批权限的人类必须介入。这是 Agent 互注入的终极兜底——人类永远是最后一道防线。

#### 🔴 危险信号（主动规避）

- ❌ 别说"Agent 之间共享上下文是安全的" → 我知道自由文本传递是注入传播的主要途径，所以我们的 Agent 只传递结构化对象。
- ❌ 别说"每个 Agent 自己管安全" → 我们是统一护栏 + 全局 FORBIDDEN_TOOLS。

#### 完整应答（口语稿）

> 多 Agent 系统中的互注入是一个很现实的威胁——Agent A 的输出是 Agent B 的输入，如果中间不设隔离，一个 Agent 被注入后整条链路都会被污染。
>
> 我们在 sre-agent 里做了三层防御。第一层是工具权限隔离。每个 specialist agent 有 category 级别的工具限制——k8s agent 只能调 query_k8s_* 前缀的工具，db agent 只能调 query_db_*。而且所有 specialist agent 有一组 FORBIDDEN_TOOLS，里面包含 execute_action。这意味着即使一个 agent 被完全攻陷，它也只能做查询，不能做任何写操作。
>
> 第二层是数据格式隔离。Agent 之间不传递自然语言文本，只传结构化对象。比如 k8s agent 的分析结果是 SpecialistAnalysis 对象，包含 confidence、run_status、evidence 这些结构化字段。这天然降低了注入传播面——攻击者不能通过自然语言在 Agent 之间传播恶意指令。
>
> 第三层是 trace_id 串联。所有工具调用和 LLM 交互通过 run_id 串联起来，Tracer 记录每个 span。如果发生安全事件，可以完整回溯：这个恶意指令是怎么从 Agent A 传到 Agent B 的，中间经过了哪些调用。加上 harness-system 的 HITL 审批机制——高危操作必须人类确认——人类是 Agent 互注入防御的最后一道防线。

---

## 模块四：隐私保护与合规

### Q13　隐私保护技术全景（DP/FL/HE/MPC/水印）

> 主接项目：sre-agent 的敏感数据脱敏 + bcrypt 密码哈希 + MySQL 只读模式。虽不直接涉及 DP/FL，但体现了"数据最小化"和"传输脱敏"的隐私工程原则。

#### 🟢 我命中的参考答案要点

- ✅[真实] 工具层敏感数据脱敏——K8s ConfigMap 查询中，password、secret、token、api_key 等 9 个敏感键的值被替换为 "***REDACTED***"。这是"数据传输最小化"的实践——即使内部调用也不暴露明文敏感信息。
  → `sre-agent/backend/app/tools/adapters/k8s_adapter.py:630-673`（SENSITIVE_KEYS + redact）

- ✅[真实] 审计日志递归脱敏——_sanitize_for_audit() 递归处理 dict 中的敏感字段（password、secret、token、api_key、access_key、authorization、cookie），写入 DB 前脱敏。满足"去标识化"要求。
  → `sre-agent/backend/app/tools/gateway.py:840-856`

- ✅[真实] 密码安全存储——使用 bcrypt + saltRounds=10 做密码哈希，API 响应中排除 password_hash 字段。
  → `rag/server/src/modules/user/user.service.ts:69-70`、`user.controller.ts:13`

- ✅[真实] MySQL 只读模式默认——`mysql_readonly: bool = True`，数据访问默认只读，写操作需要显式配置。
  → `sre-agent/backend/app/core/config.py:64`

- ⚙️[补全] 差分隐私/联邦学习/同态加密——项目不涉及模型训练（无 DP/FL），不做高敏感推理（无 HE）。但从架构角度理解各自的适用场景和 tradeoff。模型水印目前也只停留在概念层面，没有在系统中部署。

#### 🟡 我能拿到的加分项

- ✅[真实] 生产环境配置强制校验——validate_for_production() 检查 CORS 不为 *、debug 关闭、adapter 不为 mock。这是合规要求中的"安全基线"在工程层面的自动化执行，不在代码里靠人记，而是启动时就强制校验。
- ✅[真实] 两层脱敏覆盖数据全生命周期——工具参数脱敏（运行时）+ 审计日志脱敏（持久化），确保敏感数据在传输和存储两个阶段都不泄露。

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们用了差分隐私" → 没有做 DP/FL/HE。但我会说"在不需要模型训练的 Agent 系统中，数据最小化、传输脱敏、只读默认这三项比 DP/FL 更直接有效"。
- ❌ 别说"ε 取 1-10 看情况"如果被追问具体选择 → 我会诚实说"没有调过 ε，但知道理论范围是 1-10，越小越安全模型越差"。

#### 完整应答（口语稿）

> 我们的项目不做模型训练和高敏感推理，所以差分隐私、联邦学习、同态加密这些直接使用的不多。但我们在隐私保护的工程实践上有几个扎实的设计。
>
> 第一个是敏感数据脱敏覆盖全链路。工具调用层——查询 K8s ConfigMap 时，password、secret、token、api_key 等 9 个敏感键的值直接替换成 REDACTED，LLM 拿到的全是脱敏数据。审计持久化层——_sanitize_for_audit 在写入 DB 前递归脱敏 7 类敏感字段。这意味着即使有人翻审计日志，也看不到明文密码。
>
> 第二个是默认安全的设计。MySQL 连接默认只读，生产环境 CORS 不允许 * 通配符，debug 不允许开启，adapter 不允许 mock。这些通过 validate_for_production 在启动时就强制校验，不满足直接 ValueError 拒绝启动。
>
> 第三个是用户密码的 bcrypt 哈希存储——saltRounds=10 是目前的标准实践。虽然这些不是 DP/FL 这种高级技术，但在我看来，对于 Agent 系统来说，数据最小化、传输脱敏、只读默认这三项比 DP 更直接有效——你把不需要的数据不收集、不传输、不存储，从根本上降低了隐私风险。

---

### Q14　中国法规合规（个信法/网安法/生成式 AI 办法）

> 主接项目：sre-agent 的脱敏 + config validation + 日志审计。合规要求在工程中的落地。

#### 🟢 我命中的参考答案要点

- ✅[真实] 数据去标识化——《个信法》要求。审计日志脱敏（_sanitize_for_audit）将 password、secret、token 等敏感字段替换为 "***REDACTED***" 后再持久化；K8s ConfigMap 工具调用也脱敏敏感值。
  → `sre-agent/backend/app/tools/gateway.py:840-856`、`k8s_adapter.py:646-658`

- ✅[真实] 数据最小化——《网安法》要求。MySQL 只读模式默认开启，生产环境禁止危险操作（delete/drop/truncate），最小化数据访问和操作权限。
  → `sre-agent/backend/app/core/config.py:64`、`env.py:5-53`

- ✅[真实] 安全基线自动化——《生成式 AI 办法》对系统的安全要求。validate_for_production() 在启动时强制检查安全配置，CORS 不为 *、debug 关闭、API key 必须配置。不满足不启动。
  → `sre-agent/backend/app/core/config.py:117-158`

- ✅[真实] 审计可追溯——《网安法》要求日志留存。所有工具调用记录到 IncidentToolAudit 表，包含 run_id、tool_name、params、result、latency_ms。可通过 run_id 完整回溯一次请求的全链路。
  → `sre-agent/backend/app/tools/gateway.py:1084-1132`、`models/db_models.py:164-176`

- ⚙️[补全] 生成内容水印——目前没有对 AI 生成的内容加水印。如果要满足《生成式 AI 管理办法》的标识义务，会在输出中添加隐式水印标记。

#### 🟡 我能拿到的加分项

- ✅[真实] API key 多层安全——LLMClient 的 API key 通过 constructor param → Settings 对象 → os.getenv() 三层解析，不硬编码。.env 文件 gitignored，K8s secret.yaml 独立于 configmap。
  → `sre-agent/backend/app/llm_client.py:29-58`

- ✅[真实] harness-system 的审批链合规——不同风险等级对应不同审批角色：low 只需 tech，medium 需要 tech+qa，high 加 security，critical 再加 ops。这直接映射了合规中的"权限分级"要求。
  → `harness-system/platform/src/lib/approvals.ts:4-9`

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们完全合规" → 合规是持续过程，我能保证的是做了安全基线自动化、数据脱敏、审计追溯这三项核心要求。
- ❌ 别说"生成内容水印做了" → 没有部署。但我知道《生成式 AI 管理办法》有这个要求，在设计层面有考虑。

#### 完整应答（口语稿）

> 三法合规在工程层面，我们主要落了四件事。
>
> 第一件是数据去标识化——满足个信法的要求。审计日志在持久化前对敏感字段做脱敏，7 类字段全替换成 REDACTED。K8s ConfigMap 查询时也对敏感值脱敏。这意味着敏感数据不出现在日志中，满足去标识化的要求。
>
> 第二件是数据最小化——满足网安法的要求。MySQL 默认只读，生产环境禁止 delete、drop、truncate 等高危操作。EnvPolicy 在白名单里的才允许执行。这体现了"用不到的数据和权限就不给"的原则。
>
> 第三件是安全基线自动化——满足生成式 AI 管理办法对系统安全的要求。系统启动时 validate_for_production 强制检查：CORS 不能是 *，debug 不能开启，adapter 不能是 mock，API key 必须配置。任何一项不满足，启动直接报错。这确保了安全配置不是靠人记住，而是靠代码强制执行。
>
> 第四件是审计可追溯。IncidentToolAudit 表记录了每次工具调用的完整信息——谁调了什么工具、用什么参数、花了多长时间、结果是成功还是失败。通过 run_id 可以回溯一次请求的全链路。加上 API key 的三层解析——不硬编码、.env gitignored、K8s secret 独立管理——密钥管理的安全基线也是到位的。

---

## 模块五：评测体系

### Q15　安全评测基准（SafetyBench / TrustGPT / ToxiGen 等）

> 主接项目：sre-agent 的安全评估用例 + harness-system 的 spec validation。自建 mini-benchmark 的思路。

#### 🟢 我命中的参考答案要点

- ✅[真实] 自建安全评估用例——`case_08_security_incident.json` 测试凭证填充场景，验证系统输出 incident_type: "security_incident" 和 risk_decision: "NEEDS_APPROVAL"。本质上是自建的 minibench——用结构化用例验证安全行为。
  → `sre-agent/backend/app/evals/datasets/case_08_security_incident.json`

- ✅[真实] SQL 注入防御测试——验证 `"; DROP TABLE--"` 不执行注入，覆盖了安全基准中的一个关键维度。
  → `sre-agent/backend/app/tests/test_mysql_adapter.py:191-211`

- ✅[真实] 评估用例 schema 校验——_validate_case() 对所有 eval case 做结构完整性校验，保证测试数据质量。好的基准首先要有干净的测试数据。
  → `sre-agent/backend/app/evals/case_loader.py:56`

- ⚙️[补全] SafetyBench/TrustGPT/ToxiGen 等标准基准——项目中没有集成这些标准基准。但架构设计中有 Safety Eval 的规划，作为 P1 评估维度。
  → `harness-system/docs/superpowers/specs/2026-06-19-harness-prd.md:922-931`

#### 🟡 我能拿到的加分项

- ✅[真实] 多维度评估覆盖——sre-agent 的 11 个 eval case 覆盖了不同类型的安全场景：常规故障、安全事件、多证据交叉验证。虽然规模不大，但体现了"安全需要多维评测"的意识。
- ✅[真实] harness-system 的 bad_cases 表——失败案例有 root_cause 和 severity，可以持续积累评测数据。这跟 SafetyBench 的思路一样——用真实案例库做持续评测。

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们用了 SafetyBench" → 没有集成标准基准。但我会说"我们有自建的安全评估用例，结构完整可扩展，下一阶段计划对标 SafetyBench 的设计加入更多安全维度"。

#### 完整应答（口语稿）

> 安全评测这块，我们没有直接用 SafetyBench 这些标准基准，但做了自己的安全评估用例。sre-agent 有一个 security incident eval case——测试凭证填充场景，验证系统能不能正确识别为安全事件并把风险路由到 NEEDS_APPROVAL。还有一个 SQL 注入防御测试——输入经典注入字符串验证查询层不会直接拼接执行。
>
> 评估框架的基础设施是到位的——eval case loader 有 schema 校验，11 个 eval case 覆盖了不同维度的场景。harness-system 还有 bad_cases 表，记录所有失败案例的根因和严重程度。
>
> 这些自建的评估用例，本质上就是一个小型的 minibench。规模不如 SafetyBench 那么大，但维度是清楚的——安全性、鲁棒性、行为对齐都在测。下一阶段会把 SafetyBench 和 TrustGPT 的标准维度映射到我们的用例框架中，扩大覆盖面和自动化程度。

---

### Q16　对齐评测方法（LLM-as-Judge / 人类评估 / Elo / A/B Test）

> 主接项目：sre-agent 的 quality scoring + human approval 信号。多信号自动评估 + 隐式人类反馈。

#### 🟢 我命中的参考答案要点

- ✅[真实] 多信号自动评估——evidence quality scoring 用工具执行成功率自动评价 agent 输出质量。这本质上是一种 LLM-as-Judge 的变体——不是让另一个 LLM 打分，而是用可量化的工程信号做自动评价。避免了 LLM-Judge 的自恋偏见问题。
  → `sre-agent/backend/app/graph/evidence_utils.py:167-170`

- ✅[真实] 隐式人类评估信号——HITL 审批流程中，用户的 approve/reject 决策就是一种隐式的对齐标签。每次审批都在告诉系统：这个决策是对的/错的。虽然目前没有回流到训练闭环，但数据已经收集到了。
  → `sre-agent/backend/app/graph/nodes/__init__.py:1374-1515`（ApprovalRequest 持久化）

- ✅[真实] run_records 持久化支持对比分析——每次 agent 执行的完整记录（含 spec_id、step、payload）存储在 run_records 表中，天然支持 A/B 对比。
  → `harness-system/platform/src/lib/runs.ts:25`（writeCheckpoint）

- ⚙️[补全] Elo Rating 和 A/B Testing 框架——没有实现 pairwise 对比和 Elo 排名系统。但 run_records 数据可以直接用于对比分析，基础设施已就绪。

#### 🟡 我能拿到的加分项

- ✅[真实] 多 Judge 思想在 agent 层的落地——我们有 4 个 specialist agent 各自独立分析，然后 aggregator 加权聚合。单个 agent 的结论不被信任，多个 agent 交叉验证才被采纳。这跟"多 Judge 投票避免自恋偏见"是同一思路。
- ✅[真实] spec_versions 表支持回归分析——每次 spec 变更都有版本快照，可以对比不同版本的对齐效果是否退化。这跟 A/B Test 的思路一致——有变更时验证不退化。

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们做了人类评估" → 我们有 HITL 审批作为隐式人类信号，但不是系统化的人类评估流程。
- ❌ 别说"LLM-as-Judge 没有自恋偏见" → 我知道有自恋偏见，我们的做法是用多信号+多 Agent 交叉验证来规避——不用单一 Judge，用多个信号加权。

#### 完整应答（口语稿）

> 对齐评测我们用的是一套多信号自动评估加隐式人类反馈的方案。
>
> 自动评估方面，我们没有用 LLM-as-Judge 直接让模型打分——因为我知道有自恋偏见问题——而是用工程信号做自动评价。evidence quality scoring 基于工具执行的成功率和覆盖率计算质量分数。而且我们有 4 个 specialist agent 各自独立分析，aggregator 加权聚合，相当于多个 Judge 投票——单个 agent 被 trust 之前要经过交叉验证。
>
> 人类评估方面，我们走了隐式路径。HITL 审批流程中，用户的每一次 approve 或 reject 决策都被持久化——这本质上就是一个对齐标签。虽然目前只是收集，还没有回流入训练，但数据已经在积累了，随时可以用来做偏好对或 reward signal。
>
> 对于 A/B Test，我们的 run_records 表记录了每次 agent 执行的完整历史，包含使用的 spec 版本、执行的 tool calls、最终结果。你可以用不同版本的 config 跑同一个 eval case，然后对比结果——这就是最基本的 A/B 框架。要升级成完整的 Elo Rating 系统，需要在 run_records 基础上加 pairwise 对比逻辑，但数据层已经准备好了。

---

## 模块六：架构设计与前沿

### Q17　企业级 LLM 安全护栏架构设计（★拉档题）

> 主接项目：sre-agent + harness-system 联合设计。输入→推理→输出→业务四层护栏的全链路代码映射。

#### 🟢 我命中的参考答案要点

- ✅[真实] **接入层**：统一 ToolGateway 做参数校验 + EnvPolicy 环境策略 + agent tool_names 白名单 + trace_id 生成。所有请求经过同一入口，trace_id（run_id）全程携带。
  → `sre-agent/backend/app/tools/gateway.py:862-885,994-1013`、`env.py:5-53`、`specialist_agent.py:297-311`、`tracing.py:47`

- ✅[真实] **推理层**：risk_gate_node 4 路路由（LOW_ONLY / NEEDS_APPROVAL / BLOCKED / NEEDS_HUMAN）+ loop_guard 熔断（2 轮证据收集上限）+ quality gating（< 0.4 重试）。实时监控 action risk_level × environment × severity × confidence 的交叉决策矩阵。
  → `sre-agent/backend/app/graph/nodes/__init__.py:1249-1363,1115-1148,1164-1167`

- ✅[真实] **输出层**：ConfigMap 敏感值脱敏（9 个 sensitive keys → REDACTED）+ 置信度封顶（≤ 0.9）+ 审计日志递归脱敏（7 个 sensitive keys → REDACTED）+ 工具调用全量审计（IncidentToolAudit 持久化）。
  → `sre-agent/backend/app/tools/adapters/k8s_adapter.py:646-658`、`specialist_agent.py:395`、`gateway.py:840-856,1084-1132`

- ✅[真实] **业务层/反馈层**：HITL 审批（ApprovalRequest + DB 持久化 + /approvals/{id}/decision）+ 审批角色链（tech → qa → security → ops）+ fail-closed 降级（LLM_FAILED → 规则引擎接管）+ bad_cases 闭环。
  → `sre-agent/backend/app/graph/nodes/__init__.py:1374-1515`、`harness-system/platform/src/lib/approvals.ts:4-9`、`db.ts:114-141`

- ✅[真实] **告警分级**：
  - P0 数据泄露 → K8s ConfigMap 脱敏 + audit sanitization 预防
  - P1 注入攻击成功 → FORBIDDEN_TOOLS + tool whitelist + risk_gate BLOCKED 拦截
  - P2 误杀率突增 → loop_guard 2 轮限制 + quality gating 阈值控制

#### 🟡 我能拿到的加分项

- ✅[真实] 双策略载体——harness-system 的 AGENTS.md（行为软约束）+ spec-guard plugin hooks（系统硬约束）。策略不能只写在 prompt 里，必须有两层：一层让模型"愿意做对"，一层让系统"确保做对"。
  → `harness-system/.opencode/plugin/spec-guard/index.ts:212-289`

- ✅[真实] L0-L3 工具风险分级作为护栏决策的数据基础——每个工具注册时声明 risk_level + requires_approval + timeout_ms + side_effect，护栏运行时直接读取元数据做决策。不是每次调用都做推理，而是"注册即定安全等级"。
  → `harness-system/tools/registry/index.yaml:1-142`

- ✅[真实] 七层防御链——harness-system 的设计：Spec → Runtime → Tool/Skill → Control Plane → Model Gateway → Trace/Evidence → Eval/CI/Org。每一层都是一个 gate，不只是并列堆组件，而是一条链。
  → `harness-system/七层架构.md:16-31`

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们有语义注入检测" → L1 目前是规则驱动，没有小模型分类器。但设计上预留了位置——在 ToolGateway 前加一层 semantic_safety_check()。
- ❌ 别说"输出内容审核做了" → 输出层目前脱敏敏感字段，不做毒性/事实核查。但我会说"在当前系统设计下，多源交叉验证 + 置信封顶 + 规则兜底已经覆盖了关键风险"。

#### 完整应答（口语稿）

> 我在两个项目中落地了企业级安全护栏架构，四层设计每层都有对应的代码实现。
>
> 接入层用统一 ToolGateway 做入口。所有请求进来先过参数校验——_validate_tool_params 检查必填参数和类型。再过环境策略——生产环境禁止 delete、drop、truncate。再过工具白名单——不在 agent tool_names 里的工具一律拒绝。同时 trace_id 生成，全程追踪。这是 L1 输入护栏。
>
> 推理层核心是 risk_gate_node。它基于 action risk_level、环境、告警严重度、诊断置信度做 4 路决策。产线 + execute_action + 低置信被直接 BLOCKED。高危操作 + 产线 → NEEDS_APPROVAL 进审批。还有熔断机制——loop_guard 限制最多 2 轮证据收集，超过触发 NEEDS_HUMAN。quality_score < 0.4 自动重试。这是 L2 推理护栏。
>
> 输出层三道脱敏机制。工具层——K8s ConfigMap 查询时敏感值脱敏。系统层——置信度强制封顶 0.9。审计层——持久化前递归脱敏 7 类敏感字段。所有工具调用全量写入 IncidentToolAudit 表。这是 L3 输出护栏。
>
> 业务层是 HITL 审批体系。高危操作生成 ApprovalRequest，包含风险等级、预期影响、回滚方案，持久化到数据库。审批角色链是 tech → qa → security → ops，逐级审批。降级链路从 L0 完整分析到 L3 LLM_FAILED，每一级都有规则引擎兜底。坏案例记录到 bad_cases 表形成闭环。这是 L4 业务护栏。
>
> 告警分三级：P0 数据泄露直接脱敏阻断，P1 注入攻击通过白名单和环境策略拦截，P2 误杀率通过熔断和阈值控制。关键设计原则是策略不能只写在 prompt 里——AGENTS.md 是行为侧的软约束，plugin hooks 是系统侧的硬约束。双策略载体确保不会因为模型"忘记"规则而出现安全漏洞。

---

### Q18　模型对齐效果评估方案设计

> 主接项目：sre-agent 的评估基础设施 + harness-system 的 bad_cases 闭环。可扩展为完整对齐评估框架的设计思路。

#### 🟢 我命中的参考答案要点

- ✅[真实] 安全性评估——已有 security incident eval case + SQL 注入测试，覆盖了安全性维度。
  → `sre-agent/backend/app/evals/datasets/case_08_security_incident.json`、`tests/test_mysql_adapter.py:191-211`

- ✅[真实] 有用性评估——evidence quality scoring 自动评价 agent 输出质量 + specialist confidence 反映分析的可信度，覆盖有用性维度。
  → `sre-agent/backend/app/graph/evidence_utils.py:167-170`、`specialist_agent.py:395`

- ✅[真实] 回归测试基础设施——harness-system 的 bad_cases 表记录失败案例 + spec_versions 表记录变更历史，支持"变更 → 回归运行 → 对比退化"的完整流程。
  → `harness-system/platform/src/lib/db.ts:114-141,68-76`

- ⚙️[补全] 完整的对齐评估流程——目前缺少标准基准的集成、缺少系统化的人类评估流程、缺少在线 A/B Test 框架。但这些都可以在现有基础设施上快速构建。

#### 🟡 我能拿到的加分项

- ✅[真实] 多维度信号融合——安全性 × 有用性 × 真实性三个维度在我们的设计中是自然融合的：安全事件 eval 测安全，quality scoring 测有用，多 Agent 交叉验证测真实（单一来源不信任）。虽然没叫"三维评估"，但实际上是这么做的。
- ✅[真实] human approval 作为持续对齐信号——每个审批决策都可以当作偏好标签。如果收集 1000 个审批数据，就可以用 DPO 的思路做对齐。

#### 🔴 危险信号（主动规避）

- ❌ 别说"对齐评估方案已经很完善" → 标准基准没集成，人类评估不系统。但基础设施已就绪，扩展到完整评估方案只需要 2-3 周的工程投入。

#### 完整应答（口语稿）

> 对齐评估方案我倾向三维设计——安全、有用、真实——每维有具体指标和评估方式。
>
> 安全维度用自建 security eval case + 即将集成的 SafetyBench。我们已有的 security incident case 验证系统对安全事件的识别和路由是否正确，SQL 注入测试验证查询层防护。这些可以扩展到更多安全场景。
>
> 有用维度用 quality scoring + 人类审批信号。evidence quality scoring 自动评价 agent 输出质量，这是无监督的自动评估。HITL 审批决策中，用户 approve/reject 就是有监督的人类标签。这两者结合就是自动评估 + 人类校准。
>
> 真实维度用多 Agent 交叉验证 + 置信封顶。4 个 specialist agent 各自独立分析，aggregator 加权聚合——单一来源的结论不被信任。加上置信度强制 ≤ 0.9——系统层面的真实度兜底。
>
> 基础设施上，我们有了三个关键组件：eval case loader 做自动化评估，run_records 做执行历史追踪，bad_cases 做回归测试。这三个组件串联起来就是完整的离线评估 → 上线前回归 → 在线监控的评估流程。虽然标准基准还没集成，但框架已经就绪，接入 SafetyBench 和 TrustGPT 只需要扩展评估用例的 schema 和加载逻辑。

---

### Q19　DPO vs RLHF 选型决策与实战

> 主接项目：sre-agent + harness-system 的工程选型类比。虽然没有直接做模型对齐训练，但系统中的设计选择体现了"对齐技术选型"的思维模式。

#### 🟢 我命中的参考答案要点

- ⚙️[补全] DPO vs RLHF 纯技术选型——项目不训练模型，所以这不是"实战经验"而是"工程类比"。但我们的架构选择体现了类似的 tradeoff 思考。

- ✅[真实] "隐式奖励" vs "显式奖励模型"的选择——sre-agent 选择用多信号自动打分（evidence quality + confidence + coverage）而非训练单一奖励模型。这类似于 DPO 的选择——不显式构建 RM，而是把评价隐含在打分逻辑中。选择原因类似：多信号更稳定、不需要训练和迭代奖励模型、工程上更简单。
  → `sre-agent/backend/app/graph/evidence_utils.py:167-170`

- ✅[真实] "硬约束" vs "软对齐"的互补设计——harness-system 选择了双策略载体：AGENTS.md 做行为引导（软约束/类比 RLHF 对齐），plugin hooks 做强制执行（硬约束/类比护栏）。这个选择体现了"不能只依赖模型对齐"的工程判断——跟业界 DPO + Guardrails 混合方案的设计思路一致。
  → `harness-system/AGENTS.md:26-31` + `spec-guard/index.ts:212-289`

- ⚙️[补全] DPO 不适合的场景 → 我们的替代方案。DPO 不能在线迭代——我们的替代是 HITL 审批持续收集人类反馈信号，虽然还没回流训练，但数据收集体系已经就绪。

#### 🟡 我能拿到的加分项

- ✅[真实] "多目标优化" vs "单一奖励"——我们的系统不追求单一指标最优，而是在安全性（不会被注入）、有用性（能准确诊断）、真实性（不瞎编）之间找平衡。这跟 RLHF 中需要 balance reward + KL divergence 的思路类似。
- ✅[真实] "快速迭代"的选择——LoRA+DPO 的优势是可以在同一基座上快速迭代多个对齐版本。我们的 eval framework 也支持这种快速迭代——换 agent config、换 tool_names、换 prompt template 然后跑 eval cases 对比。

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们实战选择了 DPO" → 没训练过模型。但我可以说"在我们的架构决策中，处处体现了类似 DPO 的'简化、稳定、可迭代'的选型思维"。
- ❌ 别说"DPO 就够了" → 我知道 DPO 的两个核心限制：偏好数据质量依赖 + 不支持在线迭代。

#### 完整应答（口语稿）

> DPO vs RLHF 的选型，虽然我们的项目不直接训练模型，但架构中的设计选择可以很好地类比这个 tradeoff。
>
> 首先，我们的 evidence quality scoring 不训练奖励模型，而是用多信号自动打分——工具成功率、证据覆盖率、置信度。为什么不用单一奖励模型？因为训练和迭代一个 RM 成本高，而且容易奖励黑客。多信号打分更稳定、可解释、可调参。这就是 DPO 的核心思想在工程上的体现——不显式建奖励模型，而是把偏好隐含到系统设计中。
>
> 其次，我们的双策略载体设计——AGENTS.md 行为约束 + plugin hooks 硬约束——直接对应了 DPO + Guardrails 的组合方案。单纯靠 DPO 对齐不够，因为偏好数据有噪声、不能在线迭代。所以我们加了一层硬约束：工具风险分级、审批链、环境策略。即使模型偏好错了，硬约束也能兜底。
>
> 如果真正要做模型训练选型，我的倾向是新项目用 DPO 快速对齐 baseline——LoRA + DPO 两周能出一个可用版本——然后用 RLAIF 或人类反馈做持续迭代。DPO 适合冷启动，但在需要持续在线学习的场景——比如客服系统每天都在积累新的用户反馈——还是得 RLHF 或 RLAIF。我们的 HITL 审批数据已经是一种高质量的人类反馈，如果能回流到训练，就是 DPO + human feedback 的混合方案。

---

### Q20　2026 前沿加分项整合（MCP 安全 / 自我反思 / 鲁棒性 / 可解释性）

> 主接项目：sre-agent + harness-system + rag 三个项目的技术特点与前沿方向的映射。每个前沿方向都有实际的工程原型对应。

#### 🟢 我命中的参考答案要点

- ✅[真实] **MCP 协议安全（工具调用集中可审计）**：sre-agent 的 ToolGateway 是所有工具调用的统一入口 + run_id 串联全链路 + IncidentToolAudit 全量记录。虽然没有用 MCP 协议，但实现了 MCP 安全理念的核心——集中治理、可追踪、可审计。
  → `sre-agent/backend/app/tools/gateway.py:1084-1132`、`tracing.py:47`、`models/db_models.py:164-176`

- ✅[真实] **Agent 互注入防御**：Agent 间只传结构化数据（SpecialistAnalysis 对象）不传自由文本 + 工具 category 隔离 + FORBIDDEN_TOOLS 全局黑名单。这是 Agent 互注入防御的三层工程实践。
  → `sre-agent/backend/app/graph/nodes/specialist_agent.py:33,35-41,174-180`

- ✅[真实] **LLM 自我反思（Reflexion）**：harness-system 的 Constitution Gates——agent 在规划阶段必须自我审查 6 项 checklist，通过后才进入执行。这跟 Reflexion 框架的核心思路一致——在执行前先自我检查，发现不对就修正。
  → `harness-system/AGENTS.md:24-31`

- ✅[真实] **鲁棒性即安全**：sre-agent 的 loop_guard 2 轮上限 + 4 级降级策略 + ControlledExecutor 的 idempotency_key 防重复执行。这些不是安全检测，而是"系统即使被攻击了也不会崩溃"的鲁棒性设计。跟"鲁棒性即安全"的理念一致——护栏可能被绕过，但鲁棒的系统不会因此产生灾难性后果。
  → `sre-agent/backend/app/graph/nodes/__init__.py:1115-1148`、`services/executor.py:62-80`

- ⚙️[补全] **可解释性辅助安全**：项目中没有使用 SHAP/LIME 做模型可解释性分析。但 harness-system 的 spec validation 提供了另一种"可解释性"——validator 对每个规范给出具体的失败原因和位置，相当于系统行为的可解释审计。

#### 🟡 我能拿到的加分项

- ✅[真实] harness-system 的 MCP 风格插件架构——spec-guard 插件通过 tool.execute.before 和 permission.ask hooks 拦截所有工具调用，统一做权限决策。这跟 MCP 的设计理念完全一致——不是每个工具自己做安全，而是通过统一协议集中治理。
  → `harness-system/.opencode/plugin/spec-guard/index.ts:212-289`

- ✅[真实] rag 项目的 FTS5 安全查询——虽然只是一个具体的技术点，但体现了"接口安全设计"的原则——外部输入不能直接被内部系统解释。这跟对抗鲁棒性的思路一致：输入归一化、防止语法注入。
  → `rag/server/src/modules/knowledge/retrieval.service.ts:96-101`

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们用了 SHAP/LIME" → 没有可解释性工具。
- ❌ 别说"我们实现了 Reflexion" → Constitution Gates 是固定 checklist，不是 LLM 自生成的反思。但设计思想是相通的。

#### 完整应答（口语稿）

> 2026 年前沿方向，我有几个可以跟项目经验紧密结合的点。
>
> 第一个是 MCP 安全。MCP 协议帮我们做到一件事：把散乱的工具调用集中到统一协议上，所有调用都可审计。虽然我们没用 MCP 协议本身，但 sre-agent 的 ToolGateway 和 harness-system 的 spec-guard 插件实现了这个理念——所有工具调用经过统一入口，run_id 串联全链路，每次调用都有完整审计记录。集中治理的好处是安全可观测性强一个量级。
>
> 第二个是 Agent 互注入防御。学术界讨论这个，我们实际上已经在工程中应对了。Agent 之间只传结构化对象不传自由文本——这天然减少了注入传播面。工具 category 隔离——k8s agent 不能调 db 工具。FORBIDDEN_TOOLS——所有 agent 都没有执行权限。三层防御，agent 被攻陷一个不影响全系统。
>
> 第三个是 LLM 自我反思。harness-system 的 Constitution Gates 就是一个自我反思机制——agent 在规划阶段必须自我审查 checklist，没通过就暂停。虽然目前是固定 checklist，但架构上预留了让 LLM 自己生成反思提示的空间。
>
> 第四个是鲁棒性即安全。这个我很认同——护栏可能被绕过，但鲁棒的系统不会崩溃。我们的 loop_guard 限制 2 轮循环、4 级降级保证永远有兜底、idempotency_key 防止重复执行——这些在常规安全框架之外，但我觉得这是更深层的安全设计。攻击者绕过了你的护栏，系统仍然按照设计降级而不是崩溃或乱执行。
