# Voyager：一个由大语言模型驱动的开放式具身智能体

> **原文**: Voyager: An Open-Ended Embodied Agent with Large Language Models
> **作者**: Guanzhi Wang\*, Yuqi Xie\*, Yunfan Jiang, Ajay Mandlekar, Chaowei Xiao, Yuke Zhu, Linxi "Jim" Fan†, Anima Anandkumar†（\*同等贡献，†同等指导）
> **机构**: NVIDIA、Caltech、UT Austin、Stanford、UW Madison
> **日期/arXiv**: arXiv:2305.16291v2，2023-05-25 首次提交，2023-10-19 修订（v2）
> **链接**: https://arxiv.org/abs/2305.16291 ｜ 项目主页/代码: https://voyager.minedojo.org
> **译者说明**: 本文为全文翻译（含正文、全部图注表注、附录 A.1–A.6 与 B.1–B.4）。附录中体量极大的"完整提示词"（Prompt 1–7）与代码样例按内容概括为导览，不做逐字翻译；带 *（译者注）* 标记的段落为译者补充的背景说明，非原文内容。

---

## 摘要

我们提出 Voyager，这是首个由大语言模型（LLM）驱动的、在 Minecraft 中运行的具身终身学习智能体：它能够持续探索世界、习得多样化技能，并在无人工干预的情况下做出新发现。Voyager 由三个关键组件构成：1）**自动课程（automatic curriculum）**，用于最大化探索；2）**不断增长的技能库（skill library）**，由可执行代码构成，用于存储与检索复杂行为；3）**一种新的迭代提示机制（iterative prompting mechanism）**，融合环境反馈、执行错误与自我验证来改进程序。

Voyager 通过黑盒查询与 GPT-4 交互，从而绕过了对模型参数微调的需求。Voyager 所发展的技能具有时间上可延展（temporally extended）、可解释、可组合的特性，这使其能力得以快速复利式增长，并缓解了灾难性遗忘。实证上，Voyager 展现出强大的上下文内（in-context）终身学习能力，在 Minecraft 中表现出卓越的熟练度：相比此前的最优方法（SOTA），它获得的独特物品种类多 **3.3 倍**，行走距离长 **2.3 倍**，解锁关键技术树里程碑的速度快至 **15.3 倍**。Voyager 还能在新的 Minecraft 世界中利用已学到的技能库从零解决新任务，而其他方法难以泛化。我们已开源完整代码库与提示词。

> **图 1 | Voyager 通过自我驱动的探索不断发现 Minecraft 中的新物品与新技能，显著超越各基线方法。** 横轴为提示迭代次数（0–160），纵轴为累计发现的独特物品种类数（0–60）。图上方标注科技树里程碑（木制 → 石制 → 铁制 → 钻石工具），曲线上散布着对应已发现物品的图标。图例共 5 条：Voyager (Ours)（橙色）、Voyager w/o Skill Library（蓝色）、ReAct（绿色）、Reflexion（红色）、AutoGPT（紫色）。曲线终值约为：Voyager ≈ 63（且唯一解锁钻石工具）、无技能库版 ≈ 38、AutoGPT ≈ 22、ReAct 与 Reflexion 几乎停滞在 4–5。

---

## 1 引言

构建能够在开放式世界中持续探索、规划并发展新技能的通用具身智能体，是 AI 领域的一项宏大挑战 [1, 2, 3, 4, 5]。经典方法采用强化学习（RL）[6, 7] 与模仿学习 [8, 9, 10]，它们作用于原子级动作（primitive actions），在系统性探索 [11, 12, 13, 14, 15]、可解释性 [16, 17, 18] 与泛化性 [19, 20, 21] 方面都面临困难。近期，基于大语言模型（LLM）的智能体利用预训练 LLM 中蕴含的世界知识，生成连贯的行动计划或可执行策略 [16, 22, 19]，被应用于游戏与机器人等具身任务 [23, 24, 25, 26, 27]，以及无实体的 NLP 任务 [28, 29, 30]。然而，这些智能体**不是终身学习者**——它们无法在长时间跨度上逐步习得、更新、积累并迁移知识 [31, 32]。

让我们以 Minecraft 为例。与 AI 研究中大多数其他游戏 [33, 34, 10] 不同，Minecraft 并不设定预定义的终局目标或固定剧情，而是提供了一个拥有无限可能性的独特试验场 [23]。Minecraft 要求玩家探索广阔的程序化生成 3D 地形，并利用采集到的资源解锁科技树。人类玩家通常从学习基础操作（如伐木、烹饪食物）开始，再进阶到更复杂的任务（如与怪物战斗、制作钻石工具）。

我们认为，一个有效的终身学习智能体应具备与人类玩家类似的能力：（1）根据自身当前技能水平与世界状态**提出合适的任务**——例如若身处沙漠而非森林，应学会先收集沙子与仙人掌，而不是铁；（2）依据环境反馈**精炼技能**，并把已掌握的技能写入记忆，以便将来在相似情境中复用（例如打僵尸与打蜘蛛是相似的）；（3）以自我驱动的方式**持续探索世界并寻找新任务**。

> **图 2 | Voyager 由三个关键组件构成：用于开放式探索的自动课程、用于承载日益复杂行为的技能库，以及以代码为动作空间的迭代提示机制。** 图分三栏：左栏"自动课程"展示任务序列的游戏画面快照（Mine Wood Log → Make Crafting Table → Combat Zombie → Mine Diamond），并通过"Update Exploration Progress"回环更新探索进度；中栏"迭代提示机制"展示生成的代码与环境画面，环境反馈/执行错误与"Code as Actions"汇入自我验证（✗ 触发"Refine Program"回到上一轮，✓ 触发"Add New Skill"）；右栏"技能库"列出已掌握技能（Mine Wood Log、Make Crafting Table、Craft Stone Sword、Make Furnace、Craft Shield、Cook Steak、Combat Zombie），新任务经"Skill Retrieval"从中检索相关技能。

为实现这些目标，我们提出 Voyager：首个由 LLM 驱动的具身终身学习智能体，在 Minecraft 中无需人工干预即可驱动探索、掌握广泛技能并持续做出新发现。Voyager 依靠三个关键模块实现（图 2）：1）最大化探索的自动课程；2）存储与检索复杂行为的技能库；3）生成可执行代码用于具身控制的新型迭代提示机制。我们选择**以代码作为动作空间**而非底层运动指令，因为程序天然能够表达时间上延展且可组合的动作 [16, 22]，而这对 Minecraft 中众多长时程（long-horizon）任务至关重要。Voyager 通过提示与上下文学习 [36, 37, 38] 与黑盒 LLM（GPT-4 [35]）交互；这一路线无需访问模型参数，也无需显式的基于梯度的训练或微调。

更具体地说，Voyager 试图解决由自动课程提出的、难度逐步提升的任务，而课程会考虑探索进度与智能体状态。课程由 GPT-4 依据"尽可能发现更多样的事物"这一总体目标生成。该方法可被视为一种上下文内（in-context）形式的新颖性搜索（novelty search）[39, 40]。Voyager 通过存储成功求解任务的行动程序来增量地构建技能库；每个程序以其描述的嵌入向量索引，从而在未来相似情境中被检索。复杂技能可由更简单的程序组合而成，这让 Voyager 的能力随时间快速复利式增长，并缓解了其他持续学习方法中的灾难性遗忘 [31, 32]。

然而，LLM 难以稳定地一次性生成正确的动作代码 [41]。为应对这一挑战，我们提出迭代提示机制：（1）执行生成的程序，从 Minecraft 仿真中获取观测（如物品栏列表、附近生物），并从代码解释器获取错误追踪（若有）；（2）把这些反馈注入 GPT-4 的提示，进行下一轮代码精炼；（3）重复该过程，直到自我验证模块确认任务完成——此时把程序提交到技能库（例如 `craftStoneShovel()`、`combatZombieWithSword()`），并向自动课程请求下一个里程碑（图 2）。

实证上，Voyager 展现出强大的上下文内终身学习能力：它能构建一个不断增长的行动程序技能库，这些程序可复用、可解释，并能泛化到新任务。我们在开源 Minecraft AI 框架 MineDojo [23] 中，将 Voyager 与其他基于 LLM 的智能体技术（如 ReAct [29]、Reflexion [30]、AutoGPT [28]）进行了系统对比。Voyager 超越此前 SOTA：独特物品多 3.3 倍，解锁关键技术树里程碑快至 15.3 倍，行走距离长 2.3 倍。我们进一步证明，Voyager 能在新的 Minecraft 世界中利用已学技能库从零解决新任务，而其他方法难以泛化。

---

## 2 方法

Voyager 由三个新颖组件构成：（1）自动课程（2.1 节），为开放式探索建议目标；（2）技能库（2.2 节），用于发展日益复杂的行为；（3）迭代提示机制（2.3 节），生成用于具身控制的可执行代码。完整提示词见附录 A 节。

### 2.1 自动课程

> **图 3 | 自动课程提出的任务。** 为简洁起见仅展示部分提示，完整提示结构见附录 A.3 节。图中左侧是拼接后的 system+user 提示片段（总体目标与约束、智能体状态字段），右侧是 GPT-4 输出的"先推理、后给任务"内容，可见其按当前资源与进度挑选下一步该做的事。

开放式环境中的具身智能体会遇到复杂度各异的众多目标。自动课程为开放式探索带来诸多好处：保证学习过程既具挑战性又可完成，培育由好奇心驱动的内在动机，并鼓励发展通用而灵活的问题求解策略 [42, 43, 44]。我们的自动课程利用 GPT-4 内含的互联网规模知识，通过提示让它持续提供新任务或新挑战。课程以自底向上的方式展开，对探索进度与智能体当前状态有很强的适应性与响应性（图 3）。随着 Voyager 逐步推进到更难的自我驱动目标，它自然会学到多种技能，例如"开采钻石"。

输入 GPT-4 的提示包含若干组成部分：

1. **指令（Directives）**：鼓励多样化行为并施加约束，例如"我的终极目标是尽可能发现更多样的事物……下一个任务不应太难，因为我可能还没有必要的资源，或尚未学会足够的技能来完成它。"
2. **智能体当前状态**：包括物品栏（inventory）、装备、附近方块与实体、生物群系（biome）、时间、生命值与饥饿值条、以及位置。
3. **此前已完成与失败的任务**：反映智能体当前的探索进度与能力前沿。
4. **附加上下文（Additional context）**：我们还利用 GPT-3.5 基于智能体当前状态与探索进度自问自答。出于预算考虑，标准 NLP 任务上我们选择 GPT-3.5 而非 GPT-4。

### 2.2 技能库

> **图 4 | 技能库。上：添加新技能。** 每当 GPT-4 生成并验证一个新技能，我们就把它加入技能库（表示为向量数据库）：键是程序描述的嵌入向量（由 GPT-3.5 生成），值是程序本身。**下：技能检索。** 面对自动课程提出的新任务时，我们先用 GPT-3.5 生成求解该任务的一般性建议，它与环境反馈一起组成查询上下文，随后执行查询以找出 top-5 相关技能。

在自动课程持续提出日益复杂任务的同时，必须有一个技能库作为学习与演化的基础。受程序的通用性、可解释性与普适性启发 [45]，我们用可执行代码来表示每项技能，由它为自动课程提出的特定任务搭建时间上延展的动作序列。

输入 GPT-4 的提示包含以下组成部分：

1. **代码生成准则**，例如"你的函数将被复用来构建更复杂的函数，因此你应使其通用且可复用。"
2. **控制原语 API**，以及从技能库检索到的相关技能——这对上下文学习 [36, 37, 38] 发挥效果至关重要。
3. **上一轮生成的代码、环境反馈、执行错误与批评（critique）**，GPT-4 据此自我改进（2.3 节）。
4. **智能体当前状态**：物品栏、装备、附近方块与实体、生物群系、时间、生命值与饥饿值、位置。
5. **思维链提示（chain-of-thought prompting）**[46]，在生成代码前先进行推理。

我们通过新颖的迭代提示机制反复精炼程序（2.3 节），将其作为新技能加入技能库，并用其描述的嵌入向量建立索引（图 4 上）。技能检索时，我们用自行生成的任务计划与环境反馈的嵌入向量查询技能库（图 4 下）。通过持续扩展与精炼技能库，Voyager 能够学习、适应并在广泛任务中表现出色，不断拓展其在开放世界中的能力边界。

### 2.3 迭代提示机制

> **图 5 | 左：环境反馈。** GPT-4 意识到在制作木棍前还需要 2 块木板。**右：执行错误。** GPT-4 意识到它应当制作木斧而非"金合欢木斧"，因为 Minecraft 中不存在金合欢木斧这一物品。为简洁起见仅展示部分提示，代码生成的完整提示结构见附录 A.4 节。

我们提出一种迭代提示机制，通过三类反馈实现自我改进：

