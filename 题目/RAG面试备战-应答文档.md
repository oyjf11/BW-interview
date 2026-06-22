# RAG 面试备战 · 基于个人项目的应答文档

> 配套《RAG / 检索增强面试官评分手册》20 题。每题以我的真实项目为底座作答，未覆盖的考点做合理补全。
>
> **标记说明：**
> - `✅[真实]` —— 项目里确有代码/设计支撑。**自信、具体地说**，敢报细节（文件名、类名、参数）。
> - `⚙️[补全]` —— 项目未覆盖但合理延伸。**稳住说设计思路**，别报实现细节，被追问时落到"会这么设计"而非"我已经这么做了"。
>
> **主力项目缩写：** RAG = ai-chat-root（生产 RAG 平台）｜ rag2 = 离线消融实验 ｜ SRE = SRE Agent (OpsPilot) ｜ PPT = AI-PPT ｜ HS = Harness System
>
> **面试官最看重的拉档题：** Q1 / Q5 / Q7 / Q12 / Q14 / Q19 / Q20

---

# 一、索引与分块

## Q1　分块策略选型：固定窗口 / 段落 / 父子 small-to-big 的取舍

> **主接项目：RAG 的 ChunkerService 三策略 + rag2 的父子分块 small-to-big。** 这是我的硬核拉档题，全真实，自信报细节。

### 🟢 我命中的参考答案要点

- ✅[真实] **三种分块策略真实实现**：我的 `ChunkerService`（`rag/server/src/modules/knowledge/chunker.service.ts`）实现了 `paragraph` / `sliding` / `parent_child` 三种策略，默认 size=600、overlap=80、minLength=20，按业务可选。段落策略尊重文档结构，过短段（<minLength）丢弃，过长段 fallback 滑动切分。
- ✅[真实] **父子分块（small-to-big）真实落地**：`chunkParentChild()` 生成父块（size×2，完整段落）和子块（父块再滑动切成更小片段，带 `parentIndex` 钩子）。rag2 里我把这层做得更彻底——`build_child_chunks()` 只把子块（场景描述）向量化入库，父块（完整 plan + pitfalls）**不向量化、只存**，子块 metadata 带 `parent_id`，命中后用 parent_id 取回完整父块喂 Planner。
- ✅[真实] **解决「检索块短而准 vs 生成块长而全」的矛盾**：这是我做 small-to-big 的核心动机——检索时小 chunk 去匹配用户口语（召回准），命中后取回大 chunk 喂 LLM（规划要完整）。不是分块技巧，是检索块与生成块的解耦。
- ✅[真实] **同父块多子块命中聚合**：rag2 的 `SmallToBigRetriever.retrieve()` 按 `parent_id` 聚合，同父块多子块取 **MAX 分数**——一个父块的多个场景描述子块只要有一个命中，父块就进候选。

### 🟡 我能拿到的加分项

- ✅[真实] **主动点出检索块与生成块的解耦是核心价值**：我讲 small-to-big 不讲"分块技巧"，讲"检索块和生成块需求矛盾"——这是设计层判断。
- ✅[真实] **聚合策略 MAX**：rag2 用 defaultdict + MAX 聚合，`if score > parent_scores[pid]: parent_scores[pid] = score`，有代码支撑。
- ✅[真实] **过短块语义稀疏、过长块稀释信号**：我的 minLength=20 过滤过短块，rag2 的子块用场景描述（语义聚焦），父块用完整方案（信息密集）——粒度匹配用途。

### 🔴 危险信号（主动规避）

- ❌ 别说"按 500 字切片"——我有三策略 + 父子分块，按业务选。
- ❌ 别说"块越大越好"——我强调检索块要短要准，生成块要长要全，两者解耦。
- ❌ 别把 small-to-big 等同于普通分块——我强调它是检索/生成块的解耦设计。

### 完整应答（口语稿）

> 分块策略我会根据业务选，不是无脑固定窗口。我的 RAG 平台里 ChunkerService 实现了三种策略——段落、滑动窗口、父子分块，默认 600 字 80 重叠。段落策略尊重文档结构，过短段丢弃，过长段 fallback 滑动切。但最有设计价值的是父子分块 small-to-big。
>
> 我在离线实验 rag2 里把这层做得很彻底：父块是完整的应用范式（plan_steps 加 pitfalls），**不向量化、只存**；子块是同一范式的几种口语化场景描述，**只有子块入库向量化**。子块 metadata 带 parent_id。检索时小而准的子块去匹配用户口语——召回准；命中后用 parent_id 钩子取回大而全的父块喂 Planner——规划要完整。这就解决了"检索块要短要聚焦、喂 LLM 的块要长要全"的矛盾。
>
> 关键细节是同父块多子块命中时的聚合——我按 parent_id 聚合取 MAX 分数，一个父块的多个场景子块只要有一个命中，父块就进候选。这不是分块技巧，是检索块和生成块的解耦设计。粒度匹配用途：子块语义聚焦做匹配，父块信息密集做生成，过短块语义稀疏、过长块稀释信号，small-to-big 两全。

---

## Q2　embedding 选型与 query/document 不对称编码

> **主接项目：RAG 的 EmbeddingService（bge-m3 + query/document 分流）+ rag2（text-embedding-v4 + text_type）。** 真实落地，自信说。

### 🟢 我命中的参考答案要点

- ✅[真实] **选型维度**：我的 RAG 用 `BAAI/bge-m3`（SiliconFlow API），1024 维，支持中英多语言；rag2 用阿里 `text-embedding-v4`，dimension 可配。选型时看语言覆盖、维度、成本。
- ✅[真实] **非对称检索 query/document 分流编码**：我的 `EmbeddingService.embedTexts()` 支持 `textType: 'query' | 'document'` 和 `splitQueryDoc` 参数——query 编码时加 `QUERY_INSTRUCTION`（"为这个查询生成用于检索相关文档的表示："），document 不加。rag2 用 dashscope 的 `text_type='query'/'document'` 参数直接分流。让短 query 和长 document 在同一空间可比。
- ✅[真实] **批量嵌入分批 + 限速**：`BATCH_SIZE = 25`，批次间 `setTimeout(200ms)` 避免触发 SiliconFlow 频率限制。rag2 的 `EMBED_BATCH_SIZE = 10`。保序：按 index 排序后取 embedding。

### 🟡 我能拿到的加分项

- ✅[真实] **区分对称与非对称检索**：我的 splitQueryDoc 是可选的——对称场景（相似句对、聚类）关掉，检索场景打开。这是显式区分。
- ⚙️[补全] **嵌入缓存按 content hash**：我项目有 eval 模块的 embedding-cache，生产嵌入缓存是延伸设计（按 content hash 缓存，变更才重嵌）。
- ✅[真实] **多语言选型**：bge-m3 是多语言模型，我的 RAG 支持中英混合知识库，选 bge-m3 而非中文专用模型就是因为多语言覆盖。

### 🔴 危险信号（主动规避）

- ❌ 别说"随便选个 OpenAI embedding"——我选 bge-m3 有语言/维度/成本考量。
- ❌ 别让 query 和 document 用完全相同编码——我有 QUERY_INSTRUCTION 分流 + text_type 参数。
- ❌ 别批量嵌入不限速——我 batch 25 + 200ms 限速，防频率限制。

### 完整应答（口语稿）

> embedding 选型我看几个维度：语言覆盖、维度、是否指令敏感、推理成本。我的 RAG 平台用 bge-m3，1024 维，支持中英多语言，走 SiliconFlow API。选 bge-m3 而不是中文专用模型，就是因为知识库有中英混合，要多语言覆盖。
>
> 关键是 query 和 document 的不对称编码。检索是非对称场景——query 短、document 长，用完全相同的编码会让短 query 在长 document 的空间里可比性差。我的 EmbeddingService 支持 textType 参数和 splitQueryDoc 开关：query 编码时加一段 instruction——"为这个查询生成用于检索相关文档的表示："，document 不加。rag2 里我用 dashscope 的 text_type 参数直接分流 query/document。这样短 query 和长 document 在同一空间可比。对称场景比如相似句对聚类，我会关掉 splitQueryDoc。
>
> 工程上批量嵌入要分批加限速。我 batch size 25，批次间 sleep 200ms 避免触发频率限制，按 index 排序保序取 embedding。嵌入缓存我有 eval 模块的 cache，生产侧按 content hash 缓存是延伸设计——文档更新只重嵌变更块，不全量重嵌。

---

## Q3　知识库构建与更新：文档解析、增量索引、索引一致性

> **主接项目：RAG 的 ChunkerService.parseFile + 离线优先异步索引 + 状态追踪。** 真实落地，自信说；双索引切换是补全。

### 🟢 我命中的参考答案要点

- ✅[真实] **文档解析按格式选解析器 + 失败降级**：`parseFile()` 按 ext 分流——.txt/.md 直读 utf-8，.docx 用 mammoth.extractRawText，不支持的格式抛明确错误（"请上传 .txt/.md/.docx"）而非崩溃。
- ✅[真实] **离线优先异步索引 + 状态追踪**：文档处理用 `.catch()` 异步执行，不阻塞上传响应；状态字段追踪 `pending → indexing → ready | failed`。用户上传立即返回，索引在后台跑，失败可重试。
- ✅[真实] **索引流程**：解析 → 分块 → 嵌入（batch+限速）→ 写向量库（kb_chunks.embedding）+ FTS 索引（kb_chunks_fts）。
- ⚙️[补全] **增量更新处理旧块失效**：更新文档要删旧 chunk 再写新 chunk，防"幽灵块"。我项目有文档管理 API，但"更新时级联清理旧块"是设计延伸，要稳住说。

### 🟡 我能拿到的加分项

- ⚙️[补全] **重索引期间双索引切换**：保持检索可用——版本号或双索引切换。设计方向，项目没实现。
- ⚙️[补全] **文档删除向量库与 FTS 级联清理**：防幽灵块。设计原则，要稳住说。
- ✅[真实] **离线优先**：文档处理 `.catch()` 异步，上传响应不阻塞，状态字段追踪进度。

### 🔴 危险信号（主动规避）

