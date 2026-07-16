项目：医院导诊 Agent

架构：LangGraph 17 节点 + 三路由（疾病/症状/拒答）

RAG：OpenSearch BM25+kNN，三类索引

工具：MCP（值班/介绍/路线）

持久化：Redis Checkpoint + SQLite 全周期

评测：Golden 450 + Batch 100

成果：科室≥90%，急诊100%，开源可演示

技术栈：Python FastAPI LangGraph OpenSearch Redis DashScope React

差异化：12年经验 + 测试背景 → 评测与稳定性

bge-large-zh-v1.5，bge-reranker-v2-m3

## 环节一：自我介绍（约 1.5～2 分钟）

### 中文口述版（推荐背这个）

> 面试官您好，我叫刘博，本科学历，有 12 年软件工程经验，其中约 5 年做测试，2024 年起转型 AI Agent / LLM 应用开发，目前在 中软国际 担任 AI Agent 工程师。
>
> 我近期主要负责 医院导诊问答智能体 的全链路开发。这个项目面向医院小程序和导诊台场景，采用 LangGraph 17 节点状态机 做编排，症状链走 ES BM25 + kNN 混合 RAG，疾病链走结构化知识库直查，追问链通过 MCP 对接医院工具，比如值班预约、科室介绍和来院路线。
>
> 我负责的核心工作包括：状态机设计与节点实现、槽位门禁和急诊护栏、RAG 索引与检索 pipeline、MCP 工具集成、Redis 会话持久化、SQLite 导诊记录，以及 React Web + Rich CLI 双端演示 和 Golden Set 分层评测。
>
> 生产环境评测上，症状和疾病科室准确率 ≥90%，急诊识别 100%；开源参考实现 Batch 100 综合通过率 80%，意图路由 97%。项目在 GitHub 有完整可演示仓库，面试时可以现场跑 Web 或 CLI 全链路。
>
> 技术栈主要是 Python、FastAPI、LangGraph、OpenSearch、Redis、DashScope，前端 React + Vite。此前测试背景让我在 Golden Set 评测、CI 回归和稳定性 上比较有优势。
>
> 今天希望能和您进一步交流这个岗位，谢谢。

## 环节二：面试官提问（高频 + 参考答案）

### Q1：介绍一下这个导诊项目，整体架构是什么？

答：

这是一个 分层导诊 Agent，不是单轮问答，而是多轮有状态对话。

用户输入后，链路大致是：

1. trim_history 裁剪上下文
2. decision（NER） 做意图路由：疾病 / 症状 / 拒答
3. slot_fill + slot_gate：补齐年龄、性别、部位等槽位
4. emergency_gate：急诊硬规则优先
5. 分流：
   - 疾病链 → `disease_dept` 查结构化 KB
   - 症状链 → `rag_symptom_recall` 混合检索 → 澄清 → 科室规则打分 → 置信度门禁
6. mcp_followup：追问值班、科室介绍、路线
7. answer_generate 生成最终回复

持久化：Redis Checkpoint 做多轮会话，SQLite 记完整导诊周期，/ready 聚合 OpenSearch、Redis、SQLite、LangGraph 健康状态。

------

### Q2：你在项目里的具体职能？负责哪些模块？

答（按简历如实说）：

| 模块           | 我负责的内容                                                 |
| :------------- | :----------------------------------------------------------- |
| LangGraph 编排 | 17 节点状态机设计、`builder.py` 条件边、多轮续跑与拒答后状态清理 |
| 领域建模       | `AppState`、NER/路由 Schema、三路由分流                      |
| 槽位与护栏     | `TriageSlotTable`、`emergency_gate`、`dept_confidence` 低置信拒答 |
| RAG            | 三类索引 mapping、embed 入库、BM25+kNN+rerank pipeline       |
| MCP            | Stdio 客户端、`mcp_followup` 意图识别与工具路由              |
| 会话与可观测   | Redis Checkpoint、SQLite `triage_recorder`、`SessionManager` |
| 双端演示       | React Web（澄清选项、RAG 溯源）+ Rich CLI                    |
| 评测           | Golden 分层 A~E + Batch 100，支撑迭代回归                    |

测试背景带来的：Golden Set 设计、批量评测脚本、稳定性意识。

------

### Q3：项目多少人？怎么分工？你负责哪块？

答（模板，请替换【待填】）：

> 导诊 Agent 项目组大约 【待填：如 4～6 人】，采用敏捷迭代。
>
> - 产品/业务：需求、导诊规则、科室口径
> - 后端/Agent（我）：LangGraph 编排、RAG、MCP、API、评测
> - 前端【若有】：小程序或 Web 对接
> - 算法/数据【若有】：知识库维护、Golden 标注
> - 测试【若有】：用例与回归
>
> 我偏 Agent 全链路 Owner：从状态机、检索、工具调用到评测和演示环境。生产版还对接 vLLM 私有化 和 HIS，开源仓聚焦可演示核心链路。

若实际偏独立开发，可改为：

> 在 Agent 核心链路上基本是 我独立负责，与业务方、测试协作验收；知识库迭代和线上问题我会通过维护 JSONL + 重新入库闭环。

------

### Q4：为什么用 LangGraph，不用 Dify / 纯 Prompt Chain？

答：

导诊是 强状态、多分支、可恢复 的流程：

- 要 条件路由（疾病 / 症状 / 急诊 / 拒答）
- 要 多轮澄清（最多 3 轮，触顶 fallback）
- 要 Checkpoint 续跑（用户选澄清选项后从图中断点继续）
- 要 节点级可测（每个节点可单测、可观测）

LangGraph 比纯 Chain 更适合这种 状态机 + 人机协同（HITL） 场景。Dify 适合快速 POC；这个项目要 生产级可控、可评测、可对接 HIS，所以选代码级编排。

