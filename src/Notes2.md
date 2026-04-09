# Mini Claude Code 学习笔记（续：Agent Loop 复习版）

这份笔记的目标只有一个：
- 帮你快速看懂 `agent.ts` 后半段最核心的 Agent 循环

重点盯住这条主线：

```text
用户输入
-> 调 LLM
-> 看 LLM 有没有要调工具
-> 如果要调工具，就执行工具
-> 把工具结果塞回消息历史
-> 再调 LLM
-> 重复
-> 直到 LLM 不再请求工具，改为直接给最终答复
```

一句话理解：
- 这个项目的本质，就是一个 `while (true)` 循环，不断做“LLM -> 工具 -> LLM -> 工具”

这份续记会额外讲清楚两件事：
- Anthropic / OpenAI-compatible 两条 API 调用链分别怎么写
- 流式输出、tool call 格式、tool result 回流格式分别是什么

---

## 1. 先背这段：Agent Loop 最短版

最值得先背下来的骨架：

```ts
async chat(userMessage) {
  选择后端;
  进入 chatAnthropic() 或 chatOpenAI();
}

async chatXXX(userMessage) {
  把用户消息 push 进历史;

  while (true) {
    先做本地压缩 / 状态处理;
    调一次 LLM API;
    把 assistant 回复 push 进历史;

    if (assistant 没有 tool call) {
      break; // 任务完成
    }

    检查预算;
    执行工具;
    把工具结果 push 回历史;
    检查是否需要摘要压缩;
  }
}
```

任务什么时候结束：
- 不是“循环跑了多少次”
- 而是“某一轮 assistant 回复里不再包含 tool call”

也就是：
- LLM 认为不需要再调工具了
- 它直接给出最终自然语言答复
- 主循环 `break`

---

## 2. 从 CLI 到 Agent：循环是怎么被启动的

REPL 模式下：
- `cli.ts` 的 `askQuestion()` 读入用户输入
- 如果不是 `/clear`、`/plan` 这类命令
- 就执行：

```ts
await agent.chat(input);
```

代码位置：
- `cli.ts:177`
- `cli.ts:277`

one-shot 模式下：
- `main()` 最后本质上也是调用 `await agent.chat(prompt)`

所以真正进入 Agent 的总入口就是：
- `agent.chat(...)`

`chat()` 自己并不实现循环，它只做三件事：

1. 主代理首次对话时，懒加载 MCP 工具
2. 创建 `abortController`
3. 根据后端进入：
- `chatAnthropic(userMessage)`
- `chatOpenAI(userMessage)`

代码位置：
- `agent.ts:343`

一句话理解：
- `chat()` 是总入口
- 真正的循环在 `chatAnthropic()` / `chatOpenAI()` 里面

---

## 3. 通用主线：不管哪个平台，循环都在做什么

不区分后端，主循环都可以压成这 8 步：

1. 把当前用户输入放进消息历史
2. 循环开始前，先做一些“便宜的准备工作”
- `runCompressionPipeline()`
- memory prefetch 注入
- `abortController` 检查
3. 发起一次 LLM API 调用
4. 把 assistant 回复存回消息历史
5. 如果 assistant 没有 tool call，直接结束
6. 如果 assistant 有 tool call，先做预算和权限检查
7. 真正执行工具
8. 把工具结果再塞回消息历史，进入下一轮

把它再压成一句话：

```text
每一轮都是“拿当前上下文问 LLM，LLM 给出下一步动作，再把动作结果写回上下文”
```

这就是 Agent Loop 的核心。

---

## 4. 一个最重要的 while 循环：你真正要理解的是什么

你现在最该建立的心智模型，不是每个 API 细节，而是下面这个闭环：

```text
messages（当前上下文）
-> 发给 LLM
-> LLM 返回：
   - 普通文本，或者
   - 普通文本 + tool call
-> 如果有 tool call：
   - 执行工具
   - 得到工具结果
   - 把工具结果追加回 messages
-> 用新的 messages 再问下一轮 LLM
```

注意：
- LLM 并不会“自己直接执行 shell / 写文件 / 读文件”
- 它只会“请求调用某个工具”
- 真正执行工具的是本地 Node.js 代码

所以这个系统里，LLM 的角色更像：
- 决策器
- 编排器
- 下一步动作的选择者

