---
title: Reference - Reader-LM Small Language Models for Cleaning and Converting HTML to Markdown
published: 2024-10-21 20:00:00
description: 'Jina reader-lm 诞生日记'
image: ''
tags: ['LLM', 'AI']
category: 'AI'
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

声明一点，上面说的是 LM 而非 LLM，因为 LLM 必定会带来更高的成本和更慢的运行速度。

所以我们考虑采用一个小语言模型（SLM）：参数量小于 1B，且能在边缘计算设备（*原文并没有指出这个的范围，不过笔记本也可以视为非典型边缘设备的一种*）上高效运行的模型。

但根据 Scaling law，参数越少通常会导致推理与总结能力的下降。因此如果 SLM 的参数量过少，它可能难以生成任何有意义的内容。

我们回顾下 HTML2Markdown 这个任务本身：

1.   我们考虑的任务不具备典型 LLM 任务的创造性与复杂性，模型需要解决的有两个问题：

     1.   正确从 HTML 中选择需要的内容到输出，即跳过 HTML 标记；
     2.   将 HTML 中的样式转为 Mardown 语法；

     因此这个任务从直觉上会比生成文本简单。

2.   对长上下文的支持，现代 HTML 中会包含大量的噪音标记，如果 SLM 需要具备良好的转换能力，上下文必须足够大，因此 8K/16K 长度的上下文可以说没有用；



