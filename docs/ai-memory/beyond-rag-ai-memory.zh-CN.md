---
layout: page
title: "从 RAG 到 AI Memory：为什么“记住聊天记录”还不够？"
---

# 从 RAG 到 AI Memory：为什么“记住聊天记录”还不够？

## 一个面向企业 AI 助手的连续上下文架构

## 先说结论

如果把 AI Memory 简单理解成“把聊天记录存起来，再做向量检索”，系统很快会遇到一组典型问题：上下文越积越长，临时指令被误当成长期偏好，过期结论被继续使用，不同工作空间的数据边界变得模糊，删除或纠正后的内容还可能从索引和缓存里重新回来。

所以，企业级 AI Memory 真正要解决的不是“让模型记得更多”，而是“让系统只在正确边界内记住正确内容，并且每一次使用都能解释、限制和撤回”。

这篇文章想讨论的是一个更可落地的做法：把 Memory 设计成独立的、受治理的系统能力。模型可以帮助抽取候选记忆，但哪些内容可以长期保存、保存到哪个作用域、什么时候能进入 Prompt、来源变化后如何撤回，应该由后端策略和数据模型来决定。

## 问题从哪里来

很多企业 AI 助手的第一阶段，通常是把模型接到知识库上：用户问一个问题，系统检索相关资料，再把资料交给模型生成回答。这就是 RAG 最常见的价值：让模型在回答当前问题时，能够参考外部知识。

但一旦 AI 助手从“一问一答”进入连续工作，问题就变了。

例如，用户昨天让 AI 助手分析一份文档，今天继续修改同一份文档。他期望助手知道：当前文档是哪一版，昨天已经排除了哪些方案，哪些资料已经核验过，哪些结论只是临时判断，后续还有哪些待办，以及用户偏好的输出格式是什么。

这时，仅仅“检索资料”已经不够。系统需要回答的不是“这次请求要查什么”，而是“这项工作之前进行到哪里，以及哪些历史状态现在仍然可以安全使用”。

最直接的做法，是把聊天记录全部塞进 Prompt，或者把所有聊天记录写进向量数据库，之后靠相似度搜索找回来。它们看起来都能让 AI 助手“记住过去”，但很快会暴露工程问题：上下文越来越长，临时指令被误当成长期偏好，过期结论被再次使用，不同工作空间的数据边界变得模糊。更麻烦的是，系统通常会为了检索，额外生成向量、关键词索引或摘要副本；如果原始记录后来被删除或纠正，而这些派生数据没有同步清理，AI 助手仍可能在后续检索中命中旧内容。

所以，本文讨论的 AI Memory 并不是“更长的上下文”，也不是“聊天记录的向量搜索”。更稳妥的做法，是把 Memory 设计成一层独立、受治理的系统能力：它只把少量未来确实有用的信息转化为结构化对象，并持续记录这些对象从哪里来、在哪里能用、现在是否仍然有效，以及需要撤回时如何清理。

换句话说，企业级 AI Memory 的目标不是让 AI 助手记得更多，而是让它只保留那些对后续工作仍有价值、来源清楚、作用域明确、状态有效的信息，并且知道什么时候不能再使用旧内容。

## 为什么不能直接“存聊天记录再搜”

如果不把 Memory 当成独立系统能力，而只是把历史对话“存起来再搜”，常见问题可以归纳为五类：

1. **上下文不连续**：跨会话后，AI 助手不知道当前文档是哪一版、前面已经做过哪些分析、还有哪些事项未完成。
2. **内容语义混杂**：用户偏好、项目背景、文档状态、研究结论和待办事项都被当成普通文本保存，后续很难安全复用。
3. **持久化边界不清**：一次性临时指令可能被错误晋升为长期偏好，某个工作空间中的事实也可能被带到其他无关任务中。
4. **索引与权威数据倒置**：系统把向量索引中的文本当作最终事实，忽略权威主记录已经更新、过期、删除或被替代的可能性。
5. **生命周期治理缺失**：系统没有把 Memory 状态和检索结果绑定起来。即使主记录已经被纠正、删除或标记过期，向量索引、关键词索引、摘要和缓存里仍可能残留旧内容，最终把已经失效的信息重新带回 Prompt，影响 RAG 回答的准确性、合规性和可追溯性。

这些问题说明，AI Memory 的难点不是“如何做相似度检索”，而是如何让系统在每一次写入和读取时都能回答以下这些问题：什么内容值得长期留下，允许在哪个边界内使用，当前是否仍然有效，依据哪些可信来源，以及当来源变化或用户要求删除时，如何把主记录、索引、摘要和缓存一起删除。

## 这篇文章想回答什么

为了让第一次接触这个话题的读者更容易进入，先给出本文的主线。后文会围绕以下五个判断展开：

- **先区分“记住的是什么”，再讨论“保存多久”。** 一条写作偏好、一条项目背景和一条历史研究结论，即使都需要长期保存，也必须遵守不同的读取和使用规则。
- **模型只能提出候选，不能决定长期保存。** LLM 可以从对话中提取“候选记忆”（Candidate Memory），但最终是否写入、写到哪个作用域、能否进入 Prompt，应由后端策略决定。
- **作用域过滤必须早于相关性排序。** 权限和工作空间边界不是排序特征，而是硬约束。
- **权威记录和检索索引要分开。** 精确索引、关键词索引和语义索引只是帮助发现记录的派生视图，不能取代主记录。
- **每次请求都重新组装上下文。** 系统不无限追加历史，而是根据当前任务、权限、时效和 Token 预算，选择本轮真正需要的记忆。

这条主线最终形成一个完整的工程闭环：哪些内容有资格被写入 Memory，模型抽取出来的候选如何被后端审查，读取时如何先做作用域过滤，索引命中后为什么还要回到权威记录确认状态，以及删除、纠正、过期和索引重建如何共同保证“可检索的记忆仍然可信”。

## 一个可落地的 Memory Layer 应该怎么设计

### 1. 第一步不是决定存多久，而是先判断“记住的是什么”

进入具体设计前，先看一个容易混淆的地方：很多系统一开始会把 Memory 分成“短期记忆”和“长期记忆”。这个分类听起来直观，但它只回答“保存多久”，没有回答“这条内容到底是什么”。

而在真实系统里，“是什么”往往比“存多久”更重要。

比如，下面三类内容都可能需要跨会话保留：

- 用户稳定偏好：以后回答尽量先给结论，再给细节。
- 当前文档状态：这份报告已经改到第二版，下一步要补风险控制章节。
- 历史研究线索：上次查过某个方向，但当时只有初步结论，还需要重新核验。

它们都可以被称为“长期记忆”，但使用方式完全不同。偏好可以直接影响表达风格；文档状态只能用于对应文档；历史研究线索不能直接当成当前事实，必要时还要重新验证。

因此，本文使用 Typed Memory 这个说法。它的意思不是简单给文本贴标签，而是先判断一条记忆属于哪种业务含义，再决定它能不能写入、应该怎样读取、只能在哪个边界内复用、如何去重合并，以及在来源变化、内容冲突或时效到期时应该怎样处理。

如果只分 short-term 和 long-term，系统很难回答以下更关键的问题：

- 这条内容究竟是什么，应该使用哪套 Schema？
- 它能否进入长期存储，还是只能停留在过程记录中？
- 它允许在哪个用户、工作空间、项目或文档边界内复用？
- 应该通过确定性查询读取，还是允许参与关键词或语义召回？
- 进入 Prompt 后是调整表达方式、补充工作背景，还是仅提供历史探索线索？
- 来源变化、内容冲突或时效到期时，应删除、替代、标记 stale，还是重新验证？

落到工程实现上，Typed Memory 的核心不是给文本增加一个标签，而是为每种 Memory 建立一份**类型契约**。类型契约至少定义六个方面：

```text
Memory Type Contract
+ Extraction Schema   抽取时允许生成哪些字段
+ Promotion Rules     什么条件下可以成为正式记录
+ Scope Resolver      最终作用域由哪些可信上下文计算
+ Dedupe Semantics    怎样判断重复、补充、冲突和替代
+ Retrieval Policy    允许通过什么路径被读取，以及本轮用途
+ Lifecycle Policy    如何过期、刷新、更正、删除和重建索引
```

在工程实现中，`memory_type` 应直接对应系统会沉淀和治理的业务记忆类型。一个可落地的初始集合可以包括以下五类。

第一类是 `user_workstyle_preference`。它保存稳定的用户工作风格，例如输出结构、语气、详略程度、引用风格和格式偏好。它的边界也最容易被误用：不能把某个项目事实、客户事实、权限声明或一次性指令写成用户长期偏好。

第二类是 `matter_context`。它保存某个工作空间或事项下已经确认的背景，例如目标、事实、角色、约束和关键议题。这类记忆只能在绑定的工作边界内使用，不能跨项目复用，也不能被提升成用户通用偏好。

第三类是 `drafting_context`。它保存具体文档的状态，例如当前版本、修改方向、条款状态和未完成编辑项。它通常应该按文档、版本和所属工作边界确定性读取，不能把一个版本的 draft state 套到另一份文档或另一版文本上。

