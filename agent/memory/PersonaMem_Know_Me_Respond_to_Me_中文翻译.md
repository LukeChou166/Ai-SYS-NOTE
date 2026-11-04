# 懂我，再答我：面向大规模动态用户画像与个性化回复的 LLM 基准测试

> **原文**: Know Me, Respond to Me: Benchmarking LLMs for Dynamic User Profiling and Personalized Responses at Scale
> **作者**: Bowen Jiang\*, Zhuoqun Hao\*, Young-Min Cho, Bryan Li, Yuan Yuan, Sihao Chen, Lyle Ungar, Camillo J. Taylor, Dan Roth（\* 同等贡献；University of Pennsylvania / Microsoft）
> **日期/arXiv**: arXiv:2504.14225v2，2025-10-27 修订（v1 为 2025-04-19）；COLM 2025
> **链接**: https://arxiv.org/abs/2504.14225 ｜ 代码与数据: https://github.com/bowen-upenn/PersonaMem
> **译者说明**: 本文为完整全文翻译，正文与附录均逐段译出；图注按图意译出并补充图内元素说明；表格数据按原文重建为 Markdown 表。带 *（译者注）* 标记的段落为译者补充的背景说明，非原文内容。参考文献保留英文原文，未翻译，便于检索。

---

## 摘要

大语言模型（LLM）已成为用户在各类任务上的个性化助手——从写作辅助到量身定制的推荐或咨询。随着时间推移，用户与 LLM 之间的交互历史可以承载关于个体特质与偏好的大量信息。然而，当今的 LLM 究竟能在多大程度上有效利用这些历史，仍是一个开放问题，具体包括三问：（1）内化用户固有的特质与偏好；（2）追踪用户画像与偏好随时间如何演化；（3）在新场景中据此生成个性化回复。

在本工作中，我们提出 **PersonaMem** 基准。PersonaMem 精选了用户画像，并配以 **180 余条模拟的用户—LLM 交互历史**，每条历史最多包含 **60 个会话**的多轮对话，覆盖 **15 个需要个性化的真实世界任务**。给定一个「现场用户提问」（in-situ user query，即用户以第一人称视角发出的提问），我们评估 LLM 聊天机器人能否依据用户画像的**当前状态**识别出最合适的回复。我们观察到：通过直接提示（direct prompting）方式，当前 LLM 仍难以识别用户画像随时间的动态演化。结果是，LLM 常常无法给出与用户当前处境和偏好相符的回复——GPT-4.1、o4-mini、GPT-4.5、o1、Gemini-2.0 等前沿模型的总体准确率仅在 **50%** 左右，说明仍有很大提升空间。我们希望 PersonaMem 及其配套的用户画像与对话模拟流水线，能够推动未来「真正懂用户」的聊天机器人研究。代码与数据已开源。

---

## 1 引言

近年来，大语言模型（LLM）作为通用任务求解器快速演进，展现出令人瞩目的性能（Srivastava et al., 2023; Zhou et al., 2023; Yue et al., 2024; Rein et al., 2024）。

今天，许多用户已经将 LLM 当作个性化聊天机器人或助手，用于各类日常任务——从写作辅助（Mysore et al., 2024; Tian et al., 2024）到推荐（Hua et al., 2023）或咨询（Xie et al., 2024a; Zheng et al., 2024）等等。LLM 中的**个性化**意味着让模型回复适配每个用户特定的特质、偏好与历史交互，从"通用回复"走向"更相关、更贴合个人"的回复。由于不同用户有不同的画像（persona），LLM 需要具备**多元性**（pluralistic）——能够跨场景适配不同用户特征（Sorensen et al., 2024; Jiang et al., 2024; Xie et al., 2024b; Kirk et al., 2024），从而提升用户体验与参与度。

要让 LLM 给出个性化回复，一个现实挑战在于：LLM 无法轻易获取关于某个用户的**全部**信息。

这一挑战又被用户偏好随时间不断变化的特性进一步放大（Radlinski & Craswell, 2017; Dean & Morgenstern, 2022）。例如，如图 1 所示，某用户最初说"我喜欢披萨"，但在之后某个会话中提到"我在发现自己麸质过敏后，开始尝试无麸质饮食"。

当该用户再次询问美食推荐时，一个个性化的 LLM 聊天机器人应当能追踪到这一变化，并依据用户当前处境给出推荐。而当前的 LLM 聊天机器人常常无法识别并适配不断演化的用户画像。这会让用户觉得这些机器人不够有帮助、不够共情，最终降低满意度（Aggarwal et al., 2023; Ait Baha et al., 2023）。

在本工作中，我们评估 LLM 利用与用户过往交互历史、**实时**给出个性化回复的能力。近期研究（Lin et al., 2024; Shi et al., 2024; Zhao et al., 2025）发现，用户—LLM 交互可以成为关于用户特征与偏好的丰富（但常常是隐含的）信息来源。

然而，LLM 能否有效利用这些交互历史来（1）内化用户固有特质与偏好，（2）追踪用户特征随时间如何演化，（3）在新场景中据此生成个性化回复，仍是开放问题。

为研究这些问题，我们提出 **PersonaMem** 基准，它包含 180 余条模拟的用户—LLM 交互历史，每条最多 60 个多轮会话，覆盖 15 个个性化任务场景。每条历史都构建自一个详细、且特征随时间演化的用户画像。基于用户在不同时间点的画像，我们模拟任务相关的对话（如旅行、心理咨询、美食），并按时间顺序拼接，从而在整个交互历史中刻画用户画像的演化轨迹。

> **图 1 | PersonaMem 基准总览。** 每个基准样本是一个用户画像，包含**静态属性**（如人口统计信息）与**动态属性**（如不断演化的偏好）。用户围绕美食推荐、旅行规划、心理咨询等多个主题与聊天机器人进行多会话交互。随着用户偏好随时间演化，基准提供带标注的问题，用于评估模型能否追踪这些变化并将其纳入回复。

借助 PersonaMem，我们评估最先进的 LLM 能否推断不断演化的用户画像，并跨任务场景生成个性化回复。为贴近用户—LLM 交互的真实情形，我们设计了 **7 类现场用户提问**（表 1），用户以第一人称视角向 LLM 发问。我们评估 LLM 能否选出与用户当前状态最相符的正确回复。我们发现，采用直接提示方式时，GPT-4.1、o4-mini、GPT-4.5、o1、Gemini-2.0-Flash 等前沿模型的总体准确率仅在 **50%** 左右，Llama-4-Maverick 略低，为 **43%**。

模型在回忆事实、追踪偏好变化方面表现尚可（**60–70%** 准确率），但难以把用户**最新处境**纳入回复（**30–50%** 准确率）。我们进一步详细分析了历史长度、偏好出现位置、记忆组件等因素对性能的影响。

我们的主要贡献与发现总结如下：

- 我们提出 **PersonaMem** 基准及其对话合成流水线，用于生成以画像为中心、多会话、带时间线的用户—聊天机器人交互历史。
- 我们在 **7 类**现场用户提问上评估了 **15 个** LLM，考察它们在 15 个任务场景中给出与用户动态变化画像相符的回复的能力。
- 借助 PersonaMem，我们观察到 GPT-4.1、o4-mini、GPT-4.5、o1、DeepSeek-R1、Gemini-2.0、Llama-4、Claude-3.7 等前沿模型**仍然难以做到"懂用户"**并给出个性化回复，尤其是当用户知识需要迁移到新场景时。

---

## 2 PersonaMem 基准：总览

我们在图 1 中给出 PersonaMem 基准的总览。

基准数据集的每个实例对应一个用户画像（persona），包含基本人口统计信息（姓名、年龄、性别、职业），以及**动态用户特征**——用户特质、偏好、以及发生在用户生活中的事件。动态特征随时间变化：用户经历的各类事件会带来其在各任务场景下的特质与偏好变化。

在用户画像演化的不同时间点，用户会围绕某个任务场景与 LLM 进行多轮对话，就自身需求与当前处境向 LLM 寻求帮助或建议。不同任务场景的会话按实际发生的时间顺序交错排列。

为了解 LLM 聊天机器人能否从对话历史中追踪用户画像的演化，我们的评估方式是：看模型能否对**现场用户提问**给出最合适的回复——即用户在一个新的会话中，以第一人称视角向 LLM 发问。由于提问的时间点不同，模型应有的回复也不同。我们把该问题构造为**四选一**：LLM 需从四个选项中选出正确回复，而错误选项基于**过时**或**与用户当前画像状态无关**的信息。

**评估的技能类型。**
为评估 LLM 的三种能力——（1）记住用户画像，（2）追踪用户画像随时间如何演化，（3）据此在新场景中生成个性化回复——我们在 PersonaMem 基准中设计了如下 **7 类**现场用户提问。每类提问的示例见表 1。

