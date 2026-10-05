---
title: 100个面试题-Rag-SolThink版
date: 2026-10-05
tags:
  - 面试
  - AI应用工程
  - RAG
  - 小说转剧本
  - SolThink
---

> 来源：[[100个面试题-Rag-面试教练优化版]]
>
> 方法论：Miles 面试方法——围绕“实际负责 → 为什么这样做 → 替代方案 → 如何验证 → 失败与边界 → 规模/约束变化”追问。
>
> 项目依据：GitHub `lobokiol/NovelToVideo`，重点核对 `rag_documents.py`、`rag_runtime.py`、`rag_index.py`、`query_intent.py`、`query_intent_llm.py`、`embedding.py`、`vector.py`、`chat.py`。
>
> **版本标记：SolThink版**。本版不是把 100 道题简单补答案，而是把每题改造成“可直接口述 + 源码证据 + 反向追问 + 诚实边界”的项目深挖题。

# 小说转短剧 AI 平台：100 个 RAG 面试题（SolThink版）

## 0. 先记住这套回答规则

### 0.1 四层证据等级

| 标记 | 含义 | 面试口径 |
|---|---|---|
| `A 源码事实` | 当前仓库能直接定位 | 可以直接说“代码里就是这样实现的” |
| `B 源码推断` | 由多个函数的执行关系得到 | 说清“从调用链可以确认” |
| `C 优化方案` | 当前项目没有，属于改进设计 | 必须说“当前没有，我会这样改” |
| `D 未验证` | 需要真实环境、资产、数据或压测 | 明确说“仓库无法证明” |

### 0.2 个人贡献红线

本项目源码只能证明“系统怎么做”，不能自动证明“你亲自写了多少”。面试时不要说“整个 RAG 都是我做的”，而是把个人贡献落到真实写过、改过、测过的模块。可以使用的证据点包括：

- `backend/app/services/screenwriting/rag_documents.py`
- `backend/app/services/screenwriting/rag_runtime.py`
- `backend/app/services/screenwriting/rag_index.py`
- `backend/app/services/screenwriting/query_intent.py`
- `backend/app/services/screenwriting/query_intent_llm.py`
- `backend/app/core/agent/embedding.py`
- `backend/app/core/harness/memory/vector.py`
- `backend/app/services/screenwriting/chat.py`

### 0.3 项目一句话

> 这是一个面向“小说转短剧”场景的项目级 RAG。它把项目配置、小说基础信息、章节、结构化事件、视觉风格和导演手册统一成 `RagDocument`；查询侧先做意图解析，再按 metadata / event / vector / lexical 分层路由，最后把命中资料注入聊天或阶段生成上下文。

### 0.4 当前项目最重要的源码事实

| 项目 | 当前实现 |
|---|---|
| 统一文档 | `RagDocument(source_id, source_type, title, content, metadata)` |
| RAG 来源 | `project / novel / chapter / event / art_style / director_manual` |
| 默认 Chunk | 900 字符，overlap 120 |
| Embedding | `bge-small-zh-v1.5`，本地 ONNX Runtime，CPU，512 维 |
| Embedding 上限 | `agent_embedding_max_tokens=512`，这是 Embedding 输入上限，不是 LLM 上下文窗口 |
| Embedding batch | 8 |
| Chunk 写入 batch | 64 |
| warmup 文档 batch | 32 |
| 向量最低分 | 0.35 |
| 默认最终召回上限 | 50 |
| 向量候选 | 最多 `3 * limit`，默认最多 150 |
| 合并 | RRF，常数 60 |
| 独立 BM25 | 当前没有 |
| 独立 Cross-Encoder Reranker | 当前没有源码证据 |
| 意图模型 | 当前项目 `text_model`，超时默认 2 秒；失败回退本地规则 |
| 索引刷新 | API 进程内 `asyncio.create_task`，同进程 `_warmup_tasks` 去重 |
| Redis Worker | 通用任务系统存在，但 RAG warmup 当前不是 Redis Worker 任务 |
| 版本化索引 / ACTIVE 指针 | 当前没有 |
| Token 级上下文预算 | 当前没有 |
| RAG 失败是否阻断聊天 | 当前不会；准备异常转为空上下文，聊天继续 |

---

## 一、职业方向与个人贡献（1–10）

### 1. 你现在希望从事哪类 AI 应用工程工作？RAG 项目中哪段经历最能证明你的匹配度？

**SolThink口述**

如果目标岗位是 AI Application Engineer，我会把能力证明放在“检索增强链路的工程落地”而不是模型名上。这个项目能证明的核心是：把小说场景拆成不同知识源，统一为 `RagDocument`，再用意图解析和分层检索解决章节、人物、事件、配置等不同问题。真正能证明匹配度的，是我能解释一条查询为什么走某条路径、失败后怎么退化，以及如何验证检索质量。

**源码证据**

`rag_documents.py` → `rag_index.py` → `rag_runtime.py` → `chat.py`。

**追问准备**

面试官很可能继续问：“哪几个函数是你本人写的？”这一题必须把“项目事实”和“个人贡献”分开。

---

### 2. 请用两分钟说明小说转短剧平台里的 RAG 帮用户解决什么问题，用户输入和最终得到什么？

**SolThink口述**

用户不是只问一个普通聊天问题，而是会问“第几章发生了什么、人物冲突是什么、最后几章有什么变化、项目配置是什么、视觉风格是什么”。这些问题如果只靠模型记忆，很容易编造，所以系统先判断是否真的需要原著资料。需要时，查询会进入 `prepare_rag_context`，得到结构化命中，再把上下文注入生成链路。最终输出是结合当前项目资料的回答或阶段生成结果。

**源码证据**

`chat.py::_should_retrieve_chapters`、`rag_runtime.py::prepare_context`、`rag_runtime.py::build_system_prompt_with_rag`。

---

### 3. 这项 RAG 能力处于什么使用或部署阶段？你有哪些证据能确认，而不是从代码里推断用户已在使用？

**SolThink口述**

从仓库最多能证明代码链路、运行时诊断和持久化索引机制存在，不能仅凭源码证明已经有多少用户、多少请求、多少准确率或者线上 SLA。要证明“真的在用”，我需要补充真实流量、部署记录、日志、监控或评测报告。面试时我会主动把“源码完成度”和“线上业务证据”分开。

**源码边界**

仓库可以证明 `runtime` 会返回 `hitCount`、`retrievalMode`、`retrievalStrategy`、失败原因和 server timings，但业务采用量仍属于未验证证据。

---

### 4. 你本人负责了哪些 RAG 模块？请指出你写过、调过或验证过的具体代码和交付物。

**SolThink口述**

这一题我不会泛化成“我负责整个 RAG”。我会按真实参与范围，把贡献落到具体文件和函数。例如，如果我实际参与的是检索编排，我就讲 `rag_index.py` 的路由、向量召回、词法兜底和 RRF；如果还参与索引刷新，就再讲 `rag_runtime.py` 的 fingerprint、incremental refresh 和 warmup。每项贡献都配一个可复现样例、一次失败案例或者一条测试证据。

**面试原则**

源码是项目证据，不等于个人归属证据。不要把“仓库存在”直接说成“我实现”。

---

### 5. 选一次你亲自追踪过的查询，从请求进入到上下文交给生成模型，按实际执行顺序讲清楚。

**SolThink口述**

我会按执行顺序讲：首先 `chat.py` 通过 `_should_retrieve_chapters` 判断这轮是否需要章节检索；需要时调用 `prepare_rag_context`。`rag_runtime.py` 并发加载 RAG 文档和解析查询意图，然后计算文档 fingerprint；如果索引指纹不一致，只调度后台 warmup，不阻塞本轮。接下来调用 `rag_index.retrieve_context`，按 metadata、event、vector、lexical 分层检索。得到结果后，普通聊天通过 `build_system_prompt_with_rag` 把 RAG 文本注入系统提示；阶段生成则把 RAG 文本和结构化章节事件一起交给阶段 Agent。

**关键链路**

`chat._build_chat_context` → `rag_runtime.prepare_context` → `query_intent` → `rag_index.retrieve_context` → `build_system_prompt_with_rag` / `build_stage_system_prompt` → `agent.run_chat`。

---

### 6. 这条链路中最困难的故障或质量问题是什么？你怎样从现象走到根因？

**SolThink口述**

我会先把错答拆成“没检索、检索错、排序错、上下文丢失、生成忽略”五层，而不是上来改 Prompt。这个项目里向量不可用时会记录失败原因并走 lexical；意图模型异常会回退本地解析；RAG preparation 异常则转为空上下文继续聊天。因此排障时我会先看 `retrievalMode`、`retrievalStrategy`、`vectorHitCount`、`lexicalHitCount`、`intentParserFailure` 和 `failureReason`，再对照命中内容。

**证据**

`rag_runtime.py` 和 `rag_index.py` 的 runtime diagnostics。

---

