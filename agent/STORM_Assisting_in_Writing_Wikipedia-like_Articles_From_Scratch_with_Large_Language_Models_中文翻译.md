# 用大语言模型从零撰写维基百科式文章（STORM）

> **原文**: Assisting in Writing Wikipedia-like Articles From Scratch with Large Language Models
> **作者**: Yijia Shao, Yucheng Jiang, Theodore A. Kanell, Peter Xu, Omar Khattab, Monica S. Lam（斯坦福大学）
> **日期/arXiv**: 2024-02-22 提交，2024-04-08 修订（v2）；NAACL 2024 主会论文，27 页
> **链接**: https://arxiv.org/abs/2402.14207
> **代码**: https://github.com/stanford-oval/storm
> **译者说明**: 本文为完整全文翻译。图注按图意译出并补充说明；带 *（译者注）* 标记的段落为译者补充的背景说明，非原文内容。原文中的脚注以上标数字保留（编号尽量与原文一致），附录中的提示词以「Listing 6/7」形式给出并译出中文语义。

---

## 摘要

本文研究如何用大语言模型（LLM）**从零**撰写有据可依、结构清晰的长篇文章，使其广度与深度可与维基百科页面相比。这一尚未被充分探索的问题在**写作前阶段（pre-writing stage）**提出了新的挑战，包括在动笔之前如何调研主题、如何准备大纲。

我们提出 STORM —— 一个通过**检索**与**多视角提问**来**综合主题大纲**的写作系统（**S**ynthesis of **T**opic **O**utlines through **R**etrieval and **M**ulti-perspective Question Asking）。STORM 对写作前阶段建模为三步：（1）在调研给定主题时**发现多样化的视角**；（2）**模拟对话** —— 由持不同视角的"写作者"向一位"主题专家"提问，专家的回答以可信互联网来源为依据；（3）**整理**收集到的信息以创建大纲。

为评估，我们构建了 **FreshWiki** 数据集，包含近期的高质量维基百科文章，并制定了大纲评估方法来评价写作前阶段。我们还收集了资深维基百科编辑的反馈。与一个由大纲驱动的检索增强基线相比，STORM 的文章被判定为"有组织"的比例高出 **25 个百分点**（绝对值），"覆盖广泛"的比例高出 **10 个百分点**。专家反馈还揭示了生成有据长文的新挑战，例如**来源偏见迁移**与**不相关事实的过度关联**。

---

## 1 引言

> **图 1 | 我们探索从零撰写维基百科式文章，这要求在产出文章之前有一个写作前阶段。** 在这一阶段，更简单的方法（如 Direct Prompting，直接提示）规划能力有限；相比之下，STORM 通过模拟对话中由视角引导的提问来调研主题。*（图内为两组对比：左侧（A）直接提示生成的提问只有"What/When/Where"这类表层问题，右侧（B）在给定视角后，提问转向具体安排、预算、争议等纵深维度。）*

大语言模型已展现出令人印象深刻的写作能力（Yang et al., 2023; Pavlik, 2023; Wenzlaff and Spaeth, 2022; Fitria, 2023），但我们尚不清楚如何用它们撰写**有据可依的长篇文章**，例如完整的维基百科页面。这类**说明性写作（expository writing）**旨在以有条理的方式向读者介绍某个主题（Weaver III and Kintsch, 1991; Balepur et al., 2023），它要求在写作前阶段进行充分的调研与规划（Rohman, 1965），甚至早于真正的写作过程本身。

然而，以往关于维基百科文章生成的工作（Banerjee and Mitra, 2015; Minguillón et al., 2017; Liu et al., 2018; Fan and Gardent, 2022）大多**绕过了写作前阶段**：例如 Liu et al. (2018) 假定参考文档已提前给定，而 Fan and Gardent (2022) 假定文章大纲已存在、只专注于扩写每一节。这些假设在一般情况下并不成立 —— 收集参考文献与打磨大纲需要高阶的**信息素养（information literacy）**（Doyle, 1994），以便识别、评估与组织外部来源，这对即便经验丰富的写作者也是艰巨任务。自动化这一流程可以帮助个人启动对某个主题的深入学习，并大幅削减说明性写作所需的昂贵专家工时。

我们通过聚焦"如何从零生成维基百科式文章"来探索这些挑战，并把该问题分解为两个任务。第一是**做调研以生成大纲** —— 即一份多级章节列表，并收集一组参考文档。第二是**利用大纲与参考文献产出全文**。这种任务分解映照了人类写作过程，通常包括写作前、起草与修订几个阶段（Rohman, 1965; Munoz-Luna, 2015）。

由于预训练语言模型本身拥有丰富的知识，一个直接的做法是依赖其参数知识来生成大纲甚至整篇文章（**Direct Gen**）。但这一做法受限于细节缺失与幻觉（Xu et al., 2023），在应对长尾主题时尤为明显（Kandpal et al., 2023）。这凸显了利用外部来源的重要性，而当前策略常涉及检索增强生成（RAG），这又绕回到写作前阶段"如何调研主题"的问题 —— 因为很多信息无法通过简单的主题检索浮现出来。

人类学习理论（Tawfik et al., 2020; Booth et al., 2003）强调在信息获取中**提出有效问题**的重要性。尽管可以直接提示经过指令微调的模型（Ouyang et al., 2022）来生成问题，但我们发现它们通常产出基础的"What"、"When"、"Where"问题（图 1(A)），往往只触及主题的表层事实。

为赋予 LLM 进行更好调研的能力，我们提出 **STORM** 范式：通过检索与多视角提问来综合主题大纲。

STORM 的设计基于两个假设：（1）**多样化视角带来多样化问题**；（2）**提出深入问题需要迭代式调研**。基于这两点，STORM 采用一种新颖的多阶段方法：首先通过检索并分析与主题相似的维基百科文章来发现多样化视角，然后让 LLM 扮演特定视角的角色来提问（图 1(B)）。接着，为引出用于迭代调研的**追问**（图 1(C)），STORM 模拟多轮对话，其中对生成问题的回答以互联网内容为依据。最后，基于 LLM 的内部知识与收集到的信息，STORM 创建一份可按节扩写的大纲，进而发展成完整的维基百科式文章。

我们使用 **FreshWiki** 数据集（§2.1）评估 STORM，该数据集精选了近期高质量维基百科文章，以避免预训练期间的数据泄漏<sup>[1]</sup>。为便于研究写作前阶段，我们定义了对照人类撰写文章来评估大纲质量的指标。

我们还邀请了一组资深维基百科编辑进行专家评估。编辑们认为 STORM 优于由大纲驱动的 RAG 基线，尤其在文章的广度与组织性方面。他们也指出了未来研究的挑战，包括以下情形：（1）**互联网上的偏见会影响生成的文章**；（2）**LLM 会虚构不相关事实之间的关联**。这些挑战为有据写作系统开辟了新的前沿。

我们的主要贡献包括：

- 为评估 LLM 系统从零生成有据长文的能力（尤其是写作前阶段的挑战），我们构建了 **FreshWiki** 数据集，并为大纲质量与最终文章质量建立了评估标准。
- 我们提出 **STORM**，一个自动化写作前阶段的新系统。STORM 通过让 LLM 提出尖锐问题、从互联网检索可信信息，来调研主题并创建大纲。
- 自动评估与人工评估都证明了我们方法的有效性。专家反馈进一步揭示了生成有据长文的新挑战。

---

## 2 FreshWiki

我们研究从零生成维基百科式文章，重点放在写作前阶段（Rohman, 1965）—— 它包含"收集与整理相关信息"（即"调研"）这些高要求的子任务。这模拟了人类写作方式，也促使一些教育者把维基百科写作视为学术训练的练习（Tardy, 2010）。

> **表 1 | 现有文献中不同维基百科生成设定的对比。** 生成一个段落并不需要文章大纲。

| 工作 | 领域 | 范围 | 给定大纲？ | 给定参考文档？ |
| --- | --- | --- | --- | --- |
| Balepur et al. (2023) | 单一 | 一个段落 | / | 是 |
| Qian et al. (2023) | 全部 | 一个段落 | / | 否 |
| Fan and Gardent (2022) | 单一 | 全文 | 是 | 否 |
| Liu et al. (2018) | 全部 | 一个段落 | / | 是 |
| Sauper and Barzilay (2009) | 两个 | 全文 | 否 | 否 |
| **本文（Ours）** | **全部** | **全文** | **否** | **否** |

表 1 将我们的工作与此前维基百科生成的基准设置作对比。已有工作总体聚焦于评估更短片段（例如一个段落）的生成、范围更窄（例如某一领域或两个领域），或是在已显式提供大纲或参考文档的前提下进行评估。一个典型例子是 WikiSum（Liu et al., 2018），它把维基百科文章生成视为以参考文档为输入的**多文档摘要**问题。

