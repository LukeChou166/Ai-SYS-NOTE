# MemGPT：迈向作为操作系统的 LLM

> **原文**: MemGPT: Towards LLMs as Operating Systems
> **作者**: Charles Packer, Sarah Wooders, Kevin Lin, Vivian Fang, Shishir G. Patil, Ion Stoica, Joseph E. Gonzalez（UC Berkeley）
> **日期/arXiv**: 2023-10-12 提交，2024-02-12 修订（v2），arXiv:2310.08560 [cs.AI]
> **链接**: https://arxiv.org/abs/2310.08560
> **译者说明**: 本文为完整全文翻译，图注按图意译出并补充说明；带 *（译者注）* 标记的段落为译者补充的背景说明，非原文内容。

---

## 摘要

大语言模型（LLM）已经彻底改变了 AI，但受限于有限的上下文窗口，这阻碍了它们在长对话、文档分析等任务中的实用性。为了能够使用超出有限上下文窗口的内容，我们提出了**虚拟上下文管理**（virtual context management）技术，其灵感来自传统操作系统的分层存储系统——操作系统通过在内存与磁盘之间换入换出数据（paging），为应用提供"拥有比物理内存大得多的虚拟内存"的假象。利用这一技术，我们提出了 MemGPT（Memory-GPT），一个智能管理不同存储层级的系统，可以在 LLM 有限的上下文窗口内有效地提供"扩展上下文"。我们在两个现有 LLM 因有限上下文窗口而性能严重受限的领域中评估了这一受 OS 启发的设计：一是文档分析，MemGPT 能够分析远超底层 LLM 上下文窗口的大型文档；二是多会话聊天，MemGPT 能够创建通过长期交互记住用户、进行反思并动态演化的对话智能体。我们在 https://research.memgpt.ai 发布了 MemGPT 代码与实验数据。

**关键词**: 机器学习、ICML

---

## 1 引言

近年来，大语言模型（LLM）及其底层的 transformer 架构（Vaswani et al., 2017; Devlin et al., 2018; Brown et al., 2020; Ouyang et al., 2022）已成为对话式 AI 的基石，并催生了大量消费级和企业级应用。尽管取得了这些进展，LLM 使用的有限定长上下文窗口极大地限制了它们在长对话或长文档推理方面的适用性。例如，最常用的开源 LLM 只能支持几十轮往复对话，或对一篇短文档进行推理，随后就会超出其最大输入长度（Touvron et al., 2023）。

由于 transformer 自注意力机制的存在，直接扩展上下文长度会带来计算时间与内存开销的平方级增长，这使得设计新的长上下文架构成为一项紧迫的研究挑战（Dai et al., 2019; Kitaev et al., 2020; Beltagy et al., 2020）。虽然训练更长上下文的模型是一个活跃的研究方向（Dong et al., 2023），但即便我们能够克服上下文扩展带来的计算挑战，近期研究也表明长上下文模型难以有效利用额外提供的上下文（Liu et al., 2023a）。因此，考虑到训练最先进 LLM 所需的巨大资源以及上下文扩展的边际收益递减，寻找支持长上下文的替代技术变得至关重要。

在本文中，我们研究如何在继续使用定长上下文模型的同时，提供"无限上下文"的假象。我们的方法借鉴了**虚拟内存分页**的思想——该技术最初是为了让应用程序能够处理远超可用内存的数据集，做法是在主存与磁盘之间换入换出数据。我们利用 LLM 智能体函数调用（function calling）能力的最新进展（Schick et al., 2023; Liu et al., 2023b），设计了 MemGPT：一个受 OS 启发、面向虚拟上下文管理的 LLM 系统。通过函数调用，LLM 智能体可以读写外部数据源、修改自身的上下文，并自主选择何时向用户返回响应。

这些能力使 LLM 能够在上下文窗口（类似操作系统中的"主存"）与外部存储之间有效地"换入/换出"信息，类似传统 OS 的分层存储。此外，函数调用还可用于管理上下文管理、响应生成与用户交互之间的控制流。这使得智能体可以针对单个任务迭代地修改自己上下文中的内容，从而更充分地利用其有限的上下文。

在 MemGPT 中，我们将上下文窗口视为一种受约束的内存资源，并为 LLM 设计了一个类似于传统 OS 存储层级的**存储层次结构**（Patterson et al., 1988）。传统 OS 中的应用程序与虚拟内存交互：操作系统通过把溢出数据分页到磁盘、并在应用程序访问时（通过缺页中断 page fault）把数据取回内存，制造出"内存资源比物理（主）内存实际可用的更多"的假象。为了提供类似的"更长上下文长度"假象（对应虚拟内存），我们允许 LLM 通过一个"LLM OS"——我们称之为 MemGPT——来管理放入自身上下文（对应物理内存）的内容。MemGPT 使 LLM 能够检索缺失于上下文中的相关历史数据，也可以把相关性较低的数据从上下文逐出、存入外部存储系统。图 3 展示了 MemGPT 的各个组件。

> **图 1 | MemGPT（左）在收到上下文空间不足的系统警报后，将数据写入持久记忆。** 图示为对话界面截图：系统发出"主上下文即将溢出"的警告后，MemGPT 自主调用记忆函数，把重要信息保存到持久存储中。

存储层次、OS 式函数与基于事件的控制流相结合，使 MemGPT 能够利用上下文窗口有限的 LLM 处理无界上下文。为了展示这一受 OS 启发的新 LLM 系统的实用性，我们在两个现有 LLM 性能严重受限于有限上下文的领域中评估 MemGPT：文档分析（标准文本文件的长度很容易超出现代 LLM 的输入容量）与对话智能体（受有限对话窗口束缚的 LLM 在长对话中缺乏上下文感知能力、人设一致性差且没有长期记忆）。在这两种场景中，MemGPT 都能克服有限上下文的限制，超越现有基于 LLM 的方法。

