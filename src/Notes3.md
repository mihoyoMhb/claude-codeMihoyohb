# Mini Claude Code 学习笔记（逐行阅读版）

这份笔记专门做一件事：
- 逐行拆 `chatAnthropic()` 和 `chatOpenAI()`

阅读目标：
- 不是背所有细节
- 而是把“主循环每一步为什么写在这里”看明白

建议阅读方式：
1. 左边开 [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts)
2. 右边开这份笔记
3. 按行号一段一段对照着看

最先记住这句话：

```text
chatAnthropic() / chatOpenAI() 的本质都是：
把用户消息放进历史 -> while 循环调 LLM -> 如果 LLM 要工具，就执行工具 ->
把结果再放回历史 -> 继续下一轮 -> 直到 LLM 不再要工具
```

---

## 1. 阅读前先记住四个变量

在读这两个函数前，先记住下面 4 个状态：

1. `anthropicMessages` / `openaiMessages`
- 当前后端的消息历史
- 每轮都会继续往里面追加

2. `abortController`
- 控制中断
- 用户 Ctrl+C 时会影响这里

3. `lastInputTokenCount`
- 最近一轮请求用了多少 input tokens
- 后面 `checkAndCompact()` 要用

4. `contextCleared`
- plan mode 特殊信号
- 它会改变“工具结果回写到消息历史”的方式

一句话理解：
- 这两个函数本质上都在不断读写“消息历史”

---

## 2. `chatAnthropic()` 逐行阅读

函数位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L968)

### 2.1 第一句：先把用户消息放进历史

```ts
this.anthropicMessages.push({ role: "user", content: userMessage });
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L969)

怎么理解：
- 不管用户说了什么，先进入消息历史
- 后面发给 LLM 的就是这份历史

也就是说：
- `chatAnthropic()` 一开始不是先调 API
- 而是先更新上下文

### 2.2 第二段：启动 memory prefetch

```ts
let memoryPrefetch: MemoryPrefetch | null = null;
if (!this.isSubAgent) {
  const sq = this.buildSideQuery();
  if (sq) {
    memoryPrefetch = startMemoryPrefetch(...);
  }
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L972)

怎么理解：
- 主代理会尝试异步预取语义记忆
- 但这里是“先开一个后台任务”，不是立即阻塞等待

关键词：
- `non-blocking`

也就是说：
- 主循环不会因为记忆召回而卡住
- 记忆结果如果稍后准备好了，再插回上下文

### 2.3 `let firstIteration = true`

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L984)

这个变量在当前函数里其实没什么实质作用：
- 后面只看到 `firstIteration = false`
- 没看到它参与分支判断

所以阅读时可以先把它当成：
- 历史遗留 / 预留变量

真正重要的是下面的 `while (true)`。

### 2.4 核心开始：`while (true)`

```ts
while (true) {
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L986)

这是整个 Agent Loop 的核心。

每一轮都在做：
- 用当前消息历史问一次 LLM
- 看 LLM 是否需要工具
- 如果需要，就执行工具
- 把工具结果再写回历史

### 2.5 循环第一件事：检查是否中断

```ts
if (this.abortController?.signal.aborted) break;
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L987)

怎么理解：
- 用户如果中断了，就直接跳出循环

这句很早就放在循环开头，说明：
- 中断优先级很高

### 2.6 发 API 前先跑轻量压缩

```ts
this.runCompressionPipeline();
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L990)

怎么理解：
- 这一步不调模型
- 只是本地改消息历史
- 目的是减少上下文占用

可以把它理解成：
- “在发给 LLM 之前，先把能便宜压掉的历史压掉”

### 2.7 如果记忆预取已经好了，就插回当前 user 消息

这一大段代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L996)

最关键的判断是：

```ts
if (memoryPrefetch && memoryPrefetch.settled && !memoryPrefetch.consumed)
```

意思是：
- 记忆后台任务已经完成了
- 但这次循环还没消费过它

后面的核心动作是：

```ts
const injectionText = formatMemoriesForInjection(memories);
const last = this.anthropicMessages[this.anthropicMessages.length - 1];
```

然后优先把记忆追加到最后一条 `user` 消息里。

为什么不直接再 push 一条新的 `user`？
- 因为 Anthropic 这边比较注意消息交替规则
- 连续两个 user message 可能不合适

所以这里的设计是：
- 如果最后一条已经是 user，就往里面追加
- 否则才新建一条 user

### 2.8 开 spinner

```ts
if (!this.isSubAgent) startSpinner();
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1021)