我们的设定强调长文有据写作系统**调研与整理内容**的能力。具体而言，给定主题 $t$，任务是找到一组参考文献 $\mathcal{R}$ 并生成全文 $\mathcal{S}=s_1s_2\ldots s_n$，其中每个句子 $s_i$ 引用 $\mathcal{R}$ 中的一列文档<sup>[2]</sup>。

### 2.1 FreshWiki 数据集

创建一篇新的维基百科式文章不仅要求流畅写作，还要求良好的调研能力。由于现代 LLM 通常在维基百科文本上训练，我们通过显式寻找在我们所测 LLM 训练截止之后被创建（或经过大量编辑）的近期维基百科文章来缓解数据泄漏。当未来出现新 LLM 时，我们的流程可以重跑。

为应用日期准则，我们聚焦 2022 年 2 月至 2023 年 9 月期间每月**编辑次数最多的前 100 个页面**<sup>[3]</sup>。为确保参考文献质量，我们筛选只保留被 ORES<sup>[4]</sup> 评为 B 级或更高的文章。我们还排除列表类文章<sup>[5]</sup>以及没有子章节的文章。虽然高质量维基百科文章通常含结构化数据（如表格）且是多模态的，我们只考虑纯文本部分以简化任务。数据集更多细节见附录 A。

### 2.2 大纲创建与评估

生成或评估一篇全文是困难的（Xu et al., 2023; Krishna et al., 2023）。当人类教育者教授学术写作时，有时会在**大纲阶段**指导学生（Eriksson and Mäkitalo, 2015），因为一份详尽的大纲表明作者对主题有全面理解，也为撰写全文提供了坚实基础（Dietz and Foley, 2019）。受此启发，我们把 $\mathcal{S}$ 的生成分解为两个阶段。在写作前阶段，要求系统创建大纲 $\mathcal{O}$，其定义为一组多级章节标题<sup>[6]</sup>。在写作阶段，系统使用主题 $t$、参考文献 $\mathcal{R}$ 与大纲 $\mathcal{O}$ 产出全文 $\mathcal{S}$。

为评估大纲覆盖度，我们引入两个指标：**标题软召回（heading soft recall）**与**标题实体召回（heading entity recall）**。这些指标把人类撰写文章的多级章节标题视为真值，与 $\mathcal{O}$ 中的标题作比较。我们认识到两组标题之间无需精确匹配，因此用基于 Sentence-BERT（Reimers and Gurevych, 2019）标题嵌入的余弦相似度来计算标题软召回（Fränti and Mariescu-Istodor, 2023）（细节见附录 C.1）。

我们还计算标题实体召回，量化为人类撰写文章标题中的命名实体被 $\mathcal{O}$ 覆盖的百分比。实体通过 FLAIR 命名实体识别（Akbik et al., 2019）抽取。

---

## 3 方法

> **图 2 | STORM 概览：自动化写作前阶段。** 从给定主题出发，STORM 通过考察相关维基百科文章识别出覆盖该主题的多种视角（步骤 1-2）。随后它模拟一场对话：维基百科写作者在给定视角引导下提问，而由可信在线来源支撑的"专家"作答（步骤 3-6）。最终大纲基于 LLM 的内在知识以及从不同视角收集到的对话内容整理而成（步骤 7-8）。

我们提出 STORM 以自动化写作前阶段：通过有效提问调研给定主题（§3.1、§3.2）并创建大纲（§3.3）；大纲随后会被扩展为以所收集参考文献为依据的全文（§3.4）。图 2 给出 STORM 的概览，伪代码见附录 B。

### 3.1 视角引导的提问（Perspective-Guided Question Asking）

Rohman (1965) 把写作前阶段定义为写作过程中的"发现"阶段。这与商业中的**利益相关者理论**（Freeman et al., 2010）相呼应 —— 不同利益相关者会关注公司的不同侧面；同样地，持不同视角的个体在调研同一主题时会聚焦不同方面，从而发掘出多面向的信息。更进一步，特定视角可以充当**先验知识**，引导个体提出更深入的问题。例如，活动策划者可能会问"2022 年冬奥会开幕式的交通安排"与"预算"，而普通人更可能问该活动的基本信息（图 1(A)）。

给定输入主题 $t$，STORM 通过考察相似主题的既有文章来发现不同视角，并用这些视角来控制提问过程。具体而言，STORM 提示 LLM 生成一组相关主题，随后若这些主题对应的维基百科文章可通过 Wikipedia API<sup>[7]</sup> 获取（图 2 步骤 1），则抽取其目录（table of contents）。这些目录被拼接为上下文，用于提示 LLM 识别出 $N$ 个视角 $\mathcal{P}=\{p_1,\ldots,p_N\}$，这些视角合起来能贡献一篇关于 $t$ 的全面文章（图 2 步骤 2）。为确保 $t$ 的基本信息也被覆盖，我们把 $p_0$ 以"聚焦于广泛覆盖该主题基本事实的基本事实写作者"（basic fact writer）的身份加入 $\mathcal{P}$。每个视角 $p\in\mathcal{P}$ 都将并行地用于引导 LLM 的提问过程。

### 3.2 模拟对话

问题与提问理论（Ram, 1991）指出：已有问题的答案在帮助我们更全面理解主题的同时，往往也会**催生新问题**。为启动这一动态过程，STORM 模拟一位维基百科写作者与一位主题专家之间的对话。在第 $i$ 轮对话中，由 LLM 驱动的写作者基于主题 $t$、其被分配的视角 $p\in\mathcal{P}$ 以及对话历史 $\{q_1,a_1,\ldots,q_{i-1},a_{i-1}\}$ 生成单个问题 $q_i$，其中 $a_j$ 表示模拟专家的回答。对话历史使 LLM 能更新其对主题的理解并提出**追问**。实践中我们把对话限制在至多 $M$ 轮。

为确保对话历史提供的是事实性信息，我们使用互联网上的可信来源来为每个查询 $q_i$ 的回答 $a_i$ 提供依据。由于 $q_i$ 可能很复杂，我们首先提示 LLM 把 $q_i$ 拆解为一组搜索查询（图 2 步骤 4）；搜索结果随后会依据维基百科的可信来源指引<sup>[8]</sup>用基于规则的过滤器评估，以剔除不可信来源（图 2 步骤 5）。最后，LLM 综合这些可信来源生成回答 $a_i$，这些来源也会被加入 $\mathcal{R}$ 用于全文生成（§3.4）。

### 3.3 创建文章大纲

在通过 $N+1$ 场模拟对话（记作 $\{\mathcal{C}_0,\mathcal{C}_1,\ldots,\mathcal{C}_N\}$）充分调研主题之后，STORM 在真正开始写作前创建大纲。为充分利用 LLM 的内部知识，我们首先只给定主题 $t$，提示模型生成一份**草稿大纲** $\mathcal{O}_{\text{D}}$（图 2 步骤 7）。$\mathcal{O}_{\text{D}}$ 通常提供一个通用但有条理的框架。随后，把主题 $t$、草稿大纲 $\mathcal{O}_{\text{D}}$ 以及模拟对话 $\{\mathcal{C}_0,\mathcal{C}_1,\ldots,\mathcal{C}_N\}$ 一并提示给 LLM 以精炼大纲（图 2 步骤 8）。由此得到改进后的大纲 $\mathcal{O}$，用于产出全文。

### 3.4 撰写全文

基于写作前阶段收集的参考文献 $\mathcal{R}$ 与打磨出的大纲 $\mathcal{O}$，全文可以**逐节**撰写。由于通常无法把整个 $\mathcal{R}$ 放进 LLM 的上下文窗口，我们用章节标题及其所有层级子标题，基于 Sentence-BERT 嵌入计算的语义相似度从 $\mathcal{R}$ 中检索相关文档。拿到相关信息后，提示 LLM 生成带引用的该节内容。所有章节生成后拼接成全文。由于各节是并行生成的，我们再提示 LLM 对拼接后的文章删除重复信息以提升连贯性。此外，为符合维基百科的文体规范，还使用 LLM 综合出全文摘要，构成开头的**导言节（lead section）**。

---

## 4 实验

### 4.1 文章选取

STORM 有能力调研复杂主题并从详尽大纲撰写长文。但在这个受控实验中，我们把最终输出限制在至多 4000 token（约 3000 词）。为进行有意义的比较，我们从 FreshWiki 数据集中随机选取 100 个样本（见 §2.1），其人类撰写文章不超过 3000 词。

### 4.2 自动指标

如 §2.2 所述，我们通过计算标题软召回与标题实体召回，评估大纲质量以衡量写作前阶段。召回分数越高，说明相对人类撰写文章而言大纲越全面。

为评估全文质量，我们采用 ROUGE（Lin, 2004），并基于 FLAIR NER 结果计算文章级实体召回。此外，依据维基百科的优良条目标准<sup>[9]</sup>，我们从以下方面评估文章：（1）趣味性（Interest Level）、（2）连贯性与组织性（Coherence and Organization）、（3）相关性与聚焦（Relevance and Focus）、（4）覆盖度（Coverage）、（5）可验证性（Verifiability）。

