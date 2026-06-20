+++
title = "150 行 Python 代码构建全文搜索引擎"
date = "2022-04-24T21:43:52+08:00"
tags = ["Python","Search Engine"]
description = "从零实现一个能跑在 600 万条维基百科摘要上的全文搜索引擎，理解倒排索引和 TF-IDF 的核心思想。"
+++

说实话，搜索引擎这玩意儿听起来挺唬人的。Elasticsearch、Lucene、Solr 每个都是几十万行代码的庞然大物。但如果你把搜索引擎拆到最核心的骨架，其实只需要两个东西：**倒排索引**和**相关性排序**。剩下的分布式、容错、增量更新，都是工程上的加法。

Bart de Goede 写过一个很经典的教程，用不到 150 行 Python 实现了一个能检索 600 万条维基百科摘要的搜索引擎。我读完觉得这是一个绝佳的"理解搜索引擎原理"的切入点，代码量少到你可以一口气读完，但每个关键概念都落地成了可运行的代码。

下面我按照自己的理解，把整个实现拆成几个关键步骤来讲。

## 数据准备：流式解析 785MB 的 XML

搜索引擎的第一步永远是数据。这个项目用的是维基百科的摘要数据：一个 gzipped XML 文件，大约 785MB，包含 627 万条摘要。每条数据长这样：

```xml
<doc>
  <title>Wikipedia: London</title>
  <url>https://en.wikipedia.org/wiki/London</url>
  <abstract>London is the capital and largest city of England...</abstract>
</doc>
```

用 Python 的 `dataclass` 定义一个文档模型：

```python
from dataclasses import dataclass

@dataclass
class Abstract:
    """维基百科摘要"""
    ID: int
    title: str
    url: str
    abstract: str
    
    @property
    def fulltext(self):
        return ' '.join([self.title, self.abstract])
```

这里的 `fulltext` 属性值得注意：它把标题和正文拼在一起。这意味着搜索时标题里的词也会被命中，而且标题匹配本质上就获得了额外的权重，因为标题词在全文中的占比天然更高。

然后是解析逻辑。785MB 的 XML 不可能全加载到内存里，所以用 `lxml.etree.iterparse` 做流式解析：

```python
import gzip
from lxml import etree

def load_documents():
    with gzip.open('data/enwiki.latest-abstract.xml.gz', 'rb') as f:
        doc_id = 1
        for _, element in etree.iterparse(f, events=('end',), tag='doc'):
            title = element.findtext('./title')
            url = element.findtext('./url')
            abstract = element.findtext('./abstract')

            yield Abstract(ID=doc_id, title=title, url=url, abstract=abstract)

            doc_id += 1
            element.clear()  # 关键：释放已解析元素的内存
```

`element.clear()` 这一行很重要。如果不加，解析器会一直持有所有已解析的 DOM 节点，内存很快就会爆掉。这是一个典型的"用空间换时间还是用时间换空间"的工程选择：这里选择牺牲一点便利性，保证内存可控。

## 分词与分析管线：把文本变成可搜索的 token

原始文本是不能直接建索引的。你需要一条"分析管线"（analysis pipeline），把自然语言文本转换成结构化的、可搜索的 token 序列。

### 1. 分词

最朴素的做法：按空格切分：

```python
def tokenize(text):
    return text.split()
```

简单粗暴，但在这个场景下够用。生产系统当然需要处理 CJK 分词、复合词、连字符等问题，但原理是一样的。

### 2. 大小写归一化

```python
def lowercase_filter(tokens):
    return [token.lower() for token in tokens]
```

搜索引擎里几乎不会有大小写敏感的需求：你搜 "London" 和 "london" 期望的是同一个结果。

### 3. 标点符号过滤

```python
import re
import string

PUNCTUATION = re.compile('[%s]' % re.escape(string.punctuation))

def punctuation_filter(tokens):
    return [PUNCTUATION.sub('', token) for token in tokens]
```

去掉标点符号后，`"london."` 和 `"london"` 才能匹配到同一个 token。

### 4. 停用词过滤

