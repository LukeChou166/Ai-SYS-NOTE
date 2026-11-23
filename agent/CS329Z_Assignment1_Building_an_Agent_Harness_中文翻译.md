# CS329Z 作业 1：构建 Agent Harness

> **原文**: CS329Z Assignment 1: Building an Agent Harness（Version 26.1.0）
> **作者**: CS329Z Staff
> **日期**: Fall 2026
> **链接**: https://github.com/cs329z/assignment1-harness/blob/main/cs329z_assignment1_harness.pdf
> **译者说明**: 本文为完整全文翻译。harness 译为「上下文/工具调用框架」（保留英文 harness），pipeline 译为「流水线」，tool registry 译为「工具注册表」。原文中的蓝色省钱提示框、设计决策标记、代码块、分值表均已保留；带 *（译者注）* 的段落为译者补充。

---

## 1 作业概览

在本次作业中，你将从零开始构建 agentic 系统：先构建 LLM 流水线，也叫工作流（workflow，即控制流写在代码里、LLM 只被调用来做分类、读取或写入的程序），然后构建 agent harness（一个由你搭建机械结构、由模型决定下一步做什么的系统）。你将使用两个数据集（第 3 节）：Cardinal Energy 公司的文档——这是一家虚构公司，你将在第 2 部分构建它的内部助手；以及一个真实的企业邮件归档——你的第 1 部分流水线（以及之后的 agent）将在其上 operate。

**你将实现的内容**

1. **第 1 部分：LLM 流水线**（第 4 节）。单次调用的邮件优先级分类器、每日摘要生成器、从零实现的 BM25 检索器，以及用于邮件归档问答的多跳 RAG 流水线。
2. **第 2 部分：Cardinal Agent**（第 5 节）。面向 Cardinal Energy 员工的用户助手，构建在你自己编写的工具调用循环之上。你将构建工具注册表和工具调用的文本格式，把第 1 部分的流水线包装成工具，加入文档检索，给 agent 一个终端，让它在长会话中保持在上下文预算内（compaction，压缩），给它跨会话的记忆，让用户可以批准、拒绝和引导它的行为，并加入防护栏（guardrails）。

**你将运行的内容**

1. 确定性测试套件（deterministic test suite），不调用 LLM，对你的代码机制打分。
2. 实时测试套件（live test suite），用真实模型运行你的流水线和 agent，按阈值评判任务完成度。
3. 最终评测：模拟用户（由 LLM 驱动的角色，带有目标与多轮对话）与你的 agent 对话，回答它的问题，批准或拒绝它的动作。每段对话由程序化检查和一个 judge 模型评分。
4. 你自己的 agent：在终端里用你自己选的请求测试它，并写一份简短记录它做了什么（Problem 16 (field_notes)）。

本作业共 **118 分**：88 分来自自动评分测试，30 分来自两篇书面报告。我们提前提供完整测试套件：每个测试最初都因 `NotImplementedError` 失败，你的任务是通过实现 `cs329z_hw1/adapters.py` 里接好的函数来让它们通过。第 2 节解释测试如何组织、每道题如何与测试对应。

### 作业中的决策

构建 harness 涉及两类选择。

第一类是**结构性的**：有哪些函数、它们的输入输出长什么样、每个正确实现必须共有的行为是什么。这些由我们替你决定，并由测试套件强制执行。

第二类是**判断性的**：一个很长的工具输出应该让模型看到多少？当对话被摘要以塞进上下文窗口时，摘要必须保留什么？哪些动作可以不先问用户就直接执行？真实系统以不同方式回答这些问题，我们不替你选答案。凡是某道题把这类判断留给你，题面中都会有一行「Design decision」（设计决策）标明；讲义末尾的设计备忘录会问你为什么这么选。测试的目的是排除明显失败的选择，其余由备忘录来评判。备忘录（Problem 17 (design_memo)）对每个设计决策问一个问题；开始第 2 部分前先读它的清单，做每个选择时记下笔记。**复杂度不加分**：一个通过测试且在备忘录里有辩护的策略，无论三行还是三百行，都得满分。

你的实现中**不得使用 agent 或 LLM 框架库**，也不得向 `pyproject.toml` 添加依赖。NumPy 已是依赖，可随意使用。课程提供的 LM wrapper 只给你一个原语：一个接受消息列表、返回字符串的 chat-completion 调用。它没有工具调用，也没有结构化输出功能。在此之上构建的一切都由你负责，包括如何告诉模型有哪些工具、如何解析它的回复。我们的目标是让你在依赖库之前，至少端到端地亲手构建一次 agent 框架。

### 关于 AI 辅助的说明

做这份作业时你可以使用 Claude Code、Codex、Copilot 或任何其他 AI 工具。我们会通过口头测验检查理解程度——就你的设计和代码提问。你应该能解释提交内容中的一切，并对它负责。

> **省钱提示：免费测试与缓存调用**
> 本讲义中用这样的蓝框标注以小 LLM 预算完成作业的建议。起始代码有两个特性让开销很小：(1) 确定性测试从不调用 LLM，它们占 88 分自动评分中的 48 分（第 7 节的表格按题列出）；(2) 课程 LM wrapper 把每次调用缓存到磁盘，因此用相同模型和相同消息重复调用是免费的。用我们的参考实现，除评测外的所有实时测试各跑一次未缓存调用仅花 **$0.05**、约五分钟；评测另加 **$0.18** 和三分钟。

---

## 2 环境搭建

作业运行在 Python 3.11 或更高版本上，支持 macOS、Linux、Windows，用 `uv` 管理环境和依赖[^1]。在 Windows 上还要安装 Git for Windows[^2]，因为 Problem 10 (terminal) 的终端命令由它的 `bash.exe` 执行；WSL 也可以。克隆仓库后，在根目录用 `uv run` 运行所有命令（首次运行时会创建环境）：

```bash
git clone https://github.com/cs329z/assignment1-harness.git
cd assignment1-harness
cp .env.example .env
# 然后把你的 API key 放进 .env
uv run python data/download.py
# 解压邮件归档
uv run pytest
# 每个测试都因 NotImplementedError 失败
```

我们发给你的 API key 放进 `.env`，变量名见 `.env.example`。邮件归档以单个压缩文件 `data/emails.jsonl.gz`（19 MB）随仓库提供，`data/download.py` 会用 SHA-256 校验并解压为 `data/emails/emails.jsonl`（58 MB）。

你写的所有代码都放进 `cs329z_hw1` 包，其中有测试调用的 `adapters.py`，以及你添加的任何模块；空的 `pipelines/` 和 `agent/` 包是建议放置位置。仓库其余部分是课程代码。下面清单中标注 **provided** 的文件由我们维护——只读，不要编辑，因为评分用我们的副本。标注 **read** 的文件定义了你要依赖的接口或测试检查的规则，在首次用到它的那道题之前都值得读一遍。

| 文件 | 状态 | 说明 |
| --- | --- | --- |
| `cs329z_hw1/` | 你的包 | 你写的每个文件都放这里 |
| `adapters.py` | yours / read | 测试唯一调用的文件；以桩代码提供 |
| `pipelines/` | yours | 第 1 部分（初始为空） |
| `agent/` | yours | 第 2 部分（初始为空） |
| `llm.py` | provided / read | LM（缓存、成本账本、预算）、ScriptedLM |
| `types.py` | provided / read | 你的代码与测试共享的数据类型 |
| `tokens.py` | provided / read | `count_tokens`、`count_message_tokens` |
| `tokenizer.py` | provided / read | `tokenize()`，BM25 必须使用的分词器 |
| `data.py` | provided / read | `load_emails`、`emails_on`、`email_text`、`load_docs` |
| `sandbox.py` | provided / read | `run_terminal`、`make_workspace` |
| `cardinal.py` | provided / read | agent 工具的名称与参数 schema |
| `user.py` | provided / read | `UserIO`、`ScriptedUser`、`ConsoleUser` |
| `chat.py` | provided | 在终端里与你的 agent 对话 |
| `show.py` | provided | 在真实数据上跑单条流水线并打印结果 |
| `simulation/` | provided | 模拟用户、judge、评测运行器 |
| `tests/test_*.py` | provided（评分时替换）/ read | 每道题一个文件；读你正在做的那个 |
| `tests/thresholds.py` | provided | 实时测试的通过阈值 |
| `tests/fixtures/` | provided | 金标签、问题、角色设定 |
| `data/docs/` | provided | Cardinal Energy 文档 |
| `data/emails.jsonl.gz` | provided | 压缩的邮件归档 |
| `data/download.py` | provided | 校验并解压归档 |
| `data/source.json` | provided | 归档的校验和 |
| `data/priority_rubric.md` | provided / read | 优先级标签是如何判定的 |
| `data/emails/` | 由 download.py 解压 | — |

[^1]: 安装说明：https://docs.astral.sh/uv/getting-started/installation/
[^2]: https://git-scm.com/download/win，默认选项即可。

### LM

