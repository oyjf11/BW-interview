# Function Calling & 工具调用 — 面试应答文档

> 主接项目：sre-agent (OpsPilot) 的 ToolGateway + SpecialistAgent + LangGraph 编排体系
> 辅助项目：harness-system 的工具风险分级治理（L0-L3）
> 项目映射：sre-agent（核心实现，Q1-Q20 全部涉及）；harness-system（Q13 风险分级补充）

---

## 模块一：Function Calling 协议与设计

### Q1　你们项目中 function calling 的协议是怎么设计的？LLM 返回的 tool_calls 怎么处理？

> 主接项目：sre-agent 的 LLMClient + SpecialistAgent ReAct 循环。OpenAI 兼容的 tools 协议，三个 provider 统一处理。

#### 🟢 我命中的参考答案要点

- ✅[真实] LLM 返回的响应中，从 `choice.message.tool_calls` 提取工具调用列表，每个包含 `id`、`function.name`、`function.arguments`（JSON 字符串）
  → `backend/app/llm_client.py:146-150`（OpenAI）、`:187-191`（DeepSeek）、`:228-232`（MiniMax）——三个 provider 用相同方式提取

- ✅[真实] 解析后的 tool_calls 通过 `ToolRequest` 对象交给 `ToolGateway.call_tool()` 执行，执行结果组装为 `role: "tool"` 消息追加回 messages
  → `backend/app/graph/nodes/specialist_agent.py:223-254`：`for tc in valid_calls` → `asyncio.gather` 并行执行 → `messages.append({"role": "tool", "tool_call_id": tc["id"], "content": ...})`

- ✅[真实] 区分两种模式：ReAct 循环中带 tools 调用，最终分析阶段传 `tools=None` 让 LLM 返回结构化 JSON 而非继续调用工具
  → `backend/app/graph/nodes/specialist_agent.py:348-358`：`_produce_final_analysis` 中 `tools=None`

#### 🟡 我能拿到的加分项

- ✅[真实] 同一轮多个 tool_calls 通过 `asyncio.gather` 并行执行，一次性回传结果
  → `backend/app/graph/nodes/specialist_agent.py:241-244`
- ✅[真实] 处理了 LLM 不返回 tool_calls 的边界——直接 break 退出循环
  → `backend/app/graph/nodes/specialist_agent.py:246-248`

#### 🔴 危险信号（主动规避）

- ❌ 别说"tool_calls 是一个字符串" → 我知道它是数组，每个元素是 dict
- ❌ 别说"旧版 function_call 格式" → 我用的就是新版 tool_calls 协议

#### 完整应答（口语稿）

> 我们项目里 function calling 用的是 OpenAI 兼容的 tools 协议。LLM 返回的响应里，`choice.message.tool_calls` 是一个数组，每个元素有 `id`、`function.name` 和 `function.arguments` 三个字段。我们在 `LLMClient` 里统一从这三个字段提取，三个 provider——OpenAI、DeepSeek、MiniMax——用同一套解析逻辑。
>
> 拿到 tool_calls 之后，我们在 SpecialistAgent 的 ReAct 循环里处理。先校验 LLM 请求的工具是不是在白名单里，不在的话直接注入错误消息让 LLM 修正。合法的 tool_calls 用 `asyncio.gather` 并行执行，每个工具调用走 `ToolGateway.call_tool()`，返回的结果按 `role: "tool"` 和对应的 `tool_call_id` 追加回 messages 数组，然后进入下一轮 LLM 调用。如果某一轮 LLM 没返回 tool_calls，就 break 退出循环，进入最终分析阶段——这时候传 `tools=None`，让 LLM 直接输出结构化的诊断结论。

---

### Q2　怎么把内部的工具定义映射成 OpenAI 兼容的 function calling schema？

> 主接项目：sre-agent 的 ToolGateway.get_tool_schema()。从 ToolMetadata 到 OpenAI tools 格式的映射层。

#### 🟢 我命中的参考答案要点

- ✅[真实] `gateway.get_tool_schema()` 方法将内部 `ToolMetadata` 映射为 `{type: "function", function: {name, description, parameters}}` 格式
  → `backend/app/tools/gateway.py:1134-1146`：从 registry 取 metadata → 提取 name/description/parameters_schema → 组装为标准格式

- ✅[真实] SpecialistAgent 在 ReAct 循环前通过 `_build_tool_definitions` 将 tool_names 批量映射为 OpenAI schema 数组
  → `backend/app/graph/nodes/specialist_agent.py:291-295`：遍历 `self.config.tool_names`，逐个调用 `gateway.get_tool_schema`

- ✅[真实] LLMClient 将 tools 数组原样放入 API payload，加上 `tool_choice: "auto"`
  → `backend/app/llm_client.py:135-137`

#### 🟡 我能拿到的加分项

- ✅[真实] 映射是动态生成的，不是写死的——每次构建请求时根据当前 agent 的 tool_names 过滤
- ⚙️[补全] 如果要做 provider 差异适配，可以在 `get_tool_schema` 里加一个 provider 参数，针对不同 provider 做字段名转换

#### 🔴 危险信号（主动规避）

- ❌ 别说"直接把工具名丢给 LLM" → 我知道需要完整的 JSON Schema
- ❌ 别说"parameters 就是函数签名" → 我知道是 JSON Schema 的 `{type, properties, required}` 结构

#### 完整应答（口语稿）

> 我们内部工具用 `ToolMetadata` 这个 Pydantic 模型描述，包含 name、description、parameters_schema 等字段。映射到 OpenAI 格式是通过 `ToolGateway.get_tool_schema()` 做的——从全局注册表拿到 metadata 之后，组装成 `{type: "function", function: {name, description, parameters}}` 这个结构。parameters 就是内部的 JSON Schema 直接透传，因为 OpenAI 本身就兼容标准 JSON Schema。
>
> 这个映射不是一次性的。每个 SpecialistAgent 有自己的工具白名单，在 ReAct 循环开始前，`_build_tool_definitions` 方法会根据 agent 的 tool_names 列表，逐个调用 `get_tool_schema` 生成 tools 数组，然后传给 LLMClient。LLMClient 把 tools 数组和 `tool_choice: "auto"` 一起放进 API payload。这样做的好处是，工具定义和 LLM 调用完全解耦——改工具 schema 不需要动 LLM 调用代码。

---

### Q3　支持多个 LLM provider 时，function calling 的差异怎么处理？

> 主接项目：sre-agent 的 LLMClient 三 provider 实现。OpenAI / DeepSeek / MiniMax 的 tools 传参方式。

#### 🟢 我命中的参考答案要点

- ✅[真实] 三个 provider 在 tools 和 tool_choice 的传参方式上复用相同的 OpenAI 兼容格式
  → `backend/app/llm_client.py:114-153`（`_openai_chat`）、`:155-194`（`_deepseek_chat`）、`:196-235`（`_minimax_chat`）——三个方法中 tools 都是直接放入 payload，tool_choice 都设为 "auto"