怎么理解：
- 主代理在终端里要给用户一个“正在工作”的反馈
- 子代理不用，因为它的输出是收集到 buffer 里的

### 2.9 `earlyExecutions`：准备提早启动某些只读工具

```ts
const earlyExecutions = new Map<string, Promise<string>>();
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1027)

怎么理解：
- 这是 Anthropic 路径一个很特别的优化点
- 某些工具不用等整条 assistant 回复结束，就可以先启动

### 2.10 真正调 API：`callAnthropicStream(...)`

```ts
const response = await this.callAnthropicStream((block) => {
  ...
});
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1029)

这是本轮最关键的动作：
- 用当前消息历史 + tools schema 去问 Anthropic
- 流式接收模型输出

传进去的回调是做什么的？
- 一旦某个 `tool_use` block 完整结束
- 如果它属于并发安全工具，且权限允许
- 就立刻启动 `this.executeToolCall(...)`

最关键的代码是：

```ts
if (CONCURRENCY_SAFE_TOOLS.has(block.name)) {
  const perm = checkPermission(...);
  if (perm.action === "allow") {
    earlyExecutions.set(block.id, this.executeToolCall(block.name, input));
  }
}
```

怎么理解：
- Anthropic 这里是“边生成边起工具”
- 但只对安全、自动允许的工具这么做

### 2.11 API 返回后：停 spinner，记录 token 统计

```ts
if (!this.isSubAgent) stopSpinner();
this.lastApiCallTime = Date.now();
this.totalInputTokens += response.usage.input_tokens;
this.totalOutputTokens += response.usage.output_tokens;
this.lastInputTokenCount = response.usage.input_tokens;
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1038)

这里要特别记住：
- `lastInputTokenCount` 是给后面的 `checkAndCompact()` 用的
- `lastApiCallTime` 是给 `microcompact` 用的

### 2.12 从 `response.content` 里筛出所有 `tool_use`

```ts
const toolUses: Anthropic.ToolUseBlock[] = [];
for (const block of response.content) {
  if (block.type === "tool_use") {
    toolUses.push(block);
  }
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1044)

怎么理解：
- assistant 回复可能同时包含文本块和工具块
- 这里把“需要执行的动作”单独挑出来

### 2.13 先把整条 assistant 回复存回历史

```ts
this.anthropicMessages.push({
  role: "assistant",
  content: response.content,
});
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1052)

这句非常重要。

它说明：
- 不管后面会不会真的执行工具
- assistant 这一轮说了什么，先完整记进历史

也就是说：
- 上下文链永远是“先记录 assistant 的请求，再记录工具结果”

### 2.14 如果这轮没有任何工具调用，任务就结束

```ts
if (toolUses.length === 0) {
  ...
  break;
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1057)

怎么理解：
- assistant 这轮已经不需要工具了
- 那就说明它认为任务可以直接答复
- 于是循环结束

这也是主循环最核心的退出条件。

### 2.15 有工具要跑时，先检查预算

```ts
this.currentTurns++;
const budget = this.checkBudget();
if (budget.exceeded) {
  ...
  break;
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1065)

怎么理解：
- assistant 已经提出了下一步工具计划
- 但真正执行前，还要确认预算还够不够

注意这个位置很关键：
- 不是发 API 前检查
- 而是“拿到 tool call 后、执行工具前”检查

### 2.16 `toolResults`：准备收集本轮工具输出

```ts
const toolResults: Anthropic.ToolResultBlockParam[] = [];
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1072)