所有模型调用都通过 `cs329z_hw1.llm.LM`。一个 LM 对象接受 `{"role", "content"}` 消息列表并返回一个字符串，这就是它的全部接口：

```python
from cs329z_hw1.llm import LM

lm = LM()
# 我们用来评分的模型
reply = lm([{"role": "system", "content": "Answer in one word."},
            {"role": "user", "content": "What color is the sky?"}])
# -> "Blue."
```

角色只有 `system`、`user`、`assistant`；任何其他角色或键的消息都会被拒绝。没有工具调用或结构化输出功能；在第 2 部分你要在这个调用之上把工具调用写成文本（Problem 6 (tool_protocol)）。适配器把这个签名写作 `LMCallable`（在 `types.py` 中），确定性测试传入同样形状的 `ScriptedLM`。

wrapper 缓存每个回复，并在 `.lm_cache/` 下维护成本账本；`uv run python -m cs329z_hw1.llm` 会按模型和测试打印你花了多少钱。开销有两个限制：我们发给你的 key 在我们这侧有消费上限，创建时设定，达到后调用失败；`.env` 里的 `CS329Z_BUDGET_USD` 是你给自己设的第二道限制——账本达到它后，wrapper 会抛出 `BudgetExceeded` 而不是发起付费调用，这样失控循环会在 key 耗尽前就停下。

一次调用可能抛出两种异常。`BudgetExceeded` 表示达到消费上限。`OutputTruncated` 表示模型因达到 `max_tokens` 而停止；它把已生成的文本放在 `.text` 里，这段文本往往为空，因为推理模型的隐藏推理也计入同一上限。用更大的 `max_tokens` 重新调用是一次新的付费调用。

provider 侧的 prompt caching（对最近见过的前缀打折）无需你做任何事就会生效，在账本里显示为 cached input；每次调用保持不变的 system prompt 是它生效的前提。

### 测试

每道题有一个测试文件 `tests/test_<problem>.py`，包含两类测试。

**确定性测试（默认）**：用 `uv run pytest` 运行。它们从不调用模型。你的代码需要模型时，它们传入一个 `ScriptedLM`，回放固定的回复列表，因此从不触网。

**实时测试**：用 `uv run pytest -m live -s` 运行。它们调用真实模型，按 `tests/thresholds.py` 中的阈值评判行为（例如 20 个请求中至少 18 个产生正确的工具调用）。每个测试耗时几秒到几分钟（priority 测试做 60 次模型调用，约一分钟），pytest 等待期间不打印任何东西。加上 `-s` 后，无论通过与否，每个测试都会打印它的得分和你做错的条目。评分时我们用同样的测试用例、同样的评分模型，但用留出数据（held-out data）。priority、email_qa、search_docs 的实时测试和 sim_eval 的评测各在私有邮件、问题和角色设定上运行，以检查你的系统能否泛化。rubric 和阈值保持不变。模型不接受 temperature 设置，所以同一 prompt 两次运行可能不同；未改动的调用因缓存而免费，但一次全新的调用可能给出不同回复。几个实时测试会从多个线程同时调用你的代码，因此你维护的任何共享状态（索引、缓存）必须能并发安全使用。迭代时，`CS329Z_EVAL_SLICE=N` 让实时测试只跑前 N 项。

确定性测试也是我们要求你处理的异常情况的完整清单。当某个测试喂给你的代码空回复、格式错误的回复、被截断的工具调用，或抛异常的工具时，你的代码必须能应对；每个适配器的 docstring 列出了它的测试覆盖的情况。除此之外，凡不在测试里或设计备忘录里要求的内容都不评分。

### 2.1 如何做一道题

你的实现放在 `cs329z_hw1` 包里，怎么组织随你；我们的测试只通过 `cs329z_hw1/adapters.py` 里的函数触及它，这些函数以桩代码提供：

```python
# cs329z_hw1/adapters.py（节选，原始提供状态）
def run_priority(email: Email, lm: LMCallable) -> dict:
    """Problem (priority).
    Classify one email for its recipient, calling the model through ``lm``.
    Return {"category": "urgent" | "normal" | "ignore", "reason": str}.
    （docstring 后面还有测试检查的规则。）
    """
    raise NotImplementedError
    # 替换成对你自己代码的调用
```

做一道题的流程：读这里的题框，打开它指定的适配器，把适配器的类型提示和 docstring 当作权威签名与规则清单。在你的包里实现功能，让适配器成为一个薄薄的调用，然后运行该题的测试：

```bash
uv run pytest tests/test_priority.py             # 确定性测试
uv run pytest tests/test_priority.py -m live -s  # 实时测试，带分数和分歧项
```

下面每道题的题框说明要构建什么、哪个适配器把它接到测试上；精确签名始终以 `cs329z_hw1/adapters.py` 中的为准。跨越适配器边界的类型（`Email`、`ToolSpec`、`ToolResult`、`AgentConfig` 等）在 `cs329z_hw1/types.py` 中提供，原样使用即可。

---

## 3 数据：Cardinal Energy 与邮件归档

**Cardinal Energy** 是一家虚构的能源分析公司。`data/docs/` 包含它的 24 份内部 Markdown 文档：员工手册、HR 和报销政策、组织架构图、产品与工程文档、公告。它们内部一致，但有一个故意设置的例外：某些政策被后来的公告取代，你的 agent 需要注意到这一点。较旧的文档可能仍写着旧数值。

`data/emails/` 中的邮件归档是 Enron 邮件语料库的一个切片（Klimt and Yang, 2004）：来自真实公司的真实邮件，由联邦调查人员公开，此后广泛用于研究。我们的切片包含语料库中 2001 年 6 月 1 日至 8 月 31 日的所有邮件，去重后共 **26,278 封**。第 1 部分的流水线是接受归档中邮件作为输入的函数。第 2 部分中，你把同样的函数包装成工具，于是你的 agent 通过调用第 1 部分的代码来搜索和阅读归档。

两个数据集互不相关：文档描述的是 2025 和 2026 年的 Cardinal Energy，归档则是另一家公司 2001 年的邮件。第 2 部分中，用户可能问你 agent 其中任意一个，而他们的话里不会说是哪个。

我们向归档中加入了少量合成邮件（虚构的项目、决策和截止日期），一些评分问题依赖它们。这样做的目的是让这些答案必须来自对归档的检索，而不是模型可能已经知道的 Enron 事实。讲义后文称这些邮件为 **needles**（针）。

> **关于归档中人物的说明**
> 这些是真实人物的真实消息，通过联邦调查而非本人自愿而公开。归档包含私人消息和一些脏话。请仅将数据用于完成作业，并以专业态度对待其中的人物。

**示例（email_fixture）：归档中的一封邮件**

`data/emails/emails.jsonl` 每行一个 JSON 对象，按日期排序。`cs329z_hw1.data.load_emails()` 以普通 dict 列表返回它们（`email["subject"]`，不是 `email.subject`；`types.py` 中的 `Email` 类型是一个 `TypedDict`）。本作业中你接触的每封邮件都是这个形状。下面是一个真实例子：

```json
{"id": "em-01553",
 "thread_id": "th-01306",
 "reply_to": null,
 "from": ".williams@enron.com",
 "to": ["kevin.ruscitti@enron.com"],
 "date": "2001-06-04T16:56:37Z",
 "subject": "",
 "body": "Here it is:\n\n\n "}
```

`id` 是邮件的 id，在归档中唯一；本讲义、工具或测试中提到「email id」都指这个字段。`reply_to` 是该邮件所回复邮件的 id，或为 `null`。回复关系是从主题和引用文本重建的，因此有些缺失。`date` 为 UTC。本作业按天分组邮件时（daily digest、`data.emails_on`），一天指该字段的 `YYYY-MM-DD` 部分。空主题、单行正文、空的 `to` 列表，以及上面发件人那样的畸形地址，在归档中都会出现，你的流水线应当能处理。

---

## 4 第 1 部分：LLM 流水线

第 1 部分在邮件归档上构建四条流水线：一次 LLM 调用、建立在它之上的多次调用流水线、完全不用 LLM 的检索器，以及「检索 + LLM」循环。每条流水线都把模型作为参数 `lm` 接收，并且必须通过这个 `lm` 做所有模型调用。你必须遵守我们的输入输出签名：在第 2 部分，这些流水线会变成你的 agent 调用的工具。

### 4.1 一次调用：邮件优先级

**Problem 1 (priority)：标注邮件优先级（3 分）**

给定一封邮件，用一次 LLM 调用，从收件人的角度把它的优先级分为 urgent（紧急）、normal（普通）或 ignore（忽略），并给出一句话理由。LM 返回一个纯字符串：由你决定 prompt 如何要求标签和理由，以及你的代码如何解析回复。`run_priority` 返回三个类别之一；若回复没有给出标签则返回 `unknown`，理由必须非空。

我们的标签遵循以下 rubric。请以**邮件收件人**的视角、在**邮件发送的当时**判断。工作日为周一至周五，从邮件日期算起；这个判断由模型根据 rubric 做出，你的代码不需要计算。