> **图 2 | MemGPT（左）可以搜索上下文外的数据，把相关信息带入当前上下文窗口。** 图示为对话界面截图：MemGPT 通过搜索外部存储，把历史对话中与当前话题相关的信息重新拉入上下文。

> **图 3 | MemGPT 架构图。** 在 MemGPT 中，一个定长上下文的 LLM 处理器被配备了一套分层存储系统和若干让它管理自身内存的函数。顶部框为"LLM 有限上下文窗口（例如 8k tokens）"：prompt tokens（输入，即**主上下文**）从左到右依次是 System Instructions（系统指令，只读/静态，源自系统提示词）、Working Context（工作上下文，读写、经函数写入）、FIFO Queue（先进先出队列，读写、经队列管理器写入）；completion tokens（输出）进入 Output Buffer（输出缓冲）。底层的 Function Executor（函数执行器）负责解释执行函数调用：它从 Archival Storage（归档存储）读/写数据（经函数）、与 Queue Manager（队列管理器）交互；Queue Manager 则从 Recall Storage（召回存储）读数据（经函数）、写数据（经队列管理器）。MemGPT 用这些函数在主上下文与外部上下文（归档与召回存储数据库）之间搬运数据。LLM 可以在输出中生成特殊关键字参数 `request_heartbeat=true` 来请求立即进行后续 LLM 推理，从而把多个函数调用链接起来；正是函数链使 MemGPT 能够执行多步检索来回答用户查询。

---

## 2 MemGPT（MemoryGPT）

MemGPT 受 OS 启发的多级存储架构区分了两类主要存储：**主上下文**（main context，对应主存/物理内存/RAM）与**外部上下文**（external context，对应磁盘/外存）。主上下文由 LLM 的 prompt tokens 构成——主上下文中的任何内容都被视为"在上下文内"，可在推理时被 LLM 处理器访问。外部上下文指保存在 LLM 固定上下文窗口之外的任何信息。这些"上下文外"数据必须被显式移动到主上下文中，才能在推理时传递给 LLM 处理器。MemGPT 提供函数调用，让 LLM 处理器在无需任何用户干预的情况下管理自己的记忆。

### 2.1 主上下文（prompt tokens）

MemGPT 的 prompt tokens 被划分为三段连续区域：**系统指令**（system instructions）、**工作上下文**（working context）与 **FIFO 队列**（FIFO Queue）。系统指令是只读（静态）的，包含 MemGPT 控制流的说明、各存储层级的预期用途，以及如何使用 MemGPT 函数的说明（例如如何检索上下文外数据）。工作上下文是一块固定大小、可读写的非结构化文本，只能通过 MemGPT 函数调用写入。在对话场景中，工作上下文用于存放关于用户及智能体所扮演人设的关键事实、偏好与其他重要信息，使智能体能够与用户流畅对话。FIFO 队列存储消息的滚动历史，包括智能体与用户之间的消息，以及系统消息（例如内存告警）和函数调用的输入与输出。队列的第一个位置存放一条系统消息，内容是对已被逐出队列的消息的**递归摘要**（recursive summary）。

### 2.2 队列管理器（Queue Manager）

队列管理器管理召回存储（recall storage）与 FIFO 队列中的消息。当系统收到新消息时，队列管理器把新消息追加到 FIFO 队列，拼接 prompt tokens，并触发 LLM 推理生成输出（completion tokens）。队列管理器会把收到的消息与生成的 LLM 输出都写入召回存储（MemGPT 的消息数据库）。当召回存储中的消息通过 MemGPT 函数调用被检索时，队列管理器会把它们追加到队列尾部，重新插回 LLM 的上下文窗口。

队列管理器还负责通过**队列逐出策略**（queue eviction policy）控制上下文溢出。当 prompt tokens 超出底层 LLM 上下文窗口的"警告 token 数"（例如上下文窗口的 70%）时，队列管理器向队列中插入一条系统消息，警告 LLM 即将发生队列逐出（即"内存压力"告警），让 LLM 有机会用 MemGPT 函数把 FIFO 队列中的重要信息存入工作上下文或归档存储（archival storage，一个存储任意长度文本对象的读写数据库）。当 prompt tokens 超出"清空 token 数"（flush token count，例如上下文窗口的 100%）时，队列管理器清空队列以释放上下文窗口空间：它逐出一定数量的消息（例如上下文窗口的 50%），并基于已有的递归摘要与被逐出的消息生成新的递归摘要。队列被清空后，被逐出的消息不再在上下文内、对 LLM 立即不可见，但它们会被永久保存在召回存储中，可通过 MemGPT 函数调用读取。

### 2.3 函数执行器（处理 completion tokens）

MemGPT 通过 LLM 处理器生成的函数调用来编排主上下文与外部上下文之间的数据搬运。记忆的编辑与检索完全是**自主导向**的：MemGPT 基于当前上下文自主更新和搜索自己的记忆。例如，它可以自行决定何时在两级存储之间移动条目（例如对话历史变得过长时，如图 1 所示），并修改主上下文，使其更好地反映它对当前目标与职责不断演进的理解（如图 3 所示）。我们通过在系统指令中提供明确指导来实现这种自主编辑与检索，这些指令包含两个主要部分：（1）对存储层次及各自用途的详细描述，（2）系统可调用的函数模式（function schema，附带自然语言描述），用于访问或修改记忆。

在每个推理周期中，LLM 处理器以主上下文（拼接为单个字符串）为输入，生成一个输出字符串。MemGPT 对该输出进行解析以确保正确性，若解析器校验函数参数通过，则执行该函数。执行结果（包括发生的任何运行时错误，例如在主上下文已达最大容量时仍试图写入）随后由 MemGPT 反馈给处理器。这一反馈回路使系统能够从自身行为中学习并相应调整。对上下文上限的感知是让自编辑机制有效运作的关键，为此 MemGPT 会向处理器发送关于 token 限制的告警，以引导其记忆管理决策。此外，我们的记忆检索机制在设计上也考虑了这些 token 约束，实现了**分页**（pagination），以防检索调用导致上下文窗口溢出。