### 7. 你做过哪个会影响检索质量、延迟或成本的决定？当时有哪些备选方案，为什么没选？

**SolThink口述**

这个项目一个核心判断是“不让所有消息都跑完整 RAG”。`chat.py` 先判断消息是不是章节、人物、剧情、事件等内容；基础配置类问题可以直接跳过章节检索。检索内部又不是一条路走到底，而是先做低成本的 metadata / event 精确路径，向量路径失败再词法兜底。这样做的核心不是为了堆组件，而是把不同查询类型匹配到不同成本。

**追问**

为什么不用所有请求都向量检索？回答时要落到延迟、上下文噪声和查询确定性。

---

### 8. 哪些关于 RAG 的描述是你已经用代码或实验验证的？哪些只是从文档、框架或设计意图得知？

**SolThink口述**

我能从源码确认的包括：900/120 切块、512-token Embedding 上限、向量最多取 150 个候选、0.35 阈值、按 `source_id` 去重、RRF 常数 60、metadata/event/vector/lexical 的分层路由，以及 warmup 不阻塞聊天。至于“准确率提升多少、线上节省多少时间、用户满意度提升多少”，没有真实评测或业务数据就不能说。

**原则**

源码行为和业务效果必须分开陈述。

---

### 9. 如果没有真实使用量、准确率或省时数据，你会如何陈述目前的成果，并设计最小的验证实验？

**SolThink口述**

我会说“已经完成可运行的项目级 RAG 链路，并具备分层检索、增量索引和故障回退，但业务效果尚未用独立数据集验证”。最小实验先固定模型、Chunk、阈值和路由，只拿一批人工标注查询，分别跑 vector-only、vector+lexical、full routing 三个版本，比较 Recall@K、MRR、Precision@K 和端到端延迟。

**不能说**

“召回率提升了 20%”，除非能给出基线、样本量、分子分母和测试条件。

---

### 10. 目前 RAG 链路最需要改进的地方是什么？你会先补证据还是先改架构？为什么？

**SolThink口述**

我会先补证据，再改架构。当前最明确的工程缺口是没有版本化索引和原子切换、没有 Token 级上下文预算，也没有独立 BM25 和 Cross-Encoder reranker。优先级上先建立 Golden Set 和延迟/错误分类基线，然后再做单变量实验；否则架构越复杂，越难证明到底是哪一层带来了收益。

**优化方向**

P0 是索引一致性和可观测性；P1 是检索质量和上下文控制；P2 才是进一步扩规模。

---

## 二、业务问题与整体架构（11–20）

### 11. 小说转剧本里，哪些用户问题必须依赖原文或项目资料？哪些问题无需检索也能处理？

**SolThink口述**

凡是需要回答当前项目事实的问题，都应该优先依赖项目资料，例如章节、人物关系、剧情、事件、原著信息、视觉风格和导演手册。纯配置确认或不涉及原著的基本问答，可以不走章节 RAG。当前代码甚至显式把项目配置类消息和普通基础问答从章节检索路径里排除。

**源码证据**

`chat.py::_should_retrieve_chapters`、`_is_basic_config_lookup_message`。

---

### 12. 为什么不能对每条聊天消息都运行完整 RAG？请结合延迟、模型调用、上下文长度和答案正确性讨论。

**SolThink口述**

因为有些请求本来就是确定性的配置问题，不需要加载章节资料。完整 RAG 会增加文档读取、fingerprint 计算、意图模型调用、向量查询和上下文拼装成本，同时还会把无关资料带给生成模型，增加噪声。当前项目已经用 `_should_retrieve_chapters` 做了第一层路由，所以这是一个明确的工程决策，而不是概念上的“RAG 应该按需调用”。

---

### 13. 请从“收到用户问题”开始，说明查询意图解析、候选资料检索、结果筛选、提示词注入和生成分别由谁负责。

**SolThink口述**

请求进入 `chat.py` 后，先判断是否触发章节 RAG；之后 `rag_runtime.py` 负责文档加载、意图解析、fingerprint 检查和索引调度；`query_intent.py/query_intent_llm.py` 负责把自然语言变成结构化检索意图和工具计划；`rag_index.py` 负责具体检索和排序；最后 `rag_runtime.py` 或 `chat.py` 把结果注入系统提示或阶段提示，再交给 Agent 生成。

**最容易被追问的点**

“意图解析”和“Agent 工具调用”不是一回事：前者先生成内部检索计划，后者才是生成阶段的工具运行。

---

### 14. 当前 RAG 是哪几个 Python 模块协作完成的？每个模块的边界是什么？请用一次真实查询定位代码。

**SolThink口述**

`rag_documents.py` 负责把业务数据标准化成 `RagDocument`；`query_intent.py` 和 `query_intent_llm.py` 负责意图；`rag_runtime.py` 负责运行时准备、fingerprint、warmup 和上下文注入；`rag_index.py` 负责切块、索引和检索编排；`embedding.py` 负责本地 Embedding；`vector.py` 负责 Chroma 和词法基础能力；`chat.py` 负责是否触发 RAG 以及最终 Agent 链路。

**面试讲法**

不要按文件名背，要按“数据流”讲。

---

### 15. 文档为什么统一为 RagDocument？source_id、source_type、content 和 metadata 分别参与什么环节？

**SolThink口述**

统一结构的价值是把异构资料变成一套检索接口。`source_id` 用来标识原始资料并参与去重和 Chunk ID；`source_type` 决定路由和展示标签；`content` 是真正进入切块和词向量的正文；`metadata` 保存章节序号、人物、事件类型、来源标识等结构化字段，供 metadata lookup、event selector 和过滤使用。

**源码证据**

`core/rag/documents.py::RagDocument`。

---

### 16. 项目索引以项目作为隔离范围时，读取请求如何校验当前用户对项目的访问权？向量库隔离与 API 鉴权有什么区别？

**SolThink口述**

访问控制首先发生在业务层，`chat.py` 的 `_load_chat_project` 调用 `project_service.get_project_or_raise(session, project_public_id, current_user_public_id)`。RAG 的 Chroma 隔离则通过项目级 `rag_isolation_key` 和项目目录限制召回范围。两者职责不同：鉴权决定“用户能不能访问项目”，向量隔离决定“检索时在哪个索引域里找”。向量隔离不能替代权限校验。

**源码证据**

`chat.py::_load_chat_project`、`rag_runtime.py::build_project_rag_isolation_key`。

---

### 17. 用户问项目名、作者、指定章节、人物冲突或开放式剧情问题时，分别适合走哪些检索路径？

**SolThink口述**

项目名、作者这类确定字段适合 metadata lookup；“第 10–15 章”这类章节范围走 chapter index metadata；人物冲突等结构化事件问题优先 event JSON；开放式剧情问题才更依赖 chapter vector + lexical。核心思想是优先使用更确定、更便宜的结构化路径，再使用语义检索解决开放查询。

---

### 18. 当前系统为什么按元数据、结构化事件、语义向量和词法兜底分层？每层的输入和成功条件是什么？

**SolThink口述**

第一层 metadata 依赖章节范围或 field lookup 意图；第二层 event 依赖 `event_selector` 并在 event 文档上做结构化匹配；第三层 vector 从 Chroma 取候选并按最低分过滤；最后 lexical 根据词法重合和精确命中补救。成功条件也不同：metadata/event 有合法命中就可以提前返回；vector 有命中则融合 lexical；vector 空或不可用则进入 lexical fallback。

---

### 19. 精确命中后提前返回有哪些优点？哪些复合问题可能因此错过其他通道的补充证据？

**SolThink口述**

优点是确定性查询不会为了一点点额外收益再跑昂贵的向量路径，而且答案更容易解释。问题是它是一种 early return：例如一个请求既包含明确章节范围，又包含需要跨章节语义补充的问题，metadata 命中后可能直接结束，后面的 vector 证据不会进入本轮上下文。因此后续优化可以把“确定性字段答案”和“开放式补充证据”拆成双阶段路由，而不是所有精确命中都提前终止。

---

### 20. 请画出当前架构，并标出业务数据库、Chroma、Embedding、意图模型、生成模型和 API 进程分别在哪个数据边界上。

**SolThink口述**

可以用下面这张图：

```text
用户请求
   |
   v
chat.py  ---- 业务权限 / 会话状态
   |
   +--> _should_retrieve_chapters
   |
   v
rag_runtime.py
   |\
   | +--> 文档加载（DB）
   | +--> 意图解析（project.text_model，失败回退规则）
   | +--> fingerprint / warmup
   |
   v
rag_index.py
   |\
   | +--> metadata / event
   | +--> Chroma <---- Local ONNX Embedding
   | +--> lexical fallback
   | +--> RRF
   |
   v
RAG Context
   |
   +--> 普通聊天：系统提示 + Agent
   +--> 阶段生成：RAG + chapter events + stage Agent
```

**关键边界**

数据库是业务事实源；Chroma 是检索索引；Embedding 是编码层；意图模型是检索路由层；生成模型负责最终答案，不应该被当成事实数据库。

