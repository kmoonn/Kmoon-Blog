+++
title = "Google 开源 Agent OKF 新标准"
date = "2026-06-17T09:34:20+08:00"
lastUpdated = "2026-06-17"
description = "Google 开源 OKF 规范，用 Markdown + YAML frontmatter 标准化 AI 知识库格式。本质是 Karpathy LLM Wiki 模式的工业化，补上 AI 知识栈的最后一公里。"
tags = ["OKF", "AI Agent", "知识管理"]
+++

> Reference:
> - [Introducing the Open Knowledge Format](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing/)
> - [AI 寒武纪：谷歌突然开源Agent OKF新标准！](https://mp.weixin.qq.com/s/nhF1cy_lIQukq_niVFvRCA)
> - [GitHub：knowledge-catalog](https://github.com/GoogleCloudPlatform/knowledge-catalog/tree/main/okf)

上回我写了 [RAG vs LLM Wiki 的分析](/blog/rag-vs-llm-wiki/)，核心矛盾是预编译的 wiki 和原始信源之间的取舍。Karpathy 的思路是让 LLM 先把所有文档读一遍，整理成结构化的知识库，以后直接读 wiki 就行。没想到两个月不到，Google 就把这套模式标准化了。

Open Knowledge Format，简称 OKF，6 月 12 日发布，[GitHub 仓库](https://github.com/GoogleCloudPlatform/knowledge-catalog/tree/main/okf)三天冲到 3300+ stars。它要做的事情很简单：给 AI 知识库定一个通用的、可移植的格式规范。

## 为什么需要统一格式

我们平时给 AI 的文档大部分都是内部知识，来源方方面面：数据库表字段含义、接口定义规范、业务功能模块、平台操作手册。这些文档没有统一格式，不同部门关注的重点也不同。这就导致每次新开发一个 Agent，都得从头注入、清洗、梳理一遍知识库。知识没有流通，锁在创建它的那个平台里。

更麻烦的是，每个人搭建知识库的方式都不同。有的全部放在一起靠目录区分模块，有的分模块放在不同知识库；有的用长串文件名标记属性，有的更习惯用 frontmatter。对人类来说这无所谓，自己写的自己找得到。但 Agent 需要的是一份格式稳定的"说明书"，不是每次都重新学习你的目录组织习惯。

## OKF 是什么

核心设计很简单：一个 OKF bundle 就是一个 Markdown 文件目录，每个文件代表一个概念（数据表、数据集、指标、操作手册、API 等），文件路径就是这个概念的唯一标识。

目录结构长这样：

```
sales/
├── index.md
├── datasets/
│   └── orders_db.md
├── tables/
│   ├── orders.md
│   └── customers.md
└── metrics/
    └── weekly_active_users.md
```

每个文件分两部分：顶部一小块 YAML frontmatter 存结构化字段，下面是 Markdown 正文，写什么完全自由。

一个完整的概念文件：

```markdown
---
type: BigQuery Table
title: Orders
description: One row per completed customer order.
resource: https://console.cloud.google.com/bigquery?p=acme&d=sales&t=orders
tags: [sales, revenue]
timestamp: 2026-05-28T14:30:00Z
---

# Schema

| Column        | Type      | Description                              |
|---------------|-----------|------------------------------------------|
| `order_id`    | STRING    | Globally unique order identifier.        |
| `customer_id` | STRING    | FK to [customers](/tables/customers.md). |

# Joins

Joined with [customers](/tables/customers.md) on `customer_id`.
```

概念之间用普通 Markdown 链接互引，整个目录变成一张关系图，比文件系统的父子层级丰富得多。

合规条件就三条：每个非保留文件必须有可解析的 YAML frontmatter；frontmatter 必须包含非空的 `type` 字段；保留文件名（`index.md`、`log.md`）遵循规定结构。其他一切随意。

而且消费者被要求尽量宽容：不能因为缺少可选字段、遇到未知 type、多出额外 frontmatter key，甚至链接断了就拒绝解析。这设计哲学很有意思——宁可多读点废话，也别因为格式细节把合法内容挡在门外。

整个[规范文件](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md)才 451 行、14.7 KB。Google 自己说的：Read the spec, it's short! 这话倒没骗人。

## Karpathy 的影子

OKF 不是凭空出来的。Karpathy 今年 4 月发了 [LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)，两个月拿到 5000+ stars，核心思路就是"每个概念一个 Markdown 文件，LLM 负责维护，互链成图"。OKF 本质上是在标准化这个模式：把 Karpathy 的个人实践固化成一份开放规范，让不同团队、不同工具产出的知识库可以互相识别和消费。

我之前分析 [RAG vs LLM Wiki 的取舍](/blog/rag-vs-llm-wiki/)时说过，编译好的 wiki 有错误滚雪球、写入成本膨胀、分布式一致性这些问题。OKF 解决的是另一个层面的问题：如果大家都觉得 wiki 模式是对的，那格式得统一，不然你的 wiki 我的 wiki 各玩各的，还是没法流通。

Karpathy 估算过，维护得当的 Markdown 知识库相比直接灌原始文档，能减少高达 95% 的 token 消耗。OKF 想做的就是让这个"维护得当"有一个可遵循的底线。

## AI 知识的四个层次

参考文章里提了一个我觉得很有意思的分层框架，把 AI 获取知识的方式拆成四层：

- **sitemap.xml** 告诉爬虫哪些 URL 存在
- **llms.txt** 告诉 agent 哪些页面最值得读
- **EntityMap** 声明实体和它们之间的关系
- **OKF** 把内容本身递到 agent 手里，每个页面是一个干净的概念，互链成图

这四层不是替代关系，是从索引到内容的递进。sitemap 回答"有什么"，llms.txt 回答"看什么"，EntityMap 回答"谁和谁有关"，OKF 回答"具体是什么"。OKF 补的是最后一公里，不是告诉你去哪找，而是直接把知识递给你。

## 三个设计原则

### 极简主义

OKF 对每个文件只要求一个字段：`type`。其他一切都留给内容创建者。存在哪些类型、包含哪些字段、正文写哪些部分，规范都不管。它定义的是互操作性接口，不是内容模型。

说实话，`type` 是唯一必填字段，意味着你可以把任何东西塞进去，只要标上 type。灵活，但也意味着两个 OKF bundle 可能长得完全不一样。"格式统一"只统一了最外层的壳，里面的内容组织还是各写各的。

### 生产者/消费者独立性

OKF 把知识编写者和使用者清晰分离。人手写的文件，AI agent 可以读。元数据流水线生成的 bundle，可视化工具可以浏览。一个 LLM 生成的 bundle，另一个 LLM 可以查询。格式是契约，两端工具可以独立替换。

### 格式本身，不是平台

OKF 不绑定任何云服务、数据库、模型厂商或 agent 框架。读写不需要专有账号或 SDK。

不过有意思的是，发布 OKF 的同一天，Google Cloud 也更新了 Knowledge Catalog，让它原生支持导入 OKF bundle。Knowledge Catalog 管的是 OKF 故意不碰的部分：存储、服务、查询、权限控制。开放格式，付费服务层。这策略眼不眼熟？

## 3 个参考实现

Google 随规范一起发布了 3 个参考实现：

1. **enrichment agent**：工作在 BigQuery 数据集上的数据增强 Agent。先针对每个表和视图写 OKF 概念文件草稿，再跑一遍 LLM 爬取权威文档，补充 schema、引用和关联路径。基于 Google ADK 构建，Gemini 做后端模型。

2. **static HTML visualizer**：把任何 OKF bundle 转成单个自包含 HTML 文件里的交互式图形视图。无需后端，数据不会离开页面。

3. **3 个示例 bundle**：GA4 电商数据集、Stack Overflow、比特币公开数据集。都是用上面的 enrichment agent 生成的，放在 repo 里作为格式合规的示例。

## 说点实在的

OKF 的极简设计方向是对的。Markdown 是人机共读的最佳折中，用文件路径做概念 ID 很聪明，YAML frontmatter 兼顾了结构化查询和人类可读。规范本身短到 451 行，降低了采纳门槛。

但社区也有不同声音。有人直接说"这不就是一个带 YAML frontmatter 的文件夹吗？"更尖锐的质疑是：如果目标消费者是 AI，为什么选 Markdown 而不是更结构化的 JSON 或 Protobuf？Markdown 嵌套表格渲染不好，复杂 schema 表达力有限。

我觉得关键在于 OKF 不替代 RAG 和 MCP，它们是互补的。RAG 处理大规模动态文档档案，OKF 处理稳定的、精选的组织知识。MCP 管的是 agent 怎么连工具和实时数据，是"插座"；OKF 管的是 agent 对数据源知道什么，是"流过插座的知识"。一个 MCP server 完全可以把 OKF bundle 作为知识源暴露出来。三者不在一个层面上竞争。

真正的顾虑是：OKF 还只是 v0.1。三天 3300 stars 说明社区有需求，但格式规范的真正考验不在发布而在采纳。需要多少生态工具跟上，多少团队真的用 OKF 格式出包，才能形成正循环？标准这东西，用的人多了才是标准，发布了没人用就只是一份文档。

## 写在最后

OKF 定义的是知识的流通格式，不是知识的替代品。格式能打通孤岛，但填进去的内容还得靠人。
