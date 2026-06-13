+++
title = "Intent Recognition: 意图识别"
date = "2026-06-06T17:11:14+08:00"
tags = ["意图识别"]
+++

我们与 Agent 对话时，先将我们的问题发送给 Agent，后端会先做一系列复杂的操作，比如加载记忆、查询改写、拆分子问题等等，但其中最重要的是**意图识别**。
意图识别是整个 RAG 对话管道的核心环节，负责将用户问题路由到正确的知识分类节点，从而决定后续检索哪些知识库。意图识别的质量决定 Agent 能否正确理解你要干什么，才能返回给你想要的结果。

下面是跟 Agent 对话时，执行层会做的一些操作，意图识别在 StreamChatPipeline 中处于查询改写之后、检索之前的关键位置：

用户问题 → 加载记忆 → 查询改写/拆子问题 → 【意图识别】 → 歧义引导 → 检索 → 生成回答

```java
public void execute(StreamChatContext ctx) {
        // 加载记忆
        loadMemory(ctx);
        // 查询改写 / 拆分子问题
        rewriteQuery(ctx);
        // 意图识别
        resolveIntents(ctx);
        // 检索召回上下文
        RetrievalContext retrievalCtx = retrieve(ctx);

        streamRagResponse(ctx, retrievalCtx);
    }
```

我们重点看一下意图识别这个方法：

```java
private void resolveIntents(StreamChatContext ctx) {
    // 子问题 ——> 子意图
    List<SubQuestionIntent> subIntents = intentResolver.resolve(ctx.getRewriteResult());
    ctx.setSubIntents(subIntents);
}
```

这里基于每个子问题进行意图识别，子问题来源于 rewriteQuery 对用户提问的查询改写及拆分子问题后的结果，这里不细细展开，后面会重新写一篇专门讲这块。

继续下一步，这里使用 CompletableFuture 对子问题进行异步意图分类，最后将结果收集在一起。

```java
public List<SubQuestionIntent> resolve(RewriteResult rewriteResult) {
    // 检查是否存在子问题列表，没有则将改写后的问题作为唯一子问题
    List<String> subQuestions = CollUtil.isNotEmpty(rewriteResult.subQuestions())
            ? rewriteResult.subQuestions()
            : List.of(rewriteResult.rewrittenQuestion());
    // 使用 CompletableFuture 对多个子问题进行异步意图分类
    List<CompletableFuture<SubQuestionIntent>> tasks = subQuestions.stream()
            .map(q -> CompletableFuture.supplyAsync(
                    () -> {
                        // 调用 classifyIntents() 进行意图分类，若失败则降级为空意图
                        try {
                            return new SubQuestionIntent(q, classifyIntents(q));
                        } catch (Exception e) {
                            log.error("子问题意图分类失败，降级为空意图，question：{}", q, e);
                            return new SubQuestionIntent(q, List.of());
                        }
                    },
                    // 使用自定义线程池 intentClassifyExecutor 执行。
                    intentClassifyExecutor
            ))
            .toList();
    // 通过 join 方法获取结果
    List<SubQuestionIntent> subIntents = tasks.stream()
            .map(CompletableFuture::join)
            .toList();
    return capTotalIntents(subIntents);
}
```

真正的意图识别方法从 classifyIntents() 开始：

```java
private List<NodeScore> classifyIntents(String question) {
    // 获取每个意图节点分数
    List<NodeScore> scores = intentClassifier.classifyTargets(question);
    // 收集满足置信度分数的意图节点，同时限制最大意图数量
    return scores.stream()
            .filter(ns -> ns.getScore() >= INTENT_MIN_SCORE)
            .limit(MAX_INTENT_COUNT)
            .toList();
}
```

1. 加载意图树数据，提取所有叶子节点

2. 构建 LLM Prompt，遍历每个叶子节点，拼接 id / path / description / type / examples，渲染 intent-classifier.st 模板，将分类列表填入 {intent_list} 占位符

3. 调用 LLM，解析 JSON 响应，返回排序后的 NodeScore 列表

```java
public List<NodeScore> classifyTargets(String question) {
    // 每次都从 Redis 读取最新数据
    IntentTreeData data = loadIntentTreeData();
    
    // 根据叶子节点构建 Prompt
    String systemPrompt = buildPrompt(data.leafNodes);
    ChatRequest request = ChatRequest.builder()
            .messages(List.of(
                    ChatMessage.system(systemPrompt),
                    ChatMessage.user(question)
            ))
            .build();

    String raw = llmService.chat(request);

    JsonElement root = JsonParser.parseString(raw);

    JsonArray arr;
    if (root.isJsonArray()) {
        arr = root.getAsJsonArray();
    }

    // 收集分数
    List<NodeScore> scores = new ArrayList<>();
    for (JsonElement el : arr) {
        JsonObject obj = el.getAsJsonObject();

        String id = obj.get("id").getAsString();
        double score = obj.get("score").getAsDouble();

        IntentNode node = data.id2Node.get(id);
        scores.add(new NodeScore(node, score));
    }

    // 降序排序
    scores.sort(Comparator.comparingDouble(NodeScore::getScore).reversed());

    return scores;
}
```

现在每个子问题都有自己的一些候选意图，下一步需要更精细化：

```java
private List<SubQuestionIntent> capTotalIntents(List<SubQuestionIntent> subIntents) {
    int totalIntents = subIntents.stream()
            .mapToInt(si -> si.nodeScores().size())
            .sum();

    // 未超限，直接返回
    if (totalIntents <= MAX_INTENT_COUNT) {
        return subIntents;
    }

    // 步骤1：收集所有意图，按子问题索引分组
    List<IntentCandidate> allCandidates = collectAllCandidates(subIntents);

    // 步骤2：每个子问题保留最高分意图
    List<IntentCandidate> guaranteedIntents = selectTopIntentPerSubQuestion(allCandidates, subIntents.size());

    // 步骤3：计算剩余配额
    int remaining = MAX_INTENT_COUNT - guaranteedIntents.size();

    // 步骤4：从剩余候选中按分数选择
    List<IntentCandidate> additionalIntents = selectAdditionalIntents(allCandidates, guaranteedIntents, remaining);

    // 步骤5：合并并重建结果
    return rebuildSubIntents(subIntents, guaranteedIntents, additionalIntents);
}
```


> 参考：[Ragent](https://nageoffer.com/ragent/)