---

## 三、数据来源、文档模型与索引维护（21–30）

### 21. 当前 RAG 文档从哪些业务数据和 Markdown 资料生成？每种来源提供什么类型的知识？

**SolThink口述**

`rag_documents.py::load_screenwriting_knowledge_documents` 当前会组合六类来源：项目基础信息、小说基础信息、章节文档、结构化事件文档、视觉风格 Markdown、导演手册 Markdown。这里的设计目的不是“把所有东西拼成长文档”，而是把不同语义角色分开，让路由能知道这是章节事实、事件结构还是创作指导资料。

**源码证据**

`build_project_basic_documents`、`build_novel_basic_documents`、`build_chapter_documents`、`build_event_documents`、`build_project_skill_documents`。

---

### 22. 章节原文、章节事件和结构化事件各自适合检索什么问题？为什么不把它们粗略拼成单一长文档？

**SolThink口述**

章节资料适合回答原文位置和具体剧情；事件文档更适合人物、场景、冲突、结果这类结构化问题。分开后，event selector 可以在结构化字段上打分，而 chapter vector 负责开放语义检索。这样做还能避免一个大文档把不同知识类型的召回粒度混在一起。

---

### 23. source_id 如何避免不同资料互相覆盖？同一资料变更时，哪些标识应保持稳定？

**SolThink口述**

源码中不同类型带前缀：例如 `project:...`、`novel:...`、`chapter:...`、`event:...`。章节通常以 `chapter_public_id` 作为稳定身份，事件进一步带章节和 sequence。Chunk ID 则由 `sha256(source_id)[:16] + chunk_index` 生成，所以源文档身份稳定时，同一位置的 Chunk 才可能稳定。真正要改进的是把 Chunk 身份从“位置依赖”升级成“内容/语义边界稳定”的 ID。

---

### 24. metadata 应保存哪些可过滤字段，例如章节序号、人物、来源类型和版本？哪些字段不适合只放在正文里？

**SolThink口述**

适合放 metadata 的是机器需要精确判断的字段，比如 `chapter_index`、`chapter_public_id`、`chapter_id`、`event_sequence`、`characters`、`scenes`、`organizations`、`event_type`、`conflict`、`outcome`。如果这些信息只藏在正文里，就必须再次做字符串匹配或让向量相似度承担本不擅长的精确过滤任务。

---

### 25. 文档内容更新后，如何发现变化、重算相关 Chunk、跳过未变化内容并删除失效 Chunk？

**SolThink口述**

runtime 先对整批 `RagDocument` 计算 fingerprint；指纹变化就调度 warmup。warmup 进入 incremental refresh 后重新切 Chunk，使用 `chunk_content_hash` 判断“同 ID + 同内容”就跳过，变化的则 upsert；同时收集本轮所有 `indexed_ids`，最后删除当前集合里不再存在的旧 Chunk。这里不是“整本小说全部重新 Embedding”，而是尽量跳过未变化 Chunk。

**源码证据**

`rag_runtime.py::_RagDocumentsFingerprintBuilder`、`rag_index.py::upsert_documents`、`delete_stale_documents`。

---

### 26. 如果稳定 ID 依赖 Chunk 序号，在文档头部插入一段文字会造成什么后果？如何改进稳定性？

**SolThink口述**

因为当前 Chunk ID 依赖 `chunk_index`，头部插入内容会让后续 Chunk 位置整体右移，哪怕原文大部分没有变，也可能产生大量“ID 变了”的 Chunk，导致重复 Embedding 和索引抖动。改进方案可以使用语义段落 ID、内容哈希或“父节点 ID + 局部序列”混合方案；但这属于当前项目没有的优化，不应说成已经实现。

---

### 27. 内容哈希能减少哪些重复工作？它不能解决什么并发或跨存储一致性问题？

**SolThink口述**

Chunk 内容哈希能判断同一个位置上的内容有没有变化，从而跳过重复 upsert/Embedding。它解决的是“内容是否相同”的重复劳动，不解决“两个 API 进程谁有权更新索引”、构建失败后如何原子回滚、DB 与 Chroma 是否处在同一版本这些问题。后者需要分布式协调、持久任务状态和版本化索引。

---

### 28. 当前分批加载文档、分批写入 Chunk 和分批计算 Embedding 分别控制什么资源？这些默认值如何用数据调节？

**SolThink口述**

warmup 文档 batch 默认 32，控制数据库读取时的一次性持有量；Chunk 写入 batch 默认 64，控制向量库 upsert 的批量；Embedding batch 默认 8，控制 ONNX 推理的内存和吞吐。调参不能凭感觉，要观察单批内存、Embedding 吞吐、Chroma 写入耗时和 P95/P99，再选择不会让尾延迟抖动的点。

**默认配置**

`32 / 64 / 8`。

---

### 29. 页面预热和查询发现资料变化后，当前索引刷新由哪个进程和任务机制执行？它是否经过 Redis Worker？

**SolThink口述**

当前 RAG warmup 是 `ScreenwritingRagRuntime.schedule_index_warmup` 通过 API 进程内 `asyncio.create_task` 调度的，同一个进程用 `_warmup_tasks` 按 `rag_isolation_key` 做去重。仓库确实存在通用 Redis Stream 任务基础设施，但当前 RAG warmup 代码没有走那个 Worker 链路。因此多实例部署时，这个去重能力并不能跨进程生效。

**这是面试高分点**

主动把“项目有 Redis”与“RAG warmup 使用 Redis Worker”分开。

---

### 30. 如果刷新到一半时 API 进程崩溃，会留下什么状态？重启后如何确认索引可用或重新构建？

**SolThink口述**

当前没有事务式的索引切换。如果是非 incremental rebuild，warmup 会先 `clear_scope` 再逐批写入，进程崩溃可能留下部分新索引；如果是 incremental refresh，已经 upsert 的新 Chunk 也可能保留，而后续 Chunk 仍是旧版本。最关键的是 fingerprint 只有整个 warmup 成功结束后才写回，因此重启后会再次发现指纹不匹配并重新安排刷新，但当前项目没有 V1/V2 的原子回滚机制。

**优化方案：C级**

采用 V1 ACTIVE / V2 BUILDING / V2 VALIDATING / V2 READY / atomic switch，并保留旧版本回滚。

---

## 四、切块与 Embedding（31–40）

### 31. 默认按 900 个字符切块、重叠 120 个字符是如何配置的？你会怎样验证它适合小说文本？

**SolThink口述**

当前默认值是 900/120，对应滑动步长 780 字符。验证时我不会直接说“这个值合理”，而是用包含人物指代、对话、场景转换、因果链和长事件的样本做 Chunk 参数 sweep，例如比较 600/100、900/120、1200/180，分别测 Recall@K、MRR、Precision@K 和上下文长度。

---

### 32. 字符数和 Token 数有什么区别？最大 Token 限制可能让一个 900 字符 Chunk 的哪些内容没有进入 Embedding？

**SolThink口述**

字符数是切块层面的长度，Token 是 tokenizer 编码后的模型输入长度，两者不是固定换算关系。当前 Embedding 服务会先 tokenizer.encode，再用 `min(len(encoding.ids), max_tokens)` 截断，默认 `max_tokens=512`。所以一个 900 字符 Chunk 如果编码后超过 512 tokens，后半段不会参与这次 Embedding。

**源码证据**

`embedding.py::_embed_encoding_batch`。

**特别注意**

512 是 Embedding 输入上限，不是生成模型上下文窗口。

---

### 33. 章节正文按固定字符长度切块会在哪里切断语义？你会怎样比较段落、场景边界和语义切块？

**SolThink口述**

当前 `_chunk_text` 完全按字符位置滑窗，不理解段落、场景或事件边界，所以最容易切断的是对白前后因果、人物状态变化和场景切换。改进验证可以固定同一评测集，只换 Chunk 策略，比较“固定字符”“按段落合并”“按场景/事件切分”“语义 Chunk”四个版本，并重点观察跨 Chunk 事实召回率。

---

### 34. 结构化事件 JSON 与小说正文适合相同切块方式吗？事件需要原子化时如何设计索引单位？

**SolThink口述**

不完全适合。当前项目在 `RagDocument` 层已经把一个结构化事件作为一个 `event:{chapter}:{sequence}` 文档，因此事件天然比整章更原子；但真正写入向量索引时仍会统一经过 `_iter_document_chunks`。如果事件通常很短，这没问题；如果事件很长，我会进一步以事件为父文档、字段为可检索子单元，让“冲突/结果/人物”能独立评分。

---

### 35. 当前本地 Embedding 的默认模型、运行时、向量维度和设备配置是什么？哪些还需要在实际环境确认模型资产存在？

**SolThink口述**

配置默认是 `bge-small-zh-v1.5`、512 维；推理使用 `onnxruntime` 的 `CPUExecutionProvider`。资产是否真的可用，不能只看配置字符串，代码会检查 `model.onnx` 和 `tokenizer.json`/`vocab.txt` 等资产，并通过 `diagnostics()` 返回是否 ready。也就是说“默认配置是什么”和“当前机器是否真的可运行”是两件事。

