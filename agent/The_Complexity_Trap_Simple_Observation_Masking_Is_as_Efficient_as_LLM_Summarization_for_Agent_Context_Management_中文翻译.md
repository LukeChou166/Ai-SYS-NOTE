# 复杂性陷阱：简单的观察屏蔽与 LLM 摘要在智能体上下文管理中同样高效

> **原文**: The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization for Agent Context Management
> **作者**: Tobias Lindenbauer, Igor Slinko, Ludwig Felder, Egor Bogomolov, Yaroslav Zharov（JetBrains Research / 慕尼黑工业大学）
> **日期/arXiv**: 2025-08-29 提交，v3 更新于 2025-10-27（arXiv:2508.21433）
> **发表**: NeurIPS 2025 第 4 届 DL4C Workshop（Deep Learning for Code in the Agentic Era）camera-ready
> **链接**: https://arxiv.org/abs/2508.21433
> **代码**: https://github.com/JetBrains-Research/the-complexity-trap ｜ **数据**: https://huggingface.co/datasets/JetBrains-Research/the-complexity-trap
> **译者说明**: 本文为完整全文翻译（NeurIPS 检查清单为标准化模板内容，以导览表呈现）。带 *（译者注）* 标记的段落为译者补充的背景说明，非原文内容。

---

## 摘要

基于大语言模型（LLM）的智能体通过迭代推理、探索和工具调用来解决复杂任务，这一过程会产生漫长且昂贵的历史上下文。虽然 OpenHands、Cursor 等最先进的软件工程（SE）智能体使用基于 LLM 的摘要（summarization）来应对这一问题，但与简单地省略较旧的观察（observation）相比，这种增加的复杂度是否能带来切实的性能收益，尚不清楚。我们在 SWE-bench Verified 上、用 SWE-agent 智能体框架、跨五种不同的模型配置，对这两类方法进行了系统比较。此外，我们还给出了初步证据，表明我们的发现在 OpenHands 智能体框架上同样成立。

我们发现，一种简单的环境观察屏蔽（Observation Masking）策略相对于原始智能体（Raw Agent）能将成本减半，同时达到——有时甚至略微超过——LLM 摘要（LLM-Summary）的解决率。此外，我们提出了一种新颖的混合方法，与单独使用观察屏蔽或 LLM 摘要相比，可进一步分别降低 7% 和 11% 的成本。我们的发现对纯 LLM 摘要的趋势提出了质疑，并提供了初步证据表明：通过推进"效率—效果"前沿，仍有尚未挖掘的成本降低空间。我们开源了代码与数据以保证可复现性。

---

## 1 引言

创建能独立处理复杂软件工程任务的自主智能体，这一愿景正在迅速成为现实。这些由 LLM 驱动的智能体通常运行在一个迭代循环中 [39, 29]：每一轮（turn），智能体对当前状态进行推理、制定计划并执行一个工具（如读取文件、运行测试）。工具的输出（即观察）随后被加入智能体的上下文，供下一轮使用，从而不断扩展其问题求解轨迹（trajectory）（见图 3）。这种智能体循环（agentic loop）是一种强大的测试时扩展（test-time scaling）机制 [25, 17, 32]：它在每一轮利用 LLM 的推理能力 [31]，同时通过环境响应将推理落地（grounding）。

然而，这种上下文的迭代扩张带来了成本与能力（即效果 effectiveness 与效率 efficiency）之间的根本性权衡。随着智能体轨迹的增长，基于 token 计费的 LLM 调用变得极其昂贵；而且在广泛使用的 Transformer 架构 [28] 下，注意力的二次复杂度也使其效率低下。更关键的是，即使上下文窗口超过 100 万 token，LLM 仍然受到"迷失在中间"（lost in the middle）问题的困扰 [16]：LLM 虽然能处理很大的上下文窗口，却无法正确利用埋藏在庞大上下文中的相关信息 [19, 7]。这一挑战在软件工程领域被急剧放大，因为该领域的工具观察出了名地冗长且充满噪声 [15]：单个动作就可能产生数千 token——无论是读取整个源码文件、递归列出目录，还是冗长的测试套件日志。具体而言，在我们的初步实验（附录 D.4，基于 SWE-bench Lite-50 [11, 3]）中，观察 token 约占 SWE-agent 单轮平均 token 的 84%（图 1）。因此，显式地针对环境观察进行处理，为 LLM 智能体上下文管理提供了一个强有力的切入点。

> **图 1 | 环境观察 token 主导了软件工程智能体轨迹的上下文窗口。** 该图展示了原始智能体轨迹中各类 token（推理 r、动作 a、观察 o）的占比：环境观察 token 占约 84%，是绝对主导；智能体自身的推理和动作 token 只占很小份额。这正是"针对观察做压缩"能带来巨大收益的原因。

在没有针对成本效率的上下文管理策略的情况下，我们发现智能体成本可能翻倍以上（表 1）——这使得上下文管理不仅是一种优化手段，更是一种经济上的必需品。事实上，我们的结果表明，本文讨论的任何管理策略都优于不管理：它们都能持续降低成本，且常常能提升性能。这就引出一个关键问题：哪种策略提供了最佳的权衡？

一个自然的基线做法是通过观察屏蔽（observation masking）策略直接针对冗长的环境观察，显式地丢弃工具输出。SWE-agent [35]、SWE-Search [2] 等最先进系统都采用了这类相对简单的方法。与此同时，一种日益流行且更复杂的方案是让 LLM 执行轨迹摘要（trajectory summarization），用压缩后的摘要替换智能体历史的一部分（图 3）。OpenHands [30]、Cursor [9] 等知名开源与商业软件工程智能体都采用了这一方案。这两类方法的开源实现都依赖启发式触发条件，如固定的上下文窗口大小或轮数阈值。因此，两者的关键区别在于：是**丢弃**旧上下文，还是**压缩**旧上下文。尽管这一选择对智能体的成本和性能影响重大，但两类方法之间的相对权衡在很大程度上仍未被研究过。

本文对上下文管理策略的效率进行了系统比较。我们分析了具有代表性的开源观察屏蔽与 LLM 摘要实现在两方面指标上的表现：效率（美元成本）与效果（在业界标准且具挑战性的 SWE-bench Verified [5] 基准上的解决率）。下文中我们将这两个策略名首字母大写，以指代第 3.1 节中正式定义的具体策略。为了在多种模型配置间进行受控实验，我们在 SWE-agent [35] 中实现了 LLM-Summary，并改编了 OpenHands 的 LLM-Summary 提示词（图 11）。我们在 SWE-agent [35] 框架内评估这些策略，并在 OpenHands [30] 上检验其普适性。实验覆盖不同的模型家族、模型规模、许可类型（开放权重 vs. 专有）以及推理模式（thinking vs. 非 thinking）。

我们发现：观察屏蔽和 LLM 摘要都使成本相比原始智能体降低了一半以上；此外，在所有模型配置上，LLM-Summary 都无法持续或显著地超越简单的观察屏蔽策略。我们给出了这些发现在 OpenHands 上同样成立的初步证据。进一步地，我们提出一种新颖的混合方法，与单独使用观察屏蔽或 LLM 摘要相比，可进一步分别降低 7% 和 11% 的成本。这些发现挑战了当前纯 LLM 摘要的趋势，并表明纯 LLM 摘要策略很可能留下了相当可观的、尚未利用的成本节省空间。

> **图 2 | SWE-agent 框架内各上下文管理策略在 SWE-bench Verified 上的效果—效率权衡。** 横轴为每条轨迹的平均成本（越低越好），纵轴为解决率（越高越好）。每个模型配置测试三种策略：原始智能体（基线，●）、LLM-Summary（◼）、观察屏蔽（▲）。在所有模型上，观察屏蔽策略都稳定占据最有效的帕累托前沿，解决率与 LLM-Summary 相当甚至更优。以实验中表现最好的模型 Qwen3-Coder 480B [27, 8] 为例：观察屏蔽不仅比原始智能体基线便宜 52%，还将解决率提高 2.6%；甚至与 LLM-Summary 相比，每个实例的成本也低 0.03 美元（全基准 500 个实例合计 15 美元）。