第四类是 `research_finding`。它保存的是研究路径，而不是当前答案：包括已经探索的问题、来源指针、工作性发现、限制条件和待核验点。它可以帮助系统恢复探索过程，但不能替代当前事实核查或当前权威检索；对时效敏感的判断，使用前必须重新验证。

第五类是 `workflow_open_task`。它保存明确的后续动作、状态、依赖、阻塞原因和完成证据。它只记录用户明确提出或系统明确确认的事项，不能让模型自行推断“也许应该做”的任务。

这些 `memory_type` 构成正式 Memory Record 的核心分类。它们决定了抽取 Schema、晋升规则、作用域计算、去重语义、检索策略和生命周期策略。

另有两类对象很重要，但不应该混进 `memory_type`。一类是 Trace，也就是原始事件、工具调用、已完成轮次和来源引用。Trace 是证据层，用于追溯、恢复、审计和抽取输入，默认不直接进入 Prompt，也不作为 Memory Card 渲染。另一类是 Runtime Policy，也就是写入、检索、保留、重新验证和 Prompt 组装规则。Runtime Policy 属于后端控制面，由受控策略库按版本加载和执行，不能从普通对话中生成，也不能作为业务记忆被检索。

#### 1.1 不同 Memory 类型如何改变系统行为

Typed Memory 不是展示标签，而是会改变后端处理链路中多个组件的行为，包括抽取、晋升、作用域计算、去重、检索、Prompt 组装和生命周期治理：

1. **影响抽取窗口。** 稳定偏好可能从单轮明确表达中抽取；项目背景或研究路径通常需要最近若干个相关轮次，才能避免断章取义。
2. **影响最终作用域。** 偏好通常绑定用户，文档状态绑定具体文档，项目背景绑定工作空间。模型只能建议，最终边界由服务端可信上下文计算。
3. **影响去重语义。** 两条措辞不同的偏好可能是同一个逻辑对象；两条冲突的研究发现则不应被简单覆盖，而应保留来源并进入复核或重新验证流程。
4. **影响检索路径。** 当前文档状态适合确定性读取；研究路径更适合在限定范围内组合精确、关键词和语义检索。
5. **影响 Prompt 用途。** 偏好只能改变表达方式，背景记忆只能提供当前工作上下文，研究记忆不能被提升为高优先级系统指令。
6. **影响生命周期治理。** 偏好可能长期有效，文档状态会随版本替代，研究记忆在来源变化或时效到期后应重新验证或标记 stale。

#### 1.2 Memory 类型与生命周期是两个正交维度

“短期还是长期”仍然重要，但应作为生命周期策略，而不是主分类。一个对象可以同时具有类型和生命周期：

| 生命周期       | 含义                                       | 典型对象                       |
| -------------- | ------------------------------------------ | ------------------------------ |
| Ephemeral      | 只在当前请求中使用，不进入持久存储         | 工具中间状态、临时计算结果     |
| Session-scoped | 服务于当前会话连续性，按会话过期           | Completed Turn、滚动摘要       |
| Durable scoped | 可跨会话复用，但受明确作用域和保留策略约束 | 稳定偏好、项目背景、文档状态   |
| Audit-only     | 默认不参与回答，仅用于追溯和问题排查       | 晋升决策、删除任务和运行 trace |

这样的二维设计可以避免两个极端：既不会因为“长期保存”就默认允许广泛复用，也不会因为“短期数据”就忽略它在删除、追踪和重建中的重要性。

#### 1.3 正式 Memory Record 需要什么

一条正式记录通常需要以下信息：

```text
Memory Record
+ stable record ID and version
+ memory type and subtype
+ canonical content + structured payload
+ server-resolved scope
+ source references and provenance
+ confidence / freshness / status
+ content hash + logical dedupe key
+ extraction, policy, and schema versions
+ index synchronization state
```

`stable record ID and version` 用来标识一条记忆及其当前版本。稳定 ID 解决“这是不是同一条 Memory”的问题，版本号解决“这条 Memory 是否已经被更新、替代或纠正”的问题。删除、审计、索引同步、引用回溯和冲突处理都依赖这两个字段。

`memory type and subtype` 用来说明这条记录属于哪种业务记忆。`memory type` 决定主契约，例如是用户偏好、事项背景、文档状态、研究路径还是未完成任务；`subtype` 则用于同一类型下的进一步区分，例如偏好里可以区分结构偏好、语气偏好、引用风格，文档状态里可以区分条款修改、版本状态或待补内容。类型不是展示标签，而是后端选择 Schema、Policy、作用域和检索方式的依据。

`canonical content + structured payload` 分别服务人和系统。`canonical content` 是经过归一化后的可读表达，便于审查、展示和生成 Memory Card；`structured payload` 则保存机器可处理的字段，例如文档 ID、条款类型、任务状态、authority pointer、待核验项等。只保存自然语言摘要会让后续过滤、去重和删除传播变得困难；只保存结构化字段又不利于解释和人工复核，所以两者通常需要同时存在。

`server-resolved scope` 表示这条 Memory 最终允许在哪个边界内使用。它不能完全相信模型输出，而应由服务端根据可信身份、当前工作空间、项目、文档、权限和来源记录计算。scope 的作用是先排除不该被看到或不该被复用的内容，再谈相关性排序。没有可靠 scope 的 Memory，即使语义上很相似，也不应该进入当前请求。

`source references and provenance` 记录这条 Memory 从哪里来。source references 指向原始事件、已完成轮次、工具结果、文档片段或用户确认；provenance 记录抽取时间、抽取策略、生成链路和晋升决策。它们让系统能够回答“为什么保存这条 Memory”，也让后续更正、删除、审计和重新抽取有据可查。

`confidence / freshness / status` 描述这条 Memory 当前是否可靠、是否新鲜、是否可用。confidence 表示抽取和晋升时的可信程度；freshness 表示内容是否可能过期，是否需要重新验证；status 表示记录处于 active、stale、superseded、pending review 或 deleted 等状态。检索命中一条 Memory 并不意味着可以直接使用，读取路径还必须检查这些状态。

`content hash + logical dedupe key` 用于去重、合并和冲突识别。content hash 可以快速判断文本或结构化内容是否发生变化；logical dedupe key 则表达业务意义上的“同一对象”，例如同一用户在同一任务类型下的输出结构偏好、同一文档同一条款的草稿状态、同一研究问题下的研究路径。没有逻辑去重，系统容易保存大量措辞不同但含义相同的记录；只有文本 hash，无法识别同一业务对象的补充、替代或冲突。

`extraction, policy, and schema versions` 记录这条 Memory 是按哪一版抽取策略、治理规则和数据结构生成的。它们的价值在于可解释和可迁移：当抽取 Prompt、Promotion Policy、Retrieval Policy 或 Schema 升级后，系统可以判断旧记录是否需要重新评估、回填、迁移或降低使用等级。

`index synchronization state` 说明这条权威记录与派生索引是否一致。Memory Record 是 source of truth，精确索引、关键词索引、语义向量、摘要缓存只是派生视图。同步状态可以标记索引是否已生成、是否滞后、是否删除完成、是否需要重建。这样可以避免主记录已经删除或纠正，但旧向量、旧关键词或旧缓存仍被检索命中的问题。

因此，Memory Record 的正文只回答“记住了什么”；类型、作用域、来源、版本、状态和索引同步共同回答“为什么能记住、在哪里能用、现在还能不能用”。这也是 Typed Memory 相比普通聊天摘要最重要的工程差异。

### 2. 整体上，要把“写入记忆”和“读取记忆”拆开

```text
写入侧

原始事件 ──> 已完成轮次 ──> 候选记忆 ──> 晋升门 ──> 权威记忆记录
                                  │           │
                                  │           └──> 来源追踪与审计
                                  │
                                  └──────────────> 异步索引任务
                                                    ├─ 精确索引
                                                    ├─ 关键词索引
                                                    └─ 语义索引

读取侧

当前请求 + 可信作用域
        ├─ 确定性查找：用户偏好、会话摘要、当前文档、未完成事项
        └─ 混合搜索：精确匹配 / 关键词 / 语义检索
                         │
                         v
                 回源读取权威记录
                         │
                 二次策略与时效校验
                         │
                 有边界的 Memory Cards
                         │
                       主模型
```

这套架构把“保存什么”和“如何找回来”分开：Memory Record 决定系统真正记住什么，索引只负责提高发现效率。

### 3. 写入侧：从聊天记录到正式记忆，中间不能少了治理关口

Write Path 解决的不是“总结聊天记录”，而是“怎样从一次已完成的对话中生成少量、可复用且可治理的记忆”。一条推荐的写入链路如下：