```python
STOPWORDS = set(['the', 'be', 'to', 'of', 'and', 'a', 'in', 'that', 'have',
                 'I', 'it', 'for', 'not', 'on', 'with', 'he', 'as', 'you',
                 'do', 'at', 'this', 'but', 'his', 'by', 'from', 'wikipedia'])

def stopword_filter(tokens):
    return [token for token in tokens if token not in STOPWORDS]
```

停用词是那些出现频率极高但几乎没有区分度的词："the"、"a"、"of"、"is"。去掉它们能显著减小索引体积，而且对搜索质量的影响几乎为零。不过注意，停用词列表需要根据语料和场景调整，比如这里把 "wikipedia" 也加了进去，因为这个语料里每篇摘要都有这个词。

### 5. 词干提取

基于 PyStemmer（波特词干提取器）实现英文分词词干还原：

```python
import Stemmer

STEMMER = Stemmer.Stemmer('english')

def stem_filter(tokens):
    return STEMMER.stemWords(tokens)
```

词干提取（stemming）把不同形态的词归一到同一个词干：`"brewery"` 和 `"breweries"` 都变成 `"brewery"`，`"running"` 变成 `"run"`。这样搜 "run" 时也能命中包含 "running" 的文档。

注意停用词过滤必须在词干提取之前，因为停用词表里的词是未提取词干的形式。

### 组合起来

```python
def analyze(text):
    tokens = tokenize(text)
    tokens = lowercase_filter(tokens)
    tokens = punctuation_filter(tokens)
    tokens = stopword_filter(tokens)
    tokens = stem_filter(tokens)
    return [token for token in tokens if token]
```

这五个步骤构成了一条完整的分析管线。每一步都是独立的纯函数，可以单独测试、替换或扩展。这就是"管道-过滤器"模式在文本处理中的经典应用。

## 倒排索引：搜索引擎的核心数据结构

正排索引（forward index）是"文档 → 词列表"的映射，倒排索引（inverted index）反过来，是"词 → 文档 ID 列表"的映射。这个映射就是搜索引擎能"搜"的关键。

打个比方：正排索引像一本书的目录——按章节顺序告诉你每章讲了什么。倒排索引像书末的索引——告诉你某个词出现在哪些页。

实现上非常简单：

```python
class Index:
    def __init__(self):
        self.index = {}       # token → set of document IDs
        self.documents = {}   # document ID → Abstract object

    def index_document(self, document):
        if document.ID not in self.documents:
            self.documents[document.ID] = document

        for token in analyze(document.fulltext):
            if token not in self.index:
                self.index[token] = set()
            self.index[token].add(document.ID)
```

`index_document` 每次处理一个文档，把它经过 `analyze` 后产出的所有 token 注册到倒排索引中。用 `set` 存文档 ID 是因为同一个 token 可能在同一篇文档中多次出现，但我们只需要记录"这篇文档包含这个词"这个事实。

建索引的过程就是遍历所有文档，逐个调用 `index_document`。对于 627 万条摘要，这个过程在普通笔记本上大概需要几分钟。

## 布尔检索：AND 和 OR 的取舍

有了倒排索引，搜索就变成了集合运算。查询词经过同样的分析管线处理后，去索引里查每个 token 对应的文档 ID 集合，然后做交集（AND）或并集（OR）：

```python
def _results(self, analyzed_query):
    return [self.index.get(token, set()) for token in analyzed_query]

def search(self, query, search_type='AND'):
    if search_type not in ('AND', 'OR'):
        return []

    analyzed_query = analyze(query)
    results = self._results(analyzed_query)
    if search_type == 'AND':
        documents = [self.documents[doc_id] for doc_id in set.intersection(*results)]
    if search_type == 'OR':
        documents = [self.documents[doc_id] for doc_id in set.union(*results)]

    return documents
```

AND 查询要求文档同时包含所有查询词，精确但可能过严。OR 查询只要求包含任意一个，宽泛但可能返回太多结果。比如搜 "London Beer Flood"，AND 可能返回零条或个位数结果，OR 可能返回近 5 万条。

这就是布尔检索的局限：它只能告诉你文档"匹配"还是"不匹配"，无法告诉你"匹配得有多好"。结果是一个无序的列表——第 5 万条结果和第 1 条在系统眼里地位相同。这显然不是我们想要的。