---

### 36. 选择中文小说 Embedding 模型时，你会用什么任务样本和指标，而不是只看通用排行榜？

**SolThink口述**

我会直接从项目真实查询建 eval set：章节定位、人物关系、事件同义改写、跨章节因果、无答案问题。指标至少看 Recall@K、MRR、Precision@K，再加查询延迟和 Embedding 吞吐。模型排行榜只能当候选筛选，最终选择必须由项目任务分布决定。

---

### 37. 512 维向量代表什么？维度变大可能提升什么，又会增加哪些存储、计算和召回代价？

**SolThink口述**

512 维表示每个 Chunk 被编码成 512 个浮点数的向量。维度增加可能让表示空间有更丰富的表达能力，但不意味着一定提高任务 Recall，还会增加索引体积、距离计算和传输成本。真正的比较应该是“效果 / 成本”而不是“维度越高越好”。

---

### 38. Tokenizer、Padding、Attention Mask、Pooling 和 L2 normalization 在当前 Embedding 链路分别起什么作用？

**SolThink口述**

Tokenizer 把文本转成 ids；Padding 把一个 batch 对齐到同一长度；Attention Mask 区分真实 token 与 padding；如果 ONNX 输出是 3D hidden states，代码使用 mean pooling 得到句向量；最后 `_normalize` 做归一化，再把结果给 Chroma。这里每一步都直接服务于“把不同长度文本变成可比较的固定维向量”。

---

### 39. 如何判断 Embedding 输入被截断或批处理错误？你会检查输入 Token 数、模型输出形状还是检索案例？

**SolThink口述**

三个层面一起看。输入侧检查 `len(encoding.ids)` 与 `max_tokens`；推理侧检查 output shape 是否为 `(batch_size, embedding_dim)` 或合法的 sequence hidden state；业务侧用“关键信息位于 Chunk 尾部”的困难样本测试截断是否导致 Recall 下降。当前代码已经显式检查 shape，因此这部分比较容易做单元测试。

---

### 40. 更换 Embedding 模型或维度后，为什么旧索引不能直接与新查询向量混用？如何计划重建与回滚？

**SolThink口述**

因为旧索引中的向量是在旧编码空间里产生的，新查询向量如果来自新的模型，就不再处于同一个表示空间；维度不同甚至会直接导致索引层不兼容。当前 Chroma 默认 collection 名称还会包含 `model_name` 和 `embedding_dim`，能减少部分误混风险，但仍然没有完整的 RAG index version。生产方案应把 Embedding 配置、索引版本和 active pointer 绑定，先双建再切换。

---

## 五、查询意图与召回排序（41–50）

### 41. 查询意图对象包含哪些信息？章节范围、人物/场景选择器、字段查找和检索工具计划如何影响实际分支？

**SolThink口述**

`ScreenwritingQueryIntent` 包含原始查询、归一化查询、lookup terms、chapter range、chapter tail、field lookups、event selector、tool plan 和 diagnostics。章节范围会触发 metadata 路径，event selector 会触发 event JSON，tool plan 则进一步决定 vector / chapter text / lexical 等分支是否执行。

**核心概念**

它本质上是“检索计划”，不是最终答案。

---

### 42. 当前意图解析为什么先尝试项目文本模型、又保留本地规则？2 秒默认超时对查询体验和主链路有什么影响？

**SolThink口述**

项目打开意图模型是为了处理自然语言中更复杂的事件和字段选择，但又保留本地规则做确定性兜底。当前默认超时 2 秒；如果模型没配置、超时或解析报错，就回退 `parse_screenwriting_query_intent`。优点是功能鲁棒，缺点是复杂查询可能多出一次远程模型延迟，因此在低延迟 SLO 下可以考虑“简单查询直接规则，复杂查询再调用模型”。

---

### 43. 意图模型输出的 RetrievalToolPlan 与 Agent 的外部工具调用有什么不同？计划如何被 Python 检查和执行？

**SolThink口述**

`RetrievalToolPlan` 是内部数据结构，它告诉 RAG“这次该启用哪些检索通道”；它不是让 LLM 真的去执行某个工具。Python 会把允许的 tool name、参数和 priority 解析成结构化对象，再由 `rag_index.retrieve_context` 根据 `enabled_tools` 决定分支。真正的 Agent 工具调用发生在生成阶段，两者是两个层次。

---

### 44. 如果意图模型输出未知工具名、非法字段或格式错误，当前解析器如何处理？你会怎样测试这些边界？

**SolThink口述**

当前 `_tool_calls_from_payload` 只接受允许集合里的工具名，未知工具会被丢弃；字段 lookup 也会检查允许的 source type；JSON 不是合法对象则整体进入 fallback。测试时我会构造未知工具、字符串代替列表、非法 source type、缺字段、空 selector、非法章节范围和模型超时六类样本，确保任何一种异常都不会让主链路崩掉。

---

### 45. 什么查询适合元数据精确命中，什么查询适合结构化事件查找，什么时候才需要向量搜索？

**SolThink口述**

“第 10–20 章”“作者是谁”属于确定性查询，优先 metadata；“张三和谁发生冲突”“冲突结果是什么”适合 event selector；“这个人物为什么离开”“类似某种心理变化的章节有哪些”这类开放语义才更适合 vector。核心判断标准是查询有没有可结构化、可精确验证的条件。

---

### 46. 事件命中后关联原章节正文解决什么问题？事件元数据不完整时可能漏掉什么证据？

**SolThink口述**

event 文档是结构化摘要，关联 chapter 可以把事件附近的更完整章节信息带回来，补充结构化字段没有覆盖的上下文。当前 `_chapter_text_lookup_hits` 会根据 `chapter_public_id`、`chapter_id` 或 `chapter_index` 找回章节文档。问题是事件抽取如果本身就漏掉人物、冲突或结果，那么 event selector 根本可能召不出来，这时还是需要 chapter vector 或 lexical 作为补充。

---

### 47. 向量召回先取目标数量的三倍候选、再做阈值过滤和来源去重，分别想解决什么问题？每一步可能引入什么偏差？

**SolThink口述**

默认最终 limit 50，向量层先取最多 150 个候选，是为了留出过滤和去重空间。之后过滤低于 0.35 的向量分，再按 `source_id` 去重，避免同一资料占满结果。偏差也很明显：同一来源多个高质量 Chunk 会互相“挤掉”，导致证据多样性下降；阈值过高又可能把长尾正确结果直接过滤掉。

---

### 48. 当前向量与词法命中如何用 Reciprocal Rank Fusion 合并？为什么不能直接比较两路原始分数？

**SolThink口述**

向量分数来自 Chroma 的 distance 转换，大致在 0–1 的尺度；词法分可以因为精确命中加很多 bonus，量纲完全不同。如果直接比较原始分数，词法通道可能长期压过向量。因此当前代码用 `1 / (60 + rank)` 计算 RRF，再按 `source_id` 累加，最后用 RRF 总分排序；原始分数只保留用于展示。

---

### 49. 当前词法评分实现为什么不能直接称为 BM25？如果要改成 BM25，需要哪些索引、分词和评测工作？

**SolThink口述**

当前 `lexical_score` 是 token overlap + phrase bonus + 单字符出现次数 bonus，并没有 BM25 的 TF、IDF、文档长度归一化等完整公式，所以不能称 BM25。要改成 BM25，需要真正的倒排索引或搜索库、分词器、字段权重、参数 k1/b，并用同一 Golden Set 做 BM25-only 与当前 lexical-only 的对比。

---

### 50. RRF 只使用各路排名时有哪些限制？一个差距很大的高相似度命中与两个通道的一般命中会怎样竞争？

**SolThink口述**

RRF 不看绝对分数，只看 rank，所以它对“第 1 名到底比第 2 名高多少”不敏感。一个单路极高质量但只出现在一路的结果，可能输给两个通道都处于前列的一般结果。后续可以研究 rank fusion + score calibration，或者加入 cross-encoder 对 top-N 做二次精排。

---

## 六、RAG 与 Agent 生成链路（51–60）

### 51. 查询文档加载和意图解析为什么可以并发执行？它们依赖共享数据库会话或模型资源时要检查什么？

**SolThink口述**

`prepare_context` 使用 `asyncio.gather` 同时做文档加载和意图解析，因为两个任务在逻辑上互不依赖，先做可以节省等待时间。但并发不等于没有边界：文档加载使用当前 `AsyncSession`，意图模型走独立 gateway，所以要避免把同一个不支持并发的数据库操作对象或不可重入模型客户端同时复用到冲突路径。

---

### 52. 普通聊天与阶段生成各如何接收 RAG 结果？它们是否共用同一类提示词和工具调用过程？

**SolThink口述**