而工具层的角色更像：
- 执行器
- 真实碰文件系统、shell、网络的人

---

## 5. 工具 schema 是怎么给到 LLM 的

所有内建工具都定义在：
- `tools.ts:28`

比如：
- `read_file`
- `write_file`
- `edit_file`
- `list_files`
- `grep_search`
- `run_shell`
- `skill`
- `agent`
- `enter_plan_mode`
- `exit_plan_mode`

每个工具定义里都有：
- `name`
- `description`
- `input_schema`

这就相当于给 LLM 的“函数签名”。

真正送给模型的不是原始 `toolDefinitions`，而是：

```ts
getActiveToolDefinitions(this.tools)
```

代码位置：
- `tools.ts:259`

它的作用：
- 过滤掉 아직没激活的 deferred tools
- 只把当前允许暴露的工具 schema 发给模型

---

## 6. 两大平台 API 调用对照：最值得记住的差别

| 项目 | Anthropic | OpenAI-compatible |
| --- | --- | --- |
| system prompt 放哪 | 单独的 `system` 参数 | `messages[0]` 的 `role: "system"` |
| tools 格式 | 直接传 Anthropic 风格工具定义 | 先转成 `type: "function"` |
| 请求入口 | `messages.stream(...)` | `chat.completions.create(..., stream: true)` |
| 文本流式输出 | `stream.on("text")` | `delta.content` |
| tool call 流式输出 | `content_block_start/delta/stop` | `delta.tool_calls` |
| assistant 里工具格式 | `tool_use` content block | `message.tool_calls[]` |
| 工具结果回流格式 | 下一条 `user` 消息里的 `tool_result[]` | 若干条 `role: "tool"` 消息 |
| system prompt 更新方式 | 改 `this.systemPrompt` 即可 | 还要同步改 `openaiMessages[0]` |

先记一句最关键的话：
- Anthropic 是“assistant 里带 block，再由下一条 user 带 `tool_result` 回去”
- OpenAI 是“assistant 里带 `tool_calls`，再追加 `role: "tool"` 消息回去”

---

## 7. Anthropic 链路：API 调用、流式输出、tool_use 拼装

### 7.1 真实请求长什么样

`callAnthropicStream()` 最关键的请求参数是：

```ts
{
  model: this.model,
  system: this.systemPrompt,
  tools: getActiveToolDefinitions(this.tools),
  messages: this.anthropicMessages,
}
```

代码位置：
- `agent.ts:1138`

这说明 Anthropic 这里：
- system prompt 不在 messages 里
- tools 直接用 Anthropic 风格 schema
- 当前上下文来自 `this.anthropicMessages`

### 7.2 这里真正流式的是什么

这里“流式”的是输出，不是输入。

输入：
- 一次性发出整个请求体

输出：
- 通过 `stream` 持续收到事件

最重要的流式事件有两类：

1. 文本输出
- `stream.on("text", ...)`

2. 更底层的 `streamEvent`
- `content_block_start`
- `content_block_delta`
- `content_block_stop`

代码位置：
- `agent.ts:1157`
- `agent.ts:1173`

### 7.3 Anthropic 的 tool call 为什么要“拼”

因为工具调用的参数 JSON 不是一下子给全的。

代码里会：
- 先在 `content_block_start` 时识别出这是一个 `tool_use`
- 然后在 `input_json_delta` 里不断拼 `partial_json`
- 到 `content_block_stop` 时再 `JSON.parse(...)`

代码位置：
- `agent.ts:1185`

所以 Anthropic 这里的工具调用不是“直接拿到完整对象”：
- 而是“流式积累 JSON -> block 结束时组装成完整 tool_use”

### 7.4 Anthropic 的 assistant 消息长什么样

Anthropic 一轮完整回复存回历史时，是：

```ts
this.anthropicMessages.push({
  role: "assistant",
  content: response.content,
});
```

代码位置：
- `agent.ts:1052`

也就是说 assistant 不是简单一段字符串，而是一个 content blocks 数组：
- 可能有文本块
- 也可能有 `tool_use` 块

### 7.5 Anthropic 的工具结果怎么回去

执行完工具后，不是直接 push 一条普通文本。

而是先收集：

