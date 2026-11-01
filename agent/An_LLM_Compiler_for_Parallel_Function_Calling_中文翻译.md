# 面向并行函数调用的 LLM 编译器（LLMCompiler）

> **原文**: An LLM Compiler for Parallel Function Calling
> **作者**: Sehoon Kim\*, Suhong Moon\*, Ryan Tabrizi, Nicholas Lee, Michael W. Mahoney, Kurt Keutzer, Amir Gholami（UC Berkeley / ICSI / LBNL；\* 同等贡献，通讯作者 amirgh@berkeley.edu）
> **日期/arXiv**: ICML 2024；arXiv:2312.04511v3（2024-06-05）
> **链接**: https://arxiv.org/abs/2312.04511 ｜ 代码：https://github.com/SqueezeAILab/LLMCompiler
> **译者说明**: 本文为完整全文翻译，正文与附录均逐段译出；图注按图意译出并补充图内元素说明；表格数据按原文重建为 Markdown 表。带 *（译者注）* 标记的段落为译者补充的背景说明，非原文内容。参考文献保留原文格式。

---

## 摘要

近期 LLM 的**推理能力**使其能够执行**外部函数调用**，从而克服自身的固有局限——知识截止（knowledge cutoff）、算术能力弱、无法访问私有数据等。这一进展使 LLM 可以基于上下文**选择并协调多个函数**，以解决更复杂的问题。然而，当前的函数调用方法通常要求**为每个函数做顺序的"推理—行动"（reasoning and acting）**，这会导致高延迟、高成本，有时还会产生不准确的行为。

为解决这一问题，我们提出 **LLMCompiler**：通过**并行执行函数**来高效编排多个函数调用。LLMCompiler 借鉴经典编译器的原理，由三个组件实现并行函数调用：

1. **函数调用规划器（Function Calling Planner）**：为函数调用制定执行计划；
2. **任务取指单元（Task Fetching Unit）**：派发函数调用任务；
3. **执行器（Executor）**：并行执行这些任务。

LLMCompiler 能自动为函数调用生成**优化的编排（optimized orchestration）**，且同时适用于开源与闭源模型。我们在多种具有不同函数调用模式的任务上对 LLMCompiler 做了基准测试，与 ReAct 相比，观察到**稳定的延迟加速最高 3.7×**、**成本节省最高 6.7×**、**准确率提升最高约 9%**。代码已开源于 https://github.com/SqueezeAILab/LLMCompiler。

---

## 1 引言

> **图 1 |** LLMCompiler 与 ReAct（Yao et al., 2022）在运行时行为上的对比示意，输入是 HotpotQA 基准（Yang et al., 2018）中的一道样例问题。在 LLMCompiler（右）中，**规划器（Planner）**首先把查询分解为若干带相互依赖的任务；随后**执行器（Executor）**在遵守依赖关系的前提下**并行**执行多个任务；最后由 LLMCompiler 汇集所有工具执行的观测（observation）产出最终回答。相比之下，ReAct（左）等现有框架的**串行**工具执行会导致更长的端到端延迟。本例中 LLMCompiler 在 HotpotQA 上取得了 **1.8×** 的延迟加速。此处仅用一道 2 路可并行的问题做直观展示，LLMCompiler 实际可处理更复杂的依赖模式（见图 2 与第 5 节）。

大语言模型（LLM）推理能力的近期进展，已把 LLM 的适用范围从内容生成扩展到**求解复杂问题**（Wei et al., 2022; Kojima et al., 2023; Yao et al., 2023b; Besta et al., 2023; Wang et al., 2023b; Zhou et al., 2023b; Chen et al., 2023b; Yang et al., 2022; Gao et al., 2022）；近来的工作也表明，这种推理能力有助于提升复杂逻辑任务的准确率。推理能力还催生了**函数（即工具）调用**能力——LLM 可以调用给定函数，并利用函数输出帮助完成任务。这些函数既可以是执行算术运算的简单计算器，也可以是基于 LLM 的更复杂函数。

LLM 整合多种工具与函数调用的能力，可能带来我们开发 LLM 软件方式的**根本性转变**。但这引出一个重要挑战：**整合多个函数调用的最有效方式是什么？**

一个代表性方案是 **ReAct**（Yao et al., 2022）：LLM 调用一个函数、分析结果，再推理下一步动作（即下一次函数调用）。以图 1（左）的简单例子来说，当被问及 "Scott Derrickson 与 Ed Wood 是否同一国籍"时，ReAct 先分析查询，决定用搜索工具查 Scott Derrickson；该搜索结果（即观测）被拼回原始 prompt，供 LLM 推理下一步动作——再调用一次搜索工具去收集 Ed Wood 的信息。

ReAct 是函数调用领域的开创性工作，且已被集成进多个框架（Langchain; Liu, 2022）。然而，要把这一方法扩展到更复杂的应用，需要大量优化。原因在于 ReAct 的**串行本质**：它一次一个地执行函数调用并对观测做推理。这种做法，以及在其之上扩展出的各类 agent 系统（Khot et al., 2023; Yao et al., 2023b; Qin et al., 2023; Ruan et al., 2023b; Sumers et al., 2023），会因**串行函数调用**与**每个推理/行动步骤都重复调用 LLM** 而在延迟与成本上低效。此外，虽然对观测做动态推理在某些场景下有益，但把中间函数调用的结果拼接到上下文中会**干扰 LLM 的执行流**，可能反而降低准确率（Xu et al., 2023）。常见的失败案例包括：**重复调用同一函数**（原论文也强调了这一点，Yao et al., 2022），以及**基于不完整中间结果的过早停止（early stopping）**，后文第 5.1 节与附录 A 会进一步讨论。

为解决这一挑战，我们借鉴**经典编译器**——传统编程语言中"优化指令执行"已被广泛研究。编译器的一项关键优化技术就是**识别可并行执行的指令并有效管理其依赖关系**。同理，可以设想一个专为 LLM 函数调用定制的编译器，高效地编排各种函数调用及其依赖。这与近期把 LLM 与计算机系统对齐的研究思路一致（Karpathy, 2023; Packer et al., 2023）。

为此，我们提出 **LLMCompiler**：一个让 LLM 在不同模型、不同负载下实现**多工具并行执行**的新框架。据我们所知，LLMCompiler 是**第一个**不仅改善延迟与成本、还能通过**减少中间函数调用输出的干扰**来提升**准确率**的函数调用编排优化框架。具体贡献如下：

- 我们提出 LLMCompiler，一个优化 LLM 并行函数调用性能的 LLM 编译器。其核心由三个关键组件构成：(i) **函数调用规划器**（第 3.1 节）识别执行流；(ii) **任务取指单元**（第 3.2 节）并行派发函数调用；(iii) **执行器**（第 3.3 节）用对应函数执行被派发的任务。
- 我们在**完全可并行（embarrassingly parallel）**模式上评估 LLMCompiler，使用 HotpotQA（Yang et al., 2018）与 Movie Recommendation（Srivastava et al., 2022）；相比 ReAct，取得 **1.80× / 3.74×** 的加速与 **3.37× / 6.73×** 的成本下降（第 5.1 节）。
- 为测试更复杂的模式，我们提出新基准 **ParallelQA**，其中包含多种非平凡的函数调用模式；相比 ReAct，取得最高 **2.27×** 加速、**4.65×** 成本下降与 **9%** 准确率提升（第 5.2 节）。
- 我们评估了 LLMCompiler 的**动态重规划（dynamic replanning）** 能力——通过执行器向函数调用规划器的**反馈回路**实现。在需要依据中间结果反复重规划的 **Game of 24**（Yao et al., 2023b）上，LLMCompiler 相比 Tree-of-Thoughts 取得 **2×** 加速（第 5.3 节）。
- 我们展示 LLMCompiler 能高效地探索**交互式决策环境**：在 WebShop 上，相比基线取得最高 **101.7×** 加速与 **25.7%** 的成功率提升（第 5.4 节）。

---

## 2 相关工作

### 2.1 LLM 的延迟优化

已有大量研究聚焦于**模型设计**（Kim et al., 2023; Frantar et al., 2022; Lin et al., 2023; Dettmers et al., 2023; Kwon et al., 2022; Frantar & Alistarh, 2023; Kim et al., 2024; Chen et al., 2023a; Leviathan et al., 2023）与**系统层面**（Kwon et al., 2023; Yu et al., 2022; trt; tgi）的高效 LLM 推理优化。但**应用层**的优化探索较少——而从实践角度看，这一层至关重要：当面对黑盒 LLM 模型与服务时，对模型本身及其推理流水线的修改受到极大限制。

**Skeleton-of-Thought**（Ning et al., 2023）近期提出通过应用层**并行解码**降低延迟：先做骨架生成，再并行执行各骨架条目。但它主要面向完全可并行的负载，**假设骨架任务之间没有依赖**，因而无法支持本质上相互依赖的任务；这限制了它在编程（Chen et al., 2021; Madaan et al., 2023; Hendrycks et al., 2021a; Austin et al., 2021）或数学（Hendrycks et al., 2021c; Hendrycks et al., 2021b）等问题上的适用性（Ning et al., 2023 亦如此说明）。LLMCompiler 通过把输入查询翻译成一组**带相互依赖的任务**来解决这一点，从而扩大了可处理问题的范围。

与我们的工作同期，OpenAI 在其 1106 版本中引入了**并行函数调用**特性，通过同时生成多个函数调用来增强用户查询处理（OpenAI, 2023）。尽管它有降低 LLM 执行时间的潜力，但存在明显限制：**仅对 OpenAI 的专有模型可用**。而随着开源 LLM 数量增长以及参数高效训练技术（Lester et al., 2021; Hu et al., 2022; Houlsby et al., 2019）的普及，使用开源模型做微调与定制的需求正日益增长。LLMCompiler 为开源模型带来了高效的并行函数调用；并且后文第 5 节将展示，它还可能取得更好的延迟与成本表现。

> **图 2 | LLMCompiler 框架总览。** **函数调用规划器**生成带相互依赖关系的**任务 DAG**；这些任务由**任务取指单元**按依赖关系并行派发给**执行器**。示例中 Task \$1 与 \$2 被一起取出，并行执行两个独立的搜索任务；每个任务完成后结果回传任务取指单元，由其把占位变量（如 Task \$3 中的 \$1、\$2）替换为实际值，从而**解锁**依赖任务。所有任务执行完毕后，最终回答交付给用户。

### 2.2 Plan-and-Solve 策略

已有若干研究（Wolfson et al., 2020; Patel et al., 2022; Press et al., 2023; Zhou et al., 2023b; Hao et al., 2023）探索把复杂查询分解为不同粒度的提示方法，以提升 LLM 在推理任务上的表现。具体来说：**Decomposed Prompting**（Khot et al., 2023）把复杂任务拆成更简单的子任务，每个子任务由带专用 prompt 的 LLM 解决；**Step-Back Prompting**（Zheng et al., 2023）让 LLM 从细节中抽象出高层概念以增强跨任务推理；**Plan-and-Solve Prompting**（Wang et al., 2023a）把多步推理任务切分为子任务，无需人工 prompt 即可减少错误、提升准确率。但这些方法主要关注**推理基准的准确率**；LLMCompiler 则用规划器识别查询中**可并行的模式**，目标是在保持准确率的同时**降低延迟**。