怎么理解：
- 本轮可能有多个工具调用
- 它们执行完后不会立即都 push 到历史
- 先收集起来，最后统一塞成一条 `user` 消息

### 2.17 遍历每个 `tool_use`

```ts
for (const toolUse of toolUses) {
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1077)

这里是本轮“工具执行阶段”的主循环。

### 2.18 工具循环的第一句：处理中断 / context break

```ts
if (contextBreak || this.abortController?.signal.aborted) break;
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1078)

怎么理解：
- 如果 plan mode 刚刚清上下文了，就别继续处理后面的工具了
- 如果用户中断了，也别继续

### 2.19 `printToolCall(...)`

```ts
printToolCall(toolUse.name, input);
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1080)

这是 UI 行为：
- 把当前要调用哪个工具打印给用户看

### 2.20 如果这个工具已经 early-start 了，直接等结果

```ts
const earlyPromise = earlyExecutions.get(toolUse.id);
if (earlyPromise) {
  const raw = await earlyPromise;
  ...
  toolResults.push(...);
  continue;
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1083)

怎么理解：
- 前面在流式过程中已经启动过了
- 现在这里只需要等它完成，然后收结果

### 2.21 没有 early execution 时，就按普通路径走权限检查

```ts
const perm = checkPermission(...);
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1093)

分三种情况：

1. `deny`
- 直接把 denied 文本当作 tool result

2. `confirm`
- 调 `confirmDangerous(...)`
- 用户拒绝就写入 “User denied this action.”

3. `allow`
- 继续执行

这里要特别注意：
- 被 deny / 被用户拒绝，也一样会变成 tool result 回到 LLM
- 这样 LLM 才知道“这步没成功”

### 2.22 真正执行工具

```ts
const raw = await this.executeToolCall(toolUse.name, input);
const res = this.persistLargeResult(toolUse.name, raw);
printToolResult(toolUse.name, res);
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1108)

怎么理解：
- `executeToolCall(...)` 负责真正干活
- `persistLargeResult(...)` 负责超大结果写磁盘
- `printToolResult(...)` 只是打印给用户看

### 2.23 `contextCleared` 的特殊分支

```ts
if (this.contextCleared) {
  this.contextCleared = false;
  this.anthropicMessages.push({ role: "user", content: res });
  contextBreak = true;
  break;
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1112)

这是个非常特殊的分支。

正常情况：
- 工具结果会被包成 `tool_result` block

这里不是。

这里做的是：
- 直接把返回文本作为一条新的 `user` 消息塞进历史

为什么？
- 因为 plan mode 的 `clear-and-execute` 刚刚清掉了旧上下文
- 现在要把“已批准的 plan”当成新的起点，而不是普通工具结果

### 2.24 普通情况：把结果收集进 `toolResults`

```ts
toolResults.push({ type: "tool_result", tool_use_id: toolUse.id, content: res });
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1118)

这才是正常回流路径。

### 2.25 工具循环结束后：统一把结果写回历史

```ts
if (!contextBreak && !this.contextCleared && toolResults.length > 0) {
  this.anthropicMessages.push({ role: "user", content: toolResults });
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1121)

怎么理解：
- 只要不是特殊的 `contextCleared` 分支
- 就把所有工具结果作为一条新的 user message 回写

所以 Anthropic 这里的主链是：

```text
user
-> assistant(tool_use)
-> user(tool_result)
-> assistant(...)
```

### 2.26 一轮尾声：清状态、压缩、进入下一轮

```ts
this.contextCleared = false;
firstIteration = false;
await this.checkAndCompact();
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1124)

怎么理解：
- 本轮该收尾的状态先收掉
- 如果上下文太大，顺手做摘要压缩
- 然后回到 `while (true)` 顶部，准备下一轮

---

## 3. `callAnthropicStream()` 逐行伴读

虽然你问的是 `chatAnthropic()`，但这两个函数是一体的，所以最好一起看。

函数位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1138)

### 3.1 `withRetry(...)`

```ts
return withRetry(async (signal) => {
```

