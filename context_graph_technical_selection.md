# 语境图谱技术选型调研

调研日期：2026-06-11

本文从技术选型角度整理“语境图谱”的相关概念、来源、优势、劣势、适用问题和常见框架。需要先说明：语境图谱目前不像知识图谱一样有高度统一的行业标准，中文语境中的“语境图谱”通常对应英文里的 Context Graph、Contextual Graph、Contextual Knowledge Graph、Temporal Knowledge Graph、Agent Memory Graph 或 GraphRAG Context Graph 等相关概念。

因此，本文将“语境图谱”作为一个工程范式来讨论：它不是完全替代知识图谱，而是在知识图谱、事件图谱、时序图谱、上下文工程和大模型 Agent 记忆之上的组合型技术方案。

## 1. 一句话定义

语境图谱可以理解为：

> 在知识图谱的实体和关系基础上，额外表达时间、来源、任务、用户意图、决策过程、工具调用、证据、权限、置信度、历史状态等上下文信息的动态图结构。

简单对比：

```text
知识图谱：谁和谁有什么关系
语境图谱：在什么时间、什么场景、基于什么证据、为什么形成这个关系或决策
```

例如：

```text
用户A --询问--> 问题Q
问题Q --命中--> 实体E
实体E --出现在--> 文档D
文档D --支持--> 回答R
回答R --生成于--> 会话S
回答R --使用模型--> 模型M
回答R --依据--> 证据片段C
证据片段C --有效时间--> 时间T
```

从工程角度看，语境图谱通常包含：

| 组成 | 说明 | 示例 |
|---|---|---|
| 实体 | 现实对象或业务对象 | 用户、客户、公司、产品、文档、系统 |
| 关系 | 实体之间的连接 | 购买、引用、依赖、影响、审批 |
| 事件 | 某个时间点或时间段发生的动作 | 查询、审批、调用工具、提交订单 |
| 时间 | 事实或事件的发生时间、有效期、版本 | created_at、valid_from、valid_to |
| 来源 | 事实或上下文来自哪里 | 文档、数据库、日志、API、人工输入 |
| 证据 | 支撑事实或回答的片段 | 文档段落、表格行、日志记录 |
| 任务 | 当前目标或用户意图 | 风险分析、客服问答、代码修复 |
| 状态 | 会话、流程或 Agent 的中间状态 | pending、approved、failed |
| 置信度 | 自动抽取事实的可信程度 | 0.82、high、manual_verified |
| 权限 | 信息可被谁使用 | public、internal、restricted |
| 决策链 | 从输入到输出的推理或流程路径 | 输入 -> 规则 -> 证据 -> 结论 |

## 2. 为什么需要语境图谱

传统知识图谱回答的是“事实是什么”，但很多企业 AI 和业务自动化问题还需要回答：

1. 这个事实什么时候成立？
2. 这个事实来自哪里？
3. 这个事实是否已经过期？
4. 这个结论基于哪些证据？
5. 这个回答是否符合当前用户权限？
6. 这个用户在当前任务中的真实意图是什么？
7. 类似场景过去是如何处理的？
8. Agent 为什么调用了这个工具？
9. 某个决策出错后如何追溯责任链？

这些问题本质上不是单纯“知识”问题，而是“知识 + 上下文 + 过程 + 时间 + 来源 + 权限”的组合问题。

## 3. 概念来源与发展脉络

### 3.1 上下文感知计算

上下文感知计算是语境图谱的重要源头。Dey 和 Abowd 等人在 1999-2001 年左右研究 context-aware computing，强调系统应该利用用户、地点、对象、任务和环境信息来理解当前情境。

经典定义可以概括为：

```text
上下文是任何可以用来刻画实体所处情境的信息。
```

这里的实体可以是人、地点、对象、系统、服务或任务。

### 3.2 Contextual Graphs

Patrick Brezillon 等提出过 Contextual Graphs，用图结构表达上下文、流程、实践和决策条件。它更偏早期 AI、知识工程和决策支持系统，用来表示：