```text
WRITE-SIDE COMPONENT LAYOUT

┌────────────────────────────────────────────────────────────────────┐
│  Evidence Layer                                                     │
│                                                                    │
│  Raw Event Store                                                    │
│    └─ user input / assistant output / tool result / artifact refs   │
│                                                                    │
│  Admission Gate + Turn Builder                                      │
│    └─ filters incomplete, deleted, withdrawn, or out-of-scope data  │
│                                                                    │
│  Completed Turn Store                                               │
│    └─ clean interaction unit for continuity and extraction          │
└────────────────────────────────────────────────────────────────────┘
                                  │
                                  v
┌────────────────────────────────────────────────────────────────────┐
│  Extraction Layer                                                   │
│                                                                    │
│  Strategy Router + Input Builder                                    │
│    └─ chooses typed extraction strategy and builds bounded input    │
│                                                                    │
│  Typed Extractor                                                    │
│    └─ proposes candidate memory in the target schema                │
│                                                                    │
│  Candidate Store                                                    │
│    └─ reviewable proposals, not yet durable or retrievable memory   │
└────────────────────────────────────────────────────────────────────┘
                                  │
                                  v
┌────────────────────────────────────────────────────────────────────┐
│  Governance Layer                                                   │
│                                                                    │
│  Promotion Gate                                                     │
│    ├─ schema validation                                             │
│    ├─ scope resolution from trusted context                          │
│    ├─ write policy validation                                       │
│    ├─ dedupe / conflict handling                                    │
│    └─ provenance and audit decision                                 │
└────────────────────────────────────────────────────────────────────┘
                                  │
                                  v
┌────────────────────────────────────────────────────────────────────┐
│  Source-of-Truth Layer                                              │
│                                                                    │
│  Memory Record Store                                                │
│    └─ authoritative typed memory records with version and status    │
└────────────────────────────────────────────────────────────────────┘
                                  │
                                  v
┌────────────────────────────────────────────────────────────────────┐
│  Derived Index Layer                                                │
│                                                                    │
│  Transactional Index Outbox                                         │
│    └─ reliable async propagation from source of truth               │
│                                                                    │
│  Exact Index | Keyword Index | Semantic Index                       │
│    └─ discovery views only; retrieval must fetch source of truth    │
└────────────────────────────────────────────────────────────────────┘
```

#### 3.1 Admission Gating：先决定哪些事件有资格参与记忆抽取

系统可以保留用户输入、最终回答、工具结果和文档产物，用于恢复、追踪和排障，但“被记录”不等于“可作为抽取输入”。Admission Gating（准入门控）使用确定性规则排除不稳定或不应继续传播的事件。

| 事件状态                       | 是否进入 Completed Turn | 原因                                                 |
| ------------------------------ | ----------------------- | ---------------------------------------------------- |
| 用户请求与最终回答均已完成     | 是                      | 构成稳定的交互边界                                   |
| 本轮回答依赖的工具结果已返回   | 是                      | 避免只保存最终结论，却丢失检索、计算、文档读取等依据 |
| 流式片段或半截回答             | 否                      | 内容仍可能变化                                       |
| 未完成工具调用                 | 否                      | 当前轮次不完整                                       |
| 已删除、已撤回或被标记需要刷新 | 否                      | 不应继续成为可复用上下文                             |
| 自动建议但用户未选择的内容     | 否                      | 不是用户真实意图                                     |
| 与当前可信作用域不匹配         | 否                      | 不能进入当前工作边界                                 |

Admission Gating 不一定删除原始记录。原始记录仍可按保留策略存在，但不能参与会话摘要、Memory 抽取或后续 Prompt 组装。

#### 3.2 Turn Builder：把底层事件组装成稳定工作单元

一轮 AI Assistant 交互通常包含多条记录：用户请求、模型流式输出、工具调用、工具结果、最终回答以及文档更新。直接按 message 粒度抽取，可能只有用户请求，没有最终回答，或者最终回答处于中间态。

Turn Builder 负责将通过 Admission Gating 的事件组装成 Completed Turn。这里的 Completed Turn 指一轮已经稳定结束的交互单元。它通常包含用户请求、AI Assistant 的最终回答、本轮回答实际依赖的工具结果、文档引用、相关元数据和来源事件列表。它不是正式 Memory，而是后续会话摘要和 typed memory extraction 的干净输入。

Completed Turn 是 session continuity 和抽取流程的干净输入，但不是正式 Typed Memory。它回答“这一轮发生了什么”，而正式 Memory 回答“未来值得继续记住什么”。

Turn Builder 通常由异步 Worker 执行，并需要幂等 upsert：工具结果可能晚到，文档状态可能补全，源事件也可能随后被删除或标记 stale。相同内容的重复任务应 no-op（不执行写入变更）；来源或状态变化时，则更新同一个 Completed Turn 版本，而不是创建重复的 Completed Turn。

下面是一份通用化的 Completed Turn 示例。它保留可重建性和检索所需信息，但不包含真实账号、环境或业务数据：

```json
{
    "turn_id": "turn_001",
    "tenant_id": "tenant_abc",
    "actor_id_hash": "actor_hash_123",
    "workspace_id": "workspace_456",
    "project_id": "project_555",
    "document_id": "doc_789",
    "session_id": "session_789",
    "turn_index": 1,
    "user_message_ids": [
        "msg_001"
    ],
    "assistant_message_ids": [
        "msg_007"
    ],
    "tool_result_ids": [
        "msg_004",
        "msg_005"
    ],
    "artifact_message_ids": [
        "msg_006",
        "msg_008"
    ],
    "source_event_ids": [
        "evt_001",
        "evt_004",
        "evt_005",
        "evt_006",
        "evt_007",
        "evt_008"
    ],
    "user_text": "Review the current vendor onboarding guide and identify sections that need updates for the new approval workflow.",
    "assistant_final_text": "...final answer...",
    "normalized_metadata": {
        "domain": "Operations",
        "primary_topic": "vendor onboarding",
        "task_type": "document_review",
        "intent": "Analyze"
    },
    "derived_summaries": {
        "status": "generated",
        "method": "rules_and_llm",
        "tool_result_summary": "The latest guide and workflow configuration were loaded and compared.",
        "reference_summary": "Referenced the current document sections and the approved workflow configuration.",
        "draft_state_summary": "Generated an updated outline and marked three sections for revision.",
        "turn_summary": "User reviewed a vendor onboarding guide against the new approval workflow and received recommended document updates.",
        "retrieval_text": "Vendor onboarding guide review; new approval workflow; sections needing updates; updated outline and remaining revisions."
    },
    "token_estimate": 1260,
    "status": "active",
    "content_hash": "sha256:<turn-content-hash>",
    "is_completed": true,
    "completed_at": "2026-07-09T10:15:00Z"
}
```

这个示例里的字段可以分成两层理解：基础 Completed Turn 和派生摘要。基础 Completed Turn 由确定性事件组装而来，先回答“这一轮是否已经稳定结束、来源是什么、边界在哪里”。`derived_summaries` 是在基础记录之上生成的派生视图，用于压缩、检索和后续抽取；它可以晚生成、重新生成，甚至在第一版中不落库。这样可以避免把 Completed Turn 误解成“先让 LLM 总结，再拿总结去生成记忆”的循环链路。

`turn_id`、`session_id` 和 `turn_index` 用于定位和排序。`turn_id` 精确定位一个 Completed Turn；`session_id` 表示它属于哪次会话；`turn_index` 用于恢复会话内顺序。后续如果用户说“继续上一轮”或系统需要重建最近上下文，不能只靠时间戳排序，仍需要稳定的轮次 ID 和顺序号。

`tenant_id`、`actor_id_hash`、`workspace_id`、`project_id` 和 `document_id` 用于描述可信边界。它们不是让模型自由生成的标签，而应来自认证、权限、当前工作空间和文档系统。后续 Memory 抽取、检索和删除传播都需要这些字段决定“这条记录属于谁、属于哪个工作对象、能不能被当前请求读取”。

`user_message_ids`、`assistant_message_ids`、`tool_result_ids`、`artifact_message_ids` 和 `source_event_ids` 用于回源。它们把 Completed Turn 连接回底层事件、消息、工具结果和产物。这样当某条消息被删除、某个工具结果被纠正、某个文档产物被撤回时，系统可以知道哪些 Completed Turn 和下游 Memory 需要更新、标记 stale 或删除。

`user_text` 和 `assistant_final_text` 保存本轮最核心的输入和最终输出。它们不同于流式片段或中间草稿，只应记录稳定版本。实际系统里可以对这两个字段做长度控制：保留必要内容，超长正文则改为引用原始事件，避免 Completed Turn 变成另一份无限增长的 transcript。

`derived_summaries` 是可选派生层，不是 Completed Turn 的必需基础字段。基础记录可以先进入 Completed Turn Store；随后由规则、工具适配器或 LLM 生成 summary，用于降低后续抽取和上下文组装的成本。summary 字段服务不同目的，不能合并成一个万能摘要。

`tool_result_summary` 概括本轮回答实际依赖了哪些工具结果，例如加载了哪份文档、查了哪些系统、比较了哪些配置。它不是工具原文，也不是最终结论，而是帮助后续系统理解“回答依据来自哪里”。这类摘要可以由工具适配层先生成结构化摘要；如果工具结果很长，再由 LLM 在受控输入内压缩。

`reference_summary` 概括本轮引用或参考了哪些来源，例如当前文档的哪些章节、哪份配置、哪条内部政策或哪组数据。它用于审计和回源，不应写成新的事实结论。能由规则生成时优先规则生成，例如根据 document section IDs、config IDs、dataset IDs 拼出摘要；规则不足时再让 LLM 只基于已给定的 source refs 生成可读描述。