---

## 2 相关工作

当前的软件工程智能体研究主要聚焦于提升智能体的**效果**：扩展训练数据 [10, 22, 36]、从多次尝试中挑选最有希望的方案 [22, 10, 40, 2, 1, 33]、通过评审器（critic）向智能体提供无执行或基于执行的反馈 [2, 22, 10, 40, 24]，或通过显式搜索策略增强智能体的规划能力 [2, 1]。这些方法虽然提升了解决率，但代价是推理成本增加、效率下降。相比之下，软件工程智能体的高效上下文管理迄今受到的关注很少。本文研究的问题是：高效的上下文管理是否必须依赖复杂的摘要策略。为此我们在 SWE-agent [35] 和 OpenHands [30] 上进行实验。

近来有多项工作深入分析了上下文长度对 LLM 性能的影响 [16, 7, 19]，一致表明：随着上下文增大，LLM 越来越难以有效利用所提供的上下文。尽管上下文管理对智能体性能和部署成本至关重要，现有工作大多将其视为实现细节而非一等研究问题。一个最近的例外是 MEM1 [41]，它探索了多跳问答（multi-hop QA）[37, 13, 12] 和网页导航 [38] 任务中的动态状态管理。但作者并未与"省略式"方法做比较；而且其所用基准产生的轨迹相对较短（数百 token 量级）[41]，而软件工程智能体的轨迹通常要大几个数量级 [15]。

与我们同期的工作中，Xiao 等 [34] 提出了一种用于软件工程智能体高效上下文管理的 LLM-Summary 变体，但没有与观察屏蔽基线比较。与本文最相关的是，他们的 "Delete" 基线是一种删除完整轮次而非摘要的 LLM-Summary 变体。该基线没有利用软件工程轨迹向环境观察倾斜的特性（图 1），并且打断了智能体跨轮次的推理链，导致下游性能退化。尽管如此，他们的结果显示该基线在相当的下游性能下比 LLM-Summary 更高效。此外，Lu 等 [18] 研究了在软件工程和计算机操作智能体（Computer-Use Agent, CUA）的 RL 训练中使用 LLM-Summary 来应对上下文窗口限制；而 Tang 等 [26] 则在深度研究（deep research）与 CUA 智能体的 RL 训练和推理中用观察屏蔽取得了亮眼成绩。这凸显了本研究问题的时效性，也表明我们的发现很可能推广到深度研究和 CUA 智能体。

---

## 3 实验配置

> **图 3 | 本文评估的上下文管理策略概览。** 方框高度表示典型轨迹中该部分所占的 token 数。(a) 基线的 ReAct 风格 [39] 轨迹：上下文随每个"动作—观察"对不断增长；(b) LLM 摘要将较早的轮次压缩为一份持续更新的摘要，并完整保留最近几轮（如 OpenHands [30]）；(c) 观察屏蔽将早于固定窗口 M 轮（图中 M=1）的观察替换为占位符（如 SWE-agent [35]）。

在实验中，我们通过有代表性的开源实现来研究观察省略与 LLM 摘要这两类流行方法：(1) 基于滚动窗口的环境观察屏蔽 [35, 2]；(2) 基于提示词的 LLM-Summary [30, 9]。下面我们详细定义所选方法。

### 3.1 上下文管理策略

**原始智能体（Raw Agent）。** 我们研究的智能体框架采用 ReAct [39] 或 CodeAct [29] 作为智能体框架。在这些框架中，智能体的历史（即轨迹）是与环境交互的序列。我们将第 t−1 轮结束时的轨迹记为 $\tau_{t-1}$，定义为 token 序列：

$$\tau_{t-1}=(o_{sys},o_{user},(r_{1},a_{1},o_{1}),\dots,(r_{t-1},a_{t-1},o_{t-1})) \tag{1}$$

其中 $o_{sys}$ 和 $o_{user}$ 是初始化任务的不可变系统提示词和用户提示词（图 3a）。我们定义轮次 $T_i=(r_i,a_i,o_i)$，由推理 $r_i$、动作 $a_i$ 和观察 $o_i$ 组成，是第 i 轮智能体—环境交互的原子单元。这样，$\tau_{t-1}$ 可以紧凑地表示为：

$$\tau_{t-1}=(o_{sys},o_{user},T_{1},\dots,T_{t-1}) \tag{2}$$

在第 t 轮开始时，智能体策略 $\pi$（通常是 LLM）以历史 $\tau_{t-1}$ 为条件生成下一对推理与动作：$(r_t,a_t)\sim\pi(\cdot|\tau_{t-1})$。在没有任何上下文管理策略时，$\tau$ 随每一轮增长，导致过高的计算成本并最终触及上下文长度限制。

**观察屏蔽（Observation Masking）。** 该策略通过选择性地压缩过去的环境观察来管理上下文规模，同时完整保留智能体推理和动作的历史。我们定义观察屏蔽函数 $f_{mask}(\tau_{t-1},M)$，输入完整轨迹 $\tau_{t-1}$ 和整数窗口大小 M，通过将早于窗口的观察替换为占位符（图 3b）产出压缩后的轨迹 $\tau'_{t-1}$。

形式化地，给定轨迹 $\tau_{t-1}=(o_{sys},o_{user},T_1,\dots,T_{t-1})$，变换后的轨迹为：

$$\tau^{\prime}_{t-1}=(o_{sys},o_{user},(r_{1},a_{1},o^{\prime}_{1}),\dots,(r_{t-1},a_{t-1},o^{\prime}_{t-1})) \tag{3}$$

其中每一轮 i 的观察 $o'_i$ 条件定义如下：

$$o^{\prime}_{i}=\begin{cases}p_{i}&\text{若 } i<t-M\\ o_{i}&\text{若 } i\geq t-M\end{cases} \tag{4}$$

其中 $p_i$ 是代表被屏蔽观察的占位文本，例如"前 8 行因篇幅省略。"（Previous 8 lines omitted for brevity.）。在后续轮次中，智能体 LLM 以该压缩历史 $\tau'_{t-1}$ 为条件生成 $(r_t,a_t)$。该方法遵循 SWE-agent [35]，在降低远处观察保真度的同时保留完整的推理链。注意：该策略只是减缓了 $\tau$ 中 token 的增长速度，并没有解决无限增长的问题。

**LLM-Summary。** 该策略使用一个"摘要 LLM"（记为 $\pi'$）来压缩轨迹。与 $f_{mask}$ 不同，其目标是保留被处理轮次中的显著信息。它由两个参数 N 和 M 控制：N 决定智能体一次积累多少轮再触发摘要，M 决定最后多少轮保持原样不动。当距上一次摘要以来的累计轮数达到 N+M 时触发摘要。

为定义该方法，引入两个变量 $t_{last}$ 和 $s_{last}$：$t_{last}$ 是最近一次摘要所覆盖的最后一轮的索引（第 0 步时 $t_{last}=0$）；$s_{last}$ 是在 $t_{last}$ 处生成的摘要（第 0 步时 $s_{last}=o_{user}$）。摘要过程如下：首先切分轨迹得到待摘要轮次，包含上一次摘要 $s_{last}$ 及其与最后 M 轮之前的所有轮次，即 $\mathcal{T}_{sum}=(s_{last},T_{t_{last}+1},\dots,T_{t-1-M})$；然后用摘要指令 $o_{si}$ 和轨迹切片 $\mathcal{T}_{sum}$ 提示 $\pi'$ 生成新摘要 $s_t$：