```ts
{ type: "tool_result", tool_use_id, content: res }
```

然后统一塞进一条新的 `user` 消息：

```ts
this.anthropicMessages.push({ role: "user", content: toolResults });
```

代码位置：
- `agent.ts:1072`
- `agent.ts:1121`

所以 Anthropic 这一轮的来回格式是：

```text
user -> assistant(tool_use) -> user(tool_result) -> assistant(...)
```

### 7.6 Anthropic 的一个额外优化：early execution

Anthropic 这里有一个很值得注意的优化：
- 工具块一结束，不一定等整条回复都结束，某些工具就可以提前开始执行

触发条件：
- 工具名属于 `CONCURRENCY_SAFE_TOOLS`
- `checkPermission(...)` 返回 `allow`

代码位置：
- `agent.ts:1029`

所以 Anthropic 这条链比 OpenAI 多了一个特点：
- 某些安全的只读工具，可以和模型输出并行推进

---

## 8. OpenAI-compatible 链路：API 调用、流式输出、tool_calls 拼装

### 8.1 真实请求长什么样

`callOpenAIStream()` 的请求核心是：

```ts
{
  model: this.model,
  tools: toOpenAITools(getActiveToolDefinitions(this.tools)),
  messages: this.openaiMessages,
  stream: true,
  stream_options: { include_usage: true },
}
```

代码位置：
- `agent.ts:1384`

最重要的差别：
- system prompt 已经在 `openaiMessages` 里
- tools 必须先走 `toOpenAITools(...)`

### 8.2 `toOpenAITools(...)` 到底做了什么

它做的事很简单：

```ts
{
  type: "function",
  function: {
    name,
    description,
    parameters: input_schema
  }
}
```

代码位置：
- `agent.ts:106`

所以 OpenAI-compatible 这边的 function calling，本质上是把内部工具定义翻译成 OpenAI 的函数调用格式。

### 8.3 OpenAI 这里真正流式的是什么

和 Anthropic 一样：
- 输入不是流式
- 输出才是流式

流式输出里最重要的两类 delta：

1. `delta.content`
- 普通文本片段

2. `delta.tool_calls`
- 工具调用片段

代码位置：
- `agent.ts:1402`
- `agent.ts:1422`

### 8.4 OpenAI 的 tool call 为什么也要“拼”

因为 OpenAI 流里：
- `tool_calls` 会拆成很多小片段
- 尤其是 `function.arguments` 会一点点流出来

所以代码用了一个：

```ts
Map<number, { id, name, arguments }>
```

来按 `index` 逐步拼装完整调用。

代码位置：
- `agent.ts:1398`

最后再重建出一个完整的 assistant message：

```ts
{
  role: "assistant",
  content,
  tool_calls: assembledToolCalls,
}
```

代码位置：
- `agent.ts:1443`

也就是说：
- OpenAI 流里拿到的是碎片
- Agent 自己负责把碎片拼回一个完整 `ChatCompletion`

### 8.5 OpenAI 的工具结果怎么回去

OpenAI 这里不是 `tool_result block`。

它会对每个工具结果分别 push 一条：

```ts
{ role: "tool", tool_call_id, content: res }
```

代码位置：
- `agent.ts:1356`
- `agent.ts:1374`

所以 OpenAI 这一轮的来回格式是：

```text
user -> assistant(tool_calls) -> tool -> assistant(...)
```

### 8.6 OpenAI 的一个额外特点：先检查，再分批执行

OpenAI 路径里，工具执行分两段：

1. 先串行解析并做权限检查
2. 再把连续的安全工具分成 batch

其中只读、安全工具可以并行：
- `read_file`
- `list_files`
- `grep_search`
- `web_fetch`

代码位置：
- `agent.ts:1301`
- `agent.ts:1330`

所以 OpenAI 这里的特点是：
- 不像 Anthropic 那样“边流边提前启动”
- 而是“assistant 回复收齐后，先统一检查，再并行执行安全 batch”

---

## 9. 真正的 while 循环：Anthropic / OpenAI 共通骨架

如果你只想抓最核心的主循环，可以直接记这个版本：