1. **环境反馈（environment feedback）**，刻画程序执行的中间进展（图 5 左）。例如"I cannot make an iron chestplate because I need: 7 more iron ingots"（我无法制作铁胸甲，因为我还需要 7 个铁锭）指出了制作失败的原因。我们在控制原语 API 内部使用 `bot.chat()` 来产生环境反馈，并在提示中要求 GPT-4 在代码生成时也使用该函数。
2. **执行错误（execution errors）**，来自程序解释器，揭示程序中的非法操作或语法错误，对修复 bug 极有价值（图 5 右）。
3. **自我验证（self-verification）**，用于检查任务是否成功。我们没有为自动课程提出的每个新任务手工编写成功检查器，而是实例化另一个 GPT-4 智能体来做自我验证：把 Voyager 的当前状态与任务交给 GPT-4，要求它扮演评论家（critic）[47, 48, 49]，告知程序是否完成了任务；若任务失败，它还给出应如何完成任务的批评（图 6）。因此，我们的自我验证不只是自我反思（self-reflection）[30]，而是同时检查成功与反思错误，因而更为全面。

在每一轮代码生成中，我们执行生成的程序以获得环境反馈与代码解释器的执行错误，并把它们注入 GPT-4 的提示，用于下一轮代码精炼。该迭代过程重复进行，直到自我验证确认任务完成——此时我们把这一新技能加入技能库，并向自动课程请求新目标（图 2）。若智能体在 4 轮代码生成后仍卡住，我们就向课程请求另一个任务。这种迭代提示方法显著提升了面向具身控制的程序合成能力，使 Voyager 能在无人工干预的情况下持续习得多样化技能。

> **图 6 | 自我验证示例。** 为简洁起见仅展示部分提示，完整提示结构见附录 A.5 节。图中展示评论家智能体的输出格式：先给出任务是否完成的推理，再输出布尔变量 `success`，若失败则给出 `critique`（批评内容形如"你当前缺少某材料，建议先完成其前置步骤"的可执行建议）。

---

## 3 实验

### 3.1 实验设置

我们使用 OpenAI 的 `gpt-4-0314` [35] 与 `gpt-3.5-turbo-0301` [50] API 进行文本补全，并使用 `text-embedding-ada-002` [51] API 做文本嵌入。除自动课程使用 temperature = 0.1 以鼓励任务多样性外，其余 temperature 均设为 0。仿真环境构建在 MineDojo [23] 之上，并使用 Mineflayer [52] 的 JavaScript API 做运动控制。更多细节见附录 B.1 节。

### 3.2 基线方法

由于没有开箱即用、可直接用于 Minecraft 的 LLM 智能体，我们尽力选取了若干代表性算法作为基线。这些方法原本只为无实体的 NLP 任务设计，因此我们必须重新诠释它们，使其可在 MineDojo 中执行并与我们的实验设定兼容：

- **ReAct** [29]：使用思维链提示 [46]，由 LLM 同时生成推理轨迹与行动计划。我们向它提供环境反馈与智能体状态作为观测。
- **Reflexion** [30]：构建在 ReAct [29] 之上，加入自我反思以推断更直观的后续动作。我们向它提供执行错误与我们的自我验证模块。
- **AutoGPT** [28]：一个流行的软件工具，通过把高层目标分解为多个子目标并在 ReAct 风格的循环中执行来自动化 NLP 任务。我们用 GPT-4 做任务分解来重新实现 AutoGPT，并向其提供智能体状态、环境反馈与执行错误作为子目标执行的观测。与 Voyager 相比，AutoGPT 缺少用于积累知识的技能库、用于评估任务成功的自我验证，以及用于开放式探索的自动课程。

注意，我们**不**与那些以 Minecraft 屏幕像素为输入、输出底层控制的方法直接比较 [53, 54, 55]。这不是苹果对苹果的比较，因为我们依赖高层的 Mineflayer [52] API 控制智能体。本文的重点是推进 GPT-4 在具身终身学习上的极限，而不是解决 3D 感知或感觉运动控制问题。Voyager 与之正交，只要控制器提供代码 API，就可以与 VPT [8] 这类基于梯度的方法结合。我们在表 A.2 中给出了 Voyager 与此前 Minecraft 智能体的系统级比较。

### 3.3 评估结果

我们从探索表现、科技树掌握度、地图覆盖范围，以及在新世界中对新任务的零样本泛化能力四方面，系统评估 Voyager 与各基线。

**显著更好的探索。** 探索表现结果见图 1。Voyager 的优势体现在它能够持续取得新进展：在 160 次提示迭代内发现 63 种独特物品，是同类方法 3.3 倍的新颖物品数。相比之下，AutoGPT 在发现新物品上明显落后；而 ReAct 与 Reflexion 由于开放式探索目标过于抽象、缺少合适课程难以执行，几乎无法取得实质进展。

**稳定的科技树掌握。** Minecraft 科技树检验智能体制作与使用层级化工具的能力。沿这棵树推进（木制工具 → 石制工具 → 铁制工具 → 钻石工具）要求智能体掌握系统化与可组合的技能。与基线相比，Voyager 解锁木制层级快 **15.3 倍**（按提示迭代次数计）、石制层级快 **8.5 倍**、铁制层级快 **6.4 倍**，并且是唯一解锁钻石层级的方法（图 2、表 1）。这凸显了自动课程的有效性——它持续呈现复杂度恰当的挑战以推动智能体进步。

**广泛的地图遍历。** Voyager 能穿越多种地形，行进距离是基线的 **2.3 倍**；而基线智能体常常局限于局部区域，严重限制了它们发现新知识的能力（图 7）。

> **表 1 | 科技树掌握度。** 分数表示三次运行中的成功次数。0/3 表示该方法在最大提示迭代次数（160）内未能解锁该层级。数值为三次试验平均的提示迭代次数，越少越高效（加粗表示最优）。

| 方法 | 木制工具 | 石制工具 | 铁制工具 | 钻石工具 |
| --- | --- | --- | --- | --- |
| ReAct [29] | N/A (0/3) | N/A (0/3) | N/A (0/3) | N/A (0/3) |
| Reflexion [30] | N/A (0/3) | N/A (0/3) | N/A (0/3) | N/A (0/3) |
| AutoGPT [28] | 92 ± 72 (3/3) | 94 ± 72 (3/3) | 135 ± 103 (3/3) | N/A (0/3) |
| Voyager（无技能库） | **7 ± 2** (3/3) | **9 ± 4** (3/3) | 29 ± 11 (3/3) | N/A (0/3) |
| Voyager（本文） | **6 ± 2** (3/3) | **11 ± 2** (3/3) | **21 ± 7** (3/3) | **102** (1/3) |

> **图 7 | 地图覆盖：Minecraft 地图的鸟瞰图。** 图为两张地图（两次试验）的俯视截图，每个方法以半透明圆圈标出其轨迹的最小外接范围：橙色为 Voyager (Ours)、绿色为 ReAct、暗红色为 Reflexion、紫色为 AutoGPT。Voyager 的覆盖圆远大于基线（2.3× 距离），基线的小圆彼此挤在出生点附近、互相嵌套在 Voyager 的范围内。

**对新任务的高效零样本泛化。** 为评估零样本泛化，我们清空智能体物品栏、将其重置到新实例化的世界，并用未见过的任务测试它。对 Voyager 与 AutoGPT，我们都用 GPT-4 把任务分解为一系列子目标。表 2 与图 8 显示：Voyager 能稳定求解所有任务，而基线在 50 次提示迭代内无法完成任何任务。有趣的是，由终身学习构建的技能库不仅提升了 Voyager 的表现，也给 AutoGPT 带来了增益。这说明技能库是一种通用工具，可被其他方法直接使用，实际上充当了即插即用的性能增强资产。

> **表 2 | 对未见任务的零样本泛化。** 分数表示三次尝试中的成功次数，0/3 表示该方法在最大提示迭代次数（50）内未能完成任务。数值为三次试验平均的提示迭代次数，越少越高效。

| 方法 | 钻石镐 | 金剑 | 熔岩桶 | 指南针 |
| --- | --- | --- | --- | --- |
| ReAct [29] | N/A (0/3) | N/A (0/3) | N/A (0/3) | N/A (0/3) |
| Reflexion [30] | N/A (0/3) | N/A (0/3) | N/A (0/3) | N/A (0/3) |
| AutoGPT [28] | N/A (0/3) | N/A (0/3) | N/A (0/3) | N/A (0/3) |
| AutoGPT [28] + 我们的技能库 | 39 (1/3) | 30 (1/3) | N/A (0/3) | 30 (2/3) |
| Voyager（无技能库） | 36 (2/3) | 30 ± 9 (3/3) | 27 ± 9 (3/3) | 26 ± 3 (3/3) |
| Voyager（本文） | **19 ± 3** (3/3) | **18 ± 7** (3/3) | **21 ± 5** (3/3) | **18 ± 2** (3/3) |

> **图 8 | 对未见任务的零样本泛化。** 我们可视化各方法在两个任务上的中间进展（另两个任务的曲线见附录 A.3 节）。ReAct 与 Reflexion 因没有取得任何有意义的进展而未绘制。

### 3.4 消融实验

我们对 Voyager 的 6 项设计选择（自动课程、技能库、环境反馈、执行错误、自我验证、以及用于代码生成的 GPT-4）做消融，研究它们对探索表现的影响（各变体细节见附录 B.3 节）。结果见图 9，关键发现如下：

- **自动课程对智能体持续进步至关重要。** 若把课程替换为随机课程，发现的物品数下降 **93%**，因为某些任务若顺序错乱则过于困难。另一方面，人工设计的课程需要大量 Minecraft 领域专业知识，且不考虑智能体的实时状况，在实验结果中不如我们的自动课程。
- **无技能库的 Voyager 在后期呈现停滞趋势。** 这凸显了技能库在 Voyager 中的关键作用：它帮助构建更复杂的动作，通过鼓励在旧技能之上建立新技能来稳步推进能力边界。
- **自我验证是所有反馈类型中最重要的。** 移除该模块导致发现物品数显著下降（**−73%**）。自我验证是决定"何时转入新任务、何时重试此前失败任务"的关键机制。
- **GPT-4 在代码生成上显著优于 GPT-3.5**，获得的独特物品多 **5.7 倍**，因为 GPT-4 的编码能力有质的飞跃。这一发现与近期文献 [56, 57] 一致。

> **图 9 | 左：对自动课程、技能库与 GPT-4 的消融。** 纵轴为独特物品数，横轴为代码生成的提示迭代次数。图例：Ours（橙色）、w/o Skill Library（蓝色）、GPT-3.5（紫色虚线，指用 GPT-3.5 替代 GPT-4 做代码生成）、Manual Curriculum（黄色）、Random Curriculum（暗红色，终值仅 ≈4，对应 −93%）。"Ours"明显高于所有替代方案。**右：对迭代提示机制的消融。** 图例：Ours、w/o Environment Feedback、w/o Execution Error、w/o Self-Verification；去掉任一类反馈后曲线均明显低于 Ours，凸显每类反馈的必要性。

### 3.5 来自人类的多模态反馈

Voyager 目前不支持视觉感知，因为本文写作时可用的 GPT-4 API 仅支持文本。不过 Voyager 有潜力被多模态感知模型 [58, 59] 增强，从而完成更令人印象深刻的任务。我们证明：在给定人类反馈的情况下，Voyager 能在 Minecraft 中搭建复杂 3D 结构，例如下界传送门（Nether Portal）和房屋（图 10）。整合人类反馈有两种方式：

1. **人类作为评论家**（等价于 Voyager 的自我验证模块）：人类向 Voyager 提供视觉批评，使其能够修改上一轮的代码。这类反馈对纠正 3D 结构空间细节中 Voyager 无法直接感知的错误至关重要。
2. **人类作为课程**（等价于 Voyager 的自动课程模块）：人类把复杂的建造任务拆解为更小的步骤，引导 Voyager 增量完成。这种方式提升了 Voyager 处理更复杂 3D 建造任务的能力。

> **图 10 | Voyager 在人类反馈下搭建 3D 结构。** 图为两行截图序列，每行 4 张、从左到右展示建造推进：上行"Build Nether Portal"由黑曜石框架逐步搭建直至传送门被点亮（紫色）；下行"Build House"从地基到墙体逐步成型。人类在每轮之间充当"评论家"（指出空间细节错误）或"课程"（拆解子步骤）。

---

## 4 局限与未来工作

**成本（Cost）。** GPT-4 API 成本高昂，比 GPT-3.5 贵 **15 倍**。然而 Voyager 需要 GPT-4 在代码生成质量上的质变（图 9），这是 GPT-3.5 与开源 LLM 无法提供的 [60]。

**不准确（Inaccuracies）。** 尽管有迭代提示机制，仍存在智能体卡住、无法生成正确技能的情况。自动课程具有灵活性，可在之后重新尝试该任务。偶尔自我验证模块也会失败，例如未把蜘蛛丝（spider string）识别为击败蜘蛛的成功信号。