1. 标准流程。
2. 实际做法。
3. 触发分支的上下文条件。
4. 决策过程中发生的情境变化。

这类研究说明：上下文并不是一段附加文本，而可以被结构化建模。

### 3.3 语义网与溯源标准

语义网体系提供了很多表示上下文的基础能力：

| 标准 | 作用 |
|---|---|
| RDF Dataset / Named Graph | 将三元组放入命名图，表达来源、上下文或数据集边界 |
| PROV-O | 表达数据和结论的来源、生成过程、责任主体 |
| OWL-Time | 表达时间点、时间段、时间关系 |
| SHACL | 对图数据做形状和约束校验 |
| RDF-star / RDF 1.2 | 更方便地表达关于边或三元组的元数据 |

如果需要强治理、强溯源和标准化，语义网路线非常重要。

### 3.4 知识图谱增强

传统知识图谱中的事实往往被表达为：

```text
公司A --投资--> 公司B
```

但真实业务里还需要知道：

```text
投资比例是多少？
什么时候发生？
现在是否仍然有效？
数据来自工商库还是人工录入？
这个事实是否经过验证？
这个关系在哪个业务场景下有意义？
```

因此出现了带上下文、时间、来源、置信度和事件信息的知识图谱形态，也可以称为 contextual knowledge graph 或 temporal knowledge graph。

### 3.5 大模型、Agent 与 GraphRAG

大模型时代，“上下文”成为核心工程问题。模型不能无限读取所有文档、历史会话、数据库记录和工具执行结果，必须从海量信息中选择对当前任务最相关的上下文。

语境图谱在这里的价值是：

1. 将用户意图、历史对话、文档证据、工具调用和业务实体连接起来。
2. 用图遍历找到当前问题相关的上下文子图。
3. 将上下文子图压缩为可供大模型使用的 prompt 或 structured context。
4. 将模型回答、用户反馈和工具结果再写回图谱。

这也是 GraphRAG、Agent Memory Graph、Context Graph 在近几年变热的重要原因。

## 4. 和知识图谱的区别

| 维度 | 知识图谱 | 语境图谱 |
|---|---|---|
| 核心问题 | 有什么事实 | 在什么语境下，这个事实或决策成立 |
| 重点对象 | 实体、概念、关系、属性 | 实体、关系、事件、任务、证据、状态、决策链 |
| 时间特征 | 可以有时间，但不一定核心 | 时间、版本、有效期通常是核心 |
| 数据形态 | 相对稳定 | 更动态、更时序化 |
| 查询目标 | 查事实、查路径、做推理 | 查上下文、查原因、查先例、查证据、查状态 |
| 典型关系 | 任职于、购买、属于、引用 | 因为、触发、依赖、支持、反驳、过期、被批准 |
| 面向应用 | 搜索、推荐、风控、数据融合 | Agent 记忆、GraphRAG、审计、流程自动化、上下文工程 |
| 价值重点 | 组织领域知识 | 组织任务相关上下文 |
| 工程位置 | 语义知识层 | 运行时上下文层和决策记忆层 |

可以把二者关系理解为：

```text
知识图谱 = 稳定事实层
语境图谱 = 事实 + 时间 + 来源 + 任务 + 状态 + 决策过程
```

## 5. 主要解决哪些问题

语境图谱适合解决“上下文比单条事实更重要”的问题。

