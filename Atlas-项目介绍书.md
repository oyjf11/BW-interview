# Atlas · AI 应用生成平台

> **一句话介绍**：用户用自然语言描述需求，平台通过多 Agent 协同自动生成、预览、迭代并一键部署真实可用的 Web 应用——我是独立开发者，从零搭建了完整的前后端、Agent 内核、沙箱执行与部署闭环。

---

## 1. 项目概述

Atlas 是一个对话驱动的 AI 应用生成平台。无需编写代码，通过自然语言对话 + 实时预览 + 可视化微调，即可得到一个结构正确、可运行、可持续迭代的应用。

**当前产品形态**：左侧对话区接收用户意图与迭代指令，右侧在「实时预览」和「代码/文件视图」之间切换。预览区还承载可视化微调层——用户直接点选页面元素修改文案、颜色、图片，无需走对话。

**核心能力**（已全部跑通）：

| 能力 | 说明 |
|------|------|
| 对话生成应用 | 自然语言 → 可运行页面，支持轻应用/营销页和管理后台 |
| 多轮增量迭代 | 基于局部 diff 的增量修改，每轮都有可见产出 |
| 多 Agent 协同 | LangGraph 编排，Orchestrator + SubAgent 分层协作 |
| 自我修正闭环 | 构建失败 → 自动分析报错 → 生成修复 diff → 重试验证 |
| 可视化微调 | 预览中点选元素，直接修改样式/文案 |
| 一键部署 | 构建产物 → OSS → 线上可访问 |

**核心技术栈**：

| 层级 | 技术 | 
|------|------|
| 平台前端 | Next.js（全栈框架） |
| Agent 编排 | LangGraph |
| 大模型（主） | DeepSeek V4 — 负责规划与复杂生成 |
| 大模型（辅） | Qwen2.5 — 负责高频低难度任务（摘要、分类、简单 diff） |
| 执行沙箱 | Sandpack（浏览器内即时预览） |
| BaaS / 数据库 | Supabase（开源版）+ PostgreSQL |
| 产物存储 | OSS（对象存储） |
| 实时通信 | WebSocket（token 级流式输出） |

---

## 2. 核心技术架构——一次请求的完整旅程

### 2.1 分层架构

```
┌──────────────┐
│   接入层      │  WebSocket 长连接 · 鉴权 · 限流 · SSO
├──────────────┤
│  编排层       │  LangGraph 状态机 · 任务调度 · 流式响应
├──────────────┤
│  Agent 内核   │  上下文组装 · 规划 · 工具调用 · 记忆 · 自修正
├──────────────┤
│  执行沙箱层   │  Sandpack 浏览器沙箱 · 构建 · 预览
├──────────────┤
│  数据与存储   │  PostgreSQL · OSS · Redis · 向量索引
├──────────────┤
│  模型层       │  DS V4 / Qwen2.5 · 分级路由 · 缓存 · 降级
└──────────────┘
```

### 2.2 一次迭代的完整数据流

以「用户说"把导航栏改成深色"」为例，走一遍完整链路：

**Step 1 — 接入**
WebSocket 消息到达，解析 session_id、user_id，通过 Redis 恢复会话状态机，投递到 LangGraph 编排引擎。

**Step 2 — 编排层启动 Graph**
编排层创建一个新的 LangGraph run，初始状态对象包含：
```typescript
{
  session_id: string;
  user_message: "把导航栏改成深色";
  project_id: string;
  // 项目认知将在 Agent 节点中从沙箱实时重建
}
```

**Step 3 — Agent 规划节点（Plan Node）**
- 调用 `rebuild_project_context()` 从 Sandpack 沙箱拉取当前工程的文件树 + 每个文件的导出符号摘要
- 组装四层上下文（系统层 + 项目层 + 任务层 + 检索层）
- 调用 DS V4 生成高层修改计划：
  ```json
  {
    "plan": "修改导航栏背景色为深色",
    "files_to_modify": ["components/Navbar.tsx", "tailwind.config.ts"],
    "steps": [
      {"action": "read", "target": "components/Navbar.tsx"},
      {"action": "replace", "target": "components/Navbar.tsx", "description": "修改背景色 class"},
      {"action": "replace", "target": "tailwind.config.ts", "description": "如需新增颜色 token"}
    ]
  }
  ```
- 规划结果写入状态对象，进入下一个 node。

**Step 4 — Agent 执行节点（Execute Node）**
进入 ReAct 循环：
1. **Think**：模型根据当前上下文 + 计划，决定下一步调用哪个工具
2. **Act**：调用工具——先 `read_file("components/Navbar.tsx")` 拿到精确内容
3. **Observe**：工具返回文件全文，注入上下文
4. **Think**：模型定位到 `bg-white` / `bg-gray-50` 等 class，生成 str_replace diff
5. **Act**：调用 `str_replace(file, old_str, new_str)` 精确替换
6. **Observe**：返回替换成功

**Step 5 — 验证节点（Verify Node）**
- 将修改后的文件内容注入 Sandpack
- 触发 `sandpack.build()` → 等待构建完成
- 构建成功 → 状态标记 `verification: "passed"` → 流式推送预览更新
- 构建失败 → 进入自我修正子图（详见第 4 节）

**Step 6 — 响应用户**
- 流式返回：计划摘要 → 执行进度 → 最终结果 → 预览 URL

### 2.3 关键设计原则

**代码即真相**。项目的真实状态永远以 Sandpack 沙箱中的工程文件为准。Agent 每轮行动前从沙箱重建项目认知，不依赖几十轮前的对话记忆。具体做法：每次 Agent 节点启动时，调用 `SandpackClient.readFileTree()` 拉取完整文件树 + 每个文件的导出摘要，构建一个结构化的项目快照注入 prompt。从根本上规避"以为某组件还在、其实早被删了"的状态幻觉。