除上述工作外，**TPTU**（Ruan et al., 2023a）、**HuggingGPT**（Shen et al., 2023）与 **ViperGPT**（Surís et al., 2023）提出了端到端的 plan-and-solve 框架。LLMCompiler 的差异在于：它提供一个**通用框架**，能在更广泛的问题上做到高效且准确的函数调用。这源于 LLMCompiler 在 (i) 规划与重规划；(ii) 并行执行；(iii) 覆盖更广问题域 三方面的能力，详见附录 F。

另一个值得注意的工作是 **ReWOO**（Xu et al., 2023）：它用规划器把推理过程与执行、观测阶段分离，相比 ReAct 降低 token 用量与成本。我们与 ReWOO 有几点不同：第一，LLMCompiler 支持**并行函数调用**，可同时降低延迟与成本；第二，LLMCompiler 支持**动态重规划**——这对执行流无法在开始时静态确定的问题至关重要（第 5.3 节）。

### 2.3 工具增强型 LLM

LLM 增强的推理能力使其能够调用用户提供的函数，并利用其输出有效完成任务。该主题的详细讨论见附录 C.1。

---

## 3 方法

为说明 LLMCompiler 的组件，我们用图 2 中的一个简单的 2 路并行示例。要回答"**微软的市值需要增长多少才能超过苹果的市值？**"，LLM 首先需要对两家公司的市值各做一次网络搜索，然后做一次除法。包括 ReAct 在内的现有框架会串行执行这些任务，但显然它们可以并行。关键问题是：**如何自动判定哪些任务可并行、哪些相互依赖，从而据此编排各任务的执行？**

LLMCompiler 通过由以下三个组件构成的系统实现：

- **函数调用规划器**（第 3.1 节）：生成任务序列及其依赖关系；
- **任务取指单元**（第 3.2 节）：依据中间结果替换参数并取出任务；
- **执行器**（第 3.3 节）：用关联工具执行任务。

使用 LLMCompiler 时，用户只需提供**工具定义**，以及可选的、给规划器用的**上下文示例**，详见第 4.1 节。

### 3.1 函数调用规划器

函数调用规划器负责生成待执行的任务序列及其相互依赖。例如图 2 中的 Task \$1 与 \$2 是两次可并行的独立搜索；而 Task \$3 依赖前两次搜索的结果。因此规划器的职责是：利用 LLM 的强大推理能力，**自动识别所需任务、其输入参数以及相互依赖关系**，本质上构造一张**任务依赖的有向无环图（DAG）**。若某任务依赖前序任务，它会在自身中嵌入**占位变量**（如 Task 3 中的 \$1），稍后由前序任务的实际输出替换（第 3.2 节）。

LLMCompiler 的规划器利用 LLM 的推理能力，从自然语言输入中分解任务。为此，规划器 LLM 内置一段**预定义 prompt**，指导它如何构建依赖图并保证语法正确（详见附录 H）。除此之外，用户还需提供工具定义，以及可选的给规划器的上下文示例——这些示例针对具体问题演示任务分解的细节，帮助规划器更好地理解规则。关于用户输入信息的更多细节见第 4.1 节。第 4.2 节将介绍规划器的一项额外优化：**任务一生成就流式（stream）产出**，而不是等整个规划过程结束。

### 3.2 任务取指单元

任务取指单元受现代计算机体系结构中**取指单元（instruction fetching unit）** 的启发，基于**贪心策略**，只要任务已就绪就立刻（并行）取给执行器。它的另一个关键功能是**把占位变量替换为前序任务的实际输出**——这些占位符最初由规划器设定。以图 2 为例，Task \$3 中的 \$1 与 \$2 会被替换为微软与苹果的实际市值。这一功能可以用**简单的取指与排队机制**实现，无需额外的 LLM。

### 3.3 执行器

执行器**异步**执行从任务取指单元取来的任务。由于任务取指单元保证派发给执行器的所有任务彼此独立，执行器可以简单地**并发**执行它们。执行器配备了用户提供的工具，并把任务委派给关联工具——这些工具可以是计算器、Wikipedia 搜索、API 调用之类的简单函数，也可以是**为特定任务定制的 LLM agent**。如图 2 的 Executor 部分所示，每个任务都有**独立的内存**来存放其中间结果，这与典型串行框架把观测聚合成单个 prompt 的做法类似（Yao et al., 2022）。任务完成后，最终结果被前向传给依赖它的任务作为输入。

### 3.4 动态重规划

在许多应用中，执行图需要根据**先验未知的中间结果**自适应调整。编程中的类比是**分支（branching）**——执行路径要到运行时依据哪个分支条件成立才确定。这样的动态执行模式在 LLM 函数调用中同样会出现。对于简单分支（如 if-else），可以**静态编译**执行流，再依据中间结果动态选择；但对更复杂的分支，更好的做法是依据中间结果做**重新编译（recompilation）/ 重规划（replanning）**。

重规划时，中间结果从**执行器**回传给**函数调用规划器**，规划器据此生成一组新任务及其依赖；这些任务再送往任务取指单元，继而送往执行器。此循环持续到得到满意的最终结果并交付用户。第 5.3 节给出了一个示例：用 Tree-of-Thoughts 方法求解 Game of 24。

---

## 4 LLMCompiler 细节

### 4.1 用户提供的信息

LLMCompiler 需要用户提供两类输入：

1. **工具定义（Tool Definitions）**：用户需指定 LLM 可用的工具，包括其描述与参数规格。这与 ReAct、OpenAI function calling 等其他框架的要求基本一致。
2. **给规划器的上下文示例（In-context Examples）**：可选。用户可以给出"规划器应如何行动"的示例；例如在图 2 的情形下，可提供示例说明某些查询所期望的任务间依赖。这些示例能帮助规划器 LLM 理解如何使用各种工具，并以正确格式为新输入生成合适的依赖图。附录 G 收录了我们在评测中使用的示例。

### 4.2 流式规划器（Streamed Planner）

对涉及大量任务的用户查询，规划器会带来不可忽视的开销：它会**阻塞**任务取指单元与执行器——后者必须等规划器输出后才能启动。不过，类比现代计算机系统中的**指令流水线（instruction pipelining）**，可以让规划器**异步流式输出**依赖图，使每个任务在其所有依赖一解决时就能被执行器立即处理。表 C.1 给出了在各基准上"有/无流式"的延迟对比：流式带来稳定的延迟改善，在 ParallelQA 上最高 **1.3×**。原因是 ParallelQA 中数学工具执行时间较长，能有效**掩盖**规划器生成后续任务的延迟；而 HotpotQA 与 Movie Recommendation 使用的搜索工具执行时间较短，掩盖效果不明显。

> **图 3 | 不同函数调用模式的问题示例及其依赖图。** HotpotQA 与 Movie Recommendation 数据集呈现模式 (a)；ParallelQA 数据集呈现模式 (b)、(c) 等。
> - **(a)** 需要分析每家公司的**最新 10-K 年报**（多路独立搜索）。
> - **(b)** 需要为每所学校做**三次搜索**，再做一次加法与一次比较。
> - **(c)** 需要搜索每个州在每个细分领域的年度医疗支出，对各州支出求和，再做比较。

---

## 5 结果

本节我们用多种模型与问题类型评估 LLMCompiler：既使用**闭源 GPT 系列**，也使用**开源 LLaMA-2**——后者用以展示 LLMCompiler 让开源模型也具备并行函数调用的能力。此外，LLM 可处理的并行函数调用模式有多种类型：从所有任务彼此独立、可完全并行的模式，到图 3 所示的更复杂依赖模式。我们还特别在 **Game of 24** 基准上评估 LLMCompiler，该基准需要依据中间结果做**动态重规划**，以体现框架对动态依赖图的适应性。最后，我们把 LLMCompiler 用于 **WebShop** 基准，展示其在决策任务中的潜力。整体上，我们先呈现简单执行模式的结果，再过渡到更复杂的模式。

> **表 1 | LLMCompiler 与基线在不同基准（HotpotQA、Movie Recommendation、自建数据集 ParallelQA、Game of 24）上的准确率与延迟对比。** 在 HotpotQA 与 Movie Recommendation 上，我们频繁观察到 **循环（looping）** 与 **过早停止（early stopping）**；为尽量抑制这两种行为，我们加入了 **ReAct 专用 prompt**，记为 **ReAct†**；不带 † 的 ReAct 表示未加该 prompt 的原始结果。原始 ReAct 因循环与过早停止难以精确测量延迟，故不报告其延迟。

| 基准 | 方法 | GPT（闭源）准确率 (%) | GPT 延迟 (s) | GPT 加速 | LLaMA-2 70B（开源）准确率 (%) | LLaMA 延迟 (s) | LLaMA 加速 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| HotpotQA | ReAct | 61.52 | – | – | 54.74 | – | – |
| HotpotQA | ReAct† | 62.47 | 7.12 | 1.00× | 54.40 | 13.44 | 1.00× |
| HotpotQA | OAI 并行函数调用 | 62.05 | 4.42 | 1.61× | – | – | – |
| HotpotQA | **LLMCompiler** | **62.00** | **3.95** | **1.80×** | **57.83** | **9.58** | **1.40×** |
| Movie Rec. | ReAct | 68.60 | – | – | 70.00 | – | – |
| Movie Rec. | ReAct† | 72.47 | 20.47 | 1.00× | 70.60 | 33.37 | 1.00× |
| Movie Rec. | OAI 并行函数调用 | 77.00 | 7.42 | 2.76× | – | – | – |
| Movie Rec. | **LLMCompiler** | **77.13** | **5.47** | **3.74×** | **77.80** | **11.83** | **2.82×** |
| ParallelQA | ReAct | 89.09 | 35.90 | 1.00× | 59.59 | 15.47 | 1.00× |
| ParallelQA | OAI 并行函数调用 | 87.32 | 19.29 | 1.86× | – | – | – |
| ParallelQA | **LLMCompiler** | **89.38** | **16.69** | **2.15×** | **68.14** | **26.20** | **2.27×** |
| Game of 24 | Tree-of-Thoughts | 74.00 | 241.2 | 1.00× | 30.00 | 952.06 | 1.00× |
| Game of 24 | **LLMCompiler** | **75.33** | **83.6** | **2.89×** | **32.00** | **456.02** | **2.09×** |

> **表 2 | 在 HotpotQA、Movie Recommendation 与自建数据集 ParallelQA 上的输入/输出 token 消耗与估计成本。** 成本按各基准所用 GPT 模型的价格表计算。注意 ParallelQA 用的是 gpt-4-turbo，单价远高于前两者的 gpt-3.5-turbo，故其绝对成本数值更大。

| 基准 | 方法 | 输入 token | 输出 token | 成本 (\$/1k) | 成本下降 |
| --- | --- | --- | --- | --- | --- |
| HotpotQA | ReAct | 2900 | 120 | 5.00 | 1.00× |
| HotpotQA | OAI 并行函数 | 2500 | 63 | 2.66 | 1.87× |
| HotpotQA | **LLMCompiler** | **1300** | 80 | **1.47** | **3.37×** |
| Movie Rec. | ReAct | 20000 | 230 | 20.46 | 1.00× |
| Movie Rec. | OAI 并行函数 | 5800 | 160 | 6.14 | 3.33× |
| Movie Rec. | **LLMCompiler** | **2800** | 115 | **3.04** | **6.73×** |
| ParallelQA | ReAct | 46000 | 470 | 480 | 1.00× |
| ParallelQA | OAI 并行函数 | 25000 | 370 | 260 | 1.81× |
| ParallelQA | **LLMCompiler** | **9200** | 340 | **103** | **4.65×** |