- ❌ 别说"建库就是读文件切块存"——我有解析分流 + 状态机 + 异步索引。
- ❌ 别说"更新文档直接覆盖"——我主张删旧块再写新块，防残留。
- ❌ 别说"索引失败无反馈"——我有状态字段 pending/indexing/ready/failed 追踪。

### 完整应答（口语稿）

> 建库这一环很多人低估，以为是"读文件切块存"。我的 RAG 平台做了几件事。第一，文档解析按格式分流——txt/md 直读，docx 用 mammoth，不支持的格式抛明确错误让用户重传，而不是崩溃。第二，离线优先异步索引——文档处理用 catch 异步执行，不阻塞上传响应，状态字段追踪 pending 到 indexing 到 ready 或 failed，用户上传立即返回，索引在后台跑，失败可重试。
>
> 索引流程是解析、分块、嵌入、写向量库加 FTS 索引。嵌入分批加限速，前面讲过。增量更新这块我的立场是：更新文档要删旧 chunk 再写新 chunk，防止"幽灵块"——旧内容残留在向量库和 FTS 索引里，检索还会命中。文档删除要向量库和 FTS 级联清理。重索引期间保持检索可用我会用双索引或版本号切换，这是设计方向。这些我有部分落地、有部分是设计延伸，但原则很清楚：状态可追踪、失败可重试、旧块不残留。

---

## Q4　元数据与结构化过滤：检索前硬过滤 vs 检索后过滤

> **主接项目：rag2 的 metadata 硬过滤（meta_filter）+ SRE 的 doc_type/service/env 过滤。** 真实落地，自信说。

### 🟢 我命中的参考答案要点

- ✅[真实] **元数据挂 chunk 级，检索前硬过滤**：rag2 的子块 metadata 带 `parent_id + domain + complexity + multi_user + need_auth + entity_count`，`SmallToBigRetriever.retrieve(meta_filter={"multi_user": True})` 在向量计算前用 numpy mask 硬过滤——`valid_idx = np.where(mask)`，只在满足条件的子块上算相似度。
- ✅[真实] **检索前过滤缩小向量搜索空间**：SRE 的 `retrieve_memory` 用 `_build_filters()` 把 service/env 转 MetadataFilter 等值匹配，意图分类 → doc_type 过滤（`_INTENT_FILTER_MAP`），都在向量检索前缩小集合。
- ✅[真实] **结构化过滤与向量检索正交**：先按元数据缩小集合，再在子集上算相似度——rag2 的 mask 过滤后再 `cosine(qv, child_vecs[valid_idx])`，SRE 的 filters 传给 retrieve() 缩小范围。
- ⚙️[补全] **post-filter 与过滤后不足的兜底**：软约束 post-filter、过滤后结果不足时放宽过滤或跨库检索。设计延伸。

### 🟡 我能拿到的加分项

- ✅[真实] **区分硬约束 pre-filter**：rag2 的 multi_user=True 是硬约束，直接 mask；SRE 的 service=payment 是硬约束，MetadataFilter 等值匹配。都是 pre-filter。
- ✅[真实] **元数据放子块、父块取回**：rag2 的 metadata 挂子块（`children.append({"metadata": {"parent_id": ..., **p["metadata"]}})`），子块过滤、父块取回——small-to-big 场景的标准做法。
- ⚙️[补全] **过滤后不足的兜底**：放宽过滤 / 跨库 / 拒答。设计方向。

### 🔴 危险信号（主动规避）

- ❌ 别说"不做元数据过滤全靠向量召回"——我有 rag2 mask 硬过滤 + SRE MetadataFilter。
- ❌ 别说"一律 post-filter"——我主张硬约束 pre-filter，缩小搜索空间。
- ❌ 别说"元数据全塞文本"——我挂结构化字段（domain/multi_user/service/env），可硬过滤。

### 完整应答（口语稿）

> 元数据过滤我会区分硬约束和软约束。硬约束走 pre-filter，在向量计算前缩小搜索空间；软约束可以 post-filter 或加权。我的 rag2 里子块 metadata 带 domain、complexity、multi_user、need_auth 这些结构化字段，检索时传 meta_filter 比如 multi_user=True，我用 numpy mask 在算相似度前硬过滤——只在满足条件的子块上算 cosine。我的 SRE Agent 同款设计，retrieve_memory 用 MetadataFilter 按 service、env 等值匹配，意图分类还会加 doc_type 过滤，都在向量检索前缩小集合。
>
> small-to-big 场景有个细节：元数据挂子块，子块过滤、父块取回。rag2 的子块 metadata 带 parent_id 加父块的结构化特征，过滤在子块层做，命中后用 parent_id 取回完整父块。结构化过滤和向量检索是正交的——先按元数据缩小集合，再在子集上算相似度，性能和召回都更好。过滤后结果不足的兜底，比如放宽过滤或跨库检索，这是我会说的延伸方向。

---

# 二、检索与排序

## Q5　混合检索（向量 + 关键词）的必要性与融合方法

> **主接项目：RAG 的 RetrievalService（向量 + FTS5 + RRF 融合）+ rag2（向量 + 关键词 IDF + hybrid 加权）+ SRE（DashVector + FTS5 + RRF）。** 三项目都有真实落地，这是我的拉档强项。

### 🟢 我命中的参考答案要点

- ✅[真实] **纯向量检索的失败模式**：专有名词、产品名、错误码、精确 ID——向量语义匹配召回差。这是我做混合检索的动机。
- ✅[真实] **混合检索两路召回**：RAG 的 `RetrievalService.retrieve()` 同时跑 `vectorSearch()`（cosine，阈值 0.3）和 `ftsSearch()`（FTS5 MATCH），rag2 的 `hybrid_retrieve()` 跑向量 + `KeywordRetriever`（IDF 加权倒排索引）。SRE 的 `retriever.py` 跑 DashVector 向量 + FTS5 关键词。
- ✅[真实] **RRF 融合（排名倒数）**：RAG 的 `rrfMerge()` 用 `1/(rank+K)`，K=60，不依赖分数尺度——`scores.set(c.id, (scores.get(c.id) ?? 0) + 1/(rank+K))`。SRE 同款 RRF。
- ✅[真实] **加权融合**：rag2 的 `hybrid_retrieve()` 用 `alpha * vec_sims + (1-alpha) * kw_scores`，alpha 可调，向量权重高于关键词。
- ✅[真实] **RRF 对分数尺度不敏感**：向量 cosine 0~1，FTS5 rank 是负数，尺度天然不同——RRF 只看排名不看分数，跨异构检索器融合稳健。

### 🟡 我能拿到的加分项

- ✅[真实] **RRF 优于加权融合的原因**：我项目同时实现了两种——RAG 生产用 RRF（尺度不敏感），rag2 实验对比了 hybrid 加权。我的立场：跨异构检索器（向量+FTS）用 RRF 稳健，同质检索器或带调参能力时用加权。
- ✅[真实] **关键词检索的中文分词**：rag2 的 `KeywordRetriever._tokenize()` 对中文用单字 + 二元组（`seg[i]` + `seg[i:i+2]`），英文用整词，IDF 加权（`log((n+1)/(df+1))+1`）。
- ✅[真实] **query 净化防 FTS5 注入**：RAG 的 `ftsSearch()` 对 query 去特殊字符 `replace(/['"()*:^]/g, ' ')`，split 后用 OR 连接，防 FTS5 语法错误。

### 🔴 危险信号（主动规避）

- ❌ 别说"向量检索就够了"——我强调专有名词/错误码/ID 是向量弱项，要关键词补。
- ❌ 别说"加权融合不归一化"——我 RAG 生产用 RRF 回避尺度问题，rag2 加权融合做了归一化（kw_scores 除以 max_possible）。
- ❌ 别说"不知道 RRF"——我三个项目都用 RRF，K=60，能报公式。

### 完整应答（口语稿）

> 纯向量检索有明确的失败模式——专有名词、产品名、错误码、精确 ID，这些语义匹配召回差，但关键词检索是强项。所以我做混合检索，向量加关键词两路召回再融合。
>
> 我的 RAG 平台 RetrievalService 同时跑 vectorSearch（cosine，阈值 0.3）和 ftsSearch（FTS5 MATCH），再用 RRF 融合。RRF 用排名倒数 1 除以 rank 加 K，K 取 60，只看排名不看分数。这个细节很重要——向量 cosine 是 0 到 1，FTS5 的 rank 是负数，尺度天然不同，如果用加权融合不归一化就会失真。RRF 对分数尺度不敏感，是跨异构检索器融合的稳健默认。我的 SRE Agent 同款设计，DashVector 向量加 FTS5 关键词加 RRF 融合。
>
> 我在 rag2 离线实验里还实现了加权融合做对比——alpha 乘向量分加 1-alpha 乘关键词分，关键词用 IDF 加权倒排索引，中文分词用单字加二元组，英文用整词。加权融合要归一化，我把 kw_scores 除以 max_possible 归一到 0-1。我的立场是：跨异构检索器用 RRF 稳健，同质检索器或带调参能力时用加权。query 净化我也做了——FTS5 MATCH 前去掉引号括号冒号等特殊字符防语法错误，split 后用 OR 连接。

---

## Q6　RRF vs 加权融合 vs rerank 的定位与取舍

> **主接项目：RAG 的 RRF（生产）+ rag2 的加权融合 + 两阶段 rerank。** 三手段分层真实落地，自信说。

### 🟢 我命中的参考答案要点

- ✅[真实] **RRF 是排名级融合**：RAG 生产用 RRF 融合向量+FTS 两路召回，排名倒数，不依赖分数尺度，适合异构检索器粗融合。
- ✅[真实] **加权融合是分数级融合**：rag2 的 `hybrid_retrieve()` 用 `alpha * vec + (1-alpha) * kw`，需归一化，适合同质检索器或带调参。
- ✅[真实] **rerank 是 cross-encoder 精排**：rag2 的 `rerank()` 用 gte-rerank 对 query-doc 对打分，准但贵，只对粗排 top-recall_k 精排。
- ✅[真实] **三段式管线**：粗排召回（向量/关键词）→ 融合（RRF/加权）→ 精排（rerank）。我项目就是这层结构——RAG 生产是向量+FTS→RRF（无 rerank），rag2 是向量→（改写）→rerank，SRE 是向量+FTS→RRF→可选 rerank。

