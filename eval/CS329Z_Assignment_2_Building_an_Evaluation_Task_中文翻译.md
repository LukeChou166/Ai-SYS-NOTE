# CS329Z 作业二：从零构建一个评测任务

> **原文**: CS329Z Assignment 2: Building an Evaluation Task (Version 26.1.0)
> **作者**: CS329Z 课程团队（John Yang 设计并实现）
> **机构**: Stanford CS329Z *Engineering AI Agents*，2026 年秋季学期
> **链接**: https://github.com/cs329z/assignment2-evals/blob/main/cs329z_assignment2_evaluation.pdf
> **译者说明**: 本文为完整全文翻译。作业体例（Problem 编号、Deliverable 交付物清单、命令行、目录结构）均按原样保留；图注按图意译出并补充说明图内元素；带 *（译者注）* 标记的段落为译者补充的背景说明，非原文内容。

---

## 1 概述

在这次作业里，你要从零开始为一个语言模型（LM）智能体（agent）构建一个评测任务。到结束时，你会拥有一个完整的、经过仔细校验的评测任务。你需要自己决定要测量哪种能力，设计一个能够测量它的任务，并分析模型在该任务上的表现。

### 1.1 简要回顾

回顾课堂内容：基于 LM 的智能体，其基准（benchmark）的核心原则与组成部件如下。

> **图 1 | 智能体评测的核心组件：智能体接收问题描述，在环境中行动，其工作成果通过验证被评分。**
>
> 图中从左到右共四个环节：
> 1. **问题陈述（Problem Statement）**——一个开发者正在排查命令 `ffmpeg -i input.mp4 -vf scale=1280:720 output.mp4` 的输出分辨率异常（期望 1280×720，实际却变成 1920×1080），附带一句"能帮我看下吗？也许可以看看 parse_args.py"。也就是说，问题描述刻画了"必须完成什么"。
> 2. **智能体（Agent）**——"ReAct！"，表示智能体读入问题陈述后进入推理-行动循环。
> 3. **环境（Environment）**——终端（Terminal）、浏览器（Browser）、模拟器（Simulator）、真实世界（Real World）。智能体在此"干活"：发出动作、接收观测。
> 4. **提交与验证（Submission / Verification）**——单元测试或端到端测试（Unit/E2E Tests）、评分细则（Rubrics）、偏好（Preference），最终输出"通过 / 失败"（Pass / Fail）。

**基准（benchmark）** 指一组用于测试智能体能力或行为的任务。对每一个任务而言：

1. 在初始提示中把**问题陈述**交给智能体。
2. 智能体随后在**环境**中着手解决问题（例如通过终端修复代码库、通过浏览器完成网页任务）。智能体会经历多个轮次，对环境发出动作并接收观测。
3. 为了测量表现，我们把智能体的**提交**（通常是智能体结束时的环境状态）用一至多种**验证**形式来评分。验证形式有多种，即代码单元测试或评分细则。

这三个组件共同定义了：智能体必须做什么、它能与什么交互、以及它的表现如何被测量。

### 1.2 基准为什么重要？

过去十年自然语言处理（NLP）与人工智能（AI）取得进展的一个关键驱动力，是研究社区对**共享基准**的采用。

基准支撑起一个良性循环。一个提法良好的基准确立了衡量进展的共同标尺；后续工作随后提升现有方法在该基准上的表现（通常称为"爬坡"/hillclimbing），而更重要的是，这些工作识别出一组新的局限与空白，反过来催生下一个基准。

### 1.3 你将做什么

你将在基准的**构念（formulation）、构建（construction）、评测（evaluation）** 全生命周期中获得动手经验，共分四部分：

- **基础（5%）**：在引导下深入 Harbor 框架——一个任务如何被表示、一次智能体尝试如何被执行、验证如何被隔离、以及一次评测运行会产生哪些产物。
- **运行任务（10%）**：在若干现有基准（如 SWE-bench Verified、Terminal-Bench 2.1）的任务实例上运行模型。过程中你会看到任务在代码里如何被表示，以及这种表示如何随构念方式的不同而变化。
- **造一个任务（70%）**：你自己编写一个任务，并用 mini-SWE-agent 搭配 Gemini 3.8 Flash 做五次独立尝试。你要设计任务的问题陈述、环境和验证方式，然后分析是什么让这个任务有效、真实、可复现且有挑战性。
- **同行评审（15%）**：任务提交截止后，你从只读的课程画廊下载三位同学的任务，用提供的模板逐一评审，并把完成的评审提交到单独的 Gradescope 作业。

本次作业满分 100 分。

### 1.4 提交

Gradescope 上有两处提交：

- **任务提交（截止：11 月 23 日周一）**：你的仓库，通过 Gradescope 的 GitHub 集成上传。仓库中包含你的任务（含五次模型运行的产物与 report.pdf）以及你在第 2–3 节的书面回答；我们按该快照评分。
- **同行评审（截止：12 月 2 日周三）**：peer_review.pdf。

**关于 AI 辅助**

> 在做这份作业时，你可以使用 Claude Code、Codex、Copilot 或任何其他 AI 工具。我们会通过口头问答来检查你的理解，即就你的设计与代码向你提问。你应当能够解释、并且对提交内容中的一切负责。

---

## 2 基础

本节我们会下载作业起始代码，并梳理构建一个基准的技术基础。

### 2.1 环境准备

我们用 `uv` 管理 Python 环境与依赖。克隆作业仓库，然后任何命令都用 `uv run` 执行（首次使用时它会解析环境）：

```bash
git clone https://github.com/cs329z/assignment2-harness.git
cd assignment2-harness
```

把工作推送到**你自己的私有** GitHub 仓库（不要建公开 fork），这样同学在截止前看不到你的任务；你将提交那个仓库。

接下来，你需要用 API 额度来在任务上运行模型。复制模板并填入 `GEMINI_API_KEY` 或 `OPENROUTER_API_KEY`：

```bash
cp .env.example .env
```

**任务范围（Task Scope）**

如你所知，"智能体"可以指一个谱系上的各类系统。本次作业我们主要聚焦**编程智能体（coding agents）** 的基准，因为它们在 AI 智能体评测领域中相对成熟、且有清晰的技术脉络。

我们将使用 **mini-SWE-agent** 脚手架（scaffold）。mini-SWE-agent 是一个简单轻量的编程智能体，只向模型暴露一个 bash 工具。

> Harbor 在每个任务容器中安装 mini-SWE-agent 时不锁定其依赖。我们的命令通过提供的 `pinned_agent.py` 加载它，该文件会安装精确版本；你不需要修改它。

### 2.2 基本积木

让我们熟悉支撑当代 LM 基准实现与运行的基础设施：

- **Docker**：在称为容器（container）的隔离、可复现环境中运行软件。
- **Harbor 框架**：一个被广泛使用的软件包，提供定义编程智能体任务、并对 LM 进行评测所需的基础设施。

#### 2.2.1 Docker 保证可复现性

设想你在自己的 MacBook 上本地编写并评测 LM 在一个任务上的表现。测试通过、脚本能跑、一切看起来都正常。但当你把任务资产发给朋友时，他却无法复现你的结论。为什么？也许是因为操作系统不同；也许是 Python 和某些依赖的版本略有偏差；也许是你漏装了某个必要的 `.so` 库。

Docker 通过在**容器**内运行每次尝试来解决这个问题：一个受控的工作区，指定了操作系统、依赖与初始文件。

> **问题 1（docker-setup）：搭建 Docker**
>
> 按你操作系统的安装说明操作。
> - Mac 安装说明
> - Linux 安装说明
>
> 安装完成后，花些时间确保你理解 Docker 以及 Dockerfile、镜像（image）、容器（container）等术语。
>
> **交付物**：拉取 `ubuntu:22.04` 镜像，启动一个交互式容器，并在容器内部确认其操作系统。把命令与输出记录到 `writeup/2_basics/responses.md`。

#### 2.2.2 Harbor 定义评测原语

可复现的容器只解决了运行评测的一部分问题。一个基准还需要在**打包任务、调用智能体、验证工作、记录结果**上保持一致的约定。Harbor 提供了这些约定，并编排整个评测生命周期。它为图 1 中的问题陈述、环境、验证三个组件提供了标准的**磁盘表示**。一旦任务以这种结构定义好，Harbor 就提供额外的接线：启动一次智能体尝试、运行对应的验证器、并保存得到的奖励（reward）与产物。

在本次作业中：**Harbor 编排评测，Docker 提供任务环境，mini-SWE-agent 是被评测的智能体脚手架。**

> **问题 2（harbor-setup）：搭建 Harbor**
>
> 在作业仓库中，安装锁定版本的依赖并查看 Harbor 的命令行接口：
>
> ```bash
> uv sync
> uv run harbor --version
> uv run harbor --help
> docker version
> ```
>
> 此时不要运行评测。从帮助输出中，找出用于**启动评测作业**和**浏览已完成结果**的命令。
>
> **交付物**：确认 `harbor --version` 可运行、且 Docker 客户端能连上 Docker 服务端。记录你的输出与答案到 `writeup/2_basics/responses.md`。

---

## 3 运行示例任务

本节你将运行并研究三个任务，它们有效地勾画出 2021 年至今编程评测的演进脉络。运行每个任务时，花时间研究该任务、理解其组件，并观察智能体在任务上的表现。

### 3.1 任务 1：HumanEval

看看 `tasks/humaneval-0/` 文件夹，它包含一个 HumanEval 任务的本地移植版本。回想图 1 中任务的关键组件，你会看到这些关键概念组件映射到了该文件夹中的具体资产：

```
tasks/humaneval-0/
├── task.toml            # 元数据与执行设置
├── instruction.md       # 展示给智能体的问题陈述
├── environment/
│   ├── Dockerfile       # 构建智能体的起始环境
│   └── humaneval_0.py   # 初始任务资产
├── tests/
│   ├── test.sh          # 验证器入口
│   └── test_outputs.py  # 行为测试
└── solution/
    └── solve.sh         # oracle 使用的参考解
```

> **问题 3（run-example-task）：运行示例任务**
>
> 首先检查 `tasks/humaneval-0` 的每个组件。然后把它的参考解跑过完整的 Harbor 评测流水线：
>
> ```bash
> uv run harbor run \
>   --path tasks/humaneval-0 \
>   --agent oracle \
>   --n-attempts 1 \
>   --n-concurrent 1 \
>   --job-name humaneval-oracle \
>   --jobs-dir writeup/3_run_example_tasks/artifacts \
>   --yes
> ```
>
> 运行 `uv run harbor view writeup/3_run_example_tasks/artifacts` 查看结果。找到验证器日志，确认参考解拿到了满分奖励。
>
> **交付物**：确认 `writeup/3_run_example_tasks/artifacts` 中包含该问题评测运行的结果。在 `writeup/3_run_example_tasks/responses.md` 中用 3–5 句话回答下列每个问题：
>
> - 智能体工作时哪些文件对它是可见的，哪些文件只在验证阶段才可见？这种隔离为什么重要？
> - 运行 `uv run harbor run --help`。说出除 oracle 之外的两个智能体脚手架，以及除 Docker 之外的两个环境后端。这两类各自扮演什么不同角色？
> - 追踪智能体结束后的验证过程：Harbor 执行哪个文件？私有测试来自哪里？最终奖励如何回传给 Harbor？

你刚刚跑通了你的第一个编程智能体任务！这是 HumanEval 中的一个任务实例。HumanEval 由 OpenAI 的 Mark Chen 及同事在提出 Codex 时一并引入，它帮助确立了第一个被广泛采用的、专门围绕 LM 写代码能力设计的评测 [1]。该基准成为一脉编程评测的起点，后续这些评测逐步向更难、更真实的软件工程工作扩展。

> **图 2 | 一个 HumanEval 问题示例。** 提供给模型的提示以白色背景显示，模型生成的成功补全以黄色背景显示。
>
> *（译者注）HumanEval 的典型形式是"函数签名 + 文档字符串"，模型需补全函数体，随后用若干组单元测试断言（`assert`）检验正确性。图中黄色部分即模型补全，需通过 `tests/` 中的断言。*

现在用模型跑一下，看看它的表现。本节后续的模型运行，我们统一选用 **Gemini 3.8 Flash**。根据你使用的 key，请使用对应的模型名：

- Gemini API：`gemini/gemini-3.8-flash`
- OpenRouter：`openrouter/google/gemini-3.8-flash`

> **问题 4（run-example-agent）：运行一个真实的智能体**
>
> 现在用 mini-SWE-agent 运行该任务。
>
> ```bash
> uv run harbor run \
>   --path tasks/humaneval-0 \
>   --agent-import-path pinned_agent:MiniSweAgent \
>   --model <MODEL> \
>   --env-file .env \
>   --n-attempts 1 \
>   --n-concurrent 1 \
>   --job-name humaneval-agent \
>   --jobs-dir writeup/3_run_example_tasks/artifacts \
>   --yes
> ```
>
> 用 `uv run harbor view writeup/3_run_example_tasks/artifacts` 检查轨迹（trajectory）、验证器日志与奖励。
>
> **交付物**：确认 `writeup/3_run_example_tasks/artifacts` 中包含 `humaneval-agent` 作业。在 `writeup/3_run_example_tasks/responses.md` 中用 3–5 句话回答：
>
> - 概述智能体的做法。它用了哪些文件和命令？
> - 找出轨迹中一个帮助或损害了最终结果的决定，并引用相关步骤。
> - 验证器给出的奖励是否准确反映了智能体解法的质量？为什么？

### 3.2 任务 2：SWE-bench Verified

你可能已经注意到，HumanEval 任务对现在的模型来说相当容易，而且不太能代表我们如今交给编程智能体的工作。

因此，让我们更熟悉那些代表前沿模型正在持续评测的基准的任务。

我们要看的下一个任务实例来自 **SWE-bench Verified** 任务集。SWE-bench 由普林斯顿的 Carlos Jimenez、John Yang 及同事提出，要求 LM 解决真实的 GitHub issue [2]。HumanEval 聚焦于自包含的函数，而 SWE-bench 开创了**仓库级**的软件工程任务，对真实工作的保真度高得多。它的广泛采用，使"真实 issue 解决"成为衡量编程智能体进展的核心指标。

要跟着做，先从 Harbor 下载该任务：

> **图 3 | SWE-bench 通过把 GitHub issue 与"解决了相关测试的已合并 PR 解法"对应起来，从真实 Python 仓库中采集任务实例。** 给定 issue 文本与代码库快照，模型生成一个补丁（patch），并针对真实测试进行评测。
>
> *（译者注）这就是 SWE-bench 的数据构造核心：只采信"issue 提到了问题 → 对应 PR 被合并 → 且该 PR 引入了能通过测试的代码改动"的三元组，从而保证任务有客观可验证的解。*

```bash
uv run harbor tasks download swe-bench/django__django-11099@2 --output-dir tasks
```

这会创建 `tasks/django__django-11099`。在运行智能体之前，读一读 `instruction.md` 中的 issue，并大致检查仓库与验证器设置。

> **Apple Silicon Mac 用户**
>
> 该镜像是 x86_64 专用。在 Apple Silicon 上，请先执行 `export DOCKER_DEFAULT_PLATFORM=linux/amd64`；此后它会在模拟下运行，因此首次构建会很慢。

> **问题 5（run-swebench-agent）：在 SWE-bench Verified 上运行智能体**
>
> 用 mini-SWE-agent 和 Gemini 3.8 Flash 运行该任务：
>
> ```bash
> uv run harbor run \
>   --path tasks/django__django-11099 \
>   --agent-import-path pinned_agent:MiniSweAgent \
>   --model <MODEL> \
>   --env-file .env \
>   --n-attempts 1 \
>   --n-concurrent 1 \
>   --job-name swebench-django-11099 \
>   --jobs-dir writeup/3_run_example_tasks/artifacts \
>   --yes
> ```
>
> 用 `uv run harbor view writeup/3_run_example_tasks/artifacts` 检查得到的轨迹、验证器日志与奖励。
>
> **交付物**：确认 `writeup/3_run_example_tasks/artifacts` 中包含 `swebench-django-11099` 作业。在 `writeup/3_run_example_tasks/responses.md` 中用 3–5 句话回答：
>
> - 概述智能体的做法。它如何定位到相关代码，做了什么改动？
> - 找出智能体利用了哪些"自包含的 HumanEval 任务中不可能存在"的仓库上下文。
> - 把这条轨迹与 HumanEval 的轨迹对比。是什么让这个任务在理解、修改或验证上更难？
> - 最终奖励是否准确反映了智能体解法的质量？用验证器日志中的证据支持你的回答。

### 3.3 任务 3：Terminal-Bench 2.1

HumanEval 和 SWE-bench 都以编写和修改代码为中心，但许多有用的计算机任务超出了软件仓库的范围。

**Terminal-Bench** 是由 Mike Merrill、Alex Shaw、Andy Konwinski、Ludwig Schmidt 牵头、多机构合作的项目，它把编程智能体评测拓展到**通过命令行界面完成的、长时程（long-horizon）端到端工作** [3]。与绑定在仓库上的 SWE-bench 不同，Terminal-Bench 涵盖一切能在终端里完成的任务，包括系统管理、安全、机器学习、数据处理与科学计算。

> **图 4 | Terminal-Bench 把智能体放进一个容器，并用隐藏测试评测由此产生的环境状态。**
>
> *（译者注）与 HumanEval 检查"函数输出"、SWE-bench 检查"补丁是否让指定测试通过"不同，Terminal-Bench 检查的是"容器最终状态是否符合预期"，因此不限定解题手段，理论上任何能达到目标状态的做法都算通过。*

本问题中，你将运行 Terminal-Bench 2.1 里的 `model-extraction-relu-logits` 任务。Harbor 把该任务标为**困难**，并估计专家大约需要八小时完成。运行模型时，我们并不期望模型一定能得到通过的解法；你的目标是研究智能体如何着手一个困难任务，以及它的轨迹捕捉到了多少进展。

首先，从 Harbor 下载该任务：

```bash
uv run harbor tasks download \
  terminal-bench/model-extraction-relu-logits@3 \
  --output-dir tasks
```

这会创建 `tasks/model-extraction-relu-logits`。阅读其 `instruction.md`，并在运行智能体前大致检查其环境与验证器。

> **问题 6（run-terminal-bench-agent）：在 Terminal-Bench 上运行智能体**
>
> 用 mini-SWE-agent 和 Gemini 3.8 Flash 运行该任务：
>
> ```bash
> uv run harbor run \
>   --path tasks/model-extraction-relu-logits \
>   --agent-import-path pinned_agent:MiniSweAgent \
>   --model <MODEL> \
>   --env-file .env \
>   --n-attempts 1 \
>   --n-concurrent 1 \
>   --job-name terminal-bench-model-extraction \
>   --jobs-dir writeup/3_run_example_tasks/artifacts \
>   --yes
> ```
>
> 用 `uv run harbor view writeup/3_run_example_tasks/artifacts` 检查得到的轨迹、验证器日志与奖励。
>
> **交付物**：确认 `writeup/3_run_example_tasks/artifacts` 中包含 `terminal-bench-model-extraction` 作业。在 `writeup/3_run_example_tasks/responses.md` 中用 3–5 句话回答：
>
> - 概述智能体的做法。它收集了哪些信息，采取了什么样的解题策略？
> - 找出智能体取得实质进展的一处，以及它方法失效或变得低效的一处。
> - 验证器除了标量奖励之外还提供了什么证据？它是否捕捉到该任务上的部分进展？
> - 开放的终端环境如何影响了智能体做规划、检验假设、以及从错误中恢复的能力？

### 3.4 分析智能体轨迹

到这一步，你应当已经拥有对应全部三个任务的智能体轨迹。

接下来，你会面对若干问题，用来锻炼你**可视化并分析轨迹中趋势**的能力——这是在基准上有效传达模型表现结论时的一项核心技能。

> **问题 7（analyze-trajectories）：分析智能体轨迹**
>
> 使用 HumanEval、SWE-bench Verified 与 Terminal-Bench 三次运行保存下来的轨迹，制作智能体行为的可视化。你可以自选可视形式并定义合理的度量，前提是每张图都做出清晰、有证据支撑的对比。
>
> 把每张图保存为 PNG，放在 `writeup/3_run_example_tasks/` 下。使用描述性文件名，例如 `bash-command-usage.png` 或 `trajectory-length.png`。你的图应当无需打开底层 Harbor 产物即可读懂。
>
> **交付物**：创建三张可视化图，并在 `writeup/3_run_example_tasks/responses.md` 中分别为每张图作 2–3 句简要解读。对每张图说明：测量了什么、如何计算、你得出什么结论。
>
> 从下列问题中选三个：
>
> - **effort-distribution.png**：智能体如何把精力分配到检查文件、编辑代码、运行测试、调试等活动上？
> - **bash-command-usage.png**：mini-SWE-agent 通过 `execute_bash` 完成工作。它在三条轨迹中使用了哪些命令行程序与命令模式，其使用情况又如何随任务变化？
> - **exploration-to-implementation.png**：各智能体在什么时点从"探索任务"转向"实现解法"？说明你的可视化如何识别这一转变。
> - **repeated-effort.png**：智能体在哪里重复命令、重访早先的假设，或以其他方式耗费精力却没有明显进展？
> - **context-consulted.png**：智能体查阅的仓库或环境上下文的**数量与类型**，在不同任务间有何差异？
> - **failure-recovery.png**：轨迹中出现了哪些失败与恢复的模式？例如某个命令或假设失败后，智能体接下来做了什么？
> - **custom-analysis.png**：另选一个你认为能揭示智能体行为重要方面的可测量属性，并解释你为什么选它。
>
> 许多分析都能成立；我们会评估你的分析是否定义清晰、是否忠于产物、以及解读是否有意义。

---

## 4 创建你自己的任务

现在你要从零设计一个原创的 Harbor 任务。任务可以针对任何能在其环境内被演练与验证的能力；我们更在意"单个、经过仔细打磨的评测"的强度，而不是覆盖某个特定领域。寻找灵感可见附录 A 中的示例任务。

### 4.1 任务要求

把你的任务放在 `tasks/<task-id>/`，并遵循第 3 节介绍的 Harbor 任务结构。你的提交必须满足以下要求：

- 任务有清晰的问题陈述，并测试一种你能够论证其动机的能力。
- 任务必须是**有意义的多轮（multi-turn）**：智能体必须探索、行动、并对观测到的结果作出反应，而不是一步给出答案。
- 环境从干净检出（clean checkout）开始构建，并包含智能体所需的全部初始资产。
- oracle 通过与模型尝试所用的**同一个验证器**获得满分奖励。
- 验证器要**有意义地检查结果**。仅检查文件是否存在不构成充分验证。
- 验证器产生的奖励落在 [0, 1] 区间。二值、连续或离散奖励均可接受。
- 任务应当像一个可信的 LM 智能体用法，而不是靠歧义、隐含假设或脆弱基础设施人为做难的谜题。

我们看重"因为需要实质推理、工具使用或迭代而困难"的任务，而不是"因为指令含糊或环境脆弱而困难"的任务。只有当任务表述良好、可解，且失败原因与被评测能力相关联时，模型失败才算作**难度的证据**。

### 4.2 开发并校验你的任务

在整个开发过程中都用 oracle 来验证完整评测流水线：

```bash
uv run harbor run \
  --path tasks/<task-id> \
  --agent oracle \
  --n-attempts 1 \
  --n-concurrent 1 \
  --job-name <task-id>-oracle \
  --jobs-dir tasks/<task-id>/artifacts \
  --yes
```

你还应当用不正确、不完整的解法去测试验证器。一个强验证器应当**拒绝貌似合理的取巧做法**，同时**接受真正满足任务的解法**，包括与你的 oracle 不同但正确的做法。

### 4.3 运行五次模型尝试

任务定稿后，用 mini-SWE-agent 和 Gemini 3.8 Flash 做五次独立尝试。五次尝试一起跑，并提交**全部**产物，而不只是结果好看的运行。

```bash
uv run harbor run \
  --path tasks/<task-id> \
  --agent-import-path pinned_agent:MiniSweAgent \
  --model <MODEL> \
  --env-file .env \
  --n-attempts 5 \
  --n-concurrent 1 \
  --job-name <task-id>-gemini \
  --jobs-dir tasks/<task-id>/artifacts \
  --yes
```

评分时我们会用相同的脚手架、模型与评测设置重新跑五次全新尝试。**我们的运行决定经验难度分数**，你的运行则为你的报告提供证据。

### 4.4 报告与交付物

在 `tasks/<task-id>/report.pdf` 写一份**一页**报告。报告应说明：被测试的能力、环境与初始资产、验证策略、五次尝试的全部结果、以及信息量最大的那次成功或失败。可以从 `writeup/4_create_tasks/report.tex` 的模板开始。

> **交付物**：你的提交必须包含：
> - `tasks/<task-id>/` 下一个完整的 Harbor 任务
> - `tasks/<task-id>/artifacts/` 下成功的 oracle 产物与全部 5 次模型尝试产物
> - `tasks/<task-id>/report.pdf` 下完整的一页报告
>
> 提交前运行 `uv run python check_submission.py` 检查是否有缺失文件。

### 4.5 评分

该任务占 70 分。各项标准的详细说明见下表之后。

| 评分项 | 我们评估什么 | 分值 |
| --- | --- | --- |
| 可复现性（Reproducibility） | 环境能否构建、任务能否从干净检出可靠运行、oracle 是否拿到满分奖励 | 10 |
| 能力（Capability） | 任务是否测试一种清晰、动机充分、现实应用显而易见的能力，且需要有意义的多轮工作 | 15 |
| 任务–验证器保真度（Task–verifier fidelity） | 验证器是否可靠地接受正确结果、拒绝不完整或错误的结果 | 15 |
| 难度（Difficulty） | 任务是否常被前沿模型做失败，且失败反映的是真实能力局限而非任务缺陷 | 20 |
| 报告（Report） | 五次结果是否被准确报告并被有思考地解读 | 10 |
| **合计** | | **70** |

**可复现性（10 分）**
我们会从提交的 commit 构建你的任务，并让 oracle 走与模型尝试相同的验证器。满分要求：环境能构建、oracle 奖励为 1、且 oracle 产物与全部五次模型尝试产物齐备。

**能力（15 分）**
任务针对一种在真实场景中有用的具体能力。成功应当**必须**依赖该能力，而不是一个平凡策略或单条命令。任务还应是有意义的多轮：智能体必须探索、行动、观察结果并调整。

**任务–验证器保真度（15 分）**
问题陈述与验证器应当描述同一个任务。我们会评估：

- 验证器检查的每一个行为，都在问题陈述中被陈述或被清晰隐含；且陈述中的每一项要求都被检查；
- 验证器明确惩罚错误与不完整的解法；
- 环境的初始状态、或一个什么都不做的智能体，得到奖励 0；
- 验证器接受与你的 oracle 走不同路线的正确解法；
- 验证器对结果的检查比"文件是否存在"或"在源码里搜关键词"更彻底。

奖励必须落在 [0, 1]。可以是二值、连续或离散的。

**难度（20 分）**

- **5 分**：在你五次尝试中，智能体至多在其中两次上成功。
- **15 分**：智能体的失败反映出你所针对能力上的真实短板。任务不应因为歧义、隐含假设或人为约束而变难。

**报告（10 分）**
我们会检查五次结果是否都被准确报告并被有思考地解读。

---

## 5 同行评审

设计评测还需要学会区分**真实的能力测量**与**仅仅是难、脆弱或表述不足**的任务。在任务创建截止后，你将评审若干同学写的任务，并套用评估你自己工作时所用的同一套标准。用心写评审，也是对同学的尊重。

### 5.1 评审流程

提交截止后，我们会把每个仓库冻结在它提交时的 commit，并把全班的任务发布到一个只读的课程画廊供下载。每次下载包含任务文件、报告与提交的模型轨迹。随后我们会给你分配三个不属于你的任务进行评审。

对每个被分配的任务：

1. 从课程画廊下载，阅读任务描述与报告。
2. 检查任务的指令、环境、验证器与 oracle 解法。
3. 查看提交的五条轨迹与验证器结果。
4. 在 `writeup/5_peer_review/` 的模板中填写一份评审。

你也可以运行被分配的任务，但不要求。画廊只用于下载任务；你不在那里提交任何东西。取而代之，把三份完成的评审编译成一个 `peer_review.pdf`，提交到 Gradescope 上单独的 Peer Review 作业。

### 5.2 评估什么

每份评审应包含：

- 关于该任务测量何种能力的简明概述；
- 按第 4 节评分细则逐项打分：可复现性、能力、任务–验证器保真度、难度、报告；
- 支撑你判断的证据，至少引用一个具体的任务文件、验证器检查项或轨迹事件；
- 该任务最强的一点；
- 一条具体、可执行的改进建议。

你的评分不必与我们的一致。我们评分的是：你的评审是否表现出对任务的准确理解，并用具体证据支撑其结论。

> **交付物**：把 `peer_review.pdf` 提交到 Gradescope 的 Peer Review 作业。该 PDF 必须包含针对三个被分配任务的各一份完整评审。

### 5.3 评分

同行评审占 15 分，三份评审各 5 分：1 分给对任务与能力的准确概述，2 分给具体、有证据支撑的评估，1 分给具体且有用的改进建议，1 分给完整性与清晰度。

---

## 附录 A 任务灵感

除第 3 节的任务之外，Harbor Hub 还托管了数千个任务。下表实例覆盖了广泛的能力、环境与验证策略。点击某个实例可查看其指令、环境与验证器；点击某个基准可了解更多。

| 实例 | 基准 | 描述 |
| --- | --- | --- |
| `burntsushi--ripgrep.3b7fd44` | ProgramBench | 只给定二进制与文档，从零重建 ripgrep。 |
| `httpx-streaming-json-iteration` | DeepSWE | 为 httpx 增加流式 JSON 迭代，并处理 content-type 与字符集。 |
| `task-69391d8d1ce51c407be1e531` | SWE Atlas (RF) | 把一个 Go 仓库中重复的 buffer-pool 代码重构为共享包。 |
| `task-6905333b74f22949d97ba998` | SWE Atlas (QnA) | 回答关于一个 Web 应用的 API 调用、Redux action 与响应式 CSS 的问题。 |
| `polyglot_go_forth` | Aider Polyglot | 用 Go 实现 Forth 语言一个子集的求值器。 |
| `job-shop-scheduling` | AlgoTune | 写一个比参考实现更快的车间调度（job-shop scheduling）求解器。 |
| `gso-speedup-pandas-merge` | GSO | 加速 pandas 的 merge/join 操作且不改变结果。 |
| `cybergym-arvo-368` | CyberGym | 构造一个能让有漏洞程序崩溃、但不会让修补后版本崩溃的输入。 |
| `corrupted-filesystem-recovery` | OpenThoughts TBLite | 修复损坏的文件系统镜像并恢复隐藏的 flag。 |
| `tau3-bench__tau3-airline-25` | τ³-bench | 在聊天中解决一位航空客户的请求，同时遵守预订政策。 |
| `gaia2-timed-1` | Gaia2 | 按固定时间表回复收到的邮件并创建日历事件。 |
| `law433-mk-01-ec4c48f7` | APEX-Agents | 仅凭一份居住协议与员工手册，为合同纠纷的双方辩护。 |
| `corporate-ma-draft-ppa-memo` | Harvey LAB | 审阅一笔收购的交割资产负债表，起草价格调整备忘录。 |
| `court-form-filling` | SkillsBench | 根据案情描述填写加州小额诉讼法庭 PDF 表单。 |
| `freecad-0280cda4fc` | CAD-Bench | 根据描述用脚本生成一个参数化 FreeCAD 零件，再做一次设计修改。 |
| `gaia-find-chess-winning-move` | GAIA | 从一张棋盘图片中找黑方的制胜着法。 |
| `arcagi2-grid-transform-88e3` | ARC-AGI-2 | 从示例中推断网格变换规则并应用。 |
| `qcircuitbench-design-ternary-simon-circuit-n3` | QCircuitBench | 设计一个能求解三元 Simon 问题的量子电路。 |
| `gravitational-wave-detection` | SkillsBench | 用匹配滤波在噪声探测器数据中找黑洞并合信号。 |
| `bix-diff-expr-mirna` | BixBench | 分析患者 miRNA 数据以度量差异表达。 |
| `neo-orbit-determination` | TB-Science | 从原始天体测量数据中挑出近地小行星的探测点并拟合其轨道。 |
| `dna-storage-codec` | TB-Science | 为一条易错的 DNA 数据存储信道设计编码器与解码器。 |

---

## 致谢

本作业由 **John Yang** 为斯坦福大学 CS329Z *Engineering AI Agents* 课程设计并实现，Claude Opus 5.5 提供了协助。我们感谢就本作业分享想法与反馈的各位，特别是 **Michael Ryan、Diyi Yang、Shreyas Sharma、Anantharaman Iyer**。

如果你希望在自己的课程中使用本作业，欢迎使用，只需简短署名。本作业以 MIT 许可证发布。

---

## 参考文献

[1] Mark Chen, Jerry Tworek, Heewoo Jun, Qiming Yuan, Henrique Ponde de Oliveira Pinto, Jared Kaplan, Harri Edwards, Yuri Burda, Nicholas Joseph, Greg Brockman, Alex Ray, Raul Puri, Gretchen Krueger, Michael Petrov, Heidy Khlaaf, Girish Sastry, Pamela Mishkin, Brooke Chan, Scott Gray, Nick Ryder, Mikhail Pavlov, Alethea Power, Lukasz Kaiser, Mohammad Bavarian, Clemens Winter, Philippe Tillet, Felipe Petroski Such, Dave Cummings, Matthias Plappert, Fotios Chantzis, Elizabeth Barnes, Ariel Herbert-Voss, William Hebgen Guss, Alex Nichol, Alex Paino, Nikolas Tezak, Jie Tang, Igor Babuschkin, Suchir Balaji, Shantanu Jain, William Saunders, Christopher Hesse, Andrew N. Carr, Jan Leike, Josh Achiam, Vedant Misra, Evan Morikawa, Alec Radford, Matthew Knight, Miles Brundage, Mira Murati, Katie Mayer, Peter Welinder, Bob McGrew, Dario Amodei, Sam McCandlish, Ilya Sutskever, and Wojciech Zaremba. *Evaluating large language models trained on code*, 2021.（即 HumanEval / Codex 论文）

[2] Carlos E. Jimenez, John Yang, Alexander Wettig, Shunyu Yao, Kexin Pei, Ofir Press, and Karthik Narasimhan. *SWE-bench: Can language models resolve real-world GitHub issues?*, 2024.