**幻觉（Hallucinations）。** 自动课程偶尔提出无法完成的任务，例如要求智能体制作"铜剑"或"铜胸甲"——这些物品在游戏中并不存在。代码生成过程中也会出现幻觉，例如 GPT-4 倾向于把圆石（cobblestone）当作燃料输入，尽管它在游戏中不是有效燃料；此外它可能调用所提供的控制原语 API 中不存在的函数，导致代码执行错误。

我们相信，未来 GPT API 模型的改进以及开源 LLM 微调的新技术将克服这些局限。

---

## 5 相关工作

### Minecraft 中的决策智能体

Minecraft 是一个开放式的 3D 世界，其游戏机制极其灵活，支持广泛的活动。在若干著名的 Minecraft 基准 [23, 61, 62, 63, 64, 65] 之上，Minecraft 学习算法可分为两类：

1）**底层控制器**：许多早期工作利用分层强化学习从人类演示中学习 [66, 67, 68]。Kanitscheider 等人 [14] 设计了一个基于成功率的课程，但其目标局限于精选物品。MineDojo [23] 与 VPT [8] 利用 YouTube 视频进行大规模预训练。DreamerV3 [69] 则学习一个世界模型来探索环境并收集钻石。

2）**高层规划器**：Volum 等人 [70] 利用 Codex [41] 的少样本提示生成可执行策略，但需要额外的人工交互。近期工作把 LLM 作为 Minecraft 中的高层规划器，按照 Minecraft 配方把高层任务分解为若干子目标 [55, 53, 71]，因而缺乏完全的探索灵活性。

与后一类工作相似，Voyager 也通过提示 GPT-4 把 LLM 用作高层规划器，并遵循 Volum 等人 [70] 的做法使用 Mineflayer [52] 作为底层控制器。与前人不同的是，Voyager 采用以自底向上方式展开、由好奇心驱动、因而支持开放式探索的自动课程。

### 用于智能体规划的大语言模型

受 LLM 强大涌现能力（如零样本提示与复杂推理 [72, 37, 38, 36, 73, 74]）的启发，具身智能体研究 [75, 76, 77, 78] 中利用 LLM 进行规划的工作显著增加。近期努力大致可分为两类：

1）**用于机器人学习的大语言模型**：许多工作用 LLM 为机器人规划生成子目标 [27, 25, 79, 80]。Inner Monologue [26] 把环境反馈纳入 LLM 机器人规划；Code as Policies [16] 与 ProgPrompt [22] 直接用 LLM 生成可执行的机器人策略；VIMA [19] 与 PaLM-E [59] 微调预训练 LLM 以支持多模态提示。

2）**用于文本智能体的大语言模型**：ReAct [29] 利用思维链提示 [46]，由 LLM 同时生成推理轨迹与任务特定动作；Reflexion [30] 构建在 ReAct [29] 之上，用自我反思增强推理；AutoGPT [28] 是一个流行工具，通过为完成高层目标构造由多个子目标组成的课程，并结合 ReAct [29] 的推理—行动循环来自动化 NLP 任务；DERA [81] 把任务表述为两个 GPT-4 [35] 智能体之间的对话；Generative Agents [82] 利用 ChatGPT [50] 通过把智能体经验存为记忆并在规划时检索来模拟人类行为，但其智能体动作不可执行；SPRING [83] 是同期工作，用 GPT-4 从游戏手册中抽取游戏机制，据此回答按有向无环图组织的问题并预测下一步动作。

所有这些工作都缺少用于发展更复杂行为的技能库，而这正是 Voyager 在终身学习中成功的关键组件。

### 带执行的代码生成

代码生成长期以来是 NLP 的挑战 [41, 84, 85, 73, 37]，许多工作利用执行结果改进程序合成。执行引导（execution-guided）方法利用中间执行结果引导程序搜索 [86, 87, 88]；另一条研究路线用多数投票根据执行表现挑选候选 [89, 90]；LEVER [91] 训练验证器以根据执行结果区分并拒绝错误程序；CLAIRIFY [92] 为化学实验规划生成代码，并利用基于规则的验证器迭代地向 LLM 提供错误反馈。Voyager 与这些工作的区别在于：它把环境反馈、执行错误与自我验证（评估任务成功）整合进用于具身控制的迭代提示机制。

---

## 6 结论

本文提出 Voyager——首个由 LLM 驱动的具身终身学习智能体。它利用 GPT-4 持续探索世界、发展日益精密的技能，并在无人工干预下不断做出新发现。Voyager 在发现新物品、解锁 Minecraft 科技树、穿越多样地形，以及在新实例化世界中把已学技能库应用于未见过任务等方面均表现优异。Voyager 可作为开发强大通用智能体的起点，且无需调整模型参数。

---

## 7 更广泛影响

我们的研究在 Minecraft 这一安全无害的 3D 视频游戏环境中进行。尽管 Voyager 被设计为可通用到其他领域（如机器人），但把它应用于物理机器人需要额外注意，并由人类实施安全约束，以确保负责、安全的部署。

---

## 8 致谢

我们非常感谢 Ziming Zhu、Kaiyu Yang、Rafał Kocielnik、Colin White、Or Sharir、Sahin Lale、De-An Huang、Jean Kossaifi、Yuncong Yang、Charles Zhang、Minchao Huang 以及许多其他同事和朋友的反馈与富有洞见的讨论。本工作完成于 Guanzhi Wang 在 NVIDIA 实习期间。Guanzhi Wang 获 Caltech 计算与数学科学学院 Kortschak fellowship 资助。

---
## 参考文献

[1] Eric Kolve, Roozbeh Mottaghi, Winson Han, Eli VanderBilt, Luca Weihs, Alvaro Herrasti, Daniel Gordon, Yuke Zhu, Abhinav Gupta, and Ali Farhadi. Ai2-thor: An interactive 3d environment for visual ai. arXiv preprint arXiv: 1712.05474, 2017.

[2] Manolis Savva, Jitendra Malik, Devi Parikh, Dhruv Batra, Abhishek Kadian, Oleksandr Maksymets, Yili Zhao, Erik Wijmans, Bhavana Jain, Julian Straub, Jia Liu, and Vladlen Koltun. Habitat: A platform for embodied AI research. In 2019 IEEE/CVF International Conference on Computer Vision, ICCV 2019, Seoul, Korea (South), October 27 - November 2, 2019, pages 9338–9346. IEEE, 2019.

[3] Yuke Zhu, Josiah Wong, Ajay Mandlekar, and Roberto Martín-Martín. robosuite: A modular simulation framework and benchmark for robot learning. arXiv preprint arXiv: 2009.12293, 2020.

[4] Fei Xia, William B. Shen, Chengshu Li, Priya Kasimbeg, Micael Tchapmi, Alexander Toshev, Li Fei-Fei, Roberto Martín-Martín, and Silvio Savarese. Interactive gibson benchmark (igibson 0.5): A benchmark for interactive navigation in cluttered environments. arXiv preprint arXiv: 1910.14442, 2019.

[5] Bokui Shen, Fei Xia, Chengshu Li, Roberto Martín-Martín, Linxi Fan, Guanzhi Wang, Claudia Pérez-D’Arpino, Shyamal Buch, Sanjana Srivastava, Lyne P. Tchapmi, Micael E. Tchapmi, Kent Vainio, Josiah Wong, Li Fei-Fei, and Silvio Savarese. igibson 1.0: a simulation environment for interactive tasks in large realistic scenes. arXiv preprint arXiv: 2012.02924, 2020.

[6] Jens Kober, J Andrew Bagnell, and Jan Peters. Reinforcement learning in robotics: A survey. The International Journal of Robotics Research, 32(11):1238–1274, 2013.

[7] Kai Arulkumaran, Marc Peter Deisenroth, Miles Brundage, and Anil Anthony Bharath. Deep reinforcement learning: A brief survey. IEEE Signal Processing Magazine, 34(6):26–38, 2017.

[8] Bowen Baker, Ilge Akkaya, Peter Zhokhov, Joost Huizinga, Jie Tang, Adrien Ecoffet, Brandon Houghton, Raul Sampedro, and Jeff Clune. Video pretraining (vpt): Learning to act by watching unlabeled online videos. arXiv preprint arXiv: 2206.11795, 2022.

[9] DeepMind Interactive Agents Team, Josh Abramson, Arun Ahuja, Arthur Brussee, Federico Carnevale, Mary Cassin, Felix Fischer, Petko Georgiev, Alex Goldin, Mansi Gupta, Tim Harley, Felix Hill, Peter C Humphreys, Alden Hung, Jessica Landon, Timothy Lillicrap, Hamza Merzic, Alistair Muldal, Adam Santoro, Guy Scully, Tamara von Glehn, Greg Wayne, Nathaniel Wong, Chen Yan, and Rui Zhu. Creating multimodal interactive agents with imitation and self-supervised learning. arXiv preprint arXiv: 2112.03763, 2021.

[10] Oriol Vinyals, Igor Babuschkin, Junyoung Chung, Michael Mathieu, Max Jaderberg, Wojciech M Czarnecki, Andrew Dudzik, Aja Huang, Petko Georgiev, Richard Powell, et al. Alphastar: Mastering the real-time strategy game starcraft ii. DeepMind blog, 2, 2019.

[11] Adrien Ecoffet, Joost Huizinga, Joel Lehman, Kenneth O. Stanley, and Jeff Clune. Go-explore: a new approach for hard-exploration problems. arXiv preprint arXiv: 1901.10995, 2019.

[12] Joost Huizinga and Jeff Clune. Evolving multimodal robot behavior via many stepping stones with the combinatorial multiobjective evolutionary algorithm. Evolutionary computation, 30(2):131–164, 2022.

[13] Rui Wang, Joel Lehman, Aditya Rawal, Jiale Zhi, Yulun Li, Jeffrey Clune, and Kenneth O. Stanley. Enhanced POET: open-ended reinforcement learning through unbounded invention of learning challenges and their solutions. In Proceedings of the 37th International Conference on Machine Learning, ICML 2020, 13-18 July 2020, Virtual Event, volume 119 of Proceedings of Machine Learning Research, pages 9940–9951. PMLR, 2020.

[14] Ingmar Kanitscheider, Joost Huizinga, David Farhi, William Hebgen Guss, Brandon Houghton, Raul Sampedro, Peter Zhokhov, Bowen Baker, Adrien Ecoffet, Jie Tang, Oleg Klimov, and Jeff Clune. Multi-task curriculum learning in a complex, visual, hard-exploration domain: Minecraft. arXiv preprint arXiv: 2106.14876, 2021.

[15] Michael Dennis, Natasha Jaques, Eugene Vinitsky, Alexandre M. Bayen, Stuart Russell, Andrew Critch, and Sergey Levine. Emergent complexity and zero-shot transfer via unsupervised environment design. In Hugo Larochelle, Marc’Aurelio Ranzato, Raia Hadsell, Maria-Florina Balcan, and Hsuan-Tien Lin, editors, Advances in Neural Information Processing Systems 33: Annual Conference on Neural Information Processing Systems 2020, NeurIPS 2020, December 6-12, 2020, virtual, 2020.

[16] Jacky Liang, Wenlong Huang, Fei Xia, Peng Xu, Karol Hausman, Brian Ichter, Pete Florence, and Andy Zeng. Code as policies: Language model programs for embodied control. arXiv preprint arXiv: 2209.07753, 2022.

[17] Shao-Hua Sun, Te-Lin Wu, and Joseph J. Lim. Program guided agent. In 8th International Conference on Learning Representations, ICLR 2020, Addis Ababa, Ethiopia, April 26-30, 2020. OpenReview.net, 2020.

[18] Zelin Zhao, Karan Samel, Binghong Chen, and Le Song. Proto: Program-guided transformer for program-guided tasks. In Marc’Aurelio Ranzato, Alina Beygelzimer, Yann N. Dauphin, Percy Liang, and Jennifer Wortman Vaughan, editors, Advances in Neural Information Processing Systems 34: Annual Conference on Neural Information Processing Systems 2021, NeurIPS 2021, December 6-14, 2021, virtual, pages 17021–17036, 2021.

[19] Yunfan Jiang, Agrim Gupta, Zichen Zhang, Guanzhi Wang, Yongqiang Dou, Yanjun Chen, Li Fei-Fei, Anima Anandkumar, Yuke Zhu, and Linxi (Jim) Fan. Vima: General robot manipulation with multimodal prompts. ARXIV.ORG, 2022.

[20] Mohit Shridhar, Lucas Manuelli, and Dieter Fox. Cliport: What and where pathways for robotic manipulation. arXiv preprint arXiv: 2109.12098, 2021.