### 🟡 我能拿到的加分项

- ✅[真实] **三段式管线各段目标**：召回求广（向量+关键词）、融合求稳（RRF 回避尺度）、精排求准（rerank cross-encoder）。我能画出来。
- ✅[真实] **rerank 成本控制**：rag2 只对粗排 top-recall_k=8 精排，不全量 rerank——`coarse = sorted(scored)[:recall_k]` 再送 rerank。
- ✅[真实] **rerank 失败 fallback**：rag2 的 `rerank()` 失败返回 None，调用方 `if rr is not None` 才用 rerank 分，否则沿用 cosine 分，`debug["used_rerank"]` 标记，报表打 `*`。外部依赖挂了链路不断且可观测。

### 🔴 危险信号（主动规避）

- ❌ 别把 RRF 和 rerank 当二选一——我讲三段式：召回→融合→精排，各层不同目标。
- ❌ 别说"全量 rerank"——我只对粗排 top-8 精排，控成本。
- ❌ 别说"rerank 失败直接报错"——我有 fallback 回 cosine + 标记可观测。

### 完整应答（口语稿）

> RRF、加权融合、rerank 不是三选一，是三段式管线的不同层。我画一下：第一段粗排召回求广——向量加关键词两路召回；第二段融合求稳——RRF 或加权把两路合并；第三段精排求准——rerank 对候选集 cross-encoder 精排。
>
> RRF 是排名级融合，用排名倒数，不依赖分数尺度，适合异构检索器粗融合——我的 RAG 生产用它融合向量和 FTS。加权融合是分数级融合，需要归一化，适合同质检索器或带调参能力——我在 rag2 实验里实现了，alpha 乘向量加 1-alpha 乘关键词。rerank 是 cross-encoder 精排，对 query-doc 对联合编码打分，准但贵——rag2 用 gte-rerank。
>
> rerank 的成本控制很关键：我只对粗排 top-recall_k 精排，rag2 里 recall_k=8，不是全量 rerank。而且 rerank 失败要有 fallback——我失败时回退 cosine 排序，链路不断，同时打一个 used_rerank 标记让报表能区分"用了 rerank"还是"回退了"，外部依赖挂了可观测。这三个手段在我的项目里是分层组合，不是替代关系。

---

## Q7　两阶段检索（粗排 + 精排）与 rerank 的价值边界

> **主接项目：rag2 的两阶段检索 + 杀手锏结论（命中率 +0pp 但 margin 翻 2.4 倍）。** 这是我的拉档核心题，有反直觉实战结论，自信报数字。

### 🟢 我命中的参考答案要点

- ✅[真实] **两阶段：粗排向量召回 + 精排 cross-encoder**：rag2 的 `SmallToBigRetriever.retrieve(use_rerank=True, recall_k=8)`——先向量 cosine 取 top-8 候选（`coarse = sorted(scored)[:recall_k]`），再 gte-rerank 对这 8 个精排（`rerank(query, cand_docs)`）。
- ✅[真实] **rerank 价值不止命中率，更在排序质量和置信度**：我的杀手锏结论——A 裸向量/B +改写/C +rerank 三路命中率全 100%（测试集对基线太简单，命中率早打满天花板），但 **margin 从 0.21 → 0.31 → 0.51 翻 2.4 倍**。rerank 收益不在"有没有命中"，在"敢不敢下结论"。
- ✅[真实] **命中率没提升 ≠ rerank 无用**：我的归因——测试集对基线太简单，命中率天花板已打满，rerank 的收益转移到 margin 拉大，生产里能在 0.4 设置信度门槛做拒答/转人工，裸向量 margin 0.21 根本做不了。
- ✅[真实] **评测分两类指标：命中率 + 置信度**：我的四件套报告 ③ 置信度对比专门统计平均 top1 分和平均 margin——光看命中率漏掉一半故事。

### 🟡 我能拿到的加分项

- ✅[真实] **命中率 +0pp 但 margin 翻 2.4 倍的反直觉结论**：这是我的杀手锏故事，能背数字（0.21→0.51，2.4 倍）。
- ✅[真实] **margin 作为生产决策工具**：我的报告 ⑦ 置信度门控评估做阈值扫描——扫描 margin 阈值，输出精度/召回/拒答率/F1，找 F1 最高点。C 路 margin 分布更宽，可选阈值范围更大，门控灵活性最好。
- ✅[真实] **cross-encoder vs bi-encoder 本质区别**：rerank 是 cross-encoder（query-doc 联合编码），向量是 bi-encoder（独立编码再算相似度）。cross-encoder 准但贵，bi-encoder 快但粗。

### 🔴 危险信号（主动规避）

- ❌ 别说"rerank 没提升命中率就没用"——我有杀手锏结论：margin 翻 2.4 倍才是真价值。
- ❌ 别说"全量 rerank 或粗排召回数等于最终返回数"——我 recall_k=8，精排才有意义。
- ❌ 别说"不知道 margin/门控"——我有阈值扫描找 F1 最高点的实战。

### 完整应答（口语稿）

> 两阶段检索我的设计是粗排向量召回加精排 cross-encoder。rag2 里先向量 cosine 取 top-8 候选，再 gte-rerank 对这 8 个精排。recall_k=8 是成本与质量的旋钮——太小精排没意义，太大成本爆炸。
>
> 这道题我有個反直觉的实战结论，是我最想讲的。我跑 A 裸向量、B 加改写、C 加 rerank 三路消融，命中率三路全 100%，归因 +0pp，看着 rerank 白接了。但置信度那栏 margin 从 0.21 到 0.31 到 0.51 翻了 2.4 倍。这说明测试集对基线太简单，命中率早打满天花板，rerank 的收益不在"有没有命中"，在"敢不敢下结论"。生产里这意味着我能在 0.4 那条线设置信度门槛做拒答或转人工，裸向量 margin 0.21 根本做不了这件事。
>
> 所以 rerank 的价值边界是：不只是命中率，更在排序质量（MRR/NDCG）和置信度（margin）。我用 margin 做生产决策工具——我的评测报告有置信度门控阈值扫描，扫描 margin 阈值输出精度、召回、拒答率、F1，找 F1 最高点。C 路 rerank 的 margin 分布更宽，可选阈值范围更大，门控灵活性最好。cross-encoder 和 bi-encoder 的本质区别我也清楚：rerank 是 query-doc 联合编码，向量是独立编码再算相似度，cross-encoder 准但贵，所以只对粗排 top-N 精排。

---

## Q8　query 改写 / 扩展：补 vocabulary gap，规则 vs LLM

> **主接项目：rag2 的 rewrite_query（STOPWORDS + TERM_MAP）+ SRE 的 query_rewriter（21 停用词 + 35 术语映射）。** 两项目都有真实落地，自信报参数。

### 🟢 我命中的参考答案要点

- ✅[真实] **vocabulary gap 是召回失败主因**：用户说"卖东西的网站"，知识库写"电商 在线商城"——语义同但词面不同，向量召回差。这是我做 query 改写的动机。
- ✅[真实] **规则改写零延迟零成本**：rag2 的 `rewrite_query()` 两步——`STOPWORDS` 去口语噪声（帮我/做个/那个/玩意儿），`TERM_MAP` 术语扩展（"卖东西的网站" → "电商 在线商城"、"管用户权限" → "RBAC 权限控制"）。SRE 的 `query_rewriter.py` 同款：`remove_stopwords()` 去 21 个中文口语填充词 + `expand_terms()` 35 个 SRE 术语映射（"5xx" → "HTTP 500 502 503 错误"）。零配置零延迟零成本。
- ✅[真实] **LLM 改写能理解隐含意图但有成本**：rag2 的 `decompose_by_llm()` 用 qwen-plus，能补全模糊词、理解隐含意图，但有延迟和成本。
- ✅[真实] **生产策略：规则优先 fallback LLM**：rag2 的多意图拆分就是这策略——`decompose_by_rules()` 优先，拆不开时 `decompose_by_llm()` fallback。改写同思路。

### 🟡 我能拿到的加分项

- ✅[真实] **改写本质是补 vocabulary gap**：我讲改写不讲"让 query 更好懂"，讲"口语 vs 书面语的词汇鸿沟"——设计层判断。
- ✅[真实] **术语映射表可运营**：SRE 的 35 个术语映射是配置化的，可版本化可运营，不是硬编码在业务逻辑。
- ✅[真实] **改写副作用：过度扩展引入噪声**：我的评测 ② 翻盘/翻车归因专门追踪"改写带偏"——`if a["hit1"] and not b["hit1"]: flips.append("⚠️ 翻车 [改写带带]")`，改写不是只翻盘也会翻车，需评测验证。

### 🔴 危险信号（主动规避）

- ❌ 别说"向量检索能理解语义就够了"——我强调 vocabulary gap 是召回失败主因。
- ❌ 别说"一律 LLM 改写"——我规则优先（零成本零延迟），fallback LLM。
- ❌ 别说"改写不做评测"——我有翻盘/翻车归因，知道改写是救场还是带偏。

### 完整应答（口语稿）

> query 改写的本质是补 vocabulary gap——用户口语和知识库书面语的词汇鸿沟。用户说"卖东西的网站"，知识库写"电商 在线商城"，语义同但词面不同，向量检索召回差。
>
> 我的改写分两步，都在 embedding 之前做。第一步去口语停用词——"帮我"、"做个"、"那个"、"玩意儿"这些零信息量的词去掉。第二步术语扩展——口语到专业术语的映射，"卖东西的网站"补成"电商 在线商城"，"管用户权限"补成"RBAC 权限控制"。我的 rag2 用 STOPWORDS 加 TERM_MAP，SRE Agent 的 query_rewriter 更系统——21 个中文口语填充词加 35 个 SRE 术语映射，"5xx" 扩展成 "HTTP 500 502 503 错误"。纯规则引擎，零配置零延迟零成本。
>
> LLM 改写我能做但不是默认——rag2 的 decompose_by_llm 用 qwen-plus，能理解隐含意图、补全模糊词，但有延迟和成本。我的生产策略是规则优先、拆不开时 fallback LLM。要强调一个副作用：改写不是只翻盘也会翻车——过度扩展会引入噪声导致召回漂移。我的评测有翻盘翻车归因专门追踪"改写带偏"，所以改写必须评测验证，不是上了就一定好。

