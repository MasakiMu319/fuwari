---
title: Dive into RAG
published: 2024-09-20 10:00:00
description: '深入探索 RAG 工程实践。'
image: ''
tags: ['LLM', 'AI', 'RAG']
category: 'AI'
draft: true 
---

>   阅读本文需要具备一定的前置知识。

RAG 是自 LLM 出现以来最有竞争力的落地应用方案。

### BM25

BM25 是基于 TF-IDF 做的，TF-IDF 就是简单的基于单词在文本中出现的次数的倒数作为其重要程度。BM25 在此基础上考虑了文本长度，并在词频上应用了一个饱和函数，以避免常见的单词影响结果。

## Reference

1.   https://www.anthropic.com/news/contextual-retrieval