| 问题 | 语境图谱的价值 | 典型场景 |
|---|---|---|
| LLM 上下文窗口有限 | 从全量数据中选择最相关的上下文子图 | 企业知识助手、研究助手 |
| RAG 只靠向量相似度 | 用图关系补充相关但不相似的信息 | 多跳问答、复杂文档问答 |
| Agent 缺少长期记忆 | 持久化用户偏好、任务状态、历史结果 | 私人助手、客服 Agent |
| 决策不可解释 | 记录输入、规则、证据、审批、结果 | 风控、合规、审计 |
| 数据时效性差 | 给事实加时间、有效期和版本 | 金融、供应链、合同 |
| 多轮对话断裂 | 将当前会话与历史会话、实体、任务连接 | 智能客服、销售助手 |
| 工具调用难追踪 | 记录 Agent 调用了什么工具、结果是什么 | AgentOps、自动化运维 |
| 权限控制复杂 | 按用户、任务、数据来源过滤上下文 | 企业知识库、合规助手 |
| 业务流程难自动化 | 记录类似场景的处理路径和先例 | 工单、法务、ITSM |
| 事实冲突 | 基于来源、时间、置信度判断优先级 | 主数据、情报分析 |

## 6. 典型应用场景

### 6.1 企业级 GraphRAG

普通 RAG 通常流程是：

```text
问题 -> 向量召回文档片段 -> 拼 prompt -> LLM 回答
```

语境图谱增强后可以变成：

```text
问题 -> 识别实体/意图 -> 图遍历相关实体/文档/事件/证据 -> 权限过滤 -> 时间过滤 -> 构造上下文子图 -> LLM 回答
```

适合：

1. 大量文档之间存在引用、版本、主体、责任人关系。
2. 问题需要跨文档、多跳、多实体综合。
3. 需要回答来源和解释路径。

### 6.2 Agent 长期记忆

Agent 不只需要“记住用户说过什么”，还需要知道这些记忆的上下文。

例如：

```text
用户A 喜欢 简洁回答
该偏好 来源于 会话S
该偏好 适用于 技术问题
该偏好 不适用于 论文写作
该偏好 最后确认于 2026-06-01
```

这种结构比简单的文本记忆更可控。

### 6.3 决策审计与合规

在金融、医疗、法律、供应链等场景里，只给出结论通常不够，还需要解释：

1. 使用了哪些数据。
2. 哪些规则被触发。
3. 哪些人或系统参与了决策。
4. 是否存在过期或低可信数据。
5. 决策过程是否符合合规要求。

语境图谱可以将这些过程结构化保存。

### 6.4 复杂流程自动化

对于工单、售后、IT 运维、审批流，语境图谱可以记录：

1. 当前流程状态。
2. 历史类似案例。
3. 已尝试动作。
4. 失败原因。
5. 下一步建议。

### 6.5 数据血缘和影响分析

语境图谱可以表达数据从哪里来、如何被处理、被哪些模型或报告使用、产生了什么决策影响。

例如：

```text
数据表A -> ETL任务B -> 特征C -> 模型D -> 风险评分E -> 审批结果F
```

## 7. 优势

### 7.1 上下文选择更精准

语境图谱可以将多种检索方式组合起来：

1. 向量召回。
2. 关键词检索。
3. 图遍历。
4. 时间过滤。
5. 权限过滤。
6. 置信度排序。
7. 来源优先级排序。

这比单纯 top-k 向量召回更适合复杂任务。

### 7.2 更适合多跳推理

很多业务问题不是一跳关系，而是多跳关系：

```text
用户问题 -> 产品 -> 合同 -> 客户 -> 风险事件 -> 处理方案
```

语境图谱可以把中间路径显式保留下来。

### 7.3 可解释性更强

语境图谱可以返回路径和证据，例如：

```text
回答R 基于 文档D 的 段落P
段落P 描述 实体E
实体E 与 当前问题Q 通过 关系K 相关
文档D 来源于 系统S
文档D 更新时间为 T
```

### 7.4 支持时序和状态变化

语境图谱可以避免把旧事实和新事实混在一起。

例如：

```text
客户A 在 2024 年为高风险
客户A 在 2025 年风险解除
客户A 在 2026 年重新进入观察名单
```

如果没有时间语境，系统可能给出错误结论。

### 7.5 更适合 Agent 记忆

语境图谱可以表达记忆的适用范围：