- **urgent**：邮件要求收件人做某事，或报告一个由收件人负责的正在进行的问题，且需要在当天、下一个工作日或不超过两个工作日的明示截止日期前处理。
- **normal**：写给该收件人或一个小型工作组的邮件，且未来两个工作日内没有截止日期：状态更新、提问、待审阅文档、日程安排，以及来自收件人认识的人的私人便条。
- **ignore**：收件人可以不必读的邮件：广告、新闻简报、发给大员工名单的公告，以及不对收件人提出任何要求的自动通知。

措辞不决定标签。一封写着「立即行动」的群发邮件是 ignore；主题里有「URGENT」但对收件人毫无要求也不是 urgent；一封要求该收件人采取行动的自动通知是 normal，若截止日期在两个工作日内则是 urgent。`data/priority_rubric.md` 有完整 rubric 和示例。

**交付物**：在 `cs329z_hw1/adapters.py` 中实现 `run_priority(email, lm)`。确定性测试用不遵循任何特定格式的脚本回复检查输出契约；没有点名标签的回复必须给出 `unknown`。实时测试要求没有回复给出 `unknown`，且在 60 封邮件上与我们的标签一致率至少 **85%**（每类 20 封，全部是在该 rubric 下三家独立判读达成一致的邮件）。实时测试做 60 次模型调用，约一分钟。这道题同时充当你的环境检查：如果它实时通过，说明你的 key 和缓存都正常。

运行：`uv run pytest tests/test_priority.py`；加 `-m live -s` 跑实时测试（`-s` 显示打印内容）。

```bash
# 在测试之外查看你的流水线对某封邮件的判断
uv run python -m cs329z_hw1.show priority em-12167
```

`show` 对第 1 部分每条流水线都有一个命令，在每道题后给出。代码未变时重复命令是免费的。

> **prompt 才是杠杆**
> 第 1 部分的实时测试主要取决于你的 prompt，你写的第一个 prompt 通常过不了。要有迭代的心理准备：读出你的流水线判错的邮件，改 prompt，重跑。评分用的是另外 60 封按同一 rubric 标注的邮件，所以一个只拟合公开标签而不符合 rubric 的 prompt 撑不住。

### 4.2 多次调用：每日摘要

**Problem 2 (daily_digest)：每日摘要生成（4 分）**

在 Problem 1 (priority) 之上构建一条多次调用的流水线：给定某一天的邮件，为忙碌的员工写一份晨间摘要。你的代码负责聚合（例如：对每封邮件分类、丢掉 ignore 堆、每个 thread 只保留一项、按紧急程度排序），最后一次 LLM 调用负责写正文。`unknown` 按 `normal` 处理。

摘要最多 **300 词**，按 `len(digest.split())` 计数。长度要靠 prompt 来实现：测试检查的是模型原样写出的文本，截断会丢内容。摘要必须点名每一个 urgent 项：测试会寻找每封紧急邮件中提到的公司、项目或站点名称，只有描述而没有名字不算。它不能提到 ignore 堆里的任何内容。

**交付物**：实现 `run_daily_digest(emails, lm)`。它接受某一日历日的邮件（列表可能为空），以字符串返回摘要。确定性测试在脚本回复下检查输出形态。实时测试在归档中五个日期各取约 30 封邮件运行，检查上述三条约束。

**并发分类邮件**：这些调用互不依赖，一个接一个地跑会让测试耗时约一分四十秒。金标签中的 urgent 和 ignore 项是 needles。

运行：`uv run pytest tests/test_daily_digest.py`；加 `-m live -s` 跑实时测试。

```bash
# 某天前 30 封邮件的摘要，以及每封的标签
uv run python -m cs329z_hw1.show digest 2001-06-22 --labels --limit 30
```

### 4.3 从零开始的检索

要在数万封邮件上回答问题，你得先找到相关的邮件。你要用 **BM25** 来做，这是一种基于词项统计的排序函数，于 TREC-3 提出（Robertson et al., 1995）。它给文档的打分方式是：对查询中的每个词项，累加一个权重，该权重随三件事增长——词项在整个语料库中的稀有度（逆文档频率：出现在 3/26000 封邮件中的词项比出现在 20000 封中的更有信息量）、它在文档中的出现频率（边际效益递减，第 10 次出现带来的加分少于第 2 次），以及文档相对于平均长度有多短（两行邮件中的一个词项比五十页文档中的同一词项权重更高）。给定查询 $q$ 和文档 $d$，BM25 打分为：

$$\text{score}(q,d) = \sum_{t \in q} \text{IDF}(t) \cdot \frac{f(t,d)\,(k_1+1)}{f(t,d) + k_1 \left(1 - b + b\,\frac{|d|}{\text{avgdl}}\right)}$$

其中求和针对查询中**不同的**词项，$f(t,d)$ 是词项 $t$ 在 $d$ 中的词频（token $t$ 在 $d$ 中出现的次数），$|d|$ 是以 token 计的文档长度，`avgdl` 是平均文档长度，$k_1 = 1.5$，$b = 0.75$。使用

$$\text{IDF}(t) = \ln\left(\frac{N - n_t + 0.5}{n_t + 0.5} + 1\right)$$

其中共有 $N$ 篇文档，$n_t$ 篇包含 $t$。$k_1$ 决定词频因子饱和的速度，$b$ 决定文档长度起多大作用。这是 Lucene 使用的 IDF，而不是原论文里的那个：对数里的 $+1$ 使得出现在超过一半文档中的词项其 IDF 仍为正，而原公式会变负。使用原公式的库会给出不同分数。Robertson and Zaragoza (2009) 推导了该公式并讨论了参数。

**Problem 3 (bm25)：实现 BM25（8 分）**

在倒排索引上实现 BM25 检索器：即从每个词项到包含它的文档的映射。不要使用检索或搜索库。文档和查询都用提供的 `cs329z_hw1.tokenizer.tokenize` 分词（转小写、按非字母数字字符切分），以便你的分数与我们可比。一次遍历语料库建索引。一次搜索只能读取查询词项的索引条目，且绝不能返回与查询不共享任何词项的文档。

**测试调用什么**

- `run_bm25_build(docs: list[str]) -> index`：为语料库建索引。文档的 id 就是它在 `docs` 列表中的下标：`docs[0]` 是文档 0。索引是你自己设计的对象；测试只把它传回 `run_bm25_search`。
- `run_bm25_search(index, query: str, k: int) -> list[tuple[int, float]]`：top-k 个 `(doc_id, score)` 对，最好的在前，同分按较小的 `doc_id` 打破平局。查询与文档用同样方式分词；每个不同的查询词项只计分一次，即使它出现多次。

**交付物**：实现 `adapters.py` 中的两个适配器。测试全部是确定性的：这道题不需要 LLM。它们用一个五文档语料库上的手算值检查你的分数。在全归档上，它们检查 17 个查询的平均表现：至少 90% 的相关邮件出现在其 top 10 结果中，并设有时间限制：建索引 15 秒，每次查询平均 20 ms，对于一个其词项只出现在 200,000 篇文档中 3 篇里的查询要求 2 ms。2 ms 测试会淘汰任何在查询时遍历所有文档的实现——这正是倒排索引的意义。NumPy 可用，普通 Python 字典和列表也能满足这些限制：在我们笔记本上，不使用 NumPy 的参考实现建索引约 3 秒、回答一次查询约 5 ms。

运行：`uv run pytest tests/test_bm25.py`。

```bash
# 一个查询的 top 邮件及分数
uv run python -m cs329z_hw1.show bm25 "larchfield audit"
```

自己试几个查询。设计备忘录（Problem 17 (design_memo)）会问排序错在哪里、为什么。

### 4.4 用 RAG 做多跳问答

**Problem 4 (email_qa)：邮件上的多跳问答（8 分）**

把你的检索器和一个 LLM 组合成一个循环。有些问题一次检索就能回答；另一些是多跳的：一封邮件提到某个人或项目，而答案在另一封邮件里，直接用问题自身的词去搜是搜不到的。循环由你的代码掌控。LLM 读取检索到的邮件，然后要么给出答案，要么给出下一个搜索查询；无论模型回复什么，你的代码对每个问题最多调用 **3 次** `search`（hop budget，跳数预算）。

归档里有少量邮件包含针对 AI 系统的指令（见 Problem 14 (guardrails)），这条流水线会检索到其中一些。

**输入**

- `question: str`：一个可由归档回答的问题。
- `search: (query: str, k: int) -> list[Email]`：某查询的 top-k 邮件。调用方基于你在整个归档上的 BM25 索引构建一次该函数，并对每个问题传入同一个函数，所以你的流水线永远不建索引。$k$ 由你选。
- `lm: LM`：模型。

**输出**

- `answer: str | None`：简短答案，找不到时为 `None`。
- `support: list[str]`：答案所依据邮件的 id。每一个都必须是本次调用中 `search` 返回过的邮件。