怎么理解：
- Anthropic 请求支持重试
- 网络抖动 / 429 / 503 这类问题会走上层重试逻辑

### 3.2 先组装请求体 `createParams`

关键字段：

```ts
{
  model,
  max_tokens,
  system,
  tools,
  messages,
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1143)

这就是每轮真正发给 Anthropic 的上下文。

### 3.3 thinking 模式是怎么插进去的

```ts
if (this.thinkingMode === "adaptive") ...
else if (this.thinkingMode === "enabled") ...
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1151)

怎么理解：
- 这里不是主循环逻辑核心
- 只是按模型能力决定要不要开 thinking

### 3.4 真正创建 stream

```ts
const stream = this.anthropicClient!.messages.stream(createParams, { signal });
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1157)

注意：
- 输入一次性发出去
- 输出是流式事件

### 3.5 文本怎么实时打印出来

```ts
stream.on("text", (text) => {
  ...
  this.emitText(text);
});
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1161)

这就是为什么用户会看到：
- assistant 文本一边生成一边显示

### 3.6 `streamEvent` 是更底层的统一事件入口

```ts
stream.on("streamEvent" as any, (event: any) => {
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1173)

这里同时处理两类东西：
- thinking block
- tool_use block

### 3.7 tool_use 参数 JSON 是怎么拼出来的

最关键的三段：

1. block 开始时，建一个暂存对象
2. `input_json_delta` 时，不断追加字符串
3. block 停止时，再 `JSON.parse(...)`

关键代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1185)
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1197)

这就是 Anthropic tool_use 的“流式拼装”。

### 3.8 为什么 block stop 时要回调 `onToolBlockComplete(...)`

```ts
onToolBlockComplete({ type: "tool_use", id, name, input })
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1204)

怎么理解：
- 一旦这个工具块已经完整了
- 外层 `chatAnthropic()` 就可以考虑提前启动工具

### 3.9 最后拿 `finalMessage()`

```ts
const finalMessage = await stream.finalMessage();
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1210)

这一步才拿到最终汇总后的完整 assistant message。

### 3.10 最后一个小清理：去掉 thinking block

```ts
finalMessage.content = finalMessage.content.filter(
  (block: any) => block.type !== "thinking"
);
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1212)

怎么理解：
- thinking 可以实时展示
- 但不想把 thinking 结果长期存进历史

---

## 4. `chatOpenAI()` 逐行阅读

函数位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1223)

整体结构和 `chatAnthropic()` 很像，但差别在：
- tool call 的数据结构不同
- 执行工具前的组织方式不同
- 回写工具结果的消息格式不同

### 4.1 第一行：先把用户消息放进 `openaiMessages`

```ts
this.openaiMessages.push({ role: "user", content: userMessage });
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1224)

含义和 Anthropic 一样：
- 先更新上下文，再问模型

### 4.2 启动 memory prefetch

```ts
let memoryPrefetch: MemoryPrefetch | null = null;
...
memoryPrefetch = startMemoryPrefetch(...);
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1227)

跟 Anthropic 基本一致，唯一小差别是：
- 这里没把 `abortController?.signal` 传进去

阅读时先抓大意即可：
- 也是后台预取记忆，不阻塞主循环

### 4.3 进入 `while (true)`

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1238)

后面的主循环骨架仍然是：
- 中断检查
- 轻量压缩
- 插记忆
- 调 API
- 看 tool calls
- 执行工具
- 回写结果

### 4.4 循环开头：中断和轻量压缩

```ts
if (this.abortController?.signal.aborted) break;
this.runCompressionPipeline();
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1239)

跟 Anthropic 一致。

### 4.5 如果记忆已经准备好，就追加到最后一条 user 消息

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1245)

关键点和 Anthropic 一样：
- 记忆不是立即阻塞等待
- 是“好了再插”

不过 OpenAI 这边追加方式更简单：

```ts
last.content = (last.content || "") + "\n\n" + injectionText;
```

因为这里的 user content 结构更简单，通常按字符串处理。

### 4.6 真正调 API：`callOpenAIStream()`

```ts
const response = await this.callOpenAIStream();
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1266)

