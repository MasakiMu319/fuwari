---
title: MIT 6.824 - MapReduce
published: 2024-08-16 12:00:00
description: 'MIT 6.824 lab 1: MapReduce.'
image: ''
tags: ['MIT 6.824', 'Golang']
category: 'Distribute system'
draft: true 
---

MapReduce

## Base

`mrsequential.go`是基础的串行版本 `MapReduce`，核心的功能：

-   串行读取所有的文件，使用 `map` 分割文本中不同的单词，每个单词出现次数