[21] Linxi Fan, Guanzhi Wang, De-An Huang, Zhiding Yu, Li Fei-Fei, Yuke Zhu, and Animashree Anandkumar. SECANT: self-expert cloning for zero-shot generalization of visual policies. In Marina Meila and Tong Zhang, editors, Proceedings of the 38th International Conference on Machine Learning, ICML 2021, 18-24 July 2021, Virtual Event, volume 139 of Proceedings of Machine Learning Research, pages 3088–3099. PMLR, 2021.

[22] Ishika Singh, Valts Blukis, Arsalan Mousavian, Ankit Goyal, Danfei Xu, Jonathan Tremblay, Dieter Fox, Jesse Thomason, and Animesh Garg. Progprompt: Generating situated robot task plans using large language models. arXiv preprint arXiv: 2209.11302, 2022.

[23] Linxi Fan, Guanzhi Wang, Yunfan Jiang, Ajay Mandlekar, Yuncong Yang, Haoyi Zhu, Andrew Tang, De-An Huang, Yuke Zhu, and Anima Anandkumar. Minedojo: Building open-ended embodied agents with internet-scale knowledge. arXiv preprint arXiv: 2206.08853, 2022.

[24] Andy Zeng, Adrian Wong, Stefan Welker, Krzysztof Choromanski, Federico Tombari, Aveek Purohit, Michael Ryoo, Vikas Sindhwani, Johnny Lee, Vincent Vanhoucke, and Pete Florence. Socratic models: Composing zero-shot multimodal reasoning with language. arXiv preprint arXiv: 2204.00598, 2022.

[25] Michael Ahn, Anthony Brohan, Noah Brown, Yevgen Chebotar, Omar Cortes, Byron David, Chelsea Finn, Keerthana Gopalakrishnan, Karol Hausman, Alex Herzog, Daniel Ho, Jasmine Hsu, Julian Ibarz, Brian Ichter, Alex Irpan, Eric Jang, Rosario Jauregui Ruano, Kyle Jeffrey, Sally Jesmonth, Nikhil J Joshi, Ryan Julian, Dmitry Kalashnikov, Yuheng Kuang, Kuang-Huei Lee, Sergey Levine, Yao Lu, Linda Luu, Carolina Parada, Peter Pastor, Jornell Quiambao, Kanishka Rao, Jarek Rettinghouse, Diego Reyes, Pierre Sermanet, Nicolas Sievers, Clayton Tan, Alexander Toshev, Vincent Vanhoucke, Fei Xia, Ted Xiao, Peng Xu, Sichun Xu, and Mengyuan Yan. Do as i can, not as i say: Grounding language in robotic affordances. arXiv preprint arXiv: 2204.01691, 2022.

[26] Wenlong Huang, Fei Xia, Ted Xiao, Harris Chan, Jacky Liang, Pete Florence, Andy Zeng, Jonathan Tompson, Igor Mordatch, Yevgen Chebotar, Pierre Sermanet, Noah Brown, Tomas Jackson, Linda Luu, Sergey Levine, Karol Hausman, and Brian Ichter. Inner monologue: Embodied reasoning through planning with language models. arXiv preprint arXiv: 2207.05608, 2022.

[27] Wenlong Huang, Pieter Abbeel, Deepak Pathak, and Igor Mordatch. Language models as zero-shot planners: Extracting actionable knowledge for embodied agents. In Kamalika Chaudhuri, Stefanie Jegelka, Le Song, Csaba Szepesvári, Gang Niu, and Sivan Sabato, editors, International Conference on Machine Learning, ICML 2022, 17-23 July 2022, Baltimore, Maryland, USA, volume 162 of Proceedings of Machine Learning Research, pages 9118–9147. PMLR, 2022.

[28] Significant-gravitas/auto-gpt: An experimental open-source attempt to make gpt-4 fully autonomous., 2023.

[29] Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran, Karthik Narasimhan, and Yuan Cao. React: Synergizing reasoning and acting in language models. arXiv preprint arXiv: 2210.03629, 2022.

[30] Noah Shinn, Beck Labash, and Ashwin Gopinath. Reflexion: an autonomous agent with dynamic memory and self-reflection. arXiv preprint arXiv: 2303.11366, 2023.

[31] German Ignacio Parisi, Ronald Kemker, Jose L. Part, Christopher Kanan, and Stefan Wermter. Continual lifelong learning with neural networks: A review. Neural Networks, 113:54–71, 2019.

[32] Liyuan Wang, Xingxing Zhang, Hang Su, and Jun Zhu. A comprehensive survey of continual learning: Theory, method and application. arXiv preprint arXiv: 2302.00487, 2023.

[33] Volodymyr Mnih, Koray Kavukcuoglu, David Silver, Alex Graves, Ioannis Antonoglou, Daan Wierstra, and Martin Riedmiller. Playing atari with deep reinforcement learning. arXiv preprint arXiv: 1312.5602, 2013.

[34] OpenAI, :, Christopher Berner, Greg Brockman, Brooke Chan, Vicki Cheung, Przemysław Dębiak, Christy Dennison, David Farhi, Quirin Fischer, Shariq Hashme, Chris Hesse, Rafal Józefowicz, Scott Gray, Catherine Olsson, Jakub Pachocki, Michael Petrov, Henrique P. d. O. Pinto, Jonathan Raiman, Tim Salimans, Jeremy Schlatter, Jonas Schneider, Szymon Sidor, Ilya Sutskever, Jie Tang, Filip Wolski, and Susan Zhang. Dota 2 with large scale deep reinforcement learning. arXiv preprint arXiv: 1912.06680, 2019.

[35] OpenAI. Gpt-4 technical report. arXiv preprint arXiv: 2303.08774, 2023.

[36] Jason Wei, Yi Tay, Rishi Bommasani, Colin Raffel, Barret Zoph, Sebastian Borgeaud, Dani Yogatama, Maarten Bosma, Denny Zhou, Donald Metzler, Ed H. Chi, Tatsunori Hashimoto, Oriol Vinyals, Percy Liang, Jeff Dean, and William Fedus. Emergent abilities of large language models. arXiv preprint arXiv: 2206.07682, 2022.

[37] Tom B. Brown, Benjamin Mann, Nick Ryder, Melanie Subbiah, Jared Kaplan, Prafulla Dhariwal, Arvind Neelakantan, Pranav Shyam, Girish Sastry, Amanda Askell, Sandhini Agarwal, Ariel Herbert-Voss, Gretchen Krueger, Tom Henighan, Rewon Child, Aditya Ramesh, Daniel M. Ziegler, Jeffrey Wu, Clemens Winter, Christopher Hesse, Mark Chen, Eric Sigler, Mateusz Litwin, Scott Gray, Benjamin Chess, Jack Clark, Christopher Berner, Sam McCandlish, Alec Radford, Ilya Sutskever, and Dario Amodei. Language models are few-shot learners. In Hugo Larochelle, Marc’Aurelio Ranzato, Raia Hadsell, Maria-Florina Balcan, and Hsuan-Tien Lin, editors, Advances in Neural Information Processing Systems 33: Annual Conference on Neural Information Processing Systems 2020, NeurIPS 2020, December 6-12, 2020, virtual, 2020.

[38] Colin Raffel, Noam Shazeer, Adam Roberts, Katherine Lee, Sharan Narang, Michael Matena, Yanqi Zhou, Wei Li, and Peter J. Liu. Exploring the limits of transfer learning with a unified text-to-text transformer. J. Mach. Learn. Res., 21:140:1–140:67, 2020.

[39] Benjamin Eysenbach, Abhishek Gupta, Julian Ibarz, and Sergey Levine. Diversity is all you need: Learning skills without a reward function. In 7th International Conference on Learning Representations, ICLR 2019, New Orleans, LA, USA, May 6-9, 2019. OpenReview.net, 2019.

[40] Edoardo Conti, Vashisht Madhavan, Felipe Petroski Such, Joel Lehman, Kenneth O. Stanley, and Jeff Clune. Improving exploration in evolution strategies for deep reinforcement learning via a population of novelty-seeking agents. In Samy Bengio, Hanna M. Wallach, Hugo Larochelle, Kristen Grauman, Nicolò Cesa-Bianchi, and Roman Garnett, editors, Advances in Neural Information Processing Systems 31: Annual Conference on Neural Information Processing Systems 2018, NeurIPS 2018, December 3-8, 2018, Montréal, Canada, pages 5032–5043, 2018.

[41] Mark Chen, Jerry Tworek, Heewoo Jun, Qiming Yuan, Henrique Ponde de Oliveira Pinto, Jared Kaplan, Harri Edwards, Yuri Burda, Nicholas Joseph, Greg Brockman, Alex Ray, Raul Puri, Gretchen Krueger, Michael Petrov, Heidy Khlaaf, Girish Sastry, Pamela Mishkin, Brooke Chan, Scott Gray, Nick Ryder, Mikhail Pavlov, Alethea Power, Lukasz Kaiser, Mohammad Bavarian, Clemens Winter, Philippe Tillet, Felipe Petroski Such, Dave Cummings, Matthias Plappert, Fotios Chantzis, Elizabeth Barnes, Ariel Herbert-Voss, William Hebgen Guss, Alex Nichol, Alex Paino, Nikolas Tezak, Jie Tang, Igor Babuschkin, Suchir Balaji, Shantanu Jain, William Saunders, Christopher Hesse, Andrew N. Carr, Jan Leike, Josh Achiam, Vedant Misra, Evan Morikawa, Alec Radford, Matthew Knight, Miles Brundage, Mira Murati, Katie Mayer, Peter Welinder, Bob McGrew, Dario Amodei, Sam McCandlish, Ilya Sutskever, and Wojciech Zaremba. Evaluating large language models trained on code. arXiv preprint arXiv: 2107.03374, 2021.

[42] Rui Wang, Joel Lehman, Jeff Clune, and Kenneth O. Stanley. Paired open-ended trailblazer (poet): Endlessly generating increasingly complex and diverse learning environments and their solutions. arXiv preprint arXiv: 1901.01753, 2019.

[43] Rémy Portelas, Cédric Colas, Lilian Weng, Katja Hofmann, and Pierre-Yves Oudeyer. Automatic curriculum learning for deep RL: A short survey. In Christian Bessiere, editor, Proceedings of the Twenty-Ninth International Joint Conference on Artificial Intelligence, IJCAI 2020, pages 4819–4825. ijcai.org, 2020.

[44] Sébastien Forestier, Rémy Portelas, Yoan Mollard, and Pierre-Yves Oudeyer. Intrinsically motivated goal exploration processes with automatic curriculum learning. The Journal of Machine Learning Research, 23(1):6818–6858, 2022.

[45] Kevin Ellis, Catherine Wong, Maxwell Nye, Mathias Sable-Meyer, Luc Cary, Lucas Morales, Luke Hewitt, Armando Solar-Lezama, and Joshua B. Tenenbaum. Dreamcoder: Growing generalizable, interpretable knowledge with wake-sleep bayesian program learning. arXiv preprint arXiv: 2006.08381, 2020.

[46] Jason Wei, Xuezhi Wang, Dale Schuurmans, Maarten Bosma, Ed Chi, Quoc Le, and Denny Zhou. Chain of thought prompting elicits reasoning in large language models. arXiv preprint arXiv: 2201.11903, 2022.

[47] Volodymyr Mnih, Adrià Puigdomènech Badia, Mehdi Mirza, Alex Graves, Timothy P. Lillicrap, Tim Harley, David Silver, and Koray Kavukcuoglu. Asynchronous methods for deep reinforcement learning. In Maria-Florina Balcan and Kilian Q. Weinberger, editors, Proceedings of the 33nd International Conference on Machine Learning, ICML 2016, New York City, NY, USA, June 19-24, 2016, volume 48 of JMLR Workshop and Conference Proceedings, pages 1928–1937. JMLR.org, 2016.

[48] John Schulman, Filip Wolski, Prafulla Dhariwal, Alec Radford, and Oleg Klimov. Proximal policy optimization algorithms. arXiv preprint arXiv: 1707.06347, 2017.

[49] Timothy P. Lillicrap, Jonathan J. Hunt, Alexander Pritzel, Nicolas Heess, Tom Erez, Yuval Tassa, David Silver, and Daan Wierstra. Continuous control with deep reinforcement learning. In Yoshua Bengio and Yann LeCun, editors, 4th International Conference on Learning Representations, ICLR 2016, San Juan, Puerto Rico, May 2-4, 2016, Conference Track Proceedings, 2016.

[50] Introducing chatgpt, 2022.

[51] New and improved embedding model, 2022.

[52] PrismarineJS. Prismarinejs/mineflayer: Create minecraft bots with a powerful, stable, and high level javascript api., 2013.

[53] Kolby Nottingham, Prithviraj Ammanabrolu, Alane Suhr, Yejin Choi, Hanna Hajishirzi, Sameer Singh, and Roy Fox. Do embodied agents dream of pixelated sheep?: Embodied decision making using language guided world modelling. ARXIV.ORG, 2023.