$$s_{t}\sim\pi^{\prime}(\cdot|o_{si},\mathcal{T}_{sum}),\quad t_{last}=t-1-M \tag{5}$$

最后，将提供给主策略 $\pi$ 的历史重构为新的压缩轨迹 $\tau'_{t-1}$：

$$\tau^{\prime}_{t-1}=(o_{sys},o_{user},s_{t},T_{t-M},\dots,T_{t-1}) \tag{6}$$

下一轮，智能体 LLM 以这一新的紧凑历史为条件：$(r_t,a_t)\sim\pi(\cdot|\tau'_{t-1})$。该方法不仅减缓了轨迹增长，还将其**约束住**（bounded）——较早的交互被递归地折叠进一份不断演化的摘要中。

### 3.2 实验配置

我们开展了严谨的对比研究：以 SWE-agent [35] 为主，用 OpenHands [30]（v0.43.0）检验普适性。实验覆盖多样的配置：(1) 模型家族、(2) 模型规模、(3) 模型许可（开放权重 vs. 专有）、(4) 推理模式（thinking 与非 thinking）。具体而言：Qwen3-32B [27]（thinking 与非 thinking 模式，使用 YaRN [23] 扩展至 122K token 上下文窗口）、Qwen3-Coder-480B-A35B-Instruct-FP8 [27, 8]（默认 256K 上下文窗口）、Gemini 2.5 Flash [6]（默认 1M 上下文窗口，thinking 与非 thinking 模式）。除非另有说明，所有实验均在 SWE-bench Verified [5] 上进行。所有实验运行在一个共享集群上：8 块 NVIDIA H200 GPU（每块 141 GB HBM），共 8 TB 本地磁盘。对 Qwen3-32B 模型，我们使用 2 块 H200 和 15 个 SWE-agent 推理 worker；考虑到可能出现上下文超过 100K token 的长轨迹，必须为 KV-cache 预留空间，因此 worker 数取保守值。对 Qwen3-Coder 480B，使用全部 8 块 GPU 和 35 个推理 worker。Qwen 模型用 vLLM [14] 部署。Gemini 实验中，由于配额限制，SWE-agent 用 8 个推理 worker、OpenHands 用 5 个。更多细节见附录 A。

我们的初步实验（附录 D.4）表明，轨迹中的轮数会影响所研究上下文管理策略的行为。因此我们实验采用长轨迹，除非另有说明，将轮数上限设为 250。观察屏蔽策略在主实验中使用 M=10 的滚动窗口，因为在我们的实验中它使 SWE-agent [35] 取得最佳性能（附录 D.1）。我们在 SWE-agent [35] 中实现的 LLM-Summary 使用了 OpenHands 风格提示词的轻微修改版（附录 E）。与 OpenHands 的基线配置 [30] 不同，我们一次摘要 21 轮（N=21）并仅保留最后 10 轮（M=10）。除了让观察屏蔽与 LLM-Summary 的"尾部保留轮数"对齐之外，我们还发现 M=10 在我们的实验中同样使 LLM-Summary 性能最佳（附录 D.2）。智能体模型使用温度 0.8，摘要模型使用温度 0。与 Qwen3-32B (thinking) 实验不同，受成本限制，Gemini 2.5 Flash 的 thinking 预算被限制为 0 或 800 token（记作 thinking）。

---

## 4 主要结果

我们在 SWE-agent [35] 内的主实验评估了三种上下文管理策略，结果汇总于表 1。结果揭示了两个在所有测试条件下都稳健成立的核心发现：**第一**，观察屏蔽和 LLM-Summary 都显著降低成本，且不显著牺牲解决率；**第二**，LLM-Summary 在效率或效果上都未能持续或显著地超越观察屏蔽。更多细节见附录 C。

### 4.1 上下文管理的普遍收益

我们的第一个也是最基础的发现（也是本研究的动机所在）是：上下文管理不仅是优化，更是必需。如表 1 所示，放任智能体上下文无节制增长（原始智能体基线）始终是最昂贵的策略。在所有轨迹长到足以受益于高效上下文管理的实验配置中，观察屏蔽和 LLM-Summary 都显著降低了每实例成本，多数情况下降幅超过 50%。Qwen3-32B (thinking) 的异常行为在附录 B 中讨论。

此外，这种效率提升并不必然以性能为代价：在五个设置中的三个里，最高效的策略同时取得了比原始智能体更高的解决率。这表明超过某个程度后，更多的上下文会从资产变为负债，与"迷失在中间"问题 [16] 相吻合。因此，追问哪种方法在效果与效率之间提供最佳权衡，至关重要。

> **表 1 | 各上下文管理策略对比（含 95% bootstrap 置信区间）。** 相对原始智能体报告变化量与显著性（†）。解决率（效果，↑）与实例成本（效率，↓）。每个模型下，每个指标加粗最优策略。所有实验均为 SWE-agent [35] 于 SWE-bench Verified [5]。细节见附录 C。

| 模型 | 策略 | 解决率 (%, ↑) | 实例成本 ($, ↓) |
| --- | --- | --- | --- |
| Qwen3-32B | Raw Agent | 17.0 ±3.3 | 1.12 ±0.18 |
| | Observation Masking | 15.0 ±3.1 (-11.8%) | **0.55 ±0.09 (-50.9%)†** |
| | LLM-Summary | 16.0 ±3.3 (-5.9%) | **0.50 ±0.07 (-55.4%)†** |
| Qwen3-32B (thinking) | Raw Agent | 23.0 ±3.7 | 0.51 ±0.07 |
| | Observation Masking | 24.6 ±3.8 (+7.0%) | **0.46 ±0.05 (-9.8%)** |
| | LLM-Summary | **24.8 ±3.9 (+7.3%)** | 0.51 ±0.06 (0.0%) |
| Qwen3-Coder 480B | Raw Agent | 53.4 ±4.3 | 1.29 ±0.26 |
| | Observation Masking | **54.8 ±4.4 (+2.6%)** | **0.61 ±0.06 (-52.7%)†** |
| | LLM-Summary | 53.8 ±4.2 (+0.7%) | 0.64 ±0.06 (-50.4%)† |
| Gemini 2.5 Flash | Raw Agent | 32.8 ±4.1 | 0.41 ±0.08 |
| | Observation Masking | 35.6 ±4.2 (+8.5%) | **0.18 ±0.03 (-56.1%)†** |
| | LLM-Summary | **36.0 ±4.1 (+9.8%)** | 0.24 ±0.04 (-41.5%)† |
| Gemini 2.5 Flash (thinking) | Raw Agent | **40.4 ±4.3** | 0.56 ±0.10 |
| | Observation Masking | 36.4 ±4.2 (-9.9%)† | **0.24 ±0.04 (-57.1%)†** |
| | LLM-Summary | 31.4 ±4.0 (-22.3%)† | 0.25 ±0.05 (-55.4%)† |

### 4.2 观察屏蔽：以最小复杂度实现支配性效率

在确立了上下文管理的必要性之后，我们转向第二个核心发现：**简单性的惊人力量**。如表 1 所示，在五个实验设置中的四个里，观察屏蔽取得了最低的每实例成本。其原理是大幅削减每个智能体轮次中需要处理的环境观察 token 数量，且无需承担单独一次摘要调用的计算开销。鉴于软件工程中智能体上下文严重偏向环境观察（图 1），这种做法非常有效。此外，该策略比 LLM-Summary 需要的预热轮数更少（M=10 vs. N+M=31），因此即使在短轨迹上（如 Qwen3-32B (thinking)，见附录 B）也能更快、更稳健地实现成本降低。

值得注意的是，这一发现跨模型配置均成立。而且，Qwen3-Coder 480B 上 LLM-Summary 与观察屏蔽之间 0.03 美元的单实例成本差看似很小，但整个基准累计已达 15 美元。这说明即使是微小的成本效率改进，也能对大规模 LLM 智能体部署的经济可行性产生显著影响，也凸显了高效上下文管理研究的必要性。

### 4.3 挑战"复杂摘要是必需的"这一假设

观察屏蔽不仅始终是最便宜的选择，在维持高解决率方面也表现出色。这直接挑战了一种假设：必须用复杂的、语义级的摘要才能保留智能体轨迹中的关键信息。

事实上，观察屏蔽不仅能与 LLM-Summary 抗衡，甚至能超越它。在 Qwen3-Coder 480B 上，观察屏蔽达到 54.8% 的解决率，略高于 LLM-Summary 的 53.8%；在 Gemini 2.5 Flash (thinking) 上，观察屏蔽比 LLM-Summary 高出五个百分点。而在 LLM-Summary 表现更好的情形（如 Gemini 2.5 Flash），差距也极小（36.0% vs. 35.6%）。这表明观察屏蔽始终与 LLM-Summary 持平或更优。

结论很明确：**对软件工程智能体而言，最近的上下文往往就足够了**。保留完整历史、甚至其精巧摘要，可能都不是对模型有限上下文窗口和我们研究预算的最优使用。

### 4.4 轨迹拉长效应（Trajectory Elongation Effect）

由表 1 主结果引出的一个关键问题是：为什么除 Qwen3-32B 外，LLM-Summary 在所有实验中的成本效益都低于观察屏蔽？我们的分析揭示，这部分源于 LLM-Summary 上下文管理策略一个意想不到的副作用——"轨迹拉长"。

我们分析了图 4 中每实例轮数的分布：LLM-Summary 导致 Qwen3-Coder 480B 和 Gemini 2.5 Flash 的平均轨迹长度都变长。对 Gemini 2.5 Flash，LLM-Summary 的平均轨迹长度为 52 轮，比观察屏蔽（44 轮）高 15%，比原始智能体（50 轮）高 4%。同样，对 Qwen3-Coder 480B，LLM-Summary 的平均轨迹长度比原始智能体高 15%、比观察屏蔽高 13%。

值得注意的是，这种轮数拉长趋势直接转化为图 2 和表 1 中观察到的策略效率差异。唯一一个 LLM-Summary 更具成本效益的实验是 Qwen3-32B：在这个实验中，是观察屏蔽（而非 LLM-Summary）相对原始智能体使平均轨迹长度增加了 13%。这表明上下文摘要起到了**强化信号**的作用，鼓励智能体继续推进，进而造成轨迹拉长，抵消了 LLM-Summary 通过有界上下文（第 3.1 节）获得的效率收益。

> **图 4 | 上下文管理策略对轨迹长度的影响。** 箱线图展示 SWE-agent 内各策略下轨迹长度（以轮数计）的分布。LLM-Summary 持续导致更长的轨迹，暗示摘要掩盖了本会促使智能体更早终止的失败信号。星号表示平均轨迹长度。

---

## 5 讨论

本节我们在 OpenHands [30] 上检验发现的普适性，讨论摘要生成本身对 LLM-Summary 成本结构的影响，并介绍一种结合两种策略优点的新颖混合方法。

### 5.1 用 OpenHands 探究普适性

为了检验主要结果在不同框架间的普适性，我们在 SWE-bench Verified [5, 3] 的 50 实例子集上用 OpenHands + Gemini 2.5 Flash [6]（非 thinking）做了探测实验，轮数上限 250，LLM-Summary 取 N=21、M=10，观察屏蔽分别取 M=10 和 M=58。结果见图 5a。

首先我们注意到，观察屏蔽的滚动窗口大小 M 是一个**依赖具体智能体框架的超参数**，需要调优。如果直接沿用 SWE-agent [35] 的最优值，观察屏蔽的性能会急剧退化；但经过调优后，我们在这个框架上复现了主要结果。我们假设需要调优的原因是框架特有的实现细节：例如 SWE-agent [35] 会直接从对话历史中略去因语法错误而重试的轮次，而 OpenHands [30] 会保留这些轮次，因此 OpenHands 需要更大的窗口才能保留有信息量的上下文。

总体而言，这提供了初步证据：只要智能体的上下文同样偏向环境观察（软件工程智能体通常如此），我们的发现就能跨智能体框架推广。

### 5.2 摘要的成本

对成本分解的进一步检视表明，策略间的效率差距源于两个互补的效应：一是第 4.4 节讨论的"轨迹拉长"效应，二是生成摘要本身的成本。

如表 2 所示，生成摘要的直接 API 成本最高占总实例成本的 7.2%。重要的是，这些摘要调用格外昂贵，因为每次都需处理一段独一无二的轮次序列，缓存复用仅限于 LLM-Summary 的系统提示词（图 11）。考虑到许多现代 LLM API（如 Gemini）的缓存命中比未命中便宜得多（最多便宜 10 倍），这是一个关键的限制因素。从总成本中扣除摘要成本后，大多数实验中 LLM-Summary 与观察屏蔽的效率差距基本消失。这表明摘要 API 调用本身构成了效率差距的相当一部分。尽管如此，更复杂的 LLM-Summary 策略仍然未能在成本效益上持续或显著超越极其强大的观察屏蔽策略。这暗示通过未被充分探索的观察屏蔽或混合方法，仍有潜在的成本节省空间。

> **表 2 | 每个模型的平均实例 LLM-Summary 成本。** 摘要生成的 API 成本解释了 LLM-Summary 与观察屏蔽之间成本效率差距的一部分。

| 模型 | 实例 LLM-Summary 成本 ($) | 占比 (%) |
| --- | --- | --- |
| Qwen3-32B | 0.0143 | 2.86 |
| Qwen3-32B (thinking) | 0.0033 | 0.65 |
| Qwen3-Coder 480B | 0.0439 | 7.20 |
| Gemini 2.5 Flash | 0.0161 | 6.71 |
| Gemini 2.5 Flash (thinking) | 0.0131 | 5.24 |

### 5.3 混合方法：观察屏蔽 + LLM-Summary

鉴于所覆盖的上下文管理策略各自表现强劲——都能显著且持续地降低 >50% 的成本——本节我们提出一种新颖的混合方法。出于成本考虑，我们在解决率最强的模型 Qwen3-Coder 480B + SWE-agent [35] 上、在 SWE-bench Verified-50 [3] 上实验。结果见图 5b。

轨迹拉长效应（第 4.4 节）和摘要生成开销（第 5.2 节）都促使我们将 LLM-Summary 视为**最后一道防线**，尽可能推迟触发。但如果增大 N，就会增加观察到任何效果之前所需的预热轮数，且在积累阶段我们将处于昂贵的原始智能体模式；此外还有风险：在短到中等长度的轨迹上可能完全观察不到效果（附录 B）。在轮次积累阶段使用观察屏蔽，可以兼取两者之长：增大 N 推迟 LLM-Summary，把它当作约束长轨迹上下文的最后手段；与此同时，观察屏蔽在积累阶段迅速产生收益，且即使在短轨迹上也稳健有效。

我们设 N=43，因为在该轮数下，观察屏蔽模式下积累的上下文与原始智能体在 N=21 轮时积累的上下文大致相当（约 30K token，见图 9）。为避免记号冲突，混合设置中观察屏蔽的滚动窗口大小记为 W。总体上，混合设置使用 N=43、M=W=10。注意：每次用 LLM-Summary 摘要时，我们传入的是**未屏蔽**的上下文。

与观察屏蔽和 LLM-Summary 相比，该方法分别降低成本 7% 和 11%；而且，它还将下游任务性能提升 2.6 个百分点，推进了效果—效率前沿。在完整的 SWE-bench Verified [5] 基准上，这相当于相对观察屏蔽节省约 20 美元、相对 LLM-Summary 节省约 35 美元。

为消融超参数选择，我们还测试了无视策略特性、简单沿用单策略超参数（N=21, M=W=10）的朴素配置。在该配置下，由于 KV-cache 效率问题叠加 LLM-Summary 调用的开销，混合方法反而恶化了系统整体的成本效率。

> **图 5 | (a)** 在 SWE-bench Verified-50 [3] 子集上用 OpenHands [30] 探究普适性：在针对框架恰当调优滚动窗口 M 之后，观察屏蔽再次在成本与解决率上与 LLM-Summary 持平。**(b)** 我们新颖的"观察屏蔽 + LLM-Summary"混合方法在 SWE-agent 框架 + Qwen3-Coder 480B + SWE-bench Verified-50 [3] 上的表现：有效结合两种方法的优点，无论轨迹长短都能稳健实现效率收益，同时在超长轨迹上受益于有界上下文。混合方法相对原始智能体提升解决率 2.6 个百分点，同时相对观察屏蔽和 LLM-Summary 分别降低成本 7% 和 11%。

---

## 6 局限性

虽然本研究对上下文管理策略进行了严谨评估，但其范围有三个主要局限。**第一**，我们只在软件工程领域、使用 SWE-bench [11] 基准做实验。该领域以冗长繁杂的工具输出为特征，天然有利于观察屏蔽的效率发挥。因此，关于观察屏蔽优越性的结论可能无法推广到智能体—环境交互更简洁的领域。**第二**，所研究的全部策略都使用简单的、非自适应的启发式触发器：观察屏蔽采用固定大小的滚动窗口，对过去观察的相关性或陈旧性不敏感（例如文件内容在被修改后仍被保留）；LLM-Summary 则按固定的轮次计划运行，忽略语义边界或智能体子目标。**第三**，虽然我们提供了跨智能体框架推广的初步证据，但更全面的验证仍有必要。

---

## 7 结论

本工作对上下文管理策略进行了跨多样模型配置与智能体框架的全面研究。我们发现：高效的上下文管理策略能持续且显著地降低系统成本超过 50%，而不显著降低下游性能。令人惊讶的是，流行的 LLM-Summary 策略无法持续或显著地超越简单的观察屏蔽基线。这暗示只关注 LLM-Summary 的现代智能体系统存在尚未挖掘的节省空间。我们用新颖的"观察屏蔽 + LLM-Summary"混合策略实证验证了这一假设：相比单独使用观察屏蔽或 LLM-Summary，混合策略进一步分别降低成本 7% 和 11%，同时提升下游任务性能。这些发现确立了上下文管理对实现经济可行、环境可持续的 LLM 智能体部署的关键必要性；同时表明：在追求高效 LLM 智能体的过程中，**简单的解决方案可能出奇地有效**。

---

## 致谢与资助披露

我们感谢 Calvin Smith 就 OpenHands 中实现的上下文管理策略进行的高产且富有启发性的讨论；感谢 Kirill Gelvan 对本文早期版本提供的反馈。

---

## 参考文献

（保留原引用，共 41 条；完整条目与原文一致，此处按原编号保留关键信息）

[1] Aggarwal et al. *DARS: Dynamic action re-sampling to enhance coding agent performance by adaptive tree traversal.* ACL 2025.
[2] Antoniades et al. *SWE-Search: Enhancing software agents with Monte Carlo tree search and iterative refinement.* ICLR 2025.
[3] Badertdinov et al. *Scaling data collection for training software engineering agents.* Nebius blog, 2024.（SWE-bench Lite-50 来源）
[4] Brown et al. *Language Models are Few-Shot Learners.* NeurIPS 2020.
[5] Chowdhury et al. *Introducing SWE-bench Verified.* OpenAI, 2024.
[6] Comanici et al. (Google DeepMind). *Gemini 2.5 Technical Report*, 2025.
[7] Hong, Troynikov, Huber. *Context Rot: How increasing input tokens impacts LLM performance.* Chroma, 2025.
[8] Hui et al. *Qwen2.5-Coder Technical Report.* arXiv:2409.12186, 2024.
[9] Anysphere Inc. *Summarization: Context management for long conversations in chat.* Cursor docs, 2025.
[10] Jain et al. *R2E-Gym: Procedural environment generation and hybrid verifiers for scaling open-weights SWE agents.* COLM 2025.
[11] Jimenez et al. *SWE-bench: Can language models resolve real-world GitHub issues?* ICLR 2024.
[12] Jin et al. *Search-R1: Training LLMs to reason and leverage search engines with reinforcement learning.* arXiv:2503.09516, 2025.
[13] Kwiatkowski et al. *Natural Questions: a benchmark for question answering research.* TACL 2019.
[14] Kwon et al. *Efficient memory management for large language model serving with PagedAttention.* SOSP 2023.
[15] Lindenbauer, Groh, Schuetze. *From knowledge to noise: CTIM-rover and the pitfalls of episodic memory in software engineering agents.* REALM 2025.
[16] Liu et al. *Lost in the middle: How language models use long contexts.* TACL 2024.
[17] Liu et al. *Can 1B LLM surpass 405B LLM? Rethinking compute-optimal test-time scaling.* 2025.
[18] Lu et al. *Scaling LLM multi-turn RL with end-to-end summarization-based context management.* arXiv:2510.06727.
[19] Modarressi et al. *NoLiMa: Long-context evaluation beyond literal matching.* ICML 2025.
[20] OpenAI et al. *Introducing GPT-4.1 in the API.* 2025.
[21] OpenAI et al. *GPT-4 Technical Report.* arXiv:2303.08774, 2024.
[22] Pan et al. *Training Software Engineering Agents and Verifiers with SWE-Gym.* arXiv:2412.21139, 2024.
[23] Peng et al. *YaRN: Efficient context window extension of large language models.* 2023.
[24] Shinn et al. *Reflexion: Language agents with verbal reinforcement learning.* NeurIPS 2023.
[25] Snell et al. *Scaling LLM test-time compute optimally can be more effective than scaling parameters for reasoning.* ICLR 2025.
[26] Tang et al. *Beyond turn limits: Training deep search agents with dynamic context window.* arXiv:2510.08276.
[27] Qwen Team. *Qwen3 Technical Report.* arXiv:2505.09388, 2025.
[28] Vaswani et al. *Attention is all you need.* NeurIPS 2017.
[29] Wang et al. *Executable Code Actions Elicit Better LLM Agents.* ICML 2024.
[30] Wang et al. *OpenHands: An open platform for AI software developers as generalist agents.* ICLR 2025.
[31] Wei et al. *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models.* NeurIPS 2022.
[32] Wu et al. *Scaling inference computation: Compute-optimal inference for problem-solving with language models.* NeurIPS 2024 MathAI Workshop.
[33] Xia, Deng, Dunn, Zhang. *Demystifying LLM-based software engineering agents.* Proc. ACM Softw. Eng. (FSE), 2025.
[34] Xiao, Gao, Peng, Xiong. *Improving the efficiency of LLM agent systems through trajectory reduction.* arXiv:2509.23586.
[35] Yang et al. *SWE-agent: Agent-computer interfaces enable automated software engineering.* NeurIPS 2024.
[36] Yang et al. *SWE-smith: Scaling data for software engineering agents.* arXiv:2504.21798, 2025.
[37] Yang et al. *HotpotQA: A dataset for diverse, explainable multi-hop question answering.* EMNLP 2018.
[38] Yao et al. *WebShop: Towards scalable real-world web interaction with grounded language agents.* NeurIPS 2022.
[39] Yao et al. *ReAct: Synergizing reasoning and acting in language models.* ICLR 2023.
[40] Zhang et al. *Diversity empowers intelligence: Integrating expertise of software engineering agents.* ICLR 2025.
[41] Zhou et al. *MEM1: Learning to synergize memory and reasoning for efficient long-horizon agents.* 2025.

---

## 附录 A 实验配置

计算实例成本时，我们区分通过 API 访问的专有模型和自建基础设施上本地部署的模型。对通过 Vertex AI API 访问的 Gemini [6] 实验，我们报告 API 返回的成本。对自托管的 Qwen [27, 8] 实验（如第 3.2 节所述），我们根据观测到的每轮 token 用量，并按阿里云官方 API 定价事后计算成本。注意：与 Vertex AI 平台不同，阿里云官方 API 定价对 Qwen3-32B 不区分缓存命中与未命中的输入 token，这导致 Qwen3-32B 实验的成本被高估。对在不同定价方案下实验成本结构感兴趣的读者，请参阅我们的 HuggingFace 数据集（发布全部主实验数据）与代码仓库。

---

## 附录 B Qwen3-32B (thinking) 的短轨迹：假设与推论

Qwen3-32B (thinking) 配置的中位轨迹长度比 Qwen3-32B 配置短约 50%。考虑到两种配置唯一的差别就是开启了模型的 thinking 模式，这令人意外。以下详述我们对该异常结果的调查及其对结论解读的影响。

**配置验证。** 我们验证了两种实验配置在所有上下文管理策略下的配置与数据集均正确。除 thinking 模式下模型性能略有提升外，两种配置的实例退出状态分布未见偏差。

**定性分析。** 由于未发现配置错误，我们进一步对 Qwen3-32B 各配置下相同的 20 条（4%）轨迹做了定性分析。我们注意到两个模型有时都难以遵循智能体框架的函数调用格式，但如上所述，两种配置中因函数调用错误而退出的数量均未出现可疑偏差。

**推论。** 既然无法识别任何配置错误或其他可疑行为作为该异常结果的原因，我们必须假设结果有效。据此讨论其对整体结论解读的影响：第一，该配置产生了短轨迹；而两种上下文管理策略都需要若干预热轮数才开始修改轨迹并降低成本——对 LLM-Summary，按我们的超参数需要 N+M=31 轮才产生第一次摘要，中位长度仅 15 轮的轨迹远远不足以从该策略获得效率收益；观察屏蔽则在 M=10 轮后即开始屏蔽观察，因此在较短轨迹上也能看到效果，只是可能被削弱。这与经验观察完全吻合。因此，我们将 Qwen3-32B (thinking) 配置中不显著的成本节省归因于较短的轨迹长度，而非上下文管理策略本身的根本问题。以下事实进一步支持这一解释：即使在这种不利环境下，观察屏蔽仍降低约 10% 的成本，且两种策略的下游性能依然稳定。

---

## 附录 C 主结果详细数据

本节提供表 1 中置信区间与显著性指标所依据的进一步数据。表 3 给出主结果背后的完整非对称置信区间；表 1 中的对称区间由非对称边界取平均得到，例如 $17.0_{-3.2}^{+3.4}$ 得到 17.0±3.3。

> **表 3 | 各上下文管理策略对比（95% bootstrap 置信区间，展示非对称性）。** † 表示相对原始智能体显著。解决率（效果，↑）与实例成本（效率，↓）。

| 模型 | 策略 | 解决率 (%, ↑) | 实例成本 ($, ↓) |
| --- | --- | --- | --- |
| Qwen3-32B | Raw Agent | 17.0 (+3.4/−3.2) | 1.12 (+0.18/−0.17) |
| | Observation Masking | 15.0 (+3.2/−3.0) (-11.8%) | 0.55 (+0.09/−0.08) (-50.9%)† |
| | LLM-Summary | 16.0 (+3.4/−3.2) (-5.9%) | 0.50 (+0.07/−0.06) (-55.4%)† |
| Qwen3-32B (thinking) | Raw Agent | 23.0 (+3.8/−3.6) | 0.51 (+0.07/−0.06) |
| | Observation Masking | 24.6 (+3.8/−3.8) (+7.0%) | 0.46 (+0.05/−0.05) (-9.8%) |
| | LLM-Summary | 24.8 (+4.0/−3.8) (+7.3%) | 0.51 (+0.06/−0.06) (0.0%) |
| Qwen3-Coder 480B | Raw Agent | 53.4 (+4.4/−4.2) | 1.29 (+0.28/−0.24) |
| | Observation Masking | 54.8 (+4.4/−4.4) (+2.6%) | 0.61 (+0.06/−0.05) (-52.7%)† |
| | LLM-Summary | 53.8 (+4.2/−4.2) (+0.7%) | 0.64 (+0.06/−0.05) (-50.4%)† |
| Gemini 2.5 Flash | Raw Agent | 32.8 (+4.2/−4.0) | 0.41 (+0.08/−0.07) |
| | Observation Masking | 35.6 (+4.2/−4.2) (+8.5%) | 0.18 (+0.03/−0.02) (-56.1%)† |
| | LLM-Summary | 36.0 (+4.2/−4.0) (+9.8%) | 0.24 (+0.04/−0.04) (-41.5%)† |
| Gemini 2.5 Flash (thinking) | Raw Agent | 40.4 (+4.2/−4.4) | 0.56 (+0.10/−0.10) |
| | Observation Masking | 36.4 (+4.2/−4.2) (-9.9%)† | 0.24 (+0.04/−0.03) (-57.1%)† |
| | LLM-Summary | 31.4 (+4.0/−4.0) (-22.3%)† | 0.25 (+0.05/−0.04) (-55.4%)† |

### C.1 统计分析

我们使用配对非参数 bootstrap（B=10,000 次重采样）评估显著性，详细结果见表 4。对每个"模型—策略"对，在同样的 n=500 个实例上计算配对差值 Δ = mean(strategy) − mean(raw)，保留实例级相关性。我们报告：95% 分位置信区间；双侧 p 值 $p=2\times\min(\Pr(\Delta^{*}\geq 0),\Pr(\Delta^{*}\leq 0))$；以及 p<0.05 时的显著性标记（†）。p 值为 0.0000 表示所有 bootstrap 重采样中差值符号均未跨越零（分辨率 ≤10⁻⁴）。

> **表 4 | 相对原始智能体的配对 bootstrap 差值（95% 分位置信区间与双侧 p 值，B=10,000）。** Δ解决率以百分点（pp）计，Δ平均成本以美元/实例计；负成本差值表示节省成本。所有行使用每个模型 n=500 的共同实例。† 表示相对原始智能体显著。

| 模型 | 策略 | Δ解决率 (pp) [lo, hi] | p | Δ平均成本 ($) [lo, hi] | p |
| --- | --- | --- | --- | --- | --- |
| Gemini 2.5 Flash | Observation Masking | 2.8 [-0.8, 6.4] | 0.1504 | -0.2377 [-0.3202, -0.1614] | 0.0000† |
| | LLM-Summary | 3.2 [-0.4, 7.0] | 0.0948 | -0.1725 [-0.2579, -0.0936] | 0.0000† |
| Gemini 2.5 Flash (thinking) | Observation Masking | -4.0 [-7.8, -0.2] | 0.0406† | -0.3143 [-0.4096, -0.2245] | 0.0000† |
| | LLM-Summary | -9.0 [-13.0, -5.2] | 0.0000† | -0.3046 [-0.4074, -0.2043] | 0.0000† |
| Qwen3-Coder 480B | Observation Masking | 1.4 [-1.6, 4.4] | 0.3856 | -0.6762 [-0.9320, -0.4518] | 0.0000† |
| | LLM-Summary | 0.4 [-3.0, 3.8] | 0.8736 | -0.6491 [-0.9048, -0.4263] | 0.0000† |
| Qwen3-32B | Observation Masking | -2.0 [-5.0, 1.0] | 0.2086 | -0.5632 [-0.7479, -0.3817] | 0.0000† |
| | LLM-Summary | -1.0 [-4.6, 2.6] | 0.6192 | -0.6174 [-0.7904, -0.4454] | 0.0000† |
| Qwen3-32B (thinking) | Observation Masking | 1.6 [-2.0, 5.2] | 0.3980 | -0.0510 [-0.1255, 0.0187] | 0.1586 |
| | LLM-Summary | 1.8 [-1.8, 5.4] | 0.3420 | -0.0021 [-0.0785, 0.0741] | 0.9370 |

---

## 附录 D 补充研究

图 7 的评审器增强摘要器实验在 SWE-bench Lite-50 [3] 上进行。观察屏蔽滚动窗口大小 M 的敏感性实验和 LLM-Summary 配置实验，在我们随代码发布的、从 SWE-bench Verified [5] 随机抽取的 150 实例子集上进行。这些研究均使用 GPT-4.1-mini [21]。

> **图 6 | 尾部长度 M 各配置组合下的下游任务性能。** 在随机生成的 SWE-bench Verified 150 样本子集上的单次实验。发现：相对尾部长度，更大的摘要窗口能提升性能。

### D.1 观察屏蔽配置

我们实验了观察屏蔽策略的滚动窗口大小 M。从图 10 可见，该策略的性能在 M=10 处达到峰值，继续增大到 M=20 时回落。因此主实验采用 M=10。

> **图 10 | 不同上下文窗口大小下的下游任务性能。** M=10 为最优配置。

### D.2 LLM-Summary 配置

图 6 展示了 LLM-Summary 不同实验配置（以及基线）的解决率。我们发现：使用尾部长度 M>0 能提升下游性能；此外，与 OpenHands [30] 的"摘要轮数与尾部轮数 50-50 均分"不同，一次摘要更多轮次能提升解决率。因此主实验取 N=21、M=10。

### D.3 评审器增强的 LLM-Summary

一个自然的追问是：让摘要过程更"聪明"能否改进 LLM-Summary？我们通过为 LLM-Summary 增加无执行反馈（execution-free feedback）来探索——该技术在扩展软件工程智能体的测试时计算方面已显示出潜力 [22, 10, 40, 2]。具体做法：让 LLM 在生成摘要的同时生成对轨迹的批判性分析，并将两者一并纳入压缩后的上下文。这相当于在单次 rollout 内部（而非跨多次 rollout [24]）提供反思（reflection）。

与附录 E 中改进版 OpenHands 提示词相比，我们把任务框定为生成"检查点"（checkpoint）而非摘要，并提示模型对待摘要的轮次进行反思。这样做旨在鼓励模型生成有助于智能体在一次尝试中调整求解路径的输出，避免其通过平铺直叙的摘要被锚定在之前可能次优甚至有缺陷的轮次上。

为引出有意义的反思，我们用引导性问题提示 LLM，例如：智能体是否卡住或陷入循环？是否仍与最初问题陈述对齐？并结合待摘要轮次反思智能体的高层求解思路。我们还提供少样本示例，进一步引导智能体生成有意义、可执行的反思 [4]。最后，与 OpenHands 风格提示词一样，我们把上一次摘要（如无则用问题陈述）和待摘要轮次一并提供给模型。

在 SWE-bench Verified 的 150 个样本上用 SWE-agent 测试，这种使用图 12–14 提示词的评审器增强方法，相比标准 LLM-Summary 没有任何解决率提升。更令人担忧的是，我们观察到加剧的轨迹拉长模式：评审器增强的运行产生了比标准摘要更长的轨迹。这也许并不意外：评审器的反思天然鼓励智能体探索替代求解路径、尝试更多调试策略或重新考虑方案——这一切都转化为更多轮次，从而推高成本、削弱效率收益。

> **图 7 | 标准 LLM-Summary 提示词 vs. 联合评审—摘要提示词的下游任务性能对比。** 仅仅提示模型在摘要中加入反馈并不能提升解决率，还会进一步增加成本。

这一发现强化了我们关于轨迹拉长的核心洞见：无执行反馈虽然旨在改善智能体决策，却通过延长探索而悖论式地增加了计算成本。评审器的引导非但没有帮助智能体高效识别死胡同，反而提供了更多可以尝试的路径，进一步推迟终止。此外，增加的成本并未带来下游性能的提升。这表明，为 AI 智能体构建有效的记忆系统需要根本性的重新思考：向摘要中简单叠加更复杂的反馈，可能会加剧而非解决我们识别出的效率难题。

### D.4 各上下文管理策略跨轮次的行为

图 8 展示了在 SWE-bench Lite-50 [3] 上、使用 GPT-4.1-mini [20] 的初步实验结果：滚动窗口 M=5（遵循 SWE-agent [35]），LLM-Summary 取 N=21、M=10，并配以 OpenHands 提示词的轻微修改版（见附录 E）。我们观察到：观察屏蔽策略是一个强基线——尽管简单得多，其在下游任务上的表现持续与 LLM-Summary 持平或更优。

> **图 8 | SWE-bench Lite-50 上各上下文管理策略的下游任务性能。** 柱状图为三次实验的均值，误差线为标准差。令人惊讶的是，滚动窗口屏蔽 $f_{RW}$ 与花费更多算力的 LLM 策略表现相当。

为探究原因、并揭示 LLM-Summary 策略可能带来收益的潜在场景，我们在图 9 中分析了这些策略跨轮次的行为。实线为 SWE-agent [35] 的实证观测结果：每个策略运行三次实验，共得到 150 条轨迹，对每一轮取微平均。虚线为基于实证的模拟行为。生成模拟轨迹数据的方法是：在原始智能体的全部实验数据上计算每种 token 类型的平均 token 消耗：

$$\bar{x}=\frac{1}{T_{total}}\sum_{i=1}^{3}\sum_{j=1}^{50}\sum_{k=1}^{T_{local}}x_{ijk}\quad\text{where}\quad x\in\{r,a,o\} \tag{7}$$

其中 $T_{total}$ 是全部实例与实验观测到的总轮数，$T_{local}$ 是单条轨迹的轮数。然后用占位 token 生成一轮 $T_{sim}=(\bar{r},\bar{a},\bar{o})$，通过反复追加 $T_{sim}$ 构造任意长度的模拟轨迹 $\tau_{sim}=(T_{sim},\ldots,T_{sim})$；再对 $\tau_{sim}$ 应用 LLM-Summary 与观察屏蔽策略，得到模拟数据。这使我们能够研究这些策略在极大轮数下的预期行为。

图 9a、b、c 显示实验数据与模拟趋势的预期行为高度吻合。令人惊讶的是，滚动窗口屏蔽 $f_{RW}$ 在成本上与 LLM 策略具有竞争力，在上下文压缩上甚至更优。

$f_{RW}$ 是强基线的原因在于 token 在 r、a、o 三种类型间的分布：如图 1 所示，环境观察 token o 占 $T_{sim}$ 构成的约 84%，压倒性地主导，因此针对这一 token 类型进行处理极其有效。

图 9a、b 中可以看到这一效果：两种上下文管理策略的成本相近，但由于 $f_{RW}$ 的缓存行为较差，它在较低轮数下反而提供更优的压缩。而根据我们的模拟，在更长的轨迹上，预计基于 LLM 的方法将开始超越 $f_{RW}$——因为它以"模糊"的方式约束最大上下文规模，使成本和上下文窗口大小都呈锯齿函数。这正是我们把主实验的轮数上限设为 250 的动机。

> **图 9 | 支持本研究的初步实验结果。** 虚线为基于原始智能体轨迹中各轮次类型平均 token 数的预期行为；所有结果取微平均，标准差以阴影区域显示。(a)(b)(c)：观测数据与对模拟轨迹应用两种策略的预期效应高度吻合；由于上下文窗口有界，预计 LLM 方法在长轨迹上尤为有效。(d)：随轮数增加，实证数据越来越稀疏。

---

## 附录 E LLM 摘要提示词

我们在图 11 中给出摘要生成的提示词模板。与 OpenHands [30] 相比，我们移除了其中用于处理软件工程领域之外任务的部分，因为本文纯粹聚焦于软件工程领域。除图 11 的系统提示词外，我们还在图 12–14 中给出联合"评审器—摘要"提示词；其效果在第 D.3 节和第 5 节中讨论。

> **图 11 | 我们在 SWE-agent 中使用的 LLM-Summary 提示词**（OpenHands LLM-Summary 系统提示词 [30] 的轻微修改版）。此外还传入上一次摘要与待摘要轮次；若无上一次摘要，则传入任务问题陈述作为摘要生成的初始上下文。

```
You are maintaining a context-aware state summary for an interactive agent.
You will be given a list of events corresponding to actions taken by the
agent, and the most recent previous summary if one exists. Track:

USER_CONTEXT: (Preserve essential user requirements, goals, and
clarifications in concise form)
COMPLETED: (Tasks completed so far, with brief results)
PENDING: (Tasks that still need to be done)
CURRENT_STATE: (Current variables, data structures, or relevant state)

For code-specific tasks, also include:
CODE_STATE: (File paths, function signatures, data structures)
TESTS: (Failing cases, error messages, outputs)
CHANGES: (Code edits, variable updates)
DEPS: (Dependencies, imports, external calls)
VERSION_CONTROL_STATUS: (Repository state, current branch, PR status,
commit history)

PRIORITIZE:
1. Adapt tracking format to match the actual task type
2. Capture key user requirements and goals
3. Distinguish between completed and pending tasks
4. Keep all sections concise and relevant

SKIP: Tracking irrelevant details for the current task type

（以下为代码任务示例格式与 <PREVIOUS_SUMMARY>/<TURN-0>..<TURN-20> 输入槽位，详见原文）
```

> **图 12–14 | 联合"评审器—摘要"提示词（共 3 部分）。** 与主实验的 LLM-Summary 提示词（图 11）相比，还要求 LLM 在摘要的同时充当被摘要轮次的无执行评审器：先生成 `<CHECKPOINT>`（当前仓库状态与任务进度，含 USER_CONTEXT / CODE_STATE / TESTS / CHANGES / DEPS 各节），再按引导性问题清单（是否偏离用户需求？是否取得进展？是否陷入循环？有哪些问题模式？哪个关键信息能帮助回到正轨？还有哪些方法没试过？）生成最多 2 条互斥的 `<REFLECTIONS>`（每条识别一个关键阻塞问题并附 1–2 个修复建议）。提示词原文为英文，含一个 REST API 认证故障排查的完整少样本示例，详见原文。

---

## NeurIPS 论文检查清单（导览）

NeurIPS 检查清单为标准化模板内容，此处不逐条翻译，以下为各条目要点与作者的回答：

| # | 条目 | 作者回答 | 要点 |
| --- | --- | --- | --- |
| 1 | Claims（声明） | Yes | 摘要与引言的声明在第 4、5 节详述并有实验支持 |
| 2 | Limitations（局限） | Yes | 第 6 节明确记录研究局限与效度威胁 |
| 3 | Theory assumptions & proofs | N/A | 实证研究，不含理论结果 |
| 4 | Experimental result reproducibility | Yes | 第 3 节完整记录模型、超参、基准 |
| 5 | Open access to data and code | Yes | 代码（GitHub）与实验数据（HuggingFace）均已开源 |
| 6 | Experimental setting/details | Yes | 同上，含附录 D 的配置细节 |
| 7 | Statistical significance | Yes | bootstrap 置信区间与 p 值（附录 C.1） |
| 8 | Compute resources | Yes | 第 3 节披露 H200 集群等基础设施 |
| 9 | Code of ethics | Yes | 有助于降低 AI 的气候与环境影响 |
| 10 | Broader impacts | Yes | 第 7 节强调降低计算成本的环境正效应 |
| 11 | Safeguards | N/A | 使用现有模型与基准，无滥用风险 |
| 12 | Licenses for existing assets | Yes | 已恰当引用 SWE-bench、SWE-agent、OpenHands 等 |
| 13 | New assets | Yes | LLM-Summary 实现以良好文档发布 |
| 14 | Crowdsourcing / human subjects | N/A | 无众包、无人类被试 |
| 15 | IRB approvals | N/A | 无人类被试研究 |
| 16 | Declaration of LLM usage | Yes | LLM 是研究对象本身，全程明确披露 |

---

## 术语与背景速览（译者整理）

| 术语 | 含义 |
| --- | --- |
| Observation Masking（观察屏蔽） | 用占位符替换超过 M 轮前的环境观察输出，保留完整推理与动作历史（SWE-agent 的做法） |
| LLM-Summary（LLM 摘要） | 用一个摘要 LLM 定期把旧轮次压缩为一份滚动摘要，保留最近 M 轮原文（OpenHands / Cursor 的做法） |
| Raw Agent（原始智能体） | 不做任何上下文管理的基线，上下文随轮次无限增长 |
| Trajectory / Turn（轨迹/轮次） | 轨迹是智能体与环境的完整交互历史；一轮 T=(r,a,o) 由推理、动作、观察组成 |
| 轨迹拉长效应（Trajectory Elongation） | 摘要充当"强化信号"，让智能体更晚终止，轮数反而变多，抵消摘要的效率收益 |
| SWE-bench Verified | OpenAI 人工校验过的 500 道 SWE-bench 实例子集，软件工程智能体的业界标准基准 |
| SWE-agent / OpenHands | 两个主流开源软件工程智能体框架（NeurIPS 2024 / ICLR 2025） |
| 解决率（Solve Rate） | 成功解决 GitHub issue（通过测试）的实例比例，衡量"效果" |
| 实例成本（Instance Cost） | 平均每个实例的美元成本，衡量"效率" |
| lost in the middle | LLM 无法有效利用长上下文中部信息的著名现象（Liu et al., TACL 2024） |
| 无执行反馈（Execution-free Feedback） | 不实际运行代码、由 LLM 直接给出批评/反思的技术（Reflexion 等） |
| KV-cache | 推理时缓存的键值张量；前缀缓存命中可大幅降低成本，摘要调用破坏缓存复用 |
| 混合方法（Hybrid） | 先用观察屏蔽积累轮次（N=43），再触发 LLM-Summary 约束上下文上界 |

**核心数字速记**

- 环境**观察 token 占轨迹约 84%**——观察屏蔽之所以有效，正是因为瞄准了 token 构成的大头
- 不做上下文管理，成本会**翻倍以上**；两种策略都能把成本**砍掉 >50%**，且常常还提升解决率
- 观察屏蔽 5 个设置中 **4 个成本最低**；Qwen3-Coder 480B 上比 Raw Agent **便宜 52.7% 且解决率 +2.6pp**（53.4%→54.8%）
- LLM-Summary 从未显著超越观察屏蔽；Gemini 2.5 Flash (thinking) 上反而**低 5 个百分点**
- 摘要 API 调用最高占总成本 **7.2%**（Qwen3-Coder 480B），且无法利用前缀缓存
- LLM-Summary 平均轨迹长度比观察屏蔽**长 13–15%**（轨迹拉长效应）
- 混合方法（N=43, M=W=10）：比观察屏蔽再省 **7%**、比 LLM-Summary 再省 **11%**，解决率再 **+2.6pp**；全基准预计省 **$20 / $35**
- OpenHands 上观察屏蔽的窗口 M 需要**按框架重新调优**（SWE-agent 的 M=10 直接搬过去会性能崩坏，需 M=58）
- 附录 D.3：评审器增强摘要**没有收益**，反而加剧轨迹拉长——"更聪明的摘要"不是答案