普通聊天会先构造带 RAG 的系统提示，并给 Agent 配工具；阶段生成则把已经准备好的 RAG 文本直接放进阶段系统提示，同时把 `chapter_events` 直接传入，阶段 Agent 使用 `tools=[]`。代码注释明确说明这样做是为了避免阶段流式通道因为模型发起工具调用而出现零文本产出。因此，两条链路共享 RAG 准备，但生成阶段的上下文注入方式不同。

---

### 53. get_rag_context 是在工具被调用时重新检索，还是读取本轮已经准备好的结果？如何从实现验证？

**SolThink口述**

它读取的是本轮已经准备好的 `rag_context`，不是重新查 Chroma。`build_rag_tools` 直接闭包捕获 `rag_context`，返回 `text`、`runtime` 和 hits 明细。这个设计让一次请求内的检索结果保持一致，也避免 Agent 多次调用工具重复消耗检索开销。

---

### 54. 为什么阶段生成除了 RAG 命中，还会将配置范围内的结构化章节事件直接注入上下文？

**SolThink口述**

阶段生成的目标不是只回答一个查询，而是基于原著做骨架、策略或剧本生成，所以除了本轮 query 召回结果，还需要阶段计划对应的结构化事件作为任务范围信息。代码中阶段 Agent 的 system prompt 同时接收 `rag_text` 和 `chapter_events`。这是一种“检索证据 + 任务范围结构”的组合，而不是重复跑一次 RAG。

---

### 55. 如何让 Agent 在回答剧情问题时引用来源章节或事件，而不是只输出看似合理的结论？

**SolThink口述**

当前项目并没有完整的“强制引用协议”。RAG runtime payload 会保留 `sourceId`、`sourceType`、title 和 score，但 `_format_context_text` 给模型的文本主要是 `[序号] 来源标签 + 标题 + score + chunk_text`，并不会自动产生用户可见 citation。我的优化方案是把稳定 source ID 明确注入上下文，要求模型输出事实+证据来源，再增加一个 citation verifier 检查答案中的事实是否能回指到命中文档。

**边界**

这是优化方案，不应说成当前已经具备。

---

### 56. 结果按来源去重时，如果同一章节的多个 Chunk 分别包含起因和结果，会不会丢掉重要证据？你如何设计对照查询？

**SolThink口述**

会有这种风险。向量召回先按 Chunk 查，再按 `source_id` 去重，只保留一个来源命中；RRF 最终也按 `source_id` 合并。因此一个章节里“冲突起因”和“冲突结果”位于两个 Chunk 时，可能只留下其中一个。验证方法是构造一个必须跨两个 Chunk 才能回答完整的问题，分别比较 current source-dedupe 与 chunk-level top-K 的 Recall 和最终答案完整度。

---

### 57. 召回上限为 50 条时，如何估计真实 Token 预算？为什么固定条数不能保证上下文长度可控？

**SolThink口述**

因为 50 条只是 hit 数，不是 token 数。当前向量 Chunk 默认 900 字符，而 lexical fallback 的上下文还可能被截到 1200 字符，再加标题、score、系统提示和对话历史，最终上下文会随着内容长度变化。当前项目没有 tokenizer 级 context budget，所以固定 50 不能保证安全长度；优化时应该按模型 tokenizer 计算总 token，并设置 hard budget。

---

### 58. 系统提示词、对话历史、工作区、RAG 资料和本轮输入同时进入模型时，如何避免超长和信息优先级冲突？

**SolThink口述**

当前代码会保留最近一部分历史消息，并将 workspace 只做摘要，完整工作区通过工具按需读取；RAG 则作为单独上下文块注入。真正的缺口是没有统一 token budget manager。我会按“系统约束 > 本轮任务 > 原著证据 > 结构化事件 > 最近历史 > 非必要 workspace”设优先级，然后用 tokenizer 做动态裁剪，而不是按固定字符数硬截。

---

### 59. 检索为空或准备失败时，当前聊天链路怎样继续？你会怎样区分“无相关资料”和“RAG 故障”并告知用户？

**SolThink口述**

当前 vector 不可用或查询异常会转 lexical；完整 `prepare_context` 异常会捕获并生成 `empty_rag_context(failure_reason)`，聊天主链路继续。因此“无资料”与“RAG 故障”可以从 `retrievalMode` 和 `failureReason` 区分：空命中但无失败原因更接近“没找到”，有 `failureReason` 才更像“系统问题”。

---

### 60. 如果检索结果正确但最终回答编造情节，你会检查上下文格式、指令、历史污染、截断还是生成模型行为？如何逐项隔离？

**SolThink口述**

我会做隔离实验：第一步直接把最终 RAG context 原样喂给一个固定模型，排除 retrieval；第二步只保留证据、去掉对话历史，判断是否是历史污染；第三步控制 hit 数和 chunk 长度，判断是否是上下文截断；第四步固定 prompt、只换模型，判断生成行为。这样能把“召回正确但答错”从 RAG 问题里分离出来。

---

## 七、评测集与质量证明（61–70）

### 61. 如何建立覆盖章节定位、人物事件、语义改写、跨章节推理和无答案查询的 RAG 评测集？

**SolThink口述**

我会按业务查询分层建 Golden Set：章节精确定位、字段查询、人物/场景事件、语义改写、跨章节关系、长上下文、无答案/拒答。每条样本至少记录 query、gold source、可接受多个 source 的集合、关键事实和答案判定标准，这样后面才能分别评估 retrieval 与 answer。

---

### 62. Ground Truth 应标到 RagDocument、Chunk、章节还是具体事实？不同标注粒度会怎样影响指标？

**SolThink口述**

当前系统最终有“source-level 去重”，因此 retrieval eval 至少要有 source 级 gold；同时为了分析 Chunk 参数，我还会保留 Chunk 级 gold。对最终回答，还要增加“事实级” gold。三种粒度回答三个问题：找到哪个来源、找到哪段证据、答案是否覆盖事实。

---

### 63. Recall@K、Precision@K、MRR 和 NDCG 分别回答什么问题？请说明一个适合项目的计算口径。

**SolThink口述**

Recall@K 看所有相关来源里有多少在前 K；Precision@K 看前 K 里多少是真相关；MRR 关注第一个相关结果排得多高；NDCG 更适合存在多级相关性的场景。对本项目我会把 source-level Recall@K 作为主指标、MRR 看排序、Precision@K 看噪声，再用 NDCG 表示“强相关章节 > 弱相关章节”的位置质量。

---

### 64. 如果一个问题有多个正确章节，如何计算召回质量，避免“命中一个就算全对”？

**SolThink口述**

把 gold 设计成集合，而不是一个答案。比如某问题允许 3 个章节共同作为证据，那么 Recall@5 = 前 5 命中的 gold source 数 / 3。对于最终答案，也可以定义“事实覆盖率”，要求关键事实至少由一个可接受来源支持。

---

### 65. 如何把“检索正确”与“最终回答有证据支持”分开评估？需要什么人工标注或自动判分流程？

**SolThink口述**

检索评测只看 hit 与 gold 的关系；答案评测再单独做 factuality/citation 检查。可以让人工给事实三态标签：supported / contradicted / unsupported，再把每个事实映射到 source ID。自动评测可以先做 claim extraction，再做 evidence entailment，但关键样本仍需要人工抽检。

---

### 66. 如何评估回答的原文忠实度、人物设定一致性、因果完整性和来源引用准确性？

**SolThink口述**

我会拆成四个独立维度：原文事实是否可支持、人物属性有没有冲突、因果链有没有漏关键节点、引用是否真的指向对应证据。不要只用一个“答案看起来像对的”分数。每维可以 0–2 或 0–3 分，最后报告总分和错误类型分布。

---

### 67. 如何构造对当前系统有区分度的困难样本，包括别名、否定、人物同名、事件相近和长上下文？

**SolThink口述**

直接从当前路由的薄弱点造样本：别名让 lexical 和 event selector 更难；否定词测试“同词但事实相反”；同名角色测试 metadata 与 event；近似事件测试 RRF；长上下文测试 Chunk 和 token budget。困难样本不应该是“故意刁难”，而要代表真实业务中最容易错的查询。

---

### 68. 0.35 向量最低分和召回上限如何在开发集上选择？如何保留独立数据避免对评测集过拟合？

**SolThink口述**

0.35 是当前默认配置，不代表已经被数据验证。正确做法是只在 development set 上做 threshold sweep 和 K sweep，然后固定参数，在 holdout set 上只测一次。参数选择应该同时看 Recall、Precision、MRR、端到端延迟和上下文 token 数。

---

### 69. 如何设计向量、词法、元数据、事件检索的消融实验？一次改变多个分支会有什么分析困难？

**SolThink口述**

先固定 Embedding、Chunk、threshold 和数据集，只做单变量：vector-only、lexical-only、vector+lexical/RRF、metadata-only、event-only、full routing。这样才能知道每个通道带来的是 Recall 还是噪声。一次改 Embedding + Chunk + RRF + threshold，就无法解释变化来自哪里。