[54] Shaofei Cai, Zihao Wang, Xiaojian Ma, Anji Liu, and Yitao Liang. Open-world multi-task control through goal-aware representation learning and adaptive horizon prediction. arXiv preprint arXiv: 2301.10034, 2023.

[55] Zihao Wang, Shaofei Cai, Anji Liu, Xiaojian Ma, and Yitao Liang. Describe, explain, plan and select: Interactive planning with large language models enables open-world multi-task agents. arXiv preprint arXiv: 2302.01560, 2023.

[56] Sébastien Bubeck, Varun Chandrasekaran, Ronen Eldan, Johannes Gehrke, Eric Horvitz, Ece Kamar, Peter Lee, Yin Tat Lee, Yuanzhi Li, Scott Lundberg, Harsha Nori, Hamid Palangi, Marco Tulio Ribeiro, and Yi Zhang. Sparks of artificial general intelligence: Early experiments with gpt-4. arXiv preprint arXiv: 2303.12712, 2023.

[57] Yiheng Liu, Tianle Han, Siyuan Ma, Jiayue Zhang, Yuanyuan Yang, Jiaming Tian, Hao He, Antong Li, Mengshen He, Zhengliang Liu, Zihao Wu, Dajiang Zhu, Xiang Li, Ning Qiang, Dingang Shen, Tianming Liu, and Bao Ge. Summary of chatgpt/gpt-4 research and perspective towards the future of large language models. arXiv preprint arXiv: 2304.01852, 2023.

[58] Shikun Liu, Linxi Fan, Edward Johns, Zhiding Yu, Chaowei Xiao, and Anima Anandkumar. Prismer: A vision-language model with an ensemble of experts. arXiv preprint arXiv: 2303.02506, 2023.

[59] Danny Driess, Fei Xia, Mehdi S. M. Sajjadi, Corey Lynch, Aakanksha Chowdhery, Brian Ichter, Ayzaan Wahid, Jonathan Tompson, Quan Vuong, Tianhe Yu, Wenlong Huang, Yevgen Chebotar, Pierre Sermanet, Daniel Duckworth, Sergey Levine, Vincent Vanhoucke, Karol Hausman, Marc Toussaint, Klaus Greff, Andy Zeng, Igor Mordatch, and Pete Florence. Palm-e: An embodied multimodal language model. arXiv preprint arXiv: 2303.03378, 2023.

[60] Hugo Touvron, Thibaut Lavril, Gautier Izacard, Xavier Martinet, Marie-Anne Lachaux, Timothée Lacroix, Baptiste Rozière, Naman Goyal, Eric Hambro, Faisal Azhar, Aurelien Rodriguez, Armand Joulin, Edouard Grave, and Guillaume Lample. Llama: Open and efficient foundation language models. arXiv preprint arXiv: 2302.13971, 2023.

[61] William H. Guss, Brandon Houghton, Nicholay Topin, Phillip Wang, Cayden Codel, Manuela Veloso, and Ruslan Salakhutdinov. Minerl: A large-scale dataset of minecraft demonstrations. In Sarit Kraus, editor, Proceedings of the Twenty-Eighth International Joint Conference on Artificial Intelligence, IJCAI 2019, Macao, China, August 10-16, 2019, pages 2442–2448. ijcai.org, 2019.

[62] William H. Guss, Cayden Codel, Katja Hofmann, Brandon Houghton, Noboru Kuno, Stephanie Milani, Sharada Mohanty, Diego Perez Liebana, Ruslan Salakhutdinov, Nicholay Topin, Manuela Veloso, and Phillip Wang. The minerl 2019 competition on sample efficient reinforcement learning using human priors. arXiv preprint arXiv: 1904.10079, 2019.

[63] William H. Guss, Mario Ynocente Castro, Sam Devlin, Brandon Houghton, Noboru Sean Kuno, Crissman Loomis, Stephanie Milani, Sharada Mohanty, Keisuke Nakata, Ruslan Salakhutdinov, John Schulman, Shinya Shiroshita, Nicholay Topin, Avinash Ummadisingu, and Oriol Vinyals. The minerl 2020 competition on sample efficient reinforcement learning using human priors. arXiv preprint arXiv: 2101.11071, 2021.

[64] Anssi Kanervisto, Stephanie Milani, Karolis Ramanauskas, Nicholay Topin, Zichuan Lin, Junyou Li, Jianing Shi, Deheng Ye, Qiang Fu, Wei Yang, Weijun Hong, Zhongyue Huang, Haicheng Chen, Guangjun Zeng, Yue Lin, Vincent Micheli, Eloi Alonso, François Fleuret, Alexander Nikulin, Yury Belousov, Oleg Svidchenko, and Aleksei Shpilman. Minerl diamond 2021 competition: Overview, results, and lessons learned. arXiv preprint arXiv: 2202.10583, 2022.

[65] Matthew Johnson, Katja Hofmann, Tim Hutton, and David Bignell. The malmo platform for artificial intelligence experimentation. In Subbarao Kambhampati, editor, Proceedings of the Twenty-Fifth International Joint Conference on Artificial Intelligence, IJCAI 2016, New York, NY, USA, 9-15 July 2016, pages 4246–4247. IJCAI/AAAI Press, 2016.

[66] Zichuan Lin, Junyou Li, Jianing Shi, Deheng Ye, Qiang Fu, and Wei Yang. Juewu-mc: Playing minecraft with sample-efficient hierarchical reinforcement learning. arXiv preprint arXiv: 2112.04907, 2021.

[67] Hangyu Mao, Chao Wang, Xiaotian Hao, Yihuan Mao, Yiming Lu, Chengjie Wu, Jianye Hao, Dong Li, and Pingzhong Tang. Seihai: A sample-efficient hierarchical ai for the minerl competition. arXiv preprint arXiv: 2111.08857, 2021.

[68] Alexey Skrynnik, Aleksey Staroverov, Ermek Aitygulov, Kirill Aksenov, Vasilii Davydov, and Aleksandr I. Panov. Hierarchical deep q-network from imperfect demonstrations in minecraft. Cogn. Syst. Res., 65:74–78, 2021.

[69] Danijar Hafner, Jurgis Pasukonis, Jimmy Ba, and Timothy Lillicrap. Mastering diverse domains through world models. arXiv preprint arXiv: 2301.04104, 2023.

[70] Ryan Volum, Sudha Rao, Michael Xu, Gabriel DesGarennes, Chris Brockett, Benjamin Van Durme, Olivia Deng, Akanksha Malhotra, and Bill Dolan. Craft an iron sword: Dynamically generating interactive game characters by prompting large language models tuned on code. In Proceedings of the 3rd Wordplay: When Language Meets Games Workshop (Wordplay 2022), pages 25–43, Seattle, United States, 2022. Association for Computational Linguistics.

[71] Haoqi Yuan, Chi Zhang, Hongcheng Wang, Feiyang Xie, Penglin Cai, Hao Dong, and Zongqing Lu. Plan4mc: Skill reinforcement learning and planning for open-world minecraft tasks. arXiv preprint arXiv: 2303.16563, 2023.

[72] Rishi Bommasani, Drew A. Hudson, Ehsan Adeli, Russ Altman, Simran Arora, Sydney von Arx, Michael S. Bernstein, Jeannette Bohg, Antoine Bosselut, Emma Brunskill, Erik Brynjolfsson, Shyamal Buch, Dallas Card, Rodrigo Castellon, Niladri Chatterji, Annie Chen, Kathleen Creel, Jared Quincy Davis, Dora Demszky, Chris Donahue, Moussa Doumbouya, Esin Durmus, Stefano Ermon, John Etchemendy, Kawin Ethayarajh, Li Fei-Fei, Chelsea Finn, Trevor Gale, Lauren Gillespie, Karan Goel, Noah Goodman, Shelby Grossman, Neel Guha, Tatsunori Hashimoto, Peter Henderson, John Hewitt, Daniel E. Ho, Jenny Hong, Kyle Hsu, Jing Huang, Thomas Icard, Saahil Jain, Dan Jurafsky, Pratyusha Kalluri, Siddharth Karamcheti, Geoff Keeling, Fereshte Khani, Omar Khattab, Pang Wei Koh, Mark Krass, Ranjay Krishna, Rohith Kuditipudi, Ananya Kumar, Faisal Ladhak, Mina Lee, Tony Lee, Jure Leskovec, Isabelle Levent, Xiang Lisa Li, Xuechen Li, Tengyu Ma, Ali Malik, Christopher D. Manning, Suvir Mirchandani, Eric Mitchell, Zanele Munyikwa, Suraj Nair, Avanika Narayan, Deepak Narayanan, Ben Newman, Allen Nie, Juan Carlos Niebles, Hamed Nilforoshan, Julian Nyarko, Giray Ogut, Laurel Orr, Isabel Papadimitriou, Joon Sung Park, Chris Piech, Eva Portelance, Christopher Potts, Aditi Raghunathan, Rob Reich, Hongyu Ren, Frieda Rong, Yusuf Roohani, Camilo Ruiz, Jack Ryan, Christopher Ré, Dorsa Sadigh, Shiori Sagawa, Keshav Santhanam, Andy Shih, Krishnan Srinivasan, Alex Tamkin, Rohan Taori, Armin W. Thomas, Florian Tramèr, Rose E. Wang, William Wang, Bohan Wu, Jiajun Wu, Yuhuai Wu, Sang Michael Xie, Michihiro Yasunaga, Jiaxuan You, Matei Zaharia, Michael Zhang, Tianyi Zhang, Xikun Zhang, Yuhui Zhang, Lucia Zheng, Kaitlyn Zhou, and Percy Liang. On the opportunities and risks of foundation models. arXiv preprint arXiv: 2108.07258, 2021.

[73] Aakanksha Chowdhery, Sharan Narang, Jacob Devlin, Maarten Bosma, Gaurav Mishra, Adam Roberts, Paul Barham, Hyung Won Chung, Charles Sutton, Sebastian Gehrmann, Parker Schuh, Kensen Shi, Sasha Tsvyashchenko, Joshua Maynez, Abhishek Rao, Parker Barnes, Yi Tay, Noam Shazeer, Vinodkumar Prabhakaran, Emily Reif, Nan Du, Ben Hutchinson, Reiner Pope, James Bradbury, Jacob Austin, Michael Isard, Guy Gur-Ari, Pengcheng Yin, Toju Duke, Anselm Levskaya, Sanjay Ghemawat, Sunipa Dev, Henryk Michalewski, Xavier Garcia, Vedant Misra, Kevin Robinson, Liam Fedus, Denny Zhou, Daphne Ippolito, David Luan, Hyeontaek Lim, Barret Zoph, Alexander Spiridonov, Ryan Sepassi, David Dohan, Shivani Agrawal, Mark Omernick, Andrew M. Dai, Thanumalayan Sankaranarayana Pillai, Marie Pellat, Aitor Lewkowycz, Erica Moreira, Rewon Child, Oleksandr Polozov, Katherine Lee, Zongwei Zhou, Xuezhi Wang, Brennan Saeta, Mark Diaz, Orhan Firat, Michele Catasta, Jason Wei, Kathy Meier-Hellstern, Douglas Eck, Jeff Dean, Slav Petrov, and Noah Fiedel. Palm: Scaling language modeling with pathways. arXiv preprint arXiv: 2204.02311, 2022.

[74] Hyung Won Chung, Le Hou, Shayne Longpre, Barret Zoph, Yi Tay, William Fedus, Eric Li, Xuezhi Wang, Mostafa Dehghani, Siddhartha Brahma, Albert Webson, Shixiang Shane Gu, Zhuyun Dai, Mirac Suzgun, Xinyun Chen, Aakanksha Chowdhery, Sharan Narang, Gaurav Mishra, Adams Yu, Vincent Zhao, Yanping Huang, Andrew Dai, Hongkun Yu, Slav Petrov, Ed H. Chi, Jeff Dean, Jacob Devlin, Adam Roberts, Denny Zhou, Quoc V. Le, and Jason Wei. Scaling instruction-finetuned language models. arXiv preprint arXiv: 2210.11416, 2022.

[75] Jiafei Duan, Samson Yu, Hui Li Tan, Hongyuan Zhu, and Cheston Tan. A survey of embodied AI: from simulators to research tasks. IEEE Trans. Emerg. Top. Comput. Intell., 6(2):230–244, 2022.

[76] Dhruv Batra, Angel X. Chang, Sonia Chernova, Andrew J. Davison, Jia Deng, Vladlen Koltun, Sergey Levine, Jitendra Malik, Igor Mordatch, Roozbeh Mottaghi, Manolis Savva, and Hao Su. Rearrangement: A challenge for embodied ai. arXiv preprint arXiv: 2011.01975, 2020.

