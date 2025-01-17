---
title: Building Effective Agents
published: 2025-01-16 15:30:00
description: 'How to build effective agents.'
image: ''
tags: ['Agent', 'LLM']
category: 'LLM'
draft: false


---

# agents 是什么（译）

预定义的 workflow 和长时间独立操作的都被认为是 agentic systems，但是两者架构存在区别：

-   workflows：LLMs 和工具通过预定义的代码路径编排；
-   agents：LLMs 动态地引导他们自己的执行和工具使用，维持他们如何完成任务的控制系统。

## 什么时候使用/不使用 agents

当使用 LLMs 构建应用时，建议寻找可能的最简单的解决方案，并且只在必要时增加复杂度。这可能意味着根本不建立 agentic system。agentic systems 通常在延迟和成本与更好任务性能中权衡，并且你应该考虑何时这个权衡是合理的。

当更高复杂度被期待，workflows 在被良好定义任务中提供可预测性和一致性，而当需要灵活性和模型驱动的决策时，agents 是更好的选择。但是对于许多应用，通过检索和上下文示例优化单次 LLM 请求通常足够了。

## 何时与如何使用框架

有许多框架使得实现 agentic system 更加简单，包括：

-   [LangGraph](https://langchain-ai.github.io/langgraph/) from LangChain;
-   Amazon Bedrock's [AI Agent framework](https://aws.amazon.com/bedrock/agents/);
-   [Rivet](https://rivet.ironcladapp.com/), a drag and drop GUI LLM workflow builder; and
-   [Vellum](https://www.vellum.ai/), another GUI tool for building and testing complex workflows.

这些框架通过简化标准底层任务，如调用 LLMs、定义和解析工具、和链式调用一起使得非常容易开始。但是他们通常创建了额外的抽象层，以致模糊了依赖的 prompts 和 responses，使得他们难以 debug。它们还可能使添加复杂性变得诱人，而实际上更简单的设置就足够了。

我们建议开发者从直接使用 LLM APIs 开始：许多模式只需要少数行代码就能实现。如果你使用一个框架，确保你理解了依赖的代码。关于底层情况的错误假设是客户错误的常见来源。

## Building blocks, workflows, and agents

从基础构建块，增强的 LLMs，逐渐增加复杂性。

agentic system 的基础构建块是一个使用如检索，工具和记忆增强的 LLM。我们当前的模型能够使用这些能力：

-   生成他们自己的检索查询；
-   选择合适工具；
-   决定保留哪些信息；

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2Fd3083d3f40bb2b6f477901cc9a240738d3dd1371-2401x1000.png&w=3840&q=75)

我们建议集中注意力在实现的两个关键方面：

1.  根据你的特定使用场景调整这些能力；
2.  保证他们为你的 LLM 提供了一个合适，富有文档信息的接口；

有许多实现这些增强的方式，其中之一是通过我们近期发布的 Model Context Protocol，允许开发者使用一个简单的客户端实现去与逐渐成长的第三方工具生态集成。

在本文的后续部分，我们会假定每个 LLM 请求都具备这些增强能力。

### Workflow：Prompt chaining

Prompt chaining 拆解一个任务为一系列步骤，每个 LLM 请求处理前一个输出。可以在任何中间步骤添加程序检查以确保处理流程在预定轨迹上。

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F7418719e3dab222dccb379b8879e1dc08ad34c78-2401x1000.png&w=3840&q=75)

**什么时候使用 workflow**：当任务能简单地并且清晰地拆解为固定子任务时 workflow 是理想的。主要目标是通过简化每个 LLM 调用的任务，以提高准确性而牺牲延迟：

#### prompt chaining 有用的例子：

-   生成营销文案，并且将其翻译为一个不同语言；
-   编写文档大干，检查大纲是否符合标准，之后基于大纲编写文档；

### Workflow：Routing

routing 识别一个输入并且将其导向到一个指定的后续任务。这个 workflow 允许职责分离，和构建更特定的 prompt。没有这个 workflow，对于某一类输入的优化会影响其他输入的性能。

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F5c0c0e9fe4def0b584c04d37849941da55e5e71c-2401x1000.png&w=3840&q=75)