---

## Q9　多意图拆分与多路召回

> **主接项目：rag2 的 decompose_by_rules + decompose_by_llm + decomp_retrieve（union+max 聚合）+ 多意图评测（Coverage/Precision/F1）。** 真实落地，自信说。

### 🟢 我命中的参考答案要点

- ✅[真实] **多意图问题：单路检索偏好"最像"忽略次要意图**：一句话含多个需求（"管客户还要能提工单"），单路向量检索会偏好语义最像的那个，次要意图召回不到。
- ✅[真实] **拆分成子问题独立检索再聚合**：rag2 的 `decomp_retrieve()` —— `decompose_by_rules(raw_query)` 拆成子句，每个子句 `rewrite_query` 后独立 `retrieve`，结果按 parent_id union + max 聚合（`if score > parent_scores[pid]: parent_scores[pid] = score`）。
- ✅[真实] **规则拆分按连接词**：rag2 的 `_SPLIT_PATTERNS` 按中文连接词优先级递减——"既要...也要"、"还要能/还要有"、"同时"、"还能/又能"、"+/、"。`decompose_by_rules()` 试每个 pattern，找到就拆。
- ✅[真实] **LLM 拆分能理解隐含意图**：`decompose_by_llm()` 用 qwen-plus 拆，能补全模糊词、理解隐含意图，但有成本。失败 fallback 规则拆分。
- ✅[真实] **多意图评测引入覆盖度和精度**：我的 `eval_one()` 对多意图算 `coverage1`（Top1 覆盖几成意图）、`coverage3`（Top3 覆盖几成）、`precision3`（Top3 里几成相关）、`F1@3`（调和平均），不只 hit1。

### 🟡 我能拿到的加分项

- ✅[真实] **hit1 vs Coverage@k 的评测差异**：我的报告 ⑧ 多意图覆盖度专门统计——单意图用 hit1，多意图用 Coverage@k + Precision@k + F1@3，否则次要意图全丢也看不出来。
- ✅[真实] **规则优先 fallback LLM 的最佳生产策略**：我的报告 ⑨ 对比 B 不拆分 / F 规则拆分 / G LLM 拆分，结论是"优先规则拆分，拆不开时 fallback LLM"——规则零延迟，LLM 能理解隐含意图但有成本。
- ✅[真实] **过度拆分副作用**：规则拆分对"还要/同时"覆盖不全，口语绕圈拆不开；LLM 拆分能理解但要 +1 次 LLM 调用。

### 🔴 危险信号（主动规避）

- ❌ 别说"不处理多意图单路检索一把梭"——我有拆分 + union+max 聚合。
- ❌ 别说"一律 LLM 拆分"——我规则优先 fallback LLM，控成本。
- ❌ 别说"多意图还用 hit1 评测"——我引入 Coverage@k + Precision@k + F1@3。

### 完整应答（口语稿）

> 多意图是真实场景——"管客户还要能提工单"，一句话两个需求。单路向量检索会偏好语义最像的那个，次要意图召回不到。我的处理是拆分 + 多路召回 + 聚合。
>
> rag2 里我先按连接词规则拆——"既要...也要"、"还要能"、"同时"、"+/、" 这些 pattern 按优先级递减试，找到就拆。每个子句改写后独立检索，结果按 parent_id 做 union 加 max 聚合，同父块多子句取最高分。规则拆分零延迟零成本，但对口语绕圈拆不开。所以我有 LLM 拆分 fallback——用 qwen-plus 拆，能理解隐含意图、补全模糊词，但有 +1 次 LLM 调用的成本。我的生产策略是规则优先、拆不开时 fallback LLM。
>
> 评测这块要特别讲——多意图不能只用 hit1。我的 eval_one 对多意图算 Coverage@1、Coverage@3、Precision@3、F1@3。Coverage@k 是用户 N 个意图中 Top-K 覆盖了几个，Precision@k 是 Top-K 里几成相关，F1@3 是调和平均。我的报告 ⑨ 对比了 B 不拆分、F 规则拆分、G LLM 拆分三种方案，看 Coverage 和 F1@3 的差异。单意图用 hit1，多意图必须用覆盖度指标，否则次要意图全丢也看不出来。

---

# 三、生成与上下文

## Q10　检索结果如何喂给 LLM：引用、排序、截断、上下文构造

> **主接项目：RAG 的 SSE 7 事件流式 + SRE 的 aggregate 截断惩罚 + 锚点保留。** 真实落地，自信说。

### 🟢 我命中的参考答案要点

- ✅[真实] **检索结果按相关性排序注入 + 带来源标注**：RAG 的 `retrieve()` 返回 `RetrievedChunk` 带 `doc_id`、`filename`、`score`，拼回文件名后注入——`for chunk of merged: doc = db.get(filename from kb_documents)`。rag2 的父块带 `title`、`plan_steps`、`pitfalls`、`metadata`，结构化喂 Planner。
- ✅[真实] **截断策略：top-K + 相关性阈值**：RAG 向量检索有 `VEC_THRESHOLD = 0.3` 过滤低相关，topK 默认 5。SRE 的 aggregate 对超大返回截断并记**截断惩罚**到质量分——原文外置、上下文留摘要与句柄。
- ✅[真实] **上下文结构：指令 + 检索结果（带来源）+ 问题**：SRE 的 diagnose 节点拼 prompt 是 `## Collected Evidence` + `## Incident` 分段，证据按 category 组织（Deployments/Logs/Runbooks/Metrics）。
- ✅[真实] **流式输出降首响**：RAG 自建 SSE 7 事件类型（chunk/reasoning/rag_sources/done/plan/image/error），流式产出，首响不用等全跑完。SRE 用 graph.astream + SSE `GET /runs/{id}/stream`。

### 🟡 我能拿到的加分项

- ✅[真实] **检索结果带来源标注是 citation 基础**：RAG 返回带 filename/doc_id，rag2 父块带 title + metadata，都可溯源。
- ✅[真实] **超预算保留高置信 + 外置低置信句柄**：SRE 的 aggregate 截断 + Small-to-Big 的 parent_id 取回就是这思路——上下文留摘要，原文外置带句柄可回查。
- ⚙️[补全] **渲染后上下文 vs 模板**：SRE 目前 llm_request 事件记的是 metadata（provider/model/prompt_length）+ prompt sha256 版本，不是完整渲染后 prompt 文本。能定位到版本，全文落库是改进点。

### 🔴 危险信号（主动规避）

- ❌ 别说"检索结果不带来源"——我带 doc_id/filename/title，可 citation 可溯源。
- ❌ 别说"top-K 全塞无截断"——我有 VEC_THRESHOLD + aggregate 截断惩罚。
- ❌ 别说"检索内容与指令混在同一信任层"——SRE 的 prompt 分 Evidence/Incident 段，RAG 检索结果带来源标注。

### 完整应答（口语稿）

> 检索结果喂给 LLM 这一步我会做四件事。第一，带来源标注——我的 RAG 返回 RetrievedChunk 带 doc_id、filename、score，注入前拼回文件名；rag2 的父块带 title、plan_steps、pitfalls、metadata，结构化喂 Planner。这是 citation 可溯源的基础。第二，排序加截断——向量检索有 0.3 的相关性阈值过滤低相关，topK 默认 5；SRE 的 aggregate 对超大返回截断并记截断惩罚到质量分，上下文留摘要，原文外置带句柄可回查。第三，上下文结构——SRE 的 diagnose prompt 分 Collected Evidence 和 Incident 两段，证据按 category 组织，检索结果和指令有明确分隔。第四，流式输出降首响——我自建了 SSE 7 事件类型，chunk、reasoning、rag_sources、done、plan、image、error，流式产出，首响不用等整条链跑完。
>
> 要坦诚一点：渲染后上下文落库我目前记的是 metadata——provider、model、prompt_length——加 prompt 的 sha256 版本 checksum，不是完整渲染后 prompt 文本。所以拼装 bug 我能定位到版本，全文落库是改进点。但超预算的取舍我是真做的——保留高置信块，外置低置信块句柄，Small-to-Big 的 parent_id 取回就是这思路。

---

## Q11　幻觉与忠实性：grounded 生成、citation 可溯源

> **主接项目：RAG 的来源标注 + SRE 的 retrieve_memory 召回标注"参考" + AI-PPT 的 Parse+Validate 兜底。** grounded 检测偏补全，citation 真实。

### 🟢 我命中的参考答案要点

- ✅[真实] **citation 可溯源**：RAG 检索结果带 doc_id/filename，生成时可引用；rag2 父块带 title + metadata 可溯源；SRE 的 retrieve_memory 召回产出 `MemoryHit(source="runbook", relevance_score=0.89)`，来源和分数都带在结构里。
- ✅[真实] **检索为空明确拒答**：RAG 的 `retrieve()` 在 `kbIds` 为空时直接返回 `[]`；SRE 的 retrieve_memory 召回为空时 planner 仍基于当前 TriageResult 决策，不凭空生成。
- ⚙️[补全] **grounded 生成 prompt 约束**：SRE 的 diagnose prompt 约束基于证据回答，但"不得超出检索范围编造"的硬约束偏设计。我会讲设计原则。
- ⚙️[补全] **忠实性检测**：生成内容是否能在检索结果找到支撑——可程序化校验或 LLM-as-judge。我有 eval 框架但没做忠实性专项检测，是延伸。

### 🟡 我能拿到的加分项

