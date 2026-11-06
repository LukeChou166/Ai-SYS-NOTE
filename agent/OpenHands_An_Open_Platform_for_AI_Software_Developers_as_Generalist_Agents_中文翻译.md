# OpenHands：面向 AI 软件开发者（通才智能体）的开放平台

> **原文**: OpenHands: An Open Platform for AI Software Developers as Generalist Agents
> **作者**: Xingyao Wang, Boxuan Li, Yufan Song, Frank F. Xu, Xiangru Tang, Mingchen Zhuge, Jiayi Pan, Yueqi Song, Bowen Li, Jaskirat Singh 等 23 人（UIUC / CMU / Yale / UC Berkeley / 阿里巴巴 / All Hands AI 等）
> **日期/arXiv**: arXiv:2407.16741v3 [cs.SE]，2025-04-18（v1 发布于 2024-07）
> **链接**: https://arxiv.org/abs/2407.16741 ｜ 代码: https://github.com/All-Hands-AI/OpenHands
> **译者说明**: 本文为完整全文翻译（正文逐段全译；附录 H/I/J/K 为代码清单与提示词，以「要点提炼」方式呈现；带 *（译者注）* 标记的段落为译者补充说明，非原文内容）。

---

## 摘要

软件是人类所掌握的最强大工具之一；它让熟练的程序员能够以复杂而深刻的方式与世界交互。与此同时，得益于大语言模型（LLM）的进步，能够与环境交互并改变环境的 AI 智能体（agent）也在迅速发展。本文介绍 OpenHands（前身为 OpenDevin），一个用于开发强大且灵活的 AI 智能体的平台，这些智能体以与人类开发者相似的方式与世界交互：编写代码、使用命令行、浏览网页。

我们描述了该平台如何支持：实现新的智能体、与沙箱化代码执行环境进行安全交互、多智能体之间的协调，以及评测基准的集成。基于目前已集成的基准，我们在 15 项富有挑战性的任务上对智能体进行了评测，涵盖软件工程（如 SWE-bench）、网页浏览（如 WebArena）等多个领域。OpenHands 以宽松的 MIT 许可证发布，是一个横跨学术界与工业界的社区项目，已收到来自 188 位以上贡献者的超过 2100 次贡献。

| 项目 | 链接 |
| --- | --- |
| 代码 | https://github.com/All-Hands-AI/OpenHands |
| Slack | http://bit.ly/OpenHands-Slack |

---

## 1 引言

在大语言模型（LLM）的驱动下，面向用户的 AI 系统（如 ChatGPT）已越来越有能力完成复杂任务：准确回答用户提问、解决数学问题、生成代码等。其中，能够感知外部环境并对其施加作用的 AI 智能体，近期获得了越来越多的研究关注。它们正朝着完成复杂任务的方向发展：开发软件（Jimenez et al., 2024）、在真实网站上导航（Zhou et al., 2023a）、做家务（Ahn et al., 2022），甚至开展科学研究（Boiko et al., 2023; Tang et al., 2024a）。

随着 AI 智能体能够处理复杂问题，其开发与评测也变得极具挑战。近期出现了许多开源框架来降低智能体开发门槛（Hong et al., 2023; Chen et al., 2024; Wu et al., 2023）。这些智能体框架一般包含：1) 智能体与世界交互的接口（如基于 JSON 的函数调用或代码执行）；2) 智能体所处的运行环境；3) 人机或智能体间通信的交互机制。这些框架以各种方式简化并加速了开发流程（表 1、§C）。

在设计 AI 智能体时，我们也可以借鉴人类与世界交互的方式。当前人类与世界交互最强大的方式是通过**软件**——软件驱动着我们生活的方方面面，从满足基本需求的后勤系统到科学、技术与 AI 本身的进步都离不开它。鉴于软件的强大能力，以及围绕其高效开发、使用与部署所形成的成熟工具链，它为 AI 智能体以复杂方式与世界交互提供了理想接口。然而，构建能有效开发软件的智能体也有其独特挑战：如何让智能体在复杂软件系统中高效地创建和修改代码？如何为它们提供可即时收集信息的工具，用于调试问题或获取任务所需信息？如何确保开发过程安全，避免对用户系统造成负面影响？

本文介绍 **OpenHands**（前身为 OpenDevin），一个由社区驱动的平台，旨在开发通过软件与世界交互的通才（generalist）与专才（specialist）AI 智能体。¹ OpenHands 最初受 AI 软件工程师 Devin（Cognition.ai）启发，但通过多样化的社区贡献，已迅速演进为支持远超软件工程范畴的广泛应用。它具备以下特性：

1. **交互机制**：通过一个强大而灵活的**事件流（event stream）架构**，让用户界面、智能体与环境之间进行交互（§2.1）；
2. **运行时环境（runtime）**：由一个 docker 沙箱化的操作系统构成，内置 bash shell、网页浏览器与 IPython 服务器，供智能体调用（§2.2）；
3. **智能体-计算机接口**：让智能体以类似真实软件工程师的方式与环境交互（§2.3），包括 a) 创建和编辑复杂软件，b) 在沙箱中执行任意代码，c) 浏览网站收集信息；
4. **多智能体委派**：让多个专职智能体协同工作（§2.4）；
5. **评测框架**：支持在广泛任务上对智能体进行评测（§4）。

> **图 1 | OpenHands 用户界面（UI，§D）。** 用户可以查看文件、查看已执行的 bash 命令 / Python 代码、观察智能体的浏览器操作，并直接与智能体对话。UI 与事件流直连，用户可在智能体工作的任意时刻打断它并给出反馈。

重要的是，OpenHands 不只是一个概念性框架，它还提供了完整且开箱即用的智能体、环境与评测实现。截至本文撰写时，OpenHands 包含一个拥有 10 个以上已实现智能体的 AgentHub（§3），其中包括一个基于 CodeAct 架构（Wang et al., 2024a）实现的强大通才智能体，并加入网页浏览（ServiceNow）与代码编辑专才能力（Yang et al., 2024）。与用户的交互通过聊天式 UI 实现，可实时可视化智能体的当前动作并接受即时反馈（图 1、§D）。此外，评测框架目前支持 15 个基准，我们据此对智能体进行了评测（§4）。

OpenHands 以允许商业使用的宽松 MIT 许可证发布，有望支撑学术界与工业界广泛的研究创新与真实应用。OpenHands 已获得显著影响力：GitHub 上斩获 32K star，收到 188 位以上贡献者的超过 2100 次贡献。我们期望 OpenHands 成为未来研究创新与多元应用的催化剂，由广泛的实践者社区共同推动。

> *（译者注）¹ 原文脚注：项目前名 OpenDevin，为致敬与区分已更名。*

---

## 2 OpenHands 架构

> **图 2 | OpenHands 三大组件。** 1) **智能体抽象**：社区可向 AgentHub（§3）贡献不同的智能体实现（§2.1）；2) **事件流**：记录动作与观察历史的时序集合；3) **运行时（Runtime）**：把每个动作执行为观察（§2.2）。数据流为：智能体读取事件流中的历史（状态）→ 产出动作 → 运行时执行动作 → 生成观察 → 追加回事件流。

下面详细介绍 OpenHands 的使用方式，重点是：1) 如何定义并实现一个智能体（§2.1）；2) 动作如何被执行并产生观察（§2.2）；3) 如何可靠地管理和扩展智能体常用技能（§2.3）；4) 如何组合多个智能体协作解题（§2.4）。图 2 给出了总览。

### 2.1 智能体的定义与实现

智能体在解决用户指定任务时，可以感知环境状态（如先前的动作与观察），并产出待执行的动作。

**状态（State）与事件流（Event Stream）。** 在 OpenHands 中，状态是封装智能体执行所需全部相关信息的数据结构。状态的核心组件是**事件流**——按时间顺序记录的过往动作与观察的集合，包括智能体自身的动作和用户交互（如指令、反馈）。除事件流外，状态还包含支撑智能体运行的辅助信息，例如 LLM 调用的累计成本、用于追踪多智能体委派的元数据（§2.4）以及其他执行相关参数。

**动作（Actions）。** 受 CodeAct（Wang et al., 2024a）启发，OpenHands 通过一组核心的通用动作将智能体与环境连接起来：

