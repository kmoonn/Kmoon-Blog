+++
title = "Google 开源 Agent OKF 新标准重新定义 AI 知识库"
date = "2026-06-17T09:34:20+08:00"
tags = ["OKF"]
+++

> Reference: 
> - [Introducing the Open Knowledge Format](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing/)
> - [AI 寒武纪：谷歌突然开源Agent OKF新标准！](https://mp.weixin.qq.com/s/nhF1cy_lIQukq_niVFvRCA)
> - [GitHub：knowledge-catalog](https://github.com/GoogleCloudPlatform/knowledge-catalog/tree/main/okf)

## AI 知识库迎来通用格式？

谷歌昨天发布了一个叫 Open Knowledge Format（OKF）的开放规范，计划为 AI 知识库的格式和规范制定一个统一的、通用的、可移植的格式。

我们平时给 AI 的文档大部分都是内部知识，来源方方面面，比如数据库表字段含义、接口定义规范、业务功能模块、平台操作手册等等。这些文档没有统一的格式，不同部门重点关注的内容也不同，这就导致每次新开发一个 Agent，都需要从头到尾注入、清洗、梳理一遍各种知识库。知识没有流通，都锁在不同的创建它的那个平台。

而且每个人、每个团队搭建自己知识库的方式都不同，有的全部放在一起通过目录区分模块，有的更愿意分模块放在不同的知识库；有的根据长串文件名标记属性，有的更愿意通过 frontmatter 标记属性。

谷歌今天发布的 OKF，是一个格式规范，不是服务，不需要 SDK，不绑定任何云平台。

## Open Knowledge Format

核心设计很简单：一个 OKF bundle 就是一个 Markdown 文件目录，每个文件代表一个概念（可以是数据表、数据集、指标、操作手册、API 等），文件路径就是这个概念的唯一标识。

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

每个文件有两个部分：顶部一小块 YAML frontmatter，用于存储可以被查询的结构化字段，下面是 Markdown 正文，写什么内容完全自由。

一个完整的概念文件长这样：

```
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

概念之间用普通的 Markdown 链接互相引用，整个目录就变成了一张关系图，比文件系统的父子层级丰富得多。

## 设计前的三个原则

OKF 的格式刻意追求极简：一个包含 Markdown 文件和 YAML frontmatter 的目录。

### 1. 极简主义（Minimally opinionated）

OKF 对每个文件的要求只有一个：必须有 type 字段。其他（例如存在哪些类型、包含哪些其他字段、主体包含哪些部分）都留给了内容创建者。该规范定义的是互操作性接口，而非内容模型。

### 2. 生产者/消费者独立性（Producer/consumer independence）

OKF 将知识的编写者和使用者清晰地分离。人手写的知识文件，可以被 AI agent 读取。元数据导出流水线生成的 bundle，可以在可视化工具里浏览。一个 LLM 生成的 bundle，可以被另一个 LLM 查询。格式是契约，两端的工具可以独立替换。

### 3. 格式本身，不是平台（Format, not platform）

OKF 不绑定任何云服务、数据库、模型厂商或 agent 框架。读写它不需要任何专有账号或 SDK。谷歌选择把它作为开放标准发布，因为知识格式的价值来自有多少人在用它，不是来自谁拥有它。

## 3 个官方参考实现

Google 随 OKF 开放规范一起发布的还有 3 个生产者端和消费者端的参考实现。

### 1. enrichment agent

一个工作在 BigQuery 数据集上的数据增强 Agent，先针对每个表和视图写了一份 OKF 概念文件草稿，然后再跑一遍 LLM，爬取权威文档，补充 schema、引用和关联路径。

### 2. static HTML visualizer

一个静态 HTML 可视化工具，可将任何 OKF bundle 转换为单个自包含文件中的交互式图形视图；无需后端，查看端无需安装，数据不会离开页面。

### 3. Three ready-to-browse sample bundles

GA4 电商数据集、Stack Overflow、比特币公开数据集，都是用上面那个参考 agent 生成的，提交在 repo 里作为格式合规的示例。

## 写在最后

Google 在文章后面列了几点鼓励我们做的事情：

- Read the spec (it's short!)
- Write a producer for your source system, your database, your documentation site
- Write a consumer: a viewer, a search index, an agent that reasons over bundles
- Try the reference implementation against your own data

翻译过来就是说：首先阅读一下 OKF 规范，它很短；针对你自己的场景分别写一个生产者和一个消费者；在你自己的数据上尝试一下参考实现。