`draft_state_summary` 概括本轮对文档或产物状态造成了什么影响，例如生成了新大纲、修改了哪些章节、还留下哪些未完成编辑项。它是后续 `drafting_context` 抽取的重要输入，但本身仍只是 Completed Turn 的摘要，不等于正式文档状态 Memory。

`turn_summary` 是面向会话连续性的短摘要，回答“这一轮整体发生了什么”。它通常可以由 LLM 生成，因为它需要把用户意图、系统动作和结果压缩成一两句话。但生成时应有明确约束：只总结本轮已完成内容；不得新增未出现的事实；保留任务对象、用户目标、关键结果和未完成事项；不写推测性结论。

`retrieval_text` 是面向近端检索的文本，不一定适合直接展示给用户。它更像一个可搜索的压缩索引字段，应包含任务对象、关键术语、意图、结果和 open items。它可以由规则和 LLM 混合生成：规则先填入文档名、项目名、任务类型等稳定关键词，LLM 再把本轮语义压缩成适合 embedding 或 keyword search 的短文本。

`normalized_metadata` 保存结构化检索和路由信号，例如 domain、primary_topic、task_type 和 intent。它应尽量由后端规则、产品 taxonomy 或受控分类器生成，而不是让模型自由创造枚举值。模型可以提出候选分类，但最终值应经过允许列表和 scope 校验。

`token_estimate` 用于预算控制。Completed Turn 后续可能进入摘要、抽取或 Prompt 组装流程，系统需要提前估算它的大小，以便决定是否压缩、截断、回源或跳过。

`status`、`content_hash`、`is_completed` 和 `completed_at` 用于生命周期管理。`status` 表示当前是否可参与后续流程；`content_hash` 用于幂等更新和变更检测；`is_completed` 表示这一轮是否已经稳定结束；`completed_at` 记录完成时间。工具结果晚到、来源内容删除或文档状态更新时，这些字段会帮助系统判断是否需要更新同一个 Completed Turn 版本。

summary 字段是否需要 LLM，取决于输入复杂度和可结构化程度。短工具结果、明确文档引用和固定状态变化优先由规则或工具适配器生成；跨多条消息、多份文档或多个工具结果的综合摘要，可以使用 LLM。此时 LLM 的角色不是判断 Completed Turn 是否成立，而是在已完成、已定界、可回源的基础记录上生成派生摘要。为了让 LLM 生成稳定 summary，Prompt 应限制输入范围和输出格式，例如：

```text
Summarize only the completed turn below.
Do not add facts that are not present in the input.
Keep user goal, work object, actions taken, outputs produced, and open items.
Return JSON with:
- tool_result_summary
- reference_summary
- draft_state_summary
- turn_summary
- retrieval_text
Each field must be one short sentence.
If the input does not support a field, return null.
```

LLM 生成后仍应经过后端校验：字段必须是允许的 key，长度不能超预算，不能包含权限声明、secret、跨 scope 内容或与 source refs 不一致的断言。重要场景还可以保存 summary 的生成方法、Prompt / schema 版本和 source refs，方便后续人工审计或重新生成。如果 summary 缺失或生成失败，基础 Completed Turn 仍然有效，只是后续抽取可能需要回源读取更多原始内容。

#### 3.3 Strategy Router：按 Memory 类型选择抽取策略和上下文窗口

不要用一个万能 Prompt 同时抽取偏好、项目背景、文档状态、研究路径和未完成事项。Strategy Router 应先回答三个问题：

1. 当前 Completed Turn 是否包含值得记忆的信号？
2. 如果包含，应触发哪些类型的抽取策略？
3. 每种策略允许读取多少历史 Completed Turn？

可以采用“规则优先、模型补充”的方式：明确的元数据和文本信号由规则路由；只有规则无法判断、但内容可能具有持续价值时，才使用受限分类模型。模型返回的类别仍需经过后端允许列表校验。

抽取窗口也应按类型控制：

- 明确且稳定的偏好，单个 Completed Turn 可能已经足够；
- 文档状态需要当前 Completed Turn 加最近的相关文档 Completed Turn；
- 项目背景和研究路径可能需要多个相关 Completed Turn 或上一版摘要；
- 一旦工作空间、文档或主题明显切换，窗口必须停止扩展；
- Completed Turn 个数只是上限，相关性和 Token 预算才是实际边界。

#### 3.4 Extraction Input Package：给模型最小且可重建的输入

抽取器不应直接接收完整 transcript。Extraction Input Builder 应针对当前策略构造受控输入包，其中只包含：

```text
Extraction Input Package
+ strategy ID and immutable version
+ server-trusted identity and active work scope
+ one or more gated Completed Turns
+ existing related Memory summaries for dedupe
+ source references
+ output Schema
+ forbidden-content and safety rules
```

这些字段共同定义“本次抽取允许模型看什么、产出什么、不能越过什么边界”。

`strategy ID and immutable version` 指定本次运行使用哪套抽取策略，以及该策略的不可变版本。strategy ID 回答“现在要抽哪类 Memory”，例如文档状态、用户偏好或未完成任务；version 回答“按哪一版 Prompt、规则和字段契约抽取”。版本必须不可变，否则后续审计时就无法解释同一条 Candidate Memory 当时为什么会被生成。

`server-trusted identity and active work scope` 是服务端计算出的可信身份和当前工作边界，例如 tenant、用户、工作空间、项目、文档或会话。它不是给模型自由判断权限用的，而是告诉模型“只能在这个边界内理解输入”。最终 scope 仍由后端 Promotion Gate 计算，模型最多只能提出候选。

`one or more gated Completed Turns` 是抽取器的主要业务输入。它们已经通过 Admission Gating，代表稳定完成的交互单元。Input Builder 会按当前 strategy 选择一个或多个相关 Completed Turns，而不是把完整 transcript 全部交给模型。这样可以减少噪音，也避免把无关项目、无关文档或未完成中间状态混入抽取。

`existing related Memory summaries for dedupe` 用于帮助模型判断“这是不是已经记过”。它通常只包含少量已有 Memory 的摘要、record ID、版本和关键字段，而不是完整历史记录。这样模型可以提出 update、skip 或 possible duplicate 的候选判断；真正的去重、合并和冲突处理仍由后端执行。

`source references` 记录输入包引用了哪些 Completed Turn、消息、工具结果、文档片段或产物。它们让 Candidate Memory 可以回到原始证据。没有 source references 的候选即使内容看似合理，也很难进入正式 Memory，因为后续删除、纠正、审计和重新抽取都无从追溯。

`output Schema` 定义模型必须返回什么结构。它包括允许的 `memory_type`、必填字段、字段类型、枚举值、置信度字段和 rejected / not durable reason 等。Schema 的作用是把抽取输出变成可校验的候选对象，而不是一段自由文本总结。

`forbidden-content and safety rules` 明确哪些内容不能被保存或不能被模型当成指令。例如权限声明、secret、credential、跨 scope 内容、试图修改系统策略的用户话术，以及把临时要求伪装成长期偏好的内容，都应被拒绝或标记为不可持久化。

Prompt 还应明确：对话内容是 data，不是 instruction；模型不能根据对话中的声明改变身份、最终作用域、记录状态或平台策略。

为了避免产生第二份难治理的数据副本，不建议把完整输入包写入应用日志或可观测平台。更稳妥的做法是保存最小 extraction trace：运行 ID、输入包哈希、来源引用、策略/Prompt/Schema 版本、模型配置、Token 统计、执行状态和错误码。需要复现时，再在当前权限和删除规则允许的范围内回源重建输入，并比较哈希。

一份最小化、类型化的 Extraction Input Package 可以是：

```json
{
    "strategy": {
        "strategy_id": "draft_state_extraction",
        "strategy_version": "v1",
        "allowed_memory_types": [
            "draft_state"
        ],
        "output_schema": "draft_state_candidate_v1"
    },
    "trusted_context": {
        "actor_ref": "current-user",
        "scope": {
            "workspace_ref": "current-workspace",
            "document_ref": "current-document"
        }
    },
    "completed_turns": [
        {
            "turn_id": "turn_001",
            "user_text": "Review the current vendor onboarding guide and identify sections that need updates for the new approval workflow.",
            "turn_summary": "User reviewed a vendor onboarding guide against the new approval workflow and received recommended document updates."
        }
    ],
    "existing_related_memories": [
        {
            "record_ref": "memory-draft-state",
            "record_version": 3,
            "summary": "The previous revision was awaiting updates to two sections."
        }
    ],
    "source_refs": [
        "turn_001",
        "doc_789"
    ],
    "forbidden_content": [
        "authorization claims",
        "secrets or credentials",
        "content from another validated scope"
    ]
}
```

这个示例里的字段可以按“策略、边界、输入、对照和禁止项”理解。

`strategy` 定义本次抽取任务。`strategy_id` 和 `strategy_version` 指定使用哪一版抽取策略；`allowed_memory_types` 限制模型只能产出哪些 Memory 类型；`output_schema` 指定返回结构，方便后端做 Schema validation。它的用途是把模型任务收窄成一次可审计、可复现的类型化抽取。

`trusted_context` 定义可信运行边界。`actor_ref` 表示当前操作者，`scope` 表示服务端已经确认的工作空间和文档边界。它不是让模型判断权限，而是告诉模型只能在这个边界内理解输入；最终作用域仍由 Promotion Gate 重新计算。