对于方面（1）-（4），我们使用 **Prometheus**（Kim et al., 2023）—— 一个 13B 的评估器 LLM —— 依据与两位资深维基百科编辑共同制定的 5 分制评分标准为文章打分（见附录 C.2）。对于可验证性，我们按 Gao et al. (2023) 的定义计算**引用召回**与**引用精确率**，并用 Mistral 7B-Instruct（Jiang et al., 2023a）检查被引用段落是否蕴含生成的句子。

> **表 2 | 文章质量的自动评估结果。** † 表示 STORM 与最佳基线（即 oRAG）之间的配对 t 检验存在显著差异（$p<0.05$）。评分项采用 1-5 分制。

| 方法 | ROUGE-1 | ROUGE-L | 实体召回 | 趣味性 | 组织性 | 相关性 | 覆盖度 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Direct Gen | 25.62 | 12.63 | 5.08 | 2.87 | 4.60 | 3.10 | 4.16 |
| RAG | 28.52 | 13.18 | 7.57 | 3.14 | 4.22 | 3.05 | 4.08 |
| oRAG | 44.26 | 16.51 | 12.57 | 3.90 | 4.79 | 4.09 | 4.70 |
| **STORM** | **45.82** | **16.70** | **14.10†** | **3.99†** | **4.82** | **4.45†** | **4.88†** |
| w/o Outline Stage | 26.77 | 12.77 | 7.39 | 3.33 | 4.87 | 3.35 | 4.37 |

> **表 3 | 大纲质量评估结果（%）。** † 表示 STORM 与基线之间配对 t 检验存在显著差异（$p<0.05$）。

| 模型 | 方法 | 标题软召回 | 标题实体召回 |
| --- | --- | --- | --- |
| GPT-3.5 | Direct Gen | 80.23 | 32.39 |
| GPT-3.5 | RAG/oRAG | 73.59 | 33.85 |
| GPT-3.5 | RAG-expand | 74.40 | 33.85 |
| GPT-3.5 | **STORM** | **86.26†** | **40.52†** |
| GPT-3.5 | w/o Perspective | 84.49 | 40.12 |
| GPT-3.5 | w/o Conversation | 77.97 | 31.98 |
| GPT-4 | Direct Gen | 87.66 | 34.78 |
| GPT-4 | RAG/oRAG | 89.55 | 42.38 |
| GPT-4 | RAG-expand | 91.36 | 43.53 |
| GPT-4 | **STORM** | **92.73†** | **45.91** |
| GPT-4 | w/o Perspective | 92.39 | 42.70 |
| GPT-4 | w/o Conversation | 88.75 | 39.30 |

### 4.3 基线

由于此前的工作使用不同设定且不使用 LLM，难以直接比较。我们改用以下三个基于 LLM 的基线：

1. **Direct Gen**：直接提示 LLM 生成大纲，再用该大纲生成全文。
2. **RAG**：检索增强生成基线，以主题进行检索，并把检索结果连同主题 $t$ 一起用于生成大纲或整篇文章。
3. **大纲驱动的 RAG（oRAG）**：在大纲创建上与 RAG 相同，但进一步用章节标题检索额外信息，逐节生成文章。

### 4.4 STORM 实现

我们用 DSPy 框架（Khattab et al., 2023）以零样本提示构建 STORM。附录 B 包含伪代码与相应提示词。STORM 中的超参数 $N$ 与 $M$ 均设为 5。提问部分使用对话模型 `gpt-3.5-turbo`，STORM 的其他部分使用 `gpt-3.5-turbo-instruct`；我们还实验了用 `gpt-4` 起草与精炼大纲（图 2 步骤 7-8）。在报告的结果中，STORM 中的模拟主题专家以 You.com 搜索 API<sup>[10]</sup> 为依据，尽管该流程与其他搜索引擎兼容。作为真值的维基百科文章被从搜索结果中排除。

对于最终文章生成，我们只报告使用 `gpt-4` 的结果，因为 `gpt-3.5` 在生成带引用的文本时不忠于来源（Gao et al., 2023）。所有实验设置 temperature 为 1.0、top_p 为 0.9。

---

## 5 结果与分析

### 5.1 主要结果

我们用**大纲覆盖度**作为写作前阶段的代理指标（见 §2.2）。表 3 展示了标题软召回与实体召回。由 LLM 直接生成的大纲（Direct Gen）已展现出较高的标题软召回，说明 LLM 凭借丰富的参数知识就能把握主题的高层方面。然而 STORM 通过提出有效问题来调研主题，能创建召回更高的提纲，覆盖更多主题特有的方面。值得注意的是，尽管 RAG 利用了额外信息，但把**未组织的**信息塞进上下文窗口反而让较弱模型（即 GPT-3.5）的大纲生成更困难，导致性能更差。为测试 RAG 基线的上限，我们进一步扩展检索来源：以 RAG 产出的大纲为起点，用其章节标题作为搜索查询收集更多来源，再把新收集的来源与初始大纲一并输入 LLM 生成精炼大纲。这种改进做法在表 3 中记作 **"RAG-expand"**。实验结果表明，即便增加一轮检索与精炼能改善 RAG 的大纲，我们提出的 STORM 仍然超越其表现。

我们进一步评估全文质量。如表 2 所示，oRAG 显著优于 RAG，凸显了用大纲来组织结构化全文生成的有效性。尽管该方法在利用检索与大纲方面有优势，我们的方法仍优于它。有效的提问机制让文章的实体召回更高，评估器 LLM 也在"趣味性""相关性与聚焦""覆盖度"方面给出显著更高的评分。不过我们承认，评估器 LLM 可能高估机器生成的文本。我们细致的**人工评估**（§6）表明 STORM 仍有很大改进空间。

> **表 4 | 由 Mistral 7B-Instruct 判定的引用质量。**

| 方法 | 引用召回 | 引用精确率 |
| --- | --- | --- |
| STORM | 84.83 | 85.18 |

尽管本工作主要聚焦写作前阶段、并未优化带引用文本的生成，我们仍考察了本方法所产文章的引用质量。如表 4 报告，Mistral 7B-Instruct 判定 84.83% 的句子得到其引用支持。附录 C.3 考察了不受支持的句子，发现主要问题源于**不当推断**与**不准确的转述**，而非虚构不存在的内容。

### 5.2 消融实验

> **表 5 | 不同方法收集到的去重参考文献平均数量（$|\mathcal{R}|$）。**

| 方法 | $|\mathcal{R}|$ |
| --- | --- |
| STORM | 99.83 |
| w/o Perspective | 54.36 |
| w/o Conversation | 39.56 |

如 §3 所述，STORM 通过发现特定视角与模拟多轮对话来提示 LLM 提出有效问题。我们在大纲创建上做消融，把 STORM 与两个变体比较：（1）**"STORM w/o Perspective"**，在问题生成提示中去掉视角；（2）**"STORM w/o Conversation"**，提示 LLM 一次性生成固定数量的问题。为保证公平比较，我们在所有变体中控制生成问题的总数相等。表 3 显示消融结果：完整 STORM 流程产出的提纲召回最高。此外，"STORM w/o Conversation" 结果差得多，说明**阅读相关信息对生成有效问题至关重要**。我们进一步统计不同变体在 $\mathcal{R}$ 中收集的去重来源数量。如表 5 所示，完整流程发现了更多不同来源，这一趋势与大纲质量的自动指标一致。

我们还验证了 STORM 中**大纲阶段是否必要**。表 2 中 "STORM w/o Outline Stage" 表示给定主题与模拟对话直接生成整篇文章的结果。去掉大纲阶段会使所有指标上的性能显著恶化。

---

## 6 人工评估

> **表 6 | 对 STORM 与 oRAG 生成的 20 对文章的人工评估结果。** 每对文章由两位维基百科编辑评估。评分采用 1-7 分制，$\geq 4$ 分表示质量良好（见表 10）。我们做配对 t 检验并报告 $p$ 值。

| 维度 | oRAG 均值 | oRAG ≥4 比例 | STORM 均值 | STORM ≥4 比例 | $p$ 值 |
| --- | --- | --- | --- | --- | --- |
| 趣味性 | 3.63 | 57.5% | 4.03 | 70.0% | 0.077 |
| 组织性 | 3.25 | 45.0% | 4.00 | 70.0% | 0.005 |
| 相关性 | 3.93 | 62.5% | 4.15 | 65.0% | 0.347 |
| 覆盖度 | 3.58 | 57.5% | 4.00 | 67.5% | 0.084 |
| 可验证性 | 3.85 | 67.5% | 3.80 | 67.5% | 0.843 |
| 偏好计数（#Preferred） | 14 | — | 26 | — | — |

为更好理解 STORM 的优势与弱点，我们与 10 位资深维基百科编辑合作开展人工评估 —— 他们至少在维基百科上有 500 次编辑、且经验超过 1 年。我们从数据集中随机抽取 20 个主题，评估由我们的方法与（按自动评估得出的最佳基线）oRAG 生成的文章。每对文章分配给 2 位编辑。

