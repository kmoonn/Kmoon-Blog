+++
title = "SubAgent：拆活给隔离专家，做完交结果"
slug = "subagent-101"
date = "2026-09-05T18:00:00+08:00"
tags = ["AI Agent", "SubAgent"]
+++

说实话，我第一次听说 SubAgent 的时候，第一反应是：这不就是把一个 Agent 拆成多个吗？有什么难的？

后来真上手做了才发现，拆容易，拆完不出事难。

一个 Agent 干所有事，就像一个程序员既写前端又写后端又管运维又做设计——能干，但越往后越慢，context 越来越杂，到最后一个简单 bug 要翻半天聊天记录才能定位。SubAgent 的本质就是把"全栈一个人"变成"专业分工团队"，每个子智能体只管自己那一摊，做完把结果交回来。

但这不是简单的"人多了活就快"。拆了之后怎么协调、怎么避免重复劳动、怎么处理一个子智能体挂了整条链还在不在——这些才是真正要解决的问题。

这篇我先把 SubAgent 的核心概念和架构模式讲清楚，再落到我实际用过的框架讲体感和坑。

## 一个 Agent 的天花板

单 Agent 在简单任务上表现很好，但遇到复杂任务有三个硬伤：

**context 会爆炸。** 一轮对话到 40 条消息的时候，context 里已经塞了 150k+ token 的内容，大部分是中间过程的废料——调试日志、尝试失败的代码、过期的规划。Agent 在这些噪声里找信息，效率直线下落。

**工具选择会退化。** 一个 Agent 挂 50 个工具，LLM 做 tool selection 的准确率明显下降。你让它从 50 个工具里挑对的，比从 10 个里挑要难得多。这不是模型能力的问题，是信息过载。

**错误会雪球。** 单 Agent 的错误没有隔离边界。第一步规划错了，后面所有步骤都基于错误前提执行，而且 Agent 很难自己发现"我一开始就走错路了"，因为纠错需要跳出当前 context 重新审视，但 context 里已经全是基于错误前提的推理了。

这三个问题不是调 prompt 能解决的，是架构层面的瓶颈。

## SubAgent 的核心模型

SubAgent 的执行模型其实就六步：

1. 主 Agent 收到目标
2. 主 Agent 把目标拆成子任务
3. 主 Agent 为每个子任务 spawn 一个子智能体
4. 子智能体各自独立执行
5. 子智能体返回结构化结果
6. 主 Agent 聚合、验证、必要时迭代

核心区别在于：**子智能体有独立的 context、独立的工具集、独立的生命周期。** 它不是并行调用同一个 Agent，而是每个子任务交给一个全新隔离的推理过程。

这里有个容易混淆的点：SubAgent ≠ 并行 tool call。并行 tool call 是同一个 Agent 在同一个 context 里同时调用多个工具，没有独立推理。SubAgent 是每个子任务有自己的推理循环，有自己的 context，做完才把结果交回来。

打个比方：并行 tool call 是一个人同时开三个终端跑命令，SubAgent 是三个人各领一个任务分头做，做完汇总。

## 四种架构模式

目前工业界实践下来，SubAgent 的编排模式基本收敛到四种：

### Orchestrator-Worker（中心调度）

最主流的模式。一个主 Agent 负责拆任务和汇总，多个 Worker 各做各的，Worker 之间不直接通信，所有信息流经主 Agent。

```
        [Orchestrator]
       /    |    \
  [Worker1] [Worker2] [Worker3]
```

好处是控制力强、容易 debug。坏处是主 Agent 是单点——它挂了全挂，它规划错了全错。2025 年的研究发现，中心调度模式下，主 Agent 的一个错误注入可以导致 100% 的子 Agent 全部失败。

**什么时候用：** 任务有明确的领域划分（前端 / 后端 / 测试），或者有合规隔离需求。

### Map-Reduce（扇出扇入）

主 Agent 把任务拆成若干独立子任务，并行扇出给 Worker，完成后扇入合并结果。

跟 Orchestrator-Worker 的区别是：子任务之间真正独立，没有依赖关系，不需要协调。典型的比如"同时审 10 个文件的代码质量"。

**什么时候用：** 子任务之间无写冲突，纯只读分析类的活。

### Pipeline（流水线）