Anthropic 路径里，call 的时候顺手传了一个 tool block 完成回调；
OpenAI 路径没有这一步。

这意味着：
- OpenAI 这里不会边生成边提前启动工具
- 而是等整条 assistant 回复收齐

### 4.7 记录 token 统计

```ts
if (response.usage) {
  this.totalInputTokens += response.usage.prompt_tokens;
  this.totalOutputTokens += response.usage.completion_tokens;
  this.lastInputTokenCount = response.usage.prompt_tokens;
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1271)

跟 Anthropic 一样，这里的 `lastInputTokenCount` 以后也要给压缩逻辑用。

### 4.8 取出本轮最主要的 assistant message

```ts
const choice = response.choices?.[0];
if (!choice) break;
const message = choice.message;
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1277)

怎么理解：
- OpenAI 的返回结构外层多包了一层 `choices`
- 真正要用的是第一个 choice 的 message

### 4.9 先把 assistant message 存回历史

```ts
this.openaiMessages.push(message);
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1282)

这和 Anthropic 一样：
- assistant 先入历史
- 后面再处理工具结果

### 4.10 没有 `tool_calls` 就结束

```ts
const toolCalls = message.tool_calls;
if (!toolCalls || toolCalls.length === 0) {
  ...
  break;
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1285)

这就是 OpenAI 路径的主退出条件。

### 4.11 有工具时，先做预算检查

```ts
this.currentTurns++;
const budget = this.checkBudget();
if (budget.exceeded) {
  ...
  break;
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1294)

位置和 Anthropic 一样：
- assistant 已经提出下一步工具计划
- 但工具还没真正执行

### 4.12 Phase 1：先把所有 tool calls 都解析一遍，并做权限检查

这整段代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1301)

阅读时可以把它压成三件事：

1. 解析参数 JSON

```ts
input = JSON.parse(tc.function.arguments)
```

2. `printToolCall(...)`

3. `checkPermission(...)`

最后把结果收集成一个 `oaiChecked[]`

```ts
{ tc, fnName, input, allowed, result? }
```

怎么理解：
- OpenAI 这里是“先把所有工具请求过一遍安检”
- 还没真正执行

### 4.13 为什么 OpenAI 要先统一检查，再统一执行

因为这里可能涉及：
- 用户确认
- deny / allow 分支
- 是否能并行

所以先把每个工具变成“已检查对象”，后面才好统一调度。

可以把 `oaiChecked[]` 理解成：
- “assistant 给出的工具计划，经过 Agent 安检后的版本”

### 4.14 Phase 2：把连续的安全工具分成 batch

```ts
type OAIBatch = { concurrent: boolean; items: OAIChecked[] };
const oaiBatches: OAIBatch[] = [];
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1330)

这段的核心判断是：

```ts
const safe = ct.allowed && CONCURRENCY_SAFE_TOOLS.has(ct.fnName);
```

意思是：
- 只有已经允许执行、而且属于并发安全工具
- 才能进并发 batch

### 4.15 batch 执行：并发安全工具一起跑

```ts
if (batch.concurrent) {
  const results = await Promise.all(...)
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1346)

这里要注意：
- OpenAI 这里的并发是“assistant 回复已经收齐之后”的并发
- 跟 Anthropic 那种“边流边提早启动”不是一回事

### 4.16 串行 batch：逐个处理普通工具

```ts
for (const ct of batch.items) {
  ...
  const raw = await this.executeToolCall(ct.fnName, ct.input);
  ...
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1358)

这里会处理三种情况：

1. 本来就不允许执行
- 直接把拒绝文本写成 tool message

2. 允许执行
- 调 `executeToolCall(...)`
- 处理大结果
- 打印工具结果

3. `contextCleared`
- 走特殊分支

### 4.17 OpenAI 里的 `contextCleared`