- ✅[真实] 通过 `LLM_PROVIDER` 环境变量动态分发到不同 provider 的 chat 方法
  → `backend/app/llm_client.py:99-112`：`complete_async` 方法根据配置选择 `_openai_chat` / `_deepseek_chat` / `_minimax_chat`

- ⚙️[补全] 三个 provider 的 tools 传参无差异化处理——DeepSeek 因为 API 兼容 OpenAI 所以直接可行，MiniMax 用的是非标准 `/v1/text/chatcompletion_v2` 路径但 tools 格式相同。如果某个 provider 的 tool_calls 响应格式有差异（比如字段名不同），当前代码未做适配，这是基于"都兼容 OpenAI"的假设

#### 🟡 我能拿到的加分项

- ⚙️[补全] 如果需要做 provider 差异适配，我会在 `_minimax_chat` 里加一个响应格式转换层，把 MiniMax 的 tool_calls 字段名统一到 OpenAI 格式
- ⚙️[补全] provider 级别 fallback 的设计思路：主 provider 超时或报错时，降级到备 provider，但需要保证 tools 格式在两个 provider 之间兼容

#### 🔴 危险信号（主动规避）

- ❌ 别说"做了完善的 provider 差异适配" → 实际是透传，DeepSeek 和 MiniMax 恰好兼容 OpenAI 格式
- ❌ 别说"所有 provider 都完全一样" → MiniMax 的 endpoint 路径不同，只是 tools 格式恰好一致

#### 完整应答（口语稿）

> 我们项目接了三个 LLM provider：OpenAI、DeepSeek 和 MiniMax。在 function calling 这块，三个 provider 的 tools 传参方式基本一致——都是 OpenAI 兼容格式，tools 数组直接放进 payload，tool_choice 设为 auto。LLMClient 里有三条独立的 chat 路径，通过环境变量 `LLM_PROVIDER` 来切换。
>
> 说实话，三个 provider 之间我们没有做差异化的 tools 格式转换。DeepSeek 因为 API 设计就是兼容 OpenAI 的，所以直接能用。MiniMax 虽然 endpoint 路径不一样——它用的是 `/v1/text/chatcompletion_v2`——但 tools 的传参格式和 OpenAI 一致，所以也不需要额外适配。不过这里有一个隐式假设：我们假设三个 provider 的 tool_calls 响应格式完全一致。如果某个 provider 的字段名有差异，比如 `tool_calls` 叫 `function_calls`，那当前代码就需要加适配层。目前因为三个 provider 都兼容，所以没遇到这个问题。

---

### Q4　tool_choice 参数怎么用的？什么时候用 auto vs required vs none？（★拉档题）

> 主接项目：sre-agent 的 LLMClient tool_choice 使用方式。当前只用 auto，但理解全部三种模式。

#### 🟢 我命中的参考答案要点

- ✅[真实] 项目中当 tools 非空时，统一使用 `tool_choice: "auto"`，让 LLM 自行决定是否调用工具
  → `backend/app/llm_client.py:137,178,219`——三个 provider 的 chat 方法都是 `payload["tool_choice"] = "auto"`

- ✅[真实] 当不需要工具调用时（如最终分析阶段），传入 `tools=None`，此时 tool_choice 不会被设置，等同于 "none" 的效果
  → `backend/app/graph/nodes/specialist_agent.py:348-358`

- ⚙️[补全] 项目中没有使用 `required`（强制调用工具）或按函数名指定 tool_choice 的场景。如果需要强制 LLM 调用某个工具，可以用 `{"type": "function", "function": {"name": "query_logs"}}` 精确指定

#### 🟡 我能拿到的加分项

- ⚙️[补全] 场景化设计思路：信息收集阶段用 auto（给 LLM 自由度），执行阶段用 required（必须调用工具），最终总结用 none（禁止调用）
- ⚙️[补全] required 模式下的兜底：如果 LLM 没返回 tool_calls，需要重试或 fallback，不能假设一定成功
- ⚙️[补全] 强制指定函数名时 LLM 可能"生造"参数——需要参数校验兜底

#### 🔴 危险信号（主动规避）

- ❌ 别说"我们用了 required 和 none" → 实际只用了 auto
- ❌ 别说"全部用 auto 就行" → 我知道不同场景需要不同策略
- ❌ 别把 tool_choice 和 stop 参数搞混

#### 完整应答（口语稿）

> tool_choice 有三种模式：auto、required 和 none。auto 是让模型自己决定要不要调工具、调哪个工具，这是最常用的模式。required 是强制模型必须调一个工具，适用于你确定这一步一定需要工具干预的场景。none 是禁止调工具，适用于纯对话或者最终输出阶段。
>
> 我们项目里目前只用到了 auto。在 LLMClient 的三个 provider 实现里，只要传了 tools，tool_choice 就设为 auto。当不需要工具的时候——比如 SpecialistAgent 的最终分析阶段——我们直接传 tools=None，效果上等同于 none。required 和按函数名指定的场景我们还没用到，但我理解它们的适用场景。比如在 remediation 节点，如果确定需要执行某个修复操作，用 required 可以避免 LLM 犹豫不决。不过用 required 要小心，LLM 在强制调用时可能会"编造"参数来满足要求，所以参数校验一定要跟上。另外，如果 LLM 在 required 模式下没返回 tool_calls，需要做兜底——重试或者降级处理。

---

## 模块二：工具 Schema 设计

### Q5　工具的 JSON Schema 怎么定义的？有哪些字段？

> 主接项目：sre-agent 的 ToolMetadata 模型 + 各工具注册时的 parameters_schema 定义。

#### 🟢 我命中的参考答案要点

- ✅[真实] `ToolMetadata` 模型包含 name、description、parameters_schema、risk_level、requires_approval、timeout_ms、retries 七个字段
  → `backend/app/tools/schemas/__init__.py:31-38`

- ✅[真实] `parameters_schema` 是标准 JSON Schema 格式：`{type: "object", properties: {...}, required: [...]}`
  → `backend/app/tools/gateway.py:239-255`（query_logs 注册示例）：properties 包含 service(string)、env(string)、time_range(object)、query(string)、limit(integer)，required 为 ["service", "env"]

- ⚙️[补全] 没有定义 `response_schema`（输出格式 schema）——ToolMetadata 只约束输入，不约束输出

#### 🟡 我能拿到的加分项

- ✅[真实] 支持嵌套 object 类型（如 `time_range: {type: "object"}`），不是只有基础类型
- ⚙️[补全] 如果要加 enum 约束，直接在 properties 里加 `"enum": ["prod", "staging", "dev"]`，能帮 LLM 生成更精确的参数

#### 🔴 危险信号（主动规避）