- `IPythonRunCellAction` 与 `CmdRunAction` 让智能体能在沙箱环境（如安全隔离的 Linux 操作系统）中执行任意 Python 代码与 bash 命令；
- `BrowserInteractiveAction` 借助 BrowserGym（Drouin et al., 2024）引入的浏览领域专用语言与网页浏览器交互。

这些动作的选择旨在提供一套覆盖人类软件工程师与数据分析师大多数任务的、既全面又灵活的原语。基于编程语言（programming language, PL）的动作空间足够强大灵活，可以用不同形态的工具（如 Python 函数、REST API 等）完成任意任务，同时保持可靠且易于维护（Wang et al., 2024a）。

> **图 3 | 在 OpenHands 中实现一个智能体的最小示例。** 核心是 `step` 函数：接收当前状态，把历史（动作-观察对）拼装成消息列表交给 LLM，解析 LLM 返回内容并映射为对应动作类型：

```python
class MinimalAgent:
    def reset(self) -> None:
        self.system_message = "You are a helpful assistant ..."

    def step(self, state: State):
        messages: list[dict[str, str]] = [
            {"role": "system", "content": self.system_message}
        ]
        for prev_action, obs in state.history:
            messages.append(get_action_message(prev_action))
            messages.append(get_observation_message(obs))

        # 用 LLM 生成回复（如思考、动作）
        response = self.llm.do_completion(messages)
        # 解析动作并在运行时中执行
        action = self.parse_response(response)

        if self.is_finish_command(action):
            return AgentFinishAction()
        elif self.is_bash_command(action):
            return CmdRunAction(command=action.command)
        elif self.is_python_code(action):
            return IPythonRunCellAction(code=action.code)
        elif self.is_browser_action(action):
            return BrowseInteractiveAction(code=action.code)
        else:
            return MessageAction(content=action.message)
```

这一设计同样兼容现有的、需要预定义工具列表的工具调用型智能体（Chase, 2022）：用户可以轻松地用基础动作支持的编程语言定义工具（如写一个计算器的 Python 函数），并通过 JSON 风格的函数调用方式（Qin et al., 2023）提供给智能体。更进一步，框架基于 PL 的强大原语还使智能体能够在没有现成 API 时**自己创建工具**（如生成 Python 函数，Yuan et al. 2023）。关于如何把这些核心 PL 动作组合成多样化工具集，参见 §2.3。

**观察（Observations）。** 观察描述智能体所感知到的环境变化（如先前动作的执行结果、人类用户发来的文本消息等）。

**实现一个新智能体。** 智能体抽象的设计简单而强大，让用户能轻松创建和定制面向各类任务的智能体。其核心是 **`step` 函数**：接收当前状态作为输入，依据智能体自身逻辑生成合适的动作。图 3 给出了该抽象的简化示例代码。借助这一抽象，OpenHands 让用户专注于定义期望的智能体行为与逻辑，而无需操心动作执行（§2.2）的底层细节。

### 2.2 智能体运行时：动作执行如何产生观察

智能体运行时（Agent Runtime）提供了一个通用环境，为智能体配备与人类软件开发者相当的动作空间，使 OpenHands 智能体能够应对广泛的软件开发与网页任务，包括复杂的软件开发工作流、数据分析项目、网页浏览任务等。它允许智能体访问 bash 终端来运行代码和命令行工具，使用 Jupyter notebook 即席编写并执行代码，以及与网页浏览器交互完成网页任务（如信息检索）。

**Docker 沙箱。** 对每个任务会话，OpenHands 都会启动一个安全隔离的 docker 容器沙箱，事件流中的所有动作都在其中执行。OpenHands 通过沙箱内运行的 REST API 服务器（即 OpenHands 动作执行 API）连接沙箱，执行事件流中的任意动作（如 bash 命令、Python 代码），并把执行结果作为观察返回。包含用户希望智能体处理文件的可配置工作区目录会被挂载进该安全沙箱，供智能体访问。

**OpenHands 动作执行 API。** OpenHands 维护一个运行在 docker 沙箱内的 API 服务器，监听来自事件流的动作执行请求。该 API 服务器维护：

1. 一个与操作系统环境（由 docker 镜像指定）相连的 bash shell，用于命令执行；
2. 一个 Jupyter IPython 服务器，处理交互式 Python 代码执行请求并把结果返回事件流；
3. 一个基于 Playwright 的 Chromium 浏览器，提供由 BrowserGym 定义的一组浏览动作原语（如导航、点击、输入、滚动；完整动作列表见 §J）。执行这些动作后，浏览器运行时会返回关于当前浏览器状态的丰富观察，包括 HTML、DOM、可访问性树（accessibility tree）、截图、已打开标签页等。

**任意 Docker 镜像支持。** OpenHands 支持基于任意 docker 镜像的运行时，从而让智能体可以运行在具备不同软件环境的任意操作系统上。OpenHands 实现了一套构建机制：接收用户提供的任意 docker 镜像，并把 OpenHands 动作执行 API 安装进该镜像以支持智能体交互。运行时的详细描述见 §F。

### 2.3 智能体技能（AgentSkills）：可扩展的智能体-计算机接口

SWE-Agent（Yang et al., 2024）强调了精心设计的**智能体-计算机接口（Agent-Computer Interface, ACI，即面向特定任务的专业化工具）**对成功解决复杂任务的重要性。然而，创建、维护和分发大量工具是一项繁重的工程挑战，尤其是当我们想让这些工具可被不同的智能体实现（§3）共用时。为解决这些问题，我们构建了 **AgentSkills 库**——一个旨在增强智能体能力的工具箱，提供那些难以直接通过基础 bash 命令或 Python 代码获得的实用工具。

**易于创建和扩展工具。** AgentSkills 被设计为一个 Python 包，由多个实用函数（即工具）组成，并自动导入 Jupyter IPython 环境（§2.2）。用 Python 函数定义工具的低门槛让社区成员更容易向该库贡献新工具；而 Python 包的通用性也让不同智能体实现可以通过我们的核心动作 `IPythonRunCellAction`（§2.1）轻松使用这些工具。

**收录标准与理念。** 在 AgentSkills 库中，我们并不打算包装每一个 Python 包、重新教智能体如何使用（例如 LLM 已经会用 pandas 读取 CSV 文件，就没必要再造一个教智能体读该格式文件的工具）。我们只在新技能满足以下条件时才收录：(1) LLM 直接写代码不易实现（如编辑代码并替换特定行），和/或 (2) 需要调用外部模型（如语音转文字模型、代码编辑模型）。

**当前支持的技能。** AgentSkills 库收录了改编自 SWE-Agent（Yang et al., 2024）与 Aider（Gauthier）的文件编辑工具，如 `edit_file`（从指定行开始修改已有文件）、用于查看文件不同部分的滚动函数 `scroll_up` / `scroll_down`；还包含支持多模态文档读取的工具，如 `parse_image`（用视觉语言模型如 GPT-4V 从图像提取信息）与 `parse_pdf`（从 PDF 读取文本）。完整技能列表见 §I。

### 2.4 智能体委派（Agent Delegation）：协作式多智能体交互

OpenHands 也支持多个智能体之间的交互。为此，我们使用一种特殊动作类型 `AgentDelegateAction`，让一个智能体把特定子任务委派给另一个智能体。例如，对网页浏览支持有限的通才智能体 CodeActAgent，可以用 `AgentDelegateAction` 把浏览任务委派给专职的 BrowsingAgent，以完成更复杂的浏览活动（如网页导航、点击按钮、提交表单等）。

> **表 1 | 不同 AI 智能体框架的对比（§C）。** "Swe" 指软件工程。各列含义：**标准化工具库**——框架是否包含可供不同智能体实现复用的工具（§2.3）；**内置沙箱与代码执行**——是否支持对智能体生成代码的沙箱执行；**内置网页浏览器**——是否为智能体提供功能完整的浏览器；**人机协作**——是否支持多轮人机协作（如人类可在任务执行中打断智能体并/或追加反馈与指令）；**AgentHub**——是否托管多种智能体实现（§3）；**评测框架**——是否提供在挑战性基准上对智能体的系统化评测（§4）；**智能体 QC（质量控制）**——框架是否集成测试（§E）以保证框架整体的软件质量。