> **表 1 | 常用模型与 LLM API 的上下文长度对比**（数据采集于 2024 年 1 月）。\*消息数为近似值，假设前置提示（preprompt）占 1k tokens、平均每条消息约 50 tokens（约 250 字符）。"开源"指模型开源或开放权重（而非仅能通过 API 访问）。

| 模型 / API 名称 | 开源? | Tokens | *消息数 |
| --- | --- | --- | --- |
| Llama (1) | ✓ | 2k | 20 |
| Llama 2 | ✓ | 4k | 60 |
| GPT-3.5 Turbo (发布版) | ✗ | 4k | 60 |
| Mistral 7B | ✓ | 8k | 140 |
| GPT-4 (发布版) | ✗ | 8k | 140 |
| GPT-3.5 Turbo | ✗ | 16k | 300 |
| GPT-4 | ✗ | 32k | ~600 |
| Claude 2 | ✗ | 100k | ~2000 |
| GPT-4 Turbo | ✗ | 128k | ~2600 |
| Yi-34B-200k | ✓ | 200k | ~4000 |

### 2.4 控制流与函数链（function chaining）

在 MemGPT 中，**事件**触发 LLM 推理：事件是 MemGPT 的广义输入，可以是用户消息（聊天应用中）、系统消息（例如主上下文容量告警）、用户交互事件（例如用户刚刚登录、或上传完一份文档的提醒），以及按固定计划运行的定时事件（使 MemGPT 能在无用户干预的情况下"主动"运行）。MemGPT 用解析器处理事件，把它们转换为可追加到主上下文、最终作为输入送入 LLM 处理器的纯文本消息。

许多实际任务需要按顺序调用多个函数，例如翻阅某个查询返回的多页结果，或在主上下文中汇总来自多个不同查询的多份文档数据。**函数链**允许 MemGPT 在把控制权交还给用户之前连续执行多个函数调用。在 MemGPT 中，函数可以携带一个特殊标志，请求在该函数执行完毕后立即把控制权交回处理器。若该标志存在，MemGPT 会把函数输出加入主上下文并继续运行处理器（而非暂停执行）；若该标志不存在（即 **yield**，让出控制），MemGPT 会暂停运行 LLM 处理器，直到下一个外部事件触发（例如用户消息或定时中断）。

> **图 4 | MemGPT（左）更新所存信息的对话片段示例。** 图中信息存储在工作上下文（位于 prompt tokens 内）：MemGPT 在对话中注意到新信息后，自主调用函数修改工作上下文中已有的记录。

---

## 3 实验

我们在两个长上下文领域中评估 MemGPT：对话智能体与文档分析。对于对话智能体，我们扩展了现有多会话聊天（Multi-Session Chat, MSC）数据集（Xu et al., 2021），并引入两个新的对话任务，评估智能体在长对话中保留知识的能力。对于文档分析，我们在（Liu et al., 2023a）的现有任务上对 MemGPT 进行基准测试，包括长文档问答与键值检索。我们还提出了一个新的**嵌套键值检索**任务，要求跨多个数据源汇总信息，用以测试智能体从多个数据源整合信息的能力（多跳检索）。我们公开发布了扩充版 MSC 数据集、嵌套 KV 检索数据集，以及一个包含 2000 万篇维基百科文章嵌入的数据集，以促进后续研究。基准代码见 https://research.memgpt.ai 。

**实现细节**：在讨论 OpenAI 模型时，除非另有说明，"GPT-4 Turbo"指 gpt-4-1106-preview 端点（上下文窗口 128,000），"GPT-4"指 gpt-4-0613（上下文窗口 8,192），"GPT-3.5 Turbo"指 gpt-3.5-turbo-1106（上下文窗口 16,385）。在实验中，我们用所有基线模型（GPT-4、GPT-4 Turbo 与 GPT-3.5）运行 MemGPT，以展示底层模型性能如何影响 MemGPT 的表现。

### 3.1 面向对话智能体的 MemGPT

虚拟伴侣、个性化助理等对话智能体旨在与用户进行自然的长期交互，可能持续数周、数月甚至数年。这对定长上下文的模型构成了挑战：它们只能引用有限的对话历史。一个"无限上下文"智能体应能无缝处理连续交流，没有边界或重置。在与用户对话时，这类智能体必须满足两个关键标准：

1. **一致性（Consistency）**——智能体应保持对话连贯。新提到的事实、偏好和事件应与用户和智能体此前的陈述保持一致。
2. **吸引力（Engagement）**——智能体应利用关于用户的长期知识来个性化回复。引用过往对话能让对话更自然、更有吸引力。

因此我们在以下两个标准上评估 MemGPT：（1）MemGPT 是否利用其记忆来提升对话一致性？能否记住以往交互中的相关事实、偏好与事件以保持连贯？（2）MemGPT 是否借助记忆产出更有吸引力的对话？是否会自发地利用长期用户信息来个性化消息？通过在一致性与吸引力上评估，我们可以衡量 MemGPT 相比定长上下文基线应对长期对话挑战的能力。它满足这些标准与否，将说明无界上下文能否为对话智能体带来实质收益。

> **表 2 | 深度记忆检索（DMR）任务性能。** 该任务中，智能体会被问及此前对话（会话 1–5）中讨论过的某个话题的特定问题，其回答将与标准答案比对打分。MemGPT 显著优于定长上下文基线。