- ❌ 别说"tool schema 就是函数签名" → 我知道是 JSON Schema 格式
- ❌ 别说"定义了输出 schema" → 实际只定义了输入 parameters_schema

#### 完整应答（口语稿）

> 我们项目的工具 schema 用 Pydantic 的 `ToolMetadata` 模型来定义。核心字段有七个：name 是工具名，description 是给 LLM 看的自然语言描述，parameters_schema 是 JSON Schema 格式的入参定义，risk_level 是风险等级，requires_approval 标记是否需要审批，timeout_ms 是超时时间，retries 是重试次数。
>
> parameters_schema 是标准的 JSON Schema 格式，根节点 type 为 object，里面 properties 定义每个参数的类型和描述，required 列出必填字段。比如 query_logs 这个工具，properties 里有 service（string）、env（string）、time_range（object）、query（string）、limit（integer），required 是 service 和 env。支持嵌套 object 类型，比如 time_range 可以包含 start 和 end 子字段。目前我们只定义了输入 schema，输出格式没有在 ToolMetadata 里约束——这是后续可以补全的点。

---

### Q6　工具是怎么注册的？注册时机和生命周期？

> 主接项目：sre-agent 的 TOOL_REGISTRY + register_tool() 静态注册机制。

#### 🟢 我命中的参考答案要点

- ✅[真实] 工具通过 `register_tool(ToolMetadata)` 写入全局 `TOOL_REGISTRY` dict，key 为工具名
  → `backend/app/tools/schemas/__init__.py:41-45`

- ✅[真实] 注册时机是模块加载时（import 时），所有约 30 个工具的注册集中在 gateway.py 模块顶层
  → `backend/app/tools/gateway.py:235-831`——从 query_logs 到 write_evidence_to_oss，批量注册

- ✅[真实] 注册是静态的，运行期间 TOOL_REGISTRY 不变，重启服务才会刷新
  → 所有 register_tool 调用均在模块顶层，ToolGateway 没有提供动态 add/remove 方法

#### 🟡 我能拿到的加分项

- ⚙️[补全] 静态注册的好处是启动时就能校验所有工具的 schema 完整性，没有运行时竞态问题。局限是不支持热加载——新增工具需要重启服务
- ⚙️[补全] 如果要支持动态注册，可以用类加载器扫描工具目录，运行时发现新工具就注册，但需要处理版本兼容和回滚

#### 🔴 危险信号（主动规避）

- ❌ 别说"支持运行时动态注册" → 实际是静态注册
- ❌ 别说"注册表存在数据库里" → 实际是内存中的全局 dict

#### 完整应答（口语稿）

> 我们项目的工具注册用的是静态注册机制。有一个全局的 `TOOL_REGISTRY` dict，通过 `register_tool` 函数把 `ToolMetadata` 写进去，key 就是工具名。所有工具的注册都在 gateway.py 的模块顶层完成——大概 30 个工具，从 query_logs 到 write_evidence_to_oss，import 的时候就全部注册好了。
>
> 这个设计的好处是简单可靠——启动时所有工具的 schema 就确定了，没有运行时竞态。坏处是不支持热加载，想加新工具或者改 schema 必须重启服务。ToolGateway 也没有提供动态 add/remove 的接口。如果后续需要动态注册能力，可以考虑用类加载器扫描工具目录，运行时发现新模块就自动注册，但需要配套做版本管理和回滚机制。

---

### Q7　参数校验怎么做的？JSON Schema validation 的具体实现？

> 主接项目：sre-agent 的 `_validate_tool_params` 手写校验逻辑。

#### 🟢 我命中的参考答案要点

- ✅[真实] `_validate_tool_params` 函数在 `call_tool` 流程中执行，校验 required 字段是否存在 + 参数类型是否匹配
  → `backend/app/tools/gateway.py:862-885`：检查 required 字段（line 867-869）、检查类型 string/integer/object/array（line 875-883）

- ✅[真实] 校验失败返回 `ToolResponse(success=False, error=...)`，不执行工具逻辑
  → `backend/app/tools/gateway.py:1003-1013`

- ⚙️[补全] 未使用 jsonschema 库——当前是手写的类型校验，只覆盖 string/integer/object/array 四种基础类型，不支持 enum、pattern、minimum/maximum、嵌套 schema 等高级约束

#### 🟡 我能拿到的加分项

- ⚙️[补全] 如果要支持更完整的校验，应该引入 jsonschema 库做标准 validate，而不是手写 if-else
- ⚙️[补全] 校验失败时返回清晰的错误信息很重要——LLM 可以根据错误信息修正参数后重试，形成 self-correction 闭环

#### 🔴 危险信号（主动规避）

- ❌ 别说"用了 jsonschema 做校验" → 实际是手写的简单类型检查
- ❌ 别说"LLM 不会传错参数" → 参数校验是安全围栏，不能省略

#### 完整应答（口语稿）

> 参数校验是在 ToolGateway 的 call_tool 方法里做的，执行工具之前先跑 `_validate_tool_params`。这个函数做两件事：一是检查 required 字段是不是都传了，二是检查每个参数的类型对不对——支持 string、integer、object、array 四种。校验不通过就直接返回 error，不执行工具。
>
> 说实话，当前的校验实现比较基础——是手写的 if-else 类型检查，没有用 jsonschema 库。这意味着我们只能校验基础类型，像 enum 约束、pattern 正则、minimum/maximum 范围这些高级约束目前都不支持。如果要升级，引入 jsonschema 做标准 validate 是更稳妥的做法。另外，校验失败时的错误信息要足够清晰，因为 LLM 可以根据错误信息修正参数后重试——这是 self-correction 的关键环节。

---

### Q8　工具元数据（timeout、retries、risk_level）怎么管理？设计考量？

> 主接项目：sre-agent 的 ToolMetadata 元数据模型。元数据驱动 vs 硬编码的现状。

#### 🟢 我命中的参考答案要点

- ✅[真实] ToolMetadata 包含 timeout_ms（默认 30000）、retries（默认 1）、risk_level（默认 "LOW"）、requires_approval（默认 False）
  → `backend/app/tools/schemas/__init__.py:35-38`

- ✅[真实] execute_action 被标记为 risk_level="HIGH"、requires_approval=True、timeout_ms=60000，与其他只读工具形成对比
  → `backend/app/tools/gateway.py:333-338`

- ⚙️[补全] timeout_ms 作为元数据字段存在，但 gateway 的 `_execute_with_retry` 方法没有用它来控制超时——实际超时控制在上层 SpecialistAgent 的 `asyncio.wait_for` 中
- ⚙️[补全] retries 字段虽然定义了（默认 1），但 `_execute_with_retry` 里硬编码为 `stop_after_attempt(2)`，没有读取 ToolMetadata.retries

#### 🟡 我能拿到的加分项