------

### Q5：RAG 怎么做的？为什么混合检索？

答：

三类 OpenSearch 索引：

1. rag_knowledge：症状澄清、急诊规则、机制类知识
2. disease_kb：疾病 → 科室结构化直查
3. rag_department_rules：科室消歧规则

症状链用 BM25 + kNN 混合召回，再做 rerank_by_alliance（诱因/伴随症状等）。
纯向量对医学俗语、别名不稳；纯 BM25 对语义泛化弱，混合更稳。

维护方式：改 `sourceData/data/*.jsonl` → 跑入库脚本 → Golden 回归验证。

------

### Q6：急诊护栏怎么保证 100%？

答：

emergency_gate 是 硬规则优先，不依赖 LLM 自由发挥：

- 知识库里有 `type: emergency` 条目（如大出血、意识不清、剧烈胸痛等）
- 规则命中直接推急诊话术，走专门分支
- Golden E 类急诊用例 单独评测，生产目标 100%

LLM 负责澄清和生成，安全边界用规则兜。

------

### Q7：MCP 在项目里解决什么问题？

答：

导诊推荐科室后，用户常追问：

- 今天哪个医生值班？
- 这个科室在哪栋楼？
- 怎么走？

这些要走 医院 HIS/内部 API，不适合全塞进 Prompt。
用 MCP Stdio 接 `hospital_mcp`（生产接真实 HIS，开源仓用 Mock），`mcp_followup` 节点做 意图识别 → 工具选择 → 结果回填。
这样 Agent 编排与医院系统解耦，工具可独立迭代。

------

### Q8：多轮会话怎么持久化？挂了怎么办？

答：

- Redis Checkpoint：LangGraph 状态，支持多 `thread_id`
- 无 Redis 时 MemorySaver 回退（`USE_MEMORY_CHECKPOINTER=true`）
- SessionManager：`user_id` ↔ 多会话元数据
- SQLite triage_recorder：完整导诊周期（`turns_json`、outcome、state 快照），便于复盘和评测

`/ready` 会检查 OpenSearch、Redis、SQLite、图编译状态，方便运维探活。

------

### Q9：评测怎么做？指标怎么来的？

答：

Golden 分层：

| 类别 | 测什么            |
| :--- | :---------------- |
| A    | 召回              |
| B·F  | 科室（单轮/多轮） |
| C    | 疾病链            |
| D    | 拒答              |
| E    | 急诊              |

生产 450 条 Golden：症状/疾病科室 ≥90%、急诊 100%、误推 ≤5%。
开源仓 Batch 100：意图 97%、NER 81%、综合 80%。

测试背景让我把 可回归、可量化 放在迭代前面，改知识库或节点后先跑评测再上线。

------

### Q10：岗位技术栈相关（AI Agent 岗常问）

| 问题                           | 简答要点                                                     |
| :----------------------------- | :----------------------------------------------------------- |
| LangGraph vs LangChain         | Graph 有状态、条件边、Checkpoint；Chain 更适合线性流         |
| 如何做失败恢复                 | Checkpoint 续跑；拒答清空导诊状态保留 messages；低置信 fallback |
| 如何控成本                     | 规则优先减少 LLM 调用；trim_history；澄清轮次上限 3          |
| 可观测性                       | LangSmith、日志、`/ready`、SQLite 全周期记录                 |
| 前端你做到什么程度             | React 多轮澄清 UI、斜杠命令、RAG 溯源、API 代理              |
| 和第二个项目（SSG 问数）的关系 | 同样做 Pipeline 编排和 Prompt 迭代；Chrome 插件做批量验证 Agent，思路与 Golden 评测相通 |

------

### Q11：你从测试转 Agent，优势是什么？

答：

- 习惯 用例驱动：Golden Set、分层评测、Batch 回归
- 关注 边界和异常：急诊护栏、低置信拒答、RAG miss reject
- 有 自动化经验：评测脚本、CI、批量验证（SSG 项目 2～3h → 30min）
- 联调思维：HIS/MCP、OpenSearch、Redis 多组件 `/ready` 聚合

不是只会调 Prompt，而是能 交付可测、可部署、可演示 的 Agent。

------

## 环节三：面试者提问（问技术面试官）

原则：业务、岗位、技术、模块可以问；薪资留给 HR/终面。

### 推荐问题（选 3～4 个）

业务与场景

1. 贵司 AI Agent 主要落地在哪些业务场景？是内部提效还是面向客户的产品？
2. 如果也是医疗/政务类，合规和拒答策略上团队一般怎么定边界？

岗位与职责

1. 这个岗位更偏 Agent 编排与 RAG，还是偏 模型微调 / 平台开发？入职前几个月的核心交付是什么？
2. 团队里 Agent、后端、算法、测试的分工是怎样的？我这类背景会更偏哪块？

技术栈

1. 贵司 Agent 编排用 LangGraph、Dify 还是自研框架？Checkpoint 和工具调用怎么选型？
2. 检索是向量、混合还是图谱？知识库谁维护、更新频率如何？
3. 生产环境模型是私有化（vLLM）还是云 API？对成本和延迟有什么要求？

工程与质量

1. 团队有没有 Golden Set / 线上评测闭环？发布前回归标准是什么？
2. CI/CD 和可观测（LangSmith、日志、链路追踪）目前成熟度如何？

成长

1. 团队对 Agent 工程师在 评测、Prompt、工程化 上的成长路径是怎样的？

### 尽量不要问技术面试官的

- 薪资、年终奖、几薪
- 加班有没有加班费（可问工作节奏，别先谈钱）
- 能否躺平、管得严不严

可改为：「团队日常迭代节奏是怎样的？上线前一般怎么评审？」