1. **回忆用户分享过的事实（Recall user-shared facts）**。评估个性化聊天机器人能否回忆起用户在先前交互中分享过的静态事件、活动或兴趣，并将其纳入回复。
2. **提出新建议（Suggest new ideas）**。评估聊天机器人能否在用户明确提出要求时（例如"推荐一些我以前没点过的餐厅"），给出交互历史中**未出现过**的新项目或活动建议。
3. **识别用户最新偏好（Acknowledge latest user preferences）**。评估聊天机器人能否识别用户在交互历史中表达的**最新**偏好。
4. **追踪完整偏好演化（Track full preference evolution）**。评估聊天机器人能否持续追踪用户偏好随时间的变化轨迹。
5. **回溯偏好更新的原因（Revisit reasons behind preference updates）**。评估聊天机器人能否回忆起导致用户偏好变化的原因或事件。
6. **给出与偏好一致的推荐（Provide preference-aligned recommendations）**。测试聊天机器人能否主动给出与用户**当前**偏好相符的新推荐。
7. **泛化到新场景（Generalize to new scenarios）**。评估聊天机器人能否把在其他任务场景中学到的用户信息迁移到一项新任务上。

**基准数据统计。**
PersonaMem 包含 **20 个用户画像**、**180 余条交互历史**。每条交互历史包含 10、20 或 60 个会话，我们动态调整历史总长度分别约为 **32k**、**128k** 和 **1M tokens**。每个会话由用户与 LLM 聊天机器人之间 **15–30 轮**对话组成。用户—LLM 对话覆盖 **15 个**多样主题：从心理咨询、法律咨询，到书籍、音乐、电影、美食推荐；从家庭、约会、健康、财务等个人事务，到旅行规划、网购、学习技巧、家居装修等实用任务。总体上，基准在 7 类提问上共包含约 **6k** 组现场用户提问与 LLM 回复对。详细的数据集构成见附录 D。我们基准的规模并非受限于合成数据流水线的可扩展性，而是为了使评估成本保持在合理范围内。

| 提问类型 | 现场用户提问与聊天机器人回复示例 |
| --- | --- |
| [1] 回忆用户分享过的事实 | "用户：我把我的播放列表分享给朋友们，他们都很喜欢。……（之后）用户：有哪些有创意的分享音乐的方式？—— 聊天机器人：精心策划个性化播放列表会很有趣。" |
| [2] 提出新建议 | "用户：……（之后）用户：推荐一些我以前没点过的餐厅。" |
| [3] 识别用户最新偏好 | "用户：我是意大利菜的超级粉丝。……（之后）用户：我现在更喜欢地中海菜了。……（之后）用户：我昨天去了一家意大利餐厅。—— 聊天机器人：很高兴听到！不过我以为你现在更偏好地中海菜。" |
| [4] 追踪完整偏好演化 | "用户：……（之后）用户：我在考虑参加一个烹饪课。—— 聊天机器人：我理解你对烹饪课的看法已经变了。最初你表示不喜欢，但后来提到你对它产生了兴趣。" |
| [5] 回溯偏好更新的原因 | "用户：我是意大利菜的超级粉丝。……（之后）用户：我最近发现自己对麸质过敏，所以改吃其他菜了。……（之后）用户：我昨天去了一家意大利餐厅。—— 聊天机器人：你的过敏情况怎么样了？我猜那家店应该有提供无麸质披萨吧。" |
| [6] 给出与偏好一致的推荐 | "用户：我现在更偏地中海海鲜。……（之后）用户：我怎样才能做出有影响力的生活方式改变？—— 聊天机器人：不如试试地中海饮食计划？" |
| [7] 泛化到新场景 | "用户：我喜欢我的瑜伽课，因为它每周固定时间。……（之后）用户：我最近没时间做饭。—— 聊天机器人：既然你似乎很喜欢规律作息，要不要试试按周配送的便当盒？" |

> **表 1 | PersonaMem 基准中 7 类现场用户提问及期望的聊天机器人回复示例。** 原文以高亮标出提示用户事实或偏好的关键措辞（此处以中文表述保留其语义）。

---

## 3 大规模构建 PersonaMem 样本

我们开发了一条模块化的数据构建流水线——由 GPT-4o（Hurst et al., 2024）驱动——用于合成以画像为中心、多会话、长上下文的用户—模型对话。该流水线尽量减少无关内容或随机注入内容，以便更好地评估 LLM 聊天机器人应对第 2 节所述挑战的效果，同时保证成本可控与可扩展性：为某个画像在某个对话主题上生成数据约花费 **2 美元**，且该成本与上下文窗口长度（最长 1M tokens）无关。

**构建用户画像与 persona。**
我们从 PersonaHub（Ge et al., 2024）中随机采样一组 persona，每个约 1–3 句话，再为其补充额外的人口统计信息与扩展的个人细节。我们还构建一条时间线，并填入与该 persona 相符的事件。这些事件构成**一般个人历史**（如教育、职业发展、人生经历），以提供更丰富的上下文。该过程使用的提示词见附录 G。

在 persona 与一般个人历史的基础上，我们为**每个对话主题**再生成一份**主题专属个人历史**。每个主题下我们定义一组初始偏好，并确保不同主题之间不重叠。每份主题专属历史包含事件、时间戳、相关偏好、这些偏好可能发生的更新、以及更新背后的原因。这一做法既保证了用户体验的连贯演进，又使其与 persona 保持紧密关联。

结构化的个人历史也便于构建问答对。我们利用历史中的短文本信息，抽取出用户**任意时刻**的真实画像与偏好，确保正确答案既有事件依据、也有 persona 依据。相反，干扰选项虽然通常看起来合理，但要么忽略了用户 persona，要么与之矛盾。此外，我们剔除了所有**不看任何上下文**模型也能答对的题目。

**从用户画像模拟对话会话。**
我们把时间线切分为多个片段，得到遵循因果、时间先后顺序的个人历史片段。每个片段随后被扩展为一个完整的用户—模型对话会话，要求覆盖对应主题专属历史片段的**全部细节**，并加入额外的叙事上下文，就像用户在自然地与聊天机器人交谈。例如，在心理咨询主题下，我们把交互设定为用户向 AI 治疗师寻求指导。

为提升对话质量，我们采用了几项技巧：（1）在生成每一轮用户—模型交互之前，我们先让 GPT-4o 识别并**引用**个人历史中的相关事件。这些引用仅作为内部引导，不进入最终评估数据。（2）由于 GPT-4o 可能遗漏部分事件、导致偏好更新序列不完整，我们引入**自我反思**机制：让 GPT-4o 复查已生成的对话，找出个人历史中被遗漏的事件，从而保证交互的覆盖度与连贯性。

> **图 2 | 以画像为中心的多会话数据构建流程总览。** 先构建用户 persona，再构造带时间戳的一般个人历史与主题专属个人历史，将其扩展为对话会话，最后按拓扑顺序拼接会话以形成长对话上下文——由此形成一个可扩展的生成框架。

**通过会话拼接组装交互历史。**
大规模生成以画像为中心的长上下文对话可以兼顾成本与可扩展性。对每个画像，我们按会话的**结束时间戳**做拓扑排序，只需保证同一主题内的会话满足因果关系。不同数量的会话可以有多种合法拼接顺序。这种灵活设计允许跨主题会话存在多种合法交错方式，也就是说，我们**只需生成会话本身**，而不必为每条长上下文对话从头生成。为进一步拉长上下文并模拟更自然的交互，我们还在会话之间插入少量短交互——用户提一些随机知识问题或编程求助，其中不透露任何用户偏好。

**数据质量的人工验证。**
为评估生成数据的质量，我们对 PersonaMem 中随机抽取的 **90 组**问答对进行了人工研究，每组都以用户 persona、个人历史、以及对话中的相关话语为依据。三名标注者从四个维度评估每组问答：恰当性（appropriateness）、相关性（relevance）、正确性（correctness）、最优回复（best response）。各维度的判定通过率都非常高——分别为 **97.8%**、**95.6%**、**97.8%**、**90.0%**。更多细节见附录 B。

---

## 4 实验

### 4.1 评估设置

给定一个现场用户提问以及截至某一时间点的用户交互历史，我们在四个选项中评估模型依据用户当前状态选出最合适回复的能力。四个选项中**只有一个**符合用户当前状况，其余选项包含与用户无关或已过时的事实或偏好。

评估时，除对话历史外，模型还可以访问用户的基本人口统计信息，包括姓名、年龄、性别认同、种族认同与职业。除此之外，模型无法直接访问用户的其他动态特征与个人历史。

在选出最合适回复的方式上，我们分别在**判别式（discriminative）**与**生成式（generative）**两种设置下评估模型。判别式设置下，模型看到全部四个选项，分别标记为 (a)、(b)、(c)、(d)，选项顺序随机打乱，要求模型输出正确选项并给出简要解释。