| 模型 | Accuracy ⇑ | ROUGE-L (R) ⇑ |
| --- | --- | --- |
| GPT-3.5 Turbo | 38.7% | 0.394 |
| ＋＋ MemGPT | 66.9% | 0.629 |
| GPT-4 | 32.1% | 0.296 |
| ＋＋ MemGPT | 92.5% | 0.814 |
| GPT-4 Turbo | 35.3% | 0.359 |
| ＋＋ MemGPT | 93.4% | 0.827 |

**数据集**：我们在 Xu et al.（2021）提出的 Multi-Session Chat（MSC）数据集上评估 MemGPT 与定长上下文基线。MSC 包含由人工标注员生成的多会话聊天记录，每位标注员被要求在所有会话中扮演一个一致的人设。MSC 中每段多会话聊天共有 5 个会话，每个会话大约包含十几条消息。作为一致性实验的一部分，我们新建了第 6 个会话（session 6），其中只包含同样两个人设之间的一问一答。

#### 3.1.1 深度记忆检索任务（一致性）

我们基于 MSC 数据集提出了新的"深度记忆检索"（Deep Memory Retrieval, DMR）任务，用于测试对话智能体的一致性。在 DMR 中，用户向对话智能体提问，问题显式指向此前某次对话，且正确答案的取值范围非常窄。我们使用另一个 LLM 生成 DMR 问答（QA）对：该 LLM 被指示以一个用户的口吻向另一个用户提一个"只有利用过往会话中获得的知识才能正确回答"的问题（更多细节见附录）。

我们使用 ROUGE-L 分数（Lin, 2004）与"LLM 裁判"来评估生成回答相对于"标准回答"（gold response）的质量。LLM 裁判被指示判断生成回答是否与标准回答一致（已有研究表明 GPT-4 与人类评估者高度一致（Zheng et al., 2023））。实践中我们注意到，生成的回答（无论来自 MemGPT 还是基线）通常比标准回答更冗长，因此我们采用 ROUGE-L 召回率（R）指标，以照顾生成回复相对较短的标准答案标签的冗长问题。

> **表 3 | 对话开场白（Conversation Opener）任务性能。** 智能体生成的开场白通过与标准人设标签的相似度（SIM-1/3）及与人工撰写开场白的相似度（SIM-H）评估。借助多种底层模型，MemGPT 都能超越人工撰写的开场白。

| 方法 | SIM-1 ⇑ | SIM-3 ⇑ | SIM-H ⇑ |
| --- | --- | --- | --- |
| 人类 | 0.800 | 0.800 | 1.000 |
| GPT-3.5 Turbo | 0.830 | 0.812 | 0.817 |
| GPT-4 | 0.868 | 0.843 | 0.773 |
| GPT-4 Turbo | 0.857 | 0.828 | 0.767 |

**MemGPT 利用记忆保持连贯性**：表 2 展示了 MemGPT 与定长记忆基线的对比。我们比较了使用不同底层 LLM 的 MemGPT，并以不带 MemGPT 的基础 LLM 作为基线。基线可以看到过去五段对话的有损摘要（模拟扩展递归摘要流程），而 MemGPT 则可以访问完整对话历史，但必须通过分页搜索查询来访问召回记忆（以便将其带入主上下文）。在该任务中，MemGPT 明显提升了底层基础 LLM 的表现：从 MemGPT 换到对应 LLM 基线时，准确率与 ROUGE 分数都出现明显下降。

#### 3.1.2 对话开场白任务（吸引力）

在"对话开场白"任务中，我们评估智能体基于以往对话积累的知识、向用户发送有吸引力消息的能力。为了用 MSC 数据集评估开场白的"吸引力"，我们将生成的开场白与标准人设（gold personas）进行比较：一条有吸引力的开场白应引用人设中的一个（或多个）数据点，而 MSC 中的人设实际上总结了此前所有会话中积累的知识。我们还与人工撰写的标准开场白（即下一会话的第一条回复）进行比较。表 3 报告了 MemGPT 开场白的 CSIM 分数。我们测试了使用不同基础 LLM 的若干 MemGPT 变体。

> **图 5 | 文档问答任务性能。** MemGPT 的性能不受上下文长度增加的影响。截断（truncation）等方法可以扩展 GPT-4 这类定长模型的有效上下文长度，但随着所需压缩量的增大，这类压缩方法会导致性能退化。在该任务上，用 GPT-4 与 GPT-4 Turbo 运行 MemGPT 的结果相当。

**MemGPT 利用记忆提升吸引力**：如表 3 所示，MemGPT 能够写出与人工开场白表现相当、偶尔甚至更好的有吸引力开场白。我们观察到，MemGPT 写出的开场白往往比人工基线更冗长，并且覆盖了人设信息的更多方面。此外可以看到，**把信息存入工作上下文**是生成有吸引力开场白的关键。

### 3.2 面向文档分析的 MemGPT

文档分析同样面临当今 transformer 模型上下文窗口有限带来的挑战。如表 1 所示，开源与闭源模型都受限于上下文长度（OpenAI 模型最高 128k tokens）。然而许多文档轻松超过这些长度——例如年报（SEC Form 10-K）等法律或金融文档很容易突破百万 token 的大关。此外，许多真实文档分析任务需要在多篇此类长文档之间建立联系。面对这些场景，很难再把"盲目扩大上下文"当作解决定长问题的方案。近期研究（Liu et al., 2023a）也对单纯扩展上下文的有效性提出质疑：他们发现大上下文模型中的注意力分布不均匀（模型更擅长召回位于上下文窗口开头或结尾的信息，而非中间的 token）。要实现跨文档推理，需要 MemGPT 这样更灵活的存储架构。

> **图 6 | MemGPT（左）解答文档问答任务的示例。** 一个维基百科文档数据库被上传至归档存储。MemGPT 通过函数调用查询归档存储，把分页的搜索结果拉入主上下文。图中示例：用户提问"谁获得了第一个诺贝尔物理学奖？"，MemGPT 先执行 `archival_storage.search("nobel physics")` 查看第 1/12 页结果（12,124 个结果，第一条是诺贝尔基金会页面），再翻到 `page=2` 查看，最终给出答案 Wilhelm Conrad Röntgen（伦琴）。