**验证闭环优先**。模型产出 != 任务完成。每次 diff 写入后必须经过构建验证，失败则自动进入修正循环。这是生成质量的命门。

**增量优于重写**。迭代阶段一律走局部 str_replace diff。每次成功的 diff 进入版本历史，支持任意回滚——这在"越改越乱"时至关重要。

**上下文隔离**。子 Agent 各自在隔离上下文中工作，仅回传结构化结论，避免上下文互相污染。

**成本与质量分级**。DS V4 处理规划与复杂生成，Qwen2.5 处理摘要、分类、简单 diff，把昂贵算力用在刀刃上。

---

## 3. Agent 内核——怎么让模型可控地写代码

### 3.1 运行模型：Plan-and-Execute + ReAct 的 LangGraph 状态机

采用 **Plan-and-Execute 与 ReAct 相结合**的运行模型。规划阶段产出一个高层修改计划（要动哪些文件、什么顺序），执行阶段逐步实施，每一步遵循"思考 → 调用工具 → 观察结果"的 ReAct 循环。

**为什么不纯用单步 ReAct？** 应用生成涉及多文件、多步骤的协调，先规划让执行更有条理、更可控、也更容易向用户解释正在做什么。

**为什么不纯用静态计划？** 执行中常遇到预期外的报错（比如依赖冲突、文件冲突），需要 ReAct 式的动态调整。

**LangGraph 状态机设计**：

```
                    ┌──────────┐
         START ───▶ │  Plan    │ ───▶ plan ready
                    └──────────┘
                          │
                          ▼
                    ┌──────────┐
           ┌────── │ Execute  │ ◀──────────┐
           │       └──────────┘            │
           │            │                  │
           │     tool_call / done          │
           │            │                  │
           │    ┌───────┴───────┐          │
           │    ▼               ▼          │
           │ ┌──────┐    ┌──────────┐      │
           │ │ Tool │    │ Verify   │      │
           │ │ Node │    └──────────┘      │
           │ └──────┘         │            │
           │       ▲     pass │  fail      │
           │       │          │    │       │
           │       └──────────┘    ▼       │
           │                  ┌────────┐   │
           │                  │ Fix    │───┘
           │                  └────────┘
           │                       │
           ▼                       ▼
     ┌──────────┐           max retries
     │ Respond  │               exceeded
     └──────────┘
          │
          ▼
         END
```

**状态对象结构**：

```typescript
interface AgentState {
  // 会话标识
  session_id: string;
  project_id: string;

  // 当前轮次
  user_message: string;
  clarified_intent?: string;        // 经过意图澄清后的精确需求

  // 计划阶段产出
  plan?: {
    summary: string;                 // 给用户看的一句话概述
    files: string[];                 // 涉及的文件列表
    steps: PlanStep[];               // 执行步骤序列
  };

  // 执行阶段
  current_step_index: number;
  execution_log: ExecutionEntry[];   // 每步的工具调用 + 结果

  // 项目认知（每轮重建）
  project_snapshot?: {
    file_tree: FileTreeNode[];
    symbols: Record<string, string[]>;  // file → [export names]
    design_tokens: DesignTokens;
  };

  // 验证
  verification_result?: "pass" | "fail";
  verification_errors?: BuildError[];

  // 自修正
  fix_attempts: number;
  max_fix_attempts: number;          // 默认 3

  // 响应
  response_to_user?: string;
}
```

**LangGraph node 划分**：
- `plan_node`：组装上下文 → 调 DS V4 → 产出计划 → 写入 state.plan
- `execute_node`：ReAct 循环——调模型决策下一步 action → 调 tool_node → 观察结果 → 继续或完成
- `tool_node`：解析 function_call → 路由到 handler → 执行 → 将结果追加到 state.execution_log
- `verify_node`：应用修改到 Sandpack → 触发构建 → 读取结果 → 设置 state.verification_result
- `fix_node`：组装报错上下文 → 调 DS V4 生成修复 diff → 增量重试 → 更新 state.fix_attempts
- `respond_node`：组装最终响应 → 流式推送给用户

**条件边（conditional edges）**：
- Plan → Execute：plan 生成成功 → 进入执行
- Execute → Tool：模型输出 function_call → 路由到工具节点
- Execute → Verify：模型输出 finish → 进入验证
- Verify → Respond：验证通过
- Verify → Fix：验证失败且 fix_attempts < max → 进入修正
- Fix → Execute：修正 diff 生成后 → 回到执行循环
- Fix → Respond：达到最大重试次数 → 降级响应用户

### 3.2 上下文工程

上下文按**四层动态组装**，而非每轮全量塞入。核心思路：先让模型看结构，按需再给详细内容。

#### 3.2.1 四层上下文结构

**系统层**（常驻，约 800-1200 tokens）：
```
你是一个应用生成 Agent，运行在 Atlas 平台上。
- 技术栈：Next.js 14 + TypeScript + Tailwind CSS
- 输出格式：所有代码变更必须以 str_replace 格式输出，格式为：
  <<<FILE:path/to/file>>>
  <<<OLD>>>
  exact old content
  <<<NEW>>>
  replacement content
- 设计规范遵循项目的 design_tokens（颜色、间距、字体）
- 修改前必须先读取目标文件全文
- 每次只修改最小必要范围，避免大段重写
```