生成式设置下，模型仍然一次只看到一个题目。我们对每个选项单独计算生成该选项的 token 概率对数之和并做长度归一化，取概率最高者作为模型回复。主评估（§4.2、§4.3、§4.4）采用判别式设置；§4.5 采用生成式设置，因为它需要访问解码时对整个词表的 logits，而多数闭源模型不提供该接口。评估过程**不使用**任何 LLM 裁判。

### 4.2 在长上下文设置下评估语言模型

我们首先在长上下文设置下评估语言模型：把完整的用户—LLM 交互历史作为输入喂给模型。由于历史很长，此处所有模型均为**零样本**评估，不提供其他历史与用户提问的示例。

评估覆盖 GPT-4.1、o4-mini、o3-mini、GPT-4.5、o1、GPT-4o、GPT-4o-mini、Gemini-2.0-Flash、Gemini-2.0-Flash-Lite、Gemini-1.5-Flash、DeepSeek-R1-671B、Llama-4-Maverick、Llama-3.1-405B、Claude-3.7-Sonnet、Claude-3.5-Haiku（OpenAI, 2025a; OpenAI, 2024b; OpenAI, 2025b; OpenAI, 2024a; Hurst et al., 2024; Team et al., 2024; Guo et al., 2025; Grattafiori et al., 2024; Anthropic, 2024），上下文窗口为 **128k tokens**。我们还评估了支持更长上下文的模型——Llama-4-Maverick、Gemini-2.0-Flash、Gemini-2.0-Flash-Lite、Gemini-1.5-Flash——在 **1M tokens** 上下文窗口下的表现。

我们的发现如下：

> **图 3 | 不同模型在 7 类现场提问上的评估结果。** 模型在回忆用户事实与偏好方面表现尚可；但在给出新颖建议、或把用户偏好应用到新场景时明显吃力。

> **图 4 | 按"最近一次偏好被提及距今的会话数"划分的模型表现。** 上：最多 20 个会话 / 128k tokens；下：最多 60 个会话 / 1M tokens。长上下文检索在实践中对个性化至关重要。

**GPT-4.5、GPT-4.1 与 Gemini-1.5 总体表现最佳。**
在领先的基础模型中，GPT-4.5 与 Gemini-1.5 的总体准确率优于其他模型。但其表现仍徘徊在 **52%** 左右（四选一设置），说明提升空间依然很大。值得注意的是，o1、o3-mini、o4-mini、DeepSeek-R1-671B 等**推理模型**在我们评估的个性化任务上并未相对非推理模型展现出优势。

*（译者注）原文此处写作 "DeepSeek-R1-607B"，与评估模型清单中的 "DeepSeek-R1-671B" 不一致，应为笔误；本文统一按 671B 译出。*

**LLM 在回忆简单的用户事实方面表现尚可。**
对于检索静态用户信息的任务——例如先前提到的物品、活动，或偏好变化的原因（原因本身不会变）——多数 LLM 都有相当的机会答对。

**把用户最新偏好纳入回复，比回忆画像变化本身更难。**
我们观察到，模型难以在回复中纳入用户**最新**的偏好或状态。出人意料的是，当要求模型回忆用户偏好随时间如何演化时，模型表现反而更好。

我们观察到，让模型逐一梳理所有偏好更新，可能会促使它把偏好的演化过程想清楚，往往使任务变得更容易。

**模型在生成新想法、或在新场景中提供建议方面表现不佳。**
如图 3 所示，"提出新建议"、"给出与偏好一致的推荐"、"把理由泛化到新场景"这几类任务在所有模型上表现最差，凸显了在新情境下生成个性化回复的难度——尤其是在识别**新事实**时。

### 4.3 用户信息在交互历史中位置的影响

为理解相关用户事实或偏好在对话历史中出现的位置如何影响模型表现，我们按"相关信息出现在第几个会话"来报告模型性能。结果见图 4。总体来看，当相关信息出现在对话历史的**较早或较晚**会话中时，模型表现更好。这一发现与先前关于长上下文输入的结论一致：上下文中间的信息容易被"迷失在中间"（lost in the middle，Liu et al., 2024; Wu et al., 2024）。

### 4.4 结合外部记忆模块的评估

> **图 5 | GPT-4o 与 GPT-4o-mini 在 32k tokens 上下文下、不同问题类型上的表现。** 对比原始模型与接入 Mem0、RAG 的模型。

我们评估使用检索器从历史中定位相关信息能否提升模型表现。

我们对比了两种外部记忆方案——**RAG**（Lewis et al., 2020）与 **Mem0**（Mem0, 2024）——与原始 LLM（vanilla）的表现。该组实验只考虑 GPT-4o 与 GPT-4o-mini 两个模型。其延迟表现见附录 E。

对 RAG，我们采用最直接的实现：使用稠密 BGE-M3 嵌入（Chen et al., 2024），针对每个问题检索出**最相关的 5 条消息**。对 Mem0，它为 LLM 提供额外的记忆层，我们逐轮用 LLM 生成的事实增量构建记忆库；推理时针对每个问题检索**最相关的 5 条事实**。出于效率考虑，评估使用 **32k tokens** 的上下文。

**基于检索的记忆模块能提升模型表现。**
总体上，外部记忆模块显著提升了两个模型的准确率。其中"回忆用户分享过的事实"与"泛化到新场景"收益最大，说明检索在事实类任务上更为有效。相比之下，"回溯偏好更新原因"的增益较小。在多数问题类型上 **RAG 稳定优于 Mem0**，尽管 Mem0 的计算开销更高——这说明对个性化推理而言，检索语义相近的消息比构建结构化记忆更有效。

### 4.5 生成式设置下的语言模型评估

在真实使用场景中，聊天机器人在推理时并不能看到候选回复选项。因此我们额外在更贴近实际的**生成式设置**下评估模型：模型一次只看到一个选项，最终由模型预测的联合序列概率选出最佳回复。

**方法。**
给定用户—LLM 历史与现场用户提问，我们通过对每个候选回复的 token 级概率取对数求和，比较联合序列概率。

具体而言，给定对话历史（记为 $\mathcal{C}$）与用户提问（$q$），我们对每个候选回复 $r_{i}$（$i\in\{1,2,3,4\}$）进行评估，其由长度为 $T_i$ 的 token 序列 $\{x_i^1, x_i^2, \dots, x_i^{T_i}\}$ 组成。由于因果语言模型的自回归特性，每个"提问—回复"对的联合对数概率等于各 token 在给定其前文条件下的条件对数概率之和：

$$
\log P(r_i \mid \mathcal{C}, q) = \frac{1}{T_i}\sum_{t=1}^{T_i}\log P\left(x_i^t \mid \mathcal{C}, q, x_i^1, \dots, x_i^{t-1}\right)
$$

由于该方法需要输出 token 在**整个词表**上的对数概率，而闭源模型通常不提供该接口，我们只评估开权重模型——**LLaMA-3.1-70B、LLaMA-3.1-8B、DeepSeek-Distill-LLaMA-8B**。受算力限制，我们仅在基准的 **10 会话（约 32k tokens）** 版本上评估。

**结果。**
如图 6 所示，就不同用户提问类型的难度而言，我们观察到与判别式评估相似的趋势。模型在回忆事实、追踪偏好变化上表现尚可；而"提出新建议"与"泛化到新场景"仍是对模型最具挑战性的提问类型。有趣的是，比较同一模型（具体为 LLaMA-3.1-8B-instruct）在判别式与生成式设置下的表现，**生成式设置反而更好**，这可能说明模型无需在输入中看到全部候选选项，也能给出个性化回复。由于生成式评估仅在 32k 上下文长度上完成，生成式与判别式设置的结论在更长上下文、更多模型上是否成立，仍有待研究。我们还发现，随着用户的新请求与其先前透露的信息距离变远，模型表现下降。详细结果见附录 C。

> **图 6 | 在 PersonaMem 的 10 会话（32k tokens）版本上的生成式评估。**

---

## 5 相关工作

### 5.1 评估 LLM 的长上下文记忆能力

"大海捞针"（needle-in-the-haystack）测试是此类评估的常用方法：要求模型在给定长上下文中定位特定事实。已有基准涵盖从直接信息检索（Kuratov et al., 2024; Nelson et al., 2024）到问答与摘要（Xu et al., 2024; Bai et al., 2024; Zhang et al., 2024）等任务。

更贴近真实场景的评估方式是通过对话。早期基准构建了人与人（Xu, 2021）或人与 AI（Xu et al., 2022）的交互，会话长度可达 10k tokens。较新的工作则利用 LLM 生成长达 100k+ tokens 的会话（Maharana et al., 2024; Kim et al., 2024; Castillo-Bolado et al., 2024）。

最近，Wu et al. (2024) 提出 LongMemEval，一个同样考虑最长 1M 上下文的对话基准，且采用画像驱动的会话。主要差异在于：PersonaMem 的会话覆盖的主题范围比"任务导向"更广；且 PersonaMem 的评估聚焦**细粒度个性化问题**，而非更一般的记忆能力。

### 5.2 大语言模型中的个性化