- ⚙️[补全] **区分检索无关导致幻觉 vs 模型无视检索自由发挥**：设计上分别处理——前者改检索，后者加 prompt 约束 + 忠实性检测。
- ⚙️[补全] **忠实性评测 LLM-as-judge + 人工抽检**：延伸，我的 eval 框架可扩展。
- ✅[真实] **low-confidence 检索结果处置**：SRE 的 retrieve_memory 召回是"参考记忆"不是当前事实，planner 仍以 TriageResult 为准；rag2 的 margin 门控（Q12）让 low-confidence 转人工不直接喂模型。
- ✅[真实] **AI-PPT 的 Parse+Validate 兜底**：我的 AI-PPT 项目对 LLM JSON 输出先 parse 再 validate 再兜底 `getFallbackConfigForType()`，类型强覆盖 `result.config.type = type`——这是控制 LLM 输出不可靠的真实落地，同思路可用于 grounded 检测。

### 🔴 危险信号（主动规避）

- ❌ 别说"接了 RAG 就不会有幻觉"——我强调模型可能无视检索自由发挥，要有 grounded 约束 + 忠实性检测。
- ❌ 别说"生成内容无 citation"——我带 doc_id/filename/source，可溯源。
- ❌ 别说"检索为空仍让模型自由生成"——我 retrieve 空返回 []，SRE planner 以当前证据为准。

### 完整应答（口语稿）

> 先讲一个判断：接了 RAG 不等于没有幻觉——模型可能无视检索结果自由发挥。所以要做两件事：grounded 生成约束 + citation 可溯源 + 忠实性检测。
>
> citation 我有真实落地——RAG 检索结果带 doc_id、filename，生成时可引用；rag2 父块带 title 加 metadata；SRE 的 retrieve_memory 召回产出 MemoryHit 带 source 和 relevance_score，来源和分数都在结构里。这是事后可核对的基础。检索为空时我明确拒答——RAG 的 retrieve 在 kbIds 空时返回空数组，SRE 召回为空时 planner 仍基于当前 TriageResult 决策，不凭空生成。
>
> grounded 约束和忠实性检测我要坦诚——prompt 约束基于证据回答我有做，但"不得超出检索范围编造"的硬约束和忠实性专项检测偏设计延伸。我的 eval 框架可以扩展做忠实性评测。这里我能挂一个跨项目的真实落地：我的 AI-PPT 项目对 LLM JSON 输出做了 Parse + Validate + 兜底——先解析、再校验结构、最后 fallback 到 getFallbackConfigForType，类型强覆盖 result.config.type = type。这是控制 LLM 输出不可靠的真实实践，同思路可用于 grounded 检测：生成内容 parse 后校验是否能在检索结果找到支撑。low-confidence 检索结果我也不直接喂模型——rag2 的 margin 门控让它转人工，SRE 的召回是"参考"不是"真值"。

---

## Q12　低置信度拒答 / 转人工：置信度门控设计

> **主接项目：rag2 的置信度门控阈值扫描（⑦ 精度/召回/拒答率/F1）。** 这是我的拉档题，全真实，自信报细节。

### 🟢 我命中的参考答案要点

- ✅[真实] **置信度信号**：rag2 的 `eval_one()` 输出 `top1_score`、`margin = top1_score - top2_score`、检索结果数量。margin 是"敢不敢下结论"的指标。
- ✅[真实] **margin 作为门控指标**：margin 大说明 top1 与次优拉开距离，可下结论；margin 小说明竞争激烈，易错。我的杀手锏结论——C 路 rerank 把 margin 从 0.21 拉到 0.51，才让门控可用。
- ✅[真实] **门控设计：margin < 阈值 → 拒答/转人工**：rag2 的 `print_confidence_gating()` 报告 ⑦ 专门做这个——"margin < 阈值 → 拒答或转人工，避免低置信度错误"。
- ✅[真实] **阈值不拍脑袋：扫描找最佳点**：我的阈值扫描逻辑——`for thresh in thresholds: accepted = [(m,h) for m,h in arm_data if m >= thresh]`，算 precision = n_hit/n_acc、recall = n_hit/total_hits、rej = (total - n_acc)/total、F1 = 2*p*r/(p+r)，找 F1 最高点。

### 🟡 我能拿到的加分项

- ✅[真实] **光看命中率无法做门控，margin 才是门控可用性指标**：我的杀手锏——A 路 margin 0.21 根本做不了门控，C 路 margin 0.51 可以在 0.4 设阈值。这是 rerank 上生产的真正价值。
- ✅[真实] **阈值扫描输出权衡曲线**：我的报告 ⑦ 对每路方案扫描阈值，输出精度↑、召回↓、拒答率↑的权衡，标出"最高 F1"点。还能选 8 个代表性阈值展示（0.0/0.05/0.10/.../0.50）。
- ✅[真实] **rerank 提升 margin 让门控更可用**：我的报告解读——"改写/rerank 提升 margin 不是为了命中率，而是让门控更可用（相同精度下拒答率更低）"、"C 路 margin 分布更宽 → 可选阈值范围更大 → 门控灵活性最好"。

### 🔴 危险信号（主动规避）

- ❌ 别说"只看命中率无门控"——我有 margin 门控 + 阈值扫描。
- ❌ 别说"阈值拍脑袋定"——我扫描阈值找 F1 最高点，看精度/召回/拒答率权衡。
- ❌ 别说"top1 分数高就够了"——我强调 margin（top1 高但 top2 也高 = 不敢下结论）。

### 完整应答（口语稿）

> 置信度门控是 RAG 上生产的关键卡点——不该答的时候要敢拒答。我的置信度信号主要是 margin，就是 top1 分减 top2 分。margin 大说明 top1 和次优拉开距离，敢下结论；margin 小说明竞争激烈，容易错。
>
> 这道题我有实战。我的 rag2 评测报告 ⑦ 置信度门控评估专门做阈值扫描——扫描 margin 阈值，对每个阈值算精度、召回、拒答率、F1，找 F1 最高点。逻辑是：margin 大于等于阈值的接受，小于的拒答或转人工；接受里命中的算精度，命中数除以总命中数算召回，拒答比例算拒答率，F1 是精度召回的调和平均。我还会选 8 个代表性阈值展示权衡曲线，标出最高 F1 点。
>
> 这里要接 Q7 的杀手锏——光看命中率根本做不了门控。A 路 margin 0.21，阈值都没得选；C 路 rerank 把 margin 拉到 0.51，我才能在 0.4 那条线设阈值做拒答。所以 rerank 上生产的真正价值不是命中率，是让门控可用——相同精度下拒答率更低，C 路 margin 分布更宽，可选阈值范围更大，门控灵活性最好。这是"敢不敢下结论"和"能不能下结论"的区别，光看 top1 分数不够，top1 高但 top2 也高还是不敢下结论，margin 才是门控可用性指标。

---

## Q13　RAG vs 长上下文：什么时候该 RAG、什么时候塞进窗口

> **主接项目：四个项目的 RAG 实战判断 + SRE 的 retrieve_memory 分层记忆 + AI-PPT 的单文档长上下文。** 架构判断题，我有实战立场。

### 🟢 我命中的参考答案要点

- ✅[真实] **长上下文不替代 RAG 的理由**：成本（每轮塞全文 token 爆炸）、延迟（长输入推理慢）、可溯源（RAG 带来源，长上下文无法定位）、动态性（知识库更新 RAG 增量索引，长上下文要重塞）。这是我的立场。
- ✅[真实] **RAG 适合：知识库大且动态、需 citation、成本敏感、需精准召回**：我的 RAG 平台、SRE 的 retrieve_memory 都是这场景——知识库跨工单沉淀、需要 source 标注、高频调用成本敏感。
- ✅[真实] **长上下文适合：单文档深度理解、文档量小且固定、不要求溯源**：我的 AI-PPT 对单份用户文档做内容分析（mammoth/pdf-parse 解析后整篇喂 Agent1 分析内容），就是长上下文场景，不是 RAG。
- ⚙️[补全] **两者组合：RAG 召回 + 长上下文兜底**：设计思路，项目没明确组合实现。

### 🟡 我能拿到的加分项

- ⚙️[补全] **needle-in-haystack 衰减**：长上下文中间内容易被忽略，不是真"全记住"。设计认知，讲原理。
- ✅[真实] **成本对比**：RAG 每轮只塞 top-K 块（我的默认 topK=5），长上下文每轮塞全文，高频场景成本差几个量级。我的 embedding batch + 限速 + 成本预估（Q18）就是控 RAG 侧成本。
- ✅[真实] **可溯源是 RAG 本质优势**：我的 retrieve 带 doc_id/filename/source，长上下文无法定位信息来源——SRE 的 RCA 归档要能说"这个结论来自哪个 runbook"，长上下文做不到。

### 🔴 危险信号（主动规避）

- ❌ 别说"长上下文出来后 RAG 就过时了"——我给成本/延迟/溯源/动态性四维分析。
- ❌ 别说"一律 RAG"——我的 AI-PPT 单文档场景就用长上下文，不切块检索。
- ❌ 别说"不知道 needle-in-haystack"——我主动点出长上下文中间内容衰减。

### 完整应答（口语稿）

> 这道题我的立场很明确：长上下文不替代 RAG，两者适用不同场景。我从四个维度分析。第一成本——RAG 每轮只塞 top-K 块，我默认 topK=5；长上下文每轮塞全文，高频场景成本差几个量级。第二延迟——长输入推理慢，RAG 只塞相关块推理快。第三可溯源——RAG 带来源标注，我的 retrieve 带 doc_id、filename、source，长上下文无法定位信息来源；SRE 的 RCA 归档要能说"这个结论来自哪个 runbook"，长上下文做不到。第四动态性——知识库更新 RAG 增量索引，长上下文要重塞全文。
>
> 什么场景该 RAG？我的 RAG 平台和 SRE 的 retrieve_memory 都是——知识库大且动态、跨工单沉淀、需要 source 标注、高频调用成本敏感。什么场景该长上下文？单文档深度理解、文档量小且固定、不要求溯源。我的 AI-PPT 项目对单份用户文档做内容分析就是长上下文场景——mammoth 或 pdf-parse 解析后整篇喂 Agent1 分析内容，不切块不检索，因为就一份文档、要深度理解、不需要跨文档溯源。
>
> 要加分的话，我会主动点出 needle-in-haystack 衰减——长上下文不是真"全记住"，中间内容容易被忽略，这是长上下文的隐性成本。两者也可以组合：RAG 召回跨文档 + 长上下文做单文档深度，这是设计方向。所以不是"长上下文出来 RAG 就过时了"，是各自有适用场景，我四个项目正好分别用了两种思路。