---

### 70. 汇报准确率或省时结果时，为什么必须同时写基线、分子分母、样本量、测试条件和绝对/相对变化？

**SolThink口述**

因为“准确率 90%”没有基线就没有意义，“提升 10%”也不知道是 80→90 还是 10→20。面试和汇报都应该说明 metric 定义、分子分母、样本数、K、数据集划分、模型/Chunk/threshold、绝对变化还是相对变化。

---

## 八、索引新鲜度与故障恢复（71–80）

### 71. 本轮资料指纹已变化但新索引尚未构建完成时，当前查询可能读到什么？怎样在延迟与新鲜度间取舍？

**SolThink口述**

当前查询不会等 warmup。metadata 和 lexical 使用本轮刚加载的 `rag_documents`，所以这两条路径可以反映当前数据库内容；vector 依赖已持久化的 Chroma 索引，因此在刷新完成前可能还是旧版本。当前选择明显偏向低延迟和可用性；生产优化则应该引入 active version，让“旧版本可读”和“新版本构建”显式分离。

---

### 72. warmup 使用 API 进程内 asyncio.create_task 时，服务重启、部署切换或进程异常会怎样影响正在构建的索引？

**SolThink口述**

后台 task 属于当前进程生命周期，进程重启后这个 task 不会自动继续。当前又没有持久化 build state 和 active version，所以构建到一半时可能留下部分索引。重启后新的请求会重新计算 fingerprint，发现没有匹配记录，再次调度 warmup，但这属于“重新检测和重试”，不是“跨进程续传”。

---

### 73. 当前 warmup 去重数据结构是进程内的；多实例同时收到同一项目预热请求会有什么问题？

**SolThink口述**

`_warmup_tasks` 是实例成员，每个 API 进程各自一份，所以 Worker A 的“running”对 Worker B 不可见。两边都可能对同一个项目做 warmup，导致重复 Embedding、重复 Chroma 写入甚至互相覆盖。解决方案应该是 project 级 Redis 分布式锁，或者把任务状态放到持久队列中，由单独 Worker 负责。

---

### 74. 为什么同步 Embedding、文件或 Chroma 工作要从异步请求协程中卸载？asyncio.to_thread 能解决什么，不能解决什么？

**SolThink口述**

当前代码已经把 `_get_index`、fingerprint 和 `retrieve_context` 等可能阻塞的工作放到 `asyncio.to_thread`。作用是避免同步 I/O 或 CPU/库调用长期阻塞事件循环。它不能解决多进程协调、任务持久化、CPU 规模扩展或者索引原子切换；这些是架构层问题。

---

### 75. Chroma 正在增量写入时同时查询，会遇到哪些一致性或资源问题？你会用什么并发实验验证？

**SolThink口述**

当前代码没有显式的读写锁或版本快照，所以不应该直接声称“Chroma 一定会出现某种一致性错误”。我会做并发实验：持续 warmup 同一个项目，同时发高频查询，记录 query latency、异常率、命中数变化和结果版本；再对比无并发基线。如果发现读写互相抖动，再决定是限流、隔离进程还是改成版本化索引。

---

### 76. Embedding 资产不可用、向量查询异常或意图模型超时时，分别会退化到什么路径？如何让日志区分这些原因？

**SolThink口述**

Embedding 资产不可用会使 `vector_ready=False`，从而走 lexical；Chroma query 抛异常时，`vector.py` 记录 failure reason，`rag_index` 继续 lexical fallback；意图模型超时/报错时，`query_intent_llm.py` 返回 fallback intent 并记录 parser failure。日志至少要把 `intentParserFailure`、`vectorFailureReason` 和最终 `retrievalMode` 分开。

---

### 77. 如果元数据命中和结构化事件命中有数据，但向量库不可用，哪些请求仍然可以成功？怎样验证？

**SolThink口述**

章节范围查询、字段查询和事件 selector 本身不依赖 vector，所以可以成功；开放式语义查询则会退化到 lexical，质量可能下降。验证时可以人为禁用 Embedding/Chroma，分别发送“第 10–15 章”“作者是谁”“张三与谁冲突”“开放式剧情为什么……”四类请求，看前 3 类是否仍正常。

---

### 78. 如何设计索引版本和原子切换，让构建期间查询旧版本，验证成功后切换，并能回滚？

**SolThink口述**

我会把当前单一索引改成版本化：V1 ACTIVE；后台构建 V2；V2 完成后做 chunk count、样本查询和 Golden Set 验证；通过后把 active_version 从 V1 原子更新为 V2。失败时保持 V1，不让半成品进入线上。这个能力当前项目没有，属于明确的生产化优化。

**推荐流程**

```text
V1 ACTIVE
  |
  v
V2 BUILDING
  |
  v
V2 VALIDATING
  |        \
 FAIL      PASS
  |          \
 V1 ACTIVE   V2 READY
                  |
                  v
          active_version: V1 -> V2
```

---

### 79. 项目或章节删除后，如何确认数据库、Chroma、进程缓存和后台刷新任务都完成清理？

**SolThink口述**

当前代码已经有 incremental refresh 下的 stale Chunk 删除，也有 `clear_cache` 清理进程内 RAG runtime，但没有在一个事务里保证“业务数据删除 + 向量删除 + 后台任务取消 + 旧版本回收”全部完成。因此生产方案应把删除做成幂等任务，按 project_id 加锁，完成 vector count 校验，再清理 runtime/cache，并留下审计状态。

---

### 80. 当前 runtime diagnostics 和日志能告诉你哪些检索分支、命中数、分数与索引状态？还缺哪些告警和运行指标？

**SolThink口述**

当前 diagnostics 能看到 retrieval mode、strategy、hitCount、vectorHitCount、lexicalHitCount、minVectorScore、intent source/confidence/tool plan，以及 index cache hit、refresh scheduled、failure reason。日志还会输出项目、会话、模型、RAG 命中数、文档数和阶段耗时。缺口主要是单次 retrieval 子阶段耗时、token 数、queue wait、retry、索引版本、任务年龄、数据新鲜度 lag 和质量指标。

---

## 九、性能、隐私与多租户（81–90）

### 81. 一次 RAG 请求如何拆分测量文档读取、意图模型、向量召回、词法排序、上下文拼装和生成延迟？

**SolThink口述**

当前已经有 `server_timings`，能记录 project、rag、agentSetup、agentRun 等阶段，但 RAG 内部还没有把“文档加载 / fingerprint / intent / vector / lexical / format”全部拆开。下一步我会在 `prepare_context` 和 `retrieve_context` 内增加细粒度 span，同时把每阶段输入规模、hit 数和 token 数一起记录。

---

### 82. 如果每次请求都加载全部项目文档并计算指纹，数据量增加后时间和内存如何增长？如何用版本号或变更事件降低成本？

**SolThink口述**

当前 `prepare_context` 每次都会 `load_screenwriting_knowledge_documents` 并计算全量 `RagDocument` fingerprint，所以成本至少会随着项目资料量增长而线性增加。最直接的优化不是先改向量库，而是让业务变更事件产生 document/version delta，runtime 只处理发生变化的章节，再用持久化版本号快速判断缓存是否新鲜。

---

### 83. 如何评估 P50、P95、P99 延迟及排队时间？平均耗时下降但尾延迟上升时该如何解释？

**SolThink口述**

平均值只反映中心趋势，P95/P99 才能看到长尾用户。当前项目已有阶段 timings，但没有完整队列等待维度，所以我会先统一定义 request start、queue wait、RAG、Agent、external model 和 total 的时间线。若平均下降、P99 上升，通常意味着少部分慢请求变得更慢，可能是模型超时、资源竞争、并发度或超大项目造成的尾部放大，需要看分桶而不是只看平均值。

---

### 84. 项目级 Chroma 目录与隔离键如何阻止跨项目召回？文件路径和 collection 名称还需要哪些校验？

**SolThink口述**

项目 RAG 使用项目级目录，同时 Chroma query 通过 `where={"isolation_key": isolation_key}` 过滤。目录名还经过安全化处理，避免直接使用任意 project ID 作为路径。进一步优化可以在初始化时校验 model/dimension/index version，并对 collection 和 isolation key 做一致性校验，防止人工配置错误。

---

### 85. 为什么向量库隔离不能替代后端项目权限校验？怎样测试用户 A 无法检索用户 B 的项目内容？

**SolThink口述**

因为向量隔离只是在“已经进到某个索引范围后”防止跨域检索，它不能证明请求者本身有权进入那个项目。真正的测试要创建用户 A/B 和项目 A/B，让 A 请求项目 B，确认 API 在进入 RAG 前就被 `get_project_or_raise` 拦截；再额外测试直接调用 RAG 层时 isolation key 也不会跨项目。

---

### 86. 小说正文、导演资料和用户问题都可能包含提示注入内容，如何把它们标为不可信数据并降低指令污染？

**SolThink口述**