| 框架 | 领域 | 图形界面 | 工具库 | 沙箱与代码执行 | 浏览器 | 多智能体 | 人机协作 | AgentHub | 评测框架 | 智能体 QC |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| AutoGPT (Gravitas, 2023) | 通用 | ✔ | ✗ | ✗ | ✗ | ✗ | ✗ | ✔ | ✗ | ✔ |
| LangChain (Chase, 2022) | 通用 | ✗ | ✔ | ✗* | ✗* | ✗ | ✗ | ✔ | ✗ | ✗ |
| MetaGPT (Hong et al., 2023) | 通用 | ✗ | ✔ | ✗ | ✔ | ✔ | ✗ | ✔ | ✗ | ✔ |
| AutoGen (Wu et al., 2023) | 通用 | ✗ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✗ |
| AutoAgents (Chen et al., 2024) | 通用 | ✗ | ✗ | ✗ | ✗ | ✔ | ✗ | ✗ | ✗ | ✗ |
| Agents (Zhou et al., 2023b) | 通用 | ✗ | ✗ | ✗ | ✗ | ✔ | ✔ | ✗ | ✗ | ✗ |
| Xagents (Team, 2023) | 通用 | ✔ | ✔ | ✗ | ✔ | ✔ | ✗ | ✔ | ✗ | ✗ |
| OpenAgents (Xie et al., 2023) | 通用 | ✔ | ✗ | ✔ | ✔ | ✗ | ✗ | ✔ | ✗ | ✗ |
| GPTSwarm (Zhuge et al., 2024) | 通用 | ✗ | ✔ | ✗ | ✗ | ✔ | ✔ | ✗ | ✗ | ✗ |
| AutoCodeRover (Zhang et al., 2024b) | SWE | ✗ | ✗ | ✔ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| SWE-Agent (Yang et al., 2024) | SWE | ✗ | ✗ | ✔ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| **OpenHands** | 通用 | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |

\* 无原生支持，但有第三方商业方案可用。

---

## 3 AgentHub：社区贡献智能体的中心

基于我们的智能体抽象（§2.1），OpenHands 支持广泛的社区贡献智能体实现，供最终用户选择，并可作为不同智能体任务的基线。

**CodeAct Agent。** CodeActAgent 是基于 CodeAct 框架（Wang et al., 2024a）的默认通才智能体。每一步中，智能体可以 (1) 用自然语言与人类对话，请求澄清、确认等，或 (2) 通过执行代码（即 CodeAct）执行任务，包括 bash 命令、Python 代码或浏览器专用语言（§2.2）。这一通用动作空间让智能体（v1.5 及以上）能够完成编辑文件、浏览网页、运行程序等多种任务。

**Browsing Agent。** 我们实现了一个通才网页智能体 Browsing Agent，作为网页任务的简单而有效的基线。该智能体与 WebArena（Zhou et al., 2023a）中的类似，但改进了观察与动作，且只使用零样本提示。完整提示见 §K。

**GPTSwarm Agent。** GPTSwarm（Zhuge et al., 2024）开创了用可优化的图来构建智能体系统的方法，以模块化的方式统一了各类语言智能体框架。每个节点代表一种独立操作，边定义协作与通信路径。这种设计支持对节点和边的自动优化，推动多智能体系统构建的发展。

**Micro Agent(s)。** 此外，OpenHands 支持创建 **micro agent**——面向特定任务的专用智能体。micro agent 复用现有通才智能体（如 CodeAct Agent）的大部分实现，旨在降低智能体开发门槛，让社区成员可以分享对他们特定用例行之有效的专用提示词。

---

## 4 评测

> **表 2 | OpenHands 中的评测基准。**

| 类别 | 基准 | 所需能力 |
| --- | --- | --- |
| 软件 | SWE-Bench (Jimenez et al., 2024) | 修复 GitHub issue |
| 软件 | HumanEvalFix (Muennighoff et al., 2024) | 修复 bug |
| 软件 | BIRD (Li et al., 2023b) | Text-to-SQL |
| 软件 | BioCoder (Tang et al., 2024c) | 生物信息学编程 |
| 软件 | ML-Bench (Tang et al., 2024b) | 机器学习编程 |
| 软件 | Gorilla APIBench (Patil et al., 2023) | 软件 API 调用 |
| 软件 | ToolQA (Zhuang et al., 2024) | 工具使用 |
| 网页 | WebArena (Zhou et al., 2023a) | 目标规划与真实浏览 |
| 网页 | MiniWoB++ (Liu et al., 2018) | 合成网页上的短轨迹任务 |
| 综合辅助 | GAIA (Mialon et al., 2023) | 工具使用、浏览、多模态 |
| 综合辅助 | GPQA (Rein et al., 2023) | 研究生水平的"防谷歌"问答 |
| 综合辅助 | AgentBench (Liu et al., 2023) | 操作系统交互（bash） |
| 综合辅助 | MINT (Wang et al., 2024b) | 多轮数学与代码问题 |
| 综合辅助 | Entity Deduction Arena (Zhang et al., 2024a) | 状态追踪与策略规划 |
| 综合辅助 | ProofWriter (Tafjord et al., 2021) | 演绎逻辑推理 |

为了系统化地追踪构建通才数字智能体的进展，我们向 OpenHands 集成了 15 个已有基准（表 2），覆盖软件工程、网页浏览与综合辅助。本节中，我们将 OpenHands 与开源可复现的基线进行比较，这些基线均未针对基准内容做专门的人工提示工程。为简洁起见，本节其余部分用 "OH" 作为 OpenHands 的缩写。

### 4.1 结果总览

OpenHands 的目标是开发能够通过软件接口与世界交互的通用数字智能体（如 §2.1 所述的代码动作）。我们认识到，一个软件智能体不仅应擅长代码编辑，还应擅长网页浏览和各种辅助任务，例如回答关于代码仓库的问题或开展在线调研。

表 3 展示了一组精选评测结果。OpenHands 智能体虽然未必在每个类别都做到最强，但其设计以通用性为核心：值得注意的是，**同一个 CodeAct 智能体在系统提示不做任何修改的情况下**，在软件开发、网页交互与综合任务三大类别上均展现出有竞争力的表现。与通常针对特定任务类别设计与优化的基线智能体相比，这一点尤为难得。

> **表 3 | OpenHands 智能体的精选评测结果（§4）。** 软件全表见表 4，网页见表 5，综合辅助见表 6。

| 类别 | 智能体 | 模型 | SWE-Bench Lite | WebArena | GPQA | GAIA |
| --- | --- | --- | --- | --- | --- | --- |
| **软件工程智能体** | SWE-Agent (Yang et al., 2024) | gpt-4-1106-preview | 18.0 | − | − | − |
| | AutoCodeRover (Zhang et al., 2024b) | gpt-4-0125-preview | 19.0 | − | − | − |
| | Aider (Gauthier) | gpt-4o & claude-3-opus | 26.3 | − | − | − |
| | Moatless Tools (Örwall) | claude-3.5-sonnet | 26.7 | − | − | − |
| | Agentless (Xia et al., 2024) | gpt-4o | 27.3 | − | − | − |
| **网页浏览智能体** | Lemur (Xu et al., 2023) | Lemur-chat-70b | − | 5.3 | − | − |
| | Patel et al. (2024) | 合成数据训练的 72B | − | 9.4 | − | − |
| | AutoWebGLM (Lai et al., 2024) | 人/智能体标注训练的 7B | − | 18.2 | − | − |
| | Auto Eval & Refine (Pan et al., 2024) | GPT-4 + Reflexion w/ GPT-4V | − | 20.2 | − | − |
| | WebArena Agent (Zhou et al., 2023a) | gpt-4-turbo | − | 14.4 | − | − |
| **综合辅助智能体** | AutoGPT (Gravitas, 2023) | gpt-4-turbo | − | − | − | 13.2 |
| | Few-shot CoT (Rein et al., 2023) | Llama-2-70b-chat | − | − | 28.1 | − |
| | | gpt-3.5-turbo-16k | − | − | 29.6 | − |
| | | gpt-4 | − | − | 38.8 | − |
| **OpenHands 智能体** | CodeActAgent v1.8 | gpt-4o-mini-2024-07-18 | 6.3 | 8.3 | − | − |
| | | gpt-4o-2024-05-13 | 22.0 | 14.5* | 53.1* | − |
| | | claude-3-5-sonnet | 26.0 | 15.3 | 52.0 | − |
| | GPTSwarm v1.0 | gpt-4o-2024-05-13 | − | − | − | 32.1 |

\* 数字引自 CodeActAgent v1.5。

### 4.2 软件工程