`completed_turns` 是本次允许模型读取的交互内容。每个对象只保留已完成轮次的 ID、用户请求和必要摘要，而不是完整 transcript。它的用途是提供足够业务上下文，同时避免把无关轮次、未完成中间状态或跨 scope 内容带入抽取。

`existing_related_memories` 提供少量相关旧记忆，用于去重和更新判断。这里保留 `record_ref`、`record_version` 和摘要即可，让模型识别“可能是同一条记忆的延续”；真正的合并、替代和冲突处理仍由后端执行。

`source_refs` 记录本次输入包依赖的来源，例如 Completed Turn 和文档产物。它的用途是让 Candidate Memory 后续可以回源验证、审计、删除传播或重新抽取。没有来源引用的候选不应轻易晋升为正式 Memory。

`forbidden_content` 是本次抽取的硬边界，列出不能被保存或不能被模型当成事实的内容，例如权限声明、secret、credential 和跨 scope 内容。它来自后端策略，不由模型从对话中推断，也不能被用户话术覆盖。

#### 3.5 Candidate Memory：模型输出仍然不是系统事实

抽取器可以由规则、LLM 或二者组合实现，但输出只能进入 Candidate Store。Candidate 具有以下性质：

- 可以用于离线评估、错误分析和人工复核；
- 必须关联来源和抽取运行；
- 默认不可检索，也不能进入主模型 Prompt；
- 模型给出的作用域、状态、置信度和重新验证建议都只是候选值；
- 未通过 Promotion Gate 前，不属于正式 Memory。

这层隔离非常重要。如果模型输出直接进入向量索引，任何错误抽取、Prompt Injection 或临时指令都会跨请求持续生效。

Candidate 可以保留模型建议和抽取理由，但必须明确处于 `pending`：

```json
{
    "candidate_ref": "candidate-example-001",
    "extraction_run_ref": "extraction-run-example",
    "memory_type": "draft_state",
    "proposed_scope": "document",
    "candidate_text": "The current document uses the approved structure and still has two open revisions.",
    "structured_payload": {
        "revision_ref": "revision-example",
        "state": "in_progress",
        "completed_changes": [
            "preserved the approved section order"
        ],
        "open_changes": [
            "complete the implementation section",
            "verify the summary against the current document"
        ]
    },
    "source_refs": [
        "turn_001",
        "doc_789"
    ],
    "confidence_score": 0.91,
    "promotion_status": "pending"
}
```

这个示例里的字段可以按“候选标识、类型建议、内容载荷、来源证据和晋升状态”理解。

`candidate_ref` 是候选记录 ID，用于在 Candidate Store 中定位这一次模型输出。它不是正式 Memory 的稳定 ID，后续可能被 Promotion Gate 拒绝、合并到已有记录，或晋升成新的正式记录。

`extraction_run_ref` 关联本次抽取运行，用于追溯策略版本、输入包、模型配置和执行结果。这样后续发现错误抽取时，可以回到同一次运行分析原因，而不是只看到孤立的候选文本。

`memory_type` 和 `proposed_scope` 是模型给出的类型和作用域建议。它们帮助后端选择对应的校验规则，但不能直接作为最终事实；最终 Memory 类型、作用域和权限边界仍要由 Promotion Gate 按服务端可信上下文确认。

`candidate_text` 是候选记忆的可读摘要，便于人工复核、日志分析和后续生成正式记录的 canonical text。它不应被直接写入 Prompt，也不能替代结构化字段和来源验证。

`structured_payload` 保存该类型 Memory 的机器可处理字段。示例中的 `revision_ref`、`state`、`completed_changes` 和 `open_changes` 描述当前文档修订状态，用于后端做 Schema validation、去重、合并和后续确定性读取。

`source_refs` 指向产生候选的 Completed Turn 和文档来源，是候选能够被验证、审计和删除传播的证据基础。缺少有效来源时，即使 `candidate_text` 看起来合理，也应进入复核或被拒绝。

`confidence_score` 表示模型对抽取结果的置信度，只能作为 Promotion Gate 的一个输入信号。它不能绕过必需来源、作用域校验、禁止内容和去重规则。

`promotion_status` 表示候选当前还没有成为正式 Memory。`pending` 的含义是等待后端晋升决策；在此之前，它默认不可检索、不可进入主模型 Prompt，也不应触发异步索引。

#### 3.6 Promotion Gate：用确定性规则完成晋升

Promotion Gate 是 Candidate 成为正式记录之前的治理关口，建议按固定顺序执行：

1. **Schema validation**：检查必填字段、类型、受控枚举和数值范围。
2. **Source validation**：确认来源存在、仍然有效，并与抽取时的工作边界一致。
3. **Scope resolution**：根据服务端身份、当前工作空间、项目和文档关系计算最终作用域；不直接接受模型建议。
4. **Policy validation**：执行该 Memory 类型对应的最低置信度、必需来源、内容限制、保留和重新验证规则。
5. **Dedupe analysis**：计算内容指纹和逻辑去重键，查找可能对应的现有记录。
6. **Consolidation**：比较新旧结构化内容、来源、置信度、时效和冲突关系，决定如何写入。

有些检查应该是不可配置的 hard block，例如将对话中的权限声明当作真实授权、保存 secret 或 credential、以及引用跨作用域来源。它们不能通过调低阈值或人工放宽普通写入策略来绕过。

Write Policy 可以使用版本化配置表达“哪些条件可以调节”，同时把不可绕过的边界保留在代码层：

```json
{
    "policy_id": "draft_state_write_policy",
    "version": "v1",
    "applies_to": {
        "strategy_id": "draft_state_extraction",
        "memory_type": "draft_state"
    },
    "promotion_rules": {
        "minimum_confidence": 0.8,
        "required_payload_fields": [
            "revision_ref",
            "state",
            "open_changes"
        ],
        "required_source_types": [
            "completed_turn",
            "document_artifact"
        ],
        "allowed_scope_levels": [
            "document"
        ],
        "missing_source_decision": "PENDING_REVIEW"
    },
    "hard_blocks": [
        "authorization_claim",
        "secret_or_credential",
        "cross_scope_source"
    ]
}
```

这个示例里的字段可以按“策略标识、适用范围、可调规则和不可绕过边界”理解。

`policy_id` 和 `version` 标识这份写入策略及其版本。它们让每一次晋升决策都能回溯到当时使用的规则版本，方便审计、问题复现和后续策略升级。

`applies_to` 定义策略适用范围。示例中它只作用于 `draft_state_extraction` 策略产出的 `draft_state` 候选，避免一套写入规则被误用于用户偏好、研究路径或开放任务等其他 Memory 类型。

`promotion_rules` 放置可配置的晋升条件。`minimum_confidence` 是最低置信度门槛；`required_payload_fields` 要求候选必须包含关键结构化字段；`required_source_types` 要求候选必须能回到指定类型的来源；`allowed_scope_levels` 限制最终作用域层级；`missing_source_decision` 说明来源不足时应进入复核，而不是直接写入 active Memory。

`hard_blocks` 是不可通过配置放宽的硬阻断条件，例如把对话中的权限声明当作真实授权、保存 secret 或 credential、引用跨作用域来源。它的用途是把安全和治理底线固定在代码或平台策略中，不能被普通策略阈值覆盖。

晋升结果不应只有“保存/不保存”两个状态：

| 决策             | 含义                             | 典型处理                                         |
| ---------------- | -------------------------------- | ------------------------------------------------ |
| `ADD`            | 没有对应逻辑记录                 | 新建正式 Memory                                  |
| `UPDATE`         | 同一逻辑记录获得补充或更可靠来源 | 更新记录并递增版本                               |
| `SKIP`           | 内容完全重复或没有新增价值       | 不新增记录，可更新观测计数                       |
| `SUPERSEDE`      | 新状态明确替代旧状态             | 新记录生效，旧记录保留来源关系但不再参与普通检索 |
| `PENDING_REVIEW` | 内容冲突或证据不足               | 暂不进入 active Memory，等待复核                 |
| `BLOCK`          | 命中不可绕过的边界               | 拒绝写入并记录最小原因码                         |

Promotion Gate 的输出应能解释最终决策，而不只是返回布尔值：

```json
{
    "candidate_ref": "candidate-example-001",
    "decision": "UPDATE",
    "target_record_ref": "memory-draft-state",
    "resolved_scope": {
        "workspace_ref": "current-workspace",
        "document_ref": "current-document"
    },
    "write_policy_version": "draft_state_write_policy@v1",
    "reason_codes": [
        "same_logical_record",
        "newer_document_state",
        "required_sources_present"
    ],
    "resulting_record_version": 4
}
```

在去重上，需要同时保留两个概念：

- `content_hash` 判断规范化内容是否相同，主要用于任务重试和幂等写入；
- `dedupe_key` 判断是否属于同一个逻辑记忆对象，主要用于更新、合并和替代。

只用内容哈希，会把同一偏好的不同措辞保存成多条记录；只用逻辑键，又无法区分完全重复与真正补充。两者配合，才能为 Consolidation 提供稳定输入。