我们请编辑按 §4.2 定义的同样五个方面评判每篇文章，但使用 1-7 分制以获得更细粒度评价。虽然自动评估用引用质量作为可验证性的代理，人工评估则坚持维基百科"可验证、无原创研究"的标准。除打分外，编辑还需提供开放式反馈与成对偏好。评估完成后，进一步请他们把刚审阅过的、由我们方法生成的文章与人类撰写的对应文章比较，并用 1-5 的 Likert 量表报告其感知到的 STORM 有用性。更多人工评估细节见附录 D。表 6 给出评分与成对比较结果<sup>[11]</sup>。

**STORM 的文章比 oRAG 输出具有更大广度与深度。** 与 §5.1 的发现一致，编辑们认为 STORM 生成的文章比 oRAG 输出更有趣、更有组织、覆盖更广。具体而言，STORM 生成的文章中被认为"有组织"（组织性 ≥4）的多出 **25%**，被认为覆盖良好的（覆盖度 ≥4）多出 **10%**。即便与人类撰写文章相比，一位编辑称赞我们的结果"提供了多一点背景信息"，另一位则指出"我发现 AI 文章比维基百科文章更有深度"。在成对比较中 STORM 也胜过最佳基线。

**$\mathcal{R}$ 中更多信息带来的挑战不止于事实幻觉。** 我们考察了 14 份编辑更偏好 oRAG 输出的成对比较反馈。在剔除 3 个偏好与评分不一致的案例后，编辑们在我们方法生成的文章上给出更低可验证性评分的比例超过 50%。通过分析文章与编辑的自由反馈，我们发现低可验证性评分源于**红鲱鱼谬误（red herring fallacy）**或**过度推测**问题 —— 即生成的文章在 $\mathcal{R}$ 中不同信息片段之间、或信息与主题之间引入了**无法验证的关联**（示例见表 11）。与广受讨论的事实幻觉（Shuster et al., 2021; Huang et al., 2023）相比，解决这类可验证性问题更为微妙，超出了基础的事实核查（Min et al., 2023）。

> **图 3 | STORM 感知有用性的调研结果（n=10）。** *（图内为三组 Likert 分布条形图，对应"对写作前阶段有帮助""有助于编辑新主题的维基百科文章""对维基百科社区可能有用"三个陈述；正文给出的关键数字是：一致同意有助于写作前阶段、80% 认为能帮助编辑新主题条目、70% 认为对社区有用、仅 10% 不同意。）*

**生成文章仍落后于经过充分修订的人类作品。** 虽然 STORM 优于 oRAG 基线，编辑们仍评论生成文章的信息量不如真实维基百科页面。另一个被识别的主要问题是**来源偏见与语调迁移** —— 互联网来源的偏见与语气被转移到生成文章中，10 位编辑中有 7 位提到 STORM 生成的文章听起来"有情绪"或"不中立"。更多分析见附录 E。这一反馈表明，**减少写作前阶段的检索偏见**是值得投入的未来方向。

**生成文章是不错的起点。** 如图 3 所示，编辑们一致同意 STORM 能在写作前阶段帮到他们。知道这个工具对资深编辑有帮助令人欣慰。80% 的编辑认为 STORM 能帮他们为新主题编辑维基百科文章。对 STORM 之于整个维基百科社区的用处，保留意见更多；尽管如此，70% 的编辑认为有用，仅 10% 不同意。

---

## 7 相关工作

**检索增强生成（RAG）** 在推理时用检索增强语言模型，是利用外部知识库的典型方式（Ram et al., 2023; Izacard et al., 2023）。一些工作用检索为上下文学习构造示例（Li et al., 2023; Liu et al., 2022; Agrawal et al., 2023; Poesia et al., 2022; Shi et al., 2022; Khattab et al., 2022），另一类工作用检索为语言模型提供可据以落地的额外信息。Lewis et al. (2020) 在知识密集型 NLP 任务上研究 RAG，发现它提升了多样性与事实性。Semnani et al. (2023) 设计与英文维基百科对齐的 RAG 聊天机器人，以阻止 LLM 聊天机器人产生幻觉。此外，RAG 可用于生成带引用的文本（Menick et al., 2022; Gao et al., 2023），以及构建带归属的问答系统（Bohnet et al., 2023）。虽然 RAG 在问答中被广泛研究，如何将其用于长文生成则研究较少。

作为一个通用框架，RAG 在检索来源与检索时机上都很灵活。检索源可以从领域数据库（Zakka et al., 2023）、代码文档（Zhou et al., 2023）一直到整个互联网（Nakano et al., 2022; Komeili et al., 2022）。在时机上，除了生成前的一次性检索，系统也可被设计为在生成过程中自行决定何时检索（Jiang et al., 2023b; Parisi et al., 2022; Shuster et al., 2022; Yao et al., 2023）。

**自动说明性写作** 与其他类型的长文生成（Yang et al., 2022; Feng et al., 2018）不同，自动说明性写作要求以外部文档为依据，并利用阅读与写作之间的相互作用。Balepur et al. (2023) 提出 Imitate-Retrieve-Paraphrase 框架，在段落级做说明性写作，以应对多来源信息综合的挑战。超越摘要之外，Shen et al. (2023) 强调说明性写作要求作者对来源文档进行**意义建构（sensemaking）**以及良好的大纲规划。我们通过聚焦写作前阶段来应对这些挑战。

**NLP 中的提问** NLP 系统的提问能力已在多个方向扩展，包括生成澄清问题以理解用户意图（Aliannejadi et al., 2019; Rahmani et al., 2023），以及把大问题拆成小问题以改善组合推理（Press et al., 2023）。虽然人类通常通过提问来学习新知识（Tawfik et al., 2020; Booth et al., 2003），如何在信息寻求对话中优化问题的**信息量与具体性**仍研究不足。最接近的工作是 Qi et al. (2020)，它用 unigram 精确率函数定义问题信息量，并用强化学习提升问题信息量。

---

## 8 结论

我们提出 STORM —— 一个基于 LLM 的写作系统，自动化从零创建维基百科式文章的写作前阶段。我们构建了 FreshWiki 数据集并建立评估标准，以研究有据长文的生成。实验结果表明，STORM 中的提问机制同时提升了大纲质量与文章质量。凭借改进的广度与深度，STORM 通过专家评估为有据写作系统揭示了新挑战。参与研究的资深维基百科编辑一致认为 STORM 对其写作前阶段有帮助。

---

## 局限性

在本工作中，我们探索从零生成维基百科式文章，以此推动自动说明性写作与长文生成的前沿。虽然我们的方法在自动评估与人工评估中都显著优于基线方法，机器撰写文章的质量仍落后于经过充分修订的人类撰写文章，尤其在**中立性**与**可验证性**方面。尽管 STORM 在调研给定主题时发现了不同视角，收集到的信息仍可能偏向互联网上的主流来源，并可能含推广性内容。此外，本工作识别出的可验证性问题超出事实幻觉的范畴，这为有据写作系统提出了新挑战。

本工作的另一个局限是：虽然我们聚焦从零生成维基百科式文章这一任务，我们的任务设定仍被简化为只考虑**自由文本**的生成。人类撰写的高质量维基百科文章通常包含结构化数据与多模态信息。我们把多模态有据文章的生成留给未来工作。

## 致谢

我们感谢 You.com 慷慨提供支持本实验的搜索 API。也感谢 Sina J. Semnani、Shicheng Liu、Eric Zelikman 提供的有益反馈，以及 ACL ARR 审稿人的宝贵意见。本工作部分受 Verdant Foundation 与 Microsoft Azure AI credits 支持。Yijia Shao 受 Stanford School of Engineering Fellowship 支持。

## 伦理声明

与创造性生成不同，有据文章生成可能影响人们了解主题或消费来源信息的方式。本工作的所有研究与评估都旨在防止错误信息传播 —— 不在线发布生成内容，并实施严格的准确性检查。我们避免对维基百科或相关社区造成任何干扰，因为我们的系统不与线上页面交互。此外，尽管我们试图生成有据文章，我们认为本工作不存在隐私问题，因为我们只使用互联网上公开可用的信息。

本工作的主要风险在于：我们系统撰写的维基百科文章以互联网信息为依据，而互联网本身包含一些偏见性或歧视性内容。目前我们的系统依赖搜索引擎检索信息，但不含任何后处理模块。我们认为，改进检索模块以良好覆盖不同观点、并为现有系统添加内容筛选模块，将是实现生成文章更好中立性与平衡性的关键下一步。

从伦理角度看，另一个局限是我们本工作只考虑撰写英文维基百科文章。把现有系统扩展到多语言设定是未来工作的有意义方向，因为更多主题在非英语语言中尚无维基百科页面。

---

## 注释

> 原文脚注，按原编号保留。

**[1]** 我们的资源与代码发布于 https://github.com/stanford-oval/storm。

**[2]** 实践中 $\mathcal{S}$ 还包含组织结构元素，如章节与子章节标题，这些不需要引用。

