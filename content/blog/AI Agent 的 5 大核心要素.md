+++
title = "AI Agent 的 5 大核心要素"
date = "2026-04-25T21:39:28+08:00"
tags = ["AI Agent"]
+++

AI Agent 的五大核心要素，缺一不可。

这五大核心要素是当前工业界 + 学术界公认的「通用智能 Agent 必备能力」，没有任何一个可以省略 —— 缺少任意一个，都只是「LLM + 简单工具调用」，而非真正具备自主决策、执行、纠错能力的 AI Agent。

## 真正的 AI Agent = 5 要素闭环

```
[目标规划器] → [工具执行引擎] → [记忆系统]
       ↑            ↓            ↓
[自省自修正] ←─ [观测反馈闭环] ←─
```

缺一不可：
- 无规划 → 只会单步执行，无法完成复杂任务
- 无工具 → 只能聊天，无法操作现实世界
- 无记忆 → 每轮对话都是 “失忆状态”
- 无观测反馈 → 执行失败就卡死，无法容错
- 无自省修正 → 只会重复犯错，无法进化

## Goal-Oriented Planner

目标导向规划器。


不是简单 prompt engineering，而是将高层目标（如“帮我订一张下周从北京到上海的高铁票并同步到日历”）自动分解为带依赖关系的子任务图（Task Graph）：

> [查询12306余票] → [选择车次] → [调用支付接口] → [写入Google Calendar API]。

```json
{
  "task_id": "T0",
  "goal": "订下周北京→上海高铁票并同步日历",
  "sub_tasks": [
    {"id": "T1", "depends_on": [], "tool": "query_12306", "params": {...}},
    {"id": "T2", "depends_on": ["T1"], "tool": "select_train"},
    {"id": "T3", "depends_on": ["T2"], "tool": "pay_order"},
    {"id": "T4", "depends_on": ["T3"], "tool": "write_calendar"}
  ]
}
```

规划器必须支持重规划（Replan）—— 这是 Agent 和普通脚本的核心区别。

典型实现：Tree-of-Thought（ToT）或 Chain-of-Thought（CoT）+ LLM-based task decomposition，配合结构化输出约束（如JSON Schema）确保可解析性。

## Tool Integration & Execution Engine

工具集成与执行引擎。

必须支持**动态**工具注册、参数校验、沙箱化调用、超时熔断、重试策略。

工具描述需遵循标准化协议（如OpenAI Function Calling格式或LangChain Tool Interface），底层通过反射机制（Python inspect）或AST解析实现函数元信息提取。

关键点：工具调用不是字符串拼接，而是类型安全的函数对象绑定与异步IO调度（如asyncio.run_in_executor避免阻塞LLM线程）。

```python
# 工具元信息自动提取（inspect）
def query_12306(dep: str, arr: str, date: str) -> dict:
    """查询12306余票"""
    pass

# 执行引擎：反射调用
tool_func = globals()["query_12306"]
result = await asyncio.run_in_executor(None, tool_func, "北京", "上海", "2025-12-20")
```

## Memory System

记忆系统。

分为短期记忆（Short-term Memory，即当前session的token-aware context window，需做RAG式压缩或summary truncation防止overflow）和长期记忆（Long-term Memory，如向量数据库Chroma/Weaviate存储用户偏好、历史行为，支持HyDE或Self-RAG检索）。

注意：Memory不是缓存，而是带时间戳、来源标签、置信度评分的结构化知识图谱（Knowledge Graph），支持反事实查询（“上次我拒绝的酒店类型是什么？”）。

- 缓存 = 加速读取
- 记忆 = 可理解、可推理、可用于决策的结构化知识

## Observation & Feedback Loop

观测与反馈闭环。

Agent必须能接收非文本反馈：API返回的HTTP status code、JSON error字段、CLI命令的stderr、甚至浏览器自动化中的DOM变更事件。

典型设计模式是“Action-Observation Cycle”：每执行一个tool_call后，强制插入observation parser（正则/Schema校验/LLM classifier）判断是否成功；失败时触发replan（而非重试），例如支付失败→切换支付方式→重新生成订单号。

该循环在代码层面体现为while loop + state machine（如使用transitions库建模状态转移）。

## Self-Reflection & Self-Correction

自省与自修正。

超越简单retry，包含三层次机制：

- ① 内省层（Introspection）：LLM对自身上一步action生成critique（“我未检查身份证号格式，导致12306接口报错400”）；
- ② 规划层修正（Replanning）：根据critique重构task graph，插入格式校验子步骤；
- ③ 元策略层（Meta-policy）：记录高频失败模式，生成新rule加入memory（如“所有身份证字段必须经re.match(r'^\d{17}[\dXx]$')验证”）。这要求Agent具备可追溯的execution trace（如LangGraph的checkpoints或自研log schema）。

## 写在最后

![](https://cdn.kmoon.fun/2026/2026-06-17T12-39-47-784Z.png)

1. 这五大要素是 AI Agent 的充要条件，真正的工业级 Agent（如 Devika、OpenAI Code Interpreter、AutoGPT）均严格遵循这套架构；
2. 底层技术栈：LLM + 任务规划 + 函数调用 + 向量库 + 状态机 + 执行轨迹；
3. 核心判断标准：能自主规划、自主执行、自主感知、自主纠错，才算 AI Agent。