**什么时候使用 workflow：**routing 在处理复杂任务时效果良好，在这些任务中，存在更适合单独处理的不同类别，并且可以通过大型语言模型（LLM）或更传统的分类模型/算法准确进行分类。

#### routing 有用的例子：

-   将不同类型的客户服务查询（一般问题、退款请求、技术支持）引导到不同的下游流程、提示和工具中；
-   将一般/常见问题引导到像Claude 3.5 Haiku这样的小型模型，而将困难/不寻常的问题引导到像Claude 3.5 Sonnet这样的更强大模型，以优化成本和速度；

### Workflow：Parallelization

大型语言模型（LLMs）有时可以同时处理一个任务，并通过程序化方式汇总它们的输出。这种工作流程，称为并行化，表现为两种主要变体：

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F406bb032ca007fd1624f261af717d70e6ca86286-2401x1000.png&w=3840&q=75)

1.  分块：将任务分解为独立的子任务并行运行；
2.  投票：多次允许相同的任务获得输出分布；

**什么时候使用 workflow**：并行化在被划分的子任务可以并行化以提高速度，或需要多个视角或尝试以获得更高置信度的结果时是有效的。对于具有多个考虑因素的复杂任务，当每个考虑因素由单独的LLM调用处理时，LLM通常表现更好，这样可以对每个具体方面进行集中注意。

#### parallelization 有用的例子：

-   Sectioning：
    -   实施保护措施，其中一个模型实例处理用户查询，而另一个实例筛选不当内容或请求。与让同一个语言模型调用处理保护措施和核心响应相比，这种方法往往表现更好；
    -   自动化评估LLM性能评估，其中每次LLM调用评估模型在给定提示上的不同性能方面；
-   Voting：
    -   审查一段代码以查找漏洞，其中多个不同的提示会对代码进行审查，并在发现问题时标记代码；
    -   评估给定内容是否不当，采用多个提示评估不同方面或要求不同投票阈值，以平衡假阳性和假阴性；

### Workflow：Orchestrator-workers

在调度者-工人 workflow 中，一个中心 LLM 动态地分割任务，将他们委托给 worker LLMs，并组合他们的结果。

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F8985fc683fae4780fb34eab1365ab78c7e51bc8e-2401x1000.png&w=3840&q=75)

**什么时候使用这个 workflow**：这个 workflow 非常适合无法预测子任务需要的场景（在 coding，例如需要改变的文件数量和每个文件中需要发生的变化取决于任务）。经过虽然在形式上与并行化相似，但与并行化的主要区别在于它的灵活性——子任务不是预先定义的，而是由协调者根据特定输入决定的。

#### orchestrator-workers 有用的例子：

-   coding 产品每次对多个文件进行复杂改变；
-   search 任务包括搜集和分析来自多个源的信息以得到可能的相关信息。

### Workflow：Evaluator-optimizer

在评估者-优化器工作流中，一个大型语言模型的调用生成响应，而另一个则在循环中提供评估和反馈。

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F14f51e6406ccb29e695da48b17017e899a6119c7-2401x1000.png&w=3840&q=75)

**什么时候使用这个 workflow**：这种工作流程在我们拥有清晰的评估标准，并且迭代优化提供可衡量的价值时特别有效。良好契合的两个迹象是：第一，当人类表达他们的反馈时，LLM 的响应可以明显改善；第二，LLM 可以提供这样的反馈。这类似于人类作者在创作精美文档时可能经历的迭代写作过程。

#### evaluator-optimizer 有用的例子：