- ⚙️[补全] 元数据驱动的理想状态：运营人员改 timeout/retries/risk_level 不需要动代码，但目前部分字段只是"存着没用"
- ⚙️[补全] 这是明确的技术债——应该让 `_execute_with_retry` 读取 `metadata.retries` 而非硬编码，让 gateway 层用 `asyncio.wait_for` 强制执行 `metadata.timeout_ms`

#### 🔴 危险信号（主动规避）

- ❌ 别说"元数据 100% 驱动运行时行为" → retries 和 timeout_ms 字段实际未被使用
- ❌ 别搞混元数据和工具逻辑 → 元数据是配置，不影响工具执行结果

#### 完整应答（口语稿）

> 工具元数据通过 ToolMetadata 模型统一管理，包含 timeout_ms、retries、risk_level、requires_approval 这些运行期字段。比如 execute_action 这个高危工具，timeout_ms 设成 60000、risk_level 是 HIGH、requires_approval 是 true，和普通只读工具的默认值形成明显对比。
>
> 不过说实话，元数据驱动这块我们做得还不够彻底。timeout_ms 字段虽然定义了，但 gateway 的 retry 方法并没有用它来控制超时——实际超时是在 SpecialistAgent 层面用 asyncio.wait_for 做的。retries 字段也是类似的情况，定义了默认值 1，但 `_execute_with_retry` 里硬编码了 `stop_after_attempt(2)`，没读这个字段。这是明确的技术债——理想情况下，改 timeout 和 retries 应该只改元数据，不需要动代码。后续应该让 gateway 层读取 metadata 的这些字段来真正驱动运行时行为。

---

## 模块三：工具可靠性与错误处理

### Q9　工具调用的重试机制怎么实现的？（★拉档题）

> 主接项目：sre-agent 的 tenacity @retry 装饰器 + ControlledExecutor 幂等机制。

#### 🟢 我命中的参考答案要点

- ✅[真实] 使用 tenacity 库的 `@retry` 装饰器实现自动重试，配置为最多 2 次尝试、指数退避等待（1s-10s）、耗尽后抛出异常
  → `backend/app/tools/gateway.py:1065-1079`

- ✅[真实] 重试包裹在工具实际执行的外部，对工具代码透明——handler 不需要感知重试逻辑
  → `backend/app/tools/gateway.py:1074-1079`：`result = handler(**params)` 被 retry 装饰器包裹

- ⚙️[补全] 重试次数硬编码为 2，未使用 ToolMetadata.retries 字段——这是技术债
- ⚙️[补全] 不区分可重试异常和不可重试异常——所有异常都触发重试，包括 TypeError/ValueError

#### 🟡 我能拿到的加分项

- ✅[真实] ControlledExecutor 实现了幂等键（idempotency_key）检查，防止写操作重复执行
  → `backend/app/services/executor.py:62`：检查 idempotency_key 是否已执行
- ⚙️[补全] 应该让 retries 字段真正驱动重试次数，而不是硬编码
- ⚙️[补全] 应该区分可重试异常（网络超时、连接拒绝）和不可重试异常（参数错误、权限不足），用 `retry=retry_if_exception_type(...)` 过滤
- ⚙️[补全] 熔断机制：某个工具连续失败 N 次后临时熔断，避免雪崩

#### 🔴 危险信号（主动规避）

- ❌ 别说"重试就是简单的 while 循环" → 我知道指数退避
- ❌ 别说"重试次数由元数据灵活控制" → 实际硬编码为 2
- ❌ 别说"所有错误都该重试" → 参数错误重试是浪费资源

#### 完整应答（口语稿）

> 我们的重试机制是用 tenacity 库实现的。在 ToolGateway 的 `_execute_with_retry` 方法上加了 `@retry` 装饰器，配置是：最多 2 次尝试，指数退避等待——起始 1 秒，最大 10 秒，重试耗尽后 reraise 抛出异常。这个装饰器包裹在 handler 执行外面，对工具代码完全透明。
>
> 不过当前实现有两个可以改进的地方。第一，重试次数硬编码为 2，没有读取 ToolMetadata 里定义的 retries 字段——理想情况应该是元数据驱动。第二，不区分异常类型，所有异常都重试。像网络超时这种瞬时故障重试是合理的，但参数校验失败或者权限不足这种确定性错误，重试没有意义，反而浪费资源。应该用 tenacity 的 `retry_if_exception_type` 来过滤。
>
> 另外值得一提的是，对于写操作我们额外做了一层幂等保护。ControlledExecutor 在执行前会检查 idempotency_key 是否已经执行过，防止重试导致重复写入。这个和 tenacity 的重试是互补的——tenacity 管瞬时故障的自动恢复，幂等键管写操作的 exactly-once 语义。

---

### Q10　工具调用超时怎么处理？

> 主接项目：sre-agent 的分层超时控制——SpecialistAgent 层 + evidence_fanout 层。

#### 🟢 我命中的参考答案要点

- ✅[真实] SpecialistAgent 的 `_execute_tool` 使用 `asyncio.wait_for` 控制单个工具调用超时，超时时间动态计算：`min(30, max(1, remaining_ms / 1000 / len(tool_names)))`
  → `backend/app/graph/nodes/specialist_agent.py:325-335`

- ✅[真实] SpecialistAgent 的 ReAct 循环有整体 deadline 检查，remaining_ms < 2000 时退出
  → `backend/app/graph/nodes/specialist_agent.py:167,186-192`

- ✅[真实] evidence_fanout v2 有整体 300 秒超时，超时后取消所有 pending tasks
  → `backend/app/graph/nodes/__init__.py:591-601`

- ⚙️[补全] ToolGateway 层面不做超时控制——完全依赖上层调用者的 asyncio.wait_for

#### 🟡 我能拿到的加分项

- ✅[真实] 分层超时设计：agent 层面用 deadline 强制退出、fanout 层面用固定时间窗口、单工具层面用动态分配
- ⚙️[补全] gateway 层也应该加超时保护作为兜底——如果上层忘了设超时，gateway 用 metadata.timeout_ms 兜底

#### 🔴 危险信号（主动规避）

- ❌ 别说"gateway 强制执行 timeout_ms" → gateway 层没有超时控制
- ❌ 别说"设个全局 timeout 就行" → 我知道需要分层

#### 完整应答（口语稿）

> 超时控制我们做了分层设计。最底层是单个工具调用——SpecialistAgent 的 `_execute_tool` 方法用 `asyncio.wait_for` 包裹 gateway.call_tool，超时时间是根据剩余时间和工具数量动态算的，最少 1 秒最多 30 秒。中间层是 ReAct 循环的整体 deadline——每轮开始前检查 remaining_ms，小于 2 秒就直接退出，防止无限循环。最上层是 evidence_fanout 的并行收集——整体 300 秒超时，超时后取消所有 pending 的 asyncio tasks。
>
> 有一点需要说明：ToolGateway 本身不做超时控制。虽然 ToolMetadata 里定义了 timeout_ms，但 gateway 的 call_tool 方法没有用 asyncio.wait_for 包裹。目前完全依赖上层的 SpecialistAgent 和 evidence_fanout 来做超时。这其实是一个可以加固的点——gateway 作为所有工具调用的统一入口，应该用 metadata.timeout_ms 做兜底超时，防止上层遗漏。