### 5.1 完全可并行的函数调用

最简单的场景是 LLM 对**相互独立的任务重复使用同一工具**，比如并行搜索或分析不同主题的信息，即图 3(a) 的模式。这些任务彼此独立、可以并行，但 ReAct 及当前其他 LLM 方案仍需**串行**执行，从而因每次工具使用都要调用 LLM 而增加延迟与 token 消耗（如图 1 所示）。本节展示 LLMCompiler 如何识别可并行模式并**并发执行独立任务**来解决该问题。我们使用以下两个基准：

- **HotpotQA**：评估多跳推理的数据集（Yang et al., 2018）。我们只使用其中的 **comparison dev set**，包含 1.5k 道比较两个不同实体的问题，呈现 **2 路完全可并行**执行模式。示例问题见图 1。
- **Movie Recommendation**：500 个样本的数据集，要求从另外四部电影中找出与给定四部最相似的一部，呈现 **8 路完全可并行**模式（Srivastava et al., 2022）。

**实验设置。** 基线方法为 ReAct。我们沿用 ReAct 的设置（Yao et al., 2022），使用同样的 Wikipedia 搜索工具；由于 lookup 工具与我们的问题设定无关，未纳入。我们对 ReAct 与 LLMCompiler 的 prompt 及上下文示例都做了力所能及的最优调优。这些数据集上的所有实验均使用 **gpt-3.5-turbo（1106 版本）**。对 GPT 实验，我们还额外报告了 OpenAI 并行函数调用能力的结果——该能力与我们的工作同期发布。我们也展示了 LLMCompiler 如何与开源 **LLaMA-2 70B** 有效结合，为该模型赋予并行函数调用能力。所有实验都测量了准确率、端到端延迟以及输入/输出 token 用量。实验设置细节见附录 D。

**准确率与延迟。** 表 1 报告了准确率、端到端延迟与相对加速。首先，ReAct 的准确率始终低于 OpenAI 并行函数调用与 LLMCompiler。我们识别出 ReAct 的两大失败模式：(1) 倾向于**冗余地重复生成先前的函数调用**（原 ReAct 论文亦指出，Yao et al., 2022）；(2) 基于不完整的中间结果**过早停止**。附录 A 的详细分析表明，这两类高发失败显著拉低了 ReAct 的准确率，而 LLMCompiler 可将其解决，带来最高 **7–8%** 的准确率提升。此外我们还做了干预实验：为 ReAct 加入专用 prompt 以避免重复调用与过早停止——表 1 中的 **ReAct†** 即加入该 prompt 的 ReAct。该 prompt 普遍带来准确率提升，但 LLMCompiler 仍与 ReAct† 持平或更好，因为这种提示并不能彻底消除 ReAct 的错误行为。

与 ReAct† 相比，LLMCompiler 在 HotpotQA 上用 GPT 与 LLaMA 分别取得 **1.80×** 与 **1.40×** 的明显加速；在 Movie Recommendation 上分别为 **3.74×** 与 **2.82×**。注意我们以 ReAct† 作为延迟对比对象：原始 ReAct 的重复与过早停止行为使延迟不可预测，不适合公平比较。相比 OpenAI 并行函数调用（其对 ReAct 的延迟收益分别为 1.61× 与 2.76×），LLMCompiler 最高再快 **35%**。¹

> ¹ 遗憾的是我们无法断定原因，因为 OpenAI 未公开其函数调用机制的任何细节。一个猜测是：校验函数名与参数名、并转换为 system prompt 可能存在额外开销。不过我们在多天、多次运行中观察到一致的规律。

**成本。** 使用 LLM 的另一重要考量是成本，它取决于输入与输出 token 用量。GPT 实验的成本见表 2。由于 LLM 调用次数更少，LLMCompiler 在成本上比 ReAct 更高效。有趣的是，LLMCompiler 在成本效率上也优于近期 OpenAI 的并行函数调用——原因是 LLMCompiler 的**规划阶段 prompt 长度更省**：我们规划器的上下文示例相当短，且只包含计划、不含观测（见附录 H）。

### 5.2 带依赖的并行函数调用

上述场景相当简单：只用一种工具，且所有任务彼此独立。然而，如同传统代码块中的执行一样，我们也会遇到**依赖关系更复杂**的函数调用场景。为系统评估在复杂任务依赖下规划函数调用的能力，我们设计了自建基准 **ParallelQA**，其中纳入了图 3(b)(c) 所示的**三类非平凡函数调用模式**。受 IfQA 基准（Yu et al., 2023）启发，ParallelQA 含 **113** 个样本，均为关于不同实体事实属性的**数学问题**。完成任务需要两个工具（**搜索**与**数学**），且第二个工具的参数依赖于第一个工具的输出。我们精心挑选的问题**仅凭 Wikipedia 首段信息即可作答**，从而有效排除"搜索失败"导致的错误。ParallelQA 的更多细节见附录 I。

**实验设置。** 与第 5.1 节类似，主基线为 ReAct（Yao et al., 2022）。ReAct 与 LLMCompiler 均配备两个工具：(1) **搜索工具**，与 5.1 节相同；(2) **数学工具**，用于求解数学问题。数学工具受 Langchain 的 **LLMMathChain** 启发：用 LLM 作为 agent 解释输入查询，并以合适公式调用 `numexpr`，从而能处理以数学形式和自然语言形式书写的广泛数学问题。实验细节见附录 D。

**准确率与延迟。** 如表 1 的 ParallelQA 行所示，通过避免依赖图的串行执行，LLMCompiler 平均取得 **2.15×**（gpt-4-turbo）与 **2.27×**（LLaMA-2 70B）的加速。除延迟外，LLaMA-2 上 LLMCompiler 的准确率也高于 ReAct，原因同第 5.1 节。特别是在 LLaMA-2 实验中 LLMCompiler 准确率提升约 **9%**，我们注意到 **约 20%** 的样本在 ReAct 下出现重复函数调用，与附录 A 的准确率分析一致。此外，附录 B 对 LLMCompiler 的失败案例做了全面分析，其中**规划器失败极少**，凸显其在把问题分解为复杂多任务依赖上的有效性。

**成本。** 与第 5.1 节一致，如表 2 所示，相比 ReAct 与 OpenAI 并行函数调用，LLMCompiler 分别实现 **4.65×** 与 **2.57×** 的可观成本下降。这一效率源于 LLM 调用频率的降低；OpenAI 的并行函数调用虽同样降低了调用频率，但它只规划**当前可立即并行**的任务，而非**完整依赖图**。例如在图 3(c) 中，OpenAI 的方法需要三次独立的 LLM 调用（初始搜索任务、后续数学任务、最终数学任务），而 LLMCompiler 只需**一次** LLM 调用即可并发规划全部任务。

### 5.3 带重规划的并行函数调用

前文讨论的都是可**静态确定**依赖图的情形。但有些情况下依赖图必须**依据中间观测动态构建**。这里我们考虑一种动态方法：在 **Game of 24** 上使用（Yao et al., 2023b）提出的 **Tree-of-Thoughts（ToT）** 策略。Game of 24 要求用四个数字与基本四则运算生成 24，例如由 2、4、4、7 可得 $4\times(7-4)\times 2=24$。ToT 通过两个迭代式 LLM 过程求解：(i) **thought proposer（想法提出器）** 选择两个数字做一次运算，产生候选部分解（如由 2、4、4、7 算出 7−4 得 2、3、7）；(ii) **state evaluator（状态评估器）** 评估各候选的潜力。只有有希望的候选进入下一轮 proposer/evaluator 迭代，直到得到 24。Game of 24 基准与 ToT 的细节见附录 J。

ToT 在 Game of 24 上有显著提升，但其沿状态树的**串行广度优先搜索**非常耗时。LLMCompiler 提供了更快的替代方案：让 thought proposer 与随后的可行性评估器**并行执行**，类似**并行 beam search**。

**实验设置。** 尽管 LLMCompiler 有延迟优势，但用单一静态图解决该问题并不可行——因为规划器无法在从上一轮的 state evaluator 得到候选之前就规划出想法提出阶段。因此规划器只能**在一轮之内**做规划。为此我们启用 LLMCompiler 的**重规划**能力。具体地，LLMCompiler 配备三个工具：`thought_proposer` 与 `state_evaluator`（均为改编自原 ToT 框架的 LLM），以及 `top_k_select`（依据 state evaluator 的评估从 proposer 的候选中挑出 top-k）。这些工具全部执行后，若没有提案达到 24，LLMCompiler 可决定"**重规划**"，触发规划器基于上一轮 `top_k_select` 的入围状态制定新计划。如此，LLMCompiler 能**动态重新生成每一轮的计划**，从而应对需要依据上一轮计划结果迭代重规划的高复杂度任务。

为评估 Game of 24 上的表现，我们使用 **100** 个不同实例。只有当运算合法、结果为 24、且每个给定数字恰好使用一次时，才判定该题成功。实验设置细节见附录 D。

**成功率与延迟。** 表 1 的最后两行对比了 LLMCompiler 与（Yao et al., 2023b）基线在 Game of 24 上的延迟与成功率。用 gpt-4 时，LLMCompiler 延迟改善 **2.89×**，成功率还略有提升；用 LLaMA-2 时延迟改善 **2.01×**，成功率同样未退化。这些结果不仅说明大幅降延迟而不损质量，也验证了 LLMCompiler 的重规划能力可求解复杂问题。

### 5.4 应用：交互式决策任务中的 LLMCompiler

本节我们在 **WebShop**（Yao et al., 2023a）上做基准测试，说明 LLMCompiler 能有效探索基于语言的交互环境。如（Shinn et al., 2023; Yao et al., 2022; Yao et al., 2023a）所强调的，WebShop 多样性很高，需要大量探索才能买到最合适的商品。尽管近期工作提出了先进探索策略并有不错结果（Zhou et al., 2023a; Ma et al., 2023），但它们大多基于**串行且繁重的树搜索**，带来明显的延迟代价。此处 LLMCompiler 展示了一种利用并行函数调用的**既有效又高效**的探索策略：更广地探索环境中的商品，从而提高成功率；同时这种探索可以并行化，相比串行探索的基线最高取得 **101.7×** 加速。

**实验设置。** 我们在此基准上对比三个基线：ReAct（Yao et al., 2022）、LATS（Zhou et al., 2023a）与 LASER（Ma et al., 2023），使用 **500** 条 WebShop 指令。指标为成功率、平均得分与延迟。WebShop 环境与基线的更多细节见附录 K。本实验中 LLMCompiler 配备两个工具：`search` 与 `explore`。`search` 触发模型生成并发送查询，返回 WebShop 环境中的（通常十条）商品列表；`explore` 则逐个点击所找到商品的链接，取回可选规格、价格、属性与特性信息。最后，LLMCompiler 依据汇总信息决定最符合购买指令的商品。实验细节见附录 D。