[77] Harish Ravichandar, Athanasios S Polydoros, Sonia Chernova, and Aude Billard. Recent advances in robot learning from demonstration. Annual review of control, robotics, and autonomous systems, 3:297–330, 2020.

[78] Jack Collins, Shelvin Chand, Anthony Vanderkop, and David Howard. A review of physics simulators for robotic applications. IEEE Access, 9:51416–51431, 2021.

[79] So Yeon Min, Devendra Singh Chaplot, Pradeep Ravikumar, Yonatan Bisk, and R. Salakhutdinov. Film: Following instructions in language with modular methods. International Conference on Learning Representations, 2021.

[80] Valts Blukis, Chris Paxton, Dieter Fox, Animesh Garg, and Yoav Artzi. A persistent spatial semantic representation for high-level natural language instruction execution. In 5th Annual Conference on Robot Learning, 2021.

[81] Varun Nair, Elliot Schumacher, Geoffrey Tso, and Anitha Kannan. Dera: Enhancing large language model completions with dialog-enabled resolving agents. arXiv preprint arXiv: 2303.17071, 2023.

[82] Joon Sung Park, Joseph C. O’Brien, Carrie J. Cai, Meredith Ringel Morris, Percy Liang, and Michael S. Bernstein. Generative agents: Interactive simulacra of human behavior. arXiv preprint arXiv: 2304.03442, 2023.

[83] Yue Wu, Shrimai Prabhumoye, So Yeon Min, Yonatan Bisk, Ruslan Salakhutdinov, Amos Azaria, Tom Mitchell, and Yuanzhi Li. Spring: Gpt-4 out-performs rl algorithms by studying papers and reasoning. arXiv preprint arXiv: 2305.15486, 2023.

[84] Erik Nijkamp, Bo Pang, Hiroaki Hayashi, Lifu Tu, Huan Wang, Yingbo Zhou, Silvio Savarese, and Caiming Xiong. A conversational paradigm for program synthesis. arXiv preprint arXiv: 2203.13474, 2022.

[85] Hung Le, Yue Wang, Akhilesh Deepak Gotmare, Silvio Savarese, and Steven C. H. Hoi. Coderl: Mastering code generation through pretrained models and deep reinforcement learning. arXiv preprint arXiv: 2207.01780, 2022.

[86] Xinyun Chen, Chang Liu, and Dawn Song. Execution-guided neural program synthesis. In 7th International Conference on Learning Representations, ICLR 2019, New Orleans, LA, USA, May 6-9, 2019. OpenReview.net, 2019.

[87] Xinyun Chen, Dawn Song, and Yuandong Tian. Latent execution for neural program synthesis. arXiv preprint arXiv: 2107.00101, 2021.

[88] Kevin Ellis, Maxwell I. Nye, Yewen Pu, Felix Sosa, Josh Tenenbaum, and Armando Solar-Lezama. Write, execute, assess: Program synthesis with a REPL. In Hanna M. Wallach, Hugo Larochelle, Alina Beygelzimer, Florence d’Alché-Buc, Emily B. Fox, and Roman Garnett, editors, Advances in Neural Information Processing Systems 32: Annual Conference on Neural Information Processing Systems 2019, NeurIPS 2019, December 8-14, 2019, Vancouver, BC, Canada, pages 9165–9174, 2019.

[89] Yujia Li, David Choi, Junyoung Chung, Nate Kushman, Julian Schrittwieser, Rémi Leblond, Tom Eccles, James Keeling, Felix Gimeno, Agustin Dal Lago, Thomas Hubert, Peter Choy, Cyprien de Masson d’Autume, Igor Babuschkin, Xinyun Chen, Po-Sen Huang, Johannes Welbl, Sven Gowal, Alexey Cherepanov, James Molloy, Daniel J. Mankowitz, Esme Sutherland Robson, Pushmeet Kohli, Nando de Freitas, Koray Kavukcuoglu, and Oriol Vinyals. Competition-level code generation with alphacode. arXiv preprint arXiv: 2203.07814, 2022.

[90] Karl Cobbe, Vineet Kosaraju, Mohammad Bavarian, Mark Chen, Heewoo Jun, Lukasz Kaiser, Matthias Plappert, Jerry Tworek, Jacob Hilton, Reiichiro Nakano, Christopher Hesse, and John Schulman. Training verifiers to solve math word problems. arXiv preprint arXiv: 2110.14168, 2021.

[91] Ansong Ni, Srini Iyer, Dragomir Radev, Ves Stoyanov, Wen tau Yih, Sida I. Wang, and Xi Victoria Lin. Lever: Learning to verify language-to-code generation with execution. arXiv preprint arXiv: 2302.08468, 2023.

[92] Marta Skreta, Naruki Yoshikawa, Sebastian Arellano-Rubach, Zhi Ji, Lasse Bjørn Kristensen, Kourosh Darvish, Alán Aspuru-Guzik, Florian Shkurti, and Animesh Garg. Errors are useful prompts: Instruction guided task programming with verifier-assisted iterative prompting. arXiv preprint arXiv: 2303.14100, 2023.

## 附录 A 方法

### A.1 Voyager 算法

**伪代码 1：Voyager 算法。**

```python
def voyager(
    environment,        # 以代码为动作空间的环境
    curriculum_agent,   # 负责提出下一个任务的课程智能体
    action_agent,       # 负责生成代码的动作智能体
    critic_agent,       # 负责自我验证的评论家智能体
    skill_manager,      # 负责添加新技能与技能检索的技能管理器
):
    agent_state = environment.reset()
    while True:
        exploration_progress = (
            curriculum_agent.get_exploration_progress(
                curriculum_agent.get_completed_tasks(),
                curriculum_agent.get_failed_tasks(),
            )
        )
        task = curriculum_agent.propose_next_task(
            agent_state, exploration_progress
        )
        code = None
        environment_feedback = None
        execution_errors = None
        critique = None
        success = False
        # 在转入下一个任务前，最多尝试 4 轮
        for i in range(4):
            skills = skill_manager.retrieve_skills(
                task, environment_feedback
            )
            code = action_agent.generate_code(
                task,
                code,
                environment_feedback,
                execution_errors,
                critique,
                skills,
            )
            (
                agent_state,
                environment_feedback,
                execution_errors,
            ) = environment.step(code)
            success, critique = critic_agent.check_task_success(
                task, agent_state
            )
            if success:
                break
        if success:
            skill_manager.add_skill(code)
            curriculum_agent.add_completed_task(task)
        else:
            curriculum_agent.add_failed_task(task)
```

### A.2 提示（Prompting）

GPT-4 与 GPT-3.5 允许用户为每条提示消息指定三种角色之一：

- **System**：引导模型在整个对话中行为的高层指令，为交互设定总体基调与目标。
- **User**：为引导助手下一次即时回复而给出的详细指令。
- **Assistant**：模型生成的回复消息。

详见 https://platform.openai.com/docs/guides/chat/introduction 。

为节省 token 用量，我们不进行多轮对话，而是把一条 system 提示与一条 user 提示拼接，以获得助手的每次回复。

### A.3 自动课程

#### A.3.1 提示中的组件

输入 GPT-4 的提示由以下几个部分组成：

**（1）指令**：鼓励多样化行为并施加约束（使所提任务既可完成又可验证），完整提示见 A.3.4 节。

**（2）智能体当前状态：**

- **物品栏（Inventory）**：物品到数量的字典，例如 `{'cobblestone': 4, 'furnace': 1, 'stone_pickaxe': 1, 'oak_planks': 7, 'dirt': 6, 'wooden_pickaxe': 1, 'crafting_table': 1, 'raw_iron': 4, 'coal': 1}`；
- **装备（Equipment）**：智能体已装备的护甲或武器；
- **附近方块（Nearby blocks）**：距智能体 32 格内的方块名称集合，例如 `'dirt'`、`'water'`、`'spruce_planks'`、`'grass_block'`、`'dirt_path'`、`'sugar_cane'`、`'fern'`；
- **最近见过的其他方块**：既不在附近、也不在物品栏中的方块；
- **附近实体（Nearby entities）**：距智能体 32 格内的实体名称集合，例如 `'pig'`、`'cat'`、`'villager'`、`'zombie'`；
- **已见过的箱子列表**：箱子是智能体可存放物品的外部容器。若某箱子此前未被打开，其内容显示为"Unknown"；否则箱内物品会展示给智能体；
- **生物群系（Biome）**：例如 `'plains'`、`'flower_forest'`、`'meadow'`、`'river'`、`'beach'`、`'forest'`、`'snowy_slopes'`、`'frozen_peaks'`、`'old_growth_birch_forest'`、`'ocean'`、`'sunflower_plains'`、`'stony_shore'`；
- **时间**：`'sunrise'`（日出）、`'day'`（白天）、`'noon'`（正午）、`'sunset'`（日落）、`'night'`（夜晚）、`'midnight'`（午夜）之一；
- **生命值与饥饿值条**：最大值为 20；
- **位置**：智能体在 Minecraft 世界中的 3D 坐标 $(x, y, z)$。

**（3）此前已完成与失败的任务；**

**（4）附加上下文**：见 A.3.2 节；

**（5）响应中的思维链提示 [46]**：我们要求 GPT-4 先推理当前进展，再建议下一个任务。

#### A.3.2 附加上下文

我们利用 GPT-3.5 自问问题以提供附加上下文。每个问题都配有一个概念，用于从 wiki 知识库 [23] 中检索最相关的文档；我们把文档内容喂给 GPT-3.5 做自答。实践中，使用 wiki 知识库是可选的，因为 GPT-3.5 本身已对 Minecraft 游戏机制有良好理解；但如果 GPT-3.5 未在特定领域预训练过，外部知识库就会带来优势。完整提示见 A.3.4 节。

#### A.3.3 预热调度（Warm-up Schedule）

实践中，我们采用预热调度：根据智能体已完成的任务数量，逐步把智能体状态与附加上下文纳入提示。这保证了随着探索推进，提示所暴露的信息量逐步增加，因而智能体从基础技能起步，逐步迈向更复杂、更多样的技能。所有实验统一使用的预热设置见表 A.1。

> **表 A.1 | 自动课程的预热调度。**

| 提示中的信息 | 在完成多少任务后加入 |
| --- | --- |
| 核心物品栏（仅含 log、planks、stick、crafting table、furnace、dirt、coal、pickaxe、sword、axe） | 0 |
| 装备 | 0 |
| 附近方块 | 0 |
| 位置 | 0 |
| 附近实体 | 5 |
| 完整物品栏 | 7 |
| 最近见过的其他方块 | 10 |
| 生物群系 | 10 |
| 生命值条 | 15 |
| 饥饿值条 | 15 |
| 时间 | 15 |
| 附加上下文 | 15 |

#### A.3.4 完整提示（导览）

> *（译者注）原文此处给出 Prompt 1–3 的完整文本（system 提示 + user 提示），总计约 3 页，内容为纯提示词模板与 few-shot 示例，逐字翻译价值有限，故按结构概括；完整原文可在论文 PDF 附录 A.3.4 或开源代码库中查看。*

- **Prompt 1（自动课程 system 提示）**：设定角色"你是一个有用的助手，告诉我接下来在 Minecraft 中要做的即时任务"；声明终极目标（尽可能发现多样事物、完成多样任务、成为最好的 Minecraft 玩家）；随后依次给出若干"Question/Answer"自问自答对（附加上下文）、`Biome`、`Time`、`Nearby blocks`、`Nearby entities`、`Other blocks that are recently seen`、`Chests`、`Health`、`Hunger`、`Position`、`Equipment`、`Inventory`（含 counts）、`Completed tasks`、`Failed tasks`；最后给出任务约束（"下一个任务不应太难""任务必须可用给定 API 完成""不要重复已完成任务"）与输出格式要求。
- **Prompt 2（自动课程 user 提示）**：把上述状态字段实例化后填入，并要求模型先 `Reasoning:` 再 `Task:` 输出。
- **Prompt 3（few-shot 示例）**：给出 2–3 个"状态 → 推理 → 任务"的示范，用于稳定输出格式。

### A.4 技能库

#### A.4.1 提示中的组件

输入 GPT-4 的提示包含以下组成部分：

**（1）代码生成准则**：完整提示见 A.4.2 节。

**（2）我们实现的控制原语 API**：这些 API 有两个用途——演示 Mineflayer API 的用法，以及可被 GPT-4 直接调用。

- `exploreUntil(bot, direction, maxTime = 60, callback)`：让智能体沿固定方向探索至多 `maxTime`。`callback` 是由智能体实现的停止条件，用于判断何时停止探索；
- `mineBlock(bot, name, count = 1)`：在 32 格范围内开采并收集指定数量的方块；
- `craftItem(bot, name, count = 1)`：借助附近的合成台合成物品；
- `placeItem(bot, name, position)`：在指定位置放置方块；
- `smeltItem(bot, itemName, fuelName, count = 1)`：用指定燃料熔炼物品，附近必须有熔炉；
- `killMob(bot, mobName, timeout = 300)`：攻击生物并收集其掉落物；
- `getItemFromChest(bot, chestPosition, itemsToGet)`：移动到指定位置的箱子并从中取出物品；
- `depositItemIntoChest(bot, chestPosition, itemsToDeposit)`：移动到指定位置的箱子并把物品存入其中。

