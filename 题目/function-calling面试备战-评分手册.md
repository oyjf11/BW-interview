# Function Calling & 工具调用 — 面试评分手册

> 题目源：基于项目自出题 | 题量：20 题 | 模块：5 | 拉档题：Q4, Q9, Q14, Q20

---

## 模块一：Function Calling 协议与设计

### Q1　你们项目中 function calling 的协议是怎么设计的？LLM 返回的 tool_calls 怎么处理？

- 🔵 考察点：候选人是否真正理解 OpenAI function calling 协议的完整处理链路——LLM 如何返回 tool_calls、客户端如何解析、如何将结果回传。
- 🟢 参考答案要点：
  - LLM 返回的响应中，`message.tool_calls` 是一个数组，每个元素包含 `id`、`function.name`、`function.arguments`（JSON 字符串）。
  - 解析后按 `name` 路由到对应工具执行，执行结果组装为 `role: "tool"` 消息追加回 `messages` 数组。
  - 能区分"带 tools 的对话轮次"和"不带 tools 的最终分析"两种模式。
- 🟡 加分项：
  - 提到同一轮可能有多个 tool_calls，需要并行执行后一次性回传结果。
  - 提到 LLM 可能不返回 tool_calls（直接给 content），需要处理边界。
  - 提到 tool_call_id 的正确回传，保证 LLM 能把结果和调用一一对应。
- 🔴 危险信号：
  - 说"tool_calls 是一个字符串"或混淆 tool_calls 和 function_call（旧版 API）。
  - 不知道 tool role message 的格式（role/tool_call_id/content）。
  - 说不清 LLM 拿到工具结果后如何继续生成（追加 messages 而非替换）。

### Q2　怎么把内部的工具定义映射成 OpenAI 兼容的 function calling schema？

- 🔵 考察点：是否做过内部工具元数据到 OpenAI tools 格式的转换，能否解释 schema 映射的字段对应关系。
- 🟢 参考答案要点：
  - 内部工具元数据包含 name、description、parameters(JSON Schema)，需要映射为 `{type: "function", function: {name, description, parameters}}`。
  - JSON Schema 的 `properties`、`required`、`type` 等字段直接透传，OpenAI 兼容标准 JSON Schema。
  - 映射层应该是一个可复用的方法，每次构建请求时动态生成，而非写死。
- 🟡 加分项：
  - 提到不同 provider 的 tools 格式差异（如 MiniMax 的非标准路径），映射层需要做 provider 适配。
  - 提到 schema 中可以加入 `enum`、`minimum/maximum` 等约束让 LLM 生成更精确的参数。
  - 提到工具描述（description）对 LLM 选择工具有重要影响，可以用自然语言设计技巧。
- 🔴 危险信号：
  - 说"直接把工具名丢给 LLM 就行"——没意识到 schema 格式的重要性。
  - 说不清 `function.parameters` 是什么结构（必须是 JSON Schema object）。

### Q3　支持多个 LLM provider 时，function calling 的差异怎么处理？

- 🔵 考察点：是否做过多 provider 适配，对不同 provider 的 function calling API 差异有没有实际处理经验。
- 🟢 参考答案要点：
  - 三个 provider（OpenAI / DeepSeek / MiniMax）在 tools 参数和 tool_choice 的传参方式上基本兼容 OpenAI 格式。
  - DeepSeek 因为 API 设计直接兼容 OpenAI，tools 可以原样透传。
  - MiniMax 用的是非标准 endpoint（`/v1/text/chatcompletion_v2`），但 tools 传参格式相同，也保持兼容。
  - 通过环境变量或配置切换 provider，代码层面无需修改 tools 相关逻辑。
- 🟡 加分项：
  - 提到如果 provider 的 tool_calls 响应格式有差异（如字段名不同），需要在解析层做适配。
  - 提到 MiniMax 早期版本可能对 tool_choice 支持不完整，需要做降级处理。
  - 提到 provider 级别 fallback（主 provider 挂了切备 provider）的设计考量。
- 🔴 危险信号：
  - 声称"做了完善的 provider 差异适配"，但实际只是透传——过度夸大。
  - 认为所有 provider 都完全兼容 OpenAI 格式——缺乏对边界差异的认知。

### Q4　tool_choice 参数怎么用的？什么时候用 auto vs required vs none？（★拉档题）