| 记忆 | 上下文 |
|---|---|
| 用户喜欢简洁回答 | 技术讨论中成立 |
| 用户偏好中文 | 当前工作线程成立 |
| 用户之前拒绝某方案 | 针对某个项目成立 |
| 某接口曾经失败 | 特定环境和版本成立 |

这比纯文本记忆更容易治理和更新。

### 7.6 有利于降低幻觉风险

当模型回答必须绑定到证据、来源、时间和权限时，可以减少无依据生成。语境图谱不能完全消灭幻觉，但能让回答更容易被约束和审计。

## 8. 劣势与风险

### 8.1 概念不统一

Context Graph 还不是一个完全统一的标准术语。不同厂商和论文可能指代不同内容：

| 名称 | 可能含义 |
|---|---|
| Context Graph | LLM 上下文组织图、Agent 记忆图、语境关系图 |
| Contextual Graph | 早期知识工程中的上下文决策图 |
| Contextual Knowledge Graph | 带上下文元数据的知识图谱 |
| Temporal Knowledge Graph | 带时间维度的知识图谱 |
| Memory Graph | Agent 或应用的长期记忆图 |

选型时要避免只看名词，需要看具体数据模型和能力。

### 8.2 建模复杂度更高

知识图谱已经需要建模实体和关系，语境图谱还要建模：

1. 时间。
2. 来源。
3. 证据。
4. 任务。
5. 状态。
6. 权限。
7. 置信度。
8. 决策过程。
9. 上下文有效范围。

如果没有清晰边界，很容易过度设计。

### 8.3 数据采集难

很多上下文信息并不在数据库主表里，而分散在：

1. 聊天记录。
2. 工单系统。
3. 审批系统。
4. API 调用日志。
5. 文档历史版本。
6. 人工备注。
7. Agent 工具调用轨迹。
8. OpenTelemetry trace。

采集、清洗和对齐成本通常不低。

### 8.4 维护成本高

上下文具有生命周期，会过期、冲突、被撤销或被新信息覆盖。

需要考虑：

1. 事实如何失效。
2. 历史是否保留。
3. 冲突如何解决。
4. 低置信度事实如何处理。
5. 用户是否可以删除记忆。
6. 权限变化后旧上下文如何处理。

### 8.5 查询性能压力

语境图谱查询通常不仅是图遍历，还会叠加：

1. 时间过滤。
2. 权限过滤。
3. 来源过滤。
4. 置信度排序。
5. 多跳路径。
6. 向量召回融合。

如果设计不当，线上延迟可能很高。

### 8.6 隐私和合规风险更高

语境图谱可能存储用户偏好、历史交互、内部决策依据、敏感文档片段和工具调用结果。必须设计：

1. 数据脱敏。
2. 访问控制。
3. 数据保留周期。
4. 用户删除权。
5. 审计日志。
6. prompt 泄露防护。

### 8.7 自动抽取质量不稳定

使用 LLM 从文本、对话、日志中抽取上下文时，常见问题包括：

| 问题 | 示例 |
|---|---|
| 幻觉 | 文档中没有的关系被生成出来 |
| 漏抽 | 关键事件或限制条件没有提取 |
| 类型不一致 | 同一类关系被抽成多个名字 |
| 时间错误 | 发生时间、有效期被识别错 |
| 因果误判 | 相关性被误当成因果关系 |
| 权限缺失 | 抽取结果没有继承原始数据权限 |

## 9. 关键技术路线

### 9.1 RDF / 语义网路线

如果语境图谱的核心是治理、标准、溯源和互操作，RDF 体系很适合。

可用能力：

| 技术 | 作用 |
|---|---|
| RDF Dataset / Named Graph | 表示不同来源、不同上下文的数据边界 |
| PROV-O | 表示谁生成了数据、使用了什么输入、经历了什么过程 |
| OWL-Time | 表示时间点、时间段和时间关系 |
| SHACL | 校验图数据是否满足约束 |
| SPARQL | 查询语义图谱 |
| RDF-star | 表达边上的元数据，如置信度、来源、时间 |

