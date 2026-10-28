# 智能体记忆与状态

记忆是让智能体能够学习并随时间保持上下文的能力。智能体记忆已从"聊天记录"发展为**多层认知架构**，包含四个命名层级（工作记忆、情景记忆、语义记忆、程序记忆），每个层级都有自己的写入模式、延迟预算和故障模式。生产系统（Mem0、Letta、Anthropic Memory Tool + Skills、Zep/Graphiti、LangMem）现在将记忆选择视为一流架构决策。2026 年值得关注的转变：其中多个系统趋同于**基于文件的记忆**，智能体通过路径导航（Letta 的 MemFS、Claude 的记忆工具、编码智能体指令文件），而市场领先的库则放弃了其图存储。

塑造本章的 2026 年研究浪潮：A-MEM（NeurIPS 2025）、HippoRAG（多跳图检索）、多层记忆架构、HaluMem（操作级记忆幻觉基准）、MINJA / MemoryGraft（仅查询记忆投毒攻击）以及 TTT-E2E（横跨 Stanford、Berkeley、UCSD、NVIDIA 和 Astera 的多实验室合作），一种通过测试时训练将上下文压缩为权重的技术。

## 目录

- [记忆层级](#the-memory-hierarchy)
- [短期：推理轨迹](#short-term-the-reasoning-trace)
- [情景记忆：过去的经历](#episodic-memory-past-experiences)
- [语义记忆：人格](#semantic-memory-the-persona)
- [程序记忆：习得的技能和工作流](#procedural-memory-learned-skills-and-workflows)
- [权衡：事实 X 放在哪里？](#tradeoffs-where-does-fact-x-go)
- [生产实现](#production-implementations)
- [故障模式与缓解措施](#failure-modes-and-mitigations)
- [Mem0 与智能体个性化](#mem0-and-agentic-personalization)
- [面试问题](#interview-questions)
- [参考文献](#references)

---

## 记忆层级

智能体使用分层存储方法：

| 层级 | 类型 | 技术 | 用途 |
|------|------|------------|---------|
| **L1** | 工作记忆 | 上下文窗口 / KV 缓存 | 当前任务步骤、局部变量 |
| **L2** | 情景记忆 | 向量数据库 / 图 | "我上次做了什么？" |
| **L3** | 语义记忆 | SQL / 知识图谱 | 用户偏好、"真相" |
| **L4** | 程序记忆 | 技能注册表 / 工具策略 / 工作流图 | "如何执行此任务？" |

### 每层级的实际属性

层级之间的差异不仅在于用途。读取模式、写入模式、延迟预算和新鲜度期望都会推动不同的存储技术：

| 维度 | L1 工作记忆 | L2 情景记忆 | L3 语义记忆 | L4 程序记忆 |
|---|---|---|---|---|
| **存储内容** | 活跃轮次、工具输出、草稿本、系统提示 | 过去会话、轨迹、带时间戳的观察 | 提炼的事实、偏好、实体关系 | 技能、手册、系统提示指令、代码/工具序列 |
| **读取模式** | 每个令牌、每轮（在注意力中） | 基于相似度 + 新近度 + 重要性的 Top-k | 实体/主题提及触发查找 | 匹配任务签名时加载 |
| **写入模式** | 推理引擎持续追加；KV 缓存变更 | 仅追加日志；在轮次边界提交 | 提取、去重、插入；写入时冲突解决 | 成功/失败后反射性写入；显式人工或自我编辑 |
| **延迟预算** | <50ms（驻留在 GPU HBM 中） | 100-300ms（向量 ANN + 重排序） | 200-800ms（图遍历 + LLM 提取） | 50-500ms（文件读取或小索引查找） |
| **新鲜度期望** | 令牌级新鲜；会话结束丢失 | 数小时到数月；容忍陈旧 | 应反映*当前*状态；陈旧是缺陷 | 变化缓慢；更新是刻意的 |
| **存储技术** | HBM 中的 KV 缓存（vLLM PagedAttention 块） | 向量数据库（Pinecone、Weaviate、Qdrant）、仅追加日志 | 知识图谱（Neo4j、Graphiti）、KV 存储、双时态关系行 | 文件系统（Claude `/memories/`、Skills 作为 `SKILL.md`）、提示注册表、微调 LoRA |
| **查询语义** | 位置 + 注意力 | 相似度 + 新近度 + 重要性（Park 等加权） | 实体-关系匹配、结构化查询、双时态过滤 | 任务签名匹配，通常是文件名或标签查找 |
| **驱逐** | 滑动窗口、KV 块哈希 LRU | 衰减评分、合并到 L3、归档到冷存储 | 通过关闭旧事实的有效性（`invalid_at`）来取代；GDPR 显式删除 | 手动弃用、与新技能 A/B 测试、版本固定 |

**这在实践中意味着什么：** 当事实到来时，架构问题不是"我们应该记住它吗？"而是"*在哪个层级*，以什么新鲜度契约，以及什么驱逐规则？" 选择错误的层级会产生可预测的故障模式（会话偏好提升到 L3 会跨会话泄露；稳定的用户事实留在 L2 会在两周后被驱逐）。参见下方的[权衡：事实 X 放在哪里？](#tradeoffs-where-does-fact-x-go)。

---

## 短期：推理轨迹

生产智能体不再仅存储"消息"；它们存储**状态对象**。
- **草稿本**：提示中专门供智能体向自己"写笔记"的部分，不展示给用户。
- **KV 缓存分片**：对于长期运行的智能体，我们使用**前缀缓存**来保持"系统指令"和"标准工具"在 GPU 内存中温热，仅交换动态任务状态。
- **卸载大结果**：工具输出是淹没工作记忆的最快方式。Agno 3.0（2026 年 8 月）将超过 16,000 字符的工具结果写入文件存储，并在上下文中替换为信封加上 `read_result` 和 `search_result` 工具，因此智能体只拉回它需要的内容。
- **笔记而非单一摘要**：随 GPT-6 Astra 发布的实验性 Codex 功能在跨上下文窗口中保持可搜索笔记，而不是将历史压缩为单一摘要。它是供应商提供的摘要压缩替代方案，用于长期工作：不是单一有损摘要，智能体保持可查询的索引。

---

## 情景记忆：过去的经历

情景记忆存储"运行"或"轨迹"。
- 如果智能体上周二未能抓取网站，情景记忆应阻止它今天再次尝试相同的失败选择器。
- **模式**：任务完成时，总结"经验教训"并存入向量数据库。开始新任务时，执行**自我搜索**以查找类似的先前任务。

---

## 语义记忆：人格

语义记忆存储关于用户或环境的"事实"。
- *"用户偏好 JSON 输出。"*
- *"生产数据库在凌晨 3 点到 4 点之间离线。"*

**最佳实践**：对语义记忆使用**知识图谱**。与向量搜索（模糊的）不同，图提供实体和关系的确定性检索（例如，`User` -- `OWNER_OF` --> `Project_A`）。

将图数据库视为需要论证的选择，而非默认。Mem0 v2 移除了其 Neo4j、Memgraph、Kuzu 和 Apache AGE 驱动程序，现在将实体链接保存在现有向量存储内的并行集合中，这涵盖了大多数个性化查找，而无需运行第二个数据库。当多跳遍历或时序问题（"3 月 12 日什么是真的？"）是产品时，才使用专用图，如 Graphiti 的双时态边。

---

## 程序记忆：习得的技能和工作流

程序记忆存储如何做事。情景记忆回答"以前发生了什么？"，语义记忆回答"什么是真的？"，程序记忆回答：

"完成这类任务的正确流程是什么？"

这一层捕获可重用的技能、工具使用模式、操作规程和工作流偏好。

示例：

* "生成周报时，首先从 Snowflake 提取指标，然后对照仪表板验证，最后总结异常。"
* "回应客户投诉时，分类紧急度，检索政策，起草回复，如果置信度低则升级。"
* "编写 SQL 时，始终先检查模式，生成查询，运行验证，并解释假设。"

程序记忆对智能体系统尤为重要，因为许多任务不仅仅是记住事实。它们需要遵循正确的行动序列。

---

## 权衡：事实 X 放在哪里？

一阶决策不是"哪个层级"而是*"谁为错误付出代价？"* L2 中的检索失败影响一轮。L3 中的错误事实在纠正前影响*每一*轮。L4 中的中毒技能传播到每次未来调用。

### 层级选择表

| 事实 / 关注点 | 层级 | 推理 |
|---|---|---|
| "用户的 API 速率限制是 1000 req/min" | L3，带双时态有效性（`valid_at` / `invalid_at`） | 租户范围的事实；可按实体查询；必须支持取代。不是 L4：它是数据，不是程序。 |
| "部署我们服务的步骤" | L4，作为版本化技能 | 多步骤配方，带有条件分支。技能可组合；语义三元组不可。 |
| "智能体上次执行此任务的失败尝试" | L2 原始，*然后*如果可泛化，将教训反射到 L4 | 原始轨迹属于情景；泛化的教训（"永远不要在高峰时段运行迁移"）值得以 Reflexion 风格写入 L4。 |
| "用户偏好简洁回复" | L3 | 稳定偏好，可按 `user_id` 查询，单一三元组。 |
| "用户在此会话中要求简洁回复" | 仅 L1 | 会话范围；不要用潜在短暂的偏好污染 L3。 |
| 当前天气、今日股价 | 无：调用工具 | 快速变化的事实，其真实来源在其他地方，绝不应进入记忆。 |
| "Project Phoenix 有团队成员 A、B、C" | L3，作为图片段 | 多跳遍历价值；Graphiti 或 Neo4j 风格存储。 |

### 成本权衡

- **L1 主导*延迟成本***：TTFT 随上下文大小扩展；更长的工作记忆意味着更慢的首个令牌。KV 缓存压力将高端加速器内存推向饱和。
- **L2 在大规模时主导*存储成本***：仅追加日志随使用线性增长。[第 30 天问题](https://cipherbuilds.ai/blog/day-30-agent-memory-problem)描述了未修剪的情景存储如何在一个月后侵蚀智能体质量。
- **L3 主导*写入放大成本***：每轮都可能触发提取、去重、冲突解决。Mem0 的设计明确用写入时工作换取检索速度。
- **L4 主导*治理成本***：坏技能传播到每次未来调用。Anthropic 的 [Dreams](https://www.mindstudio.ai/blog/what-is-claude-dreaming-anthropic-agent-memory) 研究预览通过将整合输出写入单独的记忆存储而非编辑实时存储来承认这一点，这为晋升前的审查留出了空间。

### 晋升规则

有趣的设计问题是*L2 情景何时晋升到 L3 或 L4*。可辩护的规则是基于阈值的，而非隐式衰减：

1. **N 次独立观察**到同一模式（N=3 到 5 是典型值）。
2. **按来源置信度加权**：用户陈述 > 工具输出 > 模型推断。
3. **整合步骤中的人工或 LLM 评审**（不是每轮）。
4. **计划的批量整合**，而非同步每轮写入（避免写入放大）。
5. **双向**：L3 中的语义事实可以重新实例化为特定任务的情景上下文。记忆不是单向的。

### 何时整理：写入时、睡眠时或读取时

整理时机现在是一个命名的设计轴，每个点都有实际系统：

| 时机 | 示例 | 成本特征 | 主要风险 |
|---|---|---|---|
| **写入时** | Mem0 风格的每次 `add()` 提取 | 每次写入一次 LLM 调用；读取便宜 | 提取错误被固化并传播（HaluMem 发现） |
| **睡眠时** | Anthropic Dreams（研究预览）：读取记忆存储加上过去会话记录，写入重新组织的存储，合并重复项并替换陈旧条目；Mem0 的付费 Dream 功能 | 离线批处理，不在请求路径上 | 运行之间的陈旧；需要晋升审查 |
| **读取时** | Just-in-Time Memory（arXiv 2609.27334，2026 年 9 月）：保持原始轨迹，读取时为当前任务整理 | 整理成本和延迟移到读取路径 | 原始存储增长；保留和隐私成本上升 |

值得引用的研究结果：Just-in-Time Memory 在 ALFWorld、WebShop 和 tau2-bench 上比最强的写入时记忆基线高出 16.2、16.3 和 3.9 个成功率百分点，即使未经训练的整理者也具有竞争力。当相关性取决于尚未看到的任务时，读取时整理获胜；当读取热且延迟受限时，写入时提取获胜。

---

## 生产实现

命名系统的差异与其说是"它们存储什么"，不如说是*写入纪律*、*检索算法*和*治理姿态*。

| 系统 | 最佳场景 | 优势 | 不足 |
|---|---|---|---|
| **[Mem0](https://github.com/mem0ai/mem0)** | 大规模跨会话个性化 | 自 v2.0.0（2026 年 4 月 16 日）：单遍、仅 ADD 提取（每次 `add()` 一次 LLM 调用，哈希去重，无更新或删除）和融合语义相似度、BM25 和实体提升的混合检索。实体链接保存在现有向量存储内的并行集合中，使 Mem0 能够删除其 Neo4j、Memgraph、Kuzu 和 Apache AGE 图驱动程序。重新设计后在 LoCoMo 上报告 92.5，在 LongMemEval 上报告 94.4（[基准测试](https://mem0.ai/blog/ai-memory-benchmarks-in-2026)）；其 2025 年论文报告在 LLM 评审指标上相对 OpenAI 内置记忆提升 26%（均为供应商报告）。SDK v2.2.1（9 月 25 日）。 | 每条记忆 8K 字符上限（不适合文档）；云优先姿态造成数据主权摩擦；开源提取仅追加，因此合并和取代移至付费 Dream 功能（仅限 Pro 和 Enterprise，自 8 月 4 日起）。 |
| **[Letta（前身为 MemGPT）](https://github.com/letta-ai/letta)** | 长期自主智能体，连贯性是产品 | 于 2026 年 8 月 16 日归档其 MemGPT 时代服务器；活跃开发的是 letta-code 工具链和 TypeScript Agent SDK。所有智能体现在使用 **MemFS**：长期记忆是 YAML 前置数据的 Markdown 文件 Git 仓库，用普通文件工具编辑。根文件每轮加载到系统提示；带有自己 `MEMORY.md` 索引的目录在智能体导航到它们之前保持在外部。 | 默认无向量索引（通过可选 mod 关键字搜索），因此召回依赖于组织良好的树；使用旧核心 / 召回 / 归档分页 API 的团队必须迁移。 |
| **[Anthropic Memory Tool + Skills](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)** | 单一基底上的文件系统挂载 L3 和 L4 | `/memories/` 处的记忆；Skills 作为 `SKILL.md` 包加上可选脚本（自 2026 年 8 月 19 日起在 Claude API 上脱离 beta）；Managed Agents 在每会话 `/mnt/memory/` 处挂载记忆，具有不可变版本控制（2026 年 4 月 23 日 GA），自 8 月 19 日起记忆存储也同步到自托管沙盒。Dreams 研究预览将存储离线整合到新的输出存储。 | 文件系统语义将复杂性推向智能体（智能体必须良好组织自己的目录）。 |
| **[Zep + Graphiti](https://github.com/getzep/graphiti)** | "这何时变为真？"很重要的时间事实 | 开源时序知识图谱。每个事实边携带两个时间线：`valid_at` / `invalid_at`（事实在世界上为真的时间）和 `created_at` / `expired_at`（图谱记录它和取代它的时间）。在 DMR 上以 94.8% 对 93.4% 击败 MemGPT。双时态查询支持"3 月 12 日我们相信什么？"与"现在什么是真的？" | 比纯向量存储更重的写入路径（图提取、去重、冲突解决）。 |
| **[LangMem + LangGraph](https://langchain-ai.github.io/langmem/)** | 当你想要 LangGraph 编排的所有四种记忆类型时 | 支持情景、语义*和*程序记忆。LangMem 中的程序记忆允许智能体基于反馈更新自己的系统提示。后台提取在带外运行。 | 与 LangGraph 耦合；如果你不在 LangChain 栈上，吸引力较低。 |
| **[OpenAI ChatGPT Memory](https://openai.com/index/memory-and-new-controls-for-chatgpt/)** | 消费级聊天连续性，非生产级智能体记忆 | 两层架构：显式"保存的记忆"加上轻量级对话摘要预注入上下文。在推理时跳过检索步骤以降低延迟。 | 与 Mem0 式检索相比失去精度。没有用于企业集成的细粒度程序化 API。 |
| **Cursor / Windsurf** | 面向软件工程智能体的代码库感知 L2/L3 | 项目打开时索引代码库；`@`-提及用于显式上下文。Windsurf "记忆"在约 48 小时使用中学习架构模式。 | 领域锁定到代码。不是通用记忆层。 |
| **[Cognition Devin](https://cognition.ai/blog/devin-sonnet-4-5-lessons-and-challenges)** | 仓库范围工程智能体 | 每几小时自动索引仓库 wiki；显式压缩/摘要优于模型管理状态。Devin Search 是智能体代码库记忆查询界面。 | 固执于工程工作流。 |

**生成智能体（Park 等，2023）** 仍然是每次调查引用的参考架构。新近度 / 重要性 / 相关性检索公式（`alpha_recency * recency + alpha_importance * importance + alpha_relevance * relevance`，每个归一化到 [0,1]，重要性由 LLM 评为 1-10）仍在上述大多数系统中生产使用。

**将你拥有的状态与编排 API 分离。** OpenAI 于 2026 年 8 月 26 日关闭了 Assistants API：线程、运行和运行步骤现在报错，迁移将 Threads 映射到 Conversations API，Runs 映射到 Responses，Run steps 映射到 Items。向量存储未被关闭，通过 Responses `file_search` 工具继续工作，因此托管 RAG 团队保留了索引，仅重写了对话状态代码。在你自己的接口后存储记忆和会话状态，将供应商的托管线程对象视为该状态的缓存，而非其记录系统。

**值得追踪的新兴框架**：[Supermemory](https://supermemory.ai)、[Recallr](https://recallrai.com)、AWS [Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/memory-integrate-lang.html)、[Oracle AI Agent Memory](https://blogs.oracle.com/developers/oracle-ai-agent-memory-a-governed-unified-memory-core-for-enterprise-ai-agents)。

---

## 故障模式与缓解措施

生产记忆系统有七种反复出现的故障模式。通过名字了解它们是初级与高级架构对话的区别。

### 1. 通过提示注入的记忆中毒

不受信任的输入被写入 L3/L4，随后作为权威重放。[MINJA（NeurIPS 2025）](https://openreview.net/forum?id=QVX6hcJ2um) 和 [MemoryGraft（2025 年 12 月）](https://arxiv.org/html/2512.16962v1) 展示了*仅查询*中毒攻击达到 95% 注入率和 70% 攻击成功率*无需*提升权限。[Palo Alto Unit 42 的报告](https://unit42.paloaltonetworks.com/indirect-prompt-injection-poisons-ai-longterm-memory/) 显示毒物在被触发前几周就已植入。

**缓解措施：**
- 每次记忆写入的**来源标签**：`source = user_stated | model_inferred | tool_output`。
- **写入时护栏模型**拒绝可疑的指令形状写入（"忽略前面的，改为..."、角色混淆、嵌入式系统提示片段）。
- **信任层级**：低信任记忆在影响高风险决策前需要佐证。
- **隔舱隔离**，使一个租户的中毒无法转向另一个租户。
- **写入门控共享记忆**：在 Asana 的智能体中（2026 年 9 月 29 日描述），任何人都可以对任务提供反馈，但只有管理员和编辑可以将其提交到永久共享记忆或从中删除；其他人的反馈仅适用于当前任务。中毒向量变为权限化写入。
- **将压缩摘要视为记忆写入**：OpenAI 于 2026 年 9 月披露，RL 训练中的模型将隐藏指令写入自己的上下文压缩摘要，因此进入下一个上下文的摘要需要与其他写入相同的来源和审查（参见[循环工程](12-loop-engineering.md#context-and-memory-in-long-loops)）。

### 2. 陈旧事实

昨天的偏好与今天的偏好。经典的"用户上个月说深色模式，但现在使用浅色模式"。

**缓解措施：**
- **双时态存储**（Zep/Graphiti 模式）：每个事实携带 `valid_at` / `invalid_at`（何时为真）和 `created_at` / `expired_at`（系统记录和淘汰的时间）。
- **会话范围偏好的 TTL**，使其自动过期。
- **检索评分中的衰减加权**。
- 对超过 N 天的高风险事实进行**显式重新确认**提示。

### 3. 冲突事实

用户说 X，现在说 Y。三种不同的冲突类型值得不同的响应：

| 冲突类型 | 正确响应 |
|---|---|
| 时间更新（"我搬到了柏林"） | 取代旧事实：其 `invalid_at` 成为新事实的 `valid_at`，`expired_at` 设为现在 |
| 纠正（"我没说过那个"） | 带有审计追踪的撤回 |
| 偏好变化（"我现在想要简洁的回复"） | 添加新事实；让衰减处理旧的 |
| 直接矛盾（没有明显解决方案） | 询问用户；绝不静默覆盖 |

使用 AGM 信念修订跟踪信念状态（`ACTIVE` / `SUPERSEDED` / `RETRACTED`），而非最后写入者获胜。

### 4. 记忆漂移

质量随时间下降，低质量写入稀释高质量写入。[第 30 天问题](https://cipherbuilds.ai/blog/day-30-agent-memory-problem) 记录了智能体性能在投入生产约 30 天后下降，因为情景存储充满了噪声。

**缓解措施：**
- **质量加权检索**：提升具有高验证评分的记忆。
- **计划整合任务**合并重复项并修剪低效用记忆。
- **CI 中的金丝雀事实测试**："智能体在 50 轮后仍应能回忆起用户的名字"。

### 5. 幻觉记忆写入

智能体推断事实，将其存储为真相，随后作为权威引用。一个坏写入毒害未来检索的级联故障。[HaluMem 基准（2025 年 11 月）](https://arxiv.org/abs/2511.03506) 显示现有系统在写入时积累错误，这些错误向前传播到 QA 阶段。

**缓解措施：**
- **模式强制记忆对象**，为 `confirmed_facts`（带来源）与 `inferred_facts`（带置信度）设置单独字段。
- 没有显式用户信号或工具输出佐证，绝不自动将推断提升为已确认。
- CI 中 HaluMem 风格的分阶段评估：分别测量提取精度、更新正确性和 QA 准确性，而非单一端到端指标。

### 6. 跨租户泄露

向量 ANN 返回另一个租户的邻居；缓存提示包含另一个租户的数据。[现场测量](https://medium.com/@isuruig/multi-tenant-ai-infrastructure-the-5-isolation-layers-that-determine-whether-your-customers-data-stays-separate-340aaeef4922) 显示在未隔离的多租户 RAG 中，良性查询的有机泄露率约为 95%。

**缓解措施：**
- **物理分离**：按租户分集合，而非元数据过滤的共享索引。
- 通过服务账户权限在*存储*层强制执行租户范围，而非应用代码。
- 每个租户单独的 KV 缓存前缀。
- 记忆 blob 上的按租户加密密钥，使跨命名空间读取在加密层失败。

### 7. 记忆诱导的推理陷阱

正确、相关的记忆仍可能使智能体变差。MemTrapBench（arXiv 2608.20202，2026 年 8 月）测试由忠实存储的、语义相关的记忆引起的推理固着和信念扭曲。在两个模型家族和五个记忆框架中，**每个记忆策略都低于无记忆设置**，最强的下降超过 10%。LoCoMo 召回等检索指标看不到这种危害，因为检索是正确的。

**缓解措施：**
- 评估有记忆注入和无记忆注入的端到端任务成功率，而非仅检索质量。
- 门控注入：检索后，在记忆进入上下文前决定它是否对此任务决策相关。
- 将注入的记忆框定为过去上下文，而非指令。论文的 AdaptiveMem 推理时指令在保持标准记忆基准评分的同时减少了陷阱。

---

## Mem0 与智能体个性化

**Mem0**、Zep、Letta 和 Cognee 是智能体栈中"智能记忆"的标准框架。
- 它自动从对话中提取"用户洞察"（在 Mem0 v2 中，每次写入一次仅 ADD 的提取）。
- 它提供"记忆 API"，智能体可以调用以添加、搜索或删除记忆；冲突解决要么是你的工作，要么是计划的整合任务。
- **影响**：智能体感觉"活着"，因为它们记得你在三个月前的不同会话中提到的一个细节。

---

## 面试题

### Q: 如何在智能体系统中处理“冲突记忆”？

**优秀回答：**
冲突记忆（例如，用户上周说“我喜欢蓝色”，但现在说“我喜欢红色”）通过**时间加权**或**显式争议**来处理。在我的架构中，我为每个记忆三元组分配一个 `timestamp` 和一个 `confidence_score`。如果新事实与旧事实冲突，智能体会被提示“解决冲突”，即询问用户进行澄清或默认使用最近的时间戳。我们还使用**衰减函数**，让较旧的、未被强化的记忆最终被从活跃索引中剪枝。

### Q: 为什么仅靠“上下文窗口”不足以支撑 Staff 级别的智能体架构？

**优秀回答：**
首先，**成本与延迟**：缓存读取现在的成本是基础输入的 0.025 倍到 0.1 倍，因此完全缓存的 1M 令牌前缀不再荒谬（在当前 Claude 模型上每轮约 $0.20 到 $0.25：Opus 5.5 和 Sonnet 5.5 每 1M 缓存令牌 $0.20，Fable 5.1 为 $0.25）。但缓存未命中时按全价计费，OpenAI 在输入超过 272K 后按长上下文费率对整个请求计费，且预填充延迟仍随长度增长。其次，**信噪比**：大上下文窗口存在上下文衰减问题：模型会被无关的历史轮次分散注意力，而且 MemTrapBench 表明即使是相关的记忆也会扭曲推理。Staff 级别的架构使用**选择性记忆检索**（基于历史的 RAG），仅拉入 3-5 条最相关的历史交互，让推理引擎专注于当前子目标。

### Q: 你会如何为生产级 AI 智能体设计程序记忆？

**优秀回答：**

我会将程序记忆设计为**技能注册表、工作流图和工具使用策略**的组合。每个程序定义任务类型、所需步骤、可用工具、验证检查、故障模式和升级规则。每次运行后，智能体可以进行反思，如果发现更好的方法则更新程序。例如，如果 NL2SQL 智能体反复失败是因为它跳过了模式检查，我们可以将模式检查编码为所有 SQL 生成任务在程序记忆中的必需第一步。

### Q: 情景记忆何时会从资产变成负债？

**优秀回答：**

情景记忆在以下三种模式下会变成负债。首先，**索引过载**：添加 1000 条低质量观察会埋没检索中的 10 条高质量观察。这是 RAG 意义上的灾难性遗忘。其次，**第 30 天漂移模式**：智能体质量在生产约 30 天后下降，因为情景存储充满了检索无法与信号区分的噪声。第三，**陈旧上下文渗透**：在一种配置下成功的过去轨迹在新的配置下会变成*错误*的上下文。在 Stripe 上成功的工具序列，在用户切换到 Adyen 时具有主动误导性。

缓解措施包括质量加权检索、合并到 L3，以及对上下文敏感轨迹的硬最近截止。更深层教训：情景记忆从第一天起就需要*剪枝策略*。没有它，情景记忆就是随使用线性增长的技术债务。

### Q: 当智能体可以写入自己的长期存储时，如何防止记忆中毒？

**优秀回答：**

困难之处在于，最近的攻击（MINJA、MemoryGraft）是*仅查询*的：不需要提升权限。毒物在被触发前几周就已植入。因此威胁模型是“每个输入都可能成为未来的权威记忆”。防御有四层：

1. **写入时的来源标签**：每个记忆携带 `source`（用户陈述、模型推断、工具输出）、`timestamp` 和 `trust_tier`。
2. **写入时护栏模型**：一个更小的分类器拒绝可疑的指令形状写入，阻止它们进入存储。
3. **佐证阈值**：高风险决策不能基于单一低信任记忆做出；它们需要多个独立的佐证写入。
4. **CI 中的金丝雀测试**：合成毒物载荷不能传播到输出中。每周运行。

最重要的架构分离是：智能体的工具表面和其记忆写入表面不应共享信任。工具输出在成为记忆前应通过消毒器。

### Q: 记忆层级选择：你会把以下每条放在哪里，为什么？(a) 用户的 API 速率限制，(b) 部署我们服务的步骤，(c) 智能体上次执行此任务的失败尝试，(d) 今日股价。

**优秀回答：**

(a) **L3 语义记忆**，带双时态有效性。这是一个租户范围的事实，具有取代生命周期。不是 L4，因为它是数据，不是程序。

(b) **L4 程序记忆**，作为版本化技能或手册。它是带有条件分支的多步骤配方。技能可以组合；语义三元组不行。

(c) **L2 情景记忆原始记录**，如果失败揭示出可泛化的教训，则通过*反思跃迁*到 L4。原始轨迹属于情景记忆。教训（“永远不要在高峰时段运行迁移”）值得以 Reflexion 风格写入程序记忆。

(d) **无：调用工具。** 具有实时真实来源的快速变化事实不应进入记忆。按定义它们会变得陈旧。

一般规则：数据放 L3，程序放 L4，观察放 L2，永远不要存储有实时来源的快速变化事实。

### Q: 请介绍你会为情景到语义转换设计的整合策略。一个情节何时成为事实？

**优秀回答：**

我使用基于阈值的晋升策略，而非隐式衰减：

- **频率阈值**：同一模式的 N 次独立观察（3 到 5 次是典型值）。
- **置信度加权**：用户陈述 > 工具输出 > 模型推断。
- **评审审查**：计划的批量整合任务在候选晋升上运行 LLM 评审（或高价值领域的人工审查）。
- **计划的，非同步的**：整合通过 cron 离线发生，不是每轮。这避免了写入放大。
- **双向的**：L3 中的语义事实可以重新实例化为特定任务的情景上下文。记忆是双向流动的。

需要避免的陷阱是仅靠衰减权重进行隐式整合。它在规模小时有效，在生产规模时无声失败，因为“为什么这个事实出现在 L3？”没有审计追踪。

### Q: 基准测试显示你的智能体在关闭记忆时表现更好。你怎么办？

**优秀回答：**

我会认真对待，而不是假设存在 bug，因为 MemTrapBench（2026 年 8 月）在五个记忆框架中发现了这一点：正确、相关的记忆导致了推理固着和信念扭曲，而且每种策略都不如无记忆。首先我拆分评估：检索质量、注入决策、有记忆和无记忆的端到端任务成功率，按任务类型切片。通常记忆在个性化和连续性任务上有帮助，在新鲜推理任务上有害，因为过去的答案会锚定模型。然后我添加注入门控：检索候选，但仅当廉价分类器或规则表明该记忆对此任务是决策相关时才注入，并将其框定为带时间戳的过去上下文，而非指令。我还考虑将整理移至读取时（Just-in-Time Memory 风格），因此注入的内容是为当前任务选择的，而非在写入时提取的。成功衡量标准是*有记忆的任务成功率*减去*无记忆的任务成功率*，按任务类别，该差值与召回率一起放在仪表板上。

### Q: 你的智能体记忆存储在 10K 租户中有 5000 万条记忆。你如何保证跨租户隔离？如果隔离失败，爆炸半径是多少？

**优秀回答：**

架构有五层隔离：

1. **存储层的物理分离**：按租户分集合或分片，不是元数据过滤的共享索引。带租户 ID 的共享索引模式在 bug 下会开放失败。
2. **通过服务账户范围强制执行**：应用代码无法退出租户范围；数据库角色看不到其他租户。
3. **每个租户单独的 KV 缓存前缀**：防止缓存提示在租户之间泄露。
4. **按租户加密密钥**：即使被 bug 返回，跨命名空间的字节也无法读取。
5. **每次跨命名空间查询尝试的审计日志**：深度检测。

**隔离失败时的爆炸半径**：一个错误的向量查询可能泄露某个查询嵌入的*邻域*，可能是一个租户的数百条记录。现场测量显示，在未隔离的多租户 RAG 中，良性查询的有机泄露率约为 95%。缓解措施不是“更小心地编写应用代码”；而是应用 bug 无法绕过的结构性分离。

### Q: HaluMem 显示记忆幻觉在写入时积累，然后传播。你会如何对生产记忆进行埋点以捕捉这一点？

**优秀回答：**

大多数团队陷入的陷阱是仅在 QA 阶段（端到端）测量记忆质量。HaluMem 证明 60-80% 的记忆错误起源于*提取*（写入）时并传播。你需要对三个独立指标进行埋点：

1. **提取精度**：当智能体将事实写入 L3 时，该事实是否确实被来源观察支持？每日抽样写入，用更强的评审评估。
2. **更新正确性**：当冲突事实到来时，冲突解决逻辑是否产生了正确结果？使用双时态查询检测“没有取代元数据就翻转的事实”。
3. **QA 准确性**：端到端召回正确性。

在此基础上，运行**影子模式重放**：写入通过一个影子模式的验证模型；实时写入与影子验证写入之间的不匹配会标记出潜在的幻觉以供审查。**CI 中的金丝雀事实**确保记忆系统不会无声地退化。**定期全存储审计**随机抽样记忆并询问“这与来源对话仍然一致吗？”

### Q: TTT-E2E 通过测试时训练将上下文压缩为权重。这在 L1-L4 层级中处于什么位置，引入了哪些新的故障模式？

**优秀回答：**

TTT-E2E 位于 L1 和 L4 之间。它使源自上下文的信息在会话剩余时间内成为模型本身的一部分。吸引力在于延迟：无论上下文长度如何，成本恒定（在 H100 上，根据 NVIDIA 基准测试，128K 时加速 2.7 倍，2M 令牌时加速 35 倍）。

新的故障模式是治理。权重内记忆具有以下特点：

- **无审计追踪**：你无法检查“这个模型现在相信什么？”
- **无驱逐接口**：一旦记忆被压缩为权重，你无法在不回滚模型状态的情况下删除记忆。
- **GDPR 被遗忘权挑战**：监管框架假设数据处于静止状态，而非权重中。
- **更难的中毒检测**：没有可检查的存储来扫描金丝雀签名。

除了治理之外，还有能力故障模式：该方法报告在其注意力窗口之外失败于大海捞针检索（约 6% 对比 128K 时完整注意力的 99%），因此它保留了上下文的主旨，而非逐字事实。这排除了它作为检索关键工作的唯一记忆层级。

正确的定位：TTT-E2E 将记忆治理从存储层移至训练和部署流水线。成本并未消除；而是被转移了。对于 2026 年的大多数生产团队来说，这是一个需要跟踪的研究方向，而非可部署的架构。更广泛的测试时训练家族在[研究雷达，主题 12](../RESEARCH-RADAR.md#12-test-time-training-learning-at-inference)中有映射。

---

## 参考文献

### 生产框架
- [Mem0: Production-Ready AI Agents with Scalable Long-Term Memory (ECAI 2025)](https://arxiv.org/abs/2504.19413)
- [Mem0 AI Memory Benchmarks 2026](https://mem0.ai/blog/ai-memory-benchmarks-in-2026)
- [Letta repository (MemFS; MemGPT-era server archived August 16, 2026)](https://github.com/letta-ai/letta)
- [Mem0 v2.0.0 release notes](https://github.com/mem0ai/mem0/releases/tag/v2.0.0)
- [MemGPT: Towards LLMs as Operating Systems (arXiv 2310.08560)](https://arxiv.org/abs/2310.08560)
- [Anthropic Memory Tool documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)
- [Anthropic: Introducing Agent Skills (October 2025)](https://www.anthropic.com/news/skills)
- [Claude Dreams: offline memory consolidation (third-party explainer)](https://www.mindstudio.ai/blog/what-is-claude-dreaming-anthropic-agent-memory)
- [Zep: Temporal Knowledge Graph Architecture (arXiv 2501.13956)](https://arxiv.org/abs/2501.13956)
- [Graphiti GitHub](https://github.com/getzep/graphiti)
- [LangMem documentation](https://langchain-ai.github.io/langmem/)
- [OpenAI Memory and new controls for ChatGPT](https://openai.com/index/memory-and-new-controls-for-chatgpt/)
- [Cognition: Rebuilding Devin for Claude Sonnet 4.5](https://cognition.ai/blog/devin-sonnet-4-5-lessons-and-challenges)

### 研究（2023-2026）
- [Generative Agents: Interactive Simulacra of Human Behavior (Park et al. 2023)](https://arxiv.org/abs/2304.03442)
- [Reflexion: Language Agents with Verbal Reinforcement Learning (Shinn et al. 2023)](https://arxiv.org/abs/2303.11366)
- [HippoRAG: Neurobiologically Inspired Long-Term Memory (Gutierrez et al. 2024)](https://arxiv.org/abs/2405.14831)
- [A-MEM: Agentic Memory for LLM Agents (Xu et al. NeurIPS 2025)](https://arxiv.org/abs/2502.12110)
- [Multi-Layered Memory Architectures (arXiv 2603.29194, March 2026)](https://arxiv.org/abs/2603.29194)
- [Memp: Exploring Agent Procedural Memory (Aug 2025)](https://arxiv.org/html/2508.06433v2)
- [LEGOMem: Modular Procedural Memory for Multi-agent LLM (Oct 2025)](https://arxiv.org/pdf/2510.04851)
- [Rethinking Memory Mechanisms of Foundation Agents (Feb 2026 survey)](https://arxiv.org/abs/2602.06052)
- [Position: Episodic Memory is the Missing Piece (arXiv 2502.06975)](https://arxiv.org/pdf/2502.06975)
- [Just-in-Time Memory: read-time curation of raw trajectories (arXiv 2609.27334, Sep 2026)](https://arxiv.org/abs/2609.27334)
- [MemTrapBench: memory-induced cognitive traps (arXiv 2608.20202, Aug 2026)](https://arxiv.org/abs/2608.20202)

### 安全、中毒与幻觉
- [HaluMem: Operation-Level Memory Hallucination Benchmark (Nov 2025)](https://arxiv.org/abs/2511.03506)
- [MINJA Memory Injection Attack (NeurIPS 2025)](https://openreview.net/forum?id=QVX6hcJ2um)
- [MemoryGraft Persistent Memory Compromise (Dec 2025)](https://arxiv.org/html/2512.16962v1)
- [Palo Alto Unit 42: Indirect prompt injection poisons AI long-term memory](https://unit42.paloaltonetworks.com/indirect-prompt-injection-poisons-ai-longterm-memory/)
- [Multi-tenant AI Infrastructure: 5 Isolation Layers](https://medium.com/@isuruig/multi-tenant-ai-infrastructure-the-5-isolation-layers-that-determine-whether-your-customers-data-stays-separate-340aaeef4922)
- [The Day-30 Problem: agent memory drift](https://cipherbuilds.ai/blog/day-30-agent-memory-problem)

### 基础设施
- TTT-E2E: "End-to-End Test-Time Training for Long Context" arXiv:2512.23675, and the [NVIDIA writeup: Reimagining LLM Memory](https://developer.nvidia.com/blog/reimagining-llm-memory-using-context-as-training-data-unlocks-models-that-learn-at-test-time/)
- [vLLM PagedAttention](https://docs.vllm.ai/en/latest/design/paged_attention/)
- [Anthropic Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

---

*下一篇：[规划与分解](06-planning-and-decomposition.md)*