- 🔵 考察点：对 tool_choice 参数的深度理解——知道三种模式的行为差异和适用场景，这是拉开档次的题目。
- 🟢 参考答案要点：
  - `auto`：让 LLM 自行决定是否调用工具、调用哪个工具。适用于大多数场景，模型有自由度。
  - `required`（或 `any`）：强制 LLM 必须调用一个工具。适用于确定需要工具干预的场景。
  - `none`：禁止调用工具。适用于不需要工具的纯对话或最终输出阶段。
  - 可以通过 `{"type": "function", "function": {"name": "xxx"}}` 精确指定必须调用哪个工具。
- 🟡 加分项：
  - 提到不同场景的设计选择：信息收集阶段用 auto，执行阶段用 required，最终总结用 none。
  - 提到如果 LLM 在 required 模式下没返回 tool_calls，需要做兜底处理（重试或 fallback）。
  - 提到强制指定函数名时，LLM 可能会"生造"参数来满足要求——需要参数校验兜底。
  - 提到 tool_choice 和 max_tokens 的关系：required 下要保证 token 足够让 LLM 生成参数。
- 🔴 危险信号：
  - 不知道 auto/required/none 的区别。
  - 说"全部用 auto 就行，不需要其他模式"——缺乏场景化思考。
  - 把 tool_choice 和 stop 参数搞混。

---

## 模块二：工具 Schema 设计

### Q5　工具的 JSON Schema 怎么定义的？有哪些字段？

- 🔵 考察点：是否做过工具入参的 JSON Schema 设计，对 `properties`、`required`、类型约束有实践经验。
- 🟢 参考答案要点：
  - JSON Schema 根节点 `type: "object"`，包含 `properties`（字段定义）和 `required`（必填字段列表）。
  - 每个 property 可以定义 `type`（string / integer / object / array）、`description`（帮助 LLM 理解含义）。
  - 工具元数据除了 parameters_schema 外，还包括 name、description、risk_level、timeout_ms、retries 等运行期字段。
- 🟡 加分项：
  - 提到 `enum` 限制可选值，帮助 LLM 生成合法的参数。
  - 提到嵌套 object 的 schema 设计（如 `time_range` 包含 `start`/`end`）。
  - 提到 description 的质量直接影响 LLM 调用工具的准确性——是重要的 prompt engineering 环节。
- 🔴 危险信号：
  - 说"tool schema 就是写个函数签名"——不知道 JSON Schema 格式。
  - 分不清 `properties` 和 `required` 的关系。

### Q6　工具是怎么注册的？注册时机和生命周期？

- 🔵 考察点：对工具注册机制的设计考量——静态注册 vs 动态注册、集中管理 vs 分散注册。
- 🟢 参考答案要点：
  - 工具通过 `register_tool(ToolMetadata)` 写入全局注册表（dict），key 为工具名。
  - 注册时机是模块加载时（import 时），所有注册调用集中在 gateway 文件的模块顶层。
  - 注册后 TOOL_REGISTRY 全局唯一，运行期间不变，重启服务才会刷新。
- 🟡 加分项：
  - 提到静态注册的优点：启动时就能校验所有工具的 schema 完整性、无运行时竞态。
  - 提到静态注册的局限：不支持热加载/运行时新增工具，需要重启。
  - 提到如果要做动态注册，可以用类加载器 + 热加载目录扫描，但需要处理版本兼容和回滚。
- 🔴 危险信号：
  - 声称"支持运行时动态注册"——实际没有。
  - 不知道注册表存放在哪（全局 dict / 数据库 / 文件）。

### Q7　参数校验怎么做的？JSON Schema validation 的具体实现？

- 🔵 考察点：是否实现了工具调用的参数校验，校验的覆盖度和实现方式。
- 🟢 参考答案要点：
  - 在工具网关的 `call_tool` 流程中，执行前先校验参数。
  - 校验检查 required 字段是否存在，以及每个参数的类型是否匹配（string / integer / object / array）。
  - 校验失败直接返回错误，不执行工具逻辑。
- 🟡 加分项：
  - 提到如果需要更完整的校验（enum / pattern / minimum / maximum / 嵌套 schema），应该引入 jsonschema 库而非手写。
  - 提到参数校验的时机：应该在 LLM 生成参数后、实际执行前，作为安全围栏。
  - 提到校验失败时应该返回清晰的错误信息，帮助 LLM 修正参数后重试。