---

# 四、评测与质量

## Q14　RAG 评测体系：命中率指标 vs 置信度指标

> **主接项目：rag2 的四件套报告 + 5 臂消融 + MRR/NDCG + 杀手锏结论；RAG 的 eval 模块（hit/margin/annotation）；SRE 的 11 枚举确定性评测。** 这是我的拉档核心题，三项目叠加，全真实。

### 🟢 我命中的参考答案要点

- ✅[真实] **命中率指标**：rag2 的 `eval_one()` 输出 `hit1`（Top-1 命中）、`hit3`（Top-3 命中）、`correct_rank`。RAG 的 eval 模块 `eval_results` 表记 `hit1`/`hit3`/`correct_rank`/`top1_score`/`margin`/`ranked_doc_ids`。
- ✅[真实] **置信度指标**：rag2 报告 ③ 置信度对比——`avg_score`（平均 Top1 分）+ `avg_margin`（平均 margin = top1 − top2）。RAG eval 同样记 margin。margin 衡量"敢不敢下结论"。
- ✅[真实] **两类指标缺一不可**：我的杀手锏——命中率三路全 100% 但 margin 从 0.21→0.51 翻 2.4 倍。光看命中率漏掉置信度故事，光看置信度不知道命中率基线。
- ✅[真实] **测试集对基线太简单时，优化收益转移**：我的归因——命中率早打满天花板，rerank 收益不在命中率在 margin。
- ✅[真实] **排序质量指标 MRR/NDCG 纯排名跨方案可比**：rag2 的 `mrr()`（1/rank 均值）+ `ndcg_at_k()`（二元相关性，1/log2(rank+1)），报告 ⑤ 专门说"只看排名不看分数，A/B/C 分数尺度差异不影响结果"。

### 🟡 我能拿到的加分项

- ✅[真实] **命中率 +0pp 但 margin 翻 2.4 倍的反直觉结论**：我的杀手锏故事，能背数字（0.21→0.51，2.4 倍），能讲"rerank 收益在敢不敢下结论不在命中率"。
- ✅[真实] **MRR/NDCG 跨方案可比的原因**：纯排名指标，不受分数尺度影响——我的报告 ⑤ 解读明确写了这一点。
- ✅[真实] **按难度分档评测**：rag2 报告 ⑥ 命中率按难度分档（easy/medium/hard/multi），`print_difficulty_breakdown()` 看改写/rerank 在哪个难度区间起作用。RAG eval 的 `difficulty` 字段（easy/medium/hard）同款。
- ✅[真实] **SRE 的 11 枚举确定性评测**：我的 SRE Agent 用 11 种 IncidentType 封闭枚举做等值比较，完全确定可复现，首期故意不做 LLM-as-judge——judge 引入第二个 LLM 变量，指标变化无法归因。这是另一类"分两类指标"的判断。

### 🔴 危险信号（主动规避）

- ❌ 别说"只报命中率一个数"——我有 hit + margin + MRR + NDCG + per-class。
- ❌ 别说"命中率打满就系统没问题"——我强调要看 margin 够不够做门控。
- ❌ 别用绝对分数横比不同方案——我主动点出尺度陷阱（Q16 详讲）。

### 完整应答（口语稿）

> RAG 评测我会分两类指标：命中率指标和置信度指标，缺一不可。命中率是 hit1、hit3、MRR、NDCG，衡量"有没有召回正确答案"；置信度是平均 top1 分和平均 margin，margin 就是 top1 减 top2，衡量"敢不敢下结论"。
>
> 这道题我有一个反直觉的实战结论，最想讲。我的 rag2 跑 A 裸向量、B 加改写、C 加 rerank 三路消融，命中率三路全 100%，归因 +0pp，看着 rerank 白接了。但置信度那栏 margin 从 0.21 到 0.31 到 0.51 翻了 2.4 倍。这说明测试集对基线太简单，命中率早打满天花板，rerank 的收益不在"有没有命中"，在"敢不敢下结论"。光看命中率会漏掉这一半的故事。
>
> 排序质量指标我会用 MRR 和 NDCG——MRR 是 1/rank 的均值，NDCG 是二元相关性用 1/log2(rank+1)。这两个纯看排名不看分数，所以 A/B/C 三路分数尺度不同（cosine vs cross-encoder）也不影响横比，这是它们跨方案可比的原因。我的报告 ⑤ 专门写了这一点。我还按难度分档评测——easy/medium/hard/multi，看改写和 rerank 主要在哪个难度区间起作用，通常 medium 和 hard 差异最显著。
>
> 另一个能挂的实战：我的 SRE Agent 评测用 11 种故障类型封闭枚举做等值比较，完全确定可复现，首期故意不做 LLM-as-judge。理由是 judge 引入第二个 LLM 变量，指标变化时无法归因是模型变了还是 judge 飘了。这跟 RAG 评测里"分两类指标"是同一种判断力——要可归因、要分维度，不能一个数下结论。

---

## Q15　评测数据隔离与防过拟合

> **主接项目：RAG 的 eval 数据物理隔离（eval_chunks vs kb_chunks）+ Golden Answer 文档级；SRE 的 eval datasets 独立 + 首期不做 judge；rag2 的 TEST_SET 独立。** 真实落地，自信说。

### 🟢 我命中的参考答案要点

- ✅[真实] **评测数据物理隔离**：RAG 的 eval 模块有独立表 `eval_test_sets`/`eval_test_cases`/`eval_experiments`/`eval_runs`/`eval_results`/`eval_chunks`，与生产 `kb_chunks`/`kb_documents` 分离。eval 数据不混入生产表。
- ✅[真实] **Golden Answer 文档级而非 chunk 级**：RAG eval 的 `gold_doc_ids` 是文档 ID 列表，不是 chunk ID——"召回同文档不同 chunk 算对"，避免 chunk 级别假阴性。rag2 的 TEST_SET 答案是 parent_id（范式级），同范式不同子块算对。
- ✅[真实] **防过拟合：评测集不参与调参**：我的 eval 是独立实验流程，跑完出报告，不把评测集反喂调参。
- ✅[真实] **LLM 生成评测用例 + 人工抽检**：RAG eval 的 `generateCases()` 用 LLM 从文档生成 query + gold_doc_ids + difficulty，但 source 标 'llm' 区分，需人工抽检校准。

### 🟡 我能拿到的加分项

- ✅[真实] **评测集参与调参 = 刷分**：我的立场——SRE 的 eval 刻意做成非 CI 指标，报告顶部声明"非 CI 指标"，防刷分。理由：非确定性指标做 merge gate 会有随机红绿。
- ✅[真实] **评测集随业务演进持续扩充**：SRE 新增 IncidentType 只要枚举加一行 + 补一个 case JSON，scorer/metrics 按枚举动态聚合不硬编码，扩充成本低。
- ✅[真实] **LLM-as-judge 归因难题**：SRE 首期不做 judge 的理由——judge 引入第二个 LLM 变量，指标变化无法归因是模型变了还是 judge 飘了。接口预留但不实现。

### 🔴 危险信号（主动规避）

- ❌ 别说"评测集混生产"——我物理隔离 eval 表和生产表。
- ❌ 别说"Golden Answer 用 chunk 级别"——我用文档级 gold_doc_ids，防假阴性。
- ❌ 别说"评测集长期不变参与调参"——我刻意非 CI、支持持续扩充。

### 完整应答（口语稿）

> 评测数据隔离我有真实落地。我的 RAG 平台 eval 模块有独立的表——eval_test_sets、eval_test_cases、eval_experiments、eval_runs、eval_results、eval_chunks，跟生产的 kb_chunks、kb_documents 物理分离。eval 数据不混入生产表，这是防过拟合的第一道关。
>
> 第二，Golden Answer 粒度要文档级而不是 chunk 级。我的 eval 的 gold_doc_ids 是文档 ID 列表，不是 chunk ID——召回同一文档的不同 chunk 都算对。如果用 chunk 级别，召回同文档不同 chunk 会算错，造成假阴性虚低分数。rag2 的 TEST_SET 答案是 parent_id 范式级，同范式不同子块算对，同款思路。
>
> 第三，防过拟合——评测集不能参与调参，否则就是刷分。我的 SRE Agent 评测刻意做成非 CI 指标，报告顶部明确声明"非 CI 指标"。理由是真实 LLM 非确定，做 merge gate 会有随机红绿，今天绿明天红门禁就废了。我的定位是"开发者主动跑，指导 prompt 和模型选型"。而且评测集要随业务演进持续扩充——SRE 新增一种故障类型只要枚举加一行加一个 case JSON，scorer 和 metrics 按枚举动态聚合不硬编码，扩充成本低。
>
> LLM 生成评测用例我能做，RAG eval 的 generateCases 用 LLM 从文档生成 query 加 gold_doc_ids 加 difficulty，但 source 标 lLM 区分，需人工抽检校准。LLM-as-judge 我首期故意不做——judge 引入第二个 LLM 变量，指标变化时无法归因是模型变了还是 judge 飘了，接口预留但不实现，这跟 RAG 评测数据隔离是同一种"防评测本身不可靠"的判断。

---

## Q16　离线消融实验设计：单变量递进、归因分解、尺度陷阱

> **主接项目：rag2 的 5 臂单变量递进 + 翻盘/翻车归因 + 尺度陷阱识别。** 全真实，这是我的实验设计硬功夫。

### 🟢 我命中的参考答案要点