```ts
while (true) {
  if (abort) break;

  this.runCompressionPipeline();
  处理 memory 注入;

  const response = await callLLM();
  记录 tokens;
  把 assistant 回复 push 进历史;

  if (没有 tool call) {
    break;
  }

  this.currentTurns++;
  if (checkBudget().exceeded) {
    break;
  }

  for (每个 tool call) {
    做权限检查;
    const raw = await this.executeToolCall(...);
    const res = this.persistLargeResult(...);
    把 res 回写到消息历史;
  }

  await this.checkAndCompact();
}
```

你只要把这段搞懂，后面很多实现细节都会顺下来。

---

## 10. 工具到底怎么执行：`executeToolCall()` 是总分发器

主循环里真正执行工具时，不是直接进 `executeTool()`，而是先统一走：

```ts
await this.executeToolCall(name, input)
```

代码位置：
- `agent.ts:758`

它会按工具类型做分发：

| 工具类型 | 去哪执行 |
| --- | --- |
| `enter_plan_mode` / `exit_plan_mode` | `executePlanModeTool(...)` |
| `agent` | `executeAgentTool(...)` |
| `skill` | `executeSkillTool(...)` |
| MCP 工具 | `mcpManager.callTool(...)` |
| 普通工具 | `executeTool(name, input, this.readFileState)` |

所以：
- `executeToolCall()` 是 Agent 内部的统一分发入口
- `executeTool()` 只是更底层的普通工具执行器

---

## 11. 普通工具怎么落到本地 Node.js 代码

普通工具最后都会进：

```ts
executeTool(name, input, this.readFileState)
```

代码位置：
- `agent.ts:767`
- `tools.ts:723`

`executeTool()` 里是一个 `switch`：
- `read_file` -> `readFile(...)`
- `write_file` -> `writeFile(...)`
- `edit_file` -> `editFile(...)`
- `list_files` -> `listFiles(...)`
- `grep_search` -> `grepSearch(...)`
- `run_shell` -> `runShell(...)`

它的本质很简单：
- LLM 请求调用某个工具
- Agent 把这个请求翻译成本地 TypeScript 函数调用

### 11.1 `run_shell` 是怎么真正碰终端的

`run_shell` 最终会走到：

```ts
runShell(input)
  -> execSync(input.command, { shell: ... })
```

代码位置：
- `tools.ts:484`

一句话理解：
- 模型本身不会执行 shell
- 真正执行 shell 的，是本地 Node.js 进程里的 `execSync(...)`

### 11.2 为什么 `write_file` / `edit_file` 还要带 `readFileState`

因为普通工具执行时还会顺带做一个保护：
- 读过文件才能写
- 文件被外部改过就提示重新读取

代码位置：
- `tools.ts:723`

这也是为什么普通工具走的是：

```ts
executeTool(name, input, this.readFileState)
```

而不是简单两参数调用。

---

## 12. 权限检查插在哪里：为什么不是 LLM 一叫工具就立刻执行

真正执行前，一定会先经过：

```ts
checkPermission(toolName, input, this.permissionMode, this.planFilePath)
```

代码位置：
- `tools.ts:622`

这一步可能得到三种结果：
- `allow`
- `deny`
- `confirm`

也就是说：
- LLM 只负责“请求调用工具”
- Agent 还要先判断“这次工具调用允不允许执行”

### 12.1 shell 的权限规则尤其要记

对 `run_shell` 来说最关键的几点是：

1. `plan` 模式下直接禁止 shell
2. 危险命令会触发确认
3. `dontAsk` 模式下，需要确认的危险命令会直接 deny

代码位置：
- `tools.ts:644`
- `tools.ts:672`

---

## 13. 工具结果是怎么“喂回 LLM”的

这是理解 Agent Loop 的第二个关键点。

很多人第一次看会误以为：
- 工具执行完，只是打印给用户看就结束了

其实不是。

真正关键的是：
- 工具执行结果必须重新写回消息历史
- 下一轮 LLM 才能“知道刚才工具做了什么”

### 13.1 Anthropic 的回流方式

Anthropic 先把 assistant 回复存回去：

```ts
{ role: "assistant", content: response.content }
```

然后把本轮工具结果收集成：

```ts
[{ type: "tool_result", tool_use_id, content: res }, ...]
```

再整体塞进一条新的 `user` 消息。

所以 Anthropic 的上下文链是：

```text
user
-> assistant(tool_use)
-> user(tool_result)
-> assistant(...)
```