**项目层**（每轮从沙箱重建，约 1500-3000 tokens）：
```markdown
## 当前工程结构
src/
├── app/
│   ├── layout.tsx          — 根布局，包含 Navbar + Footer
│   └── page.tsx            — 首页
├── components/
│   ├── Navbar.tsx          — 顶部导航 [exports: Navbar]
│   ├── Footer.tsx          — 页脚 [exports: Footer]
│   ├── Button.tsx          — 通用按钮 [exports: Button, ButtonProps]
│   └── Card.tsx            — 卡片组件 [exports: Card, CardProps]
├── lib/
│   ├── db.ts               — Supabase 客户端
│   └── utils.ts            — 工具函数 [exports: cn, formatDate]
└── styles/
    └── globals.css

## 设计规范
- 主色: #3B82F6 (blue-500)
- 辅色: #10B981 (emerald-500)
- 背景: #FFFFFF / #F9FAFB
- 字体: Inter, system-ui
- 圆角: 8px (默认), 12px (卡片)

## 路由表
| 路径 | 页面 | 说明 |
| / | HomePage | 首页 |
| /login | LoginPage | 登录 |
| /dashboard | DashboardPage | 仪表盘 |
```

注意：项目层**不包含文件全文**——只提供文件树 + 职责摘要。模型先看结构，决定要改哪个文件，再通过 `read_file` 工具按需读取。

**任务层**（当前轮次，约 200-500 tokens）：
```markdown
## 用户需求
"把导航栏改成深色"

## 意图澄清结果
用户希望将顶部导航栏的背景色从浅色改为深色（dark theme navbar），
Logo 和文字颜色需要相应调整为浅色以保证可读性。
```

**检索层**（按需召回，约 500-1500 tokens）：
当用户提到特定组件名、功能名时，通过代码语义检索从向量索引中召回相关代码片段，附加到上下文中：
```markdown
## 相关代码（语义检索命中）
### components/Navbar.tsx (相似度 0.94)
[该文件的前 80 行代码...]
```

#### 3.2.2 上下文裁剪

上下文裁剪是控制 token 成本和质量的核心手段。三个层面：

**A. 文件树的摘要压缩**

完整源码全部塞入上下文既不现实也没必要。实际做法：
1. 用 tree-sitter 解析每个文件的 AST
2. 提取导出符号（函数名、组件名、interface 名）和第一行注释
3. 生成压缩摘要：`Navbar.tsx — 顶部导航组件 [exports: Navbar, NavbarProps] // 响应式导航栏，支持移动端汉堡菜单`
4. 仅当模型调用 `read_file` 工具时，才将该文件的完整源码注入上下文

压缩比约 10:1 到 20:1（一个 500 行的文件压缩为 1 行摘要）。

**B. 对话的滚动摘要**

当对话超过 N 轮（当前设为 15 轮），早期轮次被压缩为结构化摘要：

```markdown
## 历史摘要（第 1-8 轮）
- 创建了项目脚手架（Next.js + Tailwind）
- 实现了首页 Hero 区域和特性展示区
- 用户要求将 Hero 的 CTA 按钮从"了解更多"改为"立即开始"
- 添加了 Footer 组件（含社交媒体链接）
- 引入了 Supabase 用于联系表单提交
- 待办：用户提到后续需要多语言支持
```

仅保留最近 15 轮的完整对话原文。摘要由 Qwen2.5 生成（成本低、速度快）。

**C. Token 预算分配**

每次上下文组装时，预设计 token 预算：

| 层级 | 预算 | 说明 |
|------|------|------|
| 系统层 | 1500 | 固定不变 |
| 项目层 | 3000 | 随工程规模动态调整 |
| 任务层 | 1000 | 当前轮次 + 最近 N 轮原文 |
| 检索层 | 2000 | 语义召回上限 |
| 历史摘要 | 1500 | 超出 N 轮的压缩摘要 |
| 模型响应 | 4000 | 预留输出空间 |
| **总计** | **~13,000** | 控制在 DS V4 的舒适区内 |

当上下文接近预算上限时触发裁剪——优先压缩历史对话，其次压缩项目层（减少文件树深度），最后才考虑牺牲检索层精度。裁剪时机在每次 `plan_node` 启动前由 `context_manager.trim_if_needed()` 执行。

### 3.3 工具调用系统

#### 3.3.1 工具注册与接口定义

每个工具以统一 schema 注册：

```typescript
interface ToolDefinition {
  name: string;
  description: string;          // 给模型看的，决定何时调用
  parameters: JSONSchema;       // OpenAI function calling 兼容格式
  handler: (params: any, ctx: ToolContext) => Promise<ToolResult>;
  category: "file" | "execution" | "retrieval" | "external";
  requires_confirmation?: boolean; // 写操作可选确认
}
```

已注册的工具清单（精简版）：

```typescript
const TOOLS: ToolDefinition[] = [
  // 📁 文件操作
  {
    name: "read_file",
    description: "读取指定文件的完整内容。在修改任何文件前必须先读取。",
    parameters: { path: { type: "string", description: "文件相对路径" } },
    category: "file",
  },
  {
    name: "str_replace",
    description: "精确替换文件中的一段文本。old_str 必须与文件中的内容完全匹配（包括缩进和空格）。",
    parameters: {
      path: { type: "string" },
      old_str: { type: "string", description: "要替换的原文，必须精确匹配" },
      new_str: { type: "string", description: "替换后的新文本" },
    },
    category: "file",
    requires_confirmation: true,
  },
  {
    name: "write_file",
    description: "创建新文件或完全覆盖已有文件",
    parameters: {
      path: { type: "string" },
      content: { type: "string" },
    },
    category: "file",
    requires_confirmation: true,
  },
  {
    name: "list_dir",
    description: "列出目录下的文件和子目录",
    parameters: { path: { type: "string" } },
    category: "file",
  },

  // ⚙️ 构建验证
  {
    name: "run_build",
    description: "在 Sandpack 沙箱中执行构建，返回构建结果和错误信息",
    parameters: {},
    category: "execution",
  },
  {
    name: "get_build_errors",
    description: "获取最近一次构建的详细错误列表，包含文件路径和行号",
    parameters: {},
    category: "execution",
  },

  // 🔍 代码检索
  {
    name: "search_code",
    description: "语义搜索代码库，通过自然语言描述查找相关代码片段",
    parameters: {
      query: { type: "string", description: "自然语言搜索描述" },
      top_k: { type: "number", default: 5 },
    },
    category: "retrieval",
  },
  {
    name: "find_symbol",
    description: "通过符号名精确查找定义位置（组件、函数、变量）",
    parameters: { name: { type: "string" } },
    category: "retrieval",
  },

  // 🌐 外部服务
  {
    name: "web_search",
    description: "搜索 Web 获取最新文档、API 用法或解决方案",
    parameters: {
      query: { type: "string" },
      max_results: { type: "number", default: 3 },
    },
    category: "external",
  },
  {
    name: "supabase_query",
    description: "对 Supabase 数据库执行查询操作",
    parameters: {
      table: { type: "string" },
      operation: { type: "string", enum: ["select", "insert", "update", "delete"] },
      data: { type: "object" },
    },
    category: "external",
    requires_confirmation: true,
  },
];
```