- 🔴 危险信号：
  - 声称"用了 jsonschema 做校验"——实际是手写的简单类型检查。
  - 认为"LLM 不会传错参数，不需要校验"——缺乏安全意识。
  - 不知道校验发生在工具调用的哪个环节。

### Q8　工具元数据（timeout、retries、risk_level）怎么管理？设计考量？

- 🔵 考察点：对工具非功能性属性的管理方式——是元数据驱动还是代码中硬编码。
- 🟢 参考答案要点：
  - 工具元数据通过 ToolMetadata 模型统一定义，包含 name、description、parameters_schema、timeout_ms、retries、risk_level、requires_approval。
  - 元数据在注册时确定，与工具实现解耦——修改风险等级不需要改工具代码。
  - timeout_ms 默认 30000ms（30s），高危操作（如 execute_action）提升到 60000ms。
- 🟡 加分项：
  - 提到元数据驱动的好处：运营人员可以调整 timeout/retries 而不需要改代码。
  - 提到 timeout_ms 目前是元数据字段但 gateway 层面未强制使用，实际超时控制在上层 agent 中。
  - 提到 retries 字段虽然定义了，但实际重试次数被硬编码——这是技术债，应该让元数据驱动行为。
  - 提到 risk_level 应该和监控告警联动：HIGH 级工具调用次数异常时自动告警。
- 🔴 危险信号：
  - 搞混元数据和工具逻辑——以为元数据影响了工具执行结果。
  - 声称"元数据 100% 驱动运行时行为"——实际有些字段只是存着没用。

---

## 模块三：工具可靠性与错误处理

### Q9　工具调用的重试机制怎么实现的？（★拉档题）

- 🔵 考察点：重试策略的核心设计——退避算法、重试条件、幂等性考量。这是可靠性设计的核心拉档题。
- 🟢 参考答案要点：
  - 使用 tenacity 库的 `@retry` 装饰器实现自动重试。
  - 配置为最多 2 次尝试、指数退避等待（起始 1s，最大 10s）、耗尽后抛出异常（reraise=True）。
  - 重试包裹在工具实际执行的外部，对工具代码透明。
- 🟡 加分项：
  - 提到指数退避的设计原因：短期瞬时故障快速重试，长期故障避免打垮下游。
  - 提到重试条件应该区分可重试异常（网络超时、连接拒绝）和不可重试异常（参数校验失败、权限不足）。
  - 提到幂等性保证：对于写操作，重试前需要幂等键（idempotency_key）防止重复执行——项目中 ControlledExecutor 实现了这个机制。
  - 提到应该让 ToolMetadata.retries 字段真正驱动重试次数，而非硬编码——当前的技术债和改进方向。
  - 提到重试熔断：如果某个工具连续失败 N 次，应该临时熔断避免雪崩。
- 🔴 危险信号：
  - 说"重试就是简单的 while 循环"——不知道指数退避。
  - 不区分可重试和不可重试异常——所有错误都重试会放大问题。
  - 声称"重试次数由元数据灵活控制"——实际硬编码。

### Q10　工具调用超时怎么处理？

- 🔵 考察点：超时控制的分层设计——哪些层面需要超时，超时后如何处理 pending 任务。
- 🟢 参考答案要点：
  - 使用 `asyncio.wait_for` 控制异步调用的超时，分多个层级：
    - 单个 LLM 调用有 deadline 限制。
    - 单个工具调用有基于剩余时间的动态超时（`remaining_ms / len(tool_names)`）。
    - evidence_fanout 并行收集有整体 300 秒超时。
  - 超时后取消 pending 的 asyncio tasks，防止资源泄漏。
- 🟡 加分项：
  - 提到分层超时设计：agent 层面用 deadline 强制退出、fanout 层面用固定时间窗口、单工具层面用动态分配。
  - 提到超时后的降级处理——不是直接抛异常，而是标记为 PARTIAL 结果继续后续流程。
  - 提到 gateway 层面也应该加超时保护，但目前依赖上层——这是未来加固点。
- 🔴 危险信号：
  - 不知道 `asyncio.wait_for` 的用法。
  - 认为"设个全局 timeout 就行"——缺乏分层思考。

### Q11　工具调用失败后的降级策略？