适合：

1. 强合规。
2. 强审计。
3. 跨系统语义集成。
4. 数据血缘。
5. 标准化知识治理。

### 9.2 Property Graph 路线

如果核心是工程落地、路径查询、Agent 记忆和 GraphRAG，属性图通常更直接。

示例：

```text
(User {id: "u1"})
  -[:ASKED {time: "2026-06-11"}]->
(Question {id: "q1"})
  -[:SUPPORTED_BY {score: 0.91}]->
(Evidence {doc: "policy.pdf", paragraph: 12})
```

适合：

1. 低延迟图查询。
2. 会话和事件追踪。
3. GraphRAG。
4. Agent memory。
5. 决策路径解释。

代表技术：

1. Neo4j。
2. Memgraph。
3. FalkorDB。
4. TigerGraph。
5. JanusGraph。
6. NebulaGraph。

### 9.3 GraphRAG 路线

GraphRAG 强调从文本或业务数据中构建图结构，再用图来增强检索和生成。

典型流程：

```text
文档 -> 实体抽取 -> 关系抽取 -> 社区/主题聚合 -> 图检索 -> 上下文构造 -> LLM 回答
```

适合：

1. 企业文档问答。
2. 复杂报告总结。
3. 多跳知识问答。
4. 研究资料分析。
5. 长文档集合检索。

代表框架：

1. Microsoft GraphRAG。
2. Neo4j GraphRAG。
3. LlamaIndex PropertyGraphIndex。
4. LangChain + Neo4j。
5. FalkorDB GraphRAG。

### 9.4 Agent Memory Graph 路线

Agent Memory Graph 关注智能体长期记忆和动态上下文。

需要表达：

1. 用户偏好。
2. 历史任务。
3. 对话摘要。
4. 工具调用结果。
5. 环境状态。
6. 失败经验。
7. 记忆有效范围。

代表工具：

1. Graphiti / Zep。
2. Cognee。
3. Mem0。
4. LangGraph persistence。
5. 自建 Neo4j / Postgres + vector store 组合。

### 9.5 观测与 Trace 路线

对于 Agent 和复杂系统，很多上下文来自运行时 trace。

可采集：

1. 请求链路。
2. 工具调用。
3. 模型调用。
4. 输入输出。
5. 错误信息。
6. 延迟和成本。
7. 决策分支。

代表技术：

1. OpenTelemetry。
2. LangSmith。
3. Arize Phoenix。
4. Helicone。
5. 自建 trace event graph。

这一路线通常不是独立替代图谱，而是为语境图谱提供运行时数据来源。

## 10. 常见框架和产品对比

| 类型 | 推荐工具 | 适合情况 |
|---|---|---|
| Agent 时序语境图谱 | Graphiti / Zep | Agent 长期记忆、时间变化、动态事实 |
| GraphRAG 框架 | Microsoft GraphRAG | 从文档构建图结构，做全局总结和多跳问答 |
| 图数据库 + GraphRAG | Neo4j GraphRAG | 企业图数据库、Cypher、路径解释、RAG 集成 |
| LLM 图索引 | LlamaIndex PropertyGraphIndex | 快速从文档抽取属性图并查询 |
| Agent 状态编排 | LangGraph | Agent workflow、checkpoint、持久状态 |
| AI 记忆层 | Cognee、Mem0 | 长期记忆、图 + 向量混合检索、跨会话上下文 |
| 高性能属性图 | FalkorDB、Memgraph、Neo4j | 低延迟图查询、Agent memory、GraphRAG |
| 企业语义治理 | GraphDB、Stardog、Apache Jena、RDF4J | 本体、溯源、标准化、SPARQL、SHACL |
| 观测和决策痕迹 | OpenTelemetry、LangSmith、Phoenix | 采集 Agent 步骤、工具调用、服务链路 |
| 通用图分析 | NetworkX、Neo4j GDS、TigerGraph | 社区发现、路径分析、影响分析 |