接下来我们报告软件工程基准的结果（表 4）。

**SWE-Bench**（Jimenez et al., 2024）用于评估智能体解决真实 GitHub issue 的能力（如 bug 报告或功能请求）。智能体与代码仓库交互，尝试通过文件编辑与代码执行修复 issue。智能体修改后的代码仓库会用一套测试套件验证，其中包含了人类开发者修复同一 issue 时新增的测试。每个测试实例附带一段"提示文本（hint text）"，即关于如何解决问题的自然语言建议。本文报告的所有结果均**不使用**提示文本。为便于低成本高效测试，官方创建了规范子集 SWE-bench Lite；出于成本考虑，我们默认在该子集上测试。²

> *（译者注）² 原文脚注：以每实例 3 美元的保守估计，跑完 2294 个实例的完整集约需 6900 美元。*

**结果。** 如表 4 所示，我们最新版本的 CodeActAgent v1.8 使用 claude-3.5-sonnet 在 SWE-Bench Lite 上取得了 26% 的有竞争力的解决率，与其他开源 SWE 专才智能体相当。

> **表 4 | OpenHands 软件工程评测结果（§4.2）。**

| 基准（实例数） | 智能体 / 设置 | 模型 | 成功率 (%) | 平均成本 ($) |
| --- | --- | --- | --- | --- |
| **SWE-Bench Lite**（300，无提示） | SWE-Agent (Yang et al., 2024) | gpt-4-1106-preview | 18.0 | 1.67 |
| | AutoCodeRover (Zhang et al., 2024b) | gpt-4-0125-preview | 19.0 | − |
| | Aider (Gauthier) | gpt-4o & claude-3-opus | 26.3 | − |
| | OH CodeActAgent v1.8 | gpt-4o-mini-2024-07-18 | 7.0 | 0.01 |
| | | gpt-4o-2024-05-13 | 22.0 | 1.72 |
| | | claude-3-5-sonnet@20240620 | 26.0 | 1.10 |
| **HumanEvalFix**（164） | Prompting，0-shot | BLOOMZ-176B | 16.6 | − |
| | | OctoCoder-15B | 30.4 | − |
| | | DeepSeekCoder-33B-Instruct | 47.5 | − |
| | | StarCoder2-15B | 48.6 | − |
| | SWE-agent，1-shot (Yang et al., 2024) | gpt-4-turbo | 87.7 | − |
| | OH CodeActAgent v1.5（通才，0-shot） | gpt-3.5-turbo-16k-0613 | 20.1 | 0.11 |
| | | gpt-4o-2024-05-13 | 79.3 | 0.14 |
| **BIRD**（300） | Prompting，0-shot | CodeLlama-7B-Instruct | 18.3 | − |
| | | CodeQwen-7B-Chat | 31.3 | − |
| | OH CodeActAgent v1.5 | gpt-4-1106-preview | 42.7 | 0.19 |
| | | gpt-4o-2024-05-13 | 47.3 | 0.11 |
| **ML-Bench**（68） | prompting + BM25，0-shot | gpt-3.5-turbo | 11.0 | − |
| | | gpt-4-1106-preview | 22.1 | − |
| | | gpt-4o-2024-05-13 | 26.2 | − |
| | SWE-Agent (Yang et al., 2024) | gpt-4-1106-preview | 42.6 | 1.91 |
| | Aider (Gauthier) | gpt-4o | 64.4 | − |
| | OH CodeActAgent v1.5 | gpt-4o-2024-05-13 | 76.5 | 0.25 |
| | | gpt-4-1106-preview | 58.8 | 1.22 |
| | | gpt-3.5-turbo-16k-0613 | 13.2 | 0.12 |
| **BioCoder (Python)**（157） | prompting，0-shot | gpt-3.5-turbo | 11.0 | − |
| | | gpt-4-1106-preview | 12.7 | − |
| | OH CodeActAgent v1.5 | gpt-4o-2024-05-13 | 27.5 | 0.13 |
| **Gorilla APIBench**（1775） | Prompting，0-shot | claude-v1 | 8.7 | − |
| | | gpt-4-0314 | 21.2 | − |
| | | gpt-3.5-turbo-0301 | 29.7 | − |
| | Gorilla（API 调用微调），0-shot | llama-7b | 75.0 | − |
| | OH CodeActAgent v1.5 | gpt-3.5-turbo-0125 | 21.6 | 0.002 |
| | | gpt-4o-2024-05-13 | 36.4 | 0.04 |
| **ToolQA**（800，easy 子集） | Prompting，0-shot | ChatGPT + CoT | 5.1 | − |
| | | ChatGPT | 5.6 | − |
| | | Chameleon | 10.6 | − |
| | ReAct，0-shot (Yao et al., 2023; OpenAI, 2024a) | gpt-3.5-turbo | 36.8 | − |
| | | gpt-3 | 43.1 | − |
| | OH CodeActAgent v1.5 | gpt-3.5-turbo-0125 | 2.3 | 0.03 |
| | | gpt-4o-2024-05-13 | 47.2 | 0.91 |

#### 4.2.1 HumanEvalFix

**HumanEvalFix**（Muennighoff et al., 2024）要求智能体在给定测试用例的辅助下修复函数中的 bug。这些 bug 被构造成确保一个或多个测试用例失败。我们聚焦该基准的 Python 子集，允许模型通过多轮自调试（结合测试执行的反馈）来修复 bug，并遵循 Muennighoff et al. (2024) 的设置使用 pass@k（Chen et al., 2021）。

**结果。** 在表 4 中，OpenHands CodeActAgent 成功修复了 Python 子集中 79.3% 的 bug，显著优于所有非智能体方法，几乎是 StarCoder2-15B（Lozhkov et al., 2024; Li et al., 2023c）的两倍。SWE-Agent 虽达到 87.7%，但 Yang et al. (2024) 给了模型一个完整成功轨迹的演示（修复了测试集中某个 bug 的"1-shot"示范），而我们对 OpenHands 的评测是 0-shot。由于 HumanEvalFix 由人工构建且所有 bug 均经过仔细验证，在该基准上达到 100% 是完全可行的，这也是 OpenHands 未来迭代的追求目标。

**ML-Bench**（Tang et al., 2024b）评估智能体在 18 个 GitHub 仓库上解决机器学习任务的能力，包含 9641 个任务、169 个不同的 ML 问题，要求智能体根据用户指令生成 bash 脚本或 Python 代码。在沙箱环境中，智能体可以迭代执行命令并获得反馈，逐步理解仓库上下文并满足用户需求。遵循原论文设置，我们在 ML-Bench 的四分之一子集上进行评测。

**Gorilla APIBench**（Patil et al., 2023）评估智能体使用 API 的能力，涵盖 TorchHub、TensorHub 与 HuggingFace 任务。评测时给模型一个与 API 使用相关的问题，如"找出一个能把录音中语音转成文字的 API"。正确性依据模型 API 调用是否落在正确领域来判断。

**ToolQA**（Zhuang et al., 2024）评估智能体使用外部工具的能力，涵盖航班状态、咖啡价格、Yelp 数据、Airbnb 数据等主题，需要使用文本工具、数据库工具、数学工具、图工具、代码工具、系统工具等多种工具。它分 easy 与 hard 两级：easy 偏单工具使用，hard 强调推理。我们采用 easy 子集评测。

**BioCoder**（Tang et al., 2024c）是仓库级代码生成基准，评估智能体在生物信息学任务上检索并准确利用上下文的能力。原始提示包含相关代码上下文，但本研究中我们移除了上下文，以展示 OpenHands 在多轮交互中进行上下文检索、自调试与推理的能力。BioCoder 包含 157 个 Python 函数与 50 个 Java 函数，分别对应蛋白质组学、基因组学等生物信息学特定领域；它通过在现有仓库中把相关代码遮蔽后要求生成，来贴近真实代码。

**BIRD**（Li et al., 2023b）是面向真实、大规模数据库环境的 text-to-SQL 基准（即把自然语言翻译为可执行 SQL）。我们从 dev 集中选了 300 个样本集成进 OpenHands，以执行准确率评测。此外，我们扩展了设定：允许智能体进行多轮交互得出最终 SQL，使其能通过观察 SQL 执行结果来纠正历史错误。

### 4.3 网页浏览

网页浏览基准的评测结果见表 5。

