---
title: Reader-LM Small Language Models for Cleaning and Converting HTML to Markdown
published: 2024-10-21 20:00:00
description: 'Jina reader-lm 诞生日记'
image: ''
tags: ['Reference', 'LLM', 'AI']
category: 'Reference'
draft: false

---

>   Link: [Reader-LM: Small Language Models for Cleaning and Converting HTML to Markdown](https://jina.ai/news/reader-lm-small-language-models-for-cleaning-and-converting-html-to-markdown/)

## Jina Reader API  v1

1.   使用无头 Chrome 浏览器获取页面的源代码；
2.   利用 [Mozilla readability](https://github.com/mozilla/readability) 去除标题、页脚、导航栏等样式后，获取页面主要内容；
3.   使用 [regex](https://x.com/JinaAI_/status/1823756993108304135)和 [Turndown library](https://github.com/mixmark-io/turndown) 将上一步的 HTML 转换为 Markdown。

V1 版本的问题有：

-   readability 会错误删除某些内容；
-   Turndown 无法将 HTML 中的所有部分转换为 Markdown;
-   使用正则表达式或者其他方式并不能解决所有的问题，而且难以维护并支持多语言。

于是考虑过渡到采用 `E2E LM` 来解决这个问题，具体的可以参考下图：

:::note

E2E 的全程是 End to End，简单理解就是模型输入为原始内容，输出直接为最终需要的结果。

:::

![image-20241021205702365](https://typora-photos.oss-cn-shenzhen.aliyuncs.com/202410212057589.png)

## Jina Reader API  v2

声明一点，上面说的是 `LM` 而非 `LLM`，因为 `LLM` 必定会带来更高的成本和更慢的运行速度。

所以我们考虑采用一个小语言模型（SLM）：参数量小于 1B，且能在边缘计算设备（*原文并没有指出这个的范围，不过笔记本也可以视为非典型边缘设备的一种*）上高效运行的模型。

但根据 Scaling law，参数越少通常会导致推理与总结能力的下降。因此如果 SLM 的参数量过少，它可能难以生成任何有意义的内容。

我们回顾下 HTML2Markdown 这个任务本身：

1.   我们考虑的任务不具备典型 LLM 任务的创造性与复杂性，模型需要解决的有两个问题：

     -   正确从 HTML 中选择需要的内容到输出，即跳过 HTML 标记；
     -   将 HTML 中的样式转为 Mardown 语法；

     因此这个任务从直觉上会比生成文本简单。

2.   对长上下文的支持，现代 HTML 中会包含大量的噪音标记，如果 SLM 需要具备良好的转换能力，上下文必须足够大，因此 8K/16K 长度的上下文可以说没有用；

所以最终的需求变成：`浅而广`的 SLM，浅指任务只需要简单的复制粘贴，推理的 transformer 块可以减少；广指具备长上下文支持，需要注意力机制。综合过往研究表明：`上下文长度与推理能力密切相关`，对于 SLM 来说优化这两个维度并同时保持参数规模小是极具挑战性的。

Jina 提出的 reader-lm-0.5b/1.5b 两款 SLM 在 HTML2Markdown 的任务上的表现显著优于现有的通用大模型：

![image-20241022151052289](https://typora-photos.oss-cn-shenzhen.aliyuncs.com/202410221510419.png)

>   近期在测试通用 LLM 清洗邮件 HTML 的体验与上图基本一致，next token 带来的输出不确定性是最大的问题。

从披露的两款模型规格看，参数量做到了足够小，上下文长度均为 256K，隐藏层大小、层数等也有减少。

### 测试基准

评估指标：

-   ROUGE-L：常用于摘要和问答任务，衡量预测输出与参考文本之间的相似度，L 标记是指文本中连续的 n 个词序列之间的重叠程度，越高越好；
-   Token Error Rate（TER）：计算生成的 Markdown 标记未出现在原始 HMTL 内容中的比例，用于评估模型的幻觉率，越低越好；
-   Word Error Rate（WER）：常用于 OCR 和 ASR 任务，WER 考虑词序列并计算插入（ADD）、替换（SUB）、删除（DEL）等错误，用于评估预测标记与预期输出之间的不匹配程度，越低越好；

LLMs 所使用的 prompt 均为：

```
Your task is to convert the content of the provided HTML file into the corresponding markdown file. You need to convert the structure, elements, and attributes of the HTML into equivalent representations in markdown format, ensuring that no important information is lost. The output should strictly be in markdown format, without any additional explanations.
```

:::note

benchmark 结果参考性不大，基本就是验证的现行 AI 的一些问题和客观规律：

1.   相同架构的模型，参数量越大的确会带来更好的综合性能；
2.   通用模型的幻觉还是比较严重的；

基准测试中没有提到 Anthropic 系列的模型比较奇怪，猜测可能没有完全被 reader 比下去😜。另外一个点是过拟合，毕竟就这个简单转换的应用场景来说，过拟合反而是会降低模型的 WER 和 TER，可以理解为限制了想象力🤷‍♂️。

:::

### Reader-LM 使用

-   Jina 在 Google Colab 上提供了体验 reader-lm 的笔记本[Google Colab](https://colab.research.google.com/drive/1wXWyj5hOxEHY6WeHbOwEzYAC0WB1I5uA);
-   生产环境中建议使用 RTX 3090/4090 并开启 bfloat16 和 flash-attention 以减少 VRAM 使用，和在长输入场景下的性能下降；

## Reader-LM 训练

### 数据

输入数据由原始 HTML 和 Markdown 组成问答对。比较重要的点在于 SLM 对训练数据的质量特别敏感，所以 Jina 团队建立了一个数据管道，确保训练集中包含的是高质量 Markdown 条目。

:::note

特别敏感这个点其实可以认为是小参数量带来的副产物，小参数量由于学习能力有限，所以很容易过拟合，一旦训练集中出现错误的数据就会很容易被 SLM 学习。

:::

另外 Jina 团队使用 GPT-4o 合成了一部分 HTML 和 Markdown 对应物的合成数据，与真实 HTML 相比，合成数据具备更简单的结构与噪声，是更容易让 SLM 学习的。

最终的训练数据格式如下，完整训练数据总量为 25 亿 Tokens：

```
<|im_start|>system
You are a helpful assistant.<|im_end|>
<|im_start|>user
{{RAW_HTML}}<|im_end|>
<|im_start|>assistant
{{MARKDOWN}}<|im_end|>
```

:::note

我猜测这部分数据的比重不少于 20%，不多于 30%。

**为什么没有更多？**

因为真实 HTML 中一定是具备更多噪音的，如果全部使用合成数据，会有两个问题：

1.   属于直接蒸馏，涉及商业版权等问题；
2.   更重要的原因是可能会导致 SLM 的效果进一步下降，因为训练数据简单，模型学习到的内容有限，更加容易过拟合。

**那为什么一定要有合成数据呢**？

使用部分合成数据可以看作是一种软性的迁移学习，从"理想"例子开始，然后逐步适应更复杂的真实场景。

:::

### 两阶段训练

Jina 使用以下两个阶段来训练 reader-lm：

1.   简短简单的 HTML：最大序列长度（HTML+Markdown）设置为 32K tokens，这部分数据量为 15 亿  tokens；
2.   长且难的 HTML：最大序列长度拓展到 128K tokens，数据量为 12 亿 tokens，并实现 [ring flash attention](https://github.com/zhuzilin/ring-flash-attention)；

:::note

因此，虽然说 Jina 宣称 Reader-lm 支持 256K 的上下文，但是训练数据中最多只包含 128K。不过由于底座模型 Qwen2 使用的是 RoPE 位置编码方法，所以相比原版 transformer 中的正余弦位置编码方式具备了更强的长序列处理能力。

:::

训练过程中出现了以下问题：

#### 重复/循环输出 token

训练过程中发现模型在生成一些 tokens 后，会重复生成相同的 token 或者陷入循环，不断重复一小段 tokens，直到最大生成数量的限制。

解决办法综合了以下两份资料：

1.   使用 [contrastive search](https://github.com/yxuansu/SimCTG) 作为解码方法，并在训练过程中加入对比损失，实践中发现有效减少了重复生成；
2.   在 transformer pipeline 中实现了一个检测重复 token 并停止解码的机制，参考 [issue](https://github.com/huggingface/transformers/issues/32902)。

#### 长输入的训练效率

由于 transformer 中在处理长输入是会出现 OOM 错误的风险，所以 Jina 团队采用了 chunk-wise model forwarding 减少了显存的使用。这部分没有找到具体的论文说明，后续再找找看。

另外就是 Transformer Trainer 训练框架中，如果输入数据过长会被分割，导致子文本丢失上下文信息，或者是为了让输入序列的长度一致（方便批处理），填充无意义的内容，这两种情况的出现可能会导致模型依赖自身推理能力补齐上下文信息，从而导致模型产生幻觉。

于是 Jina 团队采取的策略是改进 Transformer Trainer 将多个短文本连接成长序列实现无填充训练。

最终的 0.5B 模型是能够在长上下文输入下实现选择性复制的最小模型，而 1.5B 模型是最小的更大模型，显著提高了性能，而不会在参数大小方面遭遇收益递减。

## 替代架构：仅编码器模型

这部分没有太多的参考意义，跳过。

## 总结

通篇看下来，最有意义的点在于 SLM 上进行专有任务的训练尝试，其次是长上下文注意力机制的尝试和训练中问题的处理。后续会去看看 ring flash attention 的仓库与 重复输出 token 的解决方案，先立 flag 吧，hhh