---

### Q11　工具调用失败后的降级策略？

> 主接项目：sre-agent 的 evidence_fanout 降级 + SpecialistAgent 4 级降级路径。

#### 🟢 我命中的参考答案要点

- ✅[真实] evidence_fanout 中区分可降级和不可降级工具：`degrade_on_failure=True` 的工具失败后返回 None 不阻塞流程，非降级工具失败则抛异常
  → `backend/app/graph/nodes/__init__.py:486-502`

- ✅[真实] 使用 `asyncio.gather(..., return_exceptions=True)` 捕获异常不中断并行任务
  → `backend/app/graph/nodes/__init__.py:504-507`

- ✅[真实] SpecialistAgent 实现 4 级降级：LLM 成功 → JSON 解析失败用原始数据 → LLM 完全失败用规则提取 → 彻底失败输出空结果
  → `backend/app/graph/nodes/specialist_agent.py:340-482`

- ✅[真实] `failed_evidence_tools` 在 state 中跟踪所有失败的工具
  → `backend/app/graph/nodes/__init__.py:77,867`

#### 🟡 我能拿到的加分项

- ✅[真实] 降级不是返回 null——每个降级层都有可用的输出（原始数据、规则提取结果）
- ✅[真实] 降级和恢复联动：critic 节点判定 NEED_MORE_EVIDENCE 时可以触发重试，给失败工具第二次机会

#### 🔴 危险信号（主动规避）

- ❌ 别说"失败就抛异常" → 我有完整的降级链
- ❌ 别搞混降级和重试 → 降级是接受部分失败继续走，重试是尝试恢复

#### 完整应答（口语稿）

> 降级策略我们做了两层。第一层在 evidence_fanout——并行收集证据的时候，每个工具任务可以标记 `degrade_on_failure`。标记了可降级的工具如果失败了，返回 None 不阻塞整体流程；不可降级的工具失败才会抛异常。所有并行任务用 `asyncio.gather` 的 `return_exceptions=True` 模式执行，单个失败不影响其他任务。
>
> 第二层在 SpecialistAgent 的分析阶段，有四级降级路径。最理想的是 LLM 成功返回结构化 JSON。如果 JSON 解析失败但原始数据还在，降级为 PARTIAL——用原始数据做规则提取。如果 LLM 完全调用失败但之前收集了 raw data，用规则做 fallback 分析。最差的情况是 LLM 失败且没有 raw data，输出空结果标记为 LLM_FAILED。每一层降级都有可用的输出，不是简单返回 null。而且降级不是终点——critic 节点评估证据质量后，如果判定 NEED_MORE_EVIDENCE，可以触发重试，给失败的工具第二次机会。

---

### Q12　响应数据太大怎么处理？截断策略？

> 主接项目：sre-agent 各 adapter 的独立截断实现——K8s 128KB、MySQL 64KB。

#### 🟢 我命中的参考答案要点

- ✅[真实] K8s adapter 定义 `RESPONSE_SIZE_LIMIT_KB = 128`（128KB），MySQL adapter 用 `MAX_BYTES = 64 * 1024`（64KB）
  → `backend/app/tools/adapters/k8s_adapter.py:10`、`backend/app/tools/adapters/mysql_adapter.py:102`

- ✅[真实] K8s adapter 采用渐进式截断：先清空 sample_logs → 再精简 pods 信息 → 最后截断 top_patterns
  → `backend/app/tools/adapters/k8s_adapter.py:36-58`

- ✅[真实] MySQL adapter 通过 while 循环 pop 尾部数据直到满足大小限制
  → `backend/app/tools/adapters/mysql_adapter.py:102-106`

- ⚙️[补全] 无全局统一的响应大小限制机制——每个 adapter 独立实现，策略不一致

#### 🟡 我能拿到的加分项

- ✅[真实] 渐进式截断优于一刀切：先丢非关键信息（sample_logs），保证关键信息（top_patterns）完整
- ⚙️[补全] 截断时应该加 truncation marker（如 `... (truncated)`），让 LLM 知道数据不完整
- ⚙️[补全] 应该提取为全局中间件统一策略，而不是每个 adapter 重复实现

#### 🔴 危险信号（主动规避）

- ❌ 别说"LLM context window 够大" → 响应大小不仅受 context window 限制，还有内存和网络开销
- ❌ 别说"直接 `str[:1000]` 截断" → 会破坏 JSON 结构

#### 完整应答（口语稿）

> 响应数据太大的问题我们是在各个 adapter 里分别处理的。K8s adapter 限制 128KB，MySQL adapter 限制 64KB。截断策略上，K8s 做得比较细致——渐进式截断。先清空最占空间但信息密度低的 sample_logs，还不够就精简 pods 的详细信息，最后才截断核心的 top_patterns。MySQL 那边简单一些，直接 while 循环 pop 尾部数据直到满足大小限制。
>
> 渐进式截断的思路是对的——先丢非关键信息，保证关键信息完整传递给 LLM。但目前每个 adapter 独立实现，策略不统一，也没有全局的截断中间件。理想情况下应该提取一个统一的响应截断层，配置化地定义每个工具的截断策略。另外截断后应该加一个 truncation marker，让 LLM 知道数据不完整，避免它基于不完整数据做出错误判断。

---

## 模块四：工具安全与风险管控

### Q13　工具的风险分级怎么设计的？

> 主接项目：sre-agent 的 RiskPolicy（动作类型 + 服务敏感度）+ harness-system 的 L0-L3 工具分级。

#### 🟢 我命中的参考答案要点

- ✅[真实] RiskPolicy 按动作类型分三级：HIGH（delete/terminate/force-stop/drop-table）、MEDIUM（restart/scale/update-config/rollback）、LOW（read/query/list/describe/get）
  → `backend/app/policies/risk.py:11-13`

- ✅[真实] 结合服务敏感度升级风险：对 CRITICAL_SERVICES（database/redis/kafka/etcd）执行 medium 动作自动升级为 HIGH
  → `backend/app/policies/risk.py:15-35`

- ✅[真实] `assess_risk` 返回四级 RiskLevel：LOW / MEDIUM / HIGH / CRITICAL
  → `backend/app/policies/risk.py:6,44-70`

- ✅[真实] Harness 系统定义了 L0-L3 四级工具分级：L0 只读、L1 本地写、L2 外部写/shell、L3 生产高风险需审批
  → `harness-system/tools/registry/index.yaml`

