+++
title = "RAG Semantic Gap"
date = "2026-06-05T21:32:42+08:00"

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["RAG","Semantic Gap",]
+++

本文讨论 RAG 如何解决语义鸿沟（Semantic Gap）的问题，围绕 RAG 整个流程逐个展开，从用户查询、向量化、检索、知识库各阶段。

## 查询层

用户的原始提问不适合直接拿去做向量检索，可以先把它变成更适合检索的形式。

### 查询改写（Query Rewriting）

### 假设文档嵌入（Hypothetical Document Embeddings, HyDE）

### 多查询扩展（Multi-Query）

```python
class MultiQueryRetriever(BaseRetriever):

```

## 知识库层

### 文档切分策略

### 层次化索引（Hierarchical Indexing）

### 文档增强（Document Enrichment）

HyDE 的反向操作

## 检索层

有了适合检索的用户查询，需要通过检索的方式去知识库检索。

### 向量检索

### 关键词检索

BM25

### 混合检索

## 重排

### RRF（Reciprocal Rank Fusion）

### 重排序模型（Reranker）

## Embedding 模型

### 领域微调（Fine-tuning）

## 组合拳