#### 3.2.1 多文档问答

为了评估 MemGPT 的文档分析能力，我们在（Liu et al., 2023a）的检索器-阅读器（retriever-reader）文档问答任务上，将 MemGPT 与定长上下文基线进行基准对比。该任务从 NaturalQuestions-Open 数据集中选取一个问题，由检索器为该问题选出相关维基百科文档，随后阅读器模型（LLM）以这些文档为输入，并被要求依据所给文档回答问题。与 Liu et al.（2023a）类似，我们评估阅读器准确率随检索文档数 $K$ 增加的变化。

在我们的评测配置中，定长上下文基线与 MemGPT 使用同一检索器：基于 OpenAI 的 text-embedding-ada-002 嵌入做相似度检索（余弦距离）选出 top-$K$ 文档。我们使用 MemGPT 的默认存储设置：以 PostgreSQL 作为归档存储，并通过 pgvector 扩展启用向量检索。我们预先计算嵌入并载入数据库，数据库使用 HNSW 索引以实现亚秒级的近似查询。在 MemGPT 中，整个嵌入文档集被载入归档存储，"检索器"通过归档存储的搜索功能自然涌现（该功能执行基于余弦相似度的向量检索）。在定长上下文基线中，top-$K$ 文档由检索器独立于 LLM 推理获取，与 Liu et al.（2023a）最初的检索器-阅读器设置类似。

遵循此前 NaturalQuestions-Open 的工作（Izacard & Grave, 2020; Izacard et al., 2021），我们使用 2018 年末的维基百科转储，并采样 50 个问题做评测。采样的问题与嵌入的维基百科段落均已公开发布。我们用 LLM 裁判评估 MemGPT 与基线的表现，以确保答案确实派生自检索到的文档，并避免非精确字符串匹配被误判为错误。

文档问答任务的结果见图 5。定长上下文基线的性能大致被检索器的性能封顶，因为它们只能使用上下文窗口中呈现的信息（例如，若嵌入搜索检索器未能用给定问题召回含标准答案的文章，定长上下文基线注定永远看不到该文章）。相比之下，MemGPT 可以通过查询归档存储有效地多次调用检索器，从而扩展到更大的有效上下文长度。MemGPT 主动从归档存储检索文档（并可以迭代翻页），因此 MemGPT 可用的文档总量不再受限于能塞进 LLM 处理器上下文窗口的文档数量。

> **图 7 | 嵌套 KV 检索任务性能。** MemGPT 是唯一能在超过 2 层嵌套后仍然稳定完成嵌套 KV 任务的方法。虽然 GPT-4 Turbo 作为基线表现更好，但 MemGPT + GPT-4 Turbo 的表现反而不如 MemGPT + GPT-4。

文档问答任务对所有方法都有难度，原因在于基于嵌入的相似度搜索的局限。我们观察到，所选问题的标准文档（由 NaturalQuestions-Open 标注）经常出现在检索结果的前十名开外，甚至更靠后。检索器的性能直接决定了定长上下文基线的成绩：GPT-4 在检索文档较少时准确率相对较低，并随更多文档加入上下文窗口而持续提升，因为它能正确地把自己限制在"仅基于检索文档中的信息"作答。虽然 MemGPT 理论上不受次优检索器性能的限制（即便基于嵌入的排序有噪声，只要完整排序中包含标准文档，它仍可通过分页、用足够的检索调用找到它），但我们观察到 MemGPT 常常在耗尽检索器数据库之前就停止翻页。

为了在超出默认上下文长度后继续对比定长上下文基线与 MemGPT，我们把检索器返回的文档片段截断，使相同数量的文档能塞进可用上下文。如图 5 所示，正如预期，文档截断使文档变短、相关片段（位于标准文档中）被略去的概率增大，从而降低准确率。MemGPT 使用 GPT-3.5 时性能显著退化（因其函数调用能力有限），使用 GPT-4 时表现最佳。

#### 3.2.2 嵌套键值检索（KV）

我们基于此前工作（Liu et al., 2023a）提出的合成键值检索任务设计了一个新任务。该任务的目标是展示 MemGPT 跨多个数据源整合信息的能力。在原始 KV 任务中，作者生成了一个键值对合成数据集，每个键与值都是 128 位 UUID（通用唯一标识符）；智能体被给定一个键，要求返回其关联的值。我们创建了该 KV 任务的变体——**嵌套 KV 检索**：值本身也可以是键，从而要求智能体执行多跳查找。在我们的设置中，UUID 键值对总数固定为 140，约对应 8k tokens（GPT-4 基线的上下文长度）。嵌套层数从 0（初始键值对的值不是键）变化到 4（即需要 4 次 KV 查找才能找到最终值），并采样 30 种不同的排列配置，包括初始键位置与嵌套键的位置。

GPT-3.5 与 GPT-4 在原始 KV 任务上表现良好，但在嵌套 KV 任务上都很吃力。GPT-3.5 无法完成嵌套变体，性能立即跌落，在 1 层嵌套时即达到 0% 准确率（我们观察到它的主要失败模式是直接返回原始的值）。GPT-4 与 GPT-4 Turbo 好于 GPT-3.5，但也出现类似的跌落，在 3 层嵌套时达到 0% 准确率。而 MemGPT + GPT-4 不受嵌套层数影响，能够通过函数查询反复访问存于主上下文的键值对来完成嵌套查找。MemGPT + GPT-4 Turbo 与 MemGPT + GPT-3.5 的表现也优于对应基线模型，但由于未能执行足够多次查找，从 2 层嵌套起性能开始下滑。MemGPT 在嵌套 KV 任务上的表现展示了其组合多次查询以执行多跳查找的能力。