#### 🟡 我能拿到的加分项

- ✅[真实] 静态风险（工具定义时）和动态风险（运行时参数 + 服务上下文）结合评估
- ⚙️[补全] 风险分级应该和监控告警联动——HIGH 级工具调用次数异常时自动告警

#### 🔴 危险信号（主动规避）

- ❌ 别说"工具都是安全的" → 我有完整的分级体系
- ❌ 别说"只分读和写两级" → 我做了四级

#### 完整应答（口语稿）

> 风险分级我们做了两层。第一层是 sre-agent 的 RiskPolicy——按动作类型把操作分成三级：delete、terminate 这些是 HIGH，restart、scale 这些是 MEDIUM，read、query 这些是 LOW。然后结合服务敏感度做动态升级——如果对 database、redis、kafka 这些关键服务执行 medium 级别的操作，自动升级为 HIGH。最终输出 LOW、MEDIUM、HIGH、CRITICAL 四级。
>
> 第二层是 harness-system 的平台层工具分级，从 L0 到 L3。L0 是纯只读，默认允许；L1 是本地写，在沙箱里执行；L2 是外部写或者 shell 执行，需要权限；L3 是生产环境高风险操作，必须人工审批。两层互补——RiskPolicy 管的是"这个操作在当前上下文里有多危险"，Harness 管的是"这个工具本身的能力边界在哪"。

---

### Q14　高风险操作（如 execute_action）怎么管控？审批流程怎么设计？（★拉档题）

> 主接项目：sre-agent 的 risk_gate → approval_interrupt → ControlledExecutor 全链路。

#### 🟢 我命中的参考答案要点

- ✅[真实] execute_action 注册时标记 `risk_level="HIGH"`、`requires_approval=True`
  → `backend/app/tools/gateway.py:333-338`

- ✅[真实] risk_gate_node 做四路决策：LOW_ONLY（直接执行）/ NEEDS_APPROVAL（需审批）/ BLOCKED（拒绝）/ NEEDS_HUMAN（需人工介入）
  → `backend/app/graph/nodes/__init__.py:1249-1363`

- ✅[真实] 高危 + 生产 + 低置信度 → 直接 BLOCKED，不给审批机会
  → `backend/app/graph/nodes/__init__.py:1335-1344`

- ✅[真实] approval_interrupt_node 创建 ApprovalRequest 持久化到 DB，设置 state["pending_approval"] 挂起工作流
  → `backend/app/graph/nodes/__init__.py:1374-1527`

- ✅[真实] ControlledExecutor 三层保护：幂等检查 → 前置条件检查（service_exists / deployment_healthy）→ 执行记录 + 审计
  → `backend/app/services/executor.py:49-207`

#### 🟡 我能拿到的加分项

- ✅[真实] 不是简单的"同意/拒绝"——有条件化路由，不同风险等级走不同路径
- ✅[真实] 审批挂起后通过 LangGraph checkpoint 机制恢复——不是丢失状态
- ⚙️[补全] 审批超时处理：pending 太久应该自动拒绝或降级
- ⚙️[补全] 操作回滚：高风险操作执行前保存快照，失败时可回滚

#### 🔴 危险信号（主动规避）

- ❌ 别说"高风险操作不让 agent 碰" → 我们有完整的审批 + 执行链路
- ❌ 别说"审批就是弹个框" → 有持久化、有条件路由、有恢复机制

#### 完整应答（口语稿）

> 高风险操作的管控我们做了全链路设计。以 execute_action 为例，它在注册时就标记了 risk_level=HIGH、requires_approval=True。执行路径上有三道关卡。
>
> 第一关是 risk_gate_node，做四路决策。如果所有 action 都是 LOW 级别，直接放行。如果有审批需求或者高风险或者生产环境，路由到 NEEDS_APPROVAL。如果高危操作 + 生产环境 + 低置信度三个条件同时满足，直接 BLOCKED——这种组合太危险，连审批机会都不给。还有 NEEDS_HUMAN 路径，留给完全无法自动判断的场景。
>
> 第二关是 approval_interrupt_node。需要审批时，创建 ApprovalRequest 持久化到数据库，设置 pending_approval 状态，工作流挂起等待人工审批。审批通过后通过 LangGraph 的 checkpoint 机制恢复到原节点继续执行。
>
> 第三关是 ControlledExecutor，在执行前做三层保护：先检查 idempotency_key 是否已执行过（防重复），再检查前置条件——service 是否存在、deployment 是否健康，最后才真正执行并记录审计。这三层从"该不该做"到"能不能做"到"做了什么"，形成完整的安全闭环。

---

### Q15　生产环境的操作怎么限制？

> 主接项目：sre-agent 的 EnvPolicy——RESTRICTED_ENVS + ENV_ALLOWLIST。

#### 🟢 我命中的参考答案要点

- ✅[真实] `RESTRICTED_ENVS = {"prod", "production", "prd"}`，在这些环境中禁止 delete/drop/truncate
  → `backend/app/tools/policies/env.py:6-7,27-32`

- ✅[真实] 不同环境有操作白名单：prod 只允许 restart/scale_up/scale_down；staging 额外允许 rollback；dev 额外允许 update_config
  → `backend/app/tools/policies/env.py:9-13`

- ✅[真实] `can_execute_action` 返回 `{allowed: bool, reason: str, requires_approval: bool}` 三元组
  → `backend/app/tools/policies/env.py:24-48`

- ⚙️[补全] 当前 `_can_execute_action` 在 graph/nodes 中使用的是 `gateway.describe_capability` 做能力检查，而非直接调用 EnvPolicy——ENV_ALLOWLIST 逻辑实际未被调用

#### 🟡 我能拿到的加分项

- ⚙️[补全] 环境标识应从可信来源获取（请求参数或配置），需防范伪造
- ⚙️[补全] 灰度环境的处理策略——应该按 staging 还是 prod 对待，取决于业务容忍度

#### 🔴 危险信号（主动规避）

- ❌ 别说"生产和开发用同一套权限" → 我有环境白名单
- ❌ 别说"EnvPolicy 在生产环境全面生效" → ENV_ALLOWLIST 当前未被调用，需要整合

#### 完整应答（口语稿）

> 生产环境的操作限制通过 EnvPolicy 来管理。首先定义了受限环境集合——prod、production、prd——在这些环境里，delete、drop、truncate 这类破坏性操作直接禁止。然后不同环境有操作白名单：prod 只允许 restart、scale_up、scale_down 这些相对安全的操作；staging 额外开放 rollback；dev 最宽松，还允许 update_config。`can_execute_action` 方法返回一个三元组——是否允许、原因、是否需要审批。
>
> 不过需要坦诚说，当前代码中 `_can_execute_action` 实际使用的是 gateway 的能力检查而非 EnvPolicy 的 ENV_ALLOWLIST，这意味着白名单逻辑目前没有在生产路径上生效。这是需要整合的一个点——应该让 EnvPolicy 成为 gateway 层的前置检查，在工具执行前统一拦截。