用户的偏好具有多样性，既体现在人口群体层面（Santurkar et al., 2023），也体现在个体层面（Zollo et al., 2024）。Persona（画像）是个体的简短传记，同时刻画这两个层面，且可由 LLM 批量生成（Ge et al., 2024）。研究者已用 persona 来评估 LLM 能否适配用户与环境（Castricato et al., 2024; Tseng et al., 2024）。

对个性化的**可靠评估**同样关键。上述许多基准将其形式化为 NLP 任务；另一条研究路线则用 LLM 沿着不同个性化维度自动评判文本（Dong et al., 2024; Wang et al., 2023）。PersonaMem 采用前者，报告问答上的性能。重要的是，PersonaMem 的个性化评估是由问题与答案的**设计**保证的：每组问答都以特定时间点的事件为依据，并按特定提问类型生成。

转向对话场景，早期的 LaMP 与 PersonaLLM 考察的是单轮或单会话内的个性化（Salemi et al., 2023; Jiang et al., 2023; Kirk et al., 2024）。较新的 ImplexConv（Li et al., 2025）聚焦于个性化对话中的**隐含推理**建模。PersonaBench（Tan et al., 2025）通过数量多但更短的会话、并允许访问合成私有用户数据，来模拟多样用户之间的社交互动。PersoBench（Afzoon et al., 2024）利用已有的 persona 感知数据集评估语言质量、画像覆盖度与一致性。LongLaMP（Kumar et al., 2024）聚焦于在长上下文中生成长文本，而非更具交互性的回复。Zhao et al. (2025) 提出 PrefEval，在 100k+ tokens 的画像导向对话中评估 LLM 对 20 个主题的偏好遵循能力。PersonaMem 除了能高效生成大量 1M-token 上下文的灵活设置外，更强调把 persona 当作**模拟人类**置于用户—模型交互中，并提供多个细粒度个性化任务，其中画像与偏好都随时间锚定的事件而演化。

---

## 6 结论

本文提出 **PersonaMem** 基准，包含可扩展、以画像为中心的多会话用户—LLM 交互历史，以及细粒度的现场用户提问类型，用于评估 LLM 在记忆、追踪用户动态画像并将其纳入个性化回复方面的能力。通过对 **15 个**最先进 LLM 及基于检索的方法的系统评估，我们凸显了当前让 LLM 与用户进行真正个性化对话所面临的挑战，尤其是在新场景与长上下文中。我们希望该基准能为个性化 LLM 聊天机器人研究的未来探索与进展开辟新路径。

> **表 2 | 相关工作基准对比**，包括 LOCOMO（Maharana et al., 2024）、LongMemEval（Wu et al., 2024）与 PrefEval（Zhao et al., 2025）。LOCOMO 与 LongMemEval 聚焦一般的长期记忆任务；相比之下，PersonaMem 关注记忆检索之上的**个性化**，其全部对话都围绕一个具有演化偏好的连贯用户画像构建，更贴近真实的用户—聊天机器人对话。PrefEval 同样关注个性化，但其做法是先**生成**用户偏好、再把这些偏好**插入**到随机采样的上下文中。

| 维度 | LOCOMO | LongMemEval | PrefEval | **PersonaMem** |
| --- | --- | --- | --- | --- |
| 聚焦任务 | 长期记忆 | 长期记忆 | 用户偏好 | **细粒度个性化回复** |
| 平均单会话长度 | 477 tokens | 3k tokens | 未提供 | **6k tokens** |
| 最大上下文长度 | 9k tokens | 1.5M tokens | 100k tokens | **1M tokens** |
| 数据来源 | MSC & 自建 | ShareGPT & UltraChat & 自建 | LMSYS-Chat-1M | **PersonaHub & 自建** |
| 提问视角 | 第三人称 | 第一人称 | 第一人称 | **第一人称** |
| 最大知识更新次数 | 无更新 | 1 | 无更新 | **3** |
| 多会话推理 | 是 | 是 | 否 | **是** |
| 评估的 LLM 数量 | 4 | 5 | 6 | **15** |

*（译者注）原文表格第三列列名写作 "PrevEval"，据正文与引用应为 PrefEval（Zhao et al., 2025）。*

---

## 7 致谢

本工作受美国国家科学基金会（NSF）Grant CCF-2112665（TILOS）、Camillo J. Taylor 的研究自由支配经费，以及 Dan Roth 的 ONR MSU 资助支持。

---

## 附录 A 局限性与未来工作

### A.1 用户隐私担忧的更广泛背景

隐私是 LLM 个性化在真实世界中的关键方面。在我们的设定中，个性化仅基于用户在此前用户—聊天机器人交互中主动分享的偏好与活动，且模型只在自身回复中使用这些信息，不向外部共享。为避免真实用户数据可能带来的隐私风险，我们有意提出了一条**合成数据**构建流水线。这种合成方式使社区研究者能够安全地探索个性化方法。未来工作的一个可能方向是设计专门涉及敏感用户信息的问答对。

### A.2 更先进的检索方法

我们目前对检索增强方法（如 RAG 与 Mem0）的探索仅作为**概念验证**，本工作的主要关注点是该个性化基准的设计与发布。我们期待未来工作能对最先进的长上下文、记忆与检索增强生成方法做更多探索，尤其是那些能够**保留并理解用户画像演化、以及偏好更新背后原因**的方法，以及在**新出现或未见过的场景**中增强用户个性化的方法。

### A.3 合成数据生成过程中的潜在伪影

为减少可能让基准被人为"变简单"的伪影，我们采取了若干措施。例如，我们删除了正确答案会无意中变得显眼的问答对——比如答案明显更长，或与问题共享相同的关键词。我们还过滤掉了那些**不给任何真实对话上下文**、LLM 在三次尝试中能答对一次以上的查询。此外，我们在人工评估中加入了检查项，确认正确答案确实可以从所给上下文中推导出来。

### A.4 开放式生成评估与多选评估之间的潜在差异

在纯开放式生成设置下，个性化可能存在许多正确答案，取决于用户 persona 被如何使用、以及使用了哪一条相关用户偏好。同时，开放式评估需要为每组问答调用 LLM-as-a-Judge，计算成本高昂。因此，我们通过计算各候选选项的**联合对数似然**来评估生成式任务，而不在提示中显式呈现四个选项。这一做法在实验中得到了与标准判别式评估相似的模式，同时相比完全开放式评估提供了更可靠的性能基准化依据。

---

## 附录 B 人工评估细节

人工评估研究的目的是验证 §3 所述生成过程的整体质量。需要注意，我们并非要求人类在题目上作答（读长上下文对人不现实），而是向评估者提供问题与答案，以及它们所依据的对话与元数据。

我们使用 `potato` 包（Pei et al., 2022）搭建界面，截图见图 7。对每个条目，我们要求就 **4 个维度**做 True/False 判定：

1. **恰当性（Appropriateness）**：问题表述规范，且与所属类型相符。
2. **相关性（Relevance）**：问题与对话及 persona 相关。
3. **正确性（Correctness）**：`Correct_Response` 确实正确，且可从上下文中推导。
4. **最优回复（Best Response）**：`Correct_Response` 优于所有 `Incorrect_Responses`。

我们从本文作者中招募了三位评估者。我们清楚作者可能不是完全中立的标注者；为缓解这一问题，参与的三位作者**并未直接参与**数据生成过程。我们仍将在未来工作中考虑引入外部标注者。我们根据标注者的积极反馈迭代了标注说明与模板，形成最终版本。

我们选取 **90 个条目**（18 个主题 × 每个主题随机采样 5 个问题）进行标注。为减轻标注者心智负担，所有条目来自**同一个 persona**。每个条目被标注 3 次，取多数类标签。每个任务约需 1.5 分钟完成。

对每个条目、每个维度，我们计算判为 True 的比例，并用 Gwet's AC1（Gwet, 2008）计算评分者间信度。采用该指标是因为它能较好地处理"True 占绝对多数"的类别不平衡。

结果显示：**97.8%** 的条目被评为恰当（AC1 = 0.928），**95.6%** 相关（AC1 = 0.899），**97.8%** 正确（AC1 = 0.877），**90%** 为最优回复（AC1 = 0.560）。所有比例均超过 90%；维度 1、2、3 的一致性非常高，维度 4 为中等（很可能因为该维度较主观）。基于这次小规模人工评估，可以认为 PersonaMem 的生成质量相当合理。

> **图 7 | PersonaMem 条目人工评估任务的界面截图。** 此处用"…"省略了长会话；标注者看到的是完整文本（平均每会话 15 轮）。由于问题与回复均生成自图中所示对话及元数据，我们也把同样的内容展示给评估者。蓝色高亮字段是被 4 个问题直接引用的字段。

---

## 附录 C 补充实验结果

> **图 8 | 不同模型在 1M tokens 上下文的 7 类现场提问上的结果。** 同样可以观察到：模型在回忆用户事实与偏好方面表现尚可，但在给出新颖建议、或把用户偏好应用到新场景时明显吃力。