**WebArena**（Zhou et al., 2023a）是一个可自托管、基于执行判分的网页智能体基准，允许智能体自由选择完成给定任务的路径。WebArena 包含 812 条人工整理的任务指令，覆盖购物、论坛、开发者平台与内容管理系统等多个领域。

**结果。** 由表 5 可见，我们的 BrowsingAgent 在使用领域通用提示技术（未做基准特化训练）的 LLM 智能体中取得了有竞争力的表现。

**MiniWoB++**（Liu et al., 2018）是带内置奖励函数的交互式网页基准，任务在 125 个极简网页界面上合成初始化。与 WebArena 不同，其任务更简单、页面不变化、步骤更少，并给出低层的分步指令。注意其中包含一部分需要视觉能力才能完成的任务，许多已有工作只聚焦任务子集（Kim et al., 2024; Li et al., 2023d; Shaw et al., 2023）。我们仍报告全集上的表现，且只纳入同样在全集上评测的基线。

> **表 5 | OpenHands 网页浏览评测结果（§4.3）。**

| 基准（实例数） | 智能体 | 模型 | 成功率 (%) | 平均成本 ($) |
| --- | --- | --- | --- | --- |
| **WebArena**（812） | Lemur (Xu et al., 2023) | Lemur-chat-70b | 5.3 | − |
| | Patel et al. (2024) | 自我提升合成数据训练的 72B | 9.4 | − |
| | AutoWebGLM (Lai et al., 2024) | 人/智能体混合标注训练的 7B | 18.2 | − |
| | Auto Eval & Refine (Pan et al., 2024) | GPT-4 + Reflexion w/ GPT-4V 奖励模型 | 20.2 | − |
| | WebArena Agent (Zhou et al., 2023a) | gpt-3.5-turbo | 6.2 | − |
| | | gpt-4-turbo | 14.4 | − |
| | OH BrowsingAgent v1.0 | gpt-4o-mini-2024-07-18 | 8.5 | 0.01 |
| | | gpt-4o-2024-05-13 | 14.8 | 0.15 |
| | | claude-3-5-sonnet-20240620 | 15.5 | 0.10 |
| | OH CodeActAgent v1.8（委派给 BrowsingAgent v1.0） | gpt-4o-mini-2024-07-18 | 8.3 | − |
| | | gpt-4o-2024-05-13 | 14.5 | − |
| | | claude-3-5-sonnet-20240620 | 15.3 | − |
| **MiniWoB++**（125 环境） | Workflow Guided Exploration (Liu et al., 2018) | 环境探索训练的专用模型 | 34.6 | − |
| | CC-NET (Humphreys et al., 2022) | RL + 人类标注 BC 训练的专用模型 | 91.1 | − |
| | OH BrowsingAgent v1.0 | gpt-3.5-turbo-0125 | 27.2 | 0.01 |
| | | gpt-4o-2024-05-13 | 40.8 | 0.05 |
| | OH CodeActAgent v1.8（委派给 BrowsingAgent v1.0） | gpt-4o-2024-05-13 | 39.8 | − |

### 4.4 综合辅助

综合辅助基准的结果见表 6。

**GAIA**（Mialon et al., 2023）评估智能体的通用任务解决能力，覆盖多种真实场景，需要推理、多模态理解、网页浏览与编码等多种能力。GAIA 包含 466 个人工精选任务，分三个难度级别。由于把各种工具与智能体集成的复杂性，搭建 GAIA 评测历来颇具挑战，而 OpenHands 的基础设施（如运行时 §2.2、工具 §2.3）大幅简化了这一集成。

**GPQA**（Rein et al., 2023）评估智能体在解决研究生级别难题时协调使用工具的能力。工具使用（如 Python）与网页搜索往往很有帮助：前者提供 LLM 常常不具备的精确计算能力，后者可访问 LLM 参数化知识库之外的信息。

> **表 6 | OpenHands 综合辅助评测结果（§4.4）。**

| 基准（实例数） | 智能体 | 模型 | 成功率 (%) | 平均成本 ($) |
| --- | --- | --- | --- | --- |
| **GAIA**（L1 验证集，53） | AutoGPT (Gravitas, 2023) | gpt-4-turbo | 13.2 | − |
| | OH GPTSwarm v1.0 | gpt-4-0125-preview | 30.2 | 0.110 |
| | | gpt-4o-2024-05-13 | 32.1 | 0.050 |
| **GPQA**（diamond 集，198；其他子集见 §G 表 7） | 人类 (Rein et al., 2023) | 专家人类 | 81.3 | − |
| | | 非专家人类 | 21.9 | − |
| | Few-shot Prompting + CoT (Rein et al., 2023) | gpt-3.5-turbo-16k | 29.6 | − |
| | | gpt-4 | 38.8 | − |
| | OH CodeActAgent v1.8 | claude-3-5-sonnet-20240620 | 52.0 | 0.065 |
| **AgentBench**（OS (bash) 子集，144） | AgentBench 基线智能体 (Liu et al., 2023) | gpt-4 | 42.4 | − |
| | | gpt-3.5-turbo | 32.6 | − |
| | OH CodeActAgent v1.5 | gpt-4o-2024-05-13 | 57.6 | 0.085 |
| | | gpt-3.5-turbo-0125 | 11.8 | 0.006 |
| **MINT**：math 子集（225） | MINT 基线智能体 (Wang et al., 2024b) | gpt-4-0613 | 65.8 | − |
| | OH CodeActAgent v1.5 | gpt-4o-2024-05-13 | 77.3 | 0.070 |
| | | gpt-3.5-turbo-16k-0613 | 33.8 | 0.048 |
| **MINT**：code 子集（136） | MINT 基线智能体 | gpt-4-0613 | 59.6 | − |
| | OH CodeActAgent v1.5 | gpt-4o-2024-05-13 | 50.0 | 0.087 |
| | | gpt-3.5-turbo-16k-0613 | 5.2 | 0.030 |
| **ProofWriter**（600） | Few-shot Prompting + CoT (Pan et al., 2023) | gpt4 | 68.1 | − |
| | Logic-LM (Pan et al., 2023) | gpt4 + 符号求解器 | 79.6 | − |
| | OH CodeActAgent v1.5 | gpt-4o-2024-05-13 | 78.8 | − |
| **Entity Deduction Arena**（200） | Zero-shot Prompting (Zhang et al., 2024a) | gpt-4-0314 | 40.0 | − |
| | | gpt-3.5-turbo-0613 | 27.0 | − |
| | OH CodeActAgent v1.5 | gpt-4o-2024-05-13 | 38.0 | − |
| | | gpt-3.5-turbo-16k-0613 | 24.0 | − |

**AgentBench**（Liu et al., 2023）评估智能体在多轮开放式生成设定下的推理与决策能力。我们选取有代码支撑的操作系统（OS）子集（144 个任务）。OpenHands 智能体以多轮方式用 bash 命令直接与任务专属操作系统交互，把交互与推理结合起来自动化完成任务。

**MINT**（Wang et al., 2024b）评估智能体通过多轮交互（使用工具与 GPT-4 模拟的自然语言反馈）解决困难任务的能力。我们使用 Yuan et al. (2024) 采用的 coding 与 math 子集，遵循原论文设定：允许智能体最多交互 5 轮、有两次提出解法的机会。

**ProofWriter**（Tafjord et al., 2021）是为评估 LLM 演绎推理能力而构建的合成数据集。与 Logic-LM（Pan et al., 2023）相同，我们聚焦最难的子集——600 个需要 5 跳推理的实例。为尽量减少语义解析中的潜在错误影响，我们使用 Logic-LM 提供的逻辑形式。

**Entity Deduction Arena（EDA）**（Zhang et al., 2024a）评估智能体通过策略性提问推断未知实体的能力，类似"二十个问题"游戏。该基准在多轮对话中测试智能体的状态追踪、策略规划与归纳推理能力。我们在"Things"与"Celebrities"两个数据集上评测（各 100 个实例），报告两个数据集上的平均成功率。

---

## 5 结论

我们介绍了 OpenHands——一个由社区驱动的平台，支持开发通过软件接口与世界交互的智能体。通过提供强大的交互机制、安全的沙箱环境、必备的智能体技能、多智能体协作能力与全面的评测框架，OpenHands 加速了智能体 AI 系统的研究创新与真实应用。尽管在开发安全可靠的智能体方面仍有挑战（§A），我们对蓬勃发展的社区充满期待，并期待 OpenHands 持续演进。