---

### Q16　审计日志怎么记录的？

> 主接项目：sre-agent 的 `_log_audit` + `_sanitize_for_audit` + IncidentToolAudit 模型。

#### 🟢 我命中的参考答案要点

- ✅[真实] `_log_audit` 将每次工具调用记录到内存 audit_log 列表和数据库 IncidentToolAudit 表
  → `backend/app/tools/gateway.py:1084-1132`

- ✅[真实] 审计字段：audit_id、run_id、tool_name、adapter_mode、request_json、response_json、success、error_message、latency_ms、created_at
  → `backend/app/models/db_models.py:164-177`

- ✅[真实] `_sanitize_for_audit` 脱敏敏感字段：password、secret、token、api_key、access_key、authorization、cookie → "***REDACTED***"
  → `backend/app/tools/gateway.py:834-856`

- ✅[真实] 成功和失败两条路径都记录审计，在 try/finally 中完成，写入失败不影响工具调用
  → `backend/app/tools/gateway.py:1038,1055,1106-1132`

#### 🟡 我能拿到的加分项

- ✅[真实] 敏感字段递归脱敏——dict 嵌套和 list 中的敏感 field 都能覆盖
- ✅[真实] 双写策略：内存列表用于实时查询（按 run_id 过滤），数据库用于持久化和合规

#### 🔴 危险信号（主动规避）

- ❌ 别说"没有审计" → 我有完整的审计链路
- ❌ 别说"审计记录了原始参数" → 敏感字段已脱敏

#### 完整应答（口语稿）

> 审计日志是每次工具调用的标配。ToolGateway 的 `_log_audit` 方法记录十个字段——audit_id、run_id、tool_name、adapter_mode、请求参数、响应结果、成功与否、错误信息、延迟、时间戳。成功和失败两条路径都记录，放在 try/finally 里保证即使写入失败也不影响工具调用本身。
>
> 安全方面，写入审计前会做敏感字段脱敏。`_sanitize_for_audit` 函数递归扫描请求参数中的 dict 和 list，匹配到 password、secret、token、api_key 这些敏感 key 就替换为 REDACTED。存储上用了双写——内存列表支持按 run_id 实时查询，数据库持久化用于合规审计。这样既能快速排查问题，也能满足合规要求。

---

## 模块五：外部系统集成与编排

### Q17　适配器模式怎么设计的？mock 和 real 怎么切换？

> 主接项目：sre-agent 的 select_adapter + ADAPTER_MODE 环境切换。

#### 🟢 我命中的参考答案要点

- ✅[真实] 每个工具都有 mock handler（返回模拟数据）和 real handler（调用真实外部系统），通过 `ADAPTER_MODE` 环境变量全局切换
  → `backend/app/tools/gateway.py:86-87,134-197`

- ✅[真实] `select_adapter` 函数根据 mode + 工具名查找 real handler，找不到则降级为 mock 或返回错误
  → `backend/app/tools/gateway.py:134-197`

- ✅[真实] `call_tool` 流程：eval fixture 优先 → mock/real 选择 → 重试执行
  → `backend/app/tools/gateway.py:979-1063`

- ⚙️[补全] execute_action 的 real adapter 是占位实现（返回 RuntimeError），因为这是安全边界——真正的高危操作不应该有"自动执行"的 real adapter

#### 🟡 我能拿到的加分项

- ✅[真实] real adapter 底层用独立的 client 类（mysql_client / k8s_client / slb_client），adapter 负责数据转换和错误包装
- ✅[真实] eval fixture 优先机制：调试或评估场景可以注入 fixture 响应，跳过真实调用

#### 🔴 危险信号（主动规避）

- ❌ 别说"直接调 API 不需要 adapter" → mock/real 切换是开发和生产的核心需求
- ❌ 别说"execute_action 有生产可用的 real adapter" → 当前是占位实现

#### 完整应答（口语稿）

> 适配器模式的核心是让每个工具都有 mock 和 real 两套实现。通过 `ADAPTER_MODE` 环境变量全局切换——开发调试用 mock，对接真实环境用 real。`select_adapter` 函数根据 mode 和工具名查找对应的 handler：mode 是 real 且有对应 adapter 就用 real，没有就返回错误；mode 是 mock 就用 mock handler。
>
> real adapter 的封装分了层——底层是独立的 client 类，比如 mysql_client、k8s_client、slb_client，负责和外部系统的原始通信；adapter 层负责数据转换、错误包装和响应截断。另外还有一个 eval fixture 机制，调试时可以注入预设响应，完全跳过真实调用。
>
> 有一点需要说明：execute_action 这个高危工具的 real adapter 目前是占位实现，直接返回 RuntimeError。这不是没做完——这是有意为之的安全边界。真正的高危操作不应该有"自动执行"的 real adapter，它必须走审批 + 人工确认的路径。

---

### Q18　多个工具并行调用怎么实现？

> 主接项目：sre-agent 的 asyncio.gather 并行——两个维度。

#### 🟢 我命中的参考答案要点

- ✅[真实] evidence_fanout 使用 `asyncio.gather(*tasks, return_exceptions=True)` 并行执行多个 investigation tasks
  → `backend/app/graph/nodes/__init__.py:504-507`

- ✅[真实] evidence_fanout v2 使用 `asyncio.wait(tasks_map.keys(), timeout=300.0)` 并行执行多个 SpecialistAgent
  → `backend/app/graph/nodes/__init__.py:586-593`

- ✅[真实] SpecialistAgent 内部也使用 `asyncio.gather` 并行执行同一轮内的多个 tool_calls
  → `backend/app/graph/nodes/specialist_agent.py:241-244`

- ✅[真实] 超时后取消所有 pending tasks
  → `backend/app/graph/nodes/__init__.py:595-601`

#### 🟡 我能拿到的加分项

- ✅[真实] `return_exceptions=True` 保证单个工具失败不拖垮并行组
- ⚙️[补全] 如果工具之间有依赖关系，需要分轮次执行而非无脑并行——当前是通过 ReAct 循环自然处理的

#### 🔴 危险信号（主动规避）

- ❌ 别说"并行就是用多线程" → 我们是 asyncio 协程
- ❌ 别说"不做超时控制" → 有 300s 超时 + cancel pending

#### 完整应答（口语稿）