**[3]** 取自 https://wikimedia.org/api/rest_v1/metrics/edited-pages/top-by-edits/en.wikipedia/all-editor-types/content/{year}/{month}/all-days 。

**[4]** https://www.mediawiki.org/wiki/ORES

**[5]** https://en.wikipedia.org/wiki/Wikipedia:Stand-alone_lists

**[6]** 由于语言模型处理并产生序列，我们可以通过加 "#" 表示章节标题、加 "##" 表示子章节标题等，把 $\mathcal{O}$ 线性化。

**[7]** https://pypi.org/project/Wikipedia-API/

**[8]** https://en.wikipedia.org/wiki/Wikipedia:Reliable_sources

**[9]** https://en.wikipedia.org/wiki/Wikipedia:Good_article_criteria

**[10]** https://documentation.you.com/api-reference/search

**[11]** 对各准则的 1-7 分制评分结果，我们计算 Krippendorff's Alpha 以衡量标注者间一致性（IAA），结果如下：趣味性（0.349）、组织性（0.221）、相关性（0.256）、覆盖度（0.346）、可验证性（0.388）。

**[12]** https://en.wikipedia.org/wiki/Wikipedia:Content_assessment

**[13]** https://huggingface.co/sentence-transformers/paraphrase-MiniLM-L6-v2

**[14]** https://huggingface.co/kaist-ai/prometheus-13b-v1.0

**[15]** https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.1

**[16]** 遵循 Gao et al. (2023)，我们在句子级检查引用质量，并用 NLTK 的 `sent_tokenize` 把文章切分为句子。当文章含"No.12847""Bhatia et al."等特殊写法时，`sent_tokenize` 有时无法正确切分。

**[17]** https://meta.wikimedia.org

**[18]** 因评估维基百科式文章耗时且需专业能力，我们为每位参与者支付 50 美元。

---

## 参考文献

（保留原引用，共 64 条；条目与原文编号一致，此处压缩为"引用键 + 标题"形式，完整出版信息见原文）

- [1] Agrawal et al. (2023) *In-context examples selection for machine translation*
- [2] Akbik et al. (2019) *FLAIR: An easy-to-use framework for state-of-the-art NLP*
- [3] Aliannejadi et al. (2019) *Asking clarifying questions in open-domain information-seeking conversations*
- [4] Balepur et al. (2023) *Expository text generation: Imitate, retrieve, paraphrase*
- [5] Banerjee and Mitra (2015) *WikiKreator: Improving Wikipedia stubs automatically*
- [6] Bohnet et al. (2023) *Attributed question answering: Evaluation and modeling for attributed large language models*
- [7] Booth et al. (2003) *The craft of research*
- [8] Dietz and Foley (2019) *Trec car y3: Complex answer retrieval overview*
- [9] Doyle (1994) *Information literacy in an information society: A concept for the information age*
- [10] Eriksson and Mäkitalo (2015) *Supervision at the outline stage: Introducing and encountering issues of sustainable development through academic writing assignments*
- [11] Fan and Gardent (2022) *Generating biographies on Wikipedia: The impact of gender bias on the retrieval-based generation of women biographies*
- [12] Feng et al. (2018) *Topic-to-essay generation with neural networks*
- [13] Fitria (2023) *Artificial intelligence (AI) technology in OpenAI ChatGPT application: A review of ChatGPT in writing English essay*
- [14] Fränti and Mariescu-Istodor (2023) *Soft precision and recall*
- [15] Freeman et al. (2010) *Stakeholder theory: The state of the art*
- [16] Gao et al. (2023) *Enabling large language models to generate text with citations*
- [17] Huang et al. (2023) *A survey on hallucination in large language models: Principles, taxonomy, challenges, and open questions*
- [18] Izacard et al. (2023) *Atlas: Few-shot learning with retrieval augmented language models*
- [19] Jiang et al. (2023a) *Mistral 7B*
- [20] Jiang et al. (2023b) *Active retrieval augmented generation*
- [21] Kandpal et al. (2023) *Large language models struggle to learn long-tail knowledge*
- [22] Khattab et al. (2022) *Demonstrate-search-predict: Composing retrieval and language models for knowledge-intensive NLP*
- [23] Khattab et al. (2023) *DSPy: Compiling declarative language model calls into self-improving pipelines*
- [24] Kim et al. (2023) *Prometheus: Inducing fine-grained evaluation capability in language models*
- [25] Komeili et al. (2022) *Internet-augmented dialogue generation*
- [26] Krishna et al. (2023) *LongEval: Guidelines for human evaluation of faithfulness in long-form summarization*
- [27] Lewis et al. (2020) *Retrieval-augmented generation for knowledge-intensive NLP tasks*
- [28] Li et al. (2023) *Unified demonstration retriever for in-context learning*
- [29] Lin (2004) *ROUGE: A package for automatic evaluation of summaries*
- [30] Liu et al. (2022) *What makes good in-context examples for GPT-3?*
- [31] Liu et al. (2018) *Generating Wikipedia by summarizing long sequences*
- [32] Menick et al. (2022) *Teaching language models to support answers with verified quotes*
- [33] Min et al. (2023) *FActScore: Fine-grained atomic evaluation of factual precision in long form text generation*
- [34] Minguillón et al. (2017) *Semi-automatic generation of a corpus of Wikipedia articles on science and technology*
- [35] Munoz-Luna (2015) *Main ingredients for success in L2 academic writing: Outlining, drafting and proofreading*
- [36] Nakano et al. (2022) *WebGPT: Browser-assisted question-answering with human feedback*
- [37] Ouyang et al. (2022) *Training language models to follow instructions with human feedback*
- [38] Parisi et al. (2022) *TALM: Tool augmented language models*
- [39] Pavlik (2023) *Collaborating with ChatGPT: Considering the implications of generative artificial intelligence for journalism and media education*
- [40] Poesia et al. (2022) *Synchromesh: Reliable code generation from pre-trained language models*
- [41] Press et al. (2023) *Measuring and narrowing the compositionality gap in language models*
- [42] Qi et al. (2020) *Stay hungry, stay focused: Generating informative and specific questions in information-seeking conversations*
- [43] Qian et al. (2023) *WebBrain: Learning to generate factually correct articles for queries by grounding on large web corpus*
- [44] Rahmani et al. (2023) *A survey on asking clarification questions datasets in conversational systems*
- [45] Ram (1991) *A theory of questions and question asking*
- [46] Ram et al. (2023) *In-context retrieval-augmented language models*
- [47] Reimers and Gurevych (2019) *Sentence-BERT: Sentence embeddings using Siamese BERT-networks*
- [48] Rohman (1965) *Pre-writing: The stage of discovery in the writing process*
- [49] Sauper and Barzilay (2009) *Automatically generating Wikipedia articles: A structure-aware approach*
- [50] Semnani et al. (2023) *WikiChat: Stopping the hallucination of large language model chatbots by few-shot grounding on Wikipedia*
- [51] Shen et al. (2023) *Beyond summarization: Designing AI support for real-world expository writing tasks*
- [52] Shi et al. (2022) *Nearest neighbor zero-shot inference*
- [53] Shuster et al. (2022) *Language models that seek for knowledge: Modular search & generation for dialogue and prompt completion*
- [54] Shuster et al. (2021) *Retrieval augmentation reduces hallucination in conversation*
- [55] Tardy (2010) *Writing for the world: Wikipedia as an introduction to academic writing*
- [56] Tawfik et al. (2020) *Role of questions in inquiry-based instruction: Towards a design taxonomy for question-asking and implications for design*
- [57] Weaver III and Kintsch (1991) *Expository text*
- [58] Wenzlaff and Spaeth (2022) *Smarter than humans? Validating how OpenAI's ChatGPT model explains crowdfunding, alternative finance and community finance*
- [59] Xu et al. (2023) *A critical evaluation of evaluations for long-form question answering*
- [60] Yang et al. (2023) *DOC: Improving long story coherence with detailed outline control*
- [61] Yang et al. (2022) *Re3: Generating longer stories with recursive reprompting and revision*
- [62] Yao et al. (2023) *ReAct: Synergizing reasoning and acting in language models*
- [63] Zakka et al. (2023) *Almanac: Retrieval-augmented language models for clinical medicine*
- [64] Zhou et al. (2023) *DocPrompting: Generating code by retrieving the docs*

---

## 附录 A 数据集细节

如 §2.1 所述，我们通过收集近期、高质量的英文维基百科文章构建 FreshWiki 数据集。我们选取特定时间段内**编辑次数最多的页面**，而不是以创建日期作截断，因为大多数维基百科文章在创建时只是"存根（stub）"或质量很低。质量方面，我们只考虑被预测为 B 级或更高的文章。根据维基百科统计<sup>[12]</sup>，现有维基百科页面中只有约 3% 达到该质量标准。由于 LLM 能生成相当不错的输出，我们认为使用高质量人类撰写文章作为参照对进一步研究很重要。

> **表 7 | 实验所用数据集的统计量。**