#### 3.3.2 Tool Calling 全流程

一次完整的工具调用链路：

```
1. 模型输出 function_call
   ↓
   {
     "id": "call_abc123",
     "function": {
       "name": "read_file",
       "arguments": '{"path": "components/Navbar.tsx"}'
     }
   }

2. ToolRouter 解析
   ↓
   - 校验 function.name 是否在注册表中
   - 用 JSON Schema 校验 arguments
   - 检查 requires_confirmation（若需要则挂起等用户确认）

3. 路由到 Handler
   ↓
   ToolRegistry.dispatch("read_file", { path: "components/Navbar.tsx" }, ctx)
   → ReadFileHandler.execute()
   → 从 Sandpack 沙箱拉取文件内容
   → 返回: { success: true, data: { content: "...", lines: 89 } }

4. 结果回灌 Prompt
   ↓
   将 tool result 序列化为消息追加到对话中：
   {
     role: "tool",
     tool_call_id: "call_abc123",
     content: "[文件内容](truncated if > 500 lines)..."
   }

5. 模型继续推理
   ↓
   模型基于文件内容决定下一步 action——
   可能是再次 function_call（执行修改）、或 finish（完成）
```

**并行工具调用**：当模型一次返回多个 function_call（如同时读取 Navbar.tsx 和 globals.css），LangGraph 的 `tool_node` 并行 dispatch 到对应的 handler，用 `Promise.all` 收集结果后统一回灌。

#### 3.3.3 让模型稳定输出精确 str_replace

`str_replace` 容易出两种错：old_str 找不到（拼写/缩进不完全匹配）、或匹配到多处。对策：

**A. 强制「先读后写」**：prompt 中约束模型修改任何文件前必须先 `read_file`，且 prompt 中附带了读取时间戳，防止使用过期缓存。

**B. old_str 含足够上下文**：约束模型提供的 old_str 必须包含被替换块的前后各 1-2 行不变代码作为锚点，确保唯一性。

**C. 服务端校验**：`str_replace` handler 执行时：
```typescript
async function str_replace(params: StrReplaceParams): Promise<ToolResult> {
  const fileContent = await sandpack.readFile(params.path);
  
  // 1. 精确匹配
  const occurrences = countOccurrences(fileContent, params.old_str);
  if (occurrences === 0) {
    return { success: false, error: `old_str not found in ${params.path}` };
  }
  if (occurrences > 1) {
    return { success: false, error: `old_str matched ${occurrences} times, please provide more context` };
  }
  
  // 2. 执行替换
  const newContent = fileContent.replace(params.old_str, params.new_str);
  await sandpack.writeFile(params.path, newContent);
  
  // 3. 记录版本
  await versionHistory.push({ file: params.path, old: params.old_str, new: params.new_str });
  
  return { success: true, data: { path: params.path, lines_changed: countDiffLines(params.old_str, params.new_str) } };
}
```

#### 3.3.4 工具调用的错误处理与重试

工具执行失败分三级处理：

| 级别 | 场景 | 处理 |
|------|------|------|
| L1：可自动修复 | old_str 匹配失败、文件不存在 | 将精确错误信息返回模型，让模型重新生成调用参数 |
| L2：需降级处理 | Sandpack 超时、Supabase 连接失败 | 重试 3 次 + 指数退避，仍失败则标记降级，告知用户 |
| L3：不可恢复 | 沙箱崩溃、存储不可用 | 中断当前图运行，通过 WebSocket 推送错误，引导用户重试 |

L1 处理示例——当 `str_replace` 返回 "not found" 时，错误直接被灌入模型的下一轮 ReAct 循环，模型根据错误信息调整 old_str。这实际上构成了一层内层修正回路，比外层的构建验证修正更轻量。

### 3.4 记忆系统

严格区分三类记忆，因为它们的生命周期完全不同。

#### 工作记忆（会话级 · Redis）

当前会话本轮迭代的来龙去脉。通过滚动摘要管理：早期轮次压缩为结构化摘要（Qwen2.5 生成），仅保留最近 15 轮完整对话。存于 Redis，TTL 跟随会话。

```
Key: session:{session_id}:working_memory
Value: {
  recent_turns: [...],      // 最近 15 轮完整对话
  summary: "第 1-8 轮摘要...", // 早期轮次的结构化压缩
  project_snapshot_hash: "abc123"  // 用于快速判断工程是否变化
}
```

#### 项目记忆（从沙箱实时派生）

这是最关键也最容易做错的一层。核心原则：**代码即真相——能从沙箱读到的，绝不存到记忆里**。

每次 Agent 行动前，从 Sandpack 沙箱重建项目认知：
```typescript
async function rebuild_project_context(projectId: string): Promise<ProjectSnapshot> {
  // 1. 从 Sandpack 拉取完整文件树
  const fileTree = await sandpack.listFiles(projectId);
  
  // 2. 对每个 .ts/.tsx/.css 文件提取导出符号和一行摘要
  const symbols = {};
  for (const file of fileTree) {
    if (isSourceFile(file)) {
      symbols[file.path] = await extractExports(file.content);  // tree-sitter 解析
    }
  }
  
  // 3. 提取设计 token（从 tailwind.config.ts 解析）
  const designTokens = await extractDesignTokens(fileTree);
  
  return { fileTree, symbols, designTokens, snapshotTime: Date.now() };
}
```