#### 3.7 Memory Record 与异步索引：权威状态和可发现性分离

通过 Promotion Gate 后，Candidate 才写入正式 Memory Record。主记录至少保存类型、结构化内容、服务端解析的作用域、来源、版本、状态、时效、去重信息和索引同步状态。

下面的示例展示正式记录如何把“记住什么”和“为什么可以使用”放在同一个权威对象中：

```json
{
    "record_ref": "memory-draft-state",
    "record_version": 4,
    "memory_type": "draft_state",
    "memory_subtype": "current_document_revision",
    "scope": {
        "workspace_ref": "current-workspace",
        "document_ref": "current-document"
    },
    "canonical_text": "The current document uses the approved structure and has two open revisions.",
    "structured_payload": {
        "revision_ref": "revision-example",
        "state": "in_progress",
        "open_changes": [
            "complete the implementation section",
            "verify the summary against the current document"
        ]
    },
    "source_refs": [
        "turn_001",
        "doc_789"
    ],
    "provenance": {
        "strategy": "draft_state_extraction@v1",
        "schema": "draft_state_record_v1",
        "promotion_decision": "UPDATE"
    },
    "quality": {
        "confidence": 0.91,
        "freshness": "current",
        "salience": 0.82
    },
    "content_hash": "sha256:<record-content-hash>",
    "dedupe_key": "sha256:<logical-record-key>",
    "status": "active",
    "index_sync_status": "pending"
}
```

精确、关键词和语义索引应通过 Outbox 异步派生，而不是在用户请求或主记录事务中同步调用所有外部检索服务。若主存储支持事务，Memory Record 与 Outbox 事件应在同一个本地事务中提交，避免“记录已保存但永远没有索引任务”或“索引任务已发出但主记录回滚”的双写不一致。

Outbox 任务与 Memory Record 是不同对象，因此拥有独立状态：

```json
{
    "outbox_ref": "index-job-example-001",
    "record_ref": "memory-draft-state",
    "source_record_version": 4,
    "action": "UPSERT",
    "target_indexes": [
        "keyword",
        "semantic"
    ],
    "semantic_roles": [
        "document_goal",
        "open_changes"
    ],
    "embedding_model_version": "current-approved-version",
    "status": "queued",
    "attempt_count": 0
}
```

需要区分两套状态：

- **Memory Record 状态**：记录是否 active、stale、deleted、superseded，以及当前是否已完成索引同步；
- **Outbox Task 状态**：任务处于 queued、processing、succeeded、retryable failure 或 terminal failure。

这种一致性模型允许正式记录先成功落库，再经历短暂的“尚不可通过发现检索找到”窗口。确定性读取可以直接访问主记录；发现型检索则在索引完成后可用。索引失败不会撤销已经通过治理的 Memory，而是通过重试、失败队列、redrive 或全量重建修复。

Write Path 的工程验收不应只看“是否成功写入”。还应观察 Gating 拒绝原因、Candidate 产生率、错误晋升率、各类 Consolidation 决策分布、Outbox 积压、索引延迟、重试次数和无法回源的记录数。

### 4. 读取侧：不是找相似文本，而是安全组装本轮上下文

Retrieval Context Assembly 解决的不是“怎样从向量库找相似文本”，而是“怎样把已经保存的 Memory 在下一次请求中安全、准确且有限地带回”。它是读取侧的运行时控制面：先判断本轮是否需要 Memory，再选择确定性读取或发现型检索，最后经过回源、二次策略校验、预算压缩和 Memory Card 组装后交给主模型。

一条推荐的读取链路如下：

```text
READ-SIDE REQUEST PATH

[Current Request + Server-Trusted Context]
-> [Retrieval Planner + Versioned Retrieval Policy]
-> [Path A: Known-Scope Lookup] + [Path B: Hybrid Discovery]
-> [Source-of-Truth Fetch]
-> [Second-Pass Policy and Freshness Check]
-> [Rerank + Category Budget + Compression]
-> [Bounded Memory Cards]
-> [Main LLM]

Side output: [Minimal Retrieval Trace]
```

#### 4.1 Retrieval Planner：先决定本轮是否需要 Memory

不是每个请求都需要检索历史。独立问答、与既有工作无关的新任务，或当前输入已经提供完整上下文时，可以直接跳过 Memory Retrieval。只有出现以下信号时，Planner 才生成检索计划：

- 用户明确要求继续之前的工作；
- 请求绑定现有会话、项目或文档，并依赖已有状态；
- 当前任务需要稳定偏好、未完成事项或历史探索路径；
- 当前输入包含“刚才那版”“上次讨论的方案”等近端指代。

Planner 通常先用确定性信号识别任务类别，例如“继续当前文档”“恢复未完成事项”或“参考既有研究”。只有规则无法区分时，才使用模型做受限分类；模型只能从允许列表中选择，不能创建新类别、扩大作用域或开放额外 Memory 来源。

这里需要区分 **Retrieval Policy** 和 **Retrieval Plan**：

- Retrieval Policy 是版本化的稳定规则，定义某类任务允许读取哪些 Memory 类型、必须执行哪些检查，以及可以用什么方式进入 Prompt；
- Retrieval Plan 是单次请求的临时对象，结合当前可信作用域、任务类别、查询文本和 Token 预算，说明本轮实际执行哪些路径。

一个计划通常包括：任务类别、服务端验证后的作用域、Policy 版本、Path A/Path B 的数据源、是否需要触发 revalidation、分类预算和降级行为。它不应由前端直接构造，也不需要作为业务数据长期保存。

下面的 Retrieval Plan 只描述本轮怎样查，不保存实际命中正文：

```json
{
    "task_class": "continue_current_document",
    "validated_scope": {
        "workspace_ref": "current-workspace",
        "document_ref": "current-document",
        "session_ref": "current-session"
    },
    "retrieval_policy_version": "document_continuity_policy@v1",
    "paths": {
        "known_scope_lookup": [
            "active_session_summary",
            "recent_completed_turns",
            "current_draft_state",
            "open_work"
        ],
        "hybrid_discovery": [
            "related_project_context",
            "research_trail"
        ]
    },
    "revalidation": {
        "required_if_selected": [
            "research_trail"
        ],
        "trigger": "used_for_current_factual_judgment"
    },
    "memory_budget": {
        "total_tokens": 1600,
        "session_continuity": 450,
        "preferences": 200,
        "workspace_and_draft": 550,
        "research_trail": 400
    }
}
```

这个示例里的字段可以按“任务类别、可信边界、策略版本、检索路径、重新验证和上下文预算”理解。

`task_class` 描述本轮请求属于哪类连续性任务，例如继续当前文档、恢复未完成事项或参考历史研究。它用于选择 Retrieval Policy 和默认检索路径，不应由模型自由创造新类别。

`validated_scope` 是服务端确认后的读取边界。示例中同时包含工作空间、文档和会话，表示本轮只能在这些边界内读取 Memory；缺少必要边界时，应缩小范围或跳过对应 Memory 来源。

`retrieval_policy_version` 指定本轮使用哪一版读取策略。它让后续 trace 可以解释为什么某类 Memory 被允许、丢弃、要求重新验证或限制进入 Prompt 的用途。

`paths` 定义本轮实际执行的两类检索路径。`known_scope_lookup` 读取已经知道逻辑对象的状态，例如当前会话摘要、最近轮次、当前文档状态和开放任务；`hybrid_discovery` 只在允许边界内发现可能相关的项目背景或研究路径。

`revalidation` 描述哪些候选在被选中且用于当前事实判断时需要重新验证。它把“记录本身有刷新要求”和“本轮是否真的触发刷新”分开，避免每次读取都调用外部工具。

`memory_budget` 是本轮 Memory 上下文的 Token 预算。它按用途分配额度，防止历史 Memory 挤占当前请求、实时工具结果和主模型输出空间。

#### 4.2 Path A：Known-Scope Lookup

当系统已经知道要读取哪个逻辑对象时，应该使用确定性查询，而不是相似度检索。典型对象包括：

- 当前用户与当前任务相关的稳定偏好；
- 当前会话的 active summary；
- 当前会话最近若干个 Completed Turns，用于解析近端指代；
- 当前文档的 draft state 和版本；
- 当前工作空间中状态为 open 的明确后续事项。

Known-Scope Lookup 的关键不是“返回多少条”，而是使用稳定 ID、已验证作用域和状态条件，明确加载或排除对象。例如“最近几轮”应先按会话限定范围，再按轮次顺序读取；不能依赖随机生成的 ID 推断先后顺序。

Path A 即使直接读取主记录，也仍需执行后面的 second-pass policy check。确定性定位只解决“找到谁”，不自动证明“当前仍可使用”。

#### 4.3 Path B：Exact、Keyword 与 Semantic Discovery

当系统知道允许搜索的边界，但不知道具体是哪条 Memory 相关时，才使用发现型检索。

1. **Exact lookup**：当前请求包含已验证的稳定标识或结构化键时执行。精确命中应保留为明确候选，不因语义分数较低而被替换。
2. **Keyword search**：请求包含明确术语、名称、状态或任务词时执行。它适合找“确实提到同一对象”的记录。
3. **Semantic search**：请求表达概念、意图、事实模式或待完成事项，且措辞可能与历史记录不同，才使用向量检索。