| 统计项 | 数值 |
| --- | --- |
| 平均章节数 | 8.4 |
| 平均全层级标题数 | 15.8 |
| 平均单节长度 | 327.8 |
| 平均全文长度 | 2159.1 |
| 平均参考文献数 | 90.1 |

> **图 4 | 维基百科文章编辑过程中参考文献数量的演化。** *（图内为一条随编辑轮次上升的折线：参考文献数在初期快速攀升，随后增速放缓并趋于平台，说明"引用充足"是反复编辑累积的结果。）*

> **图 5 | 实验中维基百科文章编辑次数的分布（n=100）。** *（图内为编辑次数的直方图，显示样本的编辑次数跨度很大，多数文章需要数十至上百次编辑。）*

在本工作的实验中，我们随机选取 100 个、其人类撰写文章不足 3000 词的样本以进行有意义的比较。表 7 给出数据统计量。值得注意的是，人类撰写的文章拥有大量参考文献，但这需要多次编辑才能达成。图 4 展示了文章编辑过程中参考文献数量的演化，图 5 给出实验所用人类撰写文章的编辑次数分布。

---

## 附录 B STORM 伪代码

在 §3 中，我们介绍了 STORM —— 一个通过发现不同视角、模拟信息寻求对话、创建全面大纲来自动化写作前阶段的框架。算法 1 展示 STORM 的骨架。

我们用 DSPy 框架（Khattab et al., 2023）以零样本提示实现 STORM。Listing 6 与 7 给出实现中使用的提示词。我们要强调，STORM 提供的是一个**通用框架**，用于辅助创建有据长文，并不过度依赖针对单一领域的提示工程。

**算法 1：STORM**

```text
输入：主题 t，最大视角数 N，最大对话轮数 M
输出：大纲 O，参考文献集合 R

1  P0 = "基本事实写作者 ……"          // 常量视角
2  R ← []
3  // 发现视角 P
4  related_topics ← gen_related_topics(t)
5  tocs ← []
6  for each related_t in related_topics do
7      article ← get_wiki_article(related_t)
8      if article then tocs.append(extract_toc(article))
10 end for
11 P ← gen_perspectives(t, tocs)
12 P ← [P0] + P[:N]
13 // 模拟对话
14 convos ← []
15 for each p in P do
16     convo_history ← []
17     for i = 1 to M do
18         // 提问
19         q ← gen_qn(t, p, convo_history)
20         convo_history.append(q)
21         // 回答
22         queries ← gen_queries(t, q)
23         sources ← search_and_sift(queries)
24         a ← gen_ans(t, q, sources)
25         convo_history.append(a)
26         R.append(sources)
27     end for
28     convos.append(convo_history)
29 end for
30 // 创建大纲
31 O_D ← direct_gen_outline(t)
32 O ← refine_outline(t, O_D, convos)
33 return O, R
```

**Listing 6 / 7：STORM 使用的提示词**（对应算法 1 的第 4、11、19、22、24、31、32 行；原文为英文提示词，此处给出中文译文，逐字原文见论文 Listing 6-7 与代码仓库）

| 提示词 | 用途（对应行号） | 中文译文 |
| --- | --- | --- |
| `GenRelatedTopicsPrompt` | 生成相关主题（第 4 行） | "我正在为下面提到的主题撰写一个维基百科页面。请识别并推荐一些主题密切相关的维基百科页面。我希望得到这样的例子：能提供关于该主题常见的有趣方面的洞见，或能帮助我理解类似主题的维基百科页面通常包含哪些内容与结构。请把 URL 分行列出。" |
| `GenPerspectivesPrompt` | 生成视角（第 11 行） | "你需要选出一组维基百科编辑，共同为一篇关于该主题的全面文章工作。他们每人代表与该主题相关的不同视角、角色或所属机构。你可以用其他相关主题的维基百科页面作为灵感。对每位编辑，请补充说明他将聚焦什么。按以下格式作答：1. 编辑1的简称: 描述\n2. 编辑2的简称: 描述\n……" |
| `GenQnPrompt` | 生成问题（第 19 行） | "你是一位经验丰富的维基百科写作者，想编辑某个特定页面。除维基百科写作者身份外，你在调研该主题时还有一个特定关注点。现在你在与一位专家对话以获取信息。请提出好问题以获得更有用的信息。当你没有更多问题时，说'非常感谢你的帮助！'以结束对话。请一次只问一个问题，不要重复问过的内容。你的问题应与你想写的主题相关。" |
| `GenQueriesPrompt` | 拆解搜索查询（第 22 行） | "你想用 Google 搜索来回答该问题。你在搜索框里输入什么？按以下格式写出你将使用的查询：- 查询1\n- 查询2\n……" |
| `GenAnswerPrompt` | 生成回答（第 24 行） | "你是一位能有效利用信息的专家。你正在与一位维基百科写作者对话，他想就你所了解的主题撰写维基百科页面。你已收集到相关信息，现在将用这些信息组织回答。请让回答尽可能信息丰富，并确保每一句都有所收集信息的支持。" |
| `DirectGenOutlinePrompt` | 直接生成草稿大纲（第 31 行） | "为一个维基百科页面写一份大纲。写作格式：1. 用 '# 标题' 表示章节标题，'## 标题' 表示子章节标题，'### 标题' 表示三级标题，依此类推。2. 不要包含其他信息。" |
| `RefineOutlinePrompt` | 精炼大纲（第 32 行） | "改进一个维基百科页面的大纲。你已有一份覆盖通用信息的草稿大纲。现在你要基于从信息寻求对话中学到的信息改进它，使其更全面。写作格式同上：1. 用 '#' 表示章节标题等层级；2. 不要包含其他信息。" |

---

## 附录 C 自动评估细节

### C.1 标题软召回

我们计算生成大纲中的多级标题（作为预测 $P$）与人类撰写文章中的多级标题（作为真值 $G$）之间的**软标题召回**。计算基于 Fränti and Mariescu-Istodor (2023) 的软召回定义。给定集合 $A=\{A_i\}_{i=1}^{K}$，某一项的**软计数**定义为其与集合内其他项相似度之和的倒数：

$$
\operatorname{count}(A_i)=\frac{1}{\sum_{j=1}^{K}\operatorname{Sim}(A_i,A_j)},\qquad
\operatorname{Sim}(A_i,A_j)=\cos(\operatorname{embed}(A_i),\operatorname{embed}(A_j)) \tag{1}
$$

其中式 (1) 中的 $\operatorname{embed}(\cdot)$ 由 Sentence-Transformers 库提供的 `paraphrase-MiniLM-L6-v2` 参数化<sup>[13]</sup>。$A$ 的**基数**是其各项计数之和：

$$
\operatorname{card}(A)=\sum_{i=1}^{K}\operatorname{count}(A_i) \tag{2}
$$

软标题召回计算为：

$$
\text{soft heading recall}=\frac{\operatorname{card}(G\cap P)}{\operatorname{card}(G)} \tag{3}
$$

其中交集的基数通过并集定义：

$$
\operatorname{card}(G\cap P)=\operatorname{card}(G)+\operatorname{card}(P)-\operatorname{card}(G\cup P) \tag{4}
$$

### C.2 LLM 评估器

> **表 8 | 评估器 LLM 使用的 1-5 分制评分标准。**

| 准则 | 得分 | 描述 |
| --- | --- | --- |
| **趣味性**：文章有多引人入胜、发人深思？ | 1 | 完全不吸引；没有任何抓住读者注意力的尝试。 |
| | 2 | 较为吸引，有基本叙述但缺乏深度。 |
| | 3 | 中等吸引，有若干有趣之处。 |
| | 4 | 相当吸引，叙述结构良好，并有能频繁抓住并保持注意力的要点。 |
| | 5 | 全程极为吸引，叙述有说服力并持续激发兴趣。 |
| **连贯性与组织性**：文章是否有条理、逻辑清晰？ | 1 | 杂乱无章；缺乏逻辑结构与连贯性。 |
| | 2 | 组织一般；有基本结构但未始终遵循。 |
| | 3 | 有组织；清晰的框架大体被遵循，但有若干连贯性失误。 |
| | 4 | 组织良好；结构清晰，仅有轻微连贯性失误。 |
| | 5 | 组织极佳；文章逻辑严密、过渡自然、论证清晰。 |
| **相关性与聚焦**：文章是否紧扣主题、保持清晰聚焦？ | 1 | 跑题；内容与标题或核心主题不符。 |
| | 2 | 部分相关但有多处离题；核心主题可见但未始终紧扣。 |
| | 3 | 总体相关，尽管有少数无关细节。 |
| | 4 | 多数情况下相关且聚焦；叙述与核心主题保持稳定关联，离题少见。 |
| | 5 | 极度聚焦、完全扣题；每一处信息都在为读者全面理解主题作贡献。 |
| **广泛覆盖**：文章是否对主题做了深入探讨、覆盖良好？ | 1 | 严重不足；几乎未覆盖主题的主要方面，视角极窄。 |
| | 2 | 部分覆盖；涵盖部分主要方面但遗漏其他方面，呈现不完整。 |
| | 3 | 广度可接受；覆盖多数主要方面，但可能偏入次要细节或忽略部分相关要点。 |
| | 4 | 覆盖良好；实现广泛覆盖，命中所有要点且多余信息很少。 |
| | 5 | 广度为典范；覆盖出色，详尽刻画所有关键方面且不含无关信息。 |

