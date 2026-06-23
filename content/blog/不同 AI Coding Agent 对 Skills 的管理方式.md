+++
title = "不同 AI Coding Agent 对 Skills 的管理方式"
date = "2026-06-22T19:57:10+08:00"
tags = ["Coding Agent", "Agent Skills"]
+++

前两天我在做一个工具，想统计一下自己每天到底调用了多少次 Claude Code 的 skill。

听起来很简单对吧，就是做个上报，记录一下什么时候用了什么 skill，用了几次。但我刚开始写代码就懵了。

Claude Code 的 skill 是本地 Markdown 文件，Cursor IDE 的 skill 藏在设置里的 JSON 配置，Codex CLI 的 skill 是 YAML 格式。我要适配所有 Coding Agent，却发现它们管理 skill 的方式完全是八套不同的逻辑。

不是哥们，我就想记个日志，怎么各家差别能这么大？？？

这件事让我意识到，如果你想做一个通用的 skill 调用上报工具，首先得搞清楚每家 Agent 在三个维度上的差异。

存放路径在哪，怎么加载，以及最关键的，Hook 模型是什么样的。

咱们一个个拆开了聊。

先从 CLI 工具说起。

Claude Code 是最直白的。它就放在项目目录下的 `.claude/skills/` 文件夹里，每个 skill 是一个子文件夹，里面必须有一个 `skill.md` 文件。

这种设计的好处是，skill 就是项目代码的一部分。你把项目仓库 clone 下来，skill 跟着一起下来了，完全不需要额外安装。而且因为就在文件系统里，你想做监控很容易，用文件系统监听或者直接读目录结构都行。

我跟你说，我做上报工具的时候，Claude Code 这块儿是最省心的。直接遍历 `.claude/skills/` 目录，解析每个子文件夹里的 `skill.md`，提取 frontmatter 里的 name 和 description，一个 `os.walk` 就搞定了。

Codex CLI 是 OpenAI 出的官方 Coding Agent，它的 skill 系统叫 Commands，配置放在 `.codex/commands/` 目录下，每个 command 是一个 YAML 文件。

Codex 也支持全局和项目级配置，全局的在 `~/.codex/commands/`。Codex 的设计介于 Claude Code 和 Cursor 之间，它用 YAML 而不是 Markdown，保留了一些可读性，同时保持了结构化的优势。每个 command 可以定义参数、描述、还有可选的 hooks，比如在代码提交前自动执行。

然后是 Trae CLI，也叫 CoCo，字节出的命令行工具。

CoCo 的 skill 配置放在 `.coco/skills/` 目录，格式是 TOML。TOML 比 JSON 友好，比 YAML 简单，CoCo 选择这个格式是有考量的。而且 CoCo 支持 skill 市场，你可以从远程仓库拉取别人分享的 skill，本地只是一个缓存。

这种设计给上报带来了复杂性，因为 skill 可能来自远程，本地文件只是一个副本，版本可能随时变化。

接下来是 Gemini CLI，Google 的命令行工具。

Gemini CLI 的 skill 系统叫 Code Actions，配置方式很特别，它不是本地文件，而是 Google Cloud 上的资源。你在本地项目里看不到任何 skill 配置文件，所有的 skill 都定义在 Cloud Console 里，通过项目 ID 关联。

这种云原生的设计意味着，做上报工具的时候，你必须调用 Google Cloud API 才能获取 skill 列表，本地文件系统帮不上忙。

好，CLI 说完了，聊聊 IDE。

Cursor IDE 也有 skill 系统，虽然它也在推 MCP，但 Cursor 本身有一套原生的 skill 配置。它的 skill 分为两类，一类是全局 skill，一类是项目级 skill。

全局 skill 的配置存在 `~/.cursor/settings.json` 里。项目级 skill 配置在项目根目录的 `.cursor/skills.json` 里。

注意，Cursor 的 skill 配置是纯 JSON，不像 Claude Code 是 Markdown。每个 skill 有一个 name、description、prompt，还可以定义什么时候触发。这种结构化的好处是解析简单，坏处是可读性差，写 prompt 的时候不能换行、不能用 Markdown 格式，一长串文本挤在 JSON 字符串里，看着头大。