## 11. 技术选型建议

### 11.1 按目标选择

| 目标 | 建议 |
|---|---|
| 企业文档问答增强 | Microsoft GraphRAG、Neo4j GraphRAG、LlamaIndex PropertyGraphIndex |
| Agent 长期记忆 | Graphiti/Zep、Cognee、Mem0 |
| 流程和决策追踪 | 图数据库 + OpenTelemetry + 事件日志 + PROV-O |
| 强治理、强溯源、强标准 | RDF + Named Graph + PROV-O + OWL-Time + SHACL |
| 实时业务状态图 | Neo4j、FalkorDB、Memgraph、TigerGraph |
| 已经有知识图谱 | 在现有 KG 上增加时间、来源、置信度、任务、权限、事件层 |
| 普通 FAQ/RAG | 不建议一开始就上语境图谱，先用向量检索或混合检索验证 |
| 数据血缘和影响分析 | PROV-O / RDF 或属性图都可以，取决于标准化要求 |
| 风控和审计 | 属性图 + 事件图 + 溯源模型 |

### 11.2 按成熟度选择

| 成熟度 | 建议 |
|---|---|
| 概念验证 | Neo4j、LlamaIndex、Microsoft GraphRAG、Graphiti |
| 小规模生产 | Neo4j + 向量库 + 事件日志，或 Zep/Graphiti |
| 企业级治理 | RDF/PROV-O/SHACL + GraphDB/Stardog/Jena |
| 高并发图查询 | Neo4j Enterprise、FalkorDB、Memgraph、TigerGraph |
| 大规模离线分析 | Spark GraphFrames、Neo4j GDS、TigerGraph |

### 11.3 按数据类型选择

| 数据类型 | 建议 |
|---|---|
| 文档 | GraphRAG + 实体/关系抽取 + 证据节点 |
| 多轮对话 | Agent memory graph + 时间和任务上下文 |
| 业务事件 | Event graph + temporal graph |
| 日志和 trace | OpenTelemetry -> trace graph |
| 主数据 | 知识图谱 + 语境元数据 |
| 审批和决策 | PROV-O 或决策链图谱 |
| 用户偏好 | 带适用范围和有效期的 memory graph |

### 11.4 按查询模式选择

| 查询模式 | 更适合的路线 |
|---|---|
| 查事实和关系 | 知识图谱 |
| 查证据和来源 | 语境图谱 + provenance |
| 查某个时间点的状态 | temporal graph |
| 查多跳上下文 | property graph + GraphRAG |
| 查决策过程 | provenance graph / decision graph |
| 查 Agent 记忆 | memory graph |
| 查文档主题和摘要 | GraphRAG |

### 11.5 按团队能力选择

| 团队情况 | 建议 |
|---|---|
| AI 应用团队 | LlamaIndex、LangChain、Graphiti、Mem0 起步较快 |
| Java 后端团队 | Apache Jena、RDF4J、JanusGraph |
| 图数据库经验强 | Neo4j、Memgraph、FalkorDB |
| 数据治理团队强 | RDF、PROV-O、SHACL、GraphDB、Stardog |
| 平台工程团队强 | OpenTelemetry + 图数据库 + 自建上下文服务 |
| 初创 POC | 先选 Neo4j / LlamaIndex / Graphiti，减少标准化负担 |

## 12. 推荐架构

### 12.1 通用架构

```text
业务数据 / 文档 / 会话 / 工具调用 / 日志
        ↓
抽取实体、事件、关系、时间、来源、权限、置信度
        ↓
语境图谱：实体 + 关系 + 事件 + 证据 + 时间 + 决策链
        ↓
检索：向量召回 + 图遍历 + 规则过滤 + 权限过滤
        ↓
构造上下文子图
        ↓
压缩成 Prompt / 供 Agent 调用
        ↓
回答、行动、审计、反馈写回图谱
```