我们使用 Prometheus<sup>[14]</sup>（Kim et al., 2023）—— 一个 13B 开源评估器 LLM，可依据自定义的 1-5 分制标准评估长文本 —— 从趣味性、连贯性与组织性、相关性与聚焦、覆盖度四个方面为文章打分。表 8 给出我们的评分标准。虽然 Prometheus 最佳用法是配合一个 5 分参考回答，但我们发现加入参考会超出模型上下文长度限制。由于 Kim et al. (2023) 表明无参考时 Prometheus 的评分也与人类偏好良好相关，我们省略参考，并通过从最短的章节开始迭代删减内容，把输入文章裁剪到 2000 词以内，以确保输入能放入模型上下文窗口。

### C.3 关于引用质量的更多讨论

> **表 9 | 不受支持句子的不同错误类型示例。**

| 错误类型 | 主题 | 不受支持的句子 | 来源 |
| --- | --- | --- | --- |
| 不当推断性关联 | Lahaina, Hawaii | "纵观其历史，宗教始终是拉海纳夏威夷人生活中最重要的方面，渗透在日常活动与重大事件之中[5]。" | [5] "宗教、信仰与灵性"（该来源讨论宗教作为夏威夷生活的一部分，但未提及拉海纳。） |
| 不准确的转述 | 2022 年克里米亚大桥爆炸 | "该桥于 2020 年 6 月完工，是俄军在该地区的主要补给线路，对俄罗斯关于该争议领土的主张具有重要意义[2][11]。" | [2] "克里米亚大桥 - 维基百科"（来源称"首列定期客运列车于 2019 年 12 月 25 日通过该桥，而货运列车于 2020 年 6 月 30 日通行"。） |
| 引用无关来源 | LK-99 | "例如，有人把 LK-9 的性能与《战地 2042》等电子游戏的动态分辨率能力作了比较[22]。" | [22] "《战地 2042》PC 性能指南：高帧率的最佳设置"（该来源与 LK-99 无关。） |

> **图 8 | 对 10 篇抽样文章中不受支持句子的错误分析。** *（图内为按错误类型统计的分布，正文结论是：最常见错误来自 LLM 在上下文窗口内不同信息片段之间形成不当推断性关联。）*

我们使用 Mistral 7B-Instruct<sup>[15]</sup>（Jiang et al., 2023a）检查被引用段落是否蕴含生成的句子。表 4 报告了我们方法所产文章的引用质量，显示生成文章中约 15% 的句子未得到引用支持。我们进一步通过随机抽取 10 篇文章、由一位作者人工检查其中所有不受支持句子来研究失败案例。除了被错误切分的句子<sup>[16]</sup>、缺少引用、或被作者判断为实际有支持的句子之外，我们的分析识别出三类主要错误（示例见表 9）：**不当推断性关联**、**不准确的转述**、**引用无关来源**。

我们在图 8 中展示错误分布。值得注意的是，最常见的错误源于 LLM 倾向于在上下文窗口内呈现的不同信息片段之间形成**不当的推断性关联**。我们对引用质量的分析表明：除避免幻觉之外，未来关于有据文本生成的研究还应关注防止 LLM 基于所给信息做出**过度推断的跳跃**。

---

## 附录 D 人工评估细节

我们通过在 Meta-Wiki<sup>[17]</sup> 上创建研究页面、并联系近期为维基百科批准过条目的活跃编辑，招募了 10 位资深维基百科编辑参与研究<sup>[18]</sup>。由于评估维基百科式文章耗时且需要专业知识，我们为每位参与者支付 50 美元。参与者群体包括 3 位有 1-5 年经验的编辑、4 位有 6-10 年经验的、3 位贡献超过 15 年的。本研究经我们机构的伦理审查委员会（IRB）批准，参与者在研究开始前通过 Qualtrics 问卷签署知情同意书。

为简化有据文章的评估，我们开发了一个 Web 应用，并排展示文章与其引用片段，用于收集每篇文章的评分与开放式反馈。图 9 展示了该 Web 应用的截图，STORM 产出的完整文章见表 12。人工评估使用 1-7 分制以获得更细粒度评价，评分标准见表 10。

我们通过在线问卷收集成对偏好与对 STORM 的感知有用性。具体而言，关于感知有用性，我们请编辑以 1-5 Likert 量表（对应"非常不同意、有些不同意、既不同意也不反对、有些同意、非常同意"）评价他们对以下陈述的同意程度："我认为它能对我的写作前阶段有具体帮助（例如收集相关来源、列大纲、起草）。"、"我认为它能帮我为新主题编辑维基百科文章"、"我认为它可能是对维基百科社区有用的工具"。

> **图 9 | 用于评估有据文章的 Web 应用截图。** *（界面左侧为文章正文，右侧同步高亮该句所引用的来源片段，编辑可就每个维度打分并填写自由评论。）*

> **表 10 | 人工评估使用的 1-7 分制评分标准。**（同一套五准则，在原 5 分制基础上插入中间档，形成 7 级）

| 分数 | 趣味性 | 连贯性与组织性 | 相关性与聚焦 | 广泛覆盖 | 可验证性 |
| --- | --- | --- | --- | --- | --- |
| 1 | 完全不吸引，无抓注意力尝试 | 杂乱无章，缺乏逻辑与连贯 | 跑题，与标题/核心不符 | 严重不足，视角极窄 | 无支持证据，主张无依据 |
| 2 | 略有吸引，偶有抓注意力时刻 | 组织较差，结构可见但很弱 | 大体跑题但有部分相关 | 覆盖极少，仅触及少量方面且有明显遗漏 | 很少证据，大量主张无依据 |
| 3 | 一般，有基本叙述但缺深度 | 一般，有基本结构但未始终遵循 | 大致相关但有多处离题 | 部分覆盖，含部分主要方面但遗漏其他 | 验证不一致，部分主张有支持 |
| 4 | 较为吸引，有若干有趣点 | 有组织，清晰结构大体遵循，有连贯性失误 | 总体相关，少量无关细节 | 广度可接受，覆盖多数主要方面，或有琐碎偏入 | 大体可验证，通常有证据，偶有缺失 |
| 5 | 相当吸引，结构良好，常抓住注意力 | 组织良好，结构清晰，轻微失误 | 多数相关且聚焦，离题少见 | 覆盖良好，广覆盖主要点，冗余少 | 支持良好，证据可信，无支持主张罕见 |
| 6 | 很有吸引力，叙述有说服力并基本维持注意力 | 组织很好，逻辑结构配合有效过渡引导读者 | 高度相关，叙述聚焦且有目的 | 全面，各方面充分覆盖且重点平衡 | 支持很好，几乎每条主张均有可信证据 |
| 7 | 全程极佳，持续激发兴趣 | 出色，逻辑严密、过渡无缝、论证清晰 | 极度聚焦、完全扣题，每条信息都贡献于全面理解 | 广度典范，详尽覆盖所有关键方面且无无关信息 | 验证典范，每条主张均由权威来源的可靠证据支持，严格遵循"无原创研究"政策 |

---

## 附录 E 错误分析

> **表 11 | STORM 所产文章上发现的主要问题汇总。**