Trae IDE 是字节出的桌面端，它的 skill 系统叫智能体，配置放在 `.trae/agents/` 目录下。每个智能体是一个 JSON 文件，但 Trae 的创新点是支持多轮对话模板，你可以定义一个完整的对话流程，而不只是一个静态 prompt。

这种设计让 skill 更像是一个剧本，AI 按照预设的步骤和用户交互。对于我的上报工具来说，这意味着我需要解析更复杂的结构，因为每个 skill 可能包含多个步骤的触发点。

说到 OpenCode，开源社区孵化的 Coding Agent。

OpenCode 的 skill 系统是最激进的，它完全照搬了 Claude Code 的设计，放在 `.opencode/skills/` 目录，也是 Markdown 格式。但它加了一个功能叫 skill 继承，一个项目可以继承另一个项目的 skill，形成层级结构。

这意味着 skill 的来源不只是当前项目，可能还有父项目、全局配置。做上报的时候，你得递归解析整个继承链，才能知道最终有哪些 skill 可用。

然后是 OpenClaw，飞书出的 Coding Agent。

OpenClaw 的 skill 配置放在 `.openclaw/skills/` 目录，格式是 JSON。但 OpenClaw 的特殊之处在于它和飞书生态深度绑定，skill 可以通过飞书多维表格来管理，本地文件只是同步下来的缓存。

这种设计对上报工具很不友好，因为 skill 的真相在云端，本地文件可能过期，也可能被手动修改后还没同步。

说到这儿，你可能会发现一个问题。

CLI 和 IDE 在 skill 存放路径上有个本质区别。CLI 工具基本都是项目级配置，skill 文件就在项目里，跟着代码走。IDE 则既有全局配置又有项目级配置，而且往往把配置藏在用户目录或者云端。

这个区别对上报工具的影响很大。CLI 工具的上报可以基于文件系统监听，IDE 的上报则需要同时监控多个位置，还要处理合并逻辑。

好，路径聊完了，说说加载方式。

这块儿才是真正考验适配难度的地方。

Claude Code 的加载方式是静态扫描加热重载。启动的时候，它会递归扫描 `.claude/skills/` 目录，读取每个 `skill.md` 文件，解析 YAML frontmatter 和正文内容，然后把这些 skill 注册到内存里。而且支持热重载，你修改了 skill 文件，保存后立刻生效，不需要重启。

对于我的上报工具来说，这意味着我需要监听文件系统变化，持续更新 skill 列表。

Codex CLI 的加载也是静态扫描加热重载，和 Claude Code 类似。但它多了一个条件加载，支持在 command 里定义 when 条件，比如当文件是 Python 时才显示这个 command。这意味着即使扫描到了 skill，也不一定对所有文件都可用。

CoCo 的加载需要区分本地 skill 和远程 skill。本地的是静态扫描，远程的需要调用 API 获取列表，还要处理版本缓存。如果网络不好，CoCo 会回退到本地缓存，这时候上报的数据可能不准确。

Gemini CLI 的加载完全依赖网络，启动时调用 Google Cloud API 获取项目的 Code Actions。如果网络不通，Gemini 就完全没有 skill 可用。这种设计对上报工具来说，意味着必须处理各种网络异常。

说到 IDE 的加载方式。

Cursor IDE 的加载方式是静态扫描，但它不支持热重载。你修改了 `.cursor/skills.json`，需要重启 Cursor 才能生效。而且全局 skill 和项目级 skill 是合并加载的，如果有同名 skill，项目级的会覆盖全局的。

这种合并逻辑给上报带来了麻烦，因为你得知道最终生效的是哪个 skill。我的方案是启动时读取全局配置，然后读取项目配置，自己做合并逻辑，和 Cursor 保持一致。

Trae IDE 的加载方式更复杂，因为它支持动态智能体。除了启动时扫描 `.trae/agents/`，Trae 还可以在对话过程中动态加载新的智能体，基于上下文判断用户需要什么能力。这种动态加载给上报带来了挑战，因为 skill 列表不是固定的。