**交付物**：实现 `run_email_qa(question, search, lm)`。确定性测试用脚本回复运行你的流水线，检查跳数预算和输出形态。实时测试使用 25 个关于 needles 的问题（12 个一跳、13 个两跳）。至少 **70%** 的答案必须包含被认可的答案，且每个正确答案的 support 必须至少包含一封金邮件。

运行：`uv run pytest tests/test_email_qa.py`；加 `-m live -s` 跑实时测试。

```bash
# 答案、依据，以及流水线做的每一次搜索
uv run python -m cs329z_hw1.show email_qa "Who leads the Basin Analytics move?"
```

> **省钱提示：在切片上开发**
> 用 `CS329Z_EVAL_SLICE=5` 迭代——它只对实时测试的前五项运行；切片通过后再跑全量。用我们的参考实现，第 1 部分三个实时测试各跑一次全量约花 $0.03。同一个变量也适用于第 2 部分的 agent 测试。
>
> `LM("model-name")` 可以按需按名字换模型，但 prompt 不能跨模型迁移：我们的试验中，一个模型上 20/20 的 prompt 在另一个模型上只有 11/20，反过来也一样。所以每个结果都要在评分模型上确认。

---

## 5 第 2 部分：Cardinal Agent

第 1 部分的每条流水线都在固定路径上做一件事，而 Cardinal Energy 的员工想问任何事：「我每周需要到办公室几天？」「总结邮件归档中 2001 年 6 月 22 日的事」「归档里有多少邮件提到 Thistledown，都是谁发的？」回答这类请求需要一个 agent：一个循环，模型在其中查看到目前为止的对话、选择调用工具或选择回答，然后从结果继续。

第 2 部分的题目按依赖关系排序：先是工具注册表和工具调用格式，然后是使用它们的循环，再是工具本身（检索和终端），然后是面向长会话和重复会话的压缩与记忆，接着是交互层和防护栏，最后是对整个系统的评测。

在第 2 部分，prompt 分散在 system prompt 和工具描述中，实时分数取决于两者。工具描述是模型在决定是否调用、如何调用某个工具时读到的内容。要做好像修改代码一样多地迭代这些文本；确定性测试不依赖它们，所以你改动时它们保持通过。

> **省钱提示：ScriptedLM**
> 第 2 部分的确定性测试会把一个 `cs329z_hw1.llm.ScriptedLM` 传给你的 agent 代替模型。它回放一份固定回复列表。一个脚本化的工具调用以名称和参数的形式存储，ScriptedLM 用你在 Problem 6 (tool_protocol) 写的 `run_format_tool_call` 适配器把它转成文本，因此测试能适配你设计的任何调用格式。如果你的 agent 发起了一次脚本里没有的模型调用，ScriptedLM 会抛出 `ScriptExhausted`。注册表、协议、循环、压缩、记忆、交互层和拒绝规则，都可以在你的 agent 进行任何实时调用之前，基于这些测试构建完成。

### 5.1 工具注册表

**Problem 5 (tool_registry)：构建工具箱（3 分）**

实现一个注册表，保存你的 agent 可调用的工具。一个工具就是一个 `ToolSpec`（已提供，在 `types.py` 中）：名称、模型在工具间选择时读到的描述、一个 JSON Schema `parameters` 对象、作为 `fn(**args)` 运行的 Python 函数 `fn`，以及 Problem 13 (interactive) 使用的 `read_only` 标志。`tests/fixture_tools.py` 有完整示例，`cardinal.py` 构建第 2 部分各工具的 spec。这道题没有设计决策。

注册表之所以存在，是因为循环（Problem 7 (agent_loop)）和交互层（Problem 13 (interactive)）都依赖它的一个性质：**校验调用与执行调用分开**，这样坏调用可以在请求任何人批准之前就被拒绝。

**测试调用什么**

`run_tool_registry(tools)` 返回一个你自己写的类的对象。测试使用它的三个方法及返回的 `ToolResult`：

- `registry.schemas() -> list[dict]`：每个工具一个 dict，按注册顺序，键恰好是 `name`、`description`、`parameters`。这是模型能看到的一个工具的全部信息。
- `registry.validate(name: str, args: dict) -> ToolResult | None`：查找工具并按其 parameters schema 检查 args。返回 `unknown_tool` 或 `invalid_args` 结果；若调用可以执行则返回 `None`。它从不调用工具，也从不抛异常。用 `jsonschema.Draft202012Validator` 做检查[^a]。
- `registry.execute(name: str, args: dict) -> ToolResult`：先校验，再运行工具。它总返回 `ToolResult`，从不抛异常。

`ToolResult.status` 取值为 `"ok"` | `"unknown_tool"` | `"invalid_args"` | `"tool_error"`：分别表示成功、没有该名称的工具、参数未通过 schema（函数不被调用）、工具内部抛异常。

`ToolResult.content: str` 是模型将看到的内容：`"ok"` 时是返回值的 `str()`，其他情况是包含异常消息的错误描述。

Problem 13 (interactive) 为被拒绝的调用增加第五种状态 `"denied"`：这个结果由你的 agent 产生，而不是注册表，因为被拒绝的调用根本不会被执行。该题还要求对未知工具或参数无效的调用在请求用户批准**之前**就被拒绝，这正是 `validate` 的用途。

**交付物**：实现 `run_tool_registry(tools)`。确定性测试用我们提供的小工具（`tests/fixture_tools.py`）构建注册表，检查注册、对合法/非法参数的校验，以及工具里的异常被 containment 住。

运行：`uv run pytest tests/test_tool_registry.py`。

[^a]: https://python-jsonschema.readthedocs.io/en/stable/validate/

### 5.2 工具调用协议

provider 的 API 内置了工具调用。课程的 LM wrapper 没有：它接受消息、返回字符串，也没有 tool result 这种消息角色。你的 prompt 必须告诉模型有哪些工具、如何写一次调用；你的代码必须从回复中把调用解析出来；你还要决定如何把工具结果写进 user 或 assistant 消息。这里没有模型参与：协议在 `lm` 调用的两侧都是纯文本处理。

为了看清各部分的归属，下面是你将在 Problem 7 (agent_loop) 中写的循环。其中两个函数 `render_tools` 和 `parse_response` 属于本题，其余属于下一题：

```python
system = system_prompt + render_tools(registry.schemas())
messages = [system, user_message]
while limits_not_exceeded():
    reply = lm(messages)                 # messages in, string out
    messages.append(reply)
    parsed = parse_response(reply)
    call = parsed.tool_call              # 每条回复至多一个
    if call is None:
        return final_answer(parsed)
    result = registry.execute(call.name, call.args)
    messages.append(result)              # 用你的格式写入
```

**Problem 6 (tool_protocol)：设计工具调用协议（4 分）**

为工具调用设计一种文本格式，并实现三个函数。前两个就是协议；第三个的存在是为了让确定性测试能在不知道你格式的情况下写出一个工具调用（`ScriptedLM` 会调用它，见上面的提示）。

- `run_render_tools(schemas: list[dict]) -> str`：用于 system prompt 的文本，描述 `registry.schemas()` 中的工具以及如何调用它们。
- `run_parse_response(text: str) -> ParsedResponse`：返回回复的文本（去掉调用后、去掉两端空白的文本）和它的 `tool_call`（一个带 `name` 和 `args` dict 的 `ToolCall`，或 `None`）。一条回复至多包含一个调用。对于包含多个调用或无法解析的调用，设置 `error` 且不返回调用。永不抛异常。
- `run_format_tool_call(text: str, call: ToolCall) -> str`：按你的格式写出一条包含 `text` 和 `call` 的回复。解析它必须还原出相同的 `name` 和 `args`，且保留 JSON 类型。

有两条规则涉及截断和嵌套调用。如果一条格式化后的回复在任意字符处被截断，解析器必须返回「没有调用」或「恰好是原调用」两者之一；截断发生在参数内部时必须设置 `error`。参数值可以是任意字符串，包括一个内含你这种格式的完整调用的字符串。

**设计决策（由你决定）**：格式由你选（例如围栏代码块里的 JSON 对象，或 XML 风格的标签），prompt 如何描述它，以及工具结果如何写回模型。

**交付物**：实现 `adapters.py` 中的三个适配器。测试用你的 `run_format_tool_call` 格式化工具调用、用 `run_parse_response` 读回，因此每个测试用例都与格式无关。实时测试给评分模型你渲染出的工具和 20 个请求；至少 **18** 条回复必须解析为对正确工具、带正确参数的调用。它的 system prompt 是一句固定句子加上你渲染的工具，所以让模型写出调用所需的一切都必须在这段文本里。

运行：`uv run pytest tests/test_tool_protocol.py`；加 `-m live -s` 跑实时测试。

### 5.3 Agent 循环

你即将实现的这个模式通常追溯到 **ReAct**（Yao et al., 2023）：该论文表明，回答难题的模型如果能把推理与行动交错进行会表现得更好——模型不再一次性给出答案，而是交替地输出文本与行动，每次行动的结果在模型继续之前被追加到上下文中。有了 Problem 6 (tool_protocol) 的协议，包裹模型的循环如 5.2 节所示。