- 🔵 考察点：失败后的恢复路径设计——是直接失败还是优雅降级，降级链有几层。
- 🟢 参考答案要点：
  - evidence_fanout 中区分了"可降级工具"和"不可降级工具"：非关键工具的失败不阻塞整体流程。
  - 使用 `asyncio.gather(..., return_exceptions=True)` 捕获异常不中断并行任务。
  - SpecialistAgent 有 4 级降级路径：LLM 成功 → JSON 解析失败但可用原始数据 → LLM 完全失败但可用规则提取 → 彻底失败（输出空结果）。
  - failed_evidence_tools 在 state 中跟踪所有失败的工具，供后续节点决策。
- 🟡 加分项：
  - 提到降级不是"随便失败"——每个降级层都有可用的输出，不是返回 null。
  - 提到降级和恢复的关系：critic 节点可能判定 NEED_MORE_EVIDENCE 触发重试，给失败的工具第二次机会。
  - 提到降级路径应该可配置——哪些工具可以降级、降级到哪一层。
- 🔴 危险信号：
  - 说"失败就抛异常让上层处理"——没有降级设计。
  - 分不清降级和重试的区别。

### Q12　响应数据太大怎么处理？截断策略？

- 🔵 考察点：对输出控制的意识——工具可能返回大量数据，如何防止 OOM 或 token 超限。
- 🟢 参考答案要点：
  - K8s adapter 定义了 `RESPONSE_SIZE_LIMIT_KB = 128`（128KB），MySQL adapter 用 `MAX_BYTES = 64 * 1024`（64KB）。
  - 采用渐进式截断策略：先清空最消耗空间的字段（如 sample_logs），再精简次要信息（如 pods 详情），最后截断核心数据（如 top_patterns）。
  - MySQL 查询结果通过 while 循环 pop 尾部数据直到满足大小限制。
- 🟡 加分项：
  - 提到渐进式截断优于一刀切：先丢非关键信息，保证关键信息完整传递给 LLM。
  - 提到截断时应当加 truncation marker（如 `... (truncated)`），让 LLM 知道数据不完整。
  - 提到当前每个 adapter 独立实现截断——应该提取为全局中间件，统一策略。
  - 提到还可以做语义摘要代替截断（如 K8s pods 过多时返回统计信息而非全量列表）。
- 🔴 危险信号：
  - 没考虑过响应大小问题——"LLM context window 够大就行"。
  - 说"直接用 `str[:1000]` 截断"——不考虑 JSON 结构截断后变非法格式。

---

## 模块四：工具安全与风险管控

### Q13　工具的风险分级怎么设计的？

- 🔵 考察点：工具安全的第一道防线——是否对工具做了风险分级，分级维度有哪些。
- 🟢 参考答案要点：
  - 按动作类型分为三级：HIGH（delete/terminate/force-stop/drop-table）、MEDIUM（restart/scale/update-config/rollback）、LOW（read/query/list/describe/get）。
  - 结合服务敏感度升级风险：对 CRITICAL_SERVICES（database/redis/kafka/etcd）执行 medium 动作自动升级为 HIGH。
  - 最终输出四级 RiskLevel：LOW / MEDIUM / HIGH / CRITICAL。
- 🟡 加分项：
  - 提到静态风险（工具定义时）和动态风险（运行时参数和服务上下文）的结合评估。
  - 提到 Harness 系统的 L0-L3 四级分级（只读/本地写/外部写/生产高风险）是对工具分级的平台层标准化。
  - 提到风险分级和后续的审批流程、监控告警是一体的。
- 🔴 危险信号：
  - 认为"工具都是安全的，不需要分级"。
  - 只分"读"和"写"两级，缺少中间层级。

### Q14　高风险操作（如 execute_action）怎么管控？审批流程怎么设计？（★拉档题）

- 🔵 考察点：对高风险操作的全链路管控设计——从前置检查到审批到执行到验证。这是安全设计的核心拉档题。
- 🟢 参考答案要点：
  - execute_action 在注册时标记 `risk_level="HIGH"`、`requires_approval=True`。
  - risk_gate_node 做四路决策：LOW_ONLY（直接执行）/ NEEDS_APPROVAL（需审批）/ BLOCKED（拒绝）/ NEEDS_HUMAN（需人工介入）。
  - 高危操作 + 生产环境 + 低置信度 → 直接 BLOCKED，不给审批机会。
  - 需要审批时进入 approval_interrupt_node，创建 ApprovalRequest 持久化到 DB，设置 state["pending_approval"] 挂起工作流。