```ts
if (this.contextCleared) {
  this.contextCleared = false;
  this.openaiMessages.push({ role: "user", content: res });
  oaiContextBreak = true;
  break;
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1368)

含义和 Anthropic 一样：
- 不按普通 `role: "tool"` 回写
- 直接把结果作为新的 user 消息起点

### 4.18 普通情况：把工具结果写成 `role: "tool"`

```ts
this.openaiMessages.push({ role: "tool", tool_call_id: ct.tc.id, content: res });
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1374)

这就是 OpenAI 路径的标准回流格式。

### 4.19 一轮尾声：清状态、做摘要压缩、回到下一轮

```ts
this.contextCleared = false;
await this.checkAndCompact();
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1379)

这一步和 Anthropic 一样：
- 本轮收尾
- 然后重新回到 `while (true)` 顶部

---

## 5. `callOpenAIStream()` 逐行伴读

函数位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1384)

### 5.1 也是 `withRetry(...)`

```ts
return withRetry(async (signal) => {
```

跟 Anthropic 一样：
- OpenAI-compatible 请求也支持上层重试

### 5.2 创建流式请求

```ts
const stream = await this.openaiClient!.chat.completions.create({
  model: this.model,
  max_tokens: 16384,
  tools: toOpenAITools(getActiveToolDefinitions(this.tools)),
  messages: this.openaiMessages,
  stream: true,
  stream_options: { include_usage: true },
}, { signal });
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1386)

这就是每轮真正发给 OpenAI-compatible 后端的请求体。

### 5.3 准备几个累加器

```ts
let content = "";
const toolCalls = new Map(...);
let finishReason = "";
let usage = ...;
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1395)

怎么理解：
- 因为流式返回的是碎片
- 所以要自己在本地累加成完整结果

### 5.4 `for await (const chunk of stream)`

```ts
for await (const chunk of stream) {
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1402)

这就是 OpenAI 路径真正的流式消费方式。

跟 Anthropic 的事件监听不同：
- Anthropic 更像 `on(...)`
- OpenAI 这里是异步迭代器

### 5.5 usage 是在最终 chunk 里拿到的

```ts
if (chunk.usage) {
  usage = ...
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1406)

怎么理解：
- token 统计不是一开始就有
- 通常在最后一个 chunk 才拿到

### 5.6 普通文本怎么流式展示

```ts
if (delta.content) {
  ...
  this.emitText(delta.content);
  content += delta.content;
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1415)

这就是为什么用户能看到 assistant 文本一边生成一边打印。

### 5.7 `delta.tool_calls` 怎么拼成完整函数调用

```ts
if (delta.tool_calls) {
  for (const tc of delta.tool_calls) {
    const existing = toolCalls.get(tc.index);
    if (existing) {
      if (tc.function?.arguments) existing.arguments += tc.function.arguments;
    } else {
      toolCalls.set(tc.index, ...);
    }
  }
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1422)

这是 OpenAI tool call 的关键拼装逻辑。

怎么理解：
- 同一个函数调用会拆成多段 delta
- 所以这里按 `index` 累加
- 特别是 `arguments` 会被一点点拼接起来

### 5.8 最后根据 `toolCalls` 重建 `assembledToolCalls`

```ts
const assembledToolCalls = toolCalls.size > 0
  ? ...
  : undefined;
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1444)

这一步做的事是：
- 把前面散落的碎片
- 重新变成标准 `message.tool_calls[]`

### 5.9 最终自己手工拼一个 `ChatCompletion`

```ts
return {
  ...
  choices: [
    {
      message: {
        role: "assistant",
        content,
        tool_calls: assembledToolCalls,
      },
    },
  ],
  usage,
}
```