**为什么不依赖对话历史记忆项目状态？** 因为对话历史中「我以为我创建了 Navbar」不能证明沙箱里 Navbar 还存在——可能被后续某轮删了、被另一个 SubAgent 覆盖了、或者生成失败了。代码即真相从根本上消除了这类幻觉。

#### 长期/用户记忆（PostgreSQL）

跨项目的用户偏好，需用户授权，做成显式配置 + 从行为提炼的画像：

```sql
-- 用户偏好表
CREATE TABLE user_preferences (
  user_id UUID PRIMARY KEY,
  preferred_stack TEXT[],        -- ['nextjs', 'tailwind', 'typescript']
  code_style JSONB,             -- { indent: 2, quotes: 'single', semi: true }
  ui_preferences JSONB,         -- { color_scheme: 'light', font: 'Inter' }
  updated_at TIMESTAMP
);
```

偏好通过分析用户的历史接受/拒绝行为来更新——如果用户在 10 个项目中都选了 dark theme，下次默认就是 dark。

### 3.5 多 Agent 协同

#### 3.5.1 LangGraph 的 SubGraph 机制

采用 **Orchestrator + SubAgent** 的分层模式。一个主 Graph（Orchestrator）维护全局状态与任务队列，将可独立的子任务分发给在隔离上下文中工作的 SubGraph（SubAgent）。

```typescript
// 主 Graph 中定义子图调用
const orchestratorGraph = new StateGraph(MainState)
  .addNode("plan", planNode)
  .addNode("delegate", delegateNode)     // 分派子任务
  .addNode("merge", mergeNode)           // 合并子结果
  .addNode("verify", verifyNode)
  // ...
  .addEdge("plan", "delegate")
  .addConditionalEdges("delegate", routeAfterDelegation);

// SubGraph 独立定义
const frontendSubGraph = new StateGraph(SubState)
  .addNode("plan_sub", subPlanNode)       // 子 Agent 自己的规划
  .addNode("execute_sub", subExecuteNode) // 子 Agent 自己的 ReAct 循环
  .addNode("verify_sub", subVerifyNode)
  // ...

// 注册到主 Graph
orchestratorGraph.addSubgraph("frontend_builder", frontendSubGraph);
```

#### 3.5.2 通信协议：Orchestrator → SubAgent 传什么

Orchestrator 不传原始对话，而是传**结构化任务指令 + 接口契约**：

```typescript
interface SubTask {
  task_id: string;
  type: "frontend_component" | "api_route" | "db_schema" | "integration";
  instruction: string;          // 自然语言任务描述
  context: {
    design_tokens: DesignTokens; // 来自项目层的设计规范
    interface_contract?: {       // 与其他模块的接口约定
      props?: Record<string, string>;
      api_endpoint?: string;
      db_schema?: string;
    };
  };
  files_in_scope: string[];     // 此子任务允许修改的文件白名单
}
```

SubAgent 完成后回传结构化结论，而不是大段对话：

```typescript
interface SubTaskResult {
  task_id: string;
  status: "success" | "failed" | "partial";
  changes: {
    files_modified: string[];
    files_created: string[];
    diff_summary: string;       // 一句话概述做了什么
  };
  interface_updated: {          // 更新后的接口契约
    exports: string[];          // 新增/修改的导出
    props_used?: Record<string, string>;
  };
  warnings: string[];           // 需要注意的问题
}
```

**为什么不传原始对话？** 这是多 Agent 上下文隔离的核心价值——每个 SubAgent 的对话上下文聚焦且干净，不包含其他 Agent 的试错过程、无关的推理链、或污染性的信息。Orchestrator 只关心「什么变了」和「接口变了没」。

#### 3.5.3 文件锁与并发控制

多个 SubAgent 并发修改代码时，通过**文件级写入锁**避免冲突：

```typescript
const fileLocks = new Map<string, string>(); // filePath → taskId

async function acquireLock(filePath: string, taskId: string, timeoutMs = 30000): Promise<boolean> {
  const start = Date.now();
  while (fileLocks.has(filePath)) {
    if (Date.now() - start > timeoutMs) return false;
    await sleep(100);
  }
  fileLocks.set(filePath, taskId);
  return true;
}

function releaseLock(filePath: string, taskId: string) {
  if (fileLocks.get(filePath) === taskId) {
    fileLocks.delete(filePath);
  }
}
```

Orchestrator 在分派子任务时，如果两个子任务的 `files_in_scope` 有交集，则**串行执行**而非并行。这是保守策略——牺牲一点并行度换取状态正确性。

#### 3.5.4 实际场景示例

用户说：「做一个商城首页，有商品列表和购物车，数据从 Supabase 拿」

Orchestrator 分解为三个子任务：
1. **Frontend Builder**——实现首页 UI、商品列表组件、购物车组件（files_in_scope: `components/*.tsx`, `app/page.tsx`）
2. **DB Schema Builder**——创建商品表和购物车表（files_in_scope: `lib/db.ts`, `supabase/migrations/*.sql`）
3. **API Integrator**——对接 Supabase SDK，实现数据获取和购物车操作（files_in_scope: `lib/api.ts`, `app/api/*`）

三个子任务的 files_in_scope 无交集 → 可并行执行。每个 SubAgent 在独立上下文中工作，完成后回传结构化结果。Orchestrator 合并变更 → 整体验证 → 预览。

---

## 4. 自我修正回路——怎么造一个能自己修 bug 的系统

### 4.1 完整闭环时序