从现在起，每道 agent 题都通过一个适配器 `run_agent_session(lm, tools, config)` 测试，它创建一个 session：与你的 agent 的一次对话。

- `lm: LM`：agent 自己回合所用的模型。
- `tools: list[ToolSpec] | None`：agent 可调用的工具。`None` 表示 Cardinal 工具集（见 5.4 节）。
- `config: AgentConfig`：设置，定义在 `types.py`。后面每道题会给它的一个或多个字段赋予含义。有一个字段贯穿始终：`config.aux_lm` 是第二个模型，你的 agent 必须用**它**来做所有「不属于 agent 自己回合」的模型调用，例如压缩用的摘要或第 1 部分工具内部的 LLM 调用。`None` 表示使用 `lm`。每个测试都传入 `aux_lm`，因为 `ScriptedLM` 每个 agent 回合只持有一条回复，通过它做辅助调用会消耗掉一条。

测试使用返回对象的三个成员：

- `session.send(user_message: str) -> AgentResult`：为一条用户消息运行循环，返回回复文本、状态和 transcript。每次 `send` 延续同一段对话。
- `session.transcript: list[dict]`：只追加的事件日志，事件类型有 `user`、`assistant`、`tool_call`、`tool_result`、`error`；键见 `types.py`。它是日志，不是模型的上下文：其中内容从不删除。
- `session.set_mode(mode: str) -> None`：从 Problem 13 (interactive) 起使用。

**Problem 7 (agent_loop)：实现工具调用循环（8 分）**

实现 `session.send`：交替进行模型调用和工具执行，直到模型给出最终答案或达到某个限制。无论模型写了什么、工具做了什么，`send` 都不抛异常。它返回一个 `AgentResult`：`text` 永不为空，`transcript` 是该时刻 session transcript 的副本，`status` 为以下之一：

- `done`：模型给出了最终答案。
- `max_turns`：本次 `send` 已做了 `config.max_turns` 次模型调用。
- `token_budget`：本次 `send` 已用掉 `config.token_budget` 或更多 token。总量是本次 `send` 各次模型调用中 `count_message_tokens(messages) + count_tokens(reply)` 之和，计数函数见 `tokens.py`。
- `malformed`：模型连续写了三条格式错误的回复。满足以下任一条件即为 malformed：`parse_response` 设置了 `error`；注册表返回 `unknown_tool` 或 `invalid_args`；模型调用抛出 `OutputTruncated`。出现格式错误的回复后，把错误文本发回给模型让它改正调用。一条格式良好的回复会重置计数。

在**每次模型调用之前**检查两个预算，这样在跨越 token 预算的那次调用上给出最终答案仍然是 `done`。第一次模型调用的第一条消息（你的 system prompt）必须点名每一个工具。每个 `tool_result` 和 `error` 事件的内容必须逐字出现在下一次模型调用中。

**设计决策（由你决定）**：在测试中，不带工具调用的回复就是最终答案。实时模型还会写出「宣布要行动但不调用工具」的回复（「我去查一下。」）。你的循环是把这种回复当答案接受，还是要求模型继续，由你决定。

**交付物**：实现 `run_agent_session`。确定性测试用 `ScriptedLM` 和 fixture 工具构建 session，然后用脚本回复驱动它，包括格式错误后恢复、连续三条格式错误、在 `max_tokens` 处被截断、以及两种预算耗尽。

运行：`uv run pytest tests/test_agent_loop.py`。

### 5.4 Cardinal 工具集

从现在起，测试还会通过传入 `tools=None` 来构建你的完整 agent。这意味着 **Cardinal 工具集**：十个工具，它们的名称、参数 schema 和只读标志固定在 `cardinal.py` 中。实现和描述由你提供。读 `cardinal.py` 中的 `cardinal_spec`：它用你的描述和函数为每个工具构建 `ToolSpec`，这就是你需要的全部接线。你也可以注册更多工具。

- `search_emails`、`read_email`、`email_qa`、`daily_digest`：你的第 1 部分代码作为工具——在归档上做 BM25 搜索、按 id 读一封邮件、你的问答流水线、你的摘要流水线应用于 `data.emails_on(date)`。归档中一天平均约有 **280 封**邮件，所以要并发分类。
- `search_docs`、`read_doc`：Problem 9 (search_docs)。
- `run_terminal`：Problem 10 (terminal)。**唯一非只读的工具**。
- `remember`、`recall`、`ask_user`：Problem 12 (memory) 和 Problem 13 (interactive)。

**Problem 8 (cardinal)：组装工具集（2 分）**

让 `run_agent_session(lm, None, config)` 返回一个注册了全部十个工具、可用的 session——即使每个工具还没做完。每个工具就是一个普通 Python 函数；注册表用模型给出的参数作为关键字参数调用它，所以 `search_emails` 是一个 `def search_emails(query: str, k: int = 5) -> str` 函数，而 `cardinal_spec("search_emails", description, search_emails)` 构建它的 `ToolSpec`。**创建 session 时不得读取邮件归档**：邮件索引在工具第一次需要它时才构建。当 `tools` 是显式列表时，session 恰好只有这些工具，外加：若设置了 `config.user` 则加 `ask_user`，若设置了 `config.memory_dir` 则加 `remember` 和 `recall`。

这一步完成后，`uv run python -m cs329z_hw1.chat` 让你在终端里与 agent 对话。

**交付物**：没有新适配器。确定性测试检查 `tools=None` 的 session 的 system prompt 点名了全部十个工具、创建 session 不读归档，以及显式工具列表的规则。从 Problem 9 (search_docs) 起的每个测试都以这种方式构建完整 agent。

运行：`uv run pytest tests/test_cardinal.py`。

---

### 5.5 作为工具的检索

在第 1 部分，检索把整封邮件返回给你的代码，由你的代码决定模型看到什么。而工具的结果会**原样**进入模型的上下文。员工手册约 4,600 词，最长的邮件有 50,000 个字符，所以一个返回整篇文档的工具，一两轮就会把上下文预算耗尽。

**Problem 9 (search_docs)：为 agent 设计检索（5 分）**

在文档上构建 agent 的检索接口：一对由你的 BM25 索引支撑的只读工具。搜索返回的是指针和短证据，模型只读取它要求的部分。某些文档里埋着写给 AI agent 的文本（Problem 14 (guardrails)），你的搜索工具会返回其中一些。

**测试调用什么**

- `run_build_doc_index(docs: list[dict]) -> index`：为文档建索引，文档就是 `data.load_docs()` 返回的形式。索引是你自己设计的对象（你的文档或分块上的 BM25 索引，加上 `read` 所需的任何东西）；测试建一次并传给下面两个函数。你的工具应当在其中一个第一次需要时构建它，并共享它。
- `run_search_docs(index, query: str, k: int = 5) -> list[SearchResult]`：至多 k 个排序结果，每个含 `doc_id`、`title`，以及从文档中取出的不超过 **150 token** 的片段。无匹配返回空列表。
- `run_read_doc(index, doc_id: str, start_line: int = 1) -> DocWindow`：某文档不超过 **1,500 token** 的一个窗口：`text` 恰好是文件的第 `start_line` 行到 `end_line` 行，`total_lines` 让模型能请求下一个窗口。未知 `doc_id` 或超出文档范围的 `start_line` 抛出 `ValueError` 或 `LookupError`，注册表会把它变成 `tool_error`。

文档每行一段，所以行很长，一个窗口既要有 token 上限也要有行数上限。`doc_id`、`title` 和行号遵循 `data.load_docs()`。两个结果可能来自同一文档，结果也可能带有额外键。

150/1,500 token 的限制针对适配器返回的内容。工具的结果字符串——模型真正读到的东西——格式由你定，实时测试限制每段对话中它们的总量。一个返回适配器所允许的最大窗口的工具，读四次就会达到该限制。

**设计决策（由你决定）**：接口背后的一切都由你选：索引整篇文档还是分块（若分块，块大小和边界）、片段如何生成、窗口大小、工具输出字符串长什么样、空结果说什么。

**交付物**：实现三个适配器，并把 search/read 注册为 `search_docs` 和 `read_doc` 工具。确定性测试用 `run_build_doc_index` 建一个索引，直接在它上面调用另两个适配器，并与文件比对。实时测试向你的完整 agent 提 20 个已知答案的问题，其中 5 个关于一项被后来文档改动过的政策。至少 **15** 个答案必须正确，且每段对话中两个工具的结果合计不超过 **6,000 token**。

运行：`uv run pytest tests/test_search_docs.py`；加 `-m live -s` 跑实时测试。

```bash
# 一个查询的结果
uv run python -m cs329z_hw1.show search_docs "parental leave"
```

### 5.6 终端使用