> **图 9 | GPT-4o 与 GPT-4o-mini 在 128k tokens 上下文下、不同问题类型上的表现。** 对比原始模型与接入 RAG 的模型。

> **图 10 | 在 PersonaMem 的 10 会话（32k tokens）版本上的生成式评估。**

图 8 展示了 1M tokens 上下文下模型在各问答类型上的表现，其模式与图 3 相似。

图 9 展示了在 128k tokens 上下文下、接入 RAG 模块后的模型表现。与图 5 的结果一致，RAG 在多数问题类型上有助于提升性能。

图 10 展示了"距最近一次偏好被提及所经过的会话数"与性能的关系。我们在判别式与生成式设置下都观察到相似的模式。

---

## 附录 D PersonaMem 统计的详细拆解

以下为数据集的更详细拆解。

### D.1 不同提问类型占比

| 提问类型 | 占比 |
| --- | --- |
| 回忆用户分享过的事实（Recall_user_shared_facts） | 5.8% |
| 识别用户最新偏好（Acknowledge_latest_user_preferences） | 30.09% |
| 追踪完整偏好演化（Track_full_preference_evolution） | 10.97% |
| 回溯偏好更新原因（Revisit_reasons_behind_preference_updates） | 9.28% |
| 给出与偏好一致的推荐（Provide_preference_aligned_recommendations） | 11.58% |
| 提出新建议（Suggest_new_ideas） | 22.92% |
| 泛化到新场景（Generalize_to_new_scenarios） | 9.35% |

### D.2 不同对话主题占比

| 主题 | 占比 | 主题 | 占比 |
| --- | --- | --- | --- |
| 书籍推荐 | 6.3% | 医疗咨询 | 7.2% |
| 约会咨询 | 7.2% | 电影推荐 | 5.8% |
| 家庭关系 | 5.3% | 音乐推荐 | 1.6% |
| 财务咨询 | 7.3% | 网购 | 7.2% |
| 美食推荐 | 8.4% | 运动推荐 | 7.2% |
| 家居装修 | 5.6% | 学习咨询 | 5.8% |
| 法律咨询 | 10.4% | 心理咨询 | 9.1% |
| — | — | 旅行规划 | 5.7% |

> **图 11 | 数据集中提问类型的分布。**
>
> **图 12 | 数据集中对话主题的分布。**
> （*译者注*：D.1 / D.2 的表格即这两张图的数据部分。）

### D.3 现场提问到上下文中的参考信息的距离（PersonaMem_128k，按会话计）

| 距离（会话数） | 占比 |
| --- | --- |
| 0–2 | 5.6% |
| 3–6 | 20.1% |
| 7–10 | 17.6% |
| 11–14 | 17.9% |
| 15–18 | 23.6% |
| 19–20 | 15.2% |

### D.4 同上（PersonaMem_128k，按 token 计）

| 距离（tokens） | 占比 |
| --- | --- |
| 0–9.18k | 5.7% |
| 9.18k–22.3k | 14.8% |
| 22.3k–35.4k | 11.3% |
| 35.4k–48.5k | 7.4% |
| 48.5k–61.6k | 8.2% |
| 61.6k–74.7k | 8.1% |
| 74.7k–87.8k | 8.6% |
| 87.8k–101k | 11.6% |
| 101k–114k | 17.1% |
| 114k–128k | 7.3% |

> **图 13 | 现场提问到参考信息的会话距离。**
>
> **图 14 | 现场提问到参考信息的 token 距离。**

### D.5 PersonaMem_1M

#### D.5.1 现场提问到参考信息的距离（按会话计）

| 距离（会话数） | 占比 |
| --- | --- |
| 0–7 | 5.6% |
| 8–13 | 6.1% |
| 14–19 | 10.1% |
| 20–25 | 11.4% |
| 26–31 | 8.3% |
| 32–37 | 8.9% |
| 38–43 | 9.6% |
| 44–49 | 9.9% |
| 50–55 | 11.7% |
| 56–60 | 18.3% |

#### D.5.2 现场提问到参考信息的距离（按 token 计）

| 距离（tokens） | 占比 |
| --- | --- |
| 0–101k | 6.1% |
| 101k–195k | 5.5% |
| 195k–288k | 10.3% |
| 288k–381k | 10.2% |
| 381k–474k | 12.8% |
| 474k–568k | 8.3% |
| 568k–661k | 9.1% |
| 661k–754k | 9.6% |
| 754k–847k | 11.4% |
| 847k–1M | 16.7% |

> **图 15 | 现场提问到参考信息的会话距离（1M 版本）。**
>
> **图 16 | 现场提问到参考信息的 token 距离（1M 版本）。**

---

## 附录 E 各方案接入外部检索模块后的延迟

在我们使用 GPT-4o-mini、32k tokens 上下文窗口、589 条用户查询的实验中，**RAG** 在 **6 分钟**内完成全部查询，平均 **0.61 秒/查询**（不含嵌入时间——嵌入可在预处理阶段离线完成）。RAG 的检索是**常数时间**的，与上下文长度无关。相比之下，**Mem0** 总共需要 **24 小时**，即约 **150 秒/查询**：它需要提示 LLM 在长上下文中顺序处理更新、删除与新增操作，且这些操作必须在推理时完成，因此延迟显著更高。

---

## 附录 F 错误模式分析

我们对 GPT-4o 未能选出最个性化回复的 **100 条**随机用户查询做了人工错误分析，将错误归为以下五类：

| 错误类型 | 占比 | 含义 |
| --- | --- | --- |
| 格式错误（Format Error） | 14% | 模型未能从给定选项中选出有效选项 |
| 幻觉（Hallucination） | 12% | 模型选了包含用户**从未提及**的偏好的选项 |
| 未识别偏好更新（Failure to Recognize Preference Updates） | 24% | 模型选了反映**过时**偏好、而非最新偏好的选项 |
| 缺乏个性化（Lack of Personalization） | 48% | 模型选了一个**通用合理**、但对当前用户不够个性化的选项 |
| 其他（Other） | 2% | 杂项错误 |

这些结果表明，主要失效模式源于模型难以适配不断演化的用户偏好。此外我们发现，即使多选提示中给出了更个性化的选项，模型也倾向于选择**广泛合理**的回复，而非在语境上更具个性化的回复。

---

## 附录 G PersonaMem 数据集生成使用的提示词

> **说明**：本附录收录了流水线中使用的 8 个提示词模板（图 17–24）。原文为完整英文提示词，此处给出各模板的作用与要点译文，保留结构化字段名以便复用。

| 图 | 提示词 | 作用与要点 |
| --- | --- | --- |
| 图 17 | Persona 描述 ⇒ 初始一般用户画像与偏好 | 给定 1–3 句 persona，扩展为 **10 条**十年内的个人背景历史；每条带 `MM/DD/YYYY` 时间戳，5 条标注 `Short-Term`、5 条标注 `Long-Term`；必须包含 persona 中提到的喜欢与不喜欢，并改写为合适事件；输出为 JSON（时间戳为 key） |
| 图 18 | 生成任务相关用户偏好 | 先围绕 `{task}` 列 **20 个**爱好，随机把 10 个设为喜欢、10 个设为不喜欢（其中部分标注为"刻板印象"型），再基于该主题写 **10 条以上**事件，每条用 `[Fact] Likes:` / `[Fact] Dislikes:` 标注关联的偏好；事件需体现"不喜欢 → 尝试 → 转变"的可能性 |
| 图 19 | 生成对话会话 | 把某一时间段的个人历史事件改写为 `{task}` 主题下的对话记录，**严格按事件及其时间戳**逐条显式提及，时间戳不得修改；思考 persona 与历史会如何导致用户去求助 `{agent role}`；输出为句子字符串列表 |
| 图 20 | 生成"回忆用户事实"提问 | 针对某条 `{Related User Fact}` 与 `{user utterance}`，构造用户再次提起该话题时会问的问题；只提主题或上位类名，**不得直接点名**该具体活动；输出 `{"User Question", "Model Response"}` |
| 图 21 | 生成"提出新建议"提问 | 依据用户**最新**偏好与其原话，构造一个未来可能提出的推荐请求，不显式引用先前偏好，并加入假设情景使其自然 |
| 图 22 | 生成"识别用户最新偏好"提问 | 与图 21 同类模式（原文提示词相同），用于考察模型是否记得**最近一次**偏好并据此给建议 |
| 图 23 | 生成"追踪完整偏好演化"提问 | 给定完整偏好变化序列 `{full sequence}` 及最新一次更新的话语，构造能体现模型记住**整条演化链**（而非仅最新一条）的提问与理想回复 |
| 图 24 | 生成"泛化到新场景"提问 | 聚焦偏好更新事件中的 `[Reasons of Change]` 部分，构造**另一类活动**的新提问，考察模型能否把"变化原因"作为动机泛化到新场景，而非记住事件本身 |

以下为图 17 提示词的完整译文（作为风格示例）：