代码位置：
- [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts#L1454)

怎么理解：
- OpenAI 流接口给的是一堆碎块
- `callOpenAIStream()` 的职责，就是把这些碎块重新组装成“像普通非流式返回一样”的结构

---

## 6. 两个函数一起看时，最该抓住的差异

### 6.1 相同点

它们都在做：

1. push 用户消息
2. 启动 memory prefetch
3. `while (true)` 循环
4. 发 LLM 请求
5. 存 assistant 消息
6. 没工具就结束
7. 有工具就检查预算和权限
8. 执行工具
9. 把结果写回消息历史
10. 必要时做摘要压缩

### 6.2 不同点

Anthropic：
- assistant 的内容是 content blocks
- 工具调用是 `tool_use`
- 工具结果是下一条 user message 里的 `tool_result`
- 支持边流边 early execution

OpenAI：
- assistant 的内容是 `message.tool_calls`
- 工具结果是独立的 `role: "tool"` 消息
- 先统一解析权限，再 batch 执行
- 没有 Anthropic 那种“边流边起工具”

---

## 7. 一页速记：回头只看这一页也行

如果你后面回顾时只想看最短版，就记下面这些：

1. `chatAnthropic()` / `chatOpenAI()` 一开始都会先把 user message 放进历史。
2. 真正核心是一个 `while (true)`。
3. 每轮循环开头先做：
- 中断检查
- `runCompressionPipeline()`
- 如果 memory prefetch 已完成，就把记忆插回 user 消息
4. 然后调用各自的流式 API：
- Anthropic：`callAnthropicStream()`
- OpenAI：`callOpenAIStream()`
5. assistant 回复先存回历史。
6. 如果没有工具调用，直接 `break`。
7. 如果有工具调用，先做预算检查。
8. 然后对每个工具：
- 权限检查
- `executeToolCall(...)`
- 处理大结果
- 把结果回写进消息历史
9. Anthropic 回写格式：
- `assistant(tool_use)` -> `user(tool_result)`
10. OpenAI 回写格式：
- `assistant(tool_calls)` -> `tool`
11. `contextCleared` 会改变结果回写方式，把结果直接当成新的 `user` 消息。
12. 一轮结束后 `checkAndCompact()`，然后进入下一轮。
13. 整个循环结束的真正条件是：
- assistant 不再请求任何工具

---

## 8. 补充：消息数组是怎么组装和增长的

前面我们一直在说“把消息历史交给 LLM，再把结果写回历史”。  
如果你想更具体一点地理解，那么可以直接盯住 Python 版 `mini_claude` 里的 `self._anthropic_messages`。

最关键的结论只有一句：

```text
消息数组的增长，主要发生在 _chat_anthropic()；
_call_anthropic_stream() 负责产出本轮 response，
然后 _chat_anthropic() 再把 assistant 和 tool_result 追加回历史。
```

### 8.1 历史数组从哪里开始

初始化位置：
- [agent.py](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/python/mini_claude/agent.py:219)

```python
self._anthropic_messages: list[dict] = []
```

这说明：
- 它本质上就是一个普通 Python list
- 后面每一轮都是往这个 list 继续 `append(...)`

### 8.2 每轮开始：先 append 一条 user

位置：
- [agent.py](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/python/mini_claude/agent.py:837)

```python
self._anthropic_messages.append({"role": "user", "content": user_message})
```

如果用户说：

```python
"帮我修复 bug"
```

那一开始历史会变成：

```python
[
  {"role": "user", "content": "帮我修复 bug"}
]
```

### 8.3 发请求时：把整份历史原样作为 `messages` 送进模型

位置：
- [agent.py](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/python/mini_claude/agent.py:995)

```python
"messages": self._anthropic_messages,
```

这一步非常关键，因为它说明：
- 模型每一轮看到的不是“只看当前一句”
- 而是“看当前整个消息历史”

所以 `self._anthropic_messages` 会越来越重要；它就是 Agent 的工作记忆。

### 8.4 本轮 assistant 回复回来后：先 append assistant

位置：
- [agent.py](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/python/mini_claude/agent.py:907)

```python
self._anthropic_messages.append({
    "role": "assistant",
    "content": [self._block_to_dict(b) for b in response.content],
})
```

这里做了两件事：

1. 把本轮 assistant 回复写回历史
2. 把 SDK 的 content block 转成普通 dict，再存进 list

转换函数在这里：
- [agent.py](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/python/mini_claude/agent.py:973)

也就是说，像：
- `text`
- `tool_use`

这些 block 都是在这里被装进消息数组的。

如果本轮模型回复的是：

```python
[
  {"type": "text", "text": "我先读一下文件。"},
  {"type": "tool_use", "id": "toolu_1", "name": "read_file", "input": {"file_path": "agent.py"}}
]
```

那么历史会长成：

```python
[
  {"role": "user", "content": "帮我修复 bug"},
  {
    "role": "assistant",
    "content": [
      {"type": "text", "text": "我先读一下文件。"},
      {"type": "tool_use", "id": "toolu_1", "name": "read_file", "input": {"file_path": "agent.py"}}
    ]
  }
]
```

### 8.5 工具跑完后：把结果再 append 成一条 user

位置：
- 收集 `tool_results`： [agent.py](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/python/mini_claude/agent.py:925)
- 真正 append： [agent.py](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/python/mini_claude/agent.py:966)

```python
self._anthropic_messages.append({"role": "user", "content": tool_results})
```

这就是 Anthropic 路径最重要的格式特点：
- assistant 里放 `tool_use`
- 下一条 user 里放 `tool_result`

如果 `read_file` 返回了文件内容，那么历史会继续长成：

```python
[
  {"role": "user", "content": "帮我修复 bug"},
  {
    "role": "assistant",
    "content": [
      {"type": "text", "text": "我先读一下文件。"},
      {"type": "tool_use", "id": "toolu_1", "name": "read_file", "input": {"file_path": "agent.py"}}
    ]
  },
  {
    "role": "user",
    "content": [
      {"type": "tool_result", "tool_use_id": "toolu_1", "content": "文件内容..."}
    ]
  }
]
```

然后 `while True` 不会结束，会重新回到循环顶部，再把这 3 条消息一起发给模型。

### 8.6 为什么会“越聊越长”

因为每轮都在重复这三个 append 动作：

1. 新 user 输入进历史
2. assistant 回复进历史
3. 工具结果进历史

所以你可以把一轮一轮的增长理解成：

```python
第 1 轮:
[
  {"role": "user", "content": "帮我修复 bug"},
  {"role": "assistant", "content": [text, tool_use(read_file)]},
  {"role": "user", "content": [tool_result("文件内容...")]}
]

第 2 轮:
[
  ...前 3 条,
  {"role": "assistant", "content": [text, tool_use(edit_file)]},
  {"role": "user", "content": [tool_result("编辑成功")]}
]

第 3 轮:
[
  ...前 5 条,
  {"role": "assistant", "content": [text("已修复!")]}
]
```

这里第 3 轮的关键是：
- assistant 仍然会先被 append 进历史
- 但如果没有任何 `tool_use`，循环就结束

对应代码：
- assistant append： [agent.py](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/python/mini_claude/agent.py:907)
- 无工具就 `break`： [agent.py](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/python/mini_claude/agent.py:912)

```python
if not tool_uses:
    break
```

所以最后一轮只有 assistant 文本，没有新的 `tool_result` user 消息。

### 8.7 职责怎么分工

这两个函数最好分开记：

- `_call_anthropic_stream()`：
  - 负责和 Anthropic 流式通信
  - 负责拿到本轮 `response`
  - 负责在 streaming 过程中把 `tool_use` 的输入 JSON 拼起来
  - 负责在条件满足时提前触发工具执行

- `_chat_anthropic()`：
  - 负责维护 `self._anthropic_messages`
  - 负责把 user / assistant / tool_result 一轮轮 append 回历史
  - 负责决定是否继续下一轮，还是 `break`

一句话记忆：
- `_call_anthropic_stream()` 是“拿本轮结果”
- `_chat_anthropic()` 是“养大整份消息历史”

---

## 9. 一句话总结

把这两个函数读懂之后，你就会发现：

```text
所谓 Agent Loop，其实就是：
不断把“当前消息历史”交给 LLM，
让 LLM 决定下一步是否要调用工具，
再把工具执行结果重新塞回消息历史，
直到 LLM 认为任务完成。
```