OpenCode 的加载最复杂，因为要处理 skill 继承链。它需要先加载全局 skill，然后加载父项目的 skill，最后加载当前项目的 skill，每一层都可能覆盖上一层的同名 skill。这个过程是递归的，如果项目嵌套很深，加载时间会很长。

OpenClaw 的加载需要同步飞书云端。启动时会检查本地缓存和云端的版本，如果有更新就下载。这意味着 skill 列表可能随时变化，上报工具需要持续监听同步事件。

说到加载，必须提一嘴 skill 的概念。

Claude Code 支持子 skill，你可以在一个 skill 文件夹里放多个子文件夹，每个子文件夹是一个子 skill。比如 `adk-sdd/` 下面有 `adk-sdd-plan/`、`adk-sdd-implement/` 等。它们之间可以互相调用，形成一个完整的工具链。

这种层级结构给上报带来了额外复杂度，因为你得递归扫描子目录，而且子 skill 的调用关系也得记录。

OpenCode 也支持类似的层级结构，而且它还有 skill 继承，复杂度更高。其他 Agent 像 Cursor、Trae、Codex、CoCo 都不支持子 skill，是扁平结构。

好，路径和加载都说完了，最后聊聊 Hook 模型。

这是各家差异最大的部分，也是我做上报工具时最纠结的地方。

Claude Code 的 Hook 模型，我总结为被动触发加语义匹配。你不需要显式调用 skill，Claude Code 会根据对话内容自动判断，把 skill 的 description 和 user message 做语义匹配，超过阈值就注入 prompt。

这种模型的优点是交互自然，缺点是可控性差。做上报的时候，Claude Code 没有官方 Hook 点，我只能通过解析 `.claude/logs/` 目录下的日志来间接推断 skill 是否被激活。

Codex CLI 的 Hook 模型是命令式加事件式结合。Commands 可以显式调用，也可以通过定义的 hooks 在特定事件触发，比如 onSave、onCommit。做上报的时候，你需要同时监控用户命令和编辑器事件。

CoCo 的 Hook 模型是关键词触发加场景识别，除了匹配关键词，CoCo 还会分析当前编辑场景，比如你在写测试，它就自动推荐测试相关的 skill。这种模型需要上报工具理解场景是怎么定义的。

Gemini CLI 的 Hook 模型完全是黑盒，因为 skill 在云端，触发逻辑也由服务端控制。本地只能看到最终的 AI 回复，看不到中间过程。做上报基本不可能，除非 Google 开放 API。

说到 IDE 的 Hook 模型。

Cursor IDE 的 Hook 模型是混合触发，可以通过 `/skill-name` 显式调用，也可以让 AI 自动判断。显式调用容易监控，自动判断就需要解析日志。

Trae IDE 的 Hook 模型是剧本驱动，因为支持多轮对话模板，一个 skill 可能在对话的不同阶段多次触发。比如第一步询问需求，第二步生成代码，第三步解释代码。每个步骤都是一个 Hook 点，上报工具需要追踪整个对话流程。

OpenCode 的 Hook 模型和 Claude Code 类似，也是被动触发。但由于有 skill 继承，触发的时候可能同时激活了父项目和子项目的 skill，上报工具需要记录这个层级关系。

OpenClaw 的 Hook 模型是意图识别加工作流，它会先识别用户意图，然后触发对应的工作流。工作流可能包含多个 skill 的串联调用，上报的时候你需要记录整个调用链。

说了这么多，我来帮你理一理这八家 Agent 的差异。