[3] Mike A. Merrill, Alexander G. Shaw, Nicholas Carlini, Boxuan Li, Harsh Raj, Ivan Bercovich, Lin Shi, Jeong Yeon Shin, Thomas Walshe, E. Kelly Buchanan, Junhong Shen, Guanghao Ye, Haowei Lin, Jason Poulos, Maoyu Wang, Marianna Nezhurina, Jenia Jitsev, Di Lu, Orfeas Menis Mastromichalakis, Zhiwei Xu, Zizhao Chen, Yue Liu, Robert Zhang, Leon Liangyu Chen, Anurag Kashyap, Jan-Lucas Uslu, Jeffrey Li, Jianbo Wu, Minghao Yan, Song Bian, Vedang Sharma, Ke Sun, Steven Dillmann, Akshay Anand, Andrew Lanpouthakoun, Bardia Koopah, Changran Hu, Etash Guha, Gabriel H. S. Dreiman, Jiacheng Zhu, Karl Krauth, Li Zhong, Niklas Muennighoff, Robert Amanfu, Shangyin Tan, Shreyas Pimpalgaonkar, Tushar Aggarwal, Xiangning Lin, Xin Lan, Xuandong Zhao, Yiqing Liang, Yuanli Wang, Zilong Wang, Changzhi Zhou, David Heineman, Hange Liu, Harsh Trivedi, John Yang, Junhong Lin, Manish Shetty, Michael Yang, Nabil Omi, Negin Raoof, Shanda Li, Terry Yue Zhuo, Wuwei Lin, Yiwei Dai, Yuxin Wang, Wenhao Chai, Shang Zhou, Dariush Wahdany, Ziyu She, Jiaming Hu, Zhikang Dong, Yuxuan Zhu, Sasha Cui, Ahson Saiyed, Arinbjörn Kolbeinsson, Jesse Hu, Christopher Michael Rytting, Ryan Marten, Yixin Wang, Alex Dimakis, Andy Konwinski, and Ludwig Schmidt. *Terminal-bench: Benchmarking agents on hard, realistic tasks in command line interfaces*, 2026.

---

## 术语与背景速览（译者整理）

| 术语 | 含义 |
| --- | --- |
| **Benchmark（基准）** | 一组任务的集合，用于测试智能体的能力或行为；其有效性来自"共同标尺"效应 |
| **Problem statement（问题陈述）** | 初始提示中交给智能体的任务描述，对应磁盘上的 `instruction.md` |
| **Environment（环境）** | 智能体行动的场所（终端/浏览器/模拟器/真实世界），由 `environment/Dockerfile` 构建 |
| **Verification（验证）** | 对提交（通常是环境终态）的评分机制，形式包括单元测试、评分细则、偏好模型 |
| **Harbor** | 本作业使用的评测框架：给出任务的磁盘表示，编排"启动尝试 → 跑验证器 → 存奖励与产物" |
| **Agent scaffold（脚手架）** | 驱动模型的智能体框架。本作业统一用 mini-SWE-agent：只暴露一个 bash 工具，ReAct 式多轮循环 |
| **Oracle** | 参考解智能体。跑通它意味着"环境 + 验证器"这条链路本身是通的（reward = 1） |
| **Reward（奖励）** | 验证器输出的分数，必须落在 [0, 1]；可为二值/连续/离散 |
| **Trajectory（轨迹）** | 一次尝试的完整记录：每轮的思考、发出的动作、收到的观测 |
| **Task–verifier fidelity（任务–验证器保真度）** | 问题陈述与验证器描述的是否同一件事；验证器是否既不漏放也不过严 |
| **Long-horizon（长时程）** | 需要大量连续步骤才能完成的任务，如 Terminal-Bench 的 8 小时专家任务 |
| **Clean checkout（干净检出）** | 只依赖仓库提交内容、不依赖本地残留状态的构建起点 |
| **Hillclimbing（爬坡）** | 在既有基准上不断刷高分数，直到边际收益递减、暴露新局限 |

**核心数字速记**

- 总分 **100**：基础 5 + 运行任务 10 + 造任务 70 + 同行评审 15
- 造任务 70 分拆解：可复现 10 / 能力 15 / 任务–验证器保真 15 / **难度 20** / 报告 10
- 难度 20 分的两段式给分：五次尝试中成功 ≤2 次得 **5** 分；失败须反映真实能力短板再得 **15** 分
- 模型设置固定为 **mini-SWE-agent + Gemini 3.8 Flash**，**5 次独立尝试**，`--n-concurrent 1`
- 关键截止：任务提交 **11 月 23 日（周一）**；同行评审 **12 月 2 日（周三）**
- Terminal-Bench 的 `model-extraction-relu-logits` 被标为 **hard**，专家预计约 **8 小时**
- Harbor Hub 托管 **数千** 个现成任务；附录 A 列了 **22** 个跨领域示例
- 三个示例任务构成 2021 → 2026 的谱系：**HumanEval（函数级）→ SWE-bench Verified（仓库级）→ Terminal-Bench 2.1（终端级、长时程）**

**这份文档真正的内核（译者理解）**

它表面上是一份课程作业说明，实质上给出了"如何造一个可信评测"的完整方法论骨架，可直接迁移到任何自建 eval 上：

1. **评测 = 三元组（问题陈述 / 环境 / 验证）**，三者缺一不可，且必须能被写成磁盘上的具体资产。
2. **可复现性优先于难度**：先让 oracle 拿到 1.0，再谈模型失败；环境脆弱导致的失败不算难度。
3. **保真度是双向的**：验证器检查的每一项都得在题面里说过；题面里说的每一项都得被检查。同时要能接受"非 oracle 路线"的正确解。
4. **难度要归因**：区分"能力不够"与"题面含糊 / 隐含假设 / 人为约束"，后者是缺陷不是难度。
5. **报告全部五次结果**，包括不利的运行——这是评测诚实的底线，也是同行评审时最该核对的证据。