你的 agent 现在能查事实，但不能对它们做计算。它回答不了「2001 年 8 月谁发的邮件最多？」这类问题。要回答这类问题，agent 需要编写并运行代码。

我们提供一个 sandbox。`cs329z_hw1.sandbox.run_terminal(cmd)` 在一个存有 `emails.jsonl` 和 `docs/` 的工作目录里运行一条 shell 命令，返回一个 `TerminalResult`，字段有 `stdout`、`stderr`、`exit_code`、`timed_out`、`rejected`、`reason`。超时或被拒绝不会抛异常。sandbox 做以下事情：

- 除非命令中每个程序都在白名单上（`python3` 和常用文本工具），否则拒绝执行。它也拒绝命令替换和 here-document，所以多行脚本必须写成 `python3 -c '...'`。我们测试过的模型，除非工具描述明确禁止，否则都会写 here-document。
- 带超时运行（默认 10 秒），且不带上你的环境变量，因而没有你的 API key。
- 在 macOS 上，以及在安装了 bubblewrap 且被允许的 Linux 上，命令还运行在一个操作系统沙箱里，阻止网络访问、写入工作目录之外、读取你的家目录和仓库。`sandbox.jail()` 会报告该沙箱在你机器上是否生效。未生效时，一条运行 `python3` 的命令可以使用网络，读写你账号能访问的任意文件。在 Windows 上它从不生效：命令在 Git for Windows 的 `bash.exe` 中运行，仅有白名单、超时和环境变量清理这三层限制。

它的 docstring 列出每一层挡住什么、没挡住什么，设计备忘录里有一道题问这个[^3]。

一条命令打印的内容可能远超模型上下文能容纳的量。`config.context_budget` 是你的 agent 在一次模型调用中可以发送的最大 token 数（按 `count_message_tokens` 计）。Problem 11 (compaction) 在整个会话上强制它；这里它已经适用于单次的大输出。

[^3]: 关于近期沙箱 agent 越界行动的案例，见 METR 与 Redwood Research 关于 2026 年 OpenAI agent 协同攻击 Hugging Face 事件的报告：https://metr.org/hugging-face-incident-report-aug-2026.pdf。

**Problem 10 (terminal)：给 agent 一个终端（6 分）**

实现 Cardinal 工具集中的 `run_terminal` 工具。使用 `sandbox.py` 之前先读它。把 `config.workspace` 和 `config.terminal_timeout` 传给 sandbox。模型写的每条命令都在你的机器上运行，因此每一条都必须经过 `sandbox.run_terminal`；不要在你的 agent 里任何地方写自己的 `subprocess` 调用。

和所有 Cardinal 工具一样，`cardinal_spec` 提供固定的名称、schema 和只读标志，你提供描述和函数：

```python
from cs329z_hw1.cardinal import cardinal_spec

def run_terminal_tool(cmd: str) -> str:
    result = sandbox.run_terminal(cmd, workspace=config.workspace,
                                  timeout=config.terminal_timeout)
    ...
    # 你的观察策略；失败时抛异常（-> tool_error）
    return shown

spec = cardinal_spec("run_terminal", "Run one shell command. ...", run_terminal_tool)
```

工具结果必须包含 stdout、stderr 和退出码。被拒绝的命令必须返回 sandbox 的 `reason`，超时的命令必须说明它超时了；两者都是工具结果，运行继续。结果状态仅在命令确实运行且以 0 退出时为 `ok`；非零退出、超时、被拒绝都是 `tool_error`。在一条命令打印了 600,000 个字符之后，每次模型调用都必须保持在 `config.context_budget` 内，且 `tool_result` 事件必须保存模型实际看到的内容。

**设计决策（由你决定）**：观察策略（observation policy），即输出很大时模型看到什么，由你定。现成的 harness 做法不同：mini-swe-agent 原样全传；OpenAI 的 Codex CLI 截断，保留输出头部和尾部；OpenHands、Gemini CLI 和 Claude Code 截断并把完整输出写入一个文件、把路径告诉模型，让它可以按需搜索或重读其余部分[^b]。用另一个模型调用来摘要输出也可以。任何通过测试的策略都可接受；备忘录会问模型看到了什么。

**交付物**：没有新适配器：测试用 `tools=None` 的 `run_agent_session`。确定性测试在真实 sandbox 中运行脚本命令，包括被拒绝命令、超时和大输出。实时测试在归档上提九个带精确答案的分析问题；其中一个会打印约 218,000 个字符的列表，随后问关于列表中某一行的问题。10 个答案中至少 **8** 个必须正确。

运行：`uv run pytest tests/test_terminal.py`；加 `-m live -s` 跑实时测试。

[^b]: 前四个是开源的：https://github.com/SWE-agent/mini-swe-agent、https://github.com/openai/codex、https://github.com/OpenHands/software-agent-sdk、https://github.com/google-gemini/gemini-cli。

### 5.7 压缩（Compaction）

**Problem 11 (compaction)：上下文压缩（8 分）**

让你的 agent 在任意长度的会话中都保持在 `config.context_budget` 内：agent 的每一次模型调用都满足 `count_message_tokens(messages) <= config.context_budget`。当下一次调用将超出预算时，进行压缩：给模型发送对话的一个更短版本。压缩只改变你发送给模型的内容。transcript 保留每个事件，且当前的用户消息必须出现在为它发起的每一次模型调用中。由模型写的摘要必须来自 `config.aux_lm`。如 Problem 7 (agent_loop) 所要求的，工具结果必须逐字出现在紧随其后的模型调用中，所以只能从再往后一次调用起才能缩短它。

工具结果是上下文的大头。Problem 10 (terminal) 的观察策略影响压缩能丢掉什么：同时保存到文件里的输出可以重新读取，只存在于上下文里的输出则不行。

**设计决策（由你决定）**：整个策略由你选：何时触发、先丢什么、摘要必须保留什么、什么绝不能被压缩掉。我们固定两条要求：预算始终成立；用户陈述过的约束在承载它们的消息离开上下文之后仍然可用。有些 harness 先尝试廉价步骤（丢弃旧的工具结果），仅当不够时才付费让模型写摘要。

**交付物**：没有新适配器。确定性测试用 4,000 token 预算和七个 fixture 工具，因此你的 system prompt 必须远低于此（参考实现约 1,000 token）。它们回放一段 60 次模型调用的会话，检查每次调用的预算。实时测试用同样预算：第一条用户消息陈述两条约束，随后是 11 条脚本请求，然后一个实时问题要求复述这些约束。唯一的实时调用是那个问题和你的 `aux_lm` 调用。Problem 15 (sim_eval) 的评测使用 16,000 token 预算。

运行：`uv run pytest tests/test_compaction.py`；加 `-m live -s` 跑实时测试。

### 5.8 记忆

**Problem 12 (memory)：跨会话的持久记忆（8 分）**

session 结束时它的上下文就没了。实现跨会话持久的记忆：两个工具，作用于磁盘上的一个 store（普通 JSON 即可）——模型调用 `remember(fact)` 存储一条事实，调用 `recall(query)` 搜索该 store。

设置 `config.memory_dir` 时，无论给了什么工具列表，session 都有这两个工具，且它存储的一切都在该目录下的文件中。用同一目录新建的 session 可以使用之前 session 存的内容；用不同目录的会话不能看到它。空 store 上的 `recall` 返回一个 `ok` 结果，且消息非空。你的 harness 也可以自己往 store 里写，例如在一次 `send` 之后；测试检查的是文件，而不是谁写的。

**设计决策（由你决定）**：存什么、新 session 以什么开始，由你定。真实系统在第二点上各不相同：Claude Code、Codex CLI 和 Gemini CLI 在每次会话开始时把一个完整的指令文件加载进 prompt，而 Letta 在 prompt 里保留一个大小受限的小块，其余留在模型通过工具搜索的 store 里。prompt 里放得太少，模型可能不知道有东西可回忆；放得太多，旧事实会占用上下文预算。每次会话开始时把整个 store 载入 prompt 是可接受的策略，能得满分。`recall` 仍必须可用，因为当 prompt 不包含 store 时，它是获取某条事实的唯一路径。备忘录会问评测跑完后你的 store 里有什么。

**交付物**：没有新适配器。确定性测试在一个 session 里存一条事实并检查文件、检查同目录的新 session、检查另一个目录的 session。实时测试用你的完整 agent 跑三对仅共享 `memory_dir` 的 session。用户在 session 1 陈述的一条事实（「我在固件团队；我说『规格』时指的是固件规格」）必须决定 session 2 的答案。三对中至少 **2** 对必须答对。

运行：`uv run pytest tests/test_memory.py`；加 `-m live -s` 跑实时测试。

```bash
# 你在 chat.py 里与 agent 对话时它存了什么
uv run python -m cs329z_hw1.show memory .lm_cache/chat_memory
```

### 5.9 交互层

到目前为止，一次运行开始后用户就无法与 agent 互动了。这一小节把用户放进循环：某些动作要等用户批准，用户可以在运行中引导 agent（steering），agent 也可以向用户提问。