Agent 按固定顺序串联，每个 Agent 的输出是下一个的输入。Draft → Review → Revise。

**什么时候用：** 有明确阶段边界、每步产出明确产物的任务。风险是前面的错误会传播到后面——Draft 写歪了，Review 基于歪的 Draft 审，Revise 基于歪的 Review 改。

### Peer-to-Peer（对等协作）

Agent 之间直接通信，没有中心调度。AutoGen 的 Group Chat 就是这个模式。

说实话，这个模式在生产环境基本不靠谱。2025 年的研究发现，Peer-to-Peer 模式在规模上来之后，协调崩溃、消息爆炸、共识迟滞的问题非常严重。目前各家都收敛到了 Orchestrator-Worker 作为默认模式。

## 各框架怎么实现 SubAgent

### Claude Code 的 Agent Tool

Claude Code 的 SubAgent 通过 `Agent` 工具实现，有两种核心类型：

**标准 SubAgent：** 启动时获得全新 context，只收到你显式传的 prompt，做完只返回最终结果。中间所有工具调用和推理过程对主会话不可见。

**Fork SubAgent：** 继承主会话的完整 context，但执行过程仍然隔离。Fork 的好处是复用主会话的 prompt cache，启动成本大约只有标准 SubAgent 的 10%。不能嵌套 fork。

隔离模式上，`isolation: "worktree"` 会给子智能体分配独立的 git worktree，防止并行修改同一个文件互相覆盖。

自定义 SubAgent 放在 `.claude/agents/` 目录下，用 Markdown + frontmatter 定义：

```markdown
---
name: code-reviewer
description: 审查代码质量和安全问题
tools: Read, Grep, Bash
---

# Code Reviewer

你的任务说明...
```

frontmatter 里的 `tools` 字段控制工具权限，这是 SubAgent 安全模型的核心——审查类 Agent 只给 Read，实现类 Agent 才给 Edit/Write。

我在实际使用中最深的体感是：**给 SubAgent 的 prompt 必须具体到边界。** "帮我审查这个文件"不行，"审查 src/auth/login.ts 的错误处理逻辑，关注未捕获异常和错误信息泄露"才行。模糊的指令会导致子智能体浪费多轮重新发现主 Agent 已知的信息。

### OpenAI Agents SDK

OpenAI 2025 年 3 月发布了 Agents SDK，两种委托模式：

**Handoffs（移交）：** triage Agent 把对话路由给专家 Agent，专家 Agent 接管整个对话。用户直接跟专家交互。

**Agent-as-Tool（管理器模式）：** 主 Agent 把专家 Agent 当工具调用，自己保持控制权，调用完拿回结果继续。

```python
# Agent-as-Tool
manager = Agent(
    name="Manager",
    instructions="Route to specialists",
    tools=[research_agent.as_tool(), writer_agent.as_tool()]
)
```

Handoffs 适合专家应该直接面对用户的场景（客服转接），Agent-as-Tool 适合主 Agent 必须保持控制的场景（研究报告）。

### LangGraph 的 Subgraph

LangGraph 1.0 把 subgraph composition 作为官方扩展路径。编译后的 subgraph 可以作为单个节点嵌入父图，状态通过图状态对象共享。

三种模式：Supervisor/Subagent（中心调度）、Orchestrator-Worker（map-reduce）、Subgraph Composition（子图嵌入）。

LangGraph 的特色是原生支持 fan-out/fan-in——通过 `Send` API 把任务并行分发，然后汇合结果。

Build.inc 用 25+ 个 subgraph worker 做数据中心开发，把 4 周的开发周期压到 75 分钟。这是目前我看到的 SubAgent 在生产环境最亮眼的数据。

### CrewAI 的 Hierarchical 模式

CrewAI 用 `Process.hierarchical` 实现经理-工人模型。经理 Agent 自动拆任务、派活、汇总。

```python
crew = Crew(
    agents=[research_manager, market_researcher, report_writer],
    tasks=[research_task],
    process=Process.hierarchical,
    manager_llm=some_smart_llm
)
```

CrewAI 的坑在于**无界委托**——`allow_delegation=True` 的 Agent 可以无限嵌套委托，子 Agent 委托子子 Agent，token 爆炸。解法是在工具层做预算追踪，而不是靠 prompt 告诉模型"别花太多钱"。