**性能与延迟。** 如表 3 所示，我们的方法显著优于所有基线。用 gpt-3.5-turbo 时，LLMCompiler 相比 ReAct 与 LATS 的成功率分别提升 **28.4%** 与 **6%**；用 gpt-4 时，相比 ReAct 与 LASER 分别提升 **20.4%** 与 **5.6%**。延迟方面，相比 LATS 与 LASER 分别取得 **101.7×** 与 **2.69×** 加速。我们注意到 LLMCompiler 在本基准上略慢于 ReAct（主要因为规划器开销），但成功率的收益远超这点延迟代价。

> **表 3 | WebShop 上的性能与延迟分析。** 用 gpt-4 与 gpt-3.5-turbo 两个模型评估 LLMCompiler，对比 ReAct、LATS、LASER 三个基线。成功率与平均得分以百分比报告。ReAct 的成功率与平均得分由我们复现，LATS 与 LASER 的数据取自各自论文。N 为评测样本数。

| 模型 | 方法 | 成功率 (%) | 得分 | 延迟 (s) | N |
| --- | --- | --- | --- | --- | --- |
| gpt-3.5-turbo | ReAct | 19.8 | 54.2 | 5.98 | 500 |
| gpt-3.5-turbo | LATS | 38.0 | 75.9 | 1066 | 50 |
| gpt-3.5-turbo | LLMCompiler | 44.0 | 72.8 | 10.72 | 50 |
| gpt-3.5-turbo | **LLMCompiler** | **48.2** | **74.2** | **10.48** | 500 |
| gpt-4-0613 | ReAct | 35.2 | 58.8 | 19.90 | 500 |
| gpt-4-0613 | LASER | 50.0 | 75.6 | 72.16 | 500 |
| gpt-4-0613 | **LLMCompiler** | **55.6** | **77.1** | **26.73** | 500 |

我们进一步分析 LLMCompiler 相比 ReAct 为何能有更高的成功率与得分。根据观察，ReAct agent 倾向于在**信息不完整**时就下决定——当 agent 尚未收集足够多的商品特性与规格细节时就会发生。这一观察也在（Shinn et al., 2023）中被指出：若不更多地探索环境中的商品，agent 难以区分看似相似的选项，最终做出错误决策。相比之下，LLMCompiler 会进一步探索——**访问 search 找到的全部十个商品**并取回各自相关信息。我们发现，采用有效的搜索策略对 WebShop 这类决策任务至关重要。

LATS 相对较高的性能也可用其探索方式解释：该框架对 WebShop 的状态与动作空间做**暴力搜索**，在最终购买前探索多达 **30 条轨迹**。这种做法为决策提供了更丰富的信息，但端到端执行慢到难以接受。

我们报告 LLMCompiler 平均得分比 LASER 高 **1.5**；与 LATS 相比，该差距落在我们方法的标准差范围内（gpt-3.5-turbo 下 LLMCompiler 平均得分为 **72.8 ± 4.01**）。需要强调，尽管性能差异不大，我们的方法在执行速度上有显著优势：相比 LATS 快 **101.7×**，相比 LASER 快 **2.69×**。

---

## 6 结论

现有用 LLM 调用多个函数的方法依赖**串行且动态的推理**，因此在延迟、成本与准确率上都存在低效。作为解决方案，我们提出 **LLMCompiler**——一个受编译器启发的框架，可在多种 LLM（包括 LLaMA-2 等开源模型与 OpenAI 的 GPT 系列）上实现高效的**并行函数调用**。通过把用户输入分解为具有明确相互依赖的任务，并经由**规划器、任务取指单元、执行器**三组件并发执行，LLMCompiler 在延迟（最高 **3.7×**）、成本效率（最高 **6.7×**）与准确率（最高 **约 9%**）上均有实质提升，其延迟收益甚至优于 OpenAI 的并行函数调用特性。我们期待未来在 LLMCompiler 之上继续推进工作，进一步提升 LLM 执行复杂、大规模任务的能力与效率，从而改变 LLM 应用开发的未来。

**影响声明（Impact Statement）**：本文是推进机器学习领域的研究工作。尽管我们的工作可能带来诸多潜在社会影响，但我们未发现任何一项特别值得强调。

---

## 致谢

我们感谢 Minwoo Kang 的宝贵反馈，感谢 Furiosa 团队的慷慨支持。我们也感谢微软通过其 "Accelerating Foundation Model Research" 计划给予的支持，特别是 Sean Kuno 的大力帮助。此外，我们感谢 Google Cloud、Google TRC 团队的支持，特别是 Jonathan Caton 与 David Patterson 教授。Keutzer 教授的实验室由 Intel 公司、Intel One-API、Intel VLAB 团队、Intel One-API 卓越中心以及 BDD 与 BAIR 的经费赞助。我们也感谢三星的支持，包括 Dongkyun Kim 与 David Thorsley。我们感谢 Ellick Chan、Saurabh Tangri、Andres Rodriguez 与 Kittur Ganesh 的大力支持。Sehoon Kim 与 Suhong Moon 感谢韩国高等研究院（Korea Foundation for Advanced Studies）的支持；Amir Gholami 由三星 SAIT 经费支持；Michael W. Mahoney 亦感谢 J. P. Morgan Chase 教职研究奖以及 DOE、NSF、IARPA 的支持。我们的结论不一定反映赞助方的立场或政策，也不应推断为官方背书。

---

## 参考文献（保留英文原文，未翻译，便于检索）