## TF-IDF：让搜索结果有"相关性"

有没有办法让最相关的文档排在前面？这就是 TF-IDF 要做的事。

### TF：词频

一个词在文档中出现次数越多，说明这个词对这篇文档越重要。但直接用原始计数有个问题：长文档天然有更高的词频。所以通常用对数缩放：

```python
from collections import Counter

def analyze(self):
    self.term_frequencies = Counter(analyze(self.fulltext))

def term_frequency(self, term):
    return self.term_frequencies.get(term, 0)
```

这里用的是原始计数，没有做长度归一化。对于维基百科摘要这种长度相对均匀的语料，问题不大。但在生产系统中，你通常需要把 TF 除以文档总词数来做归一化。

### IDF：逆文档频率

光有 TF 不够。像 "the" 这样的词 TF 很高，但几乎没有区分度。IDF 衡量一个词在整个语料中的"稀有程度"——越稀有的词，区分度越高：

```python
import math

def document_frequency(self, token):
    return len(self.index.get(token, set()))

def inverse_document_frequency(self, token):
    return math.log10(len(self.documents) / self.document_frequency(token))
```

用 `log10` 是为了压缩数值范围。一个词如果出现在全部 627 万篇文档中，它的 IDF 是 `log10(1) = 0`，对排序没有任何贡献。一个词如果只出现在 10 篇文档中，它的 IDF 是 `log10(627000) ≈ 5.8`，会被赋予很高的权重。

### 组合排序

最终得分 = TF × IDF 的累加和：

```python
def rank(self, analyzed_query, documents):
    results = []
    if not documents:
        return results
    for document in documents:
        score = 0.0
        for token in analyzed_query:
            tf = document.term_frequency(token)
            idf = self.inverse_document_frequency(token)
            score += tf * idf
        results.append((document, score))
    return sorted(results, key=lambda doc: doc[1], reverse=True)
```

然后把 `search` 方法扩展一下，增加一个 `rank` 参数：

```python
def search(self, query, search_type='AND', rank=True):
    analyzed_query = analyze(query)
    results = self._results(analyzed_query)
    if search_type == 'AND':
        documents = [self.documents[doc_id] for doc_id in set.intersection(*results)]
    if search_type == 'OR':
        documents = [self.documents[doc_id] for doc_id in set.union(*results)]

    if rank:
        return self.rank(analyzed_query, documents)
    return documents
```

现在搜索结果不再是随机的，最相关的文档排在前面。对 "London Beer" 搜索，讲伦敦啤酒节的文档会排在讲一般伦敦旅游的文档之前。

## 写在最后

这个项目虽然只有 150 行，但它完整覆盖了一个搜索引擎的核心链路：**数据加载 → 文本分析 → 倒排索引 → 布尔检索 → 相关性排序**。每一步都是搜索引擎领域几十年来沉淀下来的基础概念。

当然，它离生产系统还很远。Bart 在原文中也提到了几个可以扩展的方向：

- **标题加权**：标题里的词匹配应该比正文里的词匹配获得更高的分数
- **查询语法**：支持引号精确匹配、减号排除、混合 AND/OR
- **索引持久化**：把索引写到磁盘，避免每次启动都要重建
- **更丰富的分析管线**：比如 N-gram、同义词扩展、词形还原（lemmatization）替代词干提取

如果你对搜索引擎感兴趣，这个项目是一个很好的起点。150 行代码读完就能理解，改几行就能看到效果。当你理解了倒排索引和 TF-IDF 之后，再去学 Elasticsearch 或 Lucene，你会发现它们本质上就是这些基础概念在工程上的极致优化：用跳表加速集合运算、用 BM25 替代 TF-IDF、用 FST 压缩倒排索引等等。

但核心思想，从来没变过。

> - 完整代码: [GitHub: python-searchengine](https://github.com/bartdegoede/python-searchengine)
> - 参考文章: [Building a full-text search engine in 150 lines of Python](https://bart.degoe.de/building-a-full-text-search-engine-150-lines-of-code/)