当前项目没有独立的 prompt-injection 防护层，因此我会把 RAG 文档明确标成“不可信证据数据”，而不是可执行指令。系统提示应声明“RAG 内容只能作为事实/资料引用，不得覆盖系统指令”；对用户问题、小说正文、导演资料分层编码，并在高风险场景增加 output policy check。这个属于当前缺口，不要说成已经实现。

---

### 87. 查询日志要保留哪些字段用于定位召回问题？如何避免记录小说全文、密钥和不必要的个人信息？

**SolThink口述**

至少保留 request id、project id、query hash、retrieval mode、intent source、hit source IDs、top scores、timings、error type 和 model id。原始小说正文不必整段写日志，必要时存 hash 或采样片段；密钥、token、用户私密内容都不应该进入普通 debug log。日志设计要以“能复现问题”为目的，不是“越全越好”。

---

### 88. 如何限制单个超大项目占满 Embedding CPU、磁盘或请求上下文？多租户公平性怎样度量？

**SolThink口述**

可以从三个层次限制：单项目 Chunk 数 / index size quota；Embedding worker 的 per-project concurrency；查询侧 context token budget。公平性可以看每个项目的 CPU time、Embedding jobs、磁盘占用、P95 latency 和队列等待时间，避免一个超大项目把所有 Worker 占满。

---

### 89. Embedding 或索引流程改版时，如何灰度构建、比较新旧索引并在质量下降时回退？

**SolThink口述**

不要直接覆盖线上索引。先用同一项目并行构建新索引，把新旧索引在 Golden Set 上做离线对比，再让少量流量 shadow 或 canary 到新版本；如果 Recall、Precision、P95 或错误率恶化，就把 active pointer 切回旧版本。当前项目没有 active pointer，这正是应该补的能力。

---

### 90. 如果要提供团队级 RAG 服务，哪些数据保留、删除、访问审计和供应商隐私约束需要先明确？

**SolThink口述**

至少要明确数据归属、保存期限、删除语义、导出权、访问审计、跨租户隔离、第三方模型是否会看到原文、日志是否包含敏感内容，以及项目删除后向量和缓存多久清理。技术方案应该把这些策略落实到 API 权限、索引隔离、日志脱敏和供应商调用边界。

---

## 十、系统设计、验证与经验迁移（91–100）

### 91. 如果面试官给你 10 分钟设计“百万 Chunk、多个项目、多种来源”的小说知识检索，请先问清哪些使用场景、SLO、更新频率和权限条件？

**SolThink口述**

我会先问五类问题：第一，查询类型和答案是否必须引用原文；第二，项目规模和百万 Chunk 是总量还是单租户峰值；第三，P50/P95、吞吐和索引更新时效；第四，数据权限与删除要求；第五，Embedding/LLM 成本预算。没有这些约束，直接选“Milvus/ES/向量库”其实是在做技术堆砌。

---

### 92. 在每项目资料量扩大十倍后，你认为当前链路最可能先卡在全量资料读取、Embedding、Chroma 查询、词法扫描还是上下文生成？如何用测量确认？

**SolThink口述**

我会先重点怀疑“每请求加载全部 RAG 文档 + 计算 fingerprint”这一段，因为它随项目资料量线性增长，而且发生在每轮查询的同步准备路径上。然后实际测量 DB load、fingerprint、vector query、lexical scan、context format 各自耗时，不能靠猜。这里的第一优化目标通常是把“数据变更检测”从请求路径移到事件驱动索引任务。

---

### 93. 如果 P95 查询延迟必须控制在 500 毫秒，而当前意图解析需要远程模型，你会如何调整路由、缓存和超时策略？

**SolThink口述**

当前默认意图模型超时就是 2 秒，因此它与 500ms P95 目标天然冲突。我会先把确定性查询全部走本地规则，只有复杂事件查询才调用模型；对意图结果做 query normalization cache；同时把模型调用设置更短 deadline，并在超时后立即用 fallback intent。最终目标不是“让模型更快”，而是让慢模型不进入所有请求。

---

### 94. 如果高召回率导致无关上下文增多、生成质量下降，你会如何一起分析召回阈值、RRF 排序、上下文预算与答案证据？

**SolThink口述**

先固定生成模型，把 retrieval 单独评估：看 threshold、K 和 RRF 对 Precision/Recall 的影响；再固定 retrieval，只降低 context budget，观察答案变化。如果 retrieval precision 已经下降，就不能只靠 Prompt 去救；如果 retrieval 很干净但 context 太长，则应该做 dynamic top-K / token budget / query-aware truncation。最后再看 answer evidence coverage，判断是不是上下文噪声导致生成忽略重点。

---

### 95. 如果精确章节问题答错但普通语义问题正常，你会检查意图解析、章节范围、metadata、来源 ID 和早返回路径中的哪些环节？

**SolThink口述**

按链路查：先看 `ScreenwritingQueryIntent.chapter_range/chapter_tail` 是否正确；再看 `_chapter_index_metadata_hits` 是否按 `chapter_index` 找到目标；再确认 `source_id=chapter:{chapter_public_id}` 是否与文档一致；最后检查 metadata 命中后是不是提前 return，把本来需要补充的语义路径截掉。因为普通语义正常、精确章节异常，优先怀疑 deterministic routing，而不是 Embedding 模型。

---

### 96. 如果召回内容正确但回答遗漏关键事件，你会怎样构造能区分排序、去重、截断和生成忽略的最小实验？

**SolThink口述**

构造一个由两个不同 Chunk 共同组成完整事实的 query。实验 1：只给两个 gold Chunk，判断模型是否会答对；实验 2：加入其他噪声 Chunk，看排序影响；实验 3：使用当前 source-level dedupe，观察是否丢掉其中一个；实验 4：固定 hits，只增加上下文长度，判断是否出现截断/long-context 问题。四个实验可以把问题层次拆开。

---

### 97. 如果要求索引构建可跨进程恢复，你会把它改成怎样的持久任务状态、分布式协调和版本切换流程？

**SolThink口述**

我会把职责拆开：数据库任务表记录 task id、project、target version、fingerprint、status、retry、worker、started/finished/error；Redis 锁只负责“谁能执行这个 project 的构建”；MQ 负责把更新任务可靠送给 worker；索引本身采用 V1/V2 版本化，构建和验证都在 inactive version 上完成，最后原子更新 active pointer。这样任务状态、执行互斥和索引一致性分别由三个机制承担。

**注意**

这些都是生产化设计，不是当前仓库现成能力。

---

### 98. 如何构造 fake Embedding、fake 向量库和 fake 模型，使 RAG 路由、异常回退和排序测试不依赖付费 API？

**SolThink口述**

Embedding fake 可以把输入映射成确定性的固定维向量；vector fake 只实现 query/upsert/delete 并返回预设 `RetrievedChunk`；intent fake 返回固定的 tool plan；model fake 只记录传入 prompt，不真正生成。这样就可以分别测试“metadata→event→vector→lexical”“vector exception→lexical”“LLM intent timeout→fallback”“RRF 排序”这些逻辑，不依赖外部供应商。

---

### 99. 当前项目文档与代码对 RRF 的描述不一致时，你会如何确认实际行为、修正文档并补上回归证据？

**SolThink口述**

我会以代码执行路径为最终事实来源，直接定位 `_merge_ranked_hits`，确认常数、输入通道、去重键和 tie-break；然后用两个极简的预设 ranking 做单元测试，证明“原始分数量纲不同”和“RRF 只看 rank”的行为。文档修正后，把这个 case 固化成 regression test，防止后续重构再把实现和文档拉开。

---

### 100. 请为该项目制定一个两周 RAG 改进计划：先选问题、建立基线，再做单变量实验、复测保留集并说明上线或回退条件。

**SolThink口述**

第一周先做证据基线：建立 Golden Set，定义 source-level Recall@K、MRR、Precision@K、answer support、P95、token usage；再做 vector / lexical / RRF / threshold / K 的消融。第二周优先改两个高价值问题：一是 token-level context budget，二是索引一致性。索引侧先做最小的 versioned build + validation + active switch；检索侧做 reranker 或 dynamic K 的单变量实验。上线条件不是“看起来更好”，而是 holdout set 不降、P95 不超预算、关键失败类不恶化；否则保留旧版本回退。

---

# 11. 面试官评估维度（SolThink强化版）

| 维度 | 强回答 | 弱回答 |
|---|---|---|
| 个人贡献 | 能落到函数、改动、样例、测试证据 | “这个项目用了很多框架” |
| 技术判断 | 能解释为什么分层、为什么回退、为什么不用更贵方案 | “行业都这么做” |
| 源码能力 | 能按调用链说清运行顺序 | 只背术语 |
| RAG能力 | 能区分 metadata / event / vector / lexical / RRF | 把所有问题都归为向量相似度 |
| 验证意识 | 有 Golden Set、基线、分母、条件、holdout | “效果很好” |
| 故障分析 | 能分检索、排序、上下文、生成、索引陈旧 | 一律改 Prompt |
| 工程化 | 能讲幂等、恢复、锁、版本切换、监控 | 只讲算法，不讲运行时 |
| 诚实边界 | 清楚说“当前没有”并给出验证/优化方案 | 把设计方案冒充现有能力 |