| 问题 | 被提及次数 | 示例评论 |
| --- | --- | --- |
| 使用情绪化词汇、不中立 | 12 | "'significant' 一词在本文中出现了 17 次。关于更广泛政治重要性与'关键作用'的表述含糊且缺乏支持，不符合百科体例。"（评文章 *Lahaina, Hawaii*）<br>"……但他们仍未解决中立观点的问题。本文明显可见作者的立场偏向 Taylor Swift。除此之外，它在总结要点并深入挖掘方面做得不错。"（评 *Speak Now (Taylor's Version)*）<br>"'这部电影还曾在加州捐赠基金会主办的艺术与电影节上展映，凸显了故事在重塑社区叙事方面的力量。'是的，来源确实这么说，但用维基百科的语气这样写很牵强，听起来就是非中立的推广文字。"（评 *Gehraiyaan*） |
| 红鲱鱼谬误、关联无关来源 | 11 | "不应纳入美国的民调，也不应建立与气候变化的关联，除非来源明确建立了这种联系。"（评 *Typhoon Hinnamnor*）<br>"来源大体没问题，但有些并不直接相关（如 39、40）。"（评 *Gehraiyaan*）<br>"这里关于 KISS 乐队的冗长离题并不必要，应该直接链接到该乐队的条目。"（评 *2022 AFL Grand Final*） |
| 遗漏重要信息 | 6 | "'一项由劳伦斯伯克利国家实验室物理学家 Sinéad Griffin 主导的研究，用超级计算机模拟对 LK-99 的性能作了分析[20]。'关于该分析的介绍不够充分，而这些信息在文章中本来很有用。"（评 *LK-99*）<br>"虽然地震的即时后果与应对措施得到了充分覆盖，但关于长期社会经济影响与恢复过程可以写得更多。"（评 *2022 West Java earthquake*） |
| 对时效性信息处理不当 | 5 | "维基百科文章应避免使用'现在'这类词以防过时，而应改用'截至 2023 年 12 月'之类的表述。"（评 *Cyclone Batsirai*）<br>"'截至 12 月 13 日'未指明具体年份，且是过时信息。"（评 *2022 West Java earthquake*） |
| 章节组织问题 | 5 | "'恢复与重建'一节下的子章节太多。"（评 *2022 West Java earthquake*）<br>"我不喜欢这篇文章的组织方式，太多标题把文章切得很碎，可读性下降。除此之外，AI 在这篇稿子上做得很好。"（评 *2022 Crimean Bridge explosion*） |

尽管 STORM 产出的文章在自动指标与人工评估中都更受偏好，资深编辑仍然指出了机器生成文章的多个问题。我们分析自由评论并在表 11 中归纳主要问题。

被提出的首要问题是：生成文章常含情绪化语言、缺乏中立性，**这主要源于来源材料本身**。STORM 目前从互联网检索依据来源，而互联网本身并不中立、还包含大量推广内容。在写作前阶段处理这一偏见是未来研究的有价值方向。另一个主要问题是**红鲱鱼谬误**，即不相关事实的过度关联。应对这一挑战需要**高层意义建构**，而非仅停留在事实层面的核查。

> **表 12 | STORM 为"Taylor Hawkins"生成的文章。** "#""##"分别表示章节标题与子章节标题；方括号中的数字表示所引用的参考文献。*（原文该表约 2000 词，此处给出其骨架结构与导言部分摘译。）*

**生成文章的大纲结构**：# Early Life and Background（早年与背景）、# Career（职业生涯）、# Musical Style and Influences（音乐风格与影响）、# Personal Life（个人生活）、# Legacy and Impact（遗产与影响）、# Discography（作品目录）→ ### Taylor Hawkins & The Coattail Riders、### Red Light Fever、## Get the Money、# Collaborations and Guest Appearances（合作与客串）、# Tragic Passing（骤然离世）→ ## Tributes and Remembrances（悼念与纪念）。

**导言段（摘译）**："Oliver Taylor Hawkins（1972-2022）是一位美国音乐人，最广为人知的身份是摇滚乐队 Foo Fighters 的鼓手[1]。Hawkins 出生于德克萨斯州 Fort Walton，年幼时便爱上音乐，尤其是在 1982 年看过一场 Queen 的演唱会后[2][3][5]。他的职业生涯起步于为 Alanis Morissette 的乐队担任鼓手，参与其热门专辑《Jagged Little Pill》的世界巡演[8][9]。他的才华被 Dave Grohl 看中，1997 年受邀加入 Foo Fighters，由此开启了他在该乐队影响深远的任期[7][8]。"

*（译者注：该附录的价值不在文本本身，而在于可以直观看到 STORM 输出的「结构 + 引用密度 + 语气」三件事：大纲层级完整、几乎每句都带引用（有时一句挂多个引用）、并且确实出现了表 11 所说的"情绪化/推广式"措辞，例如 "unforgettable icon"、"indelible mark"、"Tragic Passing" 这类不符合百科中立语气的表述。）*

---

## 术语与背景速览（译者整理）

| 术语 | 含义 |
| --- | --- |
| 写作前阶段（Pre-writing Stage） | 动笔前的"发现"阶段：调研主题、收集来源、拟定大纲。本文的全部创新集中在这一阶段（Rohman, 1965 的写作过程理论） |
| 说明性写作（Expository Writing） | 以有条理地告知读者某主题为目的的写作（区别于叙事/创意写作），要求以外部文档为依据并做意义建构 |
| 视角（Perspective） | 一个人调研某主题时的关注点与身份（如"活动策划者""经济学家"）。本文用它作为提问的**先验**，从而引出更深入的问题 |
| 多视角提问（Multi-perspective Question Asking） | STORM 的核心：先发现 N 个视角，再让 LLM 分别扮演这些视角并行提问，从而覆盖同一主题的不同侧面 |
| 模拟对话（Simulated Conversation） | 由 LLM 扮演"写作者"提问、扮演"主题专家"作答；专家的回答必须由实时检索到的可信网页支撑。这是"迭代调研"的实现方式 |
| $p_0$ 基本事实写作者 | 恒定的第 0 个视角，专门兜住主题的基础事实，防止多视角都跑向专业细节而漏掉常识性内容 |
| Direct Gen | 基线之一：直接提示 LLM 生成大纲/整篇文章，不使用检索 |
| RAG | 基线之一：以主题做一次检索，把检索结果塞进上下文生成大纲或全文 |
| oRAG（大纲驱动的 RAG） | 本文最强基线：用 RAG 生成大纲后，再用每个章节标题去检索，逐节生成文章 |
| RAG-expand | 为探 RAG 上限而构造的增强版：在 RAG 大纲基础上，再用其章节标题检索一轮并精炼大纲 |
| 标题软召回（Soft Heading Recall） | 用 Sentence-BERT 嵌入的余弦相似度做"柔性"集合计数，允许语义相近但字面不同的标题互相匹配，避免精确匹配的过严要求 |
| 标题实体召回（Heading Entity Recall） | 人类文章标题中的命名实体（FLAIR NER 抽取）被生成大纲覆盖的百分比，衡量"是否提到了该提的专有名词" |
| FreshWiki | 本文构建的数据集：2022-02 至 2023-09 每月编辑量前 100 的页面，筛到 ORES B 级以上、非列表、有子章节，用于规避预训练数据泄漏 |
| ORES | 维基媒体官方的条目质量自动评估服务，本文用它筛"B 级以上"的高质量条目 |
| Prometheus | 13B 开源"评估器 LLM"，可依据自定义 rubric 给长文本打分，本文用它替代昂贵的人工打分 |
| 引用召回 / 引用精确率 | 参照 Gao et al. (2023)：前者衡量"该被引支持的句子是否被引"，后者衡量"引用的来源是否真的支持该句"（用 Mistral 7B-Instruct 做蕴含判断） |
| 红鲱鱼谬误（Red Herring Fallacy） | 把不相关的信息片段强行建立关联，从而把读者引向错误结论。本文发现它比"事实幻觉"更隐蔽、更难治 |
| 来源偏见迁移（Source Bias Transfer） | 检索到的网页本身携带情绪化/推广性语气，生成模型把它原样搬进文章，导致"不中立" |
| DSPy | 斯坦福 NLP 的"声明式 LM 调用"框架，把提示词写成 Signature，便于编译与复用；STORM 用零样本 Signature 实现，不做逐领域提示工程 |

**核心数字速记**

- 写作前阶段的收益是可量化的：GPT-4 下标题软召回 **92.73 vs 89.55**（oRAG）、GPT-3.5 下 **86.26 vs 73.59** —— 弱模型受益更大，因为"有组织的检索结果"比"一堆原始网页"更好用
- 提纲的改进主要来自**模拟对话**，而非视角：去掉对话后 GPT-4 软召回从 92.73 掉到 **88.75**、实体召回从 45.91 掉到 **39.30**；去掉视角只掉到 92.39 / 42.70
- 收集到的**去重来源数**是最直观的过程指标：完整 STORM **99.83**，无视角 54.36，无对话 39.56 —— 对话把检索量翻了 2.5 倍
- **大纲阶段是全文质量的命门**：去掉它 ROUGE-1 从 45.82 崩到 **26.77**，实体召回 14.10 → 7.39，几乎退回 Direct Gen 水平
- 人工评估 20 对文章、10 位资深编辑：偏好 STORM **26 : 14**；组织性 ≥4 的比例 **70% vs 45%（+25pp）**，覆盖度 **67.5% vs 57.5%（+10pp）**；唯一不显著甚至略降的是可验证性（3.80 vs 3.85）
- 引用质量不是本文优化目标，但已可用：引用召回 **84.83%**、精确率 **85.18%**；失败案例中**不当推断性关联**占比最高，而非凭空编造
- 编辑们的保留意见集中在"不中立"：**7/10** 提到文章"有情绪/不中立"，说明瓶颈已从"事实对不对"转移到"语气与关联是否正当"
- 但工具价值被认可：**100%** 编辑同意它对写作前阶段有帮助，**80%** 认为能帮自己编辑新主题条目，**70%** 认为对维基百科社区有用（仅 10% 不同意）
- 数据集画像：平均 **8.4 节 / 15.8 个全层级标题 / 全文 2159 词 / 90.1 条引用**；人类文章的引用数是靠几十上百次编辑累积出来的（图 4），这解释了为什么"一次生成就有 99.8 条来源"值得注意