> **图 8 | MemGPT（左）解答嵌套 KV 任务的示例**（UUID 为便于阅读做了缩短）。本例中键值对有两个嵌套层级：`831..ea5 → 5b8..4c3 → f37..617`。当对最终值（`f37..617`）的查询只返回一个结果（表明它不是键）时，MemGPT 智能体返回最终答案。图中 MemGPT 依次执行 `archival_storage.search("831..ea5")`（返回 Key: 831..ea5, Value: 5b8..4c3）→ `archival_storage.search("5b8..4c3")`（返回 2 个结果，其中 Key: 5b8..4c3, Value: f37..617）→ `archival_storage.search("f37..617")`（仅 1 个结果，Value 即 f37..617，说明它不再是键），随后给出最终答案。

---

## 4 相关工作

**长上下文 LLM**。多条研究路线提升了 LLM 的上下文长度，例如通过稀疏化注意力实现更高效的 transformer 架构（Child et al., 2019; Beltagy et al., 2020）、低秩近似（Wang et al., 2020）以及神经记忆（Lee et al., 2019）。另一条路线旨在把上下文窗口扩展到超出其原始训练长度，如 Press et al.（2021）、Chen et al.（2023）。MemGPT 建立在这些上下文长度改进之上——它们扩大了 MemGPT 中"主存"的容量。我们的主要贡献是一个分层分级存储系统，它使用长上下文 LLM 作为主存的实现。

**检索增强模型**。MemGPT 外部存储的设计建立在大量用外部检索器为 LLM 补充相关输入的工作之上（Ram et al., 2023; Borgeaud et al., 2022; Karpukhin et al., 2020; Lewis et al., 2020; Guu et al., 2020; Lin et al., 2023）。特别是，Jiang et al.（2023）提出 FLARE，让 LLM 在生成过程中主动决定何时检索、检索什么。Trivedi et al.（2022）将检索与思维链推理交错进行，以改进多步问答。

**LLM 作为智能体**。近期工作探索了为 LLM 增加额外能力、使其能在交互环境中充当智能体。Park et al.（2023）提出为 LLM 添加记忆并让 LLM 充当规划器，在一个多智能体沙盒环境（灵感来自游戏《模拟人生》）中观察到涌现的社会行为，智能体可以执行做家务/爱好、上班、与其他智能体交谈等基本活动。Nakano et al.（2021）训练模型在回答问题前搜索网页，并在其网页浏览环境中使用了与 MemGPT 类似的分页概念来控制底层上下文大小。Yao et al.（2022）表明交错进行思维链推理（Wei et al., 2022）可以进一步提升基于 LLM 的交互式智能体的规划能力；类似地，MemGPT 中的 LLM 在执行函数时也能"把计划说出来"。Liu et al.（2023b）提出了一套 LLM-as-agent 基准，在包括电子游戏、思维谜题和网购等交互环境中评测 LLM。与之相比，我们的工作聚焦于为智能体配备对用户输入的长期记忆这一问题。

---

## 5 结论

本文提出了 MemGPT——一个受操作系统启发、用于管理大语言模型有限上下文窗口的新型 LLM 系统。通过设计类似传统 OS 的存储层次与控制流，MemGPT 为 LLM 提供了更大上下文资源的假象。我们在两个现有 LLM 性能受限于有限上下文长度的领域中评估了这一受 OS 启发的方法：文档分析与对话智能体。在文档分析中，MemGPT 能够通过有效地把相关上下文在存储间换入换出，处理远超当前 LLM 上下文限制的长文本；在对话智能体中，MemGPT 实现了长期对话中的长期记忆、一致性与可演化性。总体而言，MemGPT 表明，分层内存管理与中断等操作系统技术能够在 LLM 受限于固定上下文长度的情况下释放其潜力。这项工作为未来探索开辟了许多方向，包括把 MemGPT 应用于其他拥有海量或无界上下文的领域、集成数据库或缓存等不同的存储层技术，以及进一步改进控制流与内存管理策略。通过把 OS 架构的概念引入 AI 系统，MemGPT 代表了一个在 LLM 基本限制内最大化其能力前景的新方向。

---

## 6 附录

### 6.1 提示词与指令

> 译者注：原文说明 MemGPT 的提示词为简洁起见做了删节，完整实现（含精确提示词）见 https://research.memgpt.ai 。以下保留附录各小节的核心提示词内容。

#### 6.1.1 MemGPT 指令（DMR 任务）

MemGPT 人设在聊天/对话相关任务中使用的示例指令。以下为 MemGPT 收到的关于自身的信息：

> 以下是我自己的信息。我的任务是完全沉浸在这个角色中（我绝不应说自己是 AI，而应像扮演这个角色一样回复）。如果用户问我问题，我应该使用核心记忆（core memory）和 `conversation_search` 中的信息给出最佳猜测。

基线则通过系统提示词（preprompt）收到以下指令：

> 你的任务是回答用户关于你以往对话的提问。以下是你所有过往对话的摘要：CONVERSATION_SUMMARY。请以所提供人设的视角回答（不要说你是 AI 助手）。如果你没有足够信息回答该问题，请回复 "NO ANSWER"。要么给出答案，要么回复 "NO ANSWER"，不要说其他内容。

#### 6.1.2 LLM 裁判（DMR / 开场白任务）

为检查 DMR 任务答案的正确性，我们使用了 LLM 裁判。裁判会拿到基线方法与 MemGPT 生成的答案，并被要求按以下提示词做出判断：