三种方式可以并行执行。结果按稳定 Record ID 合并，保留每种检索方式的命中原因，再交给回源与后续排序。Write Path 的去重减少重复记录；Retrieval 的去重则合并同一记录被多种检索方式命中的证据。

#### Semantic role 与 Named Vector

一条结构化 Memory 可能包含多种语义角色。例如研究路径同时包含“当时的问题”“阶段性发现”“来源上下文”和“未确认事项”。如果把所有字段拼成一个 embedding，查询“以前研究过什么”和查询“还有什么没确认”会共享同一向量表达，召回意图变得模糊。

更稳妥的方式是按 semantic role 生成 named vectors：

| Semantic role     | 典型输入                   | 适合回答的问题             |
| ----------------- | -------------------------- | -------------------------- |
| `topic`           | 问题、主题、分类和关键实体 | 是否做过相似主题的工作？   |
| `working_finding` | 阶段性发现和关键因素       | 之前形成过哪些工作性判断？ |
| `source_context`  | 来源名称、类别、用途和摘要 | 之前参考过哪些来源？       |
| `caveat`          | 限制、待确认项和刷新要求   | 哪些内容仍需核验或补充？   |

如果底层引擎不支持 named vectors，可以使用独立的 vector collections 或等价的独立物理索引。不要把差异很大的语义角色全部写入同一个近邻图后，只依赖 metadata filter 区分。在许多 ANN/HNSW 实现中，查询过滤并不等价于在角色专属图中搜索；其他角色的向量仍可能影响搜索路径和候选预算，导致分类内召回不稳定。

#### 4.4 Filter Before Ranking：作用域不是相关性特征

所有发现型检索都必须先根据服务端可信上下文构造 allowed-scope predicate，再在允许集合内执行 Exact、Keyword 或 Semantic Search。

不能先全局 Top-K，再对结果做权限过滤，原因有两个：

1. **边界风险**：不应访问的候选已经进入检索中间链路或诊断数据。
2. **候选饥饿**：范围外的高相似记录可能消耗 Top-K 预算，使真正相关的范围内记录根本没有机会进入结果集。

如果请求缺少某类 Memory 必需的作用域，应缩小检索范围或跳过该来源，而不是退回更宽的全局搜索。Scope 是 hard filter，similarity、freshness 和 salience 才是排序特征。

索引层过滤仍然只是第一道检查。索引是异步派生视图，可能暂时保留旧作用域、旧状态或已删除指针，因此不能替代回源。

#### 4.5 Source-of-Truth Fetch：索引命中只是指针

无论命中来自 Exact、Keyword 还是 Semantic Index，都只应返回稳定 Record ID、索引版本、命中角色和必要分数。系统随后批量回到权威存储读取最新记录。

回源至少需要两类字段：

- **治理字段**：最新作用域、记录版本、active/deleted/stale/superseded 状态、过期时间和当前授权结果；
- **使用字段**：本轮需要的结构化内容、来源引用、freshness 标记和允许的 Prompt 用途。

如果主记录不存在、已删除或无法在当前边界内读取，必须丢弃该命中，并投递幂等索引清理任务。系统不能退回使用索引中的旧正文，因为那会让派生视图反过来成为事实来源。

Known-Scope Lookup 如果已经直接读取权威记录，不需要再次做“回源查询”，但仍进入同一套二次策略过滤，保证 Path A 与 Path B 使用一致的最终判断标准。

#### 4.6 Second-Pass Policy Filter：用最新状态做最终准入

回源后，后端根据当前 Retrieval Policy 依次检查：

1. 作用域和当前授权是否仍匹配；
2. 记录是否 active，来源是否有效，是否已删除、过期或被替代；
3. 当前任务是否允许使用该 Memory 类型；
4. 该记录本轮允许扮演什么角色，例如 writing style、workspace background、draft state、open work 或 research trail；
5. 是否需要在进入 Prompt 前执行时效刷新或外部来源重新验证；
6. 最终只投影本轮用途所需字段，不把完整主记录继续向后传递。

输出可以收敛为三个简单状态：

| 状态                    | 含义                                         |
| ----------------------- | -------------------------------------------- |
| `ALLOW`                 | 当前状态和用途均满足 Policy，可以进入重排    |
| `HOLD_FOR_REVALIDATION` | 需要先完成重新验证，再决定是否允许           |
| `DROP`                  | 作用域、状态、来源或用途不满足要求，直接排除 |

这些判断由后端执行，不交给主模型自行解释。模型可以遵守 Card 中的使用说明，但不能承担授权和状态校验职责。

#### 4.7 Revalidation：持久化要求与运行时触发分开

对时效敏感的研究或分析记录，可以持久化 `requires_revalidation` 一类治理标记。这个标记表示“不能绕过当前来源直接形成新判断”，但不意味着每次读取都必须调用外部工具。

运行时只有同时满足两个条件才触发重新验证：

1. 该记录被本轮选中；
2. 当前任务会依赖其中的工作性发现形成新的事实判断或决策。

如果用户只是继续编辑格式、恢复文档状态或查看未完成事项，可以不加载旧发现，也无需触发刷新。该次读取不会把持久化标记改为 false；未来真正依赖内容时仍需重新验证。

重新验证有三种主要结果：

- 当前来源支持历史发现：可以作为 continuity context 使用，但仍保留来源和观察时间；
- 当前来源与历史发现冲突：以当前来源为准，旧记录标记 stale、被替代或进入复核；
- 来源不可用或证据不足：不把旧 finding 带入当前判断，只保留主题、来源指针或待确认事项。

这套机制同样适用于会变化的产品文档、运行手册、价格、配置能力和外部知识，不依赖特定行业。

#### 4.8 Rerank、Token Budget 与压缩

通过二次过滤和必要刷新后，候选才进入排序与预算分配。排序不应只看向量相似度，还可以综合：

```text
ranking signals
+ exact and keyword evidence
+ semantic similarity by selected role
+ task-intent match
+ scope specificity
+ record freshness and confidence
+ salience and prior successful use
- stale, conflict, and redundancy penalties
```

明确绑定当前文档的状态，即使语义分数一般，也可能比泛化的项目背景更有用。Exact hit 可以作为保留候选，但仍受状态和预算约束。

Memory Budget 最好按用途分配，例如 user preference、session continuity、workspace and draft、research trail。分类预算可以是软上限，允许复用未使用额度，但总 Memory 上下文不能超过硬上限。当前请求、实时工具结果和主模型输出空间应独立预留，不能让历史 Memory 挤占全部 Context Window。

压缩也应在类型边界内进行：Preference 保留明确规则，Draft State 保留版本和未完成修改，Research Trail 保留主题、来源指针和 caveat。不要用一个通用摘要器把不同类型重新混成无结构文本。

#### 4.9 Prompt Memory Cards：让模型看见用途和边界

最终进入主模型的不是 raw record，而是经过最小化投影的 Memory Card：

```text
[Memory type | source=<id> | record_version=<version> | scope=<scope> | usage=<usage>]
Content: <仅保留本轮需要的字段>
Boundary: <允许如何使用，以及不能推断什么>
```

一个 Prompt 可以分成清晰的信任区域：

```text
[TRUSTED RUNTIME INSTRUCTIONS]
由后端从当前 Policy 派生的少量使用规则

[CURRENT REQUEST]
用户本轮真正提出的任务

[CURRENT TOOL OR KNOWLEDGE CONTEXT]
本轮实时检索或工具结果

[MEMORY CONTINUITY CONTEXT]
经过治理的 Preference / Session / Draft / Workflow / Research Cards
```

Memory Card 是不可信的 continuity data，不是 system instruction。即使 Card 内容包含命令式文本，也不能授予权限、修改平台策略或覆盖当前工具结果。

#### 4.10 Minimal Retrieval Trace：可解释但不复制正文

每次实际使用 Memory 时，应写入最小 retrieval trace，记录请求 ID、有效作用域、任务类别、Policy 版本、实际执行路径、候选数量汇总、最终选中的 Record ID/版本/用途，以及 revalidation 状态和错误阶段。

```json
{
    "request_ref": "request-example-001",
    "status": "completed",
    "task_class": "continue_current_document",
    "retrieval_policy_version": "document_continuity_policy@v1",
    "executed_paths": [
        "known_scope_lookup",
        "hybrid_discovery"
    ],
    "candidate_summary": {
        "found": 8,
        "dropped_by_policy": 3,
        "excluded_by_budget": 2,
        "selected": 3
    },
    "selected_contexts": [
        {
            "record_ref": "memory-draft-state",
            "record_version": 4,
            "usage": "draft_state"
        },
        {
            "record_ref": "memory-user-preference",
            "record_version": 2,
            "usage": "writing_style"
        },
        {
            "record_ref": "session-summary-current",
            "record_version": 5,
            "usage": "session_continuity"
        }
    ],
    "revalidation": {
        "required": false,
        "status": "not_needed"
    }
}
```

这个 trace 示例里的字段可以按“请求标识、执行路径、候选收敛、最终上下文和重新验证状态”理解。

`request_ref`、`status`、`task_class` 和 `retrieval_policy_version` 记录本次读取运行的基本身份和规则版本，用于审计和问题复现。