---

# 12. 当前项目“已经有 / 还没有 / 为什么需要优化”总表

## P0：优先解决工程一致性

| 能力 | 当前项目 | 为什么要优化 | 建议验证 |
|---|---|---|---|
| 项目级 RAG 隔离 | 已有 | 需要继续验证跨租户与配置错误 | A/B 项目互查 |
| fingerprint + incremental refresh | 已有 | 能做增量，但不能原子切换 | 修改/删除章节后比对 Chunk |
| 进程内 warmup 去重 | 已有 | 多实例失效 | 双 Worker 并发 warmup |
| Redis 分布式锁 | 未用于 RAG warmup | 防多实例重复构建 | 多实例同项目压测 |
| 持久任务表 | 未用于 RAG warmup | 解决重启丢任务和状态不可追踪 | kill -9 后恢复实验 |
| V1/V2 索引版本 | 没有 | 防半成品索引污染在线查询 | 故意在构建中失败 |
| active pointer 原子切换 | 没有 | 支持回滚 | 切换前后请求一致性 |
| 删除全链路审计 | 不完整 | 避免 DB/Chroma/cache 残留 | 删除项目后全链路校验 |

## P1：优先提升检索质量

| 能力 | 当前项目 | 优化方向 |
|---|---|---|
| Vector | 已有 | 做 threshold / K calibration |
| Lexical fallback | 已有 | 升级到 BM25/字段加权 |
| RRF | 已有 | 做权重/score calibration 对比 |
| Cross-Encoder reranker | 当前无源码证据 | top-N 二次精排 |
| source-level dedupe | 已有 | 防止多 Chunk 证据丢失 |
| dynamic top-K | 没有 | 按 query complexity / budget 调节 |
| token budget | 没有 | tokenizer 级硬预算 |
| citation/evidence gate | 没有完整实现 | claim→evidence 校验 |
| conflict resolver | 没有完整实现 | source authority + contradiction check |

## P2：运行与多租户

| 能力 | 当前项目 | 优化方向 |
|---|---|---|
| 阶段 timings | 已有 | 增加 RAG 子阶段 span |
| failureReason | 已有 | 分类成可告警 error taxonomy |
| retry/backoff | 通用任务系统有基础能力，但 RAG warmup 当前不直接使用 | 接入 RAG task worker |
| worker heartbeat | 通用任务系统有配置 | 让 RAG 索引任务可观测 |
| queue wait | 尚缺 RAG 维度 | 增加 worker queue metrics |
| index freshness lag | 尚缺 | 记录 source version vs active version |

---

# 13. SolThink必背的 12 个“高压追问”

### 1）“你这套 RAG 到底是不是 BM25？”

> 不是。当前 lexical 是 token overlap + phrase bonus + char bonus，RRF 才是 rank fusion。

### 2）“你们有没有 Cross-Encoder reranker？”

> 当前代码没有独立 Cross-Encoder reranker 的证据。现在是 vector + lexical 后用 RRF 合并。

### 3）“warmup 是不是阻塞用户？”

> 不是。查询发现 fingerprint 不一致时只调度 background warmup，本轮直接使用已有持久化索引；RAG preparation 异常也不会阻断聊天主链路。

### 4）“你们是不是 Redis Worker 在重建 RAG？”

> 当前 RAG warmup 不是 Redis Worker，而是 API 进程内 `asyncio.create_task`；项目虽然有通用 Redis Stream task infrastructure，但当前 warmup 没接过去。

### 5）“900 字符为什么不等于 900 token？”

> 因为 Chunk 按字符切，Embedding 按 tokenizer 后的 token 数截断；当前 max_tokens=512。

### 6）“50 条结果就等于 50 个 Chunk 吗？”

> 不完全是。向量候选最多 150，但最后按 source_id 去重，最终 hit 上限是 50 个来源级结果。

### 7）“为什么不能直接比 vector score 和 lexical score？”

> 两路量纲不同，所以用 RRF，只比较各自 rank。

### 8）“RRF 是 reranker 吗？”

> 更准确地说是 rank fusion，不是 cross-encoder 或 LLM 二次精排。

### 9）“如果索引构建失败，会不会把旧索引弄坏？”

> 当前没有版本化原子切换，增量/重建过程中确实存在留下部分状态的可能；这是当前需要补的 P0。

### 10）“你们有 citation 吗？”

> runtime payload 有 sourceId/sourceType 等元数据，但当前模型上下文格式并没有完整的用户可见 citation 协议；这个可以通过 evidence gate/citation verifier 补。

### 11）“0.35 是怎么算出来的？”

> 它是当前默认配置，不代表已经被项目实验验证。正确做法是在 development set 上 sweep，再用 holdout set 验证。

### 12）“如果让你把项目做到生产，你先改哪三个？”

> 第一是索引版本化 + 原子切换；第二是 Token 级 context budget；第三是持久任务 + Redis 分布式协调，把当前进程内 warmup 变成可恢复的索引任务。

---

# 14. 一分钟项目回答模板

> 我这个项目是面向小说转短剧场景的项目级 RAG。核心问题不是“让模型会搜索”，而是不同问题需要不同证据：项目配置适合 metadata，人物和冲突适合结构化 event，开放式剧情适合 chapter vector，再用 lexical 做补偿。
>
> 在实现上，我先把项目配置、小说、章节、事件以及视觉风格/导演手册统一成 `RagDocument`。查询进来后，`chat.py` 先判断是否真的需要章节检索；`rag_runtime.py` 并发做文档加载和意图解析，并通过 fingerprint 检查索引是否新鲜。`rag_index.py` 再按 metadata → event → vector → lexical 的顺序编排，并对 vector 和 lexical 用 RRF 合并。
>
> 工程上，我特别关注失败和边界：Embedding 或 Chroma 不可用时会退化到 lexical；意图模型超时回退本地规则；RAG preparation 失败也不会阻断聊天。当前最大的生产化缺口是索引没有 V1/V2 原子切换、没有 Token 级 context budget、也没有独立 BM25/Cross-Encoder reranker。下一步我会先用 Golden Set 建立基线，再做单变量实验，而不是直接堆新组件。

---

# 15. 最终复习顺序

### 第一轮：必须背熟

`3 → 5 → 12 → 18 → 25 → 29 → 31 → 42 → 48 → 53 → 57 → 59 → 71 → 73 → 82 → 95 → 99 → 100`

### 第二轮：补工程化

`26 → 27 → 30 → 40 → 56 → 72 → 75 → 78 → 79 → 84 → 89 → 97`

### 第三轮：补评测

`61 → 62 → 63 → 64 → 65 → 68 → 69 → 70 → 94 → 96`

### 每道题都强制走这条回答阶梯

> **我做了什么 → 为什么这么做 → 还有什么方案 → 怎么验证 → 哪里会失败 → 规模变大后怎么改。**

---

# 16. 源码导航（建议面试前逐个打开）

- `backend/app/core/rag/documents.py`
- `backend/app/core/rag/constants.py`
- `backend/app/core/config.py`
- `backend/app/core/agent/embedding.py`
- `backend/app/core/harness/memory/vector.py`
- `backend/app/services/screenwriting/rag_documents.py`
- `backend/app/services/screenwriting/query_intent.py`
- `backend/app/services/screenwriting/query_intent_llm.py`
- `backend/app/services/screenwriting/rag_index.py`
- `backend/app/services/screenwriting/rag_runtime.py`
- `backend/app/services/screenwriting/chat.py`

## GitHub

- https://github.com/lobokiol/NovelToVideo
- https://github.com/lobokiol/NovelToVideo/blob/main/backend/app/services/screenwriting/rag_documents.py
- https://github.com/lobokiol/NovelToVideo/blob/main/backend/app/services/screenwriting/query_intent.py
- https://github.com/lobokiol/NovelToVideo/blob/main/backend/app/services/screenwriting/query_intent_llm.py
- https://github.com/lobokiol/NovelToVideo/blob/main/backend/app/services/screenwriting/rag_index.py
- https://github.com/lobokiol/NovelToVideo/blob/main/backend/app/services/screenwriting/rag_runtime.py
- https://github.com/lobokiol/NovelToVideo/blob/main/backend/app/core/agent/embedding.py
- https://github.com/lobokiol/NovelToVideo/blob/main/backend/app/core/harness/memory/vector.py
- https://github.com/lobokiol/NovelToVideo/blob/main/backend/app/services/screenwriting/chat.py

---

> **SolThink版底线：不要把“懂 RAG”说成“会背 RAG 名词”。真正高分的回答，是能用一次真实查询把数据、意图、检索、排序、上下文、生成、失败恢复和验证证据串起来，并且在当前项目没有某个能力时敢明确说没有。**