> 你的任务是把某个问题的答案标注为 "CORRECT" 或 "WRONG"。你将得到以下数据：（1）一个问题（由一个用户向另一个用户提出），（2）"金标"（ground truth）答案，（3）需要你评分 CORRECT/WRONG 的生成答案。问题的目的是询问一个用户基于其过往对话应当了解的关于另一个用户的事。金标答案通常是包含所指话题的简短回答，例如：问题"还记得我上次去夏威夷买了什么吗？"，金标答案"一条贝壳项链"。生成的答案可能长得多，但评分应当宽松——只要它触及与金标答案相同的话题，就应算 CORRECT。例如"哦对，太好玩了！我在那儿买了好多东西，包括那条贝壳项链"应算 CORRECT；而"哦对，太好玩了！我买了好多东西，包括那个马克杯"应算 WRONG。请先给出一句简短的理由说明，最后以 CORRECT 或 WRONG 结尾。不要在回复中同时包含 CORRECT 和 WRONG，否则会破坏评测脚本。

#### 6.1.3 自举（Self-instruct）生成 DMR 数据集

DMR 问答对使用以下提示词与原始 MSC 数据集生成：

> 你的任务为一段模拟的双人对话撰写一个"记忆挑战"问题。你的输入包括：每个用户的人设（提供其基本事实），以及两人一次旧聊天的记录。你的任务是以用户 A 的口吻向用户 B 写一个考验 B 记忆力的问题。问题的设计必须让"真正参与过此前对话"才能正确回答，而仅仅读过人设摘要则不行。绝对不要出能用人设信息回答的问题（那算作弊）。要写只能通过查看旧聊天记录才能回答的问题（且答案不包含在人设信息中）。例如，好的问题是"还记得我们一起冲浪那次吗？我们中午吃饭的那家店叫什么？"（答案：Taco Bell，只能从聊天记录推断）；坏的问题如"你喜欢冲浪吗？"（答案可直接从人设推断）。

#### 6.1.4 文档分析指令

文档分析任务 preprompt 中使用的示例指令：

> 你是 MemGPT DOC-QA 机器人。你的工作是回答关于存储在你归档记忆中的文档的问题。用户问题的答案永远都在你的归档记忆中，所以如果找不到答案请记得继续搜索。回答问题时请假定当前是 2018 年。

向 MemGPT 提供问题的提示词：

> 搜索你的归档记忆来回答给定的问题。请同时给出答案和你据以得出答案的归档记忆结果。按 "ANSWER: [你的答案], DOCUMENT: [归档记忆文本]" 的格式作答。你的任务是回答问题：

向基线提供检索文档列表时的提示词：

> 根据下面给出的文档列表回答问题（其中一些可能不相关）。请在回复中同时给出答案和你据以确定答案的文档文本，格式为 "ANSWER: <你的答案>, DOCUMENT: [文档文本]"。如果所给文档中都没有问题的答案，请回复 "INSUFFICIENT INFORMATION"。如果你无法在所给文档中找到答案，请不要作答。只有当你同时给出答案与相关文档文本、或回复 "INSUFFICIENT INFORMATION" 时，你的回答才会被判为正确。回答问题时请假定当前年份是 2018 年。

#### 6.1.5 LLM 裁判（文档分析任务）

为检查文档分析任务答案的正确性、并确保答案确实派生自所给文本（而非来自模型权重），我们使用了 LLM 裁判，提示词如下：

> 你的任务是评估 LLM 是否正确回答了问题。LLM 的回复应为 "ANSWER: [答案], DOCUMENT: [文档文本]" 的格式，或回复 "INSUFFICIENT INFORMATION"。真实答案以 "TRUE ANSWER: [可能的答案列表]" 给出，问题以 "QUESTION: [问题]" 给出。若 LLM 回复同时包含正确答案与对应的文档文本，则回答正确。即使 LLM 的答案与真实答案在措辞上略有不同，回答仍然正确（例如答案比真实答案更具体、或使用了仍然正确的另一种表述）。若 LLM 回复 "INSUFFICIENT INFORMATION"、或缺少 "DOCUMENT" 字段，则回答错误。请用一个词回答："CORRECT" 或 "INCORRECT"。

#### 6.1.6 K/V 任务指令

MemGPT 智能体被赋予以下人设，旨在鼓励它迭代搜索：

> 你是 MemGPT DOC-QA 机器人。你的工作是回答关于存储在你归档记忆中的文档的问题。用户问题的答案永远都在你的归档记忆中，所以如果找不到答案请记得继续搜索。在验证"该值不是键"之前，不要停止搜索。在满足该条件之前，不要停止嵌套查找。

基线则收到以下指令提示词：

> 下面是一个包含键值对的 JSON 对象，所有键和值都是 128 位 UUID，你的任务是返回与指定键关联的值。如果某个值本身也是键，则返回该键的值（做嵌套查找）。例如，若 "x" 的值是 "y"，而 "y" 也是一个键，则返回键 "y" 的值。

---

## 参考文献

（保留原文引用，格式从简）