```
Step 1: 模型生成 diff
  ↓  (str_replace / write_file)
Step 2: 应用变更到 Sandpack
  ↓  (sandpack.writeFile)
Step 3: 触发构建
  ↓  (sandpack.runBuild)
Step 4: 等待构建完成，读取结果
  ↓
Step 5: 判断
  ├── 构建成功 → ✅ 推送预览更新 → 响应用户
  └── 构建失败 → Step 6

Step 6: 解析报错
  输入：构建输出的原始文本（可能包含多类错误）
  处理：
    1. 用正则提取每一条错误信息
    2. 解析出 { file, line, column, message, error_type }
    3. 去重（同一行的同一错误只保留一条）
    4. 按文件分组排序
  输出：结构化错误列表

Step 7: 组装修复 prompt
  将结构化错误 + 出错文件的源码 + 原始用户意图 组装为修复 prompt

Step 8: 调用 DS V4 生成修复 diff
  模型看到：报错原因 + 出错的完整代码 + 用户想要什么
  模型输出：修复后的 str_replace diff

Step 9: 应用修复 → 重新构建（回到 Step 3）
  最多重试 3 次
```

### 4.2 报错解析

构建输出的原始文本是这样的：

```
./src/components/Navbar.tsx:12:23
Type error: Property 'bgDark' does not exist on type 'TailwindColors'.

./src/app/page.tsx:45:7
Type error: Cannot find name 'useClient'.
```

解析器将其转为结构化数据：

```typescript
interface BuildError {
  file: string;
  line: number;
  column?: number;
  message: string;
  error_type: "type_error" | "syntax_error" | "import_error" | "lint_error" | "unknown";
}
```

解析结果：
```json
[
  {
    "file": "src/components/Navbar.tsx",
    "line": 12,
    "column": 23,
    "message": "Property 'bgDark' does not exist on type 'TailwindColors'.",
    "error_type": "type_error"
  },
  {
    "file": "src/app/page.tsx",
    "line": 45,
    "column": 7,
    "message": "Cannot find name 'useClient'.",
    "error_type": "type_error"
  }
]
```

### 4.3 修复 Prompt 模板

```
## 构建失败，请修复以下错误：

### 用户原始需求
"把导航栏改成深色"

### 错误列表
1. src/components/Navbar.tsx:12 — type_error: Property 'bgDark' does not exist on type 'TailwindColors'.
2. src/app/page.tsx:45 — type_error: Cannot find name 'useClient'.

### 相关文件内容

#### src/components/Navbar.tsx
[完整文件内容]

#### src/app/page.tsx
[完整文件内容]

### 设计规范
主色: #3B82F6, 背景: #FFFFFF / #F9FAFB, 字体: Inter

请分析错误原因并生成修复 diff（使用 str_replace 格式）。
每次只修改最小必要范围。
```

### 4.4 策略升级机制

当多次小修无效时，自动升级策略：

```
第 1 次失败 → 标准修复（DS V4，局部 str_replace）
第 2 次失败 → 扩大上下文（读取相邻文件，DS V4，可能涉及多个文件）
第 3 次失败 → 升级策略（DS V4 + 更大 token 预算，允许重写受影响文件）
超过 3 次    → 降级处理（告知用户受阻，提供当前状态，请求用户指点）
```

关键设计：**每次失败后增强上下文，而不是简单重试**。第 2 次比第 1 次看到更多相邻代码，第 3 次比第 2 次有更大的 token 预算和更自由的修改权限。

### 4.5 为什么这是产品质量的分水岭

没有自我修正 = 构建报错直接甩给用户 → 用户不会修 → 产品不可用。

有自我修正 = 用户只说「改成深色」→ 模型生成错了 → 系统自己发现、自己分析、自己修 → 用户只看到最终正确的预览。

这个回路的成功率、速度和修复质量，直接决定了用户对产品「智能程度」的体感。

---

## 5. 代码生成与执行

### 5.1 增量编辑：str_replace 的工程保障

迭代阶段一律走局部 str_replace，而非整文件重写。三个关键保障：

**A. 格式契约**
所有代码变更统一用精确匹配的 str_replace 格式。不再使用 diff/patch 格式，避免行号偏移问题。

**B. 锚点唯一性校验**
```typescript
function validateReplaceAnchor(content: string, oldStr: string): AnchorResult {
  const count = countMatches(content, oldStr);
  if (count === 0) return { valid: false, reason: "not_found" };
  if (count > 1)  return { valid: false, reason: "ambiguous", matches: count };
  return { valid: true };
}
```
唯一锚点 = 保证替换只会发生在模型意图的位置。

**C. 版本回滚**
每次成功的 str_replace 写入版本栈：
```typescript
interface VersionEntry {
  id: string;
  timestamp: number;
  file: string;
  old_content: string;
  new_content: string;
}
```
回滚操作：`versionHistory.pop()` → `sandpack.writeFile(entry.file, entry.old_content)` → 重新构建验证。

### 5.2 Sandpack 沙箱集成

Sandpack 是 CodeSandbox 开源的浏览器内打包器。选它的核心原因：**预览延迟接近零**——代码变更后无需等待服务端构建，浏览器本地完成打包和热更新。

**集成方式**：

```typescript
// 初始化沙箱
const sandpack = new SandpackClient({
  template: "nextjs",
  files: {
    "/app/page.tsx": { code: initialCode },
    "/package.json": { code: packageJson },
    // ...
  },
});

// 文件变更后自动触发构建
sandpack.on("build", (result) => {
  if (result.status === "success") {
    // 预览自动更新
  } else {
    // 收集错误信息用于自我修正回路
    const errors = parseBuildErrors(result.errors);
    return errors;
  }
});

// Agent 通过工具调用写入文件
async function writeToSandbox(path: string, content: string) {
  sandpack.updateFile(path, content);  // Sandpack 自动增量构建 + HMR
}
```

**运行时错误捕获**：通过 Sandpack 的 `on("error")` 事件 + window.onerror 注入，捕获预览页面运行时抛出的异常（如组件渲染错误），同样回灌到自我修正回路。

### 5.3 可视化微调

用户在预览区直接点选元素进行轻量修改，不走对话。

**实现链路**：

```
1. 用户点击预览中的按钮
   ↓
2. 事件冒泡捕获 + 读取元素属性
   - data-source-file="components/Hero.tsx"
   - data-source-line="42"
   - data-element-type="button"
   - data-classes="bg-blue-500 text-white px-4 py-2 rounded-lg"
   ↓
3. 弹出微调面板
   显示当前样式/文案，提供快速编辑入口：
   - 文案 → 直接编辑 textContent
   - 颜色 → 取色器
   - 间距 → 滑块
   - 图片 → 上传/替换
   ↓
4. 用户确认修改
   ↓
5. 生成 str_replace diff
   从 data-source-file + data-source-line 定位源码
   → 精确替换对应的 class 或文本
   → 走标准 str_replace 工具调用
   ↓
6. 写入 Sandpack → 预览即时更新
```

**源码定位的关键**：在构建阶段，通过 Babel/SWC 插件为每个 JSX 元素注入 `data-source-file` 和 `data-source-line` 属性。这是编译时做的，不影响运行时性能。

```typescript
// Babel 插件伪代码
visitor: {
  JSXElement(path) {
    const { line } = path.node.loc.start;
    const filename = this.file.opts.filename;
    path.node.openingElement.attributes.push(
      t.jsxAttribute(t.jsxIdentifier('data-source-file'), t.stringLiteral(relativePath(filename))),
      t.jsxAttribute(t.jsxIdentifier('data-source-line'), t.numericLiteral(line))
    );
  }
}
```

---

## 6. 企业 SSO 登录接入

### 6.1 认证协议与流程

采用 **OIDC（OpenID Connect）** 协议对接企业 SSO。OIDC 基于 OAuth 2.0 之上，添加了身份认证层，是目前企业 SSO 的主流标准。

**认证流程**：

```
1. 用户访问 Atlas → 点击「企业登录」
   ↓
2. 前端发起 /api/auth/sso?provider=xxx&tenant=xxx
   ↓
3. 后端构造 OIDC Authorization Request：
   GET {idp_authorization_endpoint}
     ?client_id={atlas_client_id}
     &redirect_uri={atlas_callback_url}
     &response_type=code
     &scope=openid profile email
     &state={csrf_token}
     &nonce={anti_replay_nonce}
   ↓
4. 用户被重定向到企业 IdP（如 Okta / Azure AD / 自建 Keycloak）
   ↓
5. 用户在 IdP 完成认证（密码/MFA/生物识别）
   ↓
6. IdP 重定向回 Atlas callback：
   GET /api/auth/callback?code={authorization_code}&state={csrf_token}
   ↓
7. 后端校验 state（防 CSRF） → 用 code 换 token：
   POST {idp_token_endpoint}
     grant_type=authorization_code
     &code={code}
     &client_id={client_id}
     &client_secret={client_secret}
   ↓
8. IdP 返回 id_token（JWT）+ access_token
   ↓
9. 后端验证 id_token：
   - 验签（用 IdP 的公钥/JWKS）
   - 验 audience（必须是 atlas_client_id）
   - 验 issuer（必须是预期的 IdP URL）
   - 验时效（exp、nbf）
   - 验 nonce
   ↓
10. 从 id_token 提取用户身份：
    - sub（唯一用户标识）
    - email
    - name
    - 企业属性（tenant_id / group / role）
    ↓
11. Atlas 会话创建：
    - 签发 Atlas 自己的 session token（JWT 或 opaque token）
    - 写入 Redis（session → user_info + tenant_info）
    - Set-Cookie / 返回 token 给前端
    ↓
12. 后续请求携带 session token → 中间件验证 → 注入 req.user + req.tenant
```

### 6.2 多租户数据隔离

目前使用 Supabase 开源版，多租户隔离策略为**数据库行级安全（Row Level Security, RLS）**：

```sql
-- 每个业务表都添加 tenant_id
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  user_id UUID NOT NULL,
  name TEXT NOT NULL,
  -- ...
);

-- RLS 策略：用户只能访问自己租户的数据
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON projects
  FOR ALL
  USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

每次请求时，中间件从 session 中提取 tenant_id，设置为 PostgreSQL 的运行时参数：
```sql
SET app.current_tenant_id = 'xxx-tenant-id';
```

**当前卡点**：Supabase 开源版在 RLS + 多 tenant 场景下，`current_setting` 方式在高并发时存在性能瓶颈（每个请求都需 SET 参数），且在 Supabase 的 REST API 层无法透明传递 tenant context。备选方案包括：
- 切换到 Supabase 付费版（原生多租户支持更好）
- 自建 API 中间层绕过 Supabase REST API，直接操作 PG
- 迁移到 Citus 或使用 PG 的 schema-based 隔离

### 6.3 权限模型

采用 RBAC（基于角色的访问控制）：

```
角色层级（每个租户独立）：
  Owner    — 完全控制：管理成员、删除项目、修改设置
  Admin    — 管理权限：邀请成员、修改项目设置
  Editor   — 编辑权限：创建/编辑/删除应用
  Viewer   — 只读权限：查看应用和对话历史

数据库表结构：
  tenant_members (tenant_id, user_id, role)
  project_members (project_id, user_id, role) — 项目级可覆盖租户级