### 12.2 企业 GraphRAG 架构

```text
文档库
  ↓
解析与切分
  ↓
实体抽取 / 关系抽取 / 主题聚类
  ↓
向量索引 + 语境图谱
  ↓
问题理解
  ↓
实体链接 + 图遍历 + 向量召回
  ↓
证据排序 + 权限过滤 + 时间过滤
  ↓
LLM 回答
  ↓
来源引用 + 反馈写回
```

### 12.3 Agent 记忆架构

```text
用户输入 / Agent 动作 / 工具结果
        ↓
事件记录
        ↓
记忆抽取：偏好、事实、任务、约束、失败经验
        ↓
语境图谱：用户-任务-记忆-证据-时间
        ↓
下一轮任务检索相关记忆
        ↓
Agent 决策
        ↓
新结果写回
```

## 13. POC 建议

### 13.1 先明确边界

语境图谱很容易过度设计，POC 前建议先回答：

1. 我们要保存哪些上下文？
2. 哪些上下文会影响模型回答或业务决策？
3. 哪些上下文必须可追溯？
4. 哪些上下文有权限限制？
5. 哪些上下文会过期？
6. 哪些上下文需要写回？
7. 哪些上下文只需要日志，不需要入图？

### 13.2 设计 5 到 10 个真实问题

例如：

```text
1. 这个回答基于哪些文档和证据？
2. 这个客户当前风险判断是否使用了过期数据？
3. 用户上次在类似任务中偏好什么处理方式？
4. Agent 为什么调用这个工具，而不是另一个工具？
5. 某个审批结论经过了哪些规则和人工节点？
6. 某个文档版本变化会影响哪些回答和流程？
7. 当前用户是否有权限使用这段上下文？
8. 某个问题的答案在不同时间为什么不同？
9. 哪些历史案例与当前工单最相似？
10. 这个模型输出是否引用了低可信来源？
```

### 13.3 POC 指标

| 指标 | 说明 |
|---|---|
| 回答准确率 | 语境图谱是否提升问答质量 |
| 上下文召回率 | 是否能找回关键证据和历史上下文 |
| 幻觉率 | 回答中无依据内容是否下降 |
| 路径可解释性 | 是否能输出证据链和决策链 |
| 查询延迟 | 图遍历和过滤是否满足线上要求 |
| 写入延迟 | 会话、事件、工具调用是否能及时入图 |
| 权限正确性 | 是否能避免越权上下文进入 prompt |
| 时效性 | 是否能正确处理过期事实 |
| 维护成本 | 模型、抽取、清洗、图结构是否可维护 |
| 用户体验 | 是否比普通 RAG 更稳定、更可信 |

## 14. 落地步骤

### 阶段 1：确定业务问题

不要从“建一个语境图谱”开始，而是从具体问题开始：

1. Agent 记不住用户偏好。
2. RAG 回答没有上下文和证据。
3. 审批决策无法追溯。
4. 多轮任务状态容易丢失。
5. 历史案例不能复用。

### 阶段 2：定义最小语境模型

先定义最小可用模型，而不是完整本体。

可以从这些类型开始：

1. User。
2. Task。
3. Session。
4. Document。
5. Evidence。
6. Entity。
7. Event。
8. Decision。
9. ToolCall。
10. Memory。

### 阶段 3：接入数据源

优先接入对当前场景最关键的数据：

1. 文档库。
2. 会话记录。
3. 工具调用日志。
4. 业务数据库。
5. 审批或工单系统。
6. OpenTelemetry trace。

### 阶段 4：实现检索和上下文构造

推荐组合：

```text
实体识别 + 向量召回 + 图遍历 + 权限过滤 + 时间过滤 + 证据排序
```

最终输出给模型的不是全图，而是一个经过筛选和压缩的上下文子图。

### 阶段 5：写回与治理

系统运行后，需要将以下内容写回：