- 🟡 加分项：
  - 提到 ControlledExecutor 的三层安全保护：幂等检查（防重复执行）→ 前置条件检查（service_exists / deployment_healthy）→ 执行记录 + 审计。
  - 提到审批不是简单的"同意/拒绝"——还要校验审批人的权限等级。
  - 提到审批超时处理：如果审批 pending 太久，需要自动拒绝或降级为人工处理。
  - 提到操作回滚机制：高风险操作执行前保存快照，失败时可回滚。
- 🔴 危险信号：
  - 说"高风险操作我们不让 agent 碰，全部人工处理"——等于没有 agent 能力。
  - 审批流程只有"是/否"没有条件化路由。
  - 不知道审批挂起后工作流如何恢复（checkpoint/resume）。

### Q15　生产环境的操作怎么限制？

- 🔵 考察点：环境感知的安全策略——是否区分生产/预发/测试环境，不同环境的操作权限有何不同。
- 🟢 参考答案要点：
  - 生产环境（prod/production/prd）中禁止 delete / drop / truncate 等破坏性操作。
  - 不同环境有操作白名单：prod 只允许 restart/scale_up/scale_down；staging 额外允许 rollback；dev 额外允许 update_config。
  - 环境检查通过 `can_execute_action` 返回 {allowed, reason, requires_approval} 三元组。
- 🟡 加分项：
  - 提到环境标识的获取方式——从请求参数或配置文件读取，需防范伪造。
  - 提到灰度环境的处理：灰度环境应该按 staging 还是 prod 对待？取决于业务容忍度。
  - 提到操作回滚比禁止更实用——某些场景需要快速回滚能力。
- 🔴 危险信号：
  - 说"生产环境和开发环境用同一套权限"。
  - 不知道环境策略放在哪里执行（应该在 gateway 层，而非工具层）。

### Q16　审计日志怎么记录的？

- 🔵 考察点：审计的完整性和安全性——记录了什么、什么时候记、敏感信息怎么处理。
- 🟢 参考答案要点：
  - 每次工具调用都记录：audit_id、run_id、tool_name、adapter_mode、request_json、response_json、success、error_message、latency_ms、created_at。
  - 成功和失败两条路径都记录审计，在 try/finally 中完成，写入失败不影响工具调用结果。
  - 敏感字段脱敏：password、secret、token、api_key、access_key、authorization、cookie 等替换为 "***REDACTED***"。
- 🟡 加分项：
  - 提到敏感字段的递归脱敏——dict 嵌套和 list 中的敏感 field 都能覆盖。
  - 提到审计的存储层次：内存 + 数据库双写，内存列表用于实时查询（如按 run_id 过滤），数据库用于持久化和合规。
  - 提到审计日志的保留策略和访问权限——审计日志本身就是敏感数据。
- 🔴 危险信号：
  - 声称"没有审计机制"或"忘了做"。
  - 审计时直接记录原始参数（含密码/密钥）。

---

## 模块五：外部系统集成与编排

### Q17　适配器模式怎么设计的？mock 和 real 怎么切换？

- 🔵 考察点：工具集成的工程化设计——如何让工具在开发环境和生产环境之间平滑切换。
- 🟢 参考答案要点：
  - 核心设计是 adapter 模式：每个工具都有 mock handler（返回模拟数据）和 real handler（调用真实外部系统）。
  - 通过 `ADAPTER_MODE` 环境变量（mock / real）全局切换，无需改代码。
  - select_adapter 函数根据 mode + 工具名查找对应的 real handler，找不到则降级为 mock 或返回错误。
- 🟡 加分项：
  - 提到 adapter 和工具的对应关系是 N:1——一个工具可以有多个 adapter（比如 k8s 工具对应 k8s_adapter）。
  - 提到 real adapter 的封装模式：底层用独立的 client 类（mysql_client / k8s_client / slb_client 等），adapter 负责数据转换和错误包装。
  - 提到 eval fixture 优先机制：调试或评估场景可以注入 fixture 响应，跳过真实调用。
  - 提到当前 execute_action 的 real adapter 是占位实现（返回 RuntimeError）——这是未完成的安全边界。