### 13.2 OpenAI 的回流方式

OpenAI 先把 assistant message 存回去：

```ts
{ role: "assistant", content, tool_calls }
```

然后每个工具结果都各自 push 一条：

```ts
{ role: "tool", tool_call_id, content: res }
```

所以 OpenAI 的上下文链是：

```text
user
-> assistant(tool_calls)
-> tool
-> assistant(...)
```

### 13.3 这一步为什么是 Agent Loop 的核心

因为如果不把工具结果重新塞回消息历史：
- LLM 下一轮根本不知道工具执行了什么
- 它也就没法决定下一步

所以：
- 工具执行本身不是终点
- “把结果写回上下文”才是闭环成立的关键

---

## 14. 状态如何在循环里生效

这一段最容易看散，所以直接按“插入点”记。

### 14.1 `runCompressionPipeline()`

位置：
- 每轮发 API 之前

作用：
- 本地轻量压缩
- 尽量减少上下文占用

代码位置：
- `agent.ts:989`
- `agent.ts:1241`

### 14.2 `checkBudget()`

位置：
- assistant 已经返回 tool call 之后
- 真正开始执行工具之前

作用：
- 超过成本预算或轮数预算，就直接停

代码位置：
- `agent.ts:1065`
- `agent.ts:1294`

注意：
- 如果 assistant 这一轮根本没有 tool call，而是直接结束答复
- 那这一轮不会再进工具阶段，也就不会走这里

### 14.3 `checkAndCompact()`

位置：
- 每轮工具结果回写结束后
- 准备进入下一轮前

作用：
- 如果 `lastInputTokenCount` 太高，就触发摘要压缩

代码位置：
- `agent.ts:476`
- `agent.ts:1128`
- `agent.ts:1380`

### 14.4 `contextCleared`

位置：
- plan mode 的 `clear-and-execute` 分支里被设为 `true`

作用：
- 告诉主循环：
- “这次不要按普通 tool result 回写”
- “把返回文本当成新的 user 起点塞回去”

Anthropic 生效点：
- `agent.ts:1112`

OpenAI 生效点：
- `agent.ts:1368`

一句话理解：
- `contextCleared` 改变的是“结果回流到上下文的方式”

### 14.5 `abortController`

位置：
- 每轮循环开头
- 某些工具处理阶段中间

作用：
- 用户按 Ctrl+C 时，中止当前循环

代码位置：
- `agent.ts:357`
- `agent.ts:987`
- `agent.ts:1239`

---

## 15. 子代理结果怎么回流到主代理

子代理的关键不是“又有一套不同逻辑”，而是：
- 它本质上还是一个新的 `Agent`
- 跑完后返回一段字符串
- 父代理再把这段字符串当成普通工具结果喂回上下文

### 15.1 子代理是怎么启动的

父代理遇到 `agent` 工具时，会走：

```ts
return this.executeAgentTool(input);
```

代码位置：
- `agent.ts:763`

`executeAgentTool()` 里会：

1. 读 `type` / `description` / `prompt`
2. `getSubAgentConfig(type)` 拿子代理专用 system prompt 和工具集
3. `new Agent(...)`
4. `await subAgent.runOnce(prompt)`
5. 把 token 消耗累加回父代理
6. 返回 `result.text`

代码位置：
- `agent.ts:933`

### 15.2 为什么子代理能“返回文本”

因为 `runOnce()` 会先打开 `outputBuffer`：

```ts
this.outputBuffer = [];
await this.chat(prompt);
const text = this.outputBuffer.join("");
```

而 `emitText(...)` 看到 `outputBuffer` 存在时，不会直接打印，而是把内容 push 进去。

代码位置：
- `agent.ts:375`
- `agent.ts:393`

所以对子代理来说：
- 模型输出先被收集成字符串
- 最后作为 `runOnce()` 的 `text` 返回给父代理

### 15.3 子代理结果回到父代理后，发生了什么

父代理拿到的是一个普通字符串：

```ts
return result.text || "(Sub-agent produced no output)";
```

于是从主循环视角看：
- 这就只是 `agent` 这个工具的执行结果

接下来它会像其它工具结果一样：
- Anthropic：被包成 `tool_result`
- OpenAI：被包成 `role: "tool"` 消息