-   文学翻译中存在一些细微差别，翻译的语言模型可能起初无法捕捉，但评估的语言模型可以提供有用的批评；
-   需要多轮搜索和分析以收集全面信息的复杂搜索任务，其中评估者决定是否需要进一步搜索；

## Agents

随着大型语言模型（LLMs）在关键能力上不断成熟，代理（Agents）在生产中逐渐兴起——理解复杂输入、参与推理和规划、可靠地使用工具以及从错误中恢复。代理的工作通常从与人类用户的命令或互动讨论开始。一旦任务明确，代理就会独立规划和执行，可能会返回询问人类以获取更多信息或判断。在执行过程中，代理在每一步都至关重要地需要从环境中获取“真实情况”（如工具调用结果或代码执行），以评估其进展。代理可以在检查点或遇到障碍时暂停以获取人类反馈。任务通常在完成时终止，但也常常包括停止条件（如最大迭代次数）以保持控制。

Agents 可以处理复杂的任务，但它们的实现通常很简单。它们通常只是基于环境反馈循环使用工具的 LLM。因此，清晰和深思熟虑地设计工具集及其文档至关重要。我们在附录 2（“为您的工具进行提示工程”）中扩展了工具开发的最佳实践。

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F58d9f10c985c4eb5d53798dea315f7bb5ab6249e-2401x1000.png&w=3840&q=75)

**什么时候使用 Agents**：agents 可以用于开放性的问题，这些问题难以或不可能预测所需的步骤数量，并且你不能硬编码固定路径。大型语言模型（LLM）可能会操作多个回合，你必须对其决策能力有一定程度的信任。代理的自主性使它们在可信环境中进行任务扩展的理想选择。

代理的自主性意味着更高的成本和潜在的错误累积。我们建议在沙箱环境中进行广泛测试，并设置适当的保护措施。

#### agents 有用的例子：

以下的例子来自我们自己的实现：