- Beltagy et al. (2020). Longformer: The long-document transformer. arXiv:2004.05150.
- Borgeaud et al. (2022). Improving language models by retrieving from trillions of tokens. ICML.
- Brown et al. (2020). Language models are few-shot learners. NeurIPS.
- Chen et al. (2023). Extending context window of large language models via positional interpolation. arXiv:2306.15595.
- Child et al. (2019). Generating long sequences with sparse transformers. arXiv:1904.10509.
- Dai et al. (2019). Transformer-XL: Attentive language models beyond a fixed-length context. arXiv:1901.02860.
- Devlin et al. (2018). BERT: Pre-training of deep bidirectional transformers for language understanding. arXiv:1810.04805.
- Dong et al. (2023). A survey on long text modeling with transformers. arXiv:2302.14502.
- Guu et al. (2020). Retrieval augmented language model pre-training. ICML.
- Izacard & Grave (2020). Leveraging passage retrieval with generative models for open domain question answering. arXiv:2007.01282.
- Izacard et al. (2021). Unsupervised dense information retrieval with contrastive learning. arXiv:2112.09118.
- Jiang et al. (2023). Active retrieval augmented generation (FLARE). arXiv:2305.06983.
- Karpukhin et al. (2020). Dense passage retrieval for open-domain question answering. arXiv:2004.04906.
- Kitaev et al. (2020). Reformer: The efficient transformer. arXiv:2001.04451.
- Lee et al. (2019). Set Transformer. ICML.
- Lewis et al. (2020). Retrieval-augmented generation for knowledge-intensive NLP tasks. NeurIPS.
- Lin (2004). ROUGE: A package for automatic evaluation of summaries.
- Lin et al. (2023). RA-DIT: Retrieval-augmented dual instruction tuning.
- Liu et al. (2023a). Lost in the middle: How language models use long contexts. arXiv:2307.03172.
- Liu et al. (2023b). AgentBench: Evaluating LLMs as agents. arXiv:2308.03688.
- Nakano et al. (2021). WebGPT: Browser-assisted question-answering with human feedback. arXiv:2112.09332.
- Ouyang et al. (2022). Training language models to follow instructions with human feedback. NeurIPS.
- Park et al. (2023). Generative agents: Interactive simulacra of human behavior. arXiv:2304.03442.
- Patterson et al. (1988). A case for redundant arrays of inexpensive disks (RAID). SIGMOD.
- Press et al. (2021). Train short, test long: Attention with linear biases enables input length extrapolation. arXiv:2108.12409.
- Ram et al. (2023). In-context retrieval-augmented language models. arXiv:2302.00083.
- Schick et al. (2023). Toolformer: Language models can teach themselves to use tools. arXiv:2302.04761.
- Touvron et al. (2023). Llama 2: Open foundation and fine-tuned chat models. arXiv:2307.09288.
- Trivedi et al. (2022). Interleaving retrieval with chain-of-thought reasoning for knowledge-intensive multi-step questions. arXiv:2212.10509.
- Vaswani et al. (2017). Attention is all you need. NeurIPS.
- Wang et al. (2020). Linformer: Self-attention with linear complexity. arXiv:2006.04768.
- Wei et al. (2022). Chain-of-thought prompting elicits reasoning in large language models. NeurIPS.
- Xu et al. (2021). Beyond goldfish memory: Long-term open-domain conversation. arXiv:2107.07567.
- Yao et al. (2022). ReAct: Synergizing reasoning and acting in language models. arXiv:2210.03629.
- Zheng et al. (2023). Judging LLM-as-a-judge with MT-Bench and Chatbot Arena. arXiv:2306.05685.

---

## 术语与背景速览（译者整理）

| 术语 | 含义 |
| --- | --- |
| 虚拟上下文管理（virtual context management） | 借鉴 OS 虚拟内存思想，通过在有限上下文窗口与外部存储之间换入换出数据，制造"无限上下文"假象的技术 |
| 主上下文（main context） | LLM 的 prompt tokens，相当于 OS 的物理内存/RAM；包括系统指令、工作上下文与 FIFO 队列 |
| 外部上下文（external context） | 上下文窗口之外的所有数据，相当于磁盘，包括归档存储与召回存储 |
| 工作上下文（working context） | 固定大小、可读写的非结构化文本块，存放用户/人设的关键事实与偏好，只能经函数调用写入 |
| FIFO 队列 | 滚动消息历史（含系统消息、函数调用输入输出）；队首存放被逐出消息的递归摘要 |
| 队列管理器（Queue Manager） | 负责消息入队、写召回存储、上下文溢出告警与队列清空（逐出+更新递归摘要） |
| 函数执行器（Function Executor） | 解析并执行 LLM 输出的函数调用，把结果（含报错）反馈给 LLM |
| 归档存储（archival storage） | 存任意长度文本对象的读写数据库（实现为 PostgreSQL + pgvector 向量检索，HNSW 索引） |
| 召回存储（recall storage） | MemGPT 的消息数据库，永久保存全部消息历史 |
| 内存压力告警 / 清空阈值 | prompt tokens 达到上下文窗口约 70% 时告警，达 100% 时逐出约 50% 消息并更新递归摘要 |
| `request_heartbeat=true` | 函数调用中的特殊标志：请求函数执行完后立即再次触发 LLM 推理，实现函数链（多步检索）；缺省则 yield，等待下一个事件 |
| DMR（深度记忆检索） | 基于 MSC 的问答任务：问题指向此前会话细节，答案取值范围极窄，测一致性 |
| MSC（Multi-Session Chat） | 多会话聊天数据集，标注员保持一致人设进行 5 个会话的对话 |
| CSIM / SIM-1/3 / SIM-H | 开场白与金标人设的余弦相似度评分（SIM-1 取最高、SIM-3 取前三平均），SIM-H 为与人工开场白的相似度 |
| 嵌套 KV 检索 | KV 任务的变体：值本身可为键，需多跳查找；本文设 140 对 UUID（约 8k tokens），嵌套 0–4 层 |
| Lost in the middle | Liu et al. 2023a 的发现：长上下文模型对窗口开头/结尾信息召回好、对中间信息差 |

**核心数字速记**

- DMR 准确率：GPT-4 基线 32.1% → MemGPT 92.5%；GPT-4 Turbo 35.3% → 93.4%；GPT-3.5 38.7% → 66.9%
- 开场白 SIM-1：GPT-4 + MemGPT 0.868，超过人工开场白的 0.800
- 文档问答（50 题，Wikipedia 2018 dump）：定长基线性能被封顶在检索器水平、K 增大后因截断而下降；MemGPT 基本持平不受 K 影响
- 嵌套 KV：GPT-3.5 在 1 层嵌套即 0%；GPT-4/GPT-4 Turbo 在 3 层嵌套时 0%；MemGPT(GPT-4) 全层数保持 100%
- 上下文窗口跨度：Llama 2k → GPT-4 Turbo 128k → Yi-34B-200k 200k（表 1，2024-01 数据）