所以：
- 子代理结果没有什么神秘的特殊协议
- 它最终也只是“一段工具输出文本”

---

## 16. 一个完整例子：写 `hello.py` 并运行

假设用户说：

```text
请在当前目录写一个 hello.py，内容是打印 Hello Mini Claude，然后运行它
```

这个例子最适合拿来理解主循环。

### 第 1 步：进入 Agent

REPL 读到这句话后，会执行：

```ts
await agent.chat(input);
```

代码位置：
- `cli.ts:277`

### 第 2 步：第一轮调用 LLM

这一轮请求里，LLM 能看到：
- 用户请求
- system prompt
- 可用工具：
- `write_file`
- `run_shell`
- 等等

于是一个很常见的第一轮决策是：
- 先请求调用 `write_file`

### 第 3 步：Agent 执行 `write_file`

主循环看到 tool call 后：
- 先做权限检查
- 如果允许，就执行 `executeToolCall("write_file", input)`

执行成功后，得到一个结果字符串，比如：

```text
Successfully wrote to ./hello.py (...)
```

### 第 4 步：把写文件结果塞回上下文

这一步最关键：
- 不是只打印给用户
- 而是把结果写回消息历史

这样下一轮 LLM 才知道：
- `hello.py` 已经创建成功了

### 第 5 步：第二轮调用 LLM

因为上一轮 assistant 返回的是 tool call，不是最终答复，所以循环继续。

这时 LLM 看到新上下文，很可能请求：

```text
run_shell { command: "python hello.py" }
```

或者：

```text
run_shell { command: "python3 hello.py" }
```

### 第 6 步：Agent 执行 shell

这次仍然走同一条分发链：

```ts
executeToolCall("run_shell", input)
  -> executeTool("run_shell", input, this.readFileState)
  -> runShell(input)
  -> execSync(input.command, ...)
```

### 第 7 步：把 shell 输出再塞回上下文

假设命令输出：

```text
Hello Mini Claude
```

这个结果也会重新写回消息历史。

于是下一轮 LLM 就知道：
- 文件写好了
- 脚本运行成功了
- 输出是 `Hello Mini Claude`

### 第 8 步：LLM 不再请求工具，循环结束

最后一轮，LLM 通常会直接给自然语言总结：
- 文件已创建
- 脚本已运行
- 输出是什么

这时 assistant 回复里已经没有 tool call 了。

于是主循环命中：
- “没有 tool call，break”

整个任务结束。

### 这个例子最该记住的本质

这个任务看起来像“自动完成一整串动作”，但底层其实只是同一个闭环重复了几次：

```text
LLM 选工具
-> Agent 执行工具
-> 工具结果写回上下文
-> LLM 基于新上下文选下一步
```

如果中间运行报错，也是一样：
- LLM 看到错误文本
- 再决定要不要 `read_file` / `edit_file` / `run_shell`
- 然后继续循环

---

## 17. 最适合回顾的速记

如果你一周后回来复习，优先记这 12 条：

1. `chat()` 只是入口，真正循环在 `chatAnthropic()` / `chatOpenAI()`。
2. 主循环的本质是一个 `while (true)`。
3. 每轮都在做“LLM -> 工具 -> LLM -> 工具”。
4. assistant 没有 tool call 时，循环结束。
5. tool call 不会直接执行，先走 `checkPermission(...)`。
6. 统一工具入口是 `executeToolCall(...)`。
7. 普通工具最后才会进 `executeTool(...)`。
8. `run_shell` 最终落到 `runShell(...) -> execSync(...)`。
9. 工具执行完，最关键的是把结果重新写回消息历史。
10. Anthropic 用 `tool_use` / `tool_result`；OpenAI 用 `tool_calls` / `role: "tool"`。
11. 子代理本质上也是一个新的 `Agent`，跑完后只返回字符串结果。
12. 整个系统最核心的能力，不是“会调工具”，而是“会把工具结果继续喂回 LLM 做下一步决策”。

---

## 18. 一句话总结

这个项目里的 Agent Loop，本质上就是：

```text
用消息历史当状态，把 LLM 当决策器，把工具当执行器；
每轮让 LLM 基于当前状态选择下一步，再把执行结果写回状态，直到任务完成。
```