### Google ADK

ADK 是 Cloud Next 2025 发布的，天生 multi-agent。三种内置工作流 Agent：`SequentialAgent`（顺序）、`LoopAgent`（迭代+质量门控）、`CoordinatorAgent`（智能路由）。

子 Agent 通过 `.subAgents()` 注册，也支持 Agent-as-Tool 模式绕过 Gemini 的多工具配置限制。

## 代价：不是免费的午餐

SubAgent 解决了单 Agent 的瓶颈，但引入了新的成本。

**Token 开销翻倍。** 多 Agent 系统的总 token 消耗是单 Agent 的 8-15 倍。因为每个子 Agent 都要加载自己的 system prompt、工具描述、任务说明，这些都是重复的固定开销。一个 $15/天的单 Agent 系统，拆成多 Agent 大约 $120-225/天。

但 token 开销不等于 context 开销。子 Agent 帮主 Agent 省的是**有效 context**——一个调试内存泄漏的任务，中间过程可能产生 175k token 的日志和堆转储，用子 Agent 处理完只返回 6k token 的摘要，主 Agent 的 context 从 175k 降到 6k，97% 的缩减。

所以核心判断是：**你愿意用总 token 的增长，换主 context 的干净。** 对于 context 会爆的任务，这笔账是划算的；对于简单任务，就是浪费。

**沉默失败是真实风险。** 子 Agent 返回空结果或幻觉结果，主 Agent 很难分辨。不像单 Agent 出错你能在 context 里追溯推理链，子 Agent 的中间过程对主 Agent 不可见。解法是要求子 Agent 返回结构化结果 + 置信度，主 Agent 做验证，不能无脑信任。

**协调通信有成本。** 每个子 Agent 的 context 里要存协调消息、工具 schema、状态更新，这些是纯开销。1600 条执行 trace 的分析发现，协调通信本身就会导致 context 退化。

**并行写入是危险区。** 只读并行是 SubAgent 最强用例——同时审 10 个文件，互不干扰。但写并行不行，两个子 Agent 同时改同一个文件，后面的覆盖前面的，而且你可能都不知道覆盖了。所以 Claude Code 才引入 worktree 隔离——每个子 Agent 在独立分支上写，最后合并。

## 什么时候该用 SubAgent

这是最关键的问题。不是所有任务都该拆。

我的判断标准就一条：**你能说出具体的瓶颈是什么吗？**

如果说不出来，单 Agent 够用。如果说出来，SubAgent 才有意义。目前实践下来有效的四个瓶颈：

1. **重度并行** — 10 个模块同时跑测试，串行 50 分钟，并行 5 分钟
2. **信息量超 context** — 一个任务要读 20 个大文件，单 Agent context 装不下
3. **高频工具操作需隔离** — 比如并行写文件，不隔离就互相覆盖
4. **合规/领域边界** — 财务 Agent 和社交媒体 Agent 不该共享 prompt 和记忆

不在这四个里的场景，拆了大概率是增加复杂度但不增加收益。

## 我在做的：一个真实的 SubAgent 工作流

上面说的都是概念和框架，让我落到一个我正在做的真实项目——字节 Ads Platform 的 Platform Intelligence（PI）。

PI 是一个 AI Native 工作台，核心是一个叫 PI Agent 的主智能体，根据任务类型切模式、加载不同 Skill，复杂任务触发专属流程编排。整个需求交付链路长这样：

```
[需求澄清] → [Full Spec 生成] → [代码实现] → [测试验收]
     ↑              ↑               ↑            ↑
  PM Skills    PM + QA Skills    RD Skills    QA Agent
```

这里每个节点本质上就是一个 SubAgent，PI Agent 是 Orchestrator，负责拆任务、串流程、汇总结果。

我负责的是**测试验收节点**。这个节点在链路里有两个出场时机：

**第一场：Full Spec 阶段。** PI Agent 澄清完需求后，要生成一个 Full Spec——里面包含产品设计、技术概要设计和测试用例。测试用例不是 RD 自己编的，是 QA Skill 生成的。这时候我的角色是提供"测试用例生成"这个 Skill，作为一个被 PI Agent 调用的 SubAgent，输入是需求描述 + 技术概要设计，输出是结构化的测试用例列表。

