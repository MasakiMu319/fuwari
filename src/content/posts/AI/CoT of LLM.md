---
title: Is careful thinking the most effective method for enhancing LLMs?
published: 2024-11-10 11:30:00
description: 'Some interesting research findings about CoT.'
image: ''
tags: ['CoT', 'LLM']
category: 'LLM'
draft: false

---

CoT 是大模型提示工程开始兴起后不久即提出的概念，核心魔法🧙‍♀️浓缩成一句话就是“Let’s think step by step”.对其合理的解释是，通过让 LLMs 更多地输出与任务相关的内容以提升“next token”预测结果的准确性。近期 o1-preview 所提出的 slow think 也被视为是 CoT 的工程化实现代表，于是开始有人声称 CoT 展示了 LLMs 的无限推理能力，极有可能使得我们进一步实现 AGI。

但事实真的如此吗？

就笔者过去在各种下游 NLP 任务上尝试 LLMs 的经验来看，在某些应用场景下使用 CoT 对 LLMs 的输出确有提升，但提升有限，更多是无明显效果。

本文即解读最近有关 CoT 对 LLMs 负面影响的研究成果，原文参考： [Mind Your Step (by Step): Chain-of-Thought can Reduce Performance on Tasks where Thinking Makes Humans Worse](https://arxiv.org/abs/2410.21333)，

## 引入

虽然模型的认知过程与人类的认知过程并不完全相同，但可以==参照思考对人类“性能”产生负面影响的情况，假定思考会对模型产生负面影响==。

论文作者从心理学中选择了 6 项已被充分研究的任务类型来探讨 CoT 对 LLM 性能的影响，并验证了 CoT 并非在所有任务中都能提高模型性能，在隐性统计学习、面部识别、含例外模式的数据分类三种情况下，各种 SOTA 模型的性能都会明显下降，此外，研究本身进一步揭示了==通过人类心理学研究大模型的可行性==。

## 研究方法

基于两个关键条件：

1.   言语思考或深思熟虑会损害人类“性能”的情况；
2.   将制约人类“性能”的因素==推广==到 LM 的情况；

预设计的 6 种任务场景：

-   **隐形统计学习（Implicit Statistical Learning）**：考察模型在隐含语法结构的分类任务中使用 CoT 是否会降低表现，基于心理学中的实验结果，该研究假设人类在进行语言推理时表现较差，因此 CoT 在该场景下应该会有类似的效果；

    隐含语法结构的分类在论文中指：使用人为设置的规则批量生成字符串，并让 LLMs 判断给定的字符串是否符合已生成的字符串中隐含的规律，即二分类。

-   **面部识别（Facial Recognition）**：该任务中模型需要识别图像中的人脸。基于人类在口头描述面部特征后识别率下降的现象，研究假设 CoT 会影响模型的面部识别准确性；

-   **含例外模式的数据分类（Classifying Data with Patterns that contain Exceptions）**：该任务模拟模型在含有异常标签的数据中学习的表现，研究假设 CoT 会导致模型在遇到例外情况时增加学习轮次，因为人类通常会倾向于建立简单规则，从而忽视个别特例；

-   **解释逻辑不一致（Explaining a logical inconsistency ）**：在逻辑一致性判断任务中，模型需要识别出两句话之间的逻辑冲突，该任务通常会引发人类的语言推理困难；

-   **空间直觉（Spatial Intuitions）**：模型需要推断液体在倾斜容器中的位置，该任务依赖空间和运动直觉，心理学研究表明人类在使用语言推理时效果不佳，该研究假设模型也会遇到类似问题；

-   **特征聚合决策（Aggregating Features for a Decision）**：模型在多维度决策情景中聚合信息并做出决策，由于信息过载通常会导致人类在 CoT 模式下表现不佳，因此研究假设在该任务中，CoT 不会提高模型性能。

针对每个任务场景，构建了 zero-shot 和 CoT 提示条件，并在多个主流 LLMs 和 LMM 上进行测试（包括 GPT-4o、Claude 3.5、Llama 等）。

## 实验结果

### 隐性统计学习

该研究考察了模型在分类基于特定语法结构的序列时的表现，任务包含 4400 个分类问题，基于 100 种有限状态语法（FSG）结构，每个测试提供 15 个样例，要求模型对新序列进行分类。

实验结果显示，**使用 CoT 提示的模型表现显著下降，尤其是 OpenAI o1-preview 模型的准确率下降了 36.3%。**这表明当模型过度依赖逐步推理时，CoT 可能会抑制其对隐性统计模式的学习能力。

![image-20241111142010280](https://raw.githubusercontent.com/MasakiMu319/fuwari/main/src/assets/post-images/202411111420429.png)

### 面部识别

该研究测试了 CoT 是否会影响模型的面部识别能力，这是基于心理学中“语词遮蔽”现象进行的任务情境设计。模型需要在 500 项任务中从 5 个候选中匹配初始人脸。

语词遮蔽现象即人在阅读感知句子的时候由于某些认知或者注意力限制，无法完整识别或者处理某些词语。大脑会自动补全或者略过被遮蔽的词。

**结果表明，当被要求执行 CoT 时，每个被测试的 LMM 都显示出性能下降，与假设一致。**

![image-20241111142709389](https://raw.githubusercontent.com/MasakiMu319/fuwari/main/src/assets/post-images/202411111427470.png)

### 含例外模式的数据分类

该任务通过包含多个主次特征的分类任务来测试模型在处理含例外情况时的表现，任务要求模型在多次分类中逐步学习，目标是尽可能减少迭代次数。

实验在 GPT-4o、Claude 3.5 Sonnet 和 Claude 3 Opus 上进行，结果表明，CoT 显著增加了学习轮次。平均来看，GPT-4o 在 CoT 条件下完成正确分类所需的轮次为直接提示的四倍，而 Claude 3.5 Sonnet 和 Claude 3 Opus 的轮次需求也分别增加至直接提示的两倍多。

![image-20241111143124425](https://raw.githubusercontent.com/MasakiMu319/fuwari/main/src/assets/post-images/202411111431490.png)

在 GPT-4o 的进一步分析中发现，直接提示使模型在第二或第三轮就能达到完美分类，而使用 CoT 时模型在第四到第五轮仅能正确分类 8/10 的对象。**这表明 CoT 提示会引导模型偏向基于规则的推理方式，而忽视了已知的正确答案，导致分类效率大幅下降。**

### 解释逻辑不一致

研究发现，CoT 增加了模型忽视矛盾的可能性，模型在逐步推理时更倾向于关注复杂的逻辑结构，从而忽视了直接矛盾判定。这表明在需要精确逻辑验证的任务中，CoT 提示存在局限性。

![image-20241111143323886](https://raw.githubusercontent.com/MasakiMu319/fuwari/main/src/assets/post-images/202411111433956.png)

### 空间直觉

在该情境中，模型需要通过“倾斜杯子”的问题来推断水面的位置。这类任务依赖于人类的空间或运动直觉，而人类通常在非言语思维下表现更好。

模型接收了视觉提示和多项选择答案，实验结果显示，使用 CoT 提示对模型表现无明显影响。这说明在依赖空间或运动直觉的任务中，模型的推理方式与人类的直觉差异较大，因而 CoT 提示的负面影响较小。

![image-20241111143421004](https://raw.githubusercontent.com/MasakiMu319/fuwari/main/src/assets/post-images/202411111434068.png)

### 特征聚合决策

此任务模拟了基于多项特征的决策过程（如选房），用于测试信息超载对决策的影响。人类在类似任务中由于记忆限制，往往在 CoT 模式下表现较差。相对地，模型保留了所有上下文信息，能够无损地聚合和评估每项特征。

结果显示，==CoT 提示在高上下文记忆任务中提高了模型表现，说明在信息保留至关重要的场景下，CoT 提示能够发挥正向作用。==

![image-20241111143522845](https://raw.githubusercontent.com/MasakiMu319/fuwari/main/src/assets/post-images/202411111435913.png)

## 结语

笔者从测试数据中观察到比较奇怪的有两点，在特征聚合决策测试中：

1.    OpenAI 和 Llama 的模型存在性能下降，但是 Claude 的表现比较一致，有明显提升效果；
2.    就 Anthropic 披露的模型数据来说， Opus 规格应明显优于 Sonnet，但是在该任务中 Sonnet 却明显优于 Opus；

此外，推理方面的任务，我个人猜测 CoT 提示词设计不合理应当会严重影响最终呈现的效果，需要谨慎看待 CoT 效果下降的效果参考。