| Agent | 类型 | 路径 | 格式 | 加载方式 | Hook 模型 | 上报难度 |
|-------|------|------|------|----------|-----------|----------|
| Claude Code | CLI | `.claude/skills/` | Markdown | 静态扫描加热重载 | 被动触发 | 中等 |
| Codex CLI | CLI | `.codex/commands/` | YAML | 静态加热重载加条件 | 命令加事件 | 中等 |
| CoCo | CLI | `.coco/skills/` | TOML | 静态加远程 | 关键词加场景 | 高 |
| Gemini CLI | CLI | Cloud Console | - | API 获取 | 黑盒 | 不可能 |
| Cursor IDE | IDE | `~/.cursor/settings.json` | JSON | 静态扫描 | 混合触发 | 中等偏高 |
| Trae IDE | IDE | `.trae/agents/` | JSON | 静态加动态 | 剧本驱动 | 高 |
| OpenCode | IDE | `.opencode/skills/` | Markdown | 继承链加载 | 被动触发 | 中等偏高 |
| OpenClaw | IDE | `.openclaw/skills/` | JSON | 云端同步 | 意图加工作流 | 很高 |

从这张表你能看出什么？

首先，CLI 和 IDE 泾渭分明。CLI 工具普遍使用项目级配置，格式多样，Markdown、YAML、TOML 都有。IDE 则倾向于 JSON 配置，而且普遍有全局加项目级的双层结构。

其次，云原生是趋势。CoCo、Gemini CLI、OpenClaw 都不同程度地依赖云端，本地只是一个视图或缓存。这对上报工具提出了新要求，不能只盯着文件系统，还得调 API、处理网络异常。

最后，Hook 模型越来越复杂。从最简单的显式调用，到被动触发，再到剧本驱动、意图识别，Agent 越来越智能，但也越来越不透明。上报工具能拿到的信息越来越少，很多都是黑盒。

说真的，写到这儿，我突然意识到一个问题。

为什么各家在 skill 管理上差异这么大？

背后其实是产品哲学的不同，CLI 和 IDE 的目标用户不一样，设计思路自然也不同。

Claude Code 和 OpenCode 相信文件即真理。skill 就应该像代码一样，放在版本控制里，任何人都能看、能改、能复现。这种哲学决定了本地优先、透明可见的设计。

Codex CLI 相信配置的标准化。YAML 比 Markdown 结构化，比 JSON 可读性好，是工程化团队的首选。

CoCo 相信生态的力量。skill 不应该绑定在本地，而应该可以分享、可以订阅、可以版本管理。这种设计牺牲了离线可用性，换来了社区活力。

Gemini CLI 相信云服务的整合。skill 是 Google Cloud 生态的一部分，用户不需要关心配置，只需要用。

Cursor IDE 和 Trae IDE 相信配置的灵活性。JSON 虽然难写，但好解析，适合图形化配置界面。而且全局加项目级的双层结构，满足了个人习惯和团队协作的不同需求。

OpenClaw 相信飞书生态的协同。skill 可以通过多维表格管理，非技术人员也能参与配置。

这八种哲学没有对错，只是取舍不同。

但对于我这种想做通用上报工具的人来说，这就很痛苦了。因为没有标准，每家都有自己的玩法，你得为每家写一套适配代码。

我有时候觉得，AI Coding Agent 的发展，就像早期的浏览器大战。Netscape 和 IE 各有各的 DOM API，开发者苦不堪言。直到 W3C 标准化之后才好转。

skill 管理目前还没有统一标准，Markdown、JSON、YAML、TOML 混用，触发模型也五花八门。

短期内，这种碎片化还会继续。

写到这儿，我突然想起我做上报工具的初衷。

我其实只是想知道自己每天用了多少次 skill，哪些 skill 最有用，哪些可能可以删掉。一个简单的数据分析需求，却因为底层差异变得异常复杂。

但换个角度想，这种复杂度本身，就是 AI 工具现状的真实写照。

我们还在早期，还在探索，还在争吵什么是最好的方案。这种混乱是创新的代价，也是进化的必经之路。

或许几年后回头看，今天的这些差异都会显得幼稚。但在当下，我们必须面对它们，理解它们，在碎片中寻找规律。

这就是我这几天做适配的最大感受。

无论你用 Claude Code、Cursor、Trae 还是 Codex，skill 管理的本质问题是一样的，怎么让 AI 学会新能力，怎么让它在合适的时机调用这些能力，怎么让这一切对开发者透明、可控。

每家产品都有自己的答案，而我在这些答案之间，试图搭一座桥。
