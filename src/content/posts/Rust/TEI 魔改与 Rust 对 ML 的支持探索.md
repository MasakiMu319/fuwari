---
title: TEI 魔改与 Rust 对 ML 的支持探索
published: 2024-10-18 10:00:00
description: '对 TEI 的魔改与 Rust 在 ML 相关的支持探索'
image: ''
tags: ['Rust', 'ONNX', 'Embedding']
category: 'Rust'
draft: true
---

*（动笔之前发现距离自己第一次写有关 Rust 的 Blog 正好两个月了，这也是自己总共学习 Rust 的时间，感慨良多）*。

## 前言

### TEI 是什么？

TEI(Text Embeddings Inference) 简单理解就是一个极速的文本向量化模型推理解决方案。详细的说明参考官方 GitHub repo：

::github{repo="huggingface/text-embeddings-inference"}

### 为什么在工程中采用 TEI 作为 Embedding 服务？

核心两点，就是速度+性价比。国内/海外均有大模型公司已经提供成熟的 Embedding API 服务，不采用的原因主要出于以下考虑：

1.    API 调用产生的延迟 T95 通常在 700ms（海外服务 1s～2s），也就意味着云端每次处理请求调用链的总耗时会被强制延长 0.7，这对用户期望得到及时响应的在线场景下几乎是不可接受的；
2.   当前 Text Embedding Model 之间的差异不大。从我们实际业务场景下使用合成数据进行测试的结果看，以`text-embedding-3 ` 模型为代表的商用 API 实际表现相比开源模型甚至有一定差距，实际选取可以参考 [MTEB 榜单](https://huggingface.co/spaces/mteb/leaderboard)。

     >   :::warning
     >
     >   MTEB 是一个用于评估不同文本嵌入模型表现的基准，需要注意的是尽可能不要参考 Average 分数，因为 MTEB 基准会考量模型在不同下游任务上的表现，也就意味着**模型大概率会出现偏科**的情况，比如说你选择了一个综合排名第一的模型，但实际上它在你需要使用的场景下表现非常差，就会给你带来错误的判断。
     >
     >   :::
     >
     >   :::note
     >
     >   补充相关指标的说明：
     >
     >   1.   **CLS (Classification)**：这是分类任务的表现。CLS 指标衡量模型将文本嵌入用作分类问题中的特征时的表现。例如，将文本分类为不同类别（情感分析、主题分类等）的准确率或其他相关指标。
     >   2.   **Clustering**：这是聚类任务的表现。模型生成的嵌入用于聚类任务时的效果。嵌入的质量影响不同文本在向量空间中的聚类效果，通常使用聚类指标（如 Adjusted Rand Index, Silhouette Score）来评估。
     >   3.   **Pair_CLS (Pair Classification)**：这是成对分类任务的表现。模型需要判断一对文本是否属于某种特定关系（例如是否具有相同的情感、主题等）。这类任务衡量模型处理成对数据时的分类能力。
     >   4.   **Reranking**：指模型在重排序任务中的表现。模型生成的文本嵌入用于对候选项进行重新排序。例如，给定一个查询，模型需要根据候选答案的相关性进行排序。Reranking 任务一般用于信息检索和推荐系统。
     >   5.   **Retrieval**：指检索任务中的表现。模型生成的文本嵌入用于搜索或检索任务，例如根据查询找到相关文档。这是衡量模型在搜索引擎或类似任务中的表现的重要指标，常用的评估方法包括精确率、召回率等。
     >   6.   **STS (Semantic Textual Similarity)**：语义文本相似度任务，用于衡量模型判断两个文本在语义上的相似程度。通常，模型生成两个文本的嵌入，通过计算它们的距离（例如余弦相似度）来判断相似度。评分基于与人工标注的相似度分数的匹配程度。
     >
     >   :::

在确定使用自部署 Text Embedding Model 的情况下，选取 TEI 具备以下优点：

-   **支持直接使用 Hugging Face 上的 Embedding Model 部署推理服务**：意味着底座模型可以跟随 MTEB 榜单或者内部 benchmark 结果进行快速切换，利于后续迭代升级；
-   **天然提供 gRPC 接口**：模型通常会部署为一个单独的微服务为项目中的其他部分提供基础能力，意味着 TEI 近乎开箱即用；
-   **GPU/CPU 推理支持**：在底座模型为 ONNX 格式时，支持使用 CPU 推理（CPU 推理的速度并不绝对比 GPU 慢）；
-   **超快的推理速度**：Embedding 请求的处理延时 T99 为 80ms(*实际请以工程中采用的底座模型的推理速度为准)；

### 为什么选择魔改？

当前业界主流使用 Embedding 的思路是利用 Text Embedding Model 产生的结果属于文本在语义空间中的映射，通过相似度匹配等算法来达到近似语义理解的效果，这个结果被称为向量，更精准的描述是 Dense embedding。

对于大部分场景来说，Dense embedding 已经足够了，但也因为 Dense 会出现关键内容在最终的 embedding 中特征不够明显的问题，所以就有了 Sparse embedding 的需求，Sparse 思想的应用有很多，如 TF-IDF、BM25 等(具体可以参考之前的 Blog：[Dive into Embedding](https://myblog-jiumumu9s-projects.vercel.app/posts/ai/dive-into-embedding/))，核心思想就是抓主要矛盾，找出句子中的关键词，用这几个关键词来代表完整的句义。

显而易见地一点是，Dense 和 Sparse 都有自己的问题，并不是解决问题的 silver bullet。不过我们可以直接得到的启发就是将两者进行结合。由此，TEI 仅输出 Dense 的接口就无法满足我们的需求。

## 实现路径

### Hybrid Score

Hybrid Score 是增强 Dense embedding 应用的思路，由以下两部分内容组成：

1.   Dense score 得到的相似度分数；
2.   Sparse score 得到的稀疏权重分数；

我们以实际代码和示例作为参考，代码部分会省略掉无关紧要的部分。(*这部分需要对模型组成架构有一定了解，如果不太明白可以直接跳过*)：

#### 推理部分

```python
def _encode(
        self,
        texts: Dict[str, torch.Tensor] = None,
        ...
        return_dense: bool = True,
        return_sparse: bool = False,
    ):
  			# 对文本内容进行分词，返回 pytorch 格式的张量
        text_input = self.tokenizer(
            texts,
            padding=True,
            truncation=True,
            return_tensors="pt",
            max_length=max_length,
        )
      	# tokenizer 默认返回的结果是在 CPU 上，这一步是为了确保分词结果张量与模型加载的位置处于同一设备
        text_input = {k: v.to(self.model.device) for k, v in text_input.items()}
        # 获取 Text Embedding Model 推理结果
        model_out = self.model(**text_input, return_dict=True)

        output = {}
        if return_dense:
            dense_vecs = model_out.last_hidden_state[:, 0, :dimension]
            ...
            output["dense_embeddings"] = dense_vecs
        if return_sparse:
          	# relu 函数快速过滤掉 logits 中小于 0 的值；
            token_weights = torch.relu(model_out.logits).squeeze(-1)
            token_weights = list(
                map(
                    self._process_token_weights,
                    token_weights.detach().cpu().numpy().tolist(),
                    text_input["input_ids"].cpu().numpy().tolist(),
                )
            )
            # 获取 token weights
            output["token_weights"] = token_weights
        return output
```

#### Sparse score 计算

Dense embedding 的处理非常简单，不做另外的说明，我们主要看下如何计算 Sparse scores： 

```python
def _compute_sparse_scores(self, embs1, embs2):
        scores = 0
    		# 重复出现的关键词就能贡献到最终的 Sparse 分数，出现次数越多通常也就意味着这两段文本中关键的共同点
        for token, weight in embs1.items():
            if token in embs2:
                scores += weight * embs2[token]
        return scores
```

处理 token weights：

```python
def _process_token_weights(self, token_weights: np.ndarray, input_ids: list):
        result = defaultdict(int)
        unused_tokens = set(...)
    		# 输入的 token weights 可以认为就是 logits 层的结果；
        # logits 中对的结果表示的是每个 token 在整个句子中的重要程度；
        # 这里我们过滤掉无意义和出现概率小于 0 的 token，
        # 保留的就是对整体句子有主要贡献度的 token
        for w, idx in zip(token_weights, input_ids):
            if idx not in unused_tokens and w > 0:
                token = self.tokenizer.decode([int(idx)])
                if w > result[token]:
                    result[token] = w

        return result
```

#### Hybrid Score 计算

```python
# 即按不同权重计算 dense score + sparse score
scores = (
            self.compute_dense_scores(
                embs1["dense_embeddings"], embs2["dense_embeddings"]
            )
            * dense_weight
            + self.compute_sparse_scores(embs1["token_weights"], embs2["token_weights"])
            * sparse_weight
        )
```

:::note

一句话理解就是我们需要在计算相似度分数的同时，增加计算两句话中关键词（相同 token）的贡献度分数。

:::

## 魔改工作

到目前为止，计算 Sparse score的流程已经完全分析清楚了，那么我们需要考虑如何才能对 TEI 进行魔改了，此外出于成本考量，部署 TEI 所使用的基座模型需要为支持 CPU上推理加速的 ONNX 。

综上，在实际魔改前，我们需要确认以下几个问题：

1.   基座模型的输出结构是否支持获取 logits；
2.   转换为 ONNX 格式后的基座模型是否能保留原始模型的输出结构；
3.   `_process_token_weights` 方法是否能移植到 Rust；
4.   Rust 对 ONNX 格式模型的支持情况；

### ONNX 格式

ONNX 模型是本次需要解决的核心问题之一，由于 TEI 对 CPU 上部署推理要求模型提供 ONNX 格式，所以我们后续所有的计算参数都来自 ONNX 格式模型的输出。ONNX 简单理解就是一个模型标准，所有的 ONNX 格式模型均可以通过 ONNX runtime 进行加速推理（*通常在 CPU 上可以提速 5～6 倍，GPU 上的提升在 1.3～1.7 之间*）。

这里我们直接给出以下结论：

1.   ONNX 模型由基座模型转换为 ONNX 标准格式；
2.   ONNX 模型的输出可以与转换前的基座模型一致，也就是说基座模型能够输出 logits，那么 ONNX 模型也具备相同能力；
3.   Rust 社区提供 [ort](https://ort.pyke.io/) 来支持 ONNX 模型运行；

:::note

ONNX 格式可以类比成容器化标准 OCI，对于满足 OCI 的镜像都可以通过 Docker 或者其他容器管理工具创建并管理容器。

:::

这里需要注意的点在于，默认的 ONNX 转换是可能不保留 logits 层作为模型输出的一部分（因为大部分模型/使用场景下并不会使用到 logits 的结果），需要在导出过程中手动指定，具体可以参考下面的部分：

```python
torch.onnx.export(
    ...
    input_names=['input_ids', 'attention_mask'], 
    output_names=['logits', 'last_hidden_state'], 
		...
)
```

从 ONNX 模型输出结果中获取 logits：

```
// python
onnx_outputs[0]

// rust
onnx_outputs.get("logits")
```

在能顺利获取到 logits 后，处理 Sparse score 的方式就可以完全按照上面的 Python 代码进行。

### 魔改

编码的工作相对是轻松很多的，由于 Rust 本身编译器的特性，基本可以在编译通过后宣告修改成功。过程可以参考上面的 Python 代码，不再赘述。