`executed_paths` 表示本轮实际执行了哪些读取路径，方便区分上下文来自确定性读取、发现型检索，还是二者组合。

`candidate_summary` 只记录候选数量如何被收敛，例如找到多少、因策略丢弃多少、因预算排除多少、最终选择多少。它用于观察检索质量和策略效果，但不复制候选正文。

`selected_contexts` 只保存最终进入 Memory Cards 的 Record ID、版本和用途。这样可以回溯主模型当时受哪些 Memory 影响，同时避免 trace 变成另一份完整 Memory 副本。

`revalidation` 记录本轮是否需要刷新、刷新状态和可能的失败阶段。它帮助排查 stale Memory 是否被正确拦截，也能支持后续重试或人工复核。

标准 trace 不保存未选中候选正文、完整 Memory Cards 或完整 Prompt。候选级分数、排除原因和 Prompt snapshot 只在受控诊断场景下进入隔离存储，并使用更短的保留期，避免审计系统成为另一份敏感数据副本。

Retrieval 的验收指标除相关性外，还应覆盖：Path A 命中正确率、混合检索贡献、回源缺失率、二次过滤丢弃原因、stale Memory 使用率、revalidation 成功率、每类预算利用率、Prompt 有效 Token 比例和索引清理延迟。

### 5. 存储与索引：权威状态和发现效率要分开

这一节回答的是 Memory 落库后如何既保持权威状态正确，又让记录可以被高效发现。权威存储需要支持事务、记录版本、生命周期状态和来源反查；向量数据库或搜索引擎只负责发现。把两者分开会引入最终一致性，但换来了更清晰的故障边界：索引可以失败、延迟或重建，主记录仍保持正确。

索引载荷应保持最小，只包含稳定 Record ID、必要作用域过滤字段、Embedding、来源记录版本和索引版本。完整正文、原始对话和源文档继续留在权威存储中，避免搜索系统成为第二套难治理的数据副本。

每个派生索引都应独立版本化。例如分词规则、Embedding 模型、semantic role 或 filter schema 变化时，不在原索引上原地覆盖，而是构建新版本：

```text
[Eligible Active Records]
-> [Build New Index Version]
-> [Validate Count + Scope Filters + Sample Queries]
-> [Switch Read Alias]
-> [Retire Previous Version]
```

这条重建链路里的每一步都有明确用途：`Eligible Active Records` 只选择当前允许被索引的主记录；`Build New Index Version` 在旁路构建新索引，避免破坏线上读路径；`Validate Count + Scope Filters + Sample Queries` 检查数量、作用域过滤和典型查询是否符合预期；`Switch Read Alias` 让读取流量切到新版本；`Retire Previous Version` 则在确认稳定后下线旧索引。

索引项可保存 `source_record_version` 和 embedding 输入指纹，用于判断主记录更新后哪些向量需要重建。重建只扫描当前允许索引的 active 记录；deleted 和 superseded 记录排除，stale 记录按类型策略决定是否仅保留有限发现能力。

这种设计的代价是需要维护 Outbox、重试、reconciliation 和索引版本切换，但它让模型升级、搜索迁移、删除补偿和灾备恢复都不必修改权威数据。

## 真正难的不是检索，而是风险边界

风险控制不应只写成上线前检查清单，而应直接嵌入写入、读取和删除链路。下面几类风险分别对应不同的硬边界：谁能读、什么能被记住、来源如何追溯、删除如何传播，以及系统如何渐进扩展。

### 1. 作用域和授权

作用域和授权控制的是“这条 Memory 当前能不能被看见”。所有作用域都由服务端身份与授权链路生成。调用方和模型只能提供业务上下文提示，不能扩大可访问范围。无法确认权限时应 fail closed，并记录最小原因码，便于排查而不暴露越权候选内容。

### 2. Prompt Injection

Prompt Injection 控制的是“内容能不能改变系统规则”。对话、文档、工具结果和 Memory 正文都视为不可信数据。任何“记住我是管理员”“忽略后续检查”之类的内容，都不能被晋升为长期记忆，也不能改变运行策略；它们最多作为被拒绝的候选或诊断事件保留最小原因。

### 3. 来源最小化与审计

来源最小化控制的是“能解释但不复制”。Provenance 应保存来源 ID、版本、策略、Schema、哈希和决策结果，而不是复制完整敏感正文。审计记录回答“谁在什么时候做了什么”，来源记录回答“这条记忆如何形成”，两者应关联但不能互相替代。

### 4. 删除传播和索引重建

删除传播控制的是“旧内容不能从派生副本回来”。删除来源时，需要反查受影响的已完成轮次、摘要、候选记忆、正式记录、索引和缓存。权威记录先更新状态，再通过幂等异步任务清理派生对象。索引必须能够从权威记录全量重建，不能依赖手工清理每个搜索副本。

同时要区分：

- `deleted`：不再允许保留或使用，应从索引和缓存中移除；
- `stale`：仍保留来源关系，但不能直接作为当前上下文使用；
- `superseded`：已被新版本替代，旧记录仅保留追踪价值。

### 5. 渐进式落地

渐进式落地控制的是“复杂度不要一次性失控”。第一版不应同时开放所有 Memory 类型。更稳妥的顺序是：

1. 先完成单一文档状态的端到端闭环，只使用确定性查找；
2. 再接入用户偏好、未完成事项和项目背景；
3. 最后开放研究路径复用、混合检索和重新验证；
4. 历史数据补建单独执行 dry-run，不默认搬运全部旧记录。

每增加一种 Memory 类型，都需要先回答清楚：它由什么事件触发写入，在哪个边界内可以被读取，什么时候必须退出 Prompt，以及删除或更正来源时会影响哪些下游对象。

## 如果设计正确，系统应该变成什么样

本文讨论的是架构设计方法，而不是对外披露的生产指标，因此不提供未经验证的性能数字。这里的结果应理解为可验证的工程目标，而不是性能承诺。按照上述设计，预期可以获得以下结果：

- 跨会话继续工作时，能够恢复真正需要的文档状态、用户偏好和未完成事项；
- 临时指令、项目事实和历史研究不会因为文本相似而被错误混用；
- 每条正式 Memory 都可以解释来源、保存原因、适用边界和当前状态；
- 主记录发生删除、纠正或过期后，摘要、索引和缓存能够同步失效或重建，不会继续把旧内容带回回答；
- Prompt 中的历史内容更少但更相关，为当前资料和模型输出保留更多 Token 预算。

真正的效果仍需要通过离线数据集、影子流量和分阶段上线验证。验收时不只看回答是否更相关，还要持续观察错误晋升、跨作用域召回、过期信息复用、回源失败和删除传播延迟。

## 几个工程教训

如果把上面的设计压缩成工程取舍，我会先看这几条。它们不一定是最炫的部分，但通常决定一个 AI Memory 系统能不能长期安全运行。

1. **Memory 不是聊天记录的别名。** 它应是经过提炼、治理和作用域约束的工作状态。
2. **类型决定规则，生命周期只是其中一个字段。** 不同类型需要不同的 Schema、检索用途和风险边界。
3. **模型适合抽取，不适合授予权限或决定持久化。** 治理决策必须由后端执行。
4. **Scope before ranking。** 权限先于相关性，是架构底线。
5. **Index is not truth。** 所有索引命中都需要回源和二次校验。
6. **历史研究是导航，不是当前事实。** 对时效敏感的内容必须重新验证。
7. **删除和重建不是上线后的补丁。** 它们应从第一版进入数据模型和异步任务设计。
8. **先做窄而完整的闭环。** 单一类型、单一场景、可观测、可删除，比一次性建设“全能记忆平台”更容易成功。

## 最后回到一句话

企业级 AI Memory 的价值，不在于让模型保存更多历史，而在于把连续工作中真正有价值的状态，转化为可解释、可控制、可删除的系统对象。

一个可靠的 Memory Layer 需要形成同一条闭环：写入时，把模型提出的候选记忆变成有类型、有来源、有边界的主记录；读取时，先按可信作用域过滤，再回到主记录确认状态和时效；主记录变化时，把摘要、索引和缓存一起更新。

这条闭环的核心不是“让模型记住更多”，而是让每一次记住和每一次使用都能被解释、限制和撤回。只有这样，AI 助手才能在保持连续性的同时，不牺牲安全边界、数据治理和工程可维护性。

## 参考资料与致谢

1. Oracle Developers Blog, [*From RAG to Memory Systems: Building Stateful AI Architecture*](https://blogs.oracle.com/developers/from-rag-to-memory-systems-building-stateful-ai-architecture), June 4, 2026. 该文提出了 memory manager、typed memory、promotion gate、per-turn prompt reassembly、trace/replay 和 filter-before-ranking 等通用架构模式。本文借鉴其系统设计思想，并在此基础上结合企业级 AI 助手、知识工作连续性、作用域治理、来源追踪、重新验证和删除传播等问题进行了整理和扩展。
2. 本文在作者指导下，借助 GPT 5.5 和 GPT 5.6 协作完成；两者均参与了内容起草、结构整理和后续 refinement，最终观点、架构取舍、示例和公开表述均由作者审阅、编辑和定稿。