- NVIDIA TensorRT-LLM. https://github.com/nvidia/tensorrt-llm.
- Austin et al. (2021) Austin, J., Odena, A., Nye, M., Bosma, M., Michalewski, H., Dohan, D., Jiang, E., Cai, C., Terry, M., Le, Q., and Sutton, C. Program synthesis with large language models, 2021.
- Besta et al. (2023) Besta, M., Blach, N., Kubicek, A., Gerstenberger, R., Gianinazzi, L., Gajda, J., Lehmann, T., Podstawski, M., Niewiadomski, H., Nyczyk, P., and Hoefler, T. Graph of thoughts: Solving elaborate problems with large language models, 2023.
- Chen et al. (2023a) Chen, C., Borgeaud, S., Irving, G., Lespiau, J.-B., Sifre, L., and Jumper, J. Accelerating large language model decoding with speculative sampling, 2023a.
- Chen et al. (2021) Chen, M., Tworek, J., Jun, H., Yuan, Q., de Oliveira Pinto, H. P., Kaplan, J., Edwards, H., Burda, Y., Joseph, N., Brockman, G., Ray, A., Puri, R., Krueger, G., Petrov, M., Khlaaf, H., Sastry, G., Mishkin, P., Chan, B., Gray, S., Ryder, N., Pavlov, M., Power, A., Kaiser, L., Bavarian, M., Winter, C., Tillet, P., Such, F. P., Cummings, D., Plappert, M., Chantzis, F., Barnes, E., Herbert-Voss, A., Guss, W. H., Nichol, A., Paino, A., Tezak, N., Tang, J., Babuschkin, I., Balaji, S., Jain, S., Saunders, W., Hesse, C., Carr, A. N., Leike, J., Achiam, J., Misra, V., Morikawa, E., Radford, A., Knight, M., Brundage, M., Murati, M., Mayer, K., Welinder, P., McGrew, B., Amodei, D., McCandlish, S., Sutskever, I., and Zaremba, W. Evaluating large language models trained on code. 2021.
- Chen et al. (2023b) Chen, W., Ma, X., Wang, X., and Cohen, W. W. Program of thoughts prompting: Disentangling computation from reasoning for numerical reasoning tasks, 2023b.
- Dettmers et al. (2023) Dettmers, T., Svirschevski, R., Egiazarian, V., Kuznedelev, D., Frantar, E., Ashkboos, S., Borzunov, A., Hoefler, T., and Alistarh, D. Spqr: A sparse-quantized representation for near-lossless llm weight compression, 2023.
- Frantar & Alistarh (2023) Frantar, E. and Alistarh, D. Sparsegpt: Massive language models can be accurately pruned in one-shot, 2023.
- Frantar et al. (2022) Frantar, E., Ashkboos, S., Hoefler, T., and Alistarh, D. GPTQ: Accurate post-training compression for generative pretrained transformers. arXiv preprint arXiv:2210.17323, 2022.
- Gao et al. (2022) Gao, L., Madaan, A., Zhou, S., Alon, U., Liu, P., Yang, Y., Callan, J., and Neubig, G. Pal: Program-aided language models. arXiv preprint arXiv:2211.10435, 2022.
- Hao et al. (2023) Hao, S., Gu, Y., Ma, H., Hong, J. J., Wang, Z., Wang, D. Z., and Hu, Z. Reasoning with language model is planning with world model, 2023.
- Hendrycks et al. (2021a) Hendrycks, D., Basart, S., Kadavath, S., Mazeika, M., Arora, A., Guo, E., Burns, C., Puranik, S., He, H., Song, D., and Steinhardt, J. Measuring coding challenge competence with apps. NeurIPS, 2021a.
- Hendrycks et al. (2021b) Hendrycks, D., Burns, C., Basart, S., Zou, A., Mazeika, M., Song, D., and Steinhardt, J. Measuring massive multitask language understanding. Proceedings of the International Conference on Learning Representations (ICLR), 2021b.
- Hendrycks et al. (2021c) Hendrycks, D., Burns, C., Kadavath, S., Arora, A., Basart, S., Tang, E., Song, D., and Steinhardt, J. Measuring mathematical problem solving with the math dataset. NeurIPS, 2021c.
- Houlsby et al. (2019) Houlsby, N., Giurgiu, A., Jastrzebski, S., Morrone, B., De Laroussilhe, Q., Gesmundo, A., Attariyan, M., and Gelly, S. Parameter-efficient transfer learning for nlp. In International conference on machine learning, pp. 2790–2799. PMLR, 2019.
- Hu et al. (2022) Hu, E. J., Shen, Y., Wallis, P., Allen-Zhu, Z., Li, Y., Wang, S., Wang, L., and Chen, W. LoRA: Low-rank adaptation of large language models. In International Conference on Learning Representations, 2022.
- Karpathy (2023) Karpathy, A. Intro to large language models, 2023.
- Khot et al. (2023) Khot, T., Trivedi, H., Finlayson, M., Fu, Y., Richardson, K., Clark, P., and Sabharwal, A. Decomposed prompting: A modular approach for solving complex tasks. In The Eleventh International Conference on Learning Representations, 2023.
- Kim et al. (2023) Kim, S., Hooper, C., Gholami, A., Dong, Z., Li, X., Shen, S., Mahoney, M. W., and Keutzer, K. Squeezellm: Dense-and-sparse quantization, 2023.
- Kim et al. (2024) Kim, S., Mangalam, K., Moon, S., Malik, J., Mahoney, M. W., Gholami, A., and Keutzer, K. Speculative decoding with big little decoder, 2024.
- Kojima et al. (2023) Kojima, T., Gu, S. S., Reid, M., Matsuo, Y., and Iwasawa, Y. Large language models are zero-shot reasoners, 2023.
- Kwon et al. (2022) Kwon, W., Kim, S., Mahoney, M. W., Hassoun, J., Keutzer, K., and Gholami, A. A fast post-training pruning framework for transformers, 2022.
- Kwon et al. (2023) Kwon, W., Li, Z., Zhuang, S., Sheng, Y., Zheng, L., Yu, C. H., Gonzalez, J. E., Zhang, H., and Stoica, I. Efficient memory management for large language model serving with pagedattention. In Proceedings of the ACM SIGOPS 29th Symposium on Operating Systems Principles, 2023. (25) Langchain. https://github.com/langchain-ai/langchain.
- Lester et al. (2021) Lester, B., Al-Rfou, R., and Constant, N. The power of scale for parameter-efficient prompt tuning, 2021.
- Leviathan et al. (2023) Leviathan, Y., Kalman, M., and Matias, Y. Fast inference from transformers via speculative decoding, 2023.
- Liang et al. (2023) Liang, Y., Wu, C., Song, T., Wu, W., Xia, Y., Liu, Y., Ou, Y., Lu, S., Ji, L., Mao, S., Wang, Y., Shou, L., Gong, M., and Duan, N. Taskmatrix.ai: Completing tasks by connecting foundation models with millions of apis, 2023.
- Lin et al. (2023) Lin, J., Tang, J., Tang, H., Yang, S., Dang, X., Gan, C., and Han, S. Awq: Activation-aware weight quantization for llm compression and acceleration, 2023.
- Liu (2022) Liu, J. LlamaIndex, 11 2022. URL https://github.com/jerryjliu/llama_index.
- Ma et al. (2023) Ma, K., Zhang, H., Wang, H., Pan, X., and Yu, D. Laser: Llm agent with state-space exploration for web navigation, 2023.
- Madaan et al. (2023) Madaan, A., Tandon, N., Gupta, P., Hallinan, S., Gao, L., Wiegreffe, S., Alon, U., Dziri, N., Prabhumoye, S., Yang, Y., Gupta, S., Majumder, B. P., Hermann, K., Welleck, S., Yazdanbakhsh, A., and Clark, P. Self-refine: Iterative refinement with self-feedback, 2023.
- Ning et al. (2023) Ning, X., Lin, Z., Zhou, Z., Wang, Z., Yang, H., and Wang, Y. Skeleton-of-thought: Large language models can do parallel decoding, 2023.
- OpenAI (2023) OpenAI. Gpt-4 technical report, 2023.
- OpenAI (2023) OpenAI. New models and developer products announced at devday, 2023.
- Packer et al. (2023) Packer, C., Fang, V., Patil, S. G., Lin, K., Wooders, S., and Gonzalez, J. E. Memgpt: Towards llms as operating systems, 2023.
- Patel et al. (2022) Patel, P., Mishra, S., Parmar, M., and Baral, C. Is a question decomposition unit all we need? 2022.
- Patil et al. (2023) Patil, S. G., Zhang, T., Wang, X., and Gonzalez, J. E. Gorilla: Large language model connected with massive apis, 2023.
- Press et al. (2023) Press, O., Zhang, M., Min, S., Schmidt, L., Smith, N. A., and Lewis, M. Measuring and narrowing the compositionality gap in language models, 2023.
- Qin et al. (2023) Qin, Y., Liang, S., Ye, Y., Zhu, K., Yan, L., Lu, Y., Lin, Y., Cong, X., Tang, X., Qian, B., et al. Toolllm: Facilitating large language models to master 16000+ real-world apis. arXiv preprint arXiv:2307.16789, 2023.
- Ruan et al. (2023a) Ruan, J., Chen, Y., Zhang, B., Xu, Z., Bao, T., Du, G., Shi, S., Mao, H., Zeng, X., and Zhao, R. Tptu: Task planning and tool usage of large language model-based ai agents. arXiv preprint arXiv:2308.03427, 2023a.
- Ruan et al. (2023b) Ruan, Y., Dong, H., Wang, A., Pitis, S., Zhou, Y., Ba, J., Dubois, Y., Maddison, C. J., and Hashimoto, T. Identifying the risks of lm agents with an lm-emulated sandbox. arXiv preprint arXiv:2309.15817, 2023b.
- Schick et al. (2023) Schick, T., Dwivedi-Yu, J., Dessì, R., Raileanu, R., Lomeli, M., Zettlemoyer, L., Cancedda, N., and Scialom, T. Toolformer: Language models can teach themselves to use tools. arXiv preprint arXiv:2302.04761, 2023.
- Shen et al. (2023) Shen, Y., Song, K., Tan, X., Li, D., Lu, W., and Zhuang, Y. Hugginggpt: Solving ai tasks with chatgpt and its friends in hugging face, 2023.
- Shinn et al. (2023) Shinn, N., Cassano, F., Berman, E., Gopinath, A., Narasimhan, K., and Yao, S. Reflexion: Language agents with verbal reinforcement learning, 2023.
- Song et al. (2023) Song, Y., Xiong, W., Zhu, D., Wu, W., Qian, H., Song, M., Huang, H., Li, C., Wang, K., Yao, R., Tian, Y., and Li, S. Restgpt: Connecting large language models with real-world restful apis, 2023.
- Srivastava et al. (2022) Srivastava, A., Rastogi, A., Rao, A., Shoeb, A. A. M., Abid, A., Fisch, A., Brown, A. R., Santoro, A., Gupta, A., Garriga-Alonso, A., et al. Beyond the imitation game: Quantifying and extrapolating the capabilities of language models. arXiv preprint arXiv:2206.04615, 2022.
- Sumers et al. (2023) Sumers, T. R., Yao, S., Narasimhan, K., and Griffiths, T. L. Cognitive architectures for language agents, 2023.
- Surís et al. (2023) Surís, D., Menon, S., and Vondrick, C. Vipergpt: Visual inference via python execution for reasoning, 2023.
- Touvron et al. (2023) Touvron, H., Martin, L., Stone, K., Albert, P., Almahairi, A., Babaei, Y., Bashlykov, N., Batra, S., Bhargava, P., Bhosale, S., Bikel, D., Blecher, L., Ferrer, C. C., Chen, M., Cucurull, G., Esiobu, D., Fernandes, J., Fu, J., Fu, W., Fuller, B., Gao, C., Goswami, V., Goyal, N., Hartshorn, A., Hosseini, S., Hou, R., Inan, H., Kardas, M., Kerkez, V., Khabsa, M., Kloumann, I., Korenev, A., Koura, P. S., Lachaux, M.-A., Lavril, T., Lee, J., Liskovich, D., Lu, Y., Mao, Y., Martinet, X., Mihaylov, T., Mishra, P., Molybog, I., Nie, Y., Poulton, A., Reizenstein, J., Rungta, R., Saladi, K., Schelten, A., Silva, R., Smith, E. M., Subramanian, R., Tan, X. E., Tang, B., Taylor, R., Williams, A., Kuan, J. X., Xu, P., Yan, Z., Zarov, I., Zhang, Y., Fan, A., Kambadur, M., Narang, S., Rodriguez, A., Stojnic, R., Edunov, S., and Scialom, T. Llama 2: Open foundation and fine-tuned chat models, 2023.
- Wang et al. (2023a) Wang, L., Xu, W., Lan, Y., Hu, Z., Lan, Y., Lee, R. K.-W., and Lim, E.-P. Plan-and-solve prompting: Improving zero-shot chain-of-thought reasoning by large language models. arXiv preprint arXiv:2305.04091, 2023a.
- Wang et al. (2023b) Wang, X., Wei, J., Schuurmans, D., Le, Q., Chi, E., Narang, S., Chowdhery, A., and Zhou, D. Self-consistency improves chain of thought reasoning in language models, 2023b.
- Wei et al. (2022) Wei, J., Wang, X., Schuurmans, D., Bosma, M., Xia, F., Chi, E., Le, Q. V., Zhou, D., et al. Chain-of-thought prompting elicits reasoning in large language models. volume 35, pp. 24824–24837, 2022.
- Wolfson et al. (2020) Wolfson, T., Geva, M., Gupta, A., Gardner, M., Goldberg, Y., Deutch, D., and Berant, J. Break it down: A question understanding benchmark. Transactions of the Association for Computational Linguistics, 2020.
- Xu et al. (2023) Xu, B., Peng, Z., Lei, B., Mukherjee, S., Liu, Y., and Xu, D. Rewoo: Decoupling reasoning from observations for efficient augmented language models, 2023.
- Yang et al. (2018) Yang, Z., Qi, P., Zhang, S., Bengio, Y., Cohen, W. W., Salakhutdinov, R., and Manning, C. D. Hotpotqa: A dataset for diverse, explainable multi-hop question answering. arXiv preprint arXiv:1809.09600, 2018.
- Yang et al. (2022) Yang, Z., Dong, L., Du, X., Cheng, H., Cambria, E., Liu, X., Gao, J., and Wei, F. Language models as inductive reasoners, 2022.
- Yao et al. (2022) Yao, S., Zhao, J., Yu, D., Du, N., Shafran, I., Narasimhan, K., and Cao, Y. React: Synergizing reasoning and acting in language models. arXiv preprint arXiv:2210.03629, 2022.
- Yao et al. (2023a) Yao, S., Chen, H., Yang, J., and Narasimhan, K. Webshop: Towards scalable real-world web interaction with grounded language agents, 2023a.
- Yao et al. (2023b) Yao, S., Yu, D., Zhao, J., Shafran, I., Griffiths, T. L., Cao, Y., and Narasimhan, K. Tree of Thoughts: Deliberate problem solving with large language models, 2023b.
- Yu et al. (2022) Yu, G.-I., Jeong, J. S., Kim, G.-W., Kim, S., and Chun, B.-G. Orca: A distributed serving system for {\{Transformer-Based}\} generative models. In 16th USENIX Symposium on Operating Systems Design and Implementation (OSDI 22), pp. 521–538, 2022.
- Yu et al. (2023) Yu, W., Jiang, M., Clark, P., and Sabharwal, A. Ifqa: A dataset for open-domain question answering under counterfactual presuppositions, 2023.
- Zheng et al. (2023) Zheng, H. S., Mishra, S., Chen, X., Cheng, H.-T., Chi, E. H., Le, Q. V., and Zhou, D. Take a step back: Evoking reasoning via abstraction in large language models, 2023.
- Zhou et al. (2023a) Zhou, A., Yan, K., Shlapentokh-Rothman, M., Wang, H., and Wang, Y.-X. Language agent tree search unifies reasoning acting and planning in language models, 2023a.
- Zhou et al. (2023b) Zhou, D., Schärli, N., Hou, L., Wei, J., Scales, N., Wang, X., Schuurmans, D., Cui, C., Bousquet, O., Le, Q. V., and Chi, E. H. Least-to-most prompting enables complex reasoning in large language models. In The Eleventh International Conference on Learning Representations, 2023b.

## 附录 A 准确率分析：ReAct vs. LLMCompiler

本节详细比较 ReAct 与 LLMCompiler 的准确率，重点剖析 ReAct 中高发的两类失败案例：(i) **过早停止**；(ii) **重复函数调用**。我们进一步说明：这些失败虽拉低 ReAct 的准确率，但可被 LLMCompiler 有效解决，从而带来准确率提升。我们分析两个具体场景：

- **Movie Recommendation（GPT）**：ReAct 经常过早停止，准确率显著低于 LLMCompiler（表 1 中 **68.60 vs. 77.13**）；
- **HotpotQA（LLaMA-2 70B）**：ReAct 的重复函数调用导致准确率明显下降（表 1 中 **70.00 vs. 77.80**）。

> **图 A.1 |** 在 Movie Recommendation 基准上运行 ReAct（左）、加入专用 prompt 以避免过早停止的 ReAct（中，对应表 1 的 ReAct†）、以及 LLMCompiler（右）时**函数调用次数的分布**。LLMCompiler（右）总是完成全部 8 部电影的搜索；而 ReAct（左）经常提前退出——**约 85%** 的样本提前停止。虽然专用 prompt 把 ReAct 的直方图推向更多函数调用（中），但仍不足以保证对全部电影的完整搜索。实验使用 gpt-3.5-turbo。

> **图 A.2 |** 按 ReAct 中函数调用次数分组的样本，其 Movie Recommendation 准确率（分别在 ReAct 与 LLMCompiler 上测得）。图中显示：在 ReAct 中，函数调用次数越少准确率越低，说明**过早退出导致准确率下降**；反之，同样这些样本若用 LLMCompiler 处理（保证决策前完成全部 8 部电影的搜索），则取得更高且更稳定的准确率。实验使用 gpt-3.5-turbo，结果为 3 次运行的平均。

**ReAct 的过早停止。** ReAct 频繁出现过早停止：过早停止函数调用，基于**不完整的信息**做决策。Movie Recommendation 基准中的典型例子是：ReAct 往往还没搜满所需的 8 部电影就给出最终答案。图 A.1（左）展示了 ReAct（GPT）在该基准上函数调用次数的分布——约 **85%** 的样本出现提前停止，未完成全部 8 部电影搜索就做决策；这与 LLMCompiler（右）形成对比，后者几乎全部样本（**99%**）完成 8 部电影的完整搜索。虽然给 ReAct 加入专用 prompt 可把分布推向更多函数调用（图 A.1 中），并带来准确率从 68.60 到 72.47 的提升（表 1 的 ReAct†），但这仍是不完美的解决方案。

为进一步评估过早停止对准确率的负面影响，我们先按 ReAct 的函数调用次数对 Movie Recommendation 样本分组，再用 LLMCompiler 评测这些分组（确保 8 部电影全部搜索完成）。图 A.2 表明：ReAct 中函数调用越少，平均准确率越低（绿线）；反之，若这些样本经 LLMCompiler 处理（完成全部 8 部电影搜索），则始终取得更高准确率（紫线）。这不仅说明 ReAct 难以避免提前退出（提示工程也无法完全解决），而且**停止得越早、准确率下降越多**，共同造成了表 1 中的整体准确率下滑。相比之下，LLMCompiler 有效解决了该问题。

> **图 A.3 |** 在 HotpotQA 基准上运行 ReAct（左）与 LLMCompiler（右）时**函数调用次数的分布**。LLMCompiler（右）总在 2 次函数调用内完成任务——这符合预期，因为 HotpotQA 呈现 2 路可并行模式；而 ReAct（左）显示约 **10%** 的样本出现重复（**>4 次**）函数调用，导致框架行为发散。实验使用 LLaMA-2 70B。

> **图 A.4 |** 按 ReAct 中函数调用次数分组的样本，其 HotpotQA 准确率（分别在 ReAct 与 LLMCompiler 上测得）。图中显示：在 ReAct 中，重复调用达到或超过 4 次会因其无限循环与发散行为导致准确率大幅下降（低于 10%）；而当同样这些样本用 LLMCompiler 处理（每个样本只做两次搜索）时，准确率可达约 **50%**。实验使用 LLaMA-2 70B。

**ReAct 的重复函数调用。** ReAct 的另一个常见失败是倾向重复调用函数，常常导致无限循环或超上下文长度。该问题在 HotpotQA 上尤其明显：当 Wikipedia 搜索返回的实体信息不足时，ReAct 会反复调用同一个函数。尽管 HotpotQA 本质上是 2 路可并行的（图 A.3），我们观察到约 **10%** 的样本在 ReAct 中需要超过 4 次函数调用，通常表现为无限循环或行为发散；而 LLMCompiler 对大多数样本只执行两次函数调用。

为展示重复调用对整体准确率的影响，我们做了与前一个案例类似的准确率分析。图 A.4 按 ReAct 的函数调用次数对 HotpotQA 样本分组，再比较其在 ReAct 与 LLMCompiler 上的准确率。分析表明：在 ReAct 中发起 2 次函数调用的样本，在 LLMCompiler 中保持相同准确率；但在 ReAct 中调用超过 4 次的样本（常导致发散行为）准确率不足 **10%**，而这些样本改用 LLMCompiler 处理后可绕过重复调用，取得约 **50%** 的准确率。值得一提的是，存在少量 ReAct 调用 3 次的样本：多出的那次搜索在首次搜索失败时可用**替代实体名重试**，从而比 LLMCompiler 准确率更高——这体现了 ReAct 潜在的自适应优势，但这类样本占比不足 **3%**。

---

## 附录 B LLMCompiler 的失败案例分析

本节对 LLMCompiler 在 ParallelQA 基准上的失败案例做定性分析，失败可大致归因于**规划器**、**执行器**或**最终输出过程**。最终输出过程的失败指：LLM 未能利用工具执行收集到的观测（已被并入上下文）向用户给出正确答案。在表 1 报告的 LLMCompiler 全部失败（**10.6%**，36 个样本）中，规划器、执行器与最终输出过程分别贡献 **8%**、**64%**、**28%**。

**规划器的 8%** 是 LLMCompiler 独有的失败：例如规划器错误映射输入输出，把错误的标识符作为后续任务的输入，从而形成错误的 DAG。不过在工具定义与上下文示例充分时，规划器错误会显著减少（整个评测中总共只有 **3** 例），这也凸显了 LLM 把问题分解为复杂多任务依赖的能力。

其余 **92%** 的失败归因于执行器与最终输出过程。执行器占多数（**64%**），常见问题如数学工具选错属性或单位换算处理不当；最终输出过程（**28%**）的错误包括从收集到的观测得出错误结论，例如没能从数据中挑出最小属性。需要指出，这些问题并非 LLMCompiler 独有，ReAct 中同样会出现；不过 LLMCompiler 在这些方面的失败通常略少于 ReAct，因为它只向每个工具提供**相关的上下文**，有助于更准确地抽取信息。我们认为，优化 agent scratchpad 的**结构**（而不只是简单追加观测）有望进一步减少最终输出过程的失败。

---

## 附录 C 相关工作（续）

此处接续第 2 节的相关工作。

### C.1 工具增强型 LLM

一个代表性工作是 **Toolformer**（Schick et al., 2023）：它生成自定义的 LLM 输出，由 LLM 自行决定调用函数的输入以及把结果插回何处。这一思路启发了多种工具调用框架（Liang et al., 2023; Shen et al., 2023）。**ReAct**（Yao et al., 2022）提出让 LLM 通过推理与动作生成与外部环境交互以提升表现；**Gorilla**（Patil et al., 2023）给出为函数调用微调的 LLM；**ToolLLM**（Qin et al., 2023）与 **RestGPT**（Song et al., 2023）把 LLM 扩展到支持真实世界 API。此外，OpenAI（OpenAI, 2023）发布了其函数调用能力，使 LLM 可返回格式化 JSON 以供执行。

> **表 C.1 |** 规划器**使用/不使用流式**时的延迟对比。流式在各基准上都带来稳定的延迟改善，因为它使任务取指单元能在规划器每产出一个任务时就立即开始执行。该影响在 ParallelQA 上尤为明显——那里工具执行时间足够长，可有效掩盖规划器的执行时间。

| 基准 | 无流式 (s) | 有流式 (s) | 延迟加速 |
| --- | --- | --- | --- |
| HotpotQA | 4.00 | 3.95 | 1.01× |
| Movie Rec. | 5.64 | 5.47 | 1.03× |
| ParallelQA | 21.72 | 16.69 | 1.30× |

---

## 附录 D 实验细节

我们的实验评估两种常见场景：(1) 使用**基于 API 的闭源模型**；(2) 使用**开源模型 + 自建服务框架**。闭源模型使用 OpenAI 的 GPT 系列：HotpotQA 与 Movie Recommendation 用 **gpt-3.5-turbo（1106 版本）**，ParallelQA 用 **gpt-4-turbo（1106 版本）**，Game of 24 用 **gpt-4（0613 版本）**。HotpotQA、Movie Recommendation、ParallelQA 的实验均在 2023 年 11 月（1106 版本发布后）完成；Game of 24 的实验于 2023 年 9–10 月历时两个月完成。开源模型使用 **LLaMA-2**（Touvron et al., 2023），以 **vLLM**（Kwon et al., 2023）框架部署在 2 张 A100-80GB 上。

除 Game of 24 的 `thought_proposer` 与 `state_evaluator` 使用 **temperature 0.7** 外，所有运行均使用 **temperature 0**。由于 OpenAI 即使在 temperature 0 时输出仍有随机性，我们各运行 3 次并报告平均准确率。在 ReAct、OpenAI 并行函数调用与 LLMCompiler 之间，我们分别对 HotpotQA、Movie Recommendation、ParallelQA 采用 **3-shot、1-shot、5-shot** 学习；不同方法使用**相同**的示例以保证公平比较。Game of 24 的规划器使用 **2** 个上下文示例。除第 5.1 节中额外加入 ReAct 专用 prompt 的 ReAct† 之外，各方法使用相同的指令 prompt。WebShop 实验使用 **gpt-4-0613（8k 上下文）** 与 **gpt-3.5-turbo（16k 上下文）**。

---

## 附录 E 分析

### E.1 并行加速建模

尽管 LLMCompiler 在多种负载上都有可观延迟收益，但它**并未**在 N 路并行负载上取得 N× 的加速。这主要源于 LLMCompiler **规划器**与**最终作答**过程的开销无法并行化。在 Movie Recommendation 实验中，规划器与作答过程的平均开销分别为 **1.88 s** 与 **1.62 s**，两者合计已超过表 1 中 LLMCompiler 整体延迟的一半。另一处开销来自并行任务汇聚时的**拖尾效应（straggler effect）**：我们观察到最慢一次搜索的平均延迟为 **1.13 s**，几乎是所有任务平均延迟 **0.61 s** 的 2 倍。

下面我们给出 ReAct、LLMCompiler 以及带流式的 LLMCompiler 的解析延迟模型，并分析可达的延迟加速。

本节聚焦**完全可并行**负载（图 3(a) 模式），以便更清晰地理解各组件对潜在延迟收益的影响。精确的延迟分析考虑图 2 中的三个关键组件：规划器、任务取指单元与执行器。假设规划器生成 N 个待执行任务；定义 $P_i$ 为规划器对第 $i$ 个原子任务的输出，每个 $P_i$ 是特定原子任务 $E_i$ 的蓝图；$E_i$ 的执行即一次使用相应工具的函数调用。各单元的延迟函数定义如下：规划器的延迟记为 $T_P(P_i)$，表示生成计划 $P_i$ 的时间；执行器的延迟记为 $T_E(E_i)$，表示完成任务 $E_i$ 的时间。我们忽略任务构造单元（Task Formulation Unit）的延迟，因为本节中它可忽略不计。

**ReAct 的延迟模型。** ReAct 的总延迟可写为：

$$T^{R}=\sum_{i=1}^{N}\left(T_{P}^{R}(P_{i})+T_{E}(E_{i})\right) \tag{1}$$

其中上标 $R$ 指 ReAct。在 ReAct agent 系统中，流程通常是：先生成想法（thought），再生成动作（action），并通过与工具关联的函数调用获得观测；想法与动作的生成合起来视为生成 $P_i$ 的一部分。注意：规划器延迟带上了上标（表示 ReAct），而执行器延迟没有——因为 ReAct 与 LLMCompiler 的函数调用与工具执行完全相同。

**LLMCompiler 的延迟模型。** 在 LLMCompiler 中，所有可并行任务并发处理，总延迟由其中最慢的任务决定：

$$T^{C}=\sum_{i=1}^{N}T_{P}^{C}(P_{i})+\max_{k\in\{1,\dots,N\}}T_{E}(E_{k}) \tag{2}$$

即所有规划时间之和，加上最长任务的执行时间，体现系统对并行执行的侧重。

**带流式规划的延迟模型。** 若规划器对依赖图采用流式输出，模型修正为：

$$T^{SC}=\sum_{i=1}^{N}T_{P}^{C}(P_{i})+T_{E}(E_{N}) \tag{3}$$

注意 $T^{SC}\le T^{C}$，说明流式机制能更高效地处理任务依赖，可能进一步降低总延迟。

**加速比。** 相对 ReAct 的加速比 $\gamma$ 定义为：

$$\gamma=\frac{T^{R}}{T^{C}}=\frac{\sum_{i=1}^{N}\left(T_{P}^{R}(P_{i})+T_{E}(E_{i})\right)}{\sum_{i=1}^{N}T_{P}^{C}(P_{i})+\max_{k\in\{1,\dots,N\}}T_{E}(E_{k})} \tag{4}$$

**上界。** 若假设执行器延迟 $T_E(E_i)$ 远大于规划延迟 $T_P(P_i)$，且各任务执行延迟相同，则上界为：

$$\gamma_{\text{max}}\approx\frac{\sum_{i=1}^{N}T_{E}(E_{i})}{\max_{k\in\{1,\dots,N\}}T_{E}(E_{k})}=N \tag{5}$$

即理论最大加速比等于任务数 $N$。

**下界。** 当规划延迟占主导时加速比最低。由于 ReAct 与 LLMCompiler 的规划延迟大致相当，最小加速比近似为：

$$\gamma_{\text{min}}\approx\frac{\sum_{i=1}^{N}T_{P}^{R}(P_{i})}{\sum_{i=1}^{N}T_{P}^{C}(P_{i})}\approx 1 \tag{6}$$

由此可得出结论：要用 LLMCompiler 取得显著延迟收益，关键在于 (i) **降低规划器开销**；(ii) **尽量减少拖尾（straggler）**。

### E.2 延迟与可并行任务数的关系

> **图 E.5 |** ParallelQA 基准上按"最大可并行任务数"分组的延迟。

图 E.5 给出了 ParallelQA 上更细粒度的延迟分解，展示端到端延迟作为**并行任务数**的函数。这在 HPC 中常被称为**弱扩展性（weak-scaling）**——理想行为是任务数增加时延迟保持恒定。可以看到，ReAct 的延迟随任务数**成比例增长**（符合预期，因为它是串行执行）；而 LLMCompiler 的延迟增长速率小得多，因为它能尽可能并行执行多个函数调用。LLMCompiler 端到端延迟仍有轻微上升，原因是**规划器开销**：它必须先产出计划，且该过程无法并行。附录 E.1 对此有进一步分析。

### E.3 HotpotQA bridge 基准的补充实验

第 5.1 节的主实验使用 HotpotQA 的 **comparison** 基准，以展示 LLMCompiler 高效执行 2 路可并行负载的能力。该基准的另一部分称为 **bridge**，涉及串行任务，例如"在电影《Kiss and Tell》中饰演 Corliss Archer 的那位女士担任过什么政府职位？"。LLMCompiler 并不局限于 comparison 基准：凭借**重规划**能力，它同样适用于 bridge——先搜索在《Kiss and Tell》中饰演 Corliss Archer 的女士，再通过重规划搜索该女士担任的政府职位。

与 comparison 基准的实验类似，表 E.2 在 bridge 基准上比较 LLMCompiler 与 ReAct、以及加入"避免重复调用与过早停止"额外 prompt 的 ReAct（ReAct†）。我们分别观察到 **4%** 与 **3%** 的准确率提升，这归因于 ReAct 的重复函数调用——即便加入额外 prompt（ReAct†），仍有 **5%** 的样本因此失败。此外，重复函数调用也解释了 ReAct 相比我们略高的延迟。该实验说明 LLMCompiler 对**并行与串行**负载都能做到高效且准确的函数调用。

> **表 E.2 |** LLMCompiler 与 ReAct 在 HotpotQA **bridge** 基准上的准确率与延迟对比。ReAct† 表示加入额外 prompt（抑制循环与过早停止）的 ReAct，与表 1 类似。

| 方法 | 准确率 (%) | 延迟 (s) |
| --- | --- | --- |
| ReAct | 22.7 | 7.07 |
| ReAct† | 23.1 | 6.42 |
| **LLMCompiler** | **26.3** | **4.70** |

---

## 附录 F 关于相关工作的补充讨论

TPTU（Ruan et al., 2023a）、HuggingGPT（Shen et al., 2023）与 ViperGPT（Surís et al., 2023）提出了端到端的 plan-and-solve 框架。本节从多个角度讨论 LLMCompiler 与其他框架的区别，包括 (i) 规划与重规划；(ii) 并行执行；(iii) 覆盖更广的问题域。汇总见表 E.3。

> **表 E.3 |** LLMCompiler 与 ReAct（Yao et al., 2022）、TPTU（SA = Sequential Agent，OA = One-step Agent）（Ruan et al., 2023a）、ViperGPT（Surís et al., 2023）、HuggingGPT（Shen et al., 2023）的定性对比。“O”表示支持，“X”表示不支持，“–”表示该维度不适用/未讨论。

| 方法 | 规划 (Planning) | 重规划 (Replanning) | 并行执行 | 适用域 |
| --- | --- | --- | --- | --- |
| ReAct | X | – | X | 全部 |
| TPTU-SA | X | – | X | 全部 |
| TPTU-OA | O | X | X | 全部 |
| ViperGPT | O | X | X | 受限 |
| HuggingGPT | O | X | O | 受限 |
| **LLMCompiler** | **O** | **O** | **O** | **全部** |

**并行执行。** 并行执行是 LLMCompiler 的关键特性，使其能高效地调用函数并完成任务。TPTU 的 One-step Agent（TPTU-OA）虽包含规划，但**不支持并行函数调用**——它只把用户输入分解为一串函数及其参数，**不建模相互依赖**。ViperGPT 生成 Python 代码，但标准 Python 解释器本身不支持并行执行，因此若无专门的并行处理引擎，ViperGPT 自身无法并行。HuggingGPT 支持并行执行，但它**严格限定于 HuggingFace 上的模型**，难以应用到 LLMCompiler 所支持的广泛问题与领域。

**规划与重规划。** TPTU 的 Sequential Agent（TPTU-SA）是与 ReAct 类似的迭代式框架，每轮只执行一个动作。TPTU-OA、HuggingGPT 与 ViperGPT 都是基于规划的框架，会在执行前规划多个动作，但都**缺少重规划能力**。相比之下，LLMCompiler 引入**重规划机制**：当先前计划不足以向用户交付回答时，生成一组新任务。这使 LLMCompiler 能依据先验未知的中间结果调整计划，而无需引入复杂的分支逻辑，从而扩展了可解决问题的范围。

**问题域。** ViperGPT 与 HuggingGPT 分别面向通过 Python 代码生成与 HuggingFace 模型的**视觉任务**，在这些特定领域表现亮眼。相比之下，LLMCompiler 追求的是一个**通用框架**，在广泛问题上实现高效、准确的函数调用，而非局限于特定领域。

### F.1 LLMCompiler 与 TPTU 的定量比较

> **表 F.4 |** 在 HotpotQA comparison 基准（gpt-3.5-turbo）上，LLMCompiler 与 ReAct、TPTU 的准确率与延迟加速对比。ReAct† 与 TPTU-SA† 分别表示加入额外 prompt（抑制循环与过早停止）的 ReAct 与 TPTU-SA，类似表 1。

| 方法 | 准确率 (%) | 加速 |
| --- | --- | --- |
| ReAct | 61.52 | – |
| ReAct† | 62.47 | 1× |
| TPTU-SA | 34.16 | – |
| TPTU-SA† | 44.59 | 1.09× |
| TPTU-OA | 57.50 | 1.35× |
| **LLMCompiler** | **62.00** | **1.51×** |

表 F.4 额外给出 LLMCompiler 相对 TPTU-SA 与 TPTU-OA 的准确率与延迟加速。由于 TPTU 官方实现不可得，我们依据原论文给出的 prompt 实现了 TPTU-SA 与 TPTU-OA。如表所示，LLMCompiler 在延迟与准确率上都明显优于两者。相比 TPTU-SA，LLMCompiler 的准确率提升显著，原因是 TPTU 普遍存在**重复函数调用**问题；即便用更好的 prompt（TPTU-SA†）也未能完全缓解，导致约 **15%** 的样本因重复调用而失败。相比 TPTU-SA 与 TPTU-OA，LLMCompiler 还通过**并行任务执行**获得延迟收益。总体结论与其他基线（ReAct、OpenAI 并行函数调用）的主实验与分析一致。

---

## 附录 G 用户提供的 LLMCompiler 配置示例

LLMCompiler 提供简洁的接口：只需提供工具定义，以及给规划器的可选上下文示例，即可把框架裁剪到不同用例。下面给出用于配置 Movie Recommendation 与 Game of 24 基准的规划器示例 prompt——仅需寥寥数行。

### G.1 Movie Recommendation 示例 prompt

```text
Question: Find a movie similar to Mission Impossible, The Silence of the Lambs,
American Beauty, Star Wars Episode IV - A New Hope
Options:
Austin Powers International Man of Mystery
Alesha Popvich and Tugarin the Dragon
In Cold Blood
Rosetta

1. search("Mission Impossible")
2. search("The Silence of the Lambs")
3. search("American Beauty")
4. search("Star Wars Episode IV - A New Hope")
5. search("Austin Powers International Man of Mystery")
6. search("Alesha Popvich and Tugarin the Dragon")
7. search("In Cold Blood")
8. search("Rosetta")
Thought: I can answer the question now.
9. finish()
###
```

> *（译者注）注意这里 LLM 用一次规划就把 8 次独立搜索**全部并行展开**，并用 `finish()` 收尾；而 ReAct 需要 8 轮"搜索—推理"循环。这正是 3.74× 加速与 6.73× 成本下降的直接来源。*

### G.2 Game of 24 示例 prompt