1. 用户反馈。
2. 新发现事实。
3. 工具调用结果。
4. 决策结果。
5. 失败案例。
6. 记忆更新。

同时要治理：

1. 过期上下文。
2. 错误上下文。
3. 冲突上下文。
4. 敏感上下文。
5. 低置信度上下文。

## 15. 快速结论

如果用于技术选型，可以用下面的判断：

| 判断条件 | 结论 |
|---|---|
| 只需要组织稳定事实 | 知识图谱即可 |
| 需要回答“为什么这样决策” | 需要语境图谱 |
| 需要 Agent 记住长期状态 | 优先看 Graphiti/Zep、Cognee、Mem0 |
| 需要企业级 GraphRAG | 优先看 Neo4j GraphRAG、Microsoft GraphRAG、LlamaIndex |
| 需要标准化语义治理 | 走 RDF/PROV-O/OWL-Time/SHACL |
| 需要低延迟图遍历 | Neo4j、FalkorDB、Memgraph、TigerGraph |
| 只是普通 FAQ 或简单 RAG | 不建议一开始就上语境图谱 |

核心建议：

1. 语境图谱不要替代知识图谱，而应作为知识图谱之上的运行时上下文层和决策记忆层。
2. 先从一个高价值场景做 POC，例如企业文档问答、Agent 长期记忆、决策审计或工单自动化。
3. 不要低估上下文治理成本，尤其是时间、权限、来源、置信度和删除机制。
4. 对大模型应用来说，语境图谱的价值不是让模型知道更多，而是让模型知道当前任务里哪些信息相关、可信、有效、可用。
5. 最实用的组合通常是“向量检索 + 图数据库 + 事件日志 + 来源追踪 + 权限过滤”。

## 16. 参考资料

1. [IBM: What is a Context Graph?](https://www.ibm.com/think/topics/context-graph)
2. [Dey: Understanding and Using Context](https://sites.cc.gatech.edu/fce/ctk/pubs/PeTe5-1.pdf)
3. [Representation of Procedures and Practices in Contextual Graphs](https://www.cambridge.org/core/journals/knowledge-engineering-review/article/representation-of-procedures-and-practices-in-contextual-graphs/D315A6EBC747228E20559BB379BE1F96)
4. [W3C RDF 1.1 Concepts and Abstract Syntax](https://www.w3.org/TR/rdf11-concepts/)
5. [W3C PROV-O: The PROV Ontology](https://www.w3.org/TR/prov-o/)
6. [W3C OWL-Time](https://www.w3.org/TR/owl-time/)
7. [W3C SHACL Shapes Constraint Language](https://www.w3.org/TR/shacl/)
8. [Microsoft GraphRAG](https://microsoft.github.io/graphrag/)
9. [Neo4j GraphRAG Python Documentation](https://neo4j.com/docs/neo4j-graphrag-python/current/user_guide_rag.html)
10. [LlamaIndex Property Graph Index](https://developers.llamaindex.ai/python/framework/module_guides/indexing/lpg_index_guide/)
11. [LangGraph Documentation](https://docs.langchain.com/oss/python/langgraph/overview)
12. [Graphiti / Zep Documentation](https://help.getzep.com/graphiti/getting-started/overview)
13. [Cognee Documentation](https://docs.cognee.ai/)
14. [Mem0 Documentation](https://docs.mem0.ai/)
15. [FalkorDB GenAI Tools Documentation](https://docs.falkordb.com/genai-tools/)
16. [OpenTelemetry Traces Documentation](https://opentelemetry.io/docs/concepts/signals/traces/)
17. [Apache Jena](https://jena.apache.org/)
18. [Eclipse RDF4J](https://rdf4j.org/)
19. [GraphDB](https://www.ontotext.com/products/graphdb/)
20. [Stardog Platform](https://www.stardog.com/platform/)
21. [Neo4j Documentation](https://neo4j.com/docs/)
22. [Apache TinkerPop](https://tinkerpop.apache.org/)