**第二场：测试验收阶段。** RD 的 Coding Agent 拿到 Full Spec 开始写代码，写完跑测试用例，跑不过就修，修完再跑——这就是"跑通验证 Loop"。Loop 里的测试执行就是 QA Agent 的事。Coding Agent 不自己跑测试，它把测试执行交给 QA Agent，QA Agent 返回"哪些过了、哪些没过、失败原因是什么"，Coding Agent 拿到结果决定修哪里。

这个设计里的几个关键判断，正好能印证前面说的 SubAgent 原则：

**为什么测试用例不由 RD Agent 自己生成？** 因为 RD 和 QA 的视角不一样。RD 写测试用例容易"沿着实现写"——代码怎么写的就怎么测，测的是"代码跑得通"，不是"需求真满足了"。QA Skill 独立生成测试用例，是从需求规格出发，能覆盖 RD 视角盲区。这是**领域边界需要拆 SubAgent** 的典型案例。

**为什么测试执行不由 Coding Agent 自己做？** 因为跑测试是个高频工具操作，而且可能改环境、改数据。如果 Coding Agent 自己跑测试，它的 context 里会塞满测试框架的输出日志、失败堆栈、重试记录——这些对写代码没帮助，反而挤占有效 context。QA Agent 跑完只返回"3 个通过，2 个失败：XX 断言不匹配"，Coding Agent 的 context 保持干净。这是**信息量超 context 需要拆 SubAgent** 的典型案例。

**跑通验证 Loop 本质上是什么？** 是 Pipeline + 条件循环：Coding Agent 写代码 → QA Agent 跑测试 → 全过则交付，否则 Coding Agent 修代码再来一轮。Loop 的出口条件是"所有测试用例通过"，这是硬质量门控，不是靠 LLM 自己判断"我觉得差不多了"。

说实话，做这个项目之前，我对 SubAgent 的理解还停留在"Claude Code 的 Agent 工具"。做完之后最大的感受是：**生产级的 SubAgent 不是 spawn 一个子智能体就完事了，核心工作是定义节点间的输入输出契约、出口条件和失败回退策略。** 技术实现反而不是最难的部分。

## 我的体感

用了几个月 Claude Code 的 SubAgent，最深的感受是三句话：

**拆任务的粒度比拆任务本身更重要。** 粒度太粗，子 Agent 还是在干全栈的活，没意义；粒度太细，协调开销比干活还大。实际舒服的粒度是：一个子 Agent 的工作量在 3-8 个工具调用内完成。

**prompt 越具体，子 Agent 越靠谱。** 这跟给人派活一样——"帮我看看这个"和"帮我审查 login.ts 的错误处理，关注未捕获异常"效果天差地别。模糊指令的代价不是子 Agent 做错，是它做多——反复搜索、反复尝试，浪费 token 还浪费时间。

**Fork 是性价比最高的模式。** 标准子 Agent 启动一次要重新加载所有 context，Fork 直接继承主会话的缓存，成本约 1/10。对于需要共享上下文的子任务（比如"基于我们刚才讨论的架构方案，帮我写测试"），Fork 远优于新建。

## 写在最后

SubAgent 的本质是工程上的分治，不是能力上的增强。一个子 Agent 不会比主 Agent 更聪明，但拆开之后每个子 Agent 的 context 更干净、工具选择更精准、错误范围更可控。

这跟代码重构是一个道理——你把一个 2000 行的函数拆成 10 个 200 行的函数，不是因为小函数比大函数算得快，而是因为小函数更容易理解、更容易测试、出了 bug 更容易定位。

但跟代码重构也一样，拆分有成本。多一层抽象就多一层可能出错的东西。子 Agent 的协调、验证、容错，都是主 Agent 要额外操心的事。

所以回到核心判断：**先单 Agent，遇到具体瓶颈再拆。** 不要因为"多 Agent 更酷"就拆，要因为"单 Agent 真的不够了"才拆。

目前的行业共识也收敛到了这个结论——Anthropic、OpenAI、LangChain 在 2025-2026 年都不约而同地把 Orchestrator + isolated subagents 作为唯一生产级架构，其他花哨的模式（对等协作、无限嵌套）在规模上来后都撑不住。

SubAgent 不是银弹，是手术刀。该用的时候用，不该用的时候别拿着到处划。