---

## 作者贡献

本工作是多机构共同参与的开源协作成果。我们采用积分制来确定贡献并授予作者署名，技术贡献以 pull request（PR）为单位进行追踪与计量。³ Xingyao Wang 领导项目，统筹整体开发与论文写作。具体分工如下：

> *（译者注）³ 原文脚注：详见 https://github.com/All-Hands-AI/OpenHands/pull/1917。*

- **智能体开发（§3）**：Xingyao Wang 主导 CodeAct（Wang et al., 2024a）与 CodeActSWE 智能体实现；Frank F. Xu 主导网页浏览智能体开发（Zhou et al., 2023a）；Mingchen Zhuge 统筹 GPTSwarm 智能体集成（Zhuge et al., 2024）；Robert Brennan 与 Boxuan Li 主导 Micro Agent 开发。
- **架构开发（图 2）**：Robert Brennan 发起架构设计；Boxuan Li、Frank F. Xu、Xingyao Wang、Yufan Song、Mingzhang Zheng 进一步完善扩展。Boxuan Li 实现了集成测试（§E）的初始版本、维护 agentskills 库（§2.3）、管理配置并解决了评测中的资源泄漏；Frank F. Xu 开发了网页浏览环境（§J，供智能体执行与评测使用），并将其与智能体和前端 UI 集成；Xingyao Wang 编写了 agentskills 库与 Docker 沙箱的初始代码；Yufan Song 实现了评测成本追踪，Mingzhang Zheng 开发了镜像无关的 docker 沙箱以稳定 SWE-Bench 评测。
- **基准集成与代码评审**：Boxuan Li 与 Yufan Song 主导基准集成（协调、评测、代码评审）；Yufan Song 还负责追踪 PR 贡献。Graham Neubig、Xingyao Wang、Mingzhang Zheng、Robert Brennan、Hoang H. Tran、Frank F. Xu、Xiangru Tang、Fuqiang Li、Yanjun Shao 参与集成与代码评审。各基准的具体贡献者：SWE-Bench：Bowen Li、Xingyao Wang；WebArena 与 MiniWoB++：Frank F. Xu；GAIA：Jiayi Pan（集成）、Mingchen Zhuge（GPTSwarm 评测）；API-Bench 与 ToolQA：Yueqi Song；HumanEvalFix：Niklas Muennighoff、Xiangru Tang；ProofWriter：Ren Ma；MINT：Hoang H. Tran；AgentBench：Fuqiang Li；BIRD：Binyuan Hui；GPQA：Jaskirat Singh；BioCoder：Xiangru Tang、Bill Qian；ML-Bench：Xiangru Tang、Yanjun Shao；Entity-Deduction-Arena：Yizhe Zhang。
- **指导（Advising）**：Graham Neubig 指导项目、提供资源并深度修改论文；Heng Ji 与 Hao Peng 提供项目建议并协助论文写作；Junyang Lin 提供指导支持并赞助资源。

---

## 参考文献

> 原文参考文献共 70 余条，保留英文原文不译（可查阅 arXiv 原文 §References），此处不重复罗列。

---

## 附录 A 局限与未来工作

我们为社区在 OpenHands 中打下的基础感到振奋，并期待其持续演进。我们确定了以下几个未来方向：

- **更强的多模态支持。** 当前实现已通过预定义智能体技能支持多种文件格式，我们希望通过标准 IPython 与浏览器集成以更有原则的方式支持多模态，例如通过浏览器用视觉语言模型查看图片和视频、用代码处理 XLSX 文件等。
- **更强的智能体。** 当前智能体在复杂任务上仍很吃力，我们希望通过训练时与推理时两方面的技术构建更强的智能体。
- **智能体编辑能力改进。** 当前智能体在编辑长文件时表现很差，我们希望探索不同方法提升智能体的文件编辑性能。
- **网页浏览改进。** 得益于 OpenHands 的可扩展架构，可以轻松集成正交的改进组件。例如，Auto Eval & Refine（Pan et al., 2024）——一种结合 Reflexion（Shinn et al., 2024）提示与任务完成奖励模型的出错重试策略——将作为可选组件挂载到我们的浏览智能体上。
- **自动工作流生成。** 目前 OpenHands 的工作流仍需要大量人工。我们认为 GPTSwarm（Zhuge et al., 2024）与 LangGraph（Chase, 2022）等图框架可作为构建智能体的备选方案。特别是 GPTSwarm，用图构建智能体后，更容易引入各种优化方法（如强化学习、元提示）。OpenHands 视这些方法为未来版本中自动工作流生成的有前景方案的基础。

## 附录 B 伦理声明

当今大多数 AI 智能体仍是研究产物，尚无法可靠地在真实世界完成复杂的长程任务。但随着性能持续提升、部署日益广泛，它们在提升生产力的同时也可能给社会带来安全风险。OpenHands 通过以下方式帮助降低风险：

1. 支持对智能体的系统化评测，在广泛部署之前识别并应对风险；
2. 促进人机交互，而非放任智能体在无监督下自主运行；
3. 更重要的是，我们希望 OpenHands 让全世界的研究者都能获得最好的智能体套件，开展面向构建安全、有益智能体的前沿安全研究。

## 附录 C 相关工作

ChatGPT（OpenAI, 2024a）与 GPT-4（OpenAI et al., 2024）等大语言模型的突破显著增强了各领域自主智能体的能力（Ye et al., 2023; Tang et al., 2024d; Park et al., 2023; Cui et al., 2023），催生了大量旨在执行多样化用户任务的通才智能体方案（Gravitas, 2023; Nakajima, 2023; Wu et al., 2023），并受到开发者与大众的关注。Auto-GPT（Gravitas, 2023）等代表工作通过把用户目标分解为可执行步骤来驱动 LLM 完成任务。多智能体协作系统利用 LLM 的角色扮演与任务求解能力（Zhuge et al., 2023; Li et al., 2023a; Zhou et al., 2023b; Team, 2023）：MetaGPT（Hong et al., 2023）强调标准化操作流程，AutoGen（Wu et al., 2023）提供交互系统的会话框架；AGENTS（Zhou et al., 2023b）与 AutoAgents（Chen et al., 2024）提出可定制智能体架构的新范式；XAgent（Team, 2023）与 GPTSwarm（Zhuge et al., 2024）分别引入复杂管理系统与可优化图来增强智能体运行。

智能体开发的浪潮也催生了旨在简化智能体实现的专用框架：LangChain 与 LangGraph（Chase, 2022）提供基础构件与基本运行时支持；CrewAI（CrewAI, 2024）聚焦多智能体通信编排；BrowserGym（ServiceNow）专门面向网页浏览能力；DSPy（Khattab et al., 2024）强调端到端提示优化。AutoGen（Wu et al., 2023）在基础框架之上实现了 Python 与 bash 执行能力（但命令执行是无状态的），而 CrewAI 等框架提供沙箱化但功能有限的代码解释器。

软件开发是 LLM 智能体应用的排头兵，相关框架不断涌现（Hong et al., 2023; Qian et al., 2023）：ChatDev（Qian et al., 2023）以类似瀑布模型的方式自动化软件开发生命周期；AutoCodeRover（Zhang et al., 2024b）通过代码搜索与抽象语法树操作解决 GitHub issue；AgentCoder（Huang et al., 2024）用集成测试与反馈迭代优化代码生成；SWE-Agent（Yang et al., 2024）将 LLM 集成用于自动化修复 GitHub issue，简化软件工程流程。

## 附录 D 图形用户界面

除命令行运行外，OpenHands 提供了丰富的图形用户界面，可视化智能体的当前动作（如浏览网页、执行 bash 命令或 Python 代码等），并支持用户实时反馈。UI 截图见图 1。用户可以在智能体工作过程中的任意时刻打断它，提供额外反馈、评论或指令。该 UI 与事件流（§2.1）直接连接以控制和可视化智能体与运行时，因此对智能体与运行时的具体实现是无关的（agnostic）。

## 附录 E 质量控制：面向智能体的集成测试

集成测试（Leung & White, 1990）长久以来被软件开发者用于保证软件质量。与输入输出模式简单的大语言模型不同，智能体通常是复杂的软件，开发过程中的微小差错就很容易引入并损害最终任务表现。虽然跑一整套评测（§4）是衡量性能退化的最终手段，但每次代码变更都跑一遍可能慢得离谱且成本高昂。⁴ 在 OpenHands 中，我们首创了一个端到端的智能体测试框架，测试提示词回归、动作与沙箱环境。它结合软件工程的集成测试与基础模型 mock（以获得确定性行为），防止智能体开发过程中意外引入 bug。