-   一个编码代理，用于解决 [SWE-bench](https://www.anthropic.com/research/swe-bench-sonnet) 任务，这些任务涉及根据任务描述对多个文件进行编辑；  
-   我们的“[计算机使用](https://github.com/anthropics/anthropic-quickstarts/tree/main/computer-use-demo)”参考实现，其中Claude使用计算机来完成任务。

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F4b9a1f4eb63d5962a6e1746ac26bbc857cf3474f-2400x1666.png&w=3840&q=75)

## 结合和定制这些模式  

这些构建模块并不是一成不变的。它们是开发者可以根据不同用例塑造和组合的常见模式。成功的关键与任何大语言模型的特性一样，是衡量性能并不断迭代实现。重申一遍：只有在复杂性确实能改善结果时，您才应考虑增加复杂性。

## 总结

在大型语言模型（LLM）领域取得成功并不是构建最复杂的系统，而是构建适合您需求的正确系统。首先从简单的提示开始，通过全面评估进行优化，只有在更简单的解决方案不奏效时，才添加多步骤的自主系统。

在实施智能体时，我们尽量遵循三个核心原则：

1.   保持智能体设计的简单性。
2.   优先考虑透明性，明确展示智能体的规划步骤。
3.   通过全面的工具文档和测试，精心打造智能体与计算机的接口（ACI）。

框架可以帮助您快速入门，但在进入生产阶段时，请毫不犹豫地减少抽象层次，使用基本组件进行构建。通过遵循这些原则，您可以创建不仅强大而且可靠、可维护，且得到了用户信任的智能体。

## 附录1：实践中的智能体  

我们与客户的合作揭示了两个特别有前景的人工智能智能体应用，展示了上述模式的实际价值。这两个应用例子说明了智能体如何在需要对话和行动、具有明确成功标准、能够启用反馈循环并整合有效的人类监督的任务中创造最大的价值。

### A. 客户支持  

客户支持将熟悉的聊天机器人界面与通过工具集成增强的能力相结合。这对于更开放式的代理来说非常合适，因为：

-   支持交互自然遵循对话流程，同时需要访问外部信息和操作；  
-   可以集成工具以提取客户数据、订单历史和知识库文章；  
-   诸如发放退款或更新工单等操作可以通过编程来处理；  
-   成功可以通过用户定义的解决方案清晰地衡量。  

几家公司通过基于用量的定价模型展示了这种方法的可行性，该模型仅对成功的解决方案收费，显示了对其代理有效性的信心。

### B. 编码代理  

软件开发领域展示了大型语言模型（LLM）特性的显著潜力，其能力从代码完成演变为自主问题解决。代理特别有效，因为：

-   代码解决方案可以通过自动化测试进行验证；  
-   代理可以使用测试结果作为反馈对解决方案进行迭代；  
-   问题空间定义明确且结构化；  
-   输出质量可以客观测量。  

在我们自己的实现中，代理现在能够仅基于拉取请求描述解决真实的 GitHub 问题，符合 SWE-bench Verified 基准。然而，尽管自动化测试有助于验证功能，人类审核仍然对确保解决方案与更广泛的系统要求一致至关重要。

## 附件 2：工具的提示工程

无论您正在构建哪种代理系统，[工具](https://www.anthropic.com/news/tool-use-ga)很可能是您代理的重要组成部分。工具使 Claude 能够通过在我们的 API 中指定其确切结构和定义来与外部服务和API进行交互。当 Claude 作出回应时，如果它计划调用一个工具，它将在 API 响应中包含一个[工具使用块](https://docs.anthropic.com/en/docs/build-with-claude/tool-use#example-api-response-with-a-tool-use-content-block)。工具的定义和规范应该给予与整体提示同样多的提示工程关注。在这个简短的附录中，我们将描述如何进行工具的提示工程。

通常有几种方法可以指定相同的操作。例如，您可以通过写一个 diff 来指定文件编辑，或者重写整个文件。对于结构化输出，您可以在 markdown 或 JSON 中返回代码。在软件工程中，这些差异是表面的，可以无损地相互转换。然而，一些格式对于 LLM 来说比其他格式要更难写。写  diff 需要知道在新代码编写之前，块头中有多少行正在更改。写 JSON 中的代码（与 markdown 相比）需要额外转义换行符和引号。

我们对决定工具格式的建议如下：

-   给模型足够的令牌，以便模型在写入无法自拔之前进行“思考”。
-   保持格式尽可能接近模型在互联网上自然出现的文本。
-   确保没有格式“开销”，例如，不必准确计算成千上万行代码的数量，或对其编写的任何代码进行字符串转义。

一个经验法则是考虑人机交互（HCI）所投入的努力，然后计划在创建良好的代理计算机接口（ACI）上投入同样多的努力。以下是一些实现这一目标的想法：

-   将自己置于模型的角度。根据描述和参数，使用这个工具是否显而易见，还是需要仔细思考？如果是这样，那么对模型来说也可能是这样的。好的工具定义通常包括示例用法、边缘情况、输入格式要求以及与其他工具的明确界限。
-   您如何更改参数名称或描述以使其更明显？将这视为为您团队中的初级开发人员编写出色的文档字符串。这在使用许多类似工具时尤为重要。
-   测试模型如何使用您的工具：在我们的[工作台](https://console.anthropic.com/workbench)上运行许多示例输入，以查看模型犯了什么错误，并进行迭代。
-   为工具添加[防错](https://en.wikipedia.org/wiki/Poka-yoke)机制。更改参数，使犯错误变得更加困难。

在为 [SWE-bench](https://www.anthropic.com/research/swe-bench-sonnet) 构建我们的代理时，我们实际上花了更多时间来优化我们的工具，而不是整体提示。例如，我们发现模型在代理移出根目录后，使用相对文件路径的工具会出错。为了解决这个问题，我们将工具更改为始终需要绝对文件路径——我们发现模型使用这种方法时表现得非常出色。