```

权限检查中间件：
```typescript
async function requireRole(requiredRole: Role) {
  const { tenant_id, user_id } = req.auth;
  const member = await db.tenant_members.findUnique({ tenant_id, user_id });
  if (!member || ROLE_HIERARCHY[member.role] < ROLE_HIERARCHY[requiredRole]) {
    throw new ForbiddenError();
  }
}
```

---

## 7. 技术选型的真实权衡

| 领域 | 选型 | 为什么选 | 回头看 |
|------|------|----------|--------|
| 平台前端 | Next.js | 全栈一体——API Route、SSR、静态生成都在一个框架里，不需要额外后端服务 | 选对了。生成的应用和平台本身同技术栈，代码一致性高 |
| Agent 编排 | LangGraph | 需要状态机语义（条件边、子图、状态持久化）而非简单的链式调用；LangGraph 的 checkpoint 机制天然支持暂停/恢复/回放 | 选对了。状态机抽象让多 Agent 协同和错误恢复变得可控。但文档和社区还在早期，踩了一些 API 变更的坑 |
| 大模型（主） | DeepSeek V4 | 代码生成能力接近 GPT-4 级别，成本约为其 1/10 | 性价比极高。中文场景表现甚至更好 |
| 大模型（辅） | Qwen2.5 | 开源可自部署，摘要/分类/简单 diff 任务够用，成本接近于零 | 符合预期。在上下文裁剪场景充当了很好的「压缩器」角色 |
| 沙箱 | Sandpack | 浏览器内构建 → 预览延迟接近零；开源可控 | 选对了交互体验，但局限性是仅支持前端构建——后端代码只能走服务端通道 |
| BaaS | Supabase 开源版 | Postgres + 自动 REST API + Auth + 实时订阅一站式；开源可自托管 | 功能强大但多租户支持是弱项。如果重选，可能会直接用 PG + 自建 API 层 |
| 向量检索 | pgvector | 和业务数据同库，起步零运维成本；SQL 可直接 join 向量和业务数据 | 当前数据量下完全够用，暂不需要迁移到专用向量库 |
| 版本历史 | PG JSONB | 每次 str_replace 的 old/new 存为 JSONB，支持按时间回滚 | 简单有效，当前规模足够 |

---

## 8. 工程挑战——踩过的坑

### 8.1 多轮迭代越改越乱

**现象**：用户第一轮生成的效果不错，但迭代 5-6 轮后，Agent 开始「忘记」之前的结构，改 A 坏了 B。

**根因分析**：
1. 代码规模膨胀后，上下文塞不下所有文件，模型只能看到局部
2. 只依赖对话历史记忆项目状态 → 幻觉
3. 没有强制约束「改之前先读」，模型基于记忆中的代码做 str_replace → anchor 找不到或匹配错误

**解决措施**：
1. **代码即真相**：每轮重建文件树摘要，强制 `read_file` 先于 `str_replace`
2. **结构化锚点常驻上下文**：设计 token、路由表、组件 props 约定在每个 prompt 中都出现，防止模型「发明」新的
3. **细粒度版本回滚**：用户发现某轮改坏了，可一键回滚到之前任意版本

**效果**：迭代稳定性从「5 轮后必崩」提升到「15+ 轮仍可维持」。但仍是持续改进的重点。

### 8.2 首 Token 延迟

**现象**：用户发送指令后，要等 3-5 秒才看到第一个 token。在 demo 场景下体验不佳。

**根因**：
1. 模型需要等待完整上下文组装（包括从 Sandpack 拉文件树、从 Redis 读记忆、从向量库检索）
2. DS V4 的推理延迟本身不低（尤其规划阶段需输出结构化 JSON）

**优化措施**：
1. **上下文预处理并行化**：文件树拉取、记忆读取、向量检索三路并行，取最长路径
2. **规划 prompt 精简**：规划阶段只需文件树摘要 + 用户意图，不需要完整源码——将任务层和检索层的组装推迟到执行阶段
3. **流式响应的提前量**：在规划阶段就开始推送「正在分析你的需求…」占位消息，缩短用户感知的空白
4. **模型预热**：对高频会话保持模型连接池，消除冷启动

### 8.3 成本控制

**核心策略——模型分级路由**：

```typescript
function routeModel(task: TaskType): ModelChoice {
  switch (task) {
    // DS V4 — 高难度
    case "planning":
    case "complex_codegen":
    case "error_fix_advanced":
      return { model: "deepseek-v4", max_tokens: 4000 };

    // Qwen2.5 — 低难度
    case "summarization":
    case "simple_diff":
    case "intent_classification":
    case "context_compression":
      return { model: "qwen2.5", max_tokens: 1000 };

    // 默认 DS V4，兜底
    default:
      return { model: "deepseek-v4", max_tokens: 2000 };
  }
}
```

**其他降本手段**：
- Prompt 缓存：系统层 prompt 的 KV cache 复用（DS V4 支持）
- 上下文裁剪（见 3.2.2）减少每轮 token 消耗
- str_replace 替代整文件重写，大幅缩减输出 token
- 规划阶段仅输出结构化 JSON 而非自然语言计划，减少输出 token

### 8.4 Supabase 多租户

详见 6.2 节。当前方案可运转，但在高并发 + 复杂权限场景下需要升级。

---

## 9. 下一步

**短期（1-2 月）**：
- 多租户方案升级：评估 Supabase 付费版 vs 自建 PG API 层
- 迭代稳定性强化：引入更细粒度的文件级 diff 校验 + 自动化回归测试
- 首 Token 延迟降至 2 秒内

**中期（3-6 月）**：
- 小程序生成能力
- 更复杂的后端逻辑生成（工作流、定时任务）
- 第三方服务集成市场（支付、地图、账号、AI 能力）

**架构演进方向**：
- Sandpack + 服务端沙箱双轨——当前 Sandpack 只支持前端，需引入服务端沙箱支持全栈构建
- SubAgent 模板化——将高频子任务（建表、做登录页、接入支付）固化为预置 SubAgent 模板，减少规划成本
- 评测体系——基于积累的真实对话数据构建离线评测集，量化每次迭代对质量的影响