```text
Question: "1 2 3 4", state_list: [""]
$1 = thought_proposer("1 2 3 4", "")
$2 = state_evaluator("1 2 3 4", "$1")
$3 = top_k_select("1 2 3 4", ["$1"], ["$2"])
$4 = finish()
###
Question: "1 2 3 4", state_list: ["1+2=3(left:3 3 4)","2-1=1(left:1 3 4)",
"3-1=2(left:2 2 4)","4-1=3(left:2 3 3)","2*1=2(left:2 3 4)"]
$1 = thought_proposer("1 2 3 4", "1+2=3(left:3 3 4)")
$2 = thought_proposer("1 2 3 4", "2-1=1(left:1 3 4)")
$3 = thought_proposer("1 2 3 4", "3-1=2(left:2 2 4)")
$4 = thought_proposer("1 2 3 4", "4-1=3(left:2 3 3)")
$5 = thought_proposer("1 2 3 4", "2*1=2(left:2 3 4)")
$6 = state_evaluator("1 2 3 4", "$1")
$7 = state_evaluator("1 2 3 4", "$2")
$8 = state_evaluator("1 2 3 4", "$3")
$9 = state_evaluator("1 2 3 4", "$4")
$10 = state_evaluator("1 2 3 4", "$5")
$11 = top_k_select("1 2 3 4", ["$1", "$2", "$3", "$4", "$5"],
                   ["$6", "$7", "$8", "$9", "$10"])
$12 = finish()
###
```

> *（译者注）第二例最能体现"编译器"味道：5 个 thought_proposer 与 5 个 state_evaluator 在 DAG 中天然按层并行——$6..$10 依赖 $1..$5，于是同一层的 5 个调用并发发出，无需像 ToT 那样逐个状态串行评估。*

---

## 附录 H 预定义的规划器 Prompt

预定义的规划器 prompt 向规划器提供具体指令：如何分解任务、如何生成依赖图，并保证相关语法格式正确。其中包含的规则例如：每个任务占一行、以数字标识符开头、用 `$` 表示中间变量。

```text
- Each action described above contains input/output types and descriptions.
- You must strictly adhere to the input and output types for each action.
- The action descriptions contain the guidelines. You MUST strictly follow those
  guidelines when you use the actions.
- Each action in the plan should strictly be one of the above types. Follow the
  Python conventions for each action.
- Each action MUST have a unique ID, which is strictly increasing.
- Inputs for actions can either be constants or outputs from preceding actions.
  In the latter case, use the format $id to denote the ID of the previous action
  whose output will be the input.
- Ensure the plan maximizes parallelizability.
- Only use the provided action types. If a query cannot be addressed using these,
  invoke the finish action for the next steps.
- Never explain the plan with comments (e.g. #).
- Never introduce new actions other than the ones provided.
```

> *（译者注）上述规则中，**"unique ID 严格递增"** 保证了 DAG 的拓扑序可直接用 ID 表达依赖（`$id` 只能引用更小的 ID），**"Ensure the plan maximizes parallelizability"** 则是让 LLM 主动把无依赖调用摊平到同一层——这是整套方案能自动发现并行性的关键一句话。*

除用户提供的函数外，规划器还内置一个**硬编码的 `finish` 函数**。规划器在两种情况下使用它：一是当前计划已足以回应用户查询；二是必须先执行当前计划、在规划阶段无法继续推进（即认为需要重规划）。当规划器输出 `finish` 时，其规划生成即停止。规划器在规划中使用 `finish` 的示例见附录 G。`finish` 的定义如下，它与其他用户提供函数的定义一并进入规划器 prompt：

```text
finish():
- Collects and combines results from prior actions.
- A LLM agent is called upon invoking join to either finalize the user query
  or wait until the plans are executed.
- join should always be the last action in the plan, and will be called in two
  scenarios:
  (a) if the answer can be determined by gathering the outputs from tasks to
      generate the final response.
  (b) if the answer cannot be determined in the planning phase before you
      execute the plans.
```

---

## 附录 I ParallelQA 基准的构建

受 IfQA 基准（Yu et al., 2023）启发，我们自建的 **ParallelQA** 含 **113** 个样本，设计思路是用关于不同实体事实细节的**数学问题**来提问，因此需要**搜索与数学运算**混合、且两者以多种方式相互依赖。例如："如果把德克萨斯与佛罗里达合并成一个州、加州与密歇根也合并成一个州，这两个新州中哪个人口密度更大？"——这需要四次并行搜索任务，随后是依赖搜索结果的数学任务（这些数学任务之间也可并行）。

该基准的主要目标是**量化框架把输入分解为多个任务并推导答案的能力**。因此我们精心挑选了跨多个领域的 **56** 个不同实体，其属性均可通过 Wikipedia 搜索获取。通过尽量减少工具执行（即 Wikipedia 搜索）失败，我们希望基准能有效评估各框架把问题分解为多任务、规划执行、并基于观测推导最终答案的能力。此外，为覆盖多样的执行模式，我们构造了多种依赖模式：在搜索到问题中实体的额外信息后，执行一元与二元数学运算。我们还设计了不同**最大可并行任务数（2 到 5）**的问题，并加入了数量不等的并行函数调用之间的**汇聚（join）**以提升复杂度——例如图 3(b) 与 (c) 分别有 **2** 个与 **3** 个 join。基准共 **113** 个样本，由 GPT-4 按上述标准生成，随后由**人工标注**。

---

## 附录 J Game of 24 与 Tree-of-Thoughts 细节

Game of 24 是一个数学推理游戏：玩家需用给定的四个数字，借助加、减、乘、除四种基本运算得到 24，且每个给定数字必须**恰好使用一次**。例如给定 2、4、4、7，一个可行解是 $4\times(7-4)\times 2=24$。这对 LLM 是一个非平凡的推理基准——即便 GPT-4 配合 chain-of-thought 提示，成功率也仅 **4%**（Yao et al., 2023b）。

在 ToT 中，问题分多步求解。每一步，被称为 **thought proposer** 的 LLM 生成若干"想法"；每个想法是一个部分解，由两个数字及它们之间的一次算术运算构成。这些想法随后送入 **state evaluator**，由其为每个想法打标签：**sure（确定）**、**likely（可能）**、**impossible（不可能）**，表示"该想法结果与剩余数字再运算后能得出 24"的可能性。只有"有可能得出 24"的想法进入下一步。该过程见图 J.6。

> **图 J.6 |** Game of 24 中 Tree-of-Thoughts（ToT）的可视化。每个节点表示一个候选提案：从根节点开始，由 thought proposer 施加单次运算向外分支；后续状态由 state evaluator 评估其达到目标 24 的潜力。ToT 按评估值保留 **top-5** 状态。

---

## 附录 K WebShop 实验细节

### K.1 WebShop 环境

WebShop 环境模拟一个在线购物平台。任务要求 agent 找到与给定指令最匹配的商品。例如指令为"我在找一张黑色的、价格低于 140.00 美元的 queen 尺寸床"，agent 的任务就是精确定位同时满足"queen 尺寸""黑色""价格低于 140.00 美元"的床。每件商品都有对应的**奖励**，依据价格、商品规格以及商品页中的其他细节衡量其与指令的匹配程度。评测指标为**成功率**（所选商品满足全部要求的回合占比）与**平均得分**（各回合奖励的均值）。

### K.2 基线方法

除 ReAct 外，我们使用 **LASER**（Ma et al., 2023）与 **LATS**（Zhou et al., 2023a）作为基线。LASER 通过**状态探索**求解任务：在 WebShop 中，可能的环境页面被编码为不同状态（如搜索页、商品页、商品详情子页），动作用于在这些状态间转移（执行搜索查询、选择商品、查看详情、翻到下一页搜索结果等），于是 WebShop 探索被化约为给定状态空间图上的**搜索问题**。

LATS 使用**蒙特卡洛树搜索**的变体：通过构建决策树来规划动作，依据成功可能性评估潜在动作，并在探索与利用之间平衡地选择动作；agent 再根据环境反馈调整策略，从成功与失败中学习以改进决策。这种迭代方式使 LATS 能应对复杂的在线购物任务，但**穷尽的树搜索**使其速度慢得多。

---

## 术语与背景速览（译者整理）

| 术语 | 含义 |
| --- | --- |
| Function calling（函数调用 / 工具调用） | LLM 输出结构化调用请求，由外部执行函数并把结果回填上下文 |
| Embarrassingly parallel（完全可并行） | 任务之间无依赖、可全部同时执行的最简并行模式 |
| DAG（有向无环图） | 规划器产出的任务依赖图；节点是任务，边是数据依赖 |
| Planner（函数调用规划器） | 把自然语言查询编译成带 `$id` 占位变量的任务 DAG |
| Task Fetching Unit（任务取指单元） | 类比 CPU 取指单元；按贪心策略把"依赖已满足"的任务派发出去，并用真实输出替换 `$id` |
| Executor（执行器） | 异步并发执行任务，每个任务有独立内存存放中间结果 |
| Placeholder variable（占位变量） | 形如 `$1`、`$2`，代表前序任务的输出，由取指单元替换 |
| Streamed Planner（流式规划器） | 任务一生成就流式下发，形成"规划—执行"流水线，掩盖规划延迟 |
| Replanning（重规划） | 执行结果回传规划器触发新一轮规划，用于无法静态确定的分支/迭代 |
| ReAct | 串行"Thought → Action → Observation"循环的经典 agent 范式，本文主基线 |
| ReAct† | 加入专用 prompt 以抑制循环与过早停止的 ReAct，用于公平的延迟对比 |
| Early stopping（过早停止） | ReAct 基于不完整观测就下结论，本文统计到约 85% 的样本有此问题 |
| Repetitive function calls（重复函数调用） | ReAct 反复调用同一函数导致无限循环或超上下文 |
| Straggler（拖尾任务） | 并行任务汇聚时最慢的那个任务决定整体延迟（本文实测约 2× 平均值） |
| TPTU / HuggingGPT / ViperGPT | 其他 plan-and-solve 框架；对比见附录 F（表 E.3） |
| ParallelQA | 本文自建基准，113 题，含 2–5 路并行与 1–3 个 join 的依赖模式 |
| WebShop | 模拟电商环境，用自然语言指令衡量 agent 的商品检索与决策能力 |
| ToT / Game of 24 | Tree-of-Thoughts 与 24 点游戏；用于验证重规划能力 |

**核心数字速记**

- 整体收益（vs. ReAct）：延迟最高 **3.7×**、成本最高 **6.7×**、准确率最高 **~9%**
- HotpotQA（2 路并行，gpt-3.5-turbo）：**1.80×** 加速、**3.37×** 成本下降、准确率 62.00 vs 62.47（ReAct†）/ 61.52（原始 ReAct）
- Movie Recommendation（8 路并行）：**3.74×** 加速、**6.73×** 成本下降、准确率 77.13 vs 72.47
- ParallelQA（复杂依赖）：**2.15×**（GPT）/ **2.27×**（LLaMA-2）加速、**4.65×** 成本下降、LLaMA 上准确率 **+9%**
- Game of 24（动态重规划）：**2.89×**（GPT）/ **2.09×**（LLaMA）加速，成功率 75.33 vs 74.00
  - *（译者注）原文正文写 LLaMA 为 2.01×，表 1 为 2.09×，此处以表 1 为准。*
- WebShop（决策任务）：成功率 **48.2%** vs ReAct **19.8%**（gpt-3.5-turbo，+28.4pp）；相比 LATS **101.7×**、相比 LASER **2.69×** 加速
- 流式规划收益：HotpotQA 1.01×、Movie Rec. 1.03×、ParallelQA **1.30×**
- 失败归因（ParallelQA 的 10.6% 失败样本中）：规划器 **8%**、执行器 **64%**、最终输出 **28%**；规划器全程仅 **3** 例错误
- 理论加速上界 = 任务数 N；实测未达 N× 主因：规划器 + 作答开销（Movie Rec. 合计 3.5 s）、拖尾任务（1.13 s vs 平均 0.61 s）