> 并行调用我们做了两个维度。第一个维度是 evidence_fanout——多个 SpecialistAgent 并行收集不同类别的证据，用 `asyncio.wait` 管理，整体 300 秒超时，超时后取消所有 pending 任务。第二个维度是单个 SpecialistAgent 内部——同一轮 LLM 返回的多个 tool_calls 用 `asyncio.gather` 并行执行。
>
> 关键设计点是 `return_exceptions=True`——单个工具失败不会拖垮整个并行组，结果里混合成功和失败，后续节点再根据 failed_evidence_tools 决定降级还是重试。如果工具之间有依赖关系——比如先查 deployment 状态再查 pod 日志——这种不会硬并行，而是通过 ReAct 循环自然分轮次：第一轮查 deployment，第二轮根据结果查 pod。

---

### Q19　Agent 的 ReAct 循环中工具调用怎么编排？

> 主接项目：sre-agent 的 SpecialistAgent.run()——标准 ReAct + 工具白名单 + 安全过滤。

#### 🟢 我命中的参考答案要点

- ✅[真实] 标准 ReAct 循环：`for round_num in range(max_tool_rounds)` → LLM 调用（带 tools）→ 无 tool_calls 则 break → 并行执行工具 → 追加 tool role message → 下一轮
  → `backend/app/graph/nodes/specialist_agent.py:184-258`

- ✅[真实] 全局禁止 execute_action（FORBIDDEN_TOOLS），防止查询 agent 误执行写入操作
  → `backend/app/graph/nodes/specialist_agent.py:33`

- ✅[真实] `_tool_allowed` 验证 LLM 请求的工具是否在白名单中，不在则注入错误消息让 LLM 修正
  → `backend/app/graph/nodes/specialist_agent.py:297-311`

- ✅[真实] 每轮有时间预算检查（deadline），超时即退出
  → `backend/app/graph/nodes/specialist_agent.py:186-192`

#### 🟡 我能拿到的加分项

- ✅[真实] 工具过滤的安全意义——agent 只能看到它应该用的工具，K8s agent 看不到 DB 工具
- ✅[真实] max_tool_rounds 防止无限循环——ReAct agent 的经典陷阱

#### 🔴 危险信号（主动规避）

- ❌ 别说"ReAct 就是 while True" → 有 round 限制和超时
- ❌ 别说"所有工具对所有 agent 开放" → 有白名单和黑名单

#### 完整应答（口语稿）

> SpecialistAgent 的 ReAct 循环是标准的 Thought-Action-Observation 模式。循环最多 max_tool_rounds 轮，每轮先调 LLM——带上当前 agent 的工具白名单转换成的 tools 数组。LLM 返回后，如果没有 tool_calls 就 break 退出循环，进入最终分析。有 tool_calls 的话，先校验每个 tool_call 是否在白名单内——不在的话注入错误消息让 LLM 修正——合法的 tool_calls 用 asyncio.gather 并行执行，结果按 tool role 格式追加回 messages，进入下一轮。
>
> 安全方面做了两层过滤。第一层是全局黑名单——FORBIDDEN_TOOLS 里放了 execute_action，所有 agent 都不能调。第二层是每个 agent 的工具白名单——K8s agent 只能调 query_k8s_* 系列，DB agent 只能调 query_db_* 系列，LLM 请求不在白名单的工具会被拒绝。这样即使 LLM "幻觉"出一个不该调的工具，也会被拦截。另外每轮有 deadline 检查，超时就退出，防止无限循环。

---

### Q20　LangGraph 图编排中工具节点怎么嵌入？（★拉档题）

> 主接项目：sre-agent 的 13 节点 LangGraph 工作流——工具节点 + 条件路由 + checkpoint 恢复。

#### 🟢 我命中的参考答案要点

- ✅[真实] 工作流 13 个节点：intake → triage → retrieve_memory → planner → evidence_fanout → evidence_aggregate → diagnose → critic → remediation → risk_gate → (approval_interrupt | executor) → verify_outcome → rca
  → `backend/app/graph/builder.py:177-206`

- ✅[真实] 工具调用分布在多个节点：evidence_fanout（并行收集）、executor（经审批后执行）、verify_outcome（验证效果）、rca（写 OSS），全部通过 gateway.call_tool() 统一中转
  → `backend/app/graph/nodes/__init__.py`：evidence_fanout L422-646、executor_node L1530-1647、verify_outcome_node L1649-1822、rca_node L1825-2204

- ✅[真实] 条件路由：critic → {evidence_fanout / planner / remediation / rca}，risk_gate → {executor / approval_interrupt / rca}，verify → {executor (retry) / rca}
  → `backend/app/graph/builder.py:115-143,227-269`

- ✅[真实] dispatcher 节点支持从任意节点 resume——审批恢复和生产调试的关键能力
  → `backend/app/graph/builder.py:92-112`

#### 🟡 我能拿到的加分项

- ✅[真实] gateway 作为 "tool plane"——将重试、审计、schema 校验、超时等横切关注点从业务逻辑中分离
- ✅[真实] 条件路由的设计考量：什么时候重试、什么时候降级、什么时候直接结束——都有明确的判定逻辑
- ✅[真实] LangGraph checkpoint 机制支持审批中断后恢复——不是丢失状态
- ⚙️[补全] 为什么用图编排而不是全交给 agent 自主决定——可观测性、可控性、审批介入，这是生产级 agent 和 demo 的核心区别

#### 🔴 危险信号（主动规避）

- ❌ 别说"一个 agent 就能搞定" → 生产级需要可观测性和可控性
- ❌ 别搞混节点和工具 → 每个节点调用哪些工具要能说清

#### 完整应答（口语稿）

> 我们的 SRE agent 用 LangGraph 做了 13 个节点的有向图编排。工具调用不是集中在一个节点里，而是分布在四个关键节点中。evidence_fanout 负责并行收集 K8s 状态、metrics、logs 等证据——这是信息收集阶段。executor 在经过 risk_gate 审批后执行修复操作——这是行动阶段。verify_outcome 在执行后验证效果——查 deployment 状态、LB 健康度、流量指标。rca 最后把诊断结论和证据写到 OSS——这是输出阶段。
>
> 所有工具调用统一走 ToolGateway.call_tool()，图节点不直接调 adapter。这样做的好处是 gateway 作为一个 "tool plane"，把重试、审计、schema 校验、超时这些横切关注点从业务逻辑里抽出来了。
>
> 条件路由是图编排的核心价值。critic 节点评估证据质量后，可以路由回 evidence_fanout 补充证据、或者继续 remediation、或者直接写 RCA。risk_gate 根据风险等级决定是直接执行、走审批、还是直接拒绝。verify 验证失败可以路由回 executor 重试。这些决策逻辑如果在纯 agent 循环里做，可观测性和可控性都会差很多。另外 LangGraph 的 checkpoint 机制让我们支持审批中断后恢复——审批通过后可以从原节点继续，状态不丢失。dispatcher 节点甚至支持从任意节点 resume，这在生产调试中非常实用。

---

> 应答文档完。共 5 个模块，20 题，每题为四部分（🟢命中要点 / 🟡加分项 / 🔴规避 / 完整应答）。