**（3）Mineflayer 提供的控制原语 API：**

- `await bot.pathfinder.goto(goal)`：前往特定位置（goal 的设置方式见下）；
- `new GoalNear(x, y, z, range)`：把机器人移动到指定方块指定范围内的某个方块；
- `new GoalXZ(x, z)`：用于没有特定 Y 层的长距离目标；
- `new GoalGetToBlock(x, y, z)`：不进入该方块，而是紧邻其旁。适用于钓鱼、耕种、装水桶、使用床；
- `new GoalFollow(entity, range)`：在指定范围内跟随指定实体；
- `new GoalPlaceBlock(position, bot.world, {})`：调整机器人站位以便放置方块；
- `new GoalLookAtBlock(position, bot.world, {})`：寻路到能看见该方块某一面的位置；
- `bot.isABed(bedBlock)`：若 bedBlock 是床则返回 true；
- `bot.blockAt(position)`：返回该位置的方块；
- `await bot.equip(item, destination)`：把物品装备到指定部位，`destination` 须为 `"hand"`、`"head"`、`"torso"`、`"legs"`、`"feet"`、`"off-hand"` 之一；
- `await bot.consume()`：消耗手中的物品，须先装备。适用于进食、饮用药水等；
- `await bot.fish()`：让机器人钓鱼。调用前必须先到达水方块旁并装备钓竿；钓到鱼后自动停止；
- `await bot.sleep(bedBlock)`：睡到日出，须先到达床边；
- `await bot.activateBlock(block)`：等同于游戏中右键点击方块，适用于按钮、门等，须先到达该方块；
- `await bot.lookAt(position)`：看向指定位置，须先靠近；用水桶装水前必须先看向水；
- `await bot.activateItem()`：等同于右键使用手中物品，适用于使用水桶等，须先装备；
- `await bot.useOn(entity)`：等同于右键点击实体，适用于给羊剪毛，须先靠近该实体。

**（4）从技能库检索到的技能；**

**（5）上一轮生成的代码；**

**（6）环境反馈**：提示中的聊天记录（chat log）；

**（7）执行错误；**

**（8）自我验证模块给出的批评；**

**（9）智能体当前状态**：各字段见 A.3.1 节；

**（10）自动课程提出的任务；**

**（11）任务上下文**：我们提示 GPT-3.5 给出求解该任务的一般性建议。实践中该部分由自动课程承担，因为它具备系统化的问答机制（A.3.2 节）；

**（12）响应中的思维链提示 [46]**：我们要求 GPT-4 先解释上一轮代码失败的原因，再给出完成任务的逐步计划，最后生成代码。完整提示见 A.4.2 节。

#### A.4.2 完整提示（导览）

> *（译者注）原文此处为 Prompt 4（代码生成的 system 提示）与 Prompt 5（user 提示），约 4 页，包含全部 API 文档、字段说明与 few-shot 代码示例。*

- **Prompt 4（system 提示）**：角色设定为"你是 Minecraft 中编写 JavaScript 代码的助手"；随后给出代码准则（函数须通用可复用、只调用已提供的 API、使用 `bot.chat()` 输出中间进展、出错时 `bot.chat()` 说明原因、用 `exploreUntil` 之前先确认目标不可直接找到、以 `async function` 形式给出单个自包含函数）；接着是上面 A.4.1 中的全部控制原语 API 文档与 Mineflayer API 文档；然后是 2 个完整的成功示例（含 `Reasoning`、`Plan`、`Code` 三段）。
- **Prompt 5（user 提示）**：实例化 `CURRENT STATUS`（状态、装备、物品栏、已见过的箱子、附近方块/实体等）、`TASK`、`CONTEXT`、`PREVIOUS CODE`、`ENVIRONMENT FEEDBACK`、`EXECUTION ERRORS`、`CRITIQUE`、`RETRIEVED SKILLS`，并要求按 `Explain why the previous code failed` → `Step-by-step plan` → `Code` 的顺序作答。

#### A.4.3 示例（技能库中的 4 个程序）

> *（译者注）原文给出 4 段完整 JS 代码，此处保留函数签名与实现要点，便于理解"技能"的粒度与写法。*

**示例 1：`craftWoodenPlanks`** — 先检查物品栏中是否有任意一种原木（oak/birch/spruce/jungle/acacia/dark_oak/mangrove），没有则先调用 `mineWoodLog(bot)` 砍树；再据原木类型映射到对应的木板类型，调用 `craftItem` 合成；过程中用 `bot.chat()` 输出进展（"No wooden log in inventory. Mining a wooden log..." / "Crafting 4 oak_planks..." / "4 oak_planks crafted."）。**要点：把"前提条件检查 → 补齐前提 → 执行 → 报告"封装成一个可复用技能。**

**示例 2：`mineTenCobbledDeepslateBelowY0`** — 先从物品栏取出铁镐并 `bot.equip(..., "hand")`；再用 `exploreUntil(bot, new Vec3(1, -1, 1), 60, () => bot.findBlock({matching: mcData.blocksByName["cobbled_deepslate"].id, maxDistance: 32, position: pos => pos.y < 0}))` 在 Y<0 的范围内寻找深板岩碎块；最后 `mineBlock(bot, "cobbled_deepslate", 10)` 并报告。**要点：用带约束的谓词（`pos.y < 0`）表达"深部"这一空间语义。**

**示例 3：`smeltFiveRawIronV2`** — 先定义辅助函数 `findSuitablePosition(bot)`，在机器人四周 ±1 格偏移中找出第一个 `air` 方块；若物品栏没有熔炉则先 `craftFurnace(bot)`；找到合适位置后 `placeItem(bot, "furnace", furnacePosition)`，再 `smeltItem(bot, "raw_iron", "coal", 5)`。**要点：复杂技能内部会定义并使用子函数，体现程序化技能的可组合性。**

**示例 4：`fillBucketWithWater`** — 用 `exploreUntil` 在 32 格内寻找 `water` 方块；若未找到则 `bot.chat()` 说明并返回；找到后靠近、`bot.lookAt` 水方块，再 `bot.activateItem()` 完成装水。**要点：把"看不见的目标 → 探索搜索 → 交互"的失败分支写成显式早退，避免无效执行。**

### A.5 自我验证

#### A.5.1 提示中的组件

输入 GPT-4 的提示包含以下组成部分：

1. **智能体状态**：我们排除了"最近见过的其他方块"与"附近实体"，因为它们对判断任务是否完成没有帮助。各字段见 A.3.1 节；
2. **自动课程提出的任务**；
3. **任务上下文**：提示 GPT-3.5 给出求解该任务的一般性建议（实践中由自动课程承担，A.3.2 节）；
4. **响应中的思维链提示 [46]**：要求 GPT-4 先推理任务成功或失败，再输出表示任务结果的布尔变量，最后在任务失败时向智能体给出批评；
5. **用于上下文学习 [36, 37, 38] 的 few-shot 示例。**

#### A.5.2 完整提示（导览）

> *（译者注）原文此处为 Prompt 6（自我验证的 system 提示）与 Prompt 7，约 2 页。*

- **Prompt 6（system 提示）**：角色设定为"你是 Minecraft 的评论家，负责判断任务是否完成"；给出判断准则（如"物品只需出现在物品栏中即算获得""合成任务需最终物品存在于物品栏""任务要求数量时须满足数量"）；要求输出 JSON 风格的 `{"success": true/false, "critique": "..."}`，并在 `critique` 中给出可执行的改进建议。
- **Prompt 7（user 提示）**：填入 `Task`、`Context`、`Agent State`（物品栏、装备、生命/饥饿、位置、生物群系、时间、附近方块与箱子）。

### A.6 Voyager 与此前工作的系统级比较

我们在表 A.2 中做系统级比较。Voyager 是唯一同时具备自动课程、迭代规划与技能库三者组合的方法；此外，它在**不需要任何梯度更新**的情况下学会玩 Minecraft。

> **表 A.2 | Voyager 与此前工作的系统级比较。**（✓ 表示具备该特性）

| | VPT [8] | DreamerV3 [69] | DECKARD [53] | DEPS [55] | Plan4MC [71] | Voyager |
| --- | --- | --- | --- | --- | --- | --- |
| 演示数据 | 视频 | 无 | 视频 | 无 | 无 | 无 |
| 奖励 | 稀疏 | 稠密 | 稀疏 | 无 | 稠密 | 无 |
| 观测 | 仅像素 | 像素 + 元信息 | 像素 + 物品栏 | 反馈 + 物品栏 | 像素 + 元信息 | 反馈 + 元信息 + 物品栏 |
| 动作 | 键鼠 | 离散 | 键鼠 | 键鼠 | 离散 | **代码** |
| 自动课程 | | | ✓ | | | ✓（GPT-4 上下文内提议） |
| 迭代规划 | | | | ✓ | | ✓（3 类反馈） |
| 技能库 | | | | | ✓（预定义） | ✓（自生成） |
| 免梯度训练 | | | | | | ✓ |

---

## 附录 B 实验

### B.1 实验设置

我们的仿真环境构建在 MineDojo [23] 之上，并使用 Mineflayer [52] 的 JavaScript API 做运动控制（见 A.4.2 节）。此外，我们在 Mineflayer 函数中大量插入 `bot.chat()` 以提供丰富的环境反馈，并实现各种条件检查与 try-catch 异常处理以支持连续执行。若机器人死亡，它会在最近的地面附近复活，且物品栏得以保留，从而保证探索不中断。程序执行结束后，机器人会回收其合成台与熔炉。详细实现请参阅我们的代码库。

### B.2 基线方法

**ReAct** [29] 使用思维链提示 [46]，由 LLM 同时生成推理轨迹与行动计划。我们向其提供环境反馈与智能体状态作为观测。ReAct 先从头进行一轮代码生成，随后进行三轮代码精炼；该过程重复，直至达到最大提示迭代次数。

**Reflexion** [30] 构建在 ReAct [29] 之上并加入自我反思，以推断更直观的后续动作。我们向其提供环境反馈、智能体状态、执行错误以及我们的自我验证模块。与 ReAct 类似，Reflexion 先从头进行一轮代码生成，再做三轮代码精炼，重复至最大迭代次数。

**AutoGPT** [28] 是一个流行工具，通过把高层目标分解为多个子目标并在 ReAct 风格循环中执行来自动化 NLP 任务。我们用 GPT-4 做任务分解重新实现 AutoGPT，并向其提供智能体状态、环境反馈与执行错误作为子目标执行的观测。与 Voyager 相比，AutoGPT 缺少积累知识的技能库、评估任务成功的自我验证、以及开放式探索的自动课程。在每个子目标执行中，若未出现执行错误，我们认为该子目标完成并进入下一个；否则精炼程序，直到完成三轮代码精炼（等价于四轮代码生成）后再进入下一子目标。若连续三个子目标都未获得新物品，我们重新运行任务分解以重新规划。

所有基线的任务均为"探索世界并尽可能获得更多物品"。

> **表 A.3 | Voyager 与各基线的比较。**

| | ReAct [29] | Reflexion [30] | AutoGPT [28] | Voyager |
| --- | --- | --- | --- | --- |
| 思维链 [46] | ✓ | ✓ | ✓ | ✓ |
| 自我验证 | | ✓ | | ✓ |
| 环境反馈 | ✓ | ✓ | ✓ | ✓ |
| 执行错误 | | ✓ | ✓ | ✓ |
| 智能体状态 | ✓ | ✓ | ✓ | ✓ |
| 技能库 | | | | ✓ |
| 自动课程 | | | | ✓ |

### B.3 消融变体

我们消融 Voyager 的 6 项设计选择（自动课程、技能库、环境反馈、执行错误、自我验证、用于代码生成的 GPT-4），研究它们对探索表现的影响。

- **人工课程（Manual Curriculum）**：把自动课程替换为人工设计的挖钻石课程——"开采 3 个原木""合成 1 个合成台""合成 1 把木镐""开采 11 个圆石""合成 1 把石镐""合成 1 个熔炉""开采 3 个铁矿石""熔炼 3 个铁矿石""合成 1 把铁镐""开采 1 个钻石"。人工课程需要人力设计，且难以扩展到开放式探索。
- **随机课程（Random Curriculum）**：我们从 Voyager 获得的 101 个物品中策划出集合，通过随机选取一个物品作为下一个任务来构造随机课程。
- **无技能库（w/o Skill Library）**：移除技能库，即代码生成时不做技能检索。
- **无环境反馈（w/o Environment Feedback）**：从代码生成提示中移除环境反馈（聊天记录）。
- **无执行错误（w/o Execution Errors）**：从代码生成提示中移除执行错误。
- **无自我验证（w/o Self-Verification）**：每个任务在无自我验证的情况下生成代码，并迭代精炼 3 轮（等价于总共 4 轮代码生成）。
- **GPT-3.5**：用 GPT-3.5 替代 GPT-4 做代码生成，自动课程与自我验证模块仍保留 GPT-4。