> *（译者注）⁴ 原文脚注：用 gpt-4o 跑一次 SWE-Bench Lite 评测约需 600 美元。*

**定义集成测试。** OpenHands 的集成测试框架通过自动化任务执行与结果验证来校验端到端功能。开发者定义任务与预期结果，例如任务可能是修正名为 "bad.txt" 文档中的拼写错误。通过 OpenHands 执行任务后，输出会与预定义的"金标准文件（gold file）"比对以确保正确性。

**Mock LLM 以获得确定性行为。** 为应对 LLM 的非确定性与高成本，框架拦截所有 LLM 调用，基于提示词精确匹配提供预定义响应。这既保证了测试结果的一致性，又因减少对真实 LLM 的依赖而降低了运行成本。

**破坏性变更时重新生成 LLM 响应。** 提示-响应对由脚本管理：当引入新测试或修改已有提示时，脚本生成并存储这些配对。常规测试时框架尝试通过微调提示词来复用已有 LLM 响应；影响任务处理的实质性变更则需用真实 LLM 重新生成配对。

**集成测试的收益。** 框架提供多项优势：1) **提示词回归测试**——存储的提示-响应对便于变更追踪，也为新成员理解 LLM 交互提供参考；2) **多平台支持**——每个 pull request 与 main 分支的每次提交都会自动调度测试，覆盖 Linux 与 Mac 多平台、多种环境与智能体，包括本地、SSH 与 exec 沙箱；3) **全面错误检测**——捕获提示词生成、消息传递与沙箱执行中的错误，维持高测试覆盖率。

## 附录 F OpenHands 运行时的工作原理

### F.1 工作流

OpenHands 运行时系统采用基于 Docker 容器的客户端-服务器架构（图 4）：

> **图 4 | OpenHands 运行时工作流。**

1. **用户输入**：用户提供自定义的基础 Docker 镜像；
2. **镜像构建**：OpenHands 基于用户镜像构建新镜像（"OH 运行时镜像"），其中包含 OpenHands 专用代码，主要是"运行时客户端"（即 §2.2 所述的运行时 API 服务器）；
3. **容器启动**：OpenHands 启动时，用 OH 运行时镜像启动 Docker 容器；
4. **通信**：OpenHands 后端（runtime.py）通过 RESTful API 与运行时客户端通信，发送动作、接收观察；
5. **动作执行**：运行时客户端接收后端的动作，在沙箱环境中执行并回传观察；
6. **观察返回**：客户端把执行结果作为观察发送回 OpenHands 后端的事件流。

**客户端的角色**：作为 OpenHands 后端与沙箱环境之间的中介；在容器内安全地执行各类动作（shell 命令、文件操作、Python 代码等）；管理沙箱环境的状态（当前工作目录、已加载插件等）；格式化并返回观察，为结果处理提供一致的接口。

### F.2 OpenHands 如何构建与维护运行时镜像

OpenHands 构建与管理运行时镜像的方式保证了生产与开发环境下 Docker 镜像创建与维护的效率、一致性与灵活性。

#### F.2.1 镜像标签系统

OpenHands 对运行时镜像使用双标签体系，以平衡可复现性与灵活性：

1. **哈希标签**：`{target_image_repo}:{target_image_hash_tag}`，如 `runtime:abc123def456`。
   - 基于构建目录（包含运行时客户端及相关依赖的源码、Dockerfile）的 MD5 哈希；
   - 相同哈希标签保证镜像由完全相同的源码与 Dockerfile 构建而成；
   - 这保证了可复现性：相同哈希永远意味着相同的镜像内容。
2. **通用标签**：`{target_image_repo}:{target_image_tag}`，如 `runtime:oh_v0.9.3_ubuntu_tag_22.04`。
   - 格式为 `runtime:oh_v{VERSION}_{BASE_IMAGE}_tag_{IMAGE_TAG}`；
   - 表示特定基础镜像与 OpenHands 版本组合的最新构建；
   - 只要同一基础镜像构建出新镜像（即使源码有变），该标签就会更新。

哈希标签保证可复现性，通用标签提供指向特定配置最新版本的稳定引用。这种双标签方式让 OpenHands 能高效管理开发与生产环境。

#### F.2.2 构建流程

1. **镜像命名规范**：哈希标签 `target_image_repo:target_image_hash_tag`（如 `runtime:abc123def456`）；通用标签 `target_image_repo:target_image_tag`（如 `runtime:oh_v0.9.3_ubuntu_tag_22.04`）。
2. **构建流程**：a) 把基础镜像名转换为 OH 运行时镜像名（如 `ubuntu:22.04` → `runtime:oh_v0.9.3_ubuntu_tag_22.04`）；b) 生成构建上下文（Dockerfile 与 OpenHands 源码）并计算其哈希；c) 查找是否已有该哈希的镜像；d) 若无，查找最近的兼容镜像作为基础；e) 若兼容镜像也不存在，用原始基础镜像从零构建；f) 给新镜像同时打上哈希标签与通用标签。
3. **镜像复用与重建逻辑**（针对用户提供的（基础）镜像，如 `ubuntu:22.04`）：a) 若存在相同哈希的镜像（如 `runtime:abc123def456`），直接复用；b) 若未找到精确哈希，尝试以最新的通用标签镜像（如 `runtime:oh_v0.9.3_ubuntu_tag_22.04`）为基础重建，借助既有依赖节省时间；c) 若两者都不存在，则完全从零构建。

**缓存与效率**。系统尽可能复用已有镜像以节省构建时间：找到精确哈希匹配则直接使用无需重建；找到兼容镜像则以其为基础重建，节省依赖安装时间。构建流程的流程图见图 5。

> **图 5 | OpenHands 运行时镜像构建流程。**

## 附录 G GPQA 基准的补充结果

GPQA 基准其他子集上的更详细结果见表 7。

> **表 7 | GPQA 基准（Rein et al., 2023）完整评测结果（§4.4）。**

| 评测方法与模型 | Diamond 集 (%) | Main 集 (%) | Extended 集 (%) | 平均成本 ($) |
| --- | --- | --- | --- | --- |
| 专家人类验证者 | 81.2 | 72.5 | 65.4 | N/A |
| 非专家人类验证者 | 21.9 | 30.5 | 33.9 | N/A |
| Few-Shot CoT Llama-2-70B-chat | 28.1 | 29.1 | 30.4 | N/A |
| Few-Shot CoT GPT-3.5-turbo-16k | 29.6 | 28.0 | 28.2 | N/A |
| Few-Shot CoT GPT-4 | 38.8 | 39.7 | 38.7 | N/A |
| GPT-4 + 搜索（弃权时回退到 CoT） | 38.8 | 41.0 | 39.4 | N/A |
| OpenHands + CodeActAgent v1.5 + GPT-3.5-turbo | 27.9 | 23.4 | 26.1 | 0.012 |
| OpenHands + CodeActAgent v1.5 + GPT-4-turbo | 51.8 | 47.4 | 42.4 | 0.501 |
| OpenHands + CodeActAgent v1.5 + GPT-4o | 53.1 | 49.3 | 52.8 | 0.054 |

## 附录 H CodeActSWEAgent 的上下文演示

**要点提炼**：该附录说明 CodeActSWEAgent 的提示词直接复用自 SWE-agent 官方发布的轨迹演示（princeton-nlp/SWE-agent 仓库 `trajectories/demonstrations` 目录），提示词代码可在 OpenHands 仓库 `agenthub/codeact_swe_agent/prompt.py` 查看。即：SWE 专才智能体不做自研提示，而是沿用 SWE-agent 的成熟提示模板，保证与 SWE-Agent 基线可比。

## 附录 I 支持的 AgentSkills（要点提炼）

**要点提炼**：截至 OpenHands v0.6，AgentSkills 库提供以下技能（原文为带 docstring 的完整 Python 函数签名，此处按功能归类归纳；最新列表见仓库 `OpenHands/runtime/plugins/agent_skills/agentskills.py`）：