> 给定以下 persona，在 `{start_time}` 起的十年内扩展出 10 条此人的一般背景历史。把每一点写成项目符号格式，并为每一点加上 `MM/DD/YYYY` 格式的时间戳。注意这些事件应当是一般性的（如职业发展），且会在多个不同主题间共享。你应同时提及日常活动与重要里程碑，既有正面事件也有负面事件。还要把历史与这个人的偏好、厌恶关联起来。使用 JSON 格式，每个时间戳作为 JSON 字典的一个键。每一点还需标注 `[Short-Term]` 或 `[Long-Term]` 标签：短期事实指日常发生的事情（可以是与 persona 无关的，如今天吃了什么），应带有"今天"之类的时间限定词；长期事实指至少一年内不会改变的关键画像特征。需包含 5 条短期与 5 条长期事件。把 persona 中提到的全部 10 项喜欢与不喜欢都包含进来，并把它们改写为合适的事件。所有事件都必须有 `MM/DD/YYYY` 格式的时间戳。至少列出 10 条事件，更多亦可。
>
> 事件模板如下：
>
> ```
> "MM/DD/YYYY": {
>   "Event": xxx,
>   "Category": "Short-Term" OR "Long-Term"
> }
> ```
>
> 不要修改这些键的名称。请在模板的 `MM/DD/YYYY` 与 `xxx` 占位处填入真实数据。为生成正确的 JSON 格式，请使用**双引号**。以下是 persona：`{persona}`

*（译者注）原文图 18、图 19 的图注沿用了图 17 的文字（"Prompt for generating user profile given a short persona description"，属排版复制错误），而其实际内容分别为"生成任务相关用户偏好"与"生成对话会话"；上表按提示词**实际内容**整理。*

---

## 参考文献（保留英文原文，未翻译，便于检索）