- 🔴 危险信号：
  - 说"直接调 API，不需要 adapter"——不理解 mock/real 切换的价值。
  - 不知道怎么在不改代码的情况下从开发切到生产。

### Q18　多个工具并行调用怎么实现？

- 🔵 考察点：并行编排的能力——是否做过工具级别的并行执行，怎么管理并发。
- 🟢 参考答案要点：
  - 使用 `asyncio.gather(*tasks, return_exceptions=True)` 实现多个工具并行调用。
  - 有两个并行维度：evidence_fanout 的多个 SpecialistAgent 并行；单个 SpecialistAgent 内同一轮多个 tool_calls 并行。
  - 整体有超时控制（300s），超时后取消所有 pending tasks。
- 🟡 加分项：
  - 提到 `return_exceptions=True` 的设计——单个工具失败不拖垮并行组，结果中混合成功和失败。
  - 提到并行数量的控制——不是无限并发，受限于 asyncio 事件循环和下游系统承受能力。
  - 提到如果工具之间有依赖关系，需要分轮次而不是无脑并行。
- 🔴 危险信号：
  - 说"并行就是用多线程"——async/await 是协程，不是线程。
  - 不做超时控制——一个慢工具拖死所有并行任务。

### Q19　Agent 的 ReAct 循环中工具调用怎么编排？

- 🔵 考察点：Agent 循环的核心实现——ReAct pattern 的理解和工程落地。
- 🟢 参考答案要点：
  - 标准 ReAct 循环：for round_num in range(max_tool_rounds) → LLM 调用（带 tools） → 无 tool_calls 则 break → 并行执行工具 → 追加 tool role message → 继续下一轮。
  - 每个 SpecialistAgent 有独立的 tool 白名单（基于 agent category 过滤），LLM 请求的工具如果不在白名单则返回错误注入。
  - 全局禁止 execute_action（FORBIDDEN_TOOLS），防止查询 agent 误执行写入操作。
- 🟡 加分项：
  - 提到 max_tool_rounds 防止无限循环——这是 ReAct agent 的经典陷阱。
  - 提到每轮有时间预算（deadline 检查），超时即退出而非无限等待。
  - 提到工具过滤的安全意义——agent 只能看到它应该用的工具，而非全部工具。
  - 提到多 agent 协作时，不同 agent 共享同一个 run_id 和 audit 上下文。
- 🔴 危险信号：
  - 说"ReAct 就是 while True 循环"——没有 round 限制和超时。
  - 不限制工具的可见范围——所有工具对所有 agent 开放。

### Q20　LangGraph 图编排中工具节点怎么嵌入？（★拉档题）

- 🔵 考察点：对 Agent 工作流编排的理解——工具调用在图中的地位、工具节点和图流程的关系。这是工程架构的综合拉档题。
- 🟢 参考答案要点：
  - 工作流有 13 个节点，工具调用分布在多个节点中：evidence_fanout（并行收集 K8s/metrics/logs）、executor（经风险审批后执行操作）、verify_outcome（验证操作效果）、rca（写报告到 OSS）。
  - 所有工具调用统一通过 gateway.call_tool() 中转，图节点不直接调用 adapter。
  - 条件路由基于工具的执行结果：critic 根据证据质量决定是返回 evidence_fanout 补充证据、还是继续 remediation，verify 根据验证结果决定是重试 executor 还是写 RCA。
- 🟡 加分项：
  - 提到 gateway 作为一个 "tool plane"，将工具调用的横切关注点（重试、审计、schema 校验、超时）从业务逻辑中分离出来。
  - 提到条件路由的设计考量：什么时候走重试、什么时候走降级、什么时候直接结束。
  - 提到图的可恢复性：通过 checkpoint 机制，审批中断后可以恢复到原节点继续执行。
  - 提到 dispatcher 节点支持从任意节点 resume——这在审批恢复和生产调试中非常实用。
  - 提到为什么工具节点要在图层面编排而非全部交给一个 agent 自主决定——可观测性、可控性、审批介入。
- 🔴 危险信号：
  - 认为"一个 agent 就能搞定，不需要图编排"——缺乏对生产级可控性的理解。
  - 图节点和工具调用发生混乱——不知道每个节点调用哪些工具。
  - 说不清条件路由的条件和去向。

---

> 评分手册完。共 5 个模块，20 题，每题为四部分（🔵🟢🟡🔴）。