| 类别 | 函数 | 功能 |
| --- | --- | --- |
| 文件浏览 | `open_file(path, line_number)` | 在编辑器中打开文件，可定位到指定行 |
| | `goto_line(line_number)` | 窗口跳转到指定行 |
| | `scroll_down()` / `scroll_up()` | 窗口向下/向上滚动 100 行 |
| 文件创建/编辑 | `create_file(filename)` | 创建并打开新文件 |
| | `edit_file(start, end, content)` | 用 content 替换打开文件的第 start 到 end 行（含端点） |
| 搜索 | `search_dir(search_term, dir_path)` | 在目录所有文件中搜索关键词 |
| | `search_file(search_term, file_path)` | 在文件内搜索关键词 |
| | `find_file(file_name, dir_path)` | 在目录中按文件名查找 |
| 多模态解析 | `parse_pdf(file_path)` | 解析 PDF 内容并打印 |
| | `parse_docx(file_path)` | 解析 DOCX 内容并打印 |
| | `parse_latex(file_path)` | 解析 LaTeX 文件并打印 |
| | `parse_audio(file_path, model='whisper-1')` | 调用语音模型（默认 whisper-1）转写音频 |
| | `parse_image(file_path, task)` | 调用视觉语言模型解析图像（默认任务：尽可能详细描述图像） |
| | `parse_video(file_path, task, frame_interval=30)` | 按帧间隔抽帧后用视觉语言模型解析视频 |
| | `parse_pptx(file_path)` | 解析 PPTX 内容并打印 |

**设计观察**：除多模态解析需要调用外部模型外，其余全部是"LLM 不便直接写代码完成"的文件操作封装（正是 §2.3 的两条收录标准的体现）。

## 附录 J BrowserGym 动作（要点提炼）

**要点提炼**：原文列出了 BrowserGym v0.3.4 中全部浏览动作原语的 Python 签名与示例（定义于 browsergym/core 的 action/functions.py）。OpenHands 把它们作为浏览动作的通用原语，可按需只启用其中子集。按功能分类如下：

| 类别 | 动作 | 说明 |
| --- | --- | --- |
| 智能体控制 | `send_msg_to_user(text)` | 向用户发送消息（如汇报搜索结论） |
| | `report_infeasible(reason)` | 告知用户指令无法执行（如页面没有邮箱输入框） |
| | `noop(wait_ms=1000)` | 什么都不做，可等待指定毫秒 |
| 表单/元素交互 | `fill(bid, value)` | 填表（适用 `<input>`、`<textarea>`、contenteditable） |
| | `check(bid)` / `uncheck(bid)` | 勾选/取消复选框或单选框 |
| | `select_option(bid, options)` | 在 `<select>` 中选择一个或多个选项 |
| | `click` / `dblclick` / `hover` | 单击 / 双击 / 悬停（可指定鼠标键与修饰键） |
| | `press(bid, key_comb)` | 按键（如 `Control+a`、`Meta+Shift+t`） |
| | `focus(bid)` / `clear(bid)` | 聚焦元素 / 清空输入 |
| | `drag_and_drop(from_bid, to_bid)` | 拖拽元素 |
| | `upload_file(bid, file)` | 上传文件（支持多文件） |
| 导航 | `go_back()` / `go_forward()` / `goto(url)` | 后退 / 前进 / 跳转 URL |
| 滚动 | `scroll(delta_x, delta_y)` | 按像素滚动页面 |

其中 `bid` 是 BrowserGym 给页面元素分配的边界框标识符（bounding-box id），智能体通过可访问性树中带 `[8]`、`[b22]` 等编号的元素来引用并操作页面元素。

## 附录 K 浏览智能体细节（要点提炼）

**要点提炼**：该附录给出 Browsing Agent 使用的完整零样本提示示例。核心结构：

1. **指令**：审查页面当前状态与所有信息，找出完成目标的最佳下一步动作；输出会被程序解析执行，必须严格遵守格式。
2. **目标示例**：如"浏览 localhost:8000，告诉我生命的终极答案，任何时刻都不要向我确认"。
3. **动作空间**：列出附录 J 的全部 16 种动作签名与示例。
4. **关键技巧——一步多动作**：提示中明确允许智能体在一次回复中给出多个动作，前提是这些动作应被顺序执行且**不需要**页面反馈（例如登录页填用户名、填密码、点提交）；需要观察页面反馈时则不可合并。这为"同一页面上连续动作"的常见工作流节省了交互轮数。
5. **当前状态输入**：提供页面的可访问性树（如 `RootWebArea 'The Ultimate Answer'`、`[10] button 'Click me', clickable`）与历史动作列表（如 `goto('http://localhost:8000')`）。
6. **期望输出格式**：先写一句思维链（"为了达成目标我需要点击 bid 为 10 的按钮"），再在代码块中写动作（`click("10")`）。
7. **WebArena 特化补充**：由于部分 WebArena 任务要求对智能体回给用户的答案做精确匹配，评测时在提示中追加指令，要求 `send_msg_to_user` 只回简洁的答案字符串（如 `"$279.49"`），避免多余文字导致判分失败。

---

## 术语与背景速览（译者整理）

| 术语 | 含义 |
| --- | --- |
| OpenHands / OpenDevin | 前者为本平台现名，后者为原名（致敬 Cognition 的 Devin 后更名），MIT 许可开源 |
| Event Stream（事件流） | 按时间顺序记录所有动作与观察的核心数据结构，UI、智能体、环境三方通过它交互 |
| Agent Runtime | 每个任务会话启动的 docker 沙箱 + 内部 REST API 服务器，执行动作、返回观察 |
| ACI（Agent-Computer Interface） | 智能体-计算机接口，源自 SWE-Agent 论文，指为智能体量身设计的工具层 |
| AgentSkills | OpenHands 的标准化 Python 工具库，自动注入 IPython 环境，跨智能体复用 |
| CodeAct | 用"执行代码"作为统一动作空间的智能体范式（Wang et al., 2024a），OpenHands 默认智能体基于它 |
| AgentDelegateAction | 多智能体委派动作，如通才智能体把浏览子任务交给专职 BrowsingAgent |
| AgentHub | 社区贡献智能体的集散地，含 CodeActAgent、BrowsingAgent、GPTSwarm、Micro Agents 等 10+ 智能体 |
| BrowserGym | ServiceNow 的网页交互动作原语库，定义 fill/click/goto 等浏览 DSL |
| 可访问性树（accessibility tree） | 浏览器页面的结构化无障碍表示，网页智能体的主要观察形态 |
| hint text | SWE-Bench 附带的解题自然语言提示，本文所有结果均不使用 |
| Mock LLM 测试 | 集成测试框架按提示精确匹配回放预置 LLM 响应，保证测试确定性并省钱 |
| 双标签镜像体系 | 哈希标签保证可复现，通用标签指向最新构建，兼顾开发效率与一致性 |

**核心数字速记**

- **规模**：GitHub 32K star；188+ 贡献者；2100+ 次贡献；10+ 个社区智能体；15 个集成基准
- **SWE-Bench Lite**（300 实例，无 hint）：CodeActAgent v1.8 + claude-3.5-sonnet 达 **26.0%**（$1.10/实例），同期 SWE-Agent 18.0%、Aider 26.3%、Agentless 27.3%；gpt-4o-mini 仅 $0.01/实例
- **HumanEvalFix**（Python，0-shot）：CodeActAgent v1.5 + gpt-4o 修复 **79.3%**，接近 1-shot SWE-Agent 的 87.7%，约为 StarCoder2-15B（48.6%）的 1.6 倍
- **WebArena**：BrowsingAgent + claude-3-5-sonnet **15.5%**，超过 WebArena 官方 gpt-4-turbo（14.4%）
- **GPQA diamond**：CodeActAgent v1.8 + claude-3-5-sonnet **52.0%**，超 GPT-4 CoT（38.8%），逼近非专家人类（21.9%）两倍以上、专家人类（81.3%）以下
- **MiniWoB++**：BrowsingAgent + gpt-4o **40.8%**（全集，含需视觉任务）
- **GAIA L1**：GPTSwarm v1.0 + gpt-4o **32.1%** vs AutoGPT 13.2%
- **ML-Bench**：CodeActAgent v1.5 + gpt-4o **76.5%**，远超 SWE-Agent（42.6%）与 Aider（64.4%）
- **成本量级**：完整 SWE-Bench（2294 实例）约 $6.9K；单次 SWE-Bench Lite 评测（gpt-4o）约 $600