- **Afzoon et al. (2024)** Saleh Afzoon, Usman Naseem, Amin Beheshti, and Zahra Jamali. Persobench: Benchmarking personalized response generation in large language models. arXiv preprint arXiv:2410.03198, 2024.
- **Aggarwal et al. (2023)** Abhishek Aggarwal, Cheuk Chi Tam, Dezhi Wu, Xiaoming Li, and Shan Qiao. Artificial intelligence–based chatbots for promoting health behavioral changes: systematic review. Journal of medical Internet research, 25:e40789, 2023.
- **Ait Baha et al. (2023)** Tarek Ait Baha, Mohamed El Hajji, Youssef Es-Saady, and Hammou Fadili. The power of personalization: A systematic review of personality-adaptive chatbots. SN Computer Science, 4(5):661, 2023.
- **Anthropic (2024)** Anthropic. The claude 3 model family: Opus, sonnet, haiku. https://www-cdn.anthropic.com/de8ba9b01c9ab7cbabf5c33b80b7bbc618857627/Model_Card_Claude_3.pdf, March 2024. Accessed: 2025-04-10.
- **Bai et al. (2024)** Yushi Bai, Xin Lv, Jiajie Zhang, Hongchang Lyu, Jiankai Tang, Zhidian Huang, Zhengxiao Du, Xiao Liu, Aohan Zeng, Lei Hou, Yuxiao Dong, Jie Tang, and Juanzi Li. LongBench: A bilingual, multitask benchmark for long context understanding. In Lun-Wei Ku, Andre Martins, and Vivek Srikumar (eds.), Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pp. 3119–3137, Bangkok, Thailand, August 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.acl-long.172. URL https://aclanthology.org/2024.acl-long.172/.
- **Castillo-Bolado et al. (2024)** David Castillo-Bolado, Joseph Davidson, Finlay Gray, and Marek Rosa. Beyond prompts: Dynamic conversational benchmarking of large language models. arXiv preprint arXiv:2409.20222, 2024.
- **Castricato et al. (2024)** Louis Castricato, Nathan Lile, Rafael Rafailov, Jan-Philipp Fränken, and Chelsea Finn. Persona: A reproducible testbed for pluralistic alignment. arXiv preprint arXiv:2407.17387, 2024.
- **Chen et al. (2024)** Jianlv Chen, Shitao Xiao, Peitian Zhang, Kun Luo, Defu Lian, and Zheng Liu. Bge m3-embedding: Multi-lingual, multi-functionality, multi-granularity text embeddings through self-knowledge distillation. arXiv preprint arXiv:2402.03216, 2024.
- **Dean & Morgenstern (2022)** Sarah Dean and Jamie Morgenstern. Preference dynamics under personalized recommendations. In Proceedings of the 23rd ACM Conference on Economics and Computation, pp. 795–816, 2022.
- **Dong et al. (2024)** Yijiang River Dong, Tiancheng Hu, and Nigel Collier. Can llm be a personalized judge? arXiv preprint arXiv:2406.11657, 2024.
- **Ge et al. (2024)** Tao Ge, Xin Chan, Xiaoyang Wang, Dian Yu, Haitao Mi, and Dong Yu. Scaling synthetic data creation with 1,000,000,000 personas. arXiv preprint arXiv:2406.20094, 2024.
- **Grattafiori et al. (2024)** Aaron Grattafiori, Abhimanyu Dubey, Abhinav Jauhri, Abhinav Pandey, Abhishek Kadian, Ahmad Al-Dahle, Aiesha Letman, Akhil Mathur, Alan Schelten, Alex Vaughan, et al. The llama 3 herd of models. arXiv preprint arXiv:2407.21783, 2024.
- **Guo et al. (2025)** Daya Guo, Dejian Yang, Haowei Zhang, Junxiao Song, Ruoyu Zhang, Runxin Xu, Qihao Zhu, Shirong Ma, Peiyi Wang, Xiao Bi, et al. Deepseek-r1: Incentivizing reasoning capability in llms via reinforcement learning. arXiv preprint arXiv:2501.12948, 2025.
- **Gwet (2008)** Kilem Li Gwet. Computing inter-rater reliability and its variance in the presence of high agreement. British Journal of Mathematical and Statistical Psychology, 61(1):29–48, 2008.
- **Hua et al. (2023)** Wenyue Hua, Lei Li, Shuyuan Xu, Li Chen, and Yongfeng Zhang. Tutorial on large language models for recommendation. In Proceedings of the 17th ACM Conference on Recommender Systems, pp. 1281–1283, 2023.
- **Hurst et al. (2024)** Aaron Hurst, Adam Lerer, Adam P Goucher, Adam Perelman, Aditya Ramesh, Aidan Clark, AJ Ostrow, Akila Welihinda, Alan Hayes, Alec Radford, et al. Gpt-4o system card. arXiv preprint arXiv:2410.21276, 2024.
- **Jiang et al. (2024)** Bowen Jiang, Yangxinyu Xie, Zhuoqun Hao, Xiaomeng Wang, Tanwi Mallick, Weijie J Su, Camillo J Taylor, and Dan Roth. A peek into token bias: Large language models are not yet genuine reasoners. arXiv preprint arXiv:2406.11050, 2024.
- **Jiang et al. (2023)** Hang Jiang, Xiajie Zhang, Xubo Cao, Cynthia Breazeal, Deb Roy, and Jad Kabbara. Personallm: Investigating the ability of large language models to express personality traits. arXiv preprint arXiv:2305.02547, 2023.
- **Kim et al. (2024)** Jiho Kim, Woosog Chay, Hyeonji Hwang, Daeun Kyung, Hyunseung Chung, Eunbyeol Cho, Yohan Jo, and Edward Choi. Dialsim: A real-time simulator for evaluating long-term dialogue understanding of conversational agents. arXiv preprint arXiv:2406.13144, 2024.
- **Kirk et al. (2024)** Hannah Rose Kirk, Alexander Whitefield, Paul Röttger, Andrew Bean, Katerina Margatina, Juan Ciro, Rafael Mosquera, Max Bartolo, Adina Williams, He He, et al. The prism alignment project: What participatory, representative and individualised human feedback reveals about the subjective and multicultural alignment of large language models. arXiv preprint arXiv:2404.16019, 2024.
- **Kumar et al. (2024)** Ishita Kumar, Snigdha Viswanathan, Sushrita Yerra, Alireza Salemi, Ryan A Rossi, Franck Dernoncourt, Hanieh Deilamsalehy, Xiang Chen, Ruiyi Zhang, Shubham Agarwal, et al. Longlamp: A benchmark for personalized long-form text generation. arXiv preprint arXiv:2407.11016, 2024.
- **Kuratov et al. (2024)** Yuri Kuratov, Aydar Bulatov, Petr Anokhin, Dmitry Sorokin, Artyom Sorokin, and Mikhail Burtsev. In search of needles in a 10m haystack: Recurrent memory finds what llms miss. arXiv preprint arXiv:2402.10790, 2024.
- **Lewis et al. (2020)** Patrick Lewis, Ethan Perez, Aleksandra Piktus, Fabio Petroni, Vladimir Karpukhin, Naman Goyal, Heinrich Küttler, Mike Lewis, Wen-tau Yih, Tim Rocktäschel, et al. Retrieval-augmented generation for knowledge-intensive nlp tasks. Advances in neural information processing systems, 33:9459–9474, 2020.
- **Li et al. (2025)** Xintong Li, Jalend Bantupalli, Ria Dharmani, Yuwei Zhang, and Jingbo Shang. Toward multi-session personalized conversation: A large-scale dataset and hierarchical tree framework for implicit reasoning. arXiv preprint arXiv:2503.07018, 2025.
- **Lin et al. (2024)** Ying-Chun Lin, Jennifer Neville, Jack Stokes, Longqi Yang, Tara Safavi, Mengting Wan, Scott Counts, Siddharth Suri, Reid Andersen, Xiaofeng Xu, Deepak Gupta, Sujay Kumar Jauhar, Xia Song, Georg Buscher, Saurabh Tiwary, Brent Hecht, and Jaime Teevan. Interpretable user satisfaction estimation for conversational systems with large language models. In Lun-Wei Ku, Andre Martins, and Vivek Srikumar (eds.), Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pp. 11100–11115, Bangkok, Thailand, August 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.acl-long.598. URL https://aclanthology.org/2024.acl-long.598/.
- **Liu et al. (2024)** Nelson F Liu, Kevin Lin, John Hewitt, Ashwin Paranjape, Michele Bevilacqua, Fabio Petroni, and Percy Liang. Lost in the middle: How language models use long contexts. Transactions of the Association for Computational Linguistics, 12:157–173, 2024.
- **Maharana et al. (2024)** Adyasha Maharana, Dong-Ho Lee, Sergey Tulyakov, Mohit Bansal, Francesco Barbieri, and Yuwei Fang. Evaluating very long-term conversational memory of llm agents. arXiv preprint arXiv:2402.17753, 2024.
- **Mem0 (2024)** Mem0. Mem0: An additional memory layer for language models. https://mem0.ai, 2024. Accessed: 2025-03-27.
- **Mysore et al. (2024)** Sheshera Mysore, Zhuoran Lu, Mengting Wan, Longqi Yang, Bahareh Sarrafzadeh, Steve Menezes, Tina Baghaee, Emmanuel Barajas Gonzalez, Jennifer Neville, and Tara Safavi. Pearl: Personalizing large language model writing assistants with generation-calibrated retrievers. In Sachin Kumar, Vidhisha Balachandran, Chan Young Park, Weijia Shi, Shirley Anugrah Hayati, Yulia Tsvetkov, Noah Smith, Hannaneh Hajishirzi, Dongyeop Kang, and David Jurgens (eds.), Proceedings of the 1st Workshop on Customizable NLP: Progress and Challenges in Customizing NLP for a Domain, Application, Group, or Individual (CustomNLP4U), pp. 198–219, Miami, Florida, USA, November 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.customnlp4u-1.16. URL https://aclanthology.org/2024.customnlp4u-1.16/.
- **Nelson et al. (2024)** Elliot Nelson, Georgios Kollias, Payel Das, Subhajit Chaudhury, and Soham Dan. Needle in the haystack for memory based large language models. arXiv preprint arXiv:2407.01437, 2024.
- **OpenAI (2024a)** OpenAI. Openai o1 system card, 2024a. URL https://cdn.openai.com/o1-system-card-20241205.pdf. Accessed: 2025-03-27.
- **OpenAI (2024b)** OpenAI. Openai o3 and o4-mini system card, 2024b. URL https://openai.com/index/o3-o4-mini-system-card/. Accessed: 2025-04-18.
- **OpenAI (2025a)** OpenAI. Openai gpt-4.5 system card, 2025a. URL https://cdn.openai.com/gpt-4-5-system-card-2272025.pdf. Accessed: 2025-03-27.
- **OpenAI (2025b)** OpenAI. Openai o3-mini system card, 2025b. URL https://cdn.openai.com/o3-mini-system-card-feb10.pdf. Accessed: 2025-03-27.
- **Pei et al. (2022)** Jiaxin Pei, Aparna Ananthasubramaniam, Xingyao Wang, Naitian Zhou, Apostolos Dedeloudis, Jackson Sargent, and David Jurgens. Potato: The portable text annotation tool. In Proceedings of the 2022 Conference on Empirical Methods in Natural Language Processing: System Demonstrations, pp. 327–337, 2022.
- **Radlinski & Craswell (2017)** Filip Radlinski and Nick Craswell. A theoretical framework for conversational search. In Proceedings of the 2017 conference on conference human information interaction and retrieval, pp. 117–126, 2017.
- **Rein et al. (2024)** David Rein, Betty Li Hou, Asa Cooper Stickland, Jackson Petty, Richard Yuanzhe Pang, Julien Dirani, Julian Michael, and Samuel R Bowman. Gpqa: A graduate-level google-proof q&a benchmark. In First Conference on Language Modeling, 2024.
- **Salemi et al. (2023)** Alireza Salemi, Sheshera Mysore, Michael Bendersky, and Hamed Zamani. Lamp: When large language models meet personalization. arXiv preprint arXiv:2304.11406, 2023.
- **Santurkar et al. (2023)** Shibani Santurkar, Esin Durmus, Faisal Ladhak, Cinoo Lee, Percy Liang, and Tatsunori Hashimoto. Whose opinions do language models reflect? In International Conference on Machine Learning, pp. 29971–30004. PMLR, 2023.
- **Shi et al. (2024)** Taiwei Shi, Zhuoer Wang, Longqi Yang, Ying-Chun Lin, Zexue He, Mengting Wan, Pei Zhou, Sujay Jauhar, Sihao Chen, Shan Xia, et al. Wildfeedback: Aligning llms with in-situ user interactions and feedback. arXiv preprint arXiv:2408.15549, 2024.
- **Sorensen et al. (2024)** Taylor Sorensen, Jared Moore, Jillian Fisher, Mitchell Gordon, Niloofar Mireshghallah, Christopher Michael Rytting, Andre Ye, Liwei Jiang, Ximing Lu, Nouha Dziri, et al. Position: a roadmap to pluralistic alignment. In Proceedings of the 41st International Conference on Machine Learning, pp. 46280–46302, 2024.
- **Srivastava et al. (2023)** Aarohi Srivastava, Abhinav Rastogi, Abhishek Rao, Abu Awal Md Shoeb, Abubakar Abid, Adam Fisch, Adam R Brown, Adam Santoro, Aditya Gupta, Adrià Garriga-Alonso, et al. Beyond the imitation game: quantifying and extrapolating the capabilities of language models. Transactions on Machine Learning Research, 2023(5):1–95, 2023.
- **Tan et al. (2025)** Juntao Tan, Liangwei Yang, Zuxin Liu, Zhiwei Liu, Rithesh Murthy, Tulika Manoj Awalgaonkar, Jianguo Zhang, Weiran Yao, Ming Zhu, Shirley Kokane, et al. Personabench: Evaluating ai models on understanding personal information through accessing (synthetic) private user data. arXiv preprint arXiv:2502.20616, 2025.
- **Team et al. (2024)** Gemini Team, Petko Georgiev, Ving Ian Lei, Ryan Burnell, Libin Bai, Anmol Gulati, Garrett Tanzer, Damien Vincent, Zhufeng Pan, Shibo Wang, et al. Gemini 1.5: Unlocking multimodal understanding across millions of tokens of context. arXiv preprint arXiv:2403.05530, 2024.
- **Tian et al. (2024)** Yufei Tian, Tenghao Huang, Miri Liu, Derek Jiang, Alexander Spangher, Muhao Chen, Jonathan May, and Nanyun Peng. Are large language models capable of generating human-level narratives? In Yaser Al-Onaizan, Mohit Bansal, and Yun-Nung Chen (eds.), Proceedings of the 2024 Conference on Empirical Methods in Natural Language Processing, pp. 17659–17681, Miami, Florida, USA, November 2024. Association for Computational Linguistics. doi: 10.18653/v1/2024.emnlp-main.978. URL https://aclanthology.org/2024.emnlp-main.978/.
- **Tseng et al. (2024)** Yu-Min Tseng, Yu-Chao Huang, Teng-Yun Hsiao, Yu-Ching Hsu, Jia-Yin Foo, Chao-Wei Huang, and Yun-Nung Chen. Two tales of persona in llms: A survey of role-playing and personalization. arXiv preprint arXiv:2406.01171, 2024.
- **Wang et al. (2023)** Yaqing Wang, Jiepu Jiang, Mingyang Zhang, Cheng Li, Yi Liang, Qiaozhu Mei, and Michael Bendersky. Automated evaluation of personalized text generation using large language models. arXiv preprint arXiv:2310.11593, 2023.
- **Wu et al. (2024)** Di Wu, Hongwei Wang, Wenhao Yu, Yuwei Zhang, Kai-Wei Chang, and Dong Yu. Longmemeval: Benchmarking chat assistants on long-term interactive memory. arXiv preprint arXiv:2410.10813, 2024.
- **Xie et al. (2024a)** Jian Xie, Kai Zhang, Jiangjie Chen, Tinghui Zhu, Renze Lou, Yuandong Tian, Yanghua Xiao, and Yu Su. Travelplanner: a benchmark for real-world planning with language agents. In Proceedings of the 41st International Conference on Machine Learning, pp. 54590–54613, 2024a.
- **Xie et al. (2024b)** Yangxinyu Xie, Bowen Jiang, Tanwi Mallick, Joshua David Bergerson, John K Hutchison, Duane R Verner, Jordan Branham, M Ross Alexander, Robert B Ross, Yan Feng, et al. Wildfiregpt: Tailored large language model for wildfire analysis. arXiv preprint arXiv:2402.07877, 2024b.
- **Xu (2021)** J Xu. Beyond goldfish memory: Long-term open-domain conversation. arXiv preprint arXiv:2107.07567, 2021.
- **Xu et al. (2024)** Xiaoyue Xu, Qinyuan Ye, and Xiang Ren. Stress-testing long-context language models with lifelong icl and task haystack. arXiv preprint arXiv:2407.16695, 2024.
- **Xu et al. (2022)** Xinchao Xu, Zhibin Gou, Wenquan Wu, Zheng-Yu Niu, Hua Wu, Haifeng Wang, and Shihang Wang. Long time no see! open-domain conversation with long-term persona memory. In Smaranda Muresan, Preslav Nakov, and Aline Villavicencio (eds.), Findings of the Association for Computational Linguistics: ACL 2022, pp. 2639–2650, Dublin, Ireland, May 2022. Association for Computational Linguistics. doi: 10.18653/v1/2022.findings-acl.207. URL https://aclanthology.org/2022.findings-acl.207/.
- **Yue et al. (2024)** Xiang Yue, Yuansheng Ni, Kai Zhang, Tianyu Zheng, Ruoqi Liu, Ge Zhang, Samuel Stevens, Dongfu Jiang, Weiming Ren, Yuxuan Sun, et al. Mmmu: A massive multi-discipline multimodal understanding and reasoning benchmark for expert agi. In Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition, pp. 9556–9567, 2024.
- **Zhang et al. (2024)** Xinrong Zhang, Yingfa Chen, Shengding Hu, Zihang Xu, Junhao Chen, Moo Hao, Xu Han, Zhen Thai, Shuo Wang, Zhiyuan Liu, et al. ∞\infty bench: Extending long context evaluation beyond 100k tokens. In Proceedings of the 62nd Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers), pp. 15262–15277, 2024.
- **Zhao et al. (2025)** Siyan Zhao, Mingyi Hong, Yang Liu, Devamanyu Hazarika, and Kaixiang Lin. Do llms recognize your preferences? evaluating personalized preference following in llms. In The thirteenth international conference on learning representations, 2025.
- **Zheng et al. (2024)** Huaixiu Steven Zheng, Swaroop Mishra, Hugh Zhang, Xinyun Chen, Minmin Chen, Azade Nova, Le Hou, Heng-Tze Cheng, Quoc V Le, Ed H Chi, et al. Natural plan: Benchmarking llms on natural language planning. arXiv preprint arXiv:2406.04520, 2024.
- **Zhou et al. (2023)** Jeffrey Zhou, Tianjian Lu, Swaroop Mishra, Siddhartha Brahma, Sujoy Basu, Yi Luan, Denny Zhou, and Le Hou. Instruction-following evaluation for large language models. arXiv preprint arXiv:2311.07911, 2023.
- **Zollo et al. (2024)** Thomas P Zollo, Andrew Wei Tung Siah, Naimeng Ye, Ang Li, and Hongseok Namkoong. Personalllm: Tailoring llms to individual preferences. arXiv preprint arXiv:2409.20296, 2024.