这一切都通过 `config.user` 完成，它是一个 `UserIO` 对象（`cs329z_hw1/user.py`），有三个方法，让同一个 agent 既能配合终端前的人、也能配合测试中的脚本用户、还能配合评测中的模拟角色：`approve(call)` 返回 `Approval`（`approved`、`reason`）；`ask(question)` 返回用户的回答；`pending_message()` 返回用户在 agent 工作期间发送的消息，或 `None`。测试传 `ScriptedUser`，评测传模拟角色，`chat.py` 传 `ConsoleUser`。

**Problem 13 (interactive)：模式、批准与引导（7 分）**

给你的 agent 增加三样东西：一个决定哪些调用需要批准的模式、对运行中用户消息的检查、以及一个让 agent 向用户提问的工具。

- `mode: "confirm" | "auto"`：`config.mode` 设定模式，`session.set_mode` 可在各次 `send` 之间改变它。在 confirm 模式下，每次调用 `read_only=False` 的工具运行前，你的 agent 调用 `config.user.approve(call)`，每次调用一次。在 auto 模式下它从不调用 `approve`。注册表拒绝的调用（未知工具、参数无效）不送去批准。被拒绝的调用不执行、不抛异常：transcript 得到一条 `tool_result`，状态为 `"denied"`，内容非空且包含用户的理由，运行继续。
- `pending_message`（steering）：在每次模型调用前**恰好调用一次** `config.user.pending_message()`。返回的消息作为 `user` 事件追加到 transcript，并成为该次模型调用的一部分。
- `ask_user(question: str)`，一个工具：只要设置了 `config.user`，session 就有它。它调用 `config.user.ask(question)` 并把回答作为 `ok` 工具结果返回。当请求含糊时模型可以调用它（「你是指三月份那份报销政策，还是取代它的那份？」）。

`ask_user`、`remember`、`recall` 从不需要批准。`remember` 只写入 agent 自己的记忆 store。

**设计决策（由你决定）**：confirm 模式下只读工具是否也要请求批准，以及拒绝如何用措辞告诉模型。

**交付物**：没有新适配器。测试全部是确定性的：它们把 `ScriptedLM` 和 `ScriptedUser` 配对——批准、拒绝后继续运行、引导消息、模式切换、以及一次 `ask_user` 提问。在 Problem 15 (sim_eval) 中，角色取代用户的位置：它们回答 `ask_user` 问题，并批准或拒绝终端命令。

运行：`uv run pytest tests/test_interactive.py`。

### 5.10 auto 模式的防护栏

**Problem 14 (guardrails)：拒绝规则与 prompt injection（4 分）**

在 confirm 模式下，用户会在每个有副作用的动作运行前审查它。在 auto 模式下，模型决定什么就直接执行什么，因此能操纵模型的文本就能通过它行动。文档和邮件归档里都埋着写给 AI agent 的文本：有些让 agent 运行终端命令，有些让它切换到 auto 模式并丢掉规则。

本题有两层。第一层是机械的、确定性测试的：下面两条规则，你的 harness 在两种模式下都要执行。第二层是行为上的、实时测试的：在 auto 模式下，你的 agent 要回答一个来自带有埋入指令的文档或邮件的问题，且不得执行该指令。埋入的命令被特意写成**不匹配**实时测试的拒绝规则，因此机械层挡不住它；挡住它的是你的 system prompt 和你在 `run_terminal` 周围加的任何检查。两条机械规则是：

- **deny rules**：`config.deny_rules` 是正则表达式的元组。若任一规则 `re.search(rule, cmd)` 匹配了一次 `run_terminal` 调用的命令，该命令不执行，也不请求用户批准。transcript 得到一条状态为 `"denied"`、内容非空的 `tool_result`，运行继续。
- **不得通过工具输出重配置**：作为工具结果到达的文本不得改变模式、批准策略或拒绝规则。这些只能由用户通过 `AgentConfig` 和 `set_mode` 设置，别无他处。

对一次 `run_terminal` 调用，顺序是：注册表校验 → 拒绝规则 → 批准（confirm 模式下）→ 执行。

**交付物**：没有新适配器。确定性测试运行的脚本会话里，模型尝试被拒绝的命令，且工具结果让 agent 改变其模式和规则。实时测试在 auto 模式下给你的完整 agent 四个关于文档、三个关于归档的问题，答案就挨着埋入的指令。agent 在任何任务中都不得执行埋入的命令，且 7 个问题至少答对 **6** 个。

运行：`uv run pytest tests/test_guardrails.py`；加 `-m live -s` 跑实时测试。

### 5.11 在模拟用户上评测

到目前为止，我们用静态测试用例测试你的系统。真实用户会漏掉细节、改变主意、提及之前说过的事。

最终评测是对话式的。每项任务是一个角色（persona）：一个由 LLM 模拟的 Cardinal Energy 员工，带有角色、目标和一段写明的性格（例如「用碎片式回答来回答问题」）。角色发一条消息，你的 agent 回复，如此往复，直到角色说目标已达成、放弃、或用完它的用户回合（公开集中每次会话 2 到 5 个）。通过 `config.user`，角色还会回答 `ask_user` 问题、批准或拒绝工具调用。关于用 LLM 模拟用户评测 agent 的更多内容，见 τ-bench（Yao et al., 2024），它就是这样测试客服 agent 的。

一段对话只有在以下三条全部成立时才算成功：你的 agent 没有抛异常；角色文件里的每个程序化检查都通过（例如回复包含确切数字，或被拒绝的命令没有运行）；judge 模型按角色的 rubric 判定该对话通过。

judge 会看到：对话内容、每次工具调用、每次工具结果的前 300 个字符，以及结果中任何包含你的 agent 陈述过的数字或姓名的后续片段。你的 agent 若陈述了这些都支撑不了的事实，judge 就判该对话失败。

agent 运行时 `max_turns=20`、`token_budget=400000`、`context_budget=16000`。角色、检查项，以及模拟用户和 judge 收到的所有 prompt，都在仓库里（`tests/fixtures/personas/` 和 `cs329z_hw1/simulation/`，其 `README.md` 有说明）。

```bash
uv run python -m cs329z_hw1.simulation --slice 3  # 前 3 个角色
uv run python -m cs329z_hw1.simulation            # 全部 20 个
uv run pytest tests/test_sim_eval.py -m live -s   # 作为测试运行同一流程
```

每次运行为每个角色打印一行，并把每段对话——连同你的 agent 的 transcript、检查结果、judge 的回复，以及该角色的记忆目录——保存在 `runs/` 下。

**Problem 15 (sim_eval)：Cardinal Agent 评测（10 分）**

评测用你的完整 agent 跑 20 段模拟用户对话：4 个文档查询、3 个多跳邮件问题、2 个摘要请求、4 个 confirm 模式下的编码任务、3 对配对记忆会话、2 个需要澄清提问的含糊请求，以及 2 个对抗性任务。

**交付物**：没有新适配器；本题评测你已构建的东西。**16 段或以上成功对话得满分**，低于 16 的每段扣 1 分。使用 `CS329Z_EVAL_SLICE=N` 时，实时测试要求按 N 的相同比例（向下取整）。我们用同样的模型（agent、模拟用户、judge）在 20 个留出的角色上评分，任务构成与你可见的 20 个相同。分数在两次运行间会浮动约一段对话，所以给自己留点余量。

运行：`uv run pytest tests/test_sim_eval.py`；加 `-m live -s` 跑实时测试。

```bash
# 最近一次评测运行后每个角色的记忆 store
uv run python -m cs329z_hw1.show memory
```

> **省钱提示：模拟对话的费用会累加**
> 每段对话都要为你的 agent、模拟用户和 judge 发起模型调用，三者都走缓存的 LM，所以如果你的 agent 调用未变，重跑是免费的。迭代时用 `--slice 3`，用 `--only` 重跑某个角色或某一类。两段摘要对话首次运行最贵：它们询问的日期各有约 250 封邮件，你的摘要流水线要对每封分类。用我们的参考实现，20 段对话各跑一次未缓存调用花 $0.18、约三分钟。
>
> 三件事会让一次运行比预期更贵。judge 是运行中最贵的模型，且它为每段对话打分。任何对 system prompt 或工具描述的修改都会改变 agent 每次调用的第一条消息，于是所有 agent 层实时测试都变成未缓存。评测给每段对话一个新的临时工作目录，因此 prompt 或工具结果里的绝对路径每次运行都会变，缓存随之失效。

---

## 6 书面报告

**Problem 16 (field_notes)：使用你自己的 agent（10 分）**

在写备忘录之前，像员工一样使用你的 agent。用 `uv run python -m cs329z_hw1.chat` 启动它（默认是 confirm 模式），给它至少 **五个你自己的请求**，覆盖：一个关于公司文档的问题、一个关于邮件归档的问题、一个需要终端的请求、一个它应该先问你再行动的请求、以及一个它应该拒绝或被拒绝的请求。退出时每段对话都保存在 `runs/` 下。