- ✅[真实] **单变量递进**：rag2 的 `ARMS = [("A","A 裸向量"), ("B","B +改写"), ("C","C +rerank"), ("D","D 关键词"), ("E","E Hybrid")]`——A→B 只加改写，B→C 只加 rerank，每次只加一个变量。
- ✅[真实] **归因分解**：rag2 报告 ④ 命中率汇总 + 归因——`query 改写贡献 {top1[B]-top1[A]} pp`、`rerank 贡献 {top1[C]-top1[B]} pp`、`总提升 {top1[C]-top1[A]} pp`，把总提升拆到具体优化。
- ✅[真实] **翻盘/翻车归因**：rag2 报告 ② `print_attribution_cases()`——追踪"改写救场"（A 错 B 对）、"改写带偏"（A 对 B 错）、"rerank 救场"、"rerank 带偏"，不只看总提升还看每个 case 的翻盘翻车。
- ✅[真实] **尺度陷阱识别**：rag2 报告 ③ 明确标注"A/B 用 cosine 相似度（0~1），C 用 gte-rerank relevance_score，尺度不同，Top1 分数横比仅供定性参考，应以 ⑤ 排名指标为准"。我的杀手锏故事里就讲了第 2 条"卖东西的网站"C=0.43 看着倒退最多，实际 margin 反而最大——这是尺度陷阱的典型案例。

### 🟡 我能拿到的加分项

- ✅[真实] **平均 top1 分横比是视觉错觉**：我的实战——A=0.719、B=0.787、C=0.684，看着 C 反而降了，但 C 是 cross-encoder relevance_score 尺度不同，绝对值横比无意义；margin 是同尺度内相对差，横比才可比。我能讲清"比苹果重量和苹果甜度"的类比。
- ✅[真实] **归因分解粒度**：不只总提升，拆到每个优化的贡献 + 翻盘/翻车案例。我的报告 ② 逐条列出哪条 case 是改写救场、哪条是 rerank 带偏。
- ✅[真实] **多臂全景对比**：A/B/C/D/E 五臂同时对比向量/改写/rerank/关键词/hybrid，不只 A→B→C 单条链。

### 🔴 危险信号（主动规避）

- ❌ 别说"消融不控制变量一次改多个"——我单变量递进，A→B 只加改写，B→C 只加 rerank。
- ❌ 别说"绝对分数横比不同尺度方案"——我主动点出尺度陷阱，C 的 0.684 vs B 的 0.787 不能横比。
- ❌ 别说"只报总提升不拆归因"——我拆到改写贡献 +X pp、rerank 贡献 +Y pp，加翻盘翻车案例。

### 完整应答（口语稿）

> 消融实验我会严格控制变量 + 归因分解 + 识别尺度陷阱。我的 rag2 跑五臂单变量递进——A 裸向量、B 加改写、C 加 rerank、D 关键词、E Hybrid。A 到 B 只加改写一个变量，B 到 C 只加 rerank 一个变量，每次只动一个，这样归因才干净。
>
> 归因分解我会拆到每个优化的贡献。报告 ④ 命中率汇总写"query 改写贡献 +X pp（A 100% → B 100%）、rerank 贡献 +Y pp（B 100% → C 100%）、总提升 +Z pp"。但光看总提升不够，我还有翻盘翻车归因——报告 ② 逐条追踪哪条 case 是改写救场（A 错 B 对）、哪条是改写带偏（A 对 B 错）、哪条是 rerank 救场或带偏。比如我那次三路命中率全 100% 没有翻盘翻车，但如果有，我能精确说出哪条 case 是被改写救的还是被 rerank 带偏的。
>
> 尺度陷阱是这道题的高级判断力，我有实战。我的报告 ③ 置信度对比里，A 的平均 top1 分 0.719、B 是 0.787、C 是 0.684，看着 C 反而降了。但这是视觉错觉——A/B 用的是 cosine 相似度，0 到 1 的尺度；C 用的是 gte-rerank 的 relevance_score，cross-encoder 输出的相关性分，尺度天然不一样。直接横比 0.684 和 0.787 没有意义，就像比苹果重量和苹果甜度。我的报告里明确标注了这一点，并且说"应以 ⑤ 排名指标为准"——MRR 和 NDCG 纯看排名不看分数，跨方案可比。margin 横比也有意义，因为 margin 是同尺度内 top1 减 top2 的相对差。所以消融实验不控制变量没法归因，不识别尺度陷阱会得出错误结论，这两点我都踩过并有解。

---

# 五、工程化与成本

## Q17　多 Provider 适配与流式协议统一

> **主接项目：RAG 的 6 Provider 适配器 + 自建 SSE 7 事件协议。** 真实落地，自信报类名和事件类型。

### 🟢 我命中的参考答案要点

- ✅[真实] **适配器模式收敛 Provider 差异**：RAG 的 `AiService` 用 `ProviderAdapter` 接口（`buildRequestBody` / `parseChunk`），实现了 `deepseekAdapter` / `minimaxAdapter` / `qwenAdapter` / `doubaoAdapter` / `defaultAdapter`，`getAdapter(provider)` 按字符串选。差异收敛在 adapter——DeepSeek 的 `reasoning_content`、MiniMax 的 `mask_sensitive_info: true`、Qwen 的 `stream_options: {include_usage: true}` 都在各自适配器里。
- ✅[真实] **上层逻辑只依赖统一接口**：`streamChat()` 只调 `adapter.buildRequestBody()` 和 `adapter.parseChunk()`，不按模型名写 if-else。
- ✅[真实] **自建 SSE 7 事件协议**：chunk / reasoning / rag_sources / done / plan / image / error——把各家流式输出归一到这套。
- ✅[真实] **流式超时与空响应兜底**：`INACTIVITY_TIMEOUT = 90_000`（90s 无数据触发错误），`resetTimer()` 每收到数据重置；流结束但 `!fullText && !reasoningText` 时报"模型返回了空响应"，不显示空气泡。

### 🟡 我能拿到的加分项

- ✅[真实] **完美模型无关抽象不存在**：我自建 SSE 协议恰恰因为各家流式格式、reasoning 通道、system prompt 敏感度不一，必须自己定一层收敛。关键 prompt 仍需按模型微调。
- ⚙️[补全] **capability flag 显式管理差异**：我会把"是否支持并行 tool call、是否有 reasoning 通道、窗口多大"做成能力标志，让上层按 capability 而非按模型名分支。适配器是真的，capability 探测层是设计升级方向。
- ✅[真实] **流式空响应兜底**：`onDone` 里检查 `if (!fullText && !reasoningText) onError(new Error('模型返回了空响应'))`——防止配额不足或模型内部错误时前端永远转圈。

### 🔴 危险信号（主动规避）

- ❌ 别说"套个统一 SDK 就模型无关了"——我强调 prompt 和行为差异无法被 SDK 抹平，所以自建适配器层 + SSE 协议。
- ❌ 别把模型特定 hack 散落在业务逻辑——我全收敛在 adapter（mask_sensitive_info 在 minimaxAdapter，reasoning_content 在 deepseekAdapter）。
- ❌ 别说"流式无超时无空响应兜底"——我有 90s 不活跃超时 + 空响应报错。

### 完整应答（口语稿）

> 多 Provider 适配我用适配器模式。我的 RAG 平台 AiService 定义了 ProviderAdapter 接口，两个方法：buildRequestBody 构建请求体，parseChunk 解析 SSE 块。实现了五个适配器——deepseek、minimax、qwen、doubao、default，getAdapter 按 provider 字符串选。各家差异全收敛在 adapter：DeepSeek 有 reasoning_content 思维链字段，MiniMax 必填 mask_sensitive_info，Qwen 要 stream_options include_usage 才返回 token 统计。上层 streamChat 只调 adapter 的两个方法，不按模型名写 if-else。
>
> 流式这块我自建了一套 SSE 协议，七种事件类型——chunk、reasoning、rag_sources、done、plan、image、error，把各家格式各异的流式输出归一到这套。我自建这层恰恰说明一个判断：完美的模型无关抽象是不存在的，各家流式格式、reasoning 通道、system prompt 敏感度都不一样，必须自己定一层收敛，关键 prompt 仍需按模型微调。再加分的话，我会把能力标志做成 capability flag——是否支持并行 tool call、是否有 reasoning 通道、窗口多大——让上层按 capability 分支而不是按模型名写 if-else，这是设计升级方向。
>
> 工程兜底我很重视：流式有 90 秒不活跃超时，每收到数据重置计时器，防止豆包等模型沉默无响应；流结束但 fullText 和 reasoningText 都空时，报"模型返回了空响应，可能是订阅配额不足"，绝不显示空气泡让前端永远转圈。

---

## Q18　索引/嵌入缓存与成本控制

> **主接项目：RAG 的 embedding batch 25 + 200ms 限速 + eval embedding-cache + 成本预估 estimateEmbeddingCalls。** 真实落地，自信说。

### 🟢 我命中的参考答案要点

- ✅[真实] **批量嵌入分批 + 限速**：RAG 的 `EmbeddingService`：`BATCH_SIZE = 25`，批次间 `setTimeout(200ms)` 避免触发 SiliconFlow 频率限制。rag2 的 `EMBED_BATCH_SIZE = 10`。
- ✅[真实] **嵌入缓存**：RAG eval 模块有 `embedding-cache.ts`，按内容缓存嵌入结果，避免重复嵌入。
- ✅[真实] **成本预估**：RAG eval 的 `cost-estimate.ts` 有 `estimateEmbeddingCalls(estimatedChunks, strategy, prevStrategy)`——实验前预估 embedding 调用次数（建库 + 查询 × case 数），评估成本。rag2 的 README 也写了"embedding 1 次建库 24 条 + 每条 case 3 次 query × 10 case = 30 次，整体几分钱量级"。
- ✅[真实] **rerank 成本控制：只对粗排 top-N 精排**：rag2 的 `recall_k=8`，只对粗排 top-8 精排，不全量 rerank。

### 🟡 我能拿到的加分项

- ✅[真实] **嵌入缓存按 content hash**：eval 的 embedding-cache 按内容缓存，生产侧按 content hash 缓存是延伸——文档更新只重嵌变更块。
- ✅[真实] **成本预估在实验设计阶段**：我的 `estimateEmbeddingCalls` 在 `createExperiment` 时就算出每个 run 的预估调用次数，跑实验前就知道成本，评估值不值得。
- ✅[真实] **rerank recall_k 是成本与质量旋钮**：rag2 的 recall_k=8，太小精排没意义，太大成本爆炸——这是可调旋钮。

### 🔴 危险信号（主动规避）