---

## 术语与背景速览（译者整理）

| 术语 | 含义 |
| --- | --- |
| PersonaMem | 本文提出的基准：180+ 条模拟用户—LLM 交互历史，评测模型"记住→追踪→应用"用户动态画像的能力 |
| Persona / 用户画像 | 由 PersonaHub 采样的 1–3 句人物设定，扩展为含人口统计、人生事件、偏好的完整画像 |
| In-situ user query（现场用户提问） | 用户以**第一人称**在某时间点新开会话发出的提问，正确答案随该时间点画像状态而变 |
| Static vs. Dynamic attributes | 静态属性（姓名/年龄/性别/职业）与动态属性（随时间演化的偏好、事件） |
| Preference evolution（偏好演化） | 用户偏好因事件（如过敏、搬家）而发生的变化序列；含"变化原因" |
| 7 类提问 | 回忆事实、提新建议、识别最新偏好、追踪完整演化、回溯变化原因、偏好一致推荐、泛化到新场景 |
| 判别式设置（discriminative） | 四个选项同屏呈现、顺序打乱，模型选一个并解释；本文主评估方式 |
| 生成式设置（generative） | 一次只给一个选项，按 **长度归一化的联合对数概率** 打分选优；仅用于开权重模型 |
| Context length | 32k / 128k / 1M tokens 三档，分别对应 10 / 20 / 60 个会话 |
| Session（会话） | 一次多轮对话，15–30 轮；跨主题会话按结束时间戳**拓扑排序**后拼接 |
| Topic-specific personal history | 每个对话主题单独生成的带时间戳历史（事件+偏好+更新+原因），主题间偏好不重叠 |
| Self-reflection（自我反思） | 生成对话后用 GPT-4o 复查，补回被遗漏的历史事件，保证偏好更新序列完整 |
| Lost in the middle | 长上下文中间位置的信息更易被忽略；本文图 4 复现了该现象 |
| RAG / Mem0 | 两种外部记忆方案；RAG 检索 top-5 消息（BGE-M3 稠密向量），Mem0 用 LLM 增量构建事实库后检索 top-5 |
| AC1（Gwet's AC1） | 评分者间信度指标，对"True 占绝大多数"的不平衡更稳健 |
| COLM | Conference on Language Modeling，本文发表于 COLM 2025 |

**核心数字速记**

- 规模：**20** 个 persona、**180+** 条交互历史、每条最多 **60** 个会话（15–30 轮/会话）、**15** 个主题、约 **6k** 组问答对
- 三档长度：**32k / 128k / 1M** tokens（10 / 20 / 60 会话）；单会话约 **6k** tokens
- 成本：每个 persona × 每个主题约 **2 美元**，且与上下文长度（至 1M）无关
- 主结果（128k，四选一）：GPT-4.5 / Gemini-1.5 约 **52%**，前沿模型整体约 **50%**，Llama-4-Maverick **43%**
- 分类型：回忆事实/追踪变化 **60–70%**，把最新处境纳入回复 **30–50%**
- 位置效应：相关信息在中段最易失效（"lost in the middle"）
- 记忆模块：RAG 优于 Mem0，收益最大的是"回忆事实"与"泛化到新场景"，最小的是"回溯变化原因"
- 延迟（GPT-4o-mini, 32k, 589 条查询）：RAG **0.61 s/查询**（6 分钟跑完全部）vs. Mem0 **150 s/查询**（24 小时）
- 生成式 vs. 判别式：LLaMA-3.1-8B 在生成式下反而更好（32k 版本）
- 人工验证（90 条，3 人 ×4 维度）：恰当 **97.8%**、相关 **95.6%**、正确 **97.8%**、最优回复 **90.0%**
- 错误分析（GPT-4o 的 100 条错题）：缺乏个性化 **48%** > 未识别偏好更新 **24%** > 格式错误 **14%** > 幻觉 **12%** > 其他 **2%**
- 数据分布偏斜：提问类型中"识别最新偏好"占 **30.09%**、"提新建议"占 **22.92%**；主题中法律咨询最高 **10.4%**、音乐推荐最低 **1.6%**

---

## 翻译说明与阅读建议

- **最值得先读的三处**：① §2 的 7 类提问定义 + 表 1（决定整个基准的评测口径）；② §4.2 的四条结论（模型到底强在哪、弱在哪）；③ 附录 F 的错误分类（48% 的失败是"选了通用合理但不个性化的答案"，这是个性化对齐的核心瓶颈）。
- **对本号（搜广推/LLM 系统）读者的意义**：PersonaMem 把"用户画像"从静态画像升级为**带时间戳的偏好演化轨迹**，其评测范式与推荐系统的"兴趣漂移建模"高度同构；同时它以实测数据说明：单纯加长上下文 ≠ 能用好历史，RAG（语义检索）在 32k 下显著优于结构化记忆库（Mem0），且延迟低 2 个数量级——这对线上"长记忆"服务的工程选型有直接参考价值。
- 原文中"revisit reasons"与"generalize"两类任务准确率最低，提示当前 LLM 缺少的是**因果性记忆**（为什么变），而不仅是**事实性记忆**（变了什么）。