### B.4 评估结果

#### B.4.1 显著更好的探索

> **图 A.1 | Minecraft 物品图标及对应名称。** 图 1 曲线中每个图标代表的物品，在此以"图标—名称"对照图例列出。

我们对每种方法运行三次试验。Voyager 在各次试验中收集到的物品为：

- **试验 1**：`iron_ingot`、`stone_shovel`、`iron_leggings`、`fishing_rod`、`pufferfish`、`oak_log`、`cooked_mutton`、`green_dye`、`flint`、`chest`、`iron_sword`、`string`、`ender_pearl`、`raw_copper`、`crafting_table`、`cactus`、`lapis_lazuli`、`iron_pickaxe`、`copper_ingot`、`stone_pickaxe`、`wooden_hoe`、`scaffolding`、`stick`、`porkchop`、`copper_block`、`gravel`、`grass_block`、`white_bed`、`bone`、`dirt`、`mutton`、`white_wool`、`oak_sapling`、`coal`、`bamboo`、`wooden_pickaxe`、`rotten_flesh`、`cooked_porkchop`、`cod`、`iron_boots`、`lightning_rod`、`diorite`、`water_bucket`、`shears`、`furnace`、`andesite`、`granite`、`bucket`、`wooden_sword`、`sandstone`、`iron_helmet`、`raw_iron`、`sand`、`acacia_log`、`cooked_cod`、`oak_planks`、`azure_bluet`、`iron_shovel`、`acacia_planks`、`shield`、`iron_axe`、`iron_chestplate`、`cobblestone`（共 63 种）；
- **试验 2**：`iron_ingot`、`tuff`、`stone_shovel`、`iron_leggings`、`fishing_rod`、`cooked_mutton`、`spruce_planks`、`gunpowder`、`amethyst_shard`、`chest`、`string`、`cooked_salmon`、`iron_sword`、`raw_copper`、`crafting_table`、`torch`、`lapis_lazuli`、`iron_pickaxe`、`copper_ingot`、`stone_pickaxe`、`wooden_hoe`、`stick`、`amethyst_block`、`salmon`、`calcite`、`gravel`、`white_bed`、`bone`、`dirt`、`mutton`、`white_wool`、`spyglass`、`coal`、`wooden_pickaxe`、`cod`、`iron_boots`、`lily_pad`、`cobbled_deepslate`、`lightning_rod`、`snowball`、`stone_axe`、`smooth_basalt`、`diorite`、`water_bucket`、`furnace`、`andesite`、`bucket`、`granite`、`shield`、`iron_helmet`、`raw_iron`、`cobblestone`、`spruce_log`、`cooked_cod`、`tripwire_hook`、`stone_hoe`、`iron_chestplate`、`stone_sword`（共 58 种）；
- **试验 3**：`spruce_planks`、`dirt`、`shield`、`redstone`、`clock`、`diamond_sword`、`iron_chestplate`、`stone_pickaxe`、`leather`、`string`、`chicken`、`chest`、`diorite`、`iron_leggings`、`black_wool`、`cobblestone_wall`、`cobblestone`、`cooked_chicken`、`feather`、`stone_sword`、`raw_gold`、`gravel`、`birch_planks`、`coal`、`cobbled_deepslate`、`oak_planks`、`iron_pickaxe`、`granite`、`tuff`、`crafting_table`、`iron_helmet`、`stone_hoe`、`iron_ingot`、`stone_axe`、`birch_boat`、`stick`、`sand`、`bone`、`raw_iron`、`beef`、`rail`、`oak_sapling`、`kelp`、`gold_ingot`、`birch_log`、`wheat_seeds`、`cooked_mutton`、`furnace`、`arrow`、`stone_shovel`、`white_wool`、`andesite`、`jungle_slab`、`mutton`、`iron_sword`、`copper_ingot`、`diamond`、`torch`、`oak_log`、`cooked_beef`、`copper_block`、`flint`、`bone_meal`、`raw_copper`、`wooden_pickaxe`、`iron_boots`、`wooden_sword`（共 68 种，含钻石）。

**ReAct** [29] 各次试验收集到的物品为：

- 试验 1：`bamboo`、`dirt`、`sand`、`wheat_seeds`（4 种）；
- 试验 2：`dirt`、`rabbit`、`spruce_log`、`spruce_sapling`（4 种）；
- 试验 3：`dirt`、`pointed_dripstone`（2 种）。

**Reflexion** [30] 各次试验收集到的物品为：

- 试验 1：`crafting_table`、`orange_tulip`、`oak_planks`、`oak_log`、`dirt`（5 种）；
- 试验 2：`spruce_log`、`dirt`、`clay_ball`、`sand`、`gravel`（5 种）；
- 试验 3：`wheat_seeds`、`oak_log`、`dirt`、`birch_log`、`sand`（5 种）。

**AutoGPT** [28] 各次试验收集到的物品为：

- 试验 1：`feather`、`oak_log`、`leather`、`stick`、`porkchop`、`chicken`、`crafting_table`、`wheat_seeds`、`oak_planks`、`dirt`、`mutton`（11 种）；
- 试验 2：`wooden_pickaxe`、`iron_ingot`、`stone`、`coal`、`spruce_planks`、`string`、`raw_copper`、`crafting_table`、`diorite`、`andesite`、`furnace`、`torch`、`spruce_sapling`、`granite`、`iron_pickaxe`、`stone_pickaxe`、`wooden_axe`、`raw_iron`、`stick`、`spruce_log`、`dirt`、`cobblestone`（22 种）；
- 试验 3：`wooden_shovel`、`wooden_pickaxe`、`iron_ingot`、`stone`、`cod`、`coal`、`oak_log`、`flint`、`raw_copper`、`crafting_table`、`diorite`、`furnace`、`andesite`、`torch`、`granite`、`lapis_lazuli`、`iron_pickaxe`、`stone_pickaxe`、`raw_iron`、`stick`、`gravel`、`oak_planks`、`dirt`、`iron_axe`、`cobblestone`（25 种）。

#### B.4.2 广泛的地图遍历

> **图 A.2 | 地图覆盖：Minecraft 地图的两张鸟瞰图。** Voyager 能穿越多样地形，行进距离为基线的 2.3 倍。轨迹按各智能体与 GPT-4 交互时的位置绘制。

地图覆盖的智能体轨迹见图 A.2。图 7 是在图 A.2 基础上，为每条轨迹绘制最小外接圆得到的。Voyager 各次试验穿越的地形为：

- 试验 1：`meadow`、`desert`、`river`、`savanna`、`forest`、`plains`、`bamboo_jungle`、`dripstone_caves`；
- 试验 2：`snowy_plains`、`frozen_river`、`dripstone_caves`、`snowy_taiga`、`beach`；
- 试验 3：`flower_forest`、`meadow`、`old_growth_birch_forest`、`snowy_slopes`、`frozen_peaks`、`forest`、`river`、`beach`、`ocean`、`sunflower_plains`、`plains`、`stony_shore`。

**ReAct** [29] 各次试验穿越的地形为：试验 1：`plains`、`desert`、`jungle`；试验 2：`snowy_plains`、`snowy_taiga`、`snowy_slopes`；试验 3：`dark_forest`、`dripstone_caves`、`grove`、`jagged_peaks`。

**Reflexion** [30] 各次试验穿越的地形为：试验 1：`plains`、`flower_forest`；试验 2：`snowy_taiga`；试验 3：`old_growth_birch_forest`、`river`、`ocean`、`beach`、`plains`。

**AutoGPT** [28] 各次试验穿越的地形为：试验 1：`plains`、`dripstone_caves`、`savanna`、`meadow`；试验 2：`snowy_taiga`；试验 3：`plains`、`stony_shore`、`forest`、`ocean`。

#### B.4.3 对未见任务的高效零样本泛化

> **图 A.3 | 对未见任务的零样本泛化。** 我们可视化各方法在另外两个任务上的中间进展。ReAct 与 Reflexion 因未取得任何有意义的进展而未绘制。

另外两个任务的零样本泛化结果见图 A.3。与图 8 类似，Voyager 能稳定求解所有任务，而基线在 50 次提示迭代内无法完成任何任务。由终身学习构建的技能库不仅提升了 Voyager 的表现，也为 AutoGPT [28] 带来增益。

#### B.4.4 准确的技能检索

我们评估了技能检索（共 309 个样本），结果见表 A.4。top-5 准确率为 96.5%，说明检索过程可靠（注意：我们在提示中包含 top-5 相关技能用于合成新技能）。

> **表 A.4 | 技能检索准确率。**

| Top-1 Acc | Top-2 Acc | Top-3 Acc | Top-4 Acc | Top-5 Acc |
| --- | --- | --- | --- | --- |
| 80.2 ± 3.0 | 89.3 ± 1.8 | 93.2 ± 0.7 | 95.2 ± 1.8 | 96.5 ± 0.3 |

#### B.4.5 对模型变化的鲁棒性

正文中 Voyager 的所有实验使用 `gpt-4-0314`。我们额外用 `gpt-4-0613` 运行了新实验，发现性能大致相同（图 A.4），说明 Voyager 对模型变化具有鲁棒性。

> **图 A.4 | Voyager 在 GPT-4-0314 与 GPT-4-0613 下的表现。** 两条"发现独特物品数 vs. 提示迭代次数"曲线几乎重合。

---

## 术语与背景速览（译者整理）

| 术语 | 含义 |
| --- | --- |
| 自动课程（automatic curriculum） | 由 LLM 依据智能体当前状态与探索进度动态提出的下一个任务，替代人工设计的任务序列；本文的核心贡献之一，可视为上下文内的新颖性搜索 |
| 技能库（skill library） | 以可执行代码为技能载体的向量数据库：key 是技能描述的嵌入向量（GPT-3.5 生成），value 是 JS 程序本体；支持增量添加与 top-5 检索 |
| 迭代提示机制（iterative prompting mechanism） | 生成代码 → 执行 → 收集环境反馈与执行错误 → 自我验证给出批评 → 再生成的闭环，最多 4 轮 |
| 自我验证（self-verification） | 用另一个 GPT-4 实例充当评论家（critic），判断任务是否完成，并在失败时给出可执行的改进建议；同时覆盖"检查成功"与"反思错误" |
| 环境反馈（environment feedback） | 智能体代码内部通过 `bot.chat()` 打出的中间进展（如"还差 7 个铁锭"），作为下一轮提示的上下文 |
| 以代码为动作空间（code as action space） | 动作不是"前进/左转"等原子指令，而是一段调用 API 的程序；优点是时间上可延展、可组合、可解释、可复用 |
| MineDojo | 基于 Minecraft 的开源 AI 研究框架与基准（论文 [23]），提供任务、知识与仿真接口 |
| Mineflayer | Node.js 的 Minecraft 机器人库（论文 [52]），本文用作底层运动控制器，暴露 `bot.pathfinder`、`bot.equip`、`bot.fish` 等 API |
| 科技树（tech tree） | Minecraft 的工具升级层级：木 → 石 → 铁 → 钻石；本文用作衡量长时程能力成长的标尺 |
| 上下文学习（in-context learning） | 不改动模型参数，仅通过提示中的说明与示例让模型完成新任务；Voyager 全程不微调 |
| 灾难性遗忘（catastrophic forgetting） | 持续学习中学习新任务时遗忘旧能力；Voyager 通过把已掌握技能固化为代码并存入技能库来缓解 |
| 零样本泛化（zero-shot generalization） | 在新实例化的世界中、清空物品栏的条件下，仅靠已学技能库与任务分解解决未见过任务 |

**核心数字速记**

- 独特物品：**63** 种 / 160 次提示迭代，是基线的 **3.3×**
- 科技树解锁速度：木制 **15.3×**、石制 **8.5×**、铁制 **6.4×** 更快；唯一解锁钻石层级（102 次迭代，1/3 成功）
- 行进距离：**2.3×** 于基线，覆盖地形从 2–8 种提升到 5–12 种
- 零样本新任务：4 个任务全部成功（19 ± 3 / 18 ± 7 / 21 ± 5 / 18 ± 2 次迭代），基线全部 0/3
- 消融：随机课程 −**93%** 物品数；去自我验证 −**73%**；GPT-3.5 替代 GPT-4 仅得 1/5.7 的物品
- 技能检索 top-5 准确率 **96.5%**（309 个样本）
- 成本：GPT-4 比 GPT-3.5 贵 **15×**，但代码生成质量是 Voyager 成立的前提