- ❌ 别说"无嵌入缓存全量重嵌"——我有 eval embedding-cache，生产按 content hash 是延伸。
- ❌ 别说"批量嵌入不分批不限速"——我 batch 25 + 200ms 限速。
- ❌ 别说"rerank 全量精排"——我只对粗排 top-8 精排，recall_k 是旋钮。

### 完整应答（口语稿）

> 成本控制我有几个真实落地点。第一，批量嵌入分批加限速——我的 EmbeddingService batch size 25，批次间 sleep 200ms 避免触发 SiliconFlow 频率限制，按 index 排序保序取 embedding。第二，嵌入缓存——我的 eval 模块有 embedding-cache，按内容缓存嵌入结果，相同内容不重复嵌入；生产侧按 content hash 缓存是延伸，文档更新只重嵌变更块，不全量重嵌。第三，成本预估——我的 eval 模块有 cost-estimate，estimateEmbeddingCalls 在创建实验时就算出每个 run 的预估 embedding 调用次数，建库加查询乘 case 数，跑实验前就知道成本，评估值不值得跑。
>
> rerank 的成本控制我也做了——只对粗排 top-recall_k 精排，rag2 里 recall_k=8，不全量 rerank。这个 recall_k 是成本与质量的旋钮：太小精排没意义，太大成本爆炸。我的 rag2 README 里写了预估——embedding 1 次建库 24 条加每条 case 3 次 query 乘 10 case 等于 30 次，rerank 每条 case 1 次乘 10 等于 10 次，整体几分钱量级。所以成本控制不是一句"省钱"，是批量、限速、缓存、预估、粗排精排分段控制，每环都有具体手段和参数。

---

## Q19　生产 RAG 的可观测与 bad case 闭环

> **主接项目：SRE 的 25 事件 + tracing + failed_evidence_tools；Harness 的 bad-case-recorder skill + run_records 表；RAG eval 的 annotation（win/loss/tie）。** 这是我的拉档题，bad case 闭环真实落地。

### 🟢 我命中的参考答案要点

- ✅[真实] **可观测：每次检索记录 query、召回、分数、margin**：RAG eval 的 `eval_results` 表记 `hit1`/`hit3`/`correct_rank`/`top1_score`/`margin`/`ranked_doc_ids`——每次检索的可观测字段。SRE 的 25 种 EventBus 事件 + `IncidentToolAudit` 表记每次工具调用的 request/response/latency/adapter。
- ✅[真实] **bad case 闭环：线上错误 → 记录 → 回流评测**：Harness System 有 `bad-case-recorder` skill，把 bad case 结构化写入 `bad_cases` 表（spec_id / run_id / case_type / case_desc / evidence / root_cause / severity / status）。SRE 的 eval datasets 可持续加 case，新增 IncidentType 只要补一个 case JSON。
- ✅[真实] **bad case 结构化记录**：Harness 的 `bad_cases` 表字段——case_type / case_desc / evidence / root_cause / severity / status，可检索可统计。SRE 的 `terminal_reason = {code, stage, message, failed_tools}` 是同款结构化错误归因。
- ✅[真实] **bad case 归因分类**：RAG eval 的 `annotation` 字段分 `win`/`loss`/`tie`/`baseline`——`getExperimentResults()` 对每条 case 标注是赢、输、平、基线，区分"检索错"的归因。

### 🟡 我能拿到的加分项

- ✅[真实] **bad case 不回流 = 评测集不进化 = 系统不进步**：我的立场——Harness 的 bad-case-recorder 把线上错误回流，SRE 的 eval datasets 支持持续加 case，这是闭环。不回流就退化。
- ⚙️[补全] **采样与全量分层存储**：SRE 目前全量事件落 DB，采样分层是延伸——正常采样、异常全量，平衡成本。
- ✅[真实] **bad case 归因分类（检索错/生成错/评测错）**：RAG eval 的 annotation（win/loss/tie）+ SRE 的 terminal_reason.code（RISK_BLOCKED/NO_PLAN/AUTOMATION_CAPABILITY_UNAVAILABLE）分别对应不同优化路径。

### 🔴 危险信号（主动规避）

- ❌ 别说"只记最终结果无从复现"——我有 eval_results + EventBus + IncidentToolAudit 全链路。
- ❌ 别说"无 bad case 闭环"——我有 bad-case-recorder skill + bad_cases 表 + eval datasets 持续扩充。
- ❌ 别说"无指标看板"——Harness 有 stats API 聚合，SRE 有 25 事件可观测。

### 完整应答（口语稿）

> 生产 RAG 的可观测我会做全链路记录加 bad case 闭环。可观测上，我的 RAG eval 的 eval_results 表记每次检索的 hit1、hit3、correct_rank、top1_score、margin、ranked_doc_ids，SRE 的 25 种 EventBus 事件加 IncidentToolAudit 表记每次工具调用的 request、response、latency、adapter 模式。出问题能复现检索过程，不是只记最终结果。
>
> bad case 闭环是这道题的重点。我的 Harness System 有一个 bad-case-recorder skill，把线上负反馈或低置信 case 结构化写入 bad_cases 表——字段有 case_type、case_desc、evidence、root_cause、severity、status，可检索可统计。这个闭环的关键是：bad case 不回流等于评测集不进化等于系统不进步。所以 bad case 记下来之后要回流评测集——SRE 的 eval datasets 支持持续加 case，新增一种故障类型只要补一个 case JSON，scorer 和 metrics 按枚举动态聚合不硬编码。这样线上错误回流评测，验证优化是否修复，形成闭环。
>
> bad case 归因分类我也做了。RAG eval 的 annotation 字段分 win、loss、tie、baseline——对每条 case 标注是赢、输、平、还是基线，区分检索错的归因。SRE 的 terminal_reason 有 code、stage、message、failed_tools，code 分 RISK_BLOCKED、NO_PLAN、AUTOMATION_CAPABILITY_UNAVAILABLE 这些，分别对应不同优化路径——是检索没召回、是生成错了、还是评测本身有问题。采样和全量的分层存储我目前全量落库，分层采样是延伸方向。

---

## Q20　RAG 系统的演进路线：从 baseline 到生产级的迭代判断

> **主接项目：rag2 的演进路线实战（A 裸向量 → B 改写 → C rerank → 门控 → 闭环）+ 四项目的分阶段判断。** 这是我的收官题，有实战演进路线和判断框架。

### 🟢 我命中的参考答案要点

- ✅[真实] **演进路线**：rag2 的 ARMS 就是这条线——A 裸向量（baseline）→ B +改写（补 vocabulary gap）→ C +rerank（精排）→ 置信度门控（Q12 阈值扫描）→ bad case 闭环（Q19）。我真实走过这条演进路。
- ✅[真实] **每步优化用消融实验量化贡献**：rag2 的归因分解——改写贡献 +X pp、rerank 贡献 +Y pp。不盲目堆优化，每步都量化。
- ✅[真实] **业务阶段匹配**：rag2 是离线实验（demo 阶段），RAG 平台是生产（加了 SSE + 适配器 + eval 模块），SRE 是生产级（加了 checkpoint + 审批 + 25 事件）。不同阶段不同深度。
- ✅[真实] **迭代原则：先做评测基线再逐个加优化**：我的做法就是先建 A 基线，再 B、再 C，每步评测验证，不做无评测的黑盒堆叠。

### 🟡 我能拿到的加分项

- ✅[真实] **优化必须评测驱动**：我的立场——rag2 五臂消融 + 归因分解就是评测驱动，不做无归因的黑盒堆叠。每个优化都要回答"贡献了几 pp、翻盘还是翻车"。
- ✅[真实] **不同业务场景演进终点不同**：FAQ 够 baseline 加改写，知识密集型要 rerank 加门控加闭环。我的 RAG 平台是知识库问答（要 rerank），AI-PPT 是单文档生成（不用 RAG），SRE 是故障处置（要确定性评测加审批）——不同场景演进终点不同。
- ✅[真实] **测试集难度要随系统进化**：我的杀手锏结论——测试集对基线太简单导致命中率打满天花板，要换更难的测试集（口语更绕、范式相互更接近、加干扰类目）才能看出优化的命中率收益。这是评测集本身要演进的判断。

### 🔴 危险信号（主动规避）

- ❌ 别说"一上来全堆 rerank+hybrid+改写+门控"——我先建 A 基线，逐个加优化并验证。
- ❌ 别说"优化越多越好"——我量化每个优化的贡献，没贡献的砍掉。
- ❌ 别说"无演进视角一锤定音"——我随业务阶段调整深度，不同场景终点不同。

### 完整应答（口语稿）

> RAG 系统的演进我有实战路线，不是一锤定音。我的 rag2 走过这条线：A 裸向量做 baseline，B 加 query 改写补 vocabulary gap，C 加 rerank 做精排，然后置信度门控做拒答转人工，最后 bad case 闭环持续优化。每一步都用消融实验量化贡献——改写贡献几个 pp、rerank 贡献几个 pp、有没有翻盘翻车，不盲目堆优化。
>
> 核心原则是评测驱动，不做无归因的黑盒堆叠。我先建 A 基线，再逐个加 B、C，每步评测验证。每个优化都要回答两个问题：贡献了几 pp、是翻盘还是翻车。没贡献的砍掉，翻车多的要重新设计。这不是"优化越多越好"，是"每个优化都要有数据支撑"。
>
> 不同业务场景的演进终点不一样。FAQ 场景可能 baseline 加改写就够；知识密集型要 rerank 加门控加闭环。我的四个项目正好是不同场景：RAG 平台是知识库问答要 rerank，AI-PPT 是单文档生成不用 RAG 走长上下文，SRE 是故障处置要确定性评测加审批。演进终点按业务价值分配，不是全堆。
>
> 最后一个判断是测试集本身要随系统进化。我的杀手锏结论就是——测试集对基线太简单，命中率早打满天花板，rerank 的命中率收益看不出来。要换更难的测试集：口语更绕、范式相互更接近、加干扰类目，才能看出优化的命中率收益。否则系统在进步但评测看不出来，等于盲飞。所以演进路线不只是加优化，还包括评测集本身要进化，这是闭环的最后一环。

---

_应答文档完 · 全 20 题 / 5 模块 · 配套《RAG / 检索增强面试官评分手册》_