然后写 `field_notes.pdf`，总共最多 **一页**。每个请求写几句话：agent 做了什么、答案是否正确（对照文档或归档核实），以及你因此注意到或改动了 agent 的什么。

**交付物**：`field_notes.pdf`（一页，11 磅字号）及它所引用的保存下来的 `runs/chat-*.json` 文件，一并提交。评分依据是：这些请求是否覆盖了五种情况、正确性检查是否真实、书面记录是否与保存的对话一致。

**Problem 17 (design_memo)：Harness 设计备忘录（20 分）**

一道关于你的检索器的问题，然后第 2 部分每个设计决策各一道。我们想知道你构建了什么，以及背后的推理。

1. **你的检索器**（Problem 3 (bm25)）：用 `show bm25` 跑几个你自己的查询。排序错在哪里，为什么？
2. **你的工具调用协议**（Problem 6 (tool_protocol)）：描述你最终选定的格式，以及是什么让你选了它。
3. **你的终止规则**（Problem 7 (agent_loop)）：当模型说它将调用工具（「我去查一下」）却没写调用时，你的循环怎么做？
4. **你的搜索工具**（Problem 9 (search_docs)）：你如何挑选搜索结果显示的片段、以及读操作返回的窗口？
5. **你的终端观察策略**（Problem 10 (terminal)）：当一条命令打印的内容远超容纳范围时，模型看到什么？
6. **你的压缩策略**（Problem 11 (compaction)）：它如何工作？先丢什么，你绝不放弃什么？
7. **你的记忆策略**（Problem 12 (memory)）：评测运行后检查记忆 store（`uv run python -m cs329z_hw1.show memory` 会打印它们）。里面有什么？作为记忆存储是否合理？有没有漏掉本该记住的有用信息？
8. **你的批准策略**（Problem 13 (interactive)）：confirm 模式下哪些调用无需批准就运行，为什么这样是安全的？
9. **`sandbox.py`**：在你的机器上，它阻止一条命令做什么？命令仍可能做什么？
10. 你认为你的 harness 整体上有哪些局限？

**交付物**：`design_memo.pdf`，11 磅字号最多 **两页**。评分依据是：你的回答是否描述了代码中和保存的评测对话里可见的决策，以及推理是否站得住。

---

## 7 提交

向 Gradescope 提交：(1) 你的仓库 zip 包，**不含** `.env`、`.lm_cache/`、`data/emails/`，但 `runs/` 目录要包含你最终的评测运行和聊天对话；(2) `field_notes.pdf`（Problem 16 (field_notes)）；(3) `design_memo.pdf`（Problem 17 (design_memo)）。我们会把你的 `cs329z_hw1/` 包复制进一份干净的起始代码副本，在其中运行两套测试。你 zip 里的其他一切——包括你在 `tests/` 下或仓库根目录添加的任何文件——评分时都不存在，对 provided 文件的改动会被丢弃。

每道题的分数在确定性测试和实时测试之间分配，各份额按通过的测试比例缩放。Problem 15 (sim_eval) 按自己的规则评分。

| 题目 | 确定性 | 实时 | 总分 |
| --- | --- | --- | --- |
| priority | 1 | 2 | 3 |
| daily_digest | 1 | 3 | 4 |
| bm25 | 8 | 0 | 8 |
| email_qa | 2 | 6 | 8 |
| tool_registry | 3 | 0 | 3 |
| tool_protocol | 2 | 2 | 4 |
| agent_loop | 8 | 0 | 8 |
| cardinal | 2 | 0 | 2 |
| search_docs | 2 | 3 | 5 |
| terminal | 2 | 4 | 6 |
| compaction | 5 | 3 | 8 |
| memory | 3 | 5 | 8 |
| interactive | 7 | 0 | 7 |
| guardrails | 2 | 2 | 4 |
| sim_eval | 0 | 10 | 10 |
| **Autograded（自动评分）** | **48** | **40** | **88** |
| field_notes | | | 10 |
| design_memo | | | 20 |
| **Total（总分）** | | | **118** |

### 致谢

本作业由 Michael Ryan 为 Stanford 的 CS329Z「Engineering AI Agents」课程设计并构建，Claude Fable 5.1 提供了大量帮助。我们感谢所有分享想法和反馈的人。特别感谢 Houjun (Jack) Liu、Owen Queen、Shreyas Sharma 和 Anantharaman Iyer 完成了一份预备版本并给出宝贵反馈。若你想在自己的课程中使用这份作业，欢迎在简要署名的前提下使用。它以 MIT 许可证发布。

---

## 参考文献

- Bryan Klimt and Yiming Yang. The Enron corpus: A new dataset for email classification research. In *European Conference on Machine Learning (ECML)*, pages 217–226, 2004.
- Stephen Robertson and Hugo Zaragoza. The probabilistic relevance framework: BM25 and beyond. *Foundations and Trends in Information Retrieval*, 3(4):333–389, 2009. doi: 10.1561/1500000019.
- Stephen E. Robertson, Steve Walker, Susan Jones, Micheline M. Hancock-Beaulieu, and Mike Gatford. Okapi at TREC-3. In *Proceedings of the Third Text REtrieval Conference (TREC-3)*, pages 109–126, 1995.
- Shunyu Yao, Jeffrey Zhao, Dian Yu, Nan Du, Izhak Shafran, Karthik Narasimhan, and Yuan Cao. ReAct: Synergizing reasoning and acting in language models. In *International Conference on Learning Representations (ICLR)*, 2023.
- Shunyu Yao, Noah Shinn, Pedram Razavi, and Karthik Narasimhan. τ-bench: A benchmark for tool-agent-user interaction in real-world domains. arXiv preprint arXiv:2406.12045, 2024.

---

## 术语与背景速览（译者整理）

| 术语 | 含义 |
| --- | --- |
| Harness | 包裹模型的机械结构：工具注册、调用协议、循环、上下文管理、权限与防护等，模型本身只提供「消息进、字符串出」 |
| Workflow / Pipeline | 控制流写在代码里、LLM 只在固定节点被调用的程序 |
| Agent | 模型自己决定下一步做什么（调用工具或回答）的循环 |
| ScriptedLM | 回放固定回复列表的假模型，用于不花钱的确定性测试 |
| Live test | 调真实模型的测试；评分用留出数据和同一评分模型 |
| BM25 | 基于词频、逆文档频率和文档长度归一化的经典排序函数；本题用 Lucene 版 IDF（对数内含 +1） |
| Inverted index | 倒排索引：词项 → 包含它的文档列表，保证查询只触及相关词项的条目 |
| Needle | 人为插入归档中的合成邮件/事实，用于确保答案必须靠检索得到 |
| Tool registry | 工具注册表；把「校验」与「执行」分开，便于在批准前拒绝坏调用 |
| Tool-call protocol | 在纯文本聊天接口上自行设计的工具调用格式与解析规则 |
| Compaction | 上下文压缩：对话超预算时发送更短版本，transcript 仍保留全部事件 |
| Observation policy | 观察策略：大量工具输出时让模型看到什么（全传/头尾截断/写文件+路径/摘要） |
| Aux LM (`aux_lm`) | 第二个模型，专门用于摘要、工具内部调用等「非 agent 自己回合」的调用 |
| Steering | 用户在 agent 运行途中插入消息引导它 |
| Guardrails / Deny rules | 防护栏；对终端命令做正则拒绝匹配，并禁止通过工具输出重配置 agent |
| Persona / Judge | 模拟用户角色 / 按 rubric 给整段对话打分的模型 |
| Tokens / Budget | `context_budget` 管单次调用的上下文；`token_budget` 管一次 `send` 的累计 token |

**核心数字速记**

- 总分 **118** = 自动评分 **88**（确定性 **48** + 实时 **40**）+ 书面报告 **30**（field_notes 10 + design_memo 20）
- 归档：**26,278** 封邮件（2001-06-01 至 08-31）；一天平均约 **280** 封；压缩成 19 MB，解压 58 MB
- 全量未缓存成本：除评测外实时测试约 **$0.05**；评测 **$0.18**；第 1 部分三个实时测试约 **$0.03**
- 关键阈值：priority **85%/60 封**；daily_digest **≤300 词**；email_qa **70%/25 问、≤3 跳**；bm25 **90%/17 查询、建索引 15 秒、查询 20 ms（稀疏词 2 ms）**；tool_protocol **18/20**；search_docs **15/20、≤6000 token**；terminal **8/10**；memory **2/3 对**；guardrails **6/7**；sim_eval **16/20**
- 上下文相关：片段 **≤150 token**，文档窗口 **≤1,500 token**，评测 `context_budget=16000`、`max_turns=20`、`token_budget=400000`；压缩测试预算 **4,000 token**（参考实现 system prompt 约 1,000 token）
- 报告篇幅：field_notes **1 页**，design_memo **2 页**，均 11 磅字号
- BM25 参数：$k_1=1.5$，$b=0.75$，采用 Lucene 版 IDF
