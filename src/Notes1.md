# Mini Claude Code 学习笔记

这份笔记按源码阅读顺序整理，当前覆盖：
- `cli.ts` 的启动和 REPL 逻辑
- `agent.ts` 开头的辅助函数、配置、构造函数
- `agent.ts` 中从 `togglePlanMode()` 到 `// ─── Anthropic backend ─` 之前的全部模块

目标：
- 先建立“启动链路”和“Agent 类功能地图”
- 再按模块理解每个方法做什么
- 每一段都顺手记录容易混淆的 TS / JS 语法

---

## 1. 启动主线：先记住整体链路

最重要的主线：

```text
package.json -> dist/cli.js -> src/cli.ts -> main() -> new Agent(...) -> agent.chat(...) / runRepl(agent)
```

可以先这样分层：
- `cli.ts`：程序入口、参数解析、REPL 交互
- `agent.ts`：Agent 内核，负责模型调用、工具调用、权限控制、上下文维护

一句话理解：
- `cli.ts` 决定“程序怎么启动”
- `agent.ts` 决定“agent 怎么工作”

---

## 2. `cli.ts`：程序入口层

### 2.1 `parseArgs()`

作用：
- 解析命令行参数
- 整理成一个结构化对象

会解析的内容：
- 权限模式：`--yolo`、`--plan`、`--accept-edits`、`--dont-ask`
- 模型：`--model`
- API 地址：`--api-base`
- 是否恢复会话：`--resume`
- 预算：`--max-cost`、`--max-turns`
- 位置参数 prompt

返回值类型：

```ts
interface ParsedArgs {
  permissionMode: PermissionMode;
  model: string;
  apiBase?: string;
  prompt?: string;
  resume?: boolean;
  thinking?: boolean;
  maxCost?: number;
  maxTurns?: number;
}
```

重点：
- 这里只是把用户输入整理好
- 还没有真正创建 agent

### 2.2 `runRepl(agent)`

作用：
- 创建 `readline` 实例
- 绑定交互能力
- 循环读取用户输入

它负责：
- `agent.setConfirmFn(...)`
- `agent.setPlanApprovalFn(...)`
- `Ctrl + C` 中断处理
- 调用 `askQuestion()` 进入循环

### 2.3 `askQuestion()`

作用：
- 一次读取用户输入的一整行
- 处理后再等待下一行

核心写法：

```ts
const askQuestion = (): void => {
  printUserPrompt();
  rl.once("line", async (line) => {
    ...
    askQuestion();
  });
};
```

怎么理解：
- 先打印提示符
- 通过 `rl.once("line", ...)` 等用户输入
- 输入完成后处理这一轮
- 最后再次调用 `askQuestion()`

它看起来像递归，但本质上更接近事件循环，不是同步递归。

为什么用 `once("line")`：
- 一次只处理一条输入
- 当前输入处理完，才开始下一轮

### 2.4 REPL 输入怎么分流

有两大类：

1. REPL 命令
- `/clear`
- `/plan`
- `/cost`
- `/compact`
- `/memory`
- `/skills`

2. 普通用户提问

最终都会走：

```ts
await agent.chat(input);
```

也就是把这一轮输入交给 Agent 内核。

### 2.5 `main()`

作用：
- 启动整个程序
- 决定是 one-shot 模式还是 REPL 模式

大致流程：
1. `parseArgs()`
2. 读取环境变量中的 API key / API base
3. `new Agent(...)`
4. 如果 `--resume`，恢复会话
5. 如果命令行里直接有 prompt，就 `await agent.chat(prompt)`
6. 否则进入 `await runRepl(agent)`

---

## 3. `agent.ts` 开头：辅助能力和配置

在 `export class Agent` 之前，先定义了一批帮助函数和常量。

### 3.1 重试机制：`isRetryable()` + `withRetry<T>()`

作用：
- 处理模型 API 或网络的临时失败
- 自动重试几次，并且每次失败后等待更久

`isRetryable(error)`：
- 判断某个错误是否值得重试
- 比如 `429`、`503`、`ECONNRESET`、`ETIMEDOUT`

`withRetry<T>(fn, signal, maxRetries)`：
- 接收一个异步函数 `fn`
- 执行它
- 成功就返回
- 失败且可重试就等待后重试
- 失败且不可重试就抛错

这里的 `T` 是泛型，表示：
- `withRetry` 本身不关心最终返回什么类型
- 由传进去的 `fn` 决定

### 3.2 模型能力配置

`MODEL_CONTEXT` + `getContextWindow(model)`：
- 根据模型名决定上下文窗口大小

`modelSupportsThinking(model)`：
- 判断某个模型是否支持 thinking

`modelSupportsAdaptiveThinking(model)`：
- 判断某个模型是否支持 adaptive thinking

`getMaxOutputTokens(model)`：
- 根据模型决定单次最大输出 token

### 3.3 `toOpenAITools(...)`

作用：
- 把内部工具定义转换成 OpenAI-compatible API 所需的 tools 格式

一句话理解：
- 这是“工具 schema 适配器”

### 3.4 压缩常量

这些常量用于后面的上下文压缩：
- `SNIPPABLE_TOOLS`
- `SNIP_PLACEHOLDER`
- `SNIP_THRESHOLD`
- `MICROCOMPACT_IDLE_MS`
- `KEEP_RECENT_RESULTS`

它们本身不执行逻辑，只是给后面的压缩逻辑提供参数。

---

## 4. `AgentOptions`、类字段、构造函数：Agent 的“配置 + 状态 + 初始化”

### 4.1 `interface AgentOptions`

作用：
- 定义 `new Agent({...})` 时允许传入哪些配置项

可以分三类：

1. 运行环境
- `model`
- `apiBase`
- `anthropicBaseURL`
- `apiKey`

2. 行为控制
- `permissionMode`
- `yolo`
- `thinking`
- `maxCostUsd`
- `maxTurns`
- `confirmFn`

3. 子代理配置
- `customSystemPrompt`
- `customTools`
- `isSubAgent`

### 4.2 `export class Agent { private ... }`

作用：
- 定义一个 Agent 实例内部需要维护的运行状态

比如：
- 用哪个后端：`anthropicClient` / `openaiClient`
- 当前模型：`model`
- 当前模式：`permissionMode`
- token 统计：`totalInputTokens`
- 会话状态：`sessionId`
- 消息历史：`anthropicMessages` / `openaiMessages`
- plan mode 状态：`prePlanMode`、`planFilePath`
- 中断控制：`abortController`

### 4.3 `constructor(options: AgentOptions = {})`

作用：
- 把外部传入的 `options` 转成 Agent 实例的初始状态

核心步骤：
1. 读取配置
2. 给默认值
3. 计算派生值
4. 构建 system prompt
5. 初始化 OpenAI 或 Anthropic client

例如：

```ts
this.permissionMode = options.permissionMode
  || (options.yolo ? "bypassPermissions" : "default");
```

这里体现了：
- `permissionMode` 优先
- 如果没有，就看 `yolo`
- 再没有，就用默认 `default`

---

## 5. Agent 中段功能总表

下面这张表覆盖 [agent.ts](/home/mihoyohb/Claude_Code_Repo/claude-codeMihoyohb/src/agent.ts) 中从 `togglePlanMode()` 到 `// ─── Anthropic backend ─` 之前的主要方法。

| 模块 | 方法 | 作用 | 关键状态 / 返回值 |
| --- | --- | --- | --- |
| 回调注册 | `setConfirmFn` | 设置危险操作确认回调 | 写入 `this.confirmFn` |
| 回调注册 | `setPlanApprovalFn` | 设置 plan 审批回调 | 写入 `this.planApprovalFn` |
| 模式切换 | `togglePlanMode` | REPL 中手动开关 plan mode | 修改 `permissionMode`、`planFilePath`、`systemPrompt` |
| 状态读取 | `getPermissionMode` | 返回当前权限模式 | 返回字符串 |
| 状态读取 | `getTokenUsage` | 返回 token 统计 | `{ input, output }` |
| 对话入口 | `chat` | 主入口，按后端分发到 `chatOpenAI` 或 `chatAnthropic` | 管理中断、自动保存 |
| 子代理入口 | `runOnce` | 让子代理执行一轮并返回文本结果 | 返回 `{ text, tokens }` |
| 输出辅助 | `emitText` | 主代理打印输出，子代理写入 buffer | `outputBuffer` / `printAssistantText` |
| REPL 配套 | `clearHistory` | 清空对话历史和 token 统计 | 重置 messages、计数器 |
| REPL 配套 | `showCost` | 打印当前 token 和费用估算 | 调用 `getCurrentCostUsd` |
| 预算控制 | `getCurrentCostUsd` | 计算费用估算 | 返回 number |
| 预算控制 | `checkBudget` | 判断是否超预算或超轮数 | 返回 `{ exceeded, reason? }` |
| 压缩入口 | `compact` | REPL 里配合 `/compact` 手动触发摘要压缩 | 只是转调 `compactConversation()` |
| 会话恢复 | `restoreSession` | 从保存的会话恢复消息历史 | 覆盖 messages |
| 会话恢复 | `getMessageCount` | 获取当前消息条数 | 根据后端取数组长度 |
| 自动保存 | `autoSave` | 把当前会话写入 session 文件 | 调用 `saveSession` |
| 自动压缩 | `checkAndCompact` | 单轮结束后检查是否需要做摘要压缩 | 根据 `lastInputTokenCount` 和 `effectiveWindow` 判断 |
| 压缩入口 | `compactConversation` | 摘要压缩统一入口 | 按后端调 `compactOpenAI()` / `compactAnthropic()`，最后打印 `Conversation compacted.` |
| 摘要压缩 | `compactAnthropic` | 用 Anthropic 再发一次“总结对话”请求，替换旧历史 | 重建 `anthropicMessages`，并尽量保留最后一条用户消息 |
| 摘要压缩 | `compactOpenAI` | 用 OpenAI-compatible 接口生成摘要后替换旧历史 | 重建 `openaiMessages`，保留 `systemMsg` 和最后一条用户消息 |
| 多层压缩 | `runCompressionPipeline` | 每次真正发 API 前，执行 budget / snip / microcompact 三层本地压缩 | `chatAnthropic()` / `chatOpenAI()` 循环里都会先调用 |
| 多层压缩 | `budgetToolResults*` | 上下文利用率升高时，先裁剪过长工具结果 | 只改工具结果，保头保尾截断中间内容 |
| 多层压缩 | `snipStaleResults*` | 把较旧或重复工具结果替换成占位符 | 用 `SNIP_PLACEHOLDER` 代替，Anthropic 版还会结合 `tool_use_id` 判断工具来源 |
| 多层压缩 | `microcompact*` | 闲置超过 5 分钟后，更激进清理旧工具结果 | 保留最近 `KEEP_RECENT_RESULTS` 条，其余替换成 `[Old result cleared]` |
| 查找辅助 | `findToolUseById` | 根据 tool use id 找到原工具信息 | 返回 `{ name, input } \| null` |
| 工具分发 | `executeToolCall` | 统一处理普通工具、plan tools、skill、sub-agent、MCP 工具分发 | 普通工具会带上 `readFileState`，MCP 工具转给 `mcpManager` |
| Skill | `executeSkillTool` | 执行 skill，支持 inline 和 fork 两种模式 | 可能返回 prompt，也可能运行子代理 |
| Plan mode | `generatePlanFilePath` | 生成 plan 文件路径 | 使用 `~/.claude/plans` |
| Plan mode | `buildPlanModePrompt` | 构建 plan mode 专用 system prompt 追加内容 | 返回大段字符串 |
| Plan mode | `executePlanModeTool` | 处理 `enter_plan_mode` / `exit_plan_mode` | 切模式、审批 plan、决定后续执行方式 |
| Plan mode | `clearHistoryKeepSystem` | 清空对话但保留 system prompt | 重置 messages，保留 system |
| 子代理 | `executeAgentTool` | 启动 sub-agent 完成一个子任务 | 返回子代理文本结果 |

---

## 6. 模块一：回调注册与模式切换

这一组主要包括：
- `setConfirmFn`
- `setPlanApprovalFn`
- `togglePlanMode`
- `getPermissionMode`
- `getTokenUsage`

### 6.1 `setConfirmFn(fn)`

作用：
- 把一个“危险操作确认函数”交给 Agent 保存

代码逻辑非常简单：

```ts
setConfirmFn(fn: (message: string) => Promise<boolean>) {
  this.confirmFn = fn;
}
```

怎么理解：
- `Agent` 不自己决定怎么跟终端交互
- 外部的 `cli.ts` 用 `readline` 实现一个确认函数
- 然后把这个函数注入给 Agent

以后当 Agent 需要确认危险操作时，就调用 `this.confirmFn(...)`。

这里体现的是“依赖注入”的思想：
- UI 交互逻辑在 `cli.ts`
- Agent 只关心“我需要一个确认函数”

### 6.2 `setPlanApprovalFn(fn)`

作用：
- 把一个“plan 审批函数”注入给 Agent

函数类型比较长：

```ts
fn: (planContent: string) => Promise<{
  choice: "clear-and-execute" | "execute" | "manual-execute" | "keep-planning";
  feedback?: string;
}>
```

如何拆解：
- `fn` 是一个函数
- 参数是 `planContent: string`
- 返回值是一个 Promise
- Promise 里最终 resolve 出一个对象

这个对象里的 `choice` 不是任意字符串，而是固定几种字面量：
- `"clear-and-execute"`
- `"execute"`
- `"manual-execute"`
- `"keep-planning"`

这属于 TS 中的“字符串字面量联合类型”。

意义：
- 返回值更明确
- 后面处理逻辑时，TS 能知道只会出现这几种分支

### 6.3 `togglePlanMode(): string`

作用：
- 给 REPL 的 `/plan` 命令使用
- 手动切换 plan mode 的开和关

一句话理解：

```ts
togglePlanMode()
```

等于：

“如果当前是普通模式，就进入 plan mode；如果当前已经是 plan mode，就退出并恢复原模式。”

#### 进入 plan mode 时做了什么

1. 记住原来的权限模式

```ts
this.prePlanMode = this.permissionMode;
```

2. 把当前模式切到 `plan`

```ts
this.permissionMode = "plan";
```

3. 生成 plan 文件路径

```ts
this.planFilePath = this.generatePlanFilePath();
```

4. 修改 system prompt

```ts
this.systemPrompt = this.baseSystemPrompt + this.buildPlanModePrompt();
```

含义：
- 原始 prompt 还在
- 额外再拼上一段“当前是 plan mode，只能读、不能改，只能写 plan 文件”的约束

5. 如果使用 OpenAI-compatible 后端，还要同步更新 `openaiMessages[0]`

```ts
if (this.useOpenAI && this.openaiMessages.length > 0) {
  (this.openaiMessages[0] as any).content = this.systemPrompt;
}
```

原因：
- OpenAI 这边把 system prompt 放在 `messages[0]`
- 所以 prompt 变了，第一条 system message 也要一起更新

6. 打印提示并返回 `"plan"`

#### 退出 plan mode 时做了什么

1. 恢复进入前的模式

```ts
this.permissionMode = this.prePlanMode || "default";
```

2. 清理 plan mode 的临时状态

```ts
this.prePlanMode = null;
this.planFilePath = null;
```

3. 恢复原始 system prompt

```ts
this.systemPrompt = this.baseSystemPrompt;
```

4. 如果是 OpenAI-compatible，同步恢复 `openaiMessages[0]`

5. 打印提示并返回当前模式名

#### 这个函数最重要的三件事

它不是单纯改一个布尔值，而是同时维护：
- 权限模式
- plan 文件路径
- system prompt

所以 plan mode 是一个“完整运行状态切换”，不是简单的 UI 标记。

### 6.4 `getPermissionMode()`

作用：
- 返回当前权限模式

```ts
getPermissionMode(): string {
  return this.permissionMode;
}
```

这是一个非常典型的 getter 风格方法。

### 6.5 `getTokenUsage()`

作用：
- 返回当前 token 统计

```ts
return { input: this.totalInputTokens, output: this.totalOutputTokens };
```

这里返回的是一个对象字面量，不是数组。

可以记住：
- `input`：累计输入 token
- `output`：累计输出 token

### 6.6 这一组的 TS 语法重点

`fn: (...) => Promise<boolean>`：
- 函数类型标注
- 说明这个参数本身是一个函数

`feedback?: string`：
- 可选属性
- 有时有，有时没有

`"a" | "b" | "c"`：
- 联合类型
- 说明这个值只能是其中之一

`(this.openaiMessages[0] as any).content`：
- 类型断言
- 告诉 TS：“这里先别严格检查，按 `any` 处理”
- 常用于“我知道这里运行时没问题，但类型系统不够精确”的场景

---

## 7. 模块二：主对话入口与子代理输出

这一组包括：
- `chat`
- `runOnce`
- `emitText`

### 7.1 `async chat(userMessage: string): Promise<void>`

这是 Agent 的主入口之一，也是最关键的方法之一。

作用：
- 接收用户一轮输入
- 创建中断控制器
- 根据当前后端，选择走 `chatOpenAI()` 或 `chatAnthropic()`
- 在对话结束后做收尾工作

核心逻辑：

```ts
this.abortController = new AbortController();
try {
  if (this.useOpenAI) {
    await this.chatOpenAI(userMessage);
  } else {
    await this.chatAnthropic(userMessage);
  }
} finally {
  this.abortController = null;
}
```

重点：
- `AbortController` 用来支持中断
- `try/finally` 确保无论成功还是失败，最后都把 `abortController` 清掉

最后还有一段：

```ts
if (!this.isSubAgent) {
  printDivider();
  this.autoSave();
}
```

意思是：
- 主代理完成一轮对话后，打印分隔线并自动保存
- 子代理不做这些 UI 层的动作

### 7.2 `runOnce(prompt)`

作用：
- 给 sub-agent 用的一次性执行入口
- 跑完一轮后把文本结果和 token 使用情况打包返回

流程：
1. 开启 `outputBuffer`
2. 记录调用前的 token 计数
3. 调用 `await this.chat(prompt)`
4. 把 buffer 里的文本拼接起来
5. 计算本轮实际消耗的 token
6. 返回 `{ text, tokens }`

这里的关键思路是：
- 子代理不直接把输出打印到终端
- 而是先写进 buffer
- 主代理调用完子代理后，再拿到结果字符串

### 7.3 `private emitText(text: string): void`

作用：
- 统一处理“文本输出到哪里”

逻辑：

```ts
if (this.outputBuffer) {
  this.outputBuffer.push(text);
} else {
  printAssistantText(text);
}
```

怎么理解：
- 如果当前是子代理模式，输出写到 `outputBuffer`
- 如果当前是主代理模式，直接打印到终端

这是一种很常见的“输出抽象”：
- 上层调用不用关心当前是打印还是缓存
- 统一走 `emitText()`

### 7.4 这一组的 TS / JS 语法重点

`Promise<void>`：
- 表示这是个异步函数
- 会返回 Promise
- 但没有有意义的返回值

`string[] | null`：
- 联合类型
- 表示 `outputBuffer` 可能是字符串数组，也可能是 `null`

`try { ... } finally { ... }`：
- 无论是否抛错，`finally` 都会执行
- 常用于资源清理

---

## 8. 模块三：REPL 配套、预算控制、会话恢复

这一组包括：
- `clearHistory`
- `showCost`
- `getCurrentCostUsd`
- `checkBudget`
- `compact`
- `restoreSession`
- `getMessageCount`
- `autoSave`

### 8.1 `clearHistory()`

作用：
- 清空当前对话历史和 token 统计

它会做这些事：
- 清空 `anthropicMessages`
- 清空 `openaiMessages`
- 如果当前是 OpenAI-compatible，再重新补一条 system message
- 把输入输出 token 统计归零

为什么 OpenAI 这里要补 system message：
- 因为 OpenAI 的 system prompt 放在消息数组里
- 清空数组后要重新塞回去

### 8.2 `showCost()`

作用：
- 在 REPL 中打印当前 token 和费用估算

这里会根据是否设置预算上限拼接额外信息：

```ts
const budgetInfo = this.maxCostUsd ? ` / $${this.maxCostUsd} budget` : "";
const turnInfo = this.maxTurns ? ` | Turns: ${this.currentTurns}/${this.maxTurns}` : "";
```

说明：
- 这是典型的条件字符串拼接
- 有值就拼，没有就拼空字符串

### 8.3 `private getCurrentCostUsd(): number`

作用：
- 按 token 数估算当前费用

公式：
- 输入 token：每百万 3 美元
- 输出 token：每百万 15 美元

```ts
const costIn = (this.totalInputTokens / 1_000_000) * 3;
const costOut = (this.totalOutputTokens / 1_000_000) * 15;
return costIn + costOut;
```

### 8.4 `private checkBudget()`

作用：
- 检查是否超出成本预算或轮数预算

返回值不是布尔值，而是一个对象：

```ts
{ exceeded: boolean; reason?: string }
```

为什么这样设计：
- 既能告诉外部“有没有超”
- 也能顺便提供原因文字

检查规则：
- 如果设置了 `maxCostUsd` 且当前成本超过上限，就返回超限
- 如果设置了 `maxTurns` 且轮数超过上限，就返回超限
- 否则返回 `{ exceeded: false }`

### 8.5 `async compact()`

作用：
- 手动触发一次对话压缩

本质上只是一个简单转调：

```ts
await this.compactConversation();
```

实际用法：
- 在 REPL 里输入 `/compact`
- `cli.ts` 会调用 `agent.compact()`
- 然后这里再进入 `compactConversation()`

一句话理解：
- `compact()` 本身不决定怎么压缩
- 它只是给 REPL 暴露一个“手动压缩入口”

### 8.6 `restoreSession(data)`

作用：
- 从保存的 session 数据恢复消息历史

逻辑：
- 如果传入了 `anthropicMessages`，就恢复它
- 如果传入了 `openaiMessages`，就恢复它
- 然后打印恢复提示

注意：
- 这里只恢复消息历史
- 并没有重新计算 token 统计

### 8.7 `private getMessageCount()`

作用：
- 根据当前后端，返回当前消息历史条数

逻辑很简单：

```ts
return this.useOpenAI ? this.openaiMessages.length : this.anthropicMessages.length;
```

### 8.8 `private autoSave()`

作用：
- 自动保存当前会话到 session 存储

保存内容包括：
- 元信息：session id、model、cwd、startTime、messageCount
- 当前后端对应的消息历史

这里包了一个空 `catch {}`：

```ts
try {
  saveSession(...);
} catch {}
```

意思是：
- 自动保存失败时不要打断主流程
- 这是“尽力而为”的辅助功能

### 8.9 这一组的 TS / JS 语法重点

三元表达式：

```ts
condition ? a : b
```

意思是：
- 条件成立取 `a`
- 否则取 `b`

对象返回值：

```ts
return { exceeded: true, reason: "..." };
```

比单独返回多个值更适合表达“状态 + 附加信息”。

空 `catch {}`：
- 表示忽略错误
- 在关键流程之外的辅助逻辑里比较常见

---

## 9. 模块四：自动压缩与摘要压缩

这一组包括：
- `checkAndCompact`
- `compactConversation`
- `compactAnthropic`
- `compactOpenAI`

### 9.1 `checkAndCompact()`

作用：
- 在上下文逼近上限时，自动触发一次压缩

关键判断：

```ts
if (this.lastInputTokenCount > this.effectiveWindow * 0.85)
```

意思是：
- 如果最近一次请求的输入 token 已经超过有效窗口的 85%
- 就开始压缩

为什么用 `effectiveWindow` 而不是完整窗口：
- 因为构造函数里已经预留了安全缓冲

### 9.2 `compactConversation()`

作用：
- 统一入口，根据后端选择对应压缩实现

```ts
if (this.useOpenAI) {
  await this.compactOpenAI();
} else {
  await this.compactAnthropic();
}
```

压缩完成后打印提示：

```ts
printInfo("Conversation compacted.");
```

这一步的实际用法有两个入口：
- 手动入口：用户在 REPL 输入 `/compact`，最终会走到这里
- 自动入口：`chatAnthropic()` / `chatOpenAI()` 每轮结束后都会 `await this.checkAndCompact()`，满足阈值时也会走到这里

所以可以把它理解成：
- “摘要压缩总闸门”
- 不管是手动还是自动，真正执行摘要压缩都要经过它

### 9.3 `compactAnthropic()`

作用：
- 用 Anthropic 再请求一次“摘要模型调用”
- 把旧对话压缩成一段摘要

核心思路：
1. 如果消息太少，就不压缩
2. 取出最后一条用户消息
3. 构造一个“请总结之前对话”的请求
4. 调 `anthropicClient.messages.create(...)`
5. 拿到摘要文本
6. 用“摘要 + 一句确认话术 + 最后一条用户消息”重建消息历史

为什么要保留最后一条用户消息：
- 避免压缩之后丢掉当前任务上下文

代码里的重建结果可以直接记成：

```ts
[
  { role: "user", content: `[Previous conversation summary]\n${summaryText}` },
  { role: "assistant", content: "Understood..." },
  // 如果最后一条原本就是 user，再把它补回去
]
```

也就是说：
- 旧历史不会原样保留
- 会被折叠成“一条摘要 user 消息 + 一条确认 assistant 消息”
- 当前这轮尚未处理完的最后一条用户请求，尽量继续保留

### 9.4 `compactOpenAI()`

作用：
- 跟 `compactAnthropic()` 类似，只是换成 OpenAI-compatible 的消息格式

区别点：
- OpenAI 这边 `system` 是消息数组的第一条
- 所以压缩后要把原来的 `systemMsg` 保留在最前面

可以把重建后的结构理解成：

```ts
[
  systemMsg,
  { role: "user", content: `[Previous conversation summary]\n${summaryText}` },
  { role: "assistant", content: "Understood..." },
  // 必要时补回 lastUserMsg
]
```

所以它和 Anthropic 版最大的差别不是“摘要逻辑不同”，而是：
- OpenAI 的 system prompt 在消息数组里
- Anthropic 的 system prompt 是单独参数传给 API

### 9.5 这两种压缩的本质

它们都不是简单“删旧消息”，而是：
- 重新请求模型生成摘要
- 用摘要代替大段旧消息

这样可以节省上下文，同时尽量保留任务延续所需的信息。

### 9.6 这一组的 TS / JS 语法重点

数组展开：

```ts
[...this.anthropicMessages.slice(0, -1), ...summaryReq]
```

意思是：
- 先取前面的旧消息
- 再拼上新的摘要请求

`slice(0, -1)`：
- 取数组除最后一个元素以外的部分

可选链：

```ts
summaryResp.content[0]?.type
```

表示：
- 如果 `content[0]` 存在，再访问它的 `type`

---

## 10. 模块五：多层压缩流水线

这一组包括：
- `runCompressionPipeline`
- `budgetToolResultsAnthropic`
- `budgetToolResultsOpenAI`
- `snipStaleResultsAnthropic`
- `snipStaleResultsOpenAI`
- `microcompactAnthropic`
- `microcompactOpenAI`
- `findToolUseById`

这一组的特点是：
- 不通过额外 API 生成摘要
- 直接在本地消息数组上做低成本压缩

作者把它称为四层压缩中的前 3 层：
- budget
- snip
- microcompact
- auto-compact

其中真正生成摘要的 `compactConversation()` 属于最后的 auto-compact。

### 10.1 `runCompressionPipeline()`

作用：
- 统一执行前 3 层本地压缩逻辑

```ts
if (this.useOpenAI) {
  this.budgetToolResultsOpenAI();
  this.snipStaleResultsOpenAI();
  this.microcompactOpenAI();
} else {
  this.budgetToolResultsAnthropic();
  this.snipStaleResultsAnthropic();
  this.microcompactAnthropic();
}
```

实际调用位置非常重要：

```ts
while (true) {
  ...
  this.runCompressionPipeline();
  const response = await this.callXXXStream(...);
  ...
}
```

也就是：
- 每次真正请求模型前，都会先跑一次本地三层压缩
- 这三层不需要额外 API 成本
- 它们的目标是尽量避免太早进入更重的摘要压缩

### 10.2 第一层：`budgetToolResults*`

作用：
- 当上下文使用率开始升高时
- 对特别长的工具结果做“保头保尾”的裁剪

它只处理工具结果，不处理普通用户消息或 assistant 文本：
- Anthropic: 只看 `user` 消息里的 `tool_result block`
- OpenAI: 只看 `role === "tool"` 的消息

判断逻辑：

```ts
const utilization = this.lastInputTokenCount / this.effectiveWindow;
if (utilization < 0.5) return;
```

意思是：
- 使用率不到 50%，先不动
- 超过 50% 开始裁剪

预算值：

```ts
const budget = utilization > 0.7 ? 15000 : 30000;
```

意思是：
- 越接近上限，裁得越狠

保留策略：
- 保留开头一段
- 保留结尾一段
- 中间替换成截断说明

这和很多日志截断策略类似。

### 10.3 第二层：`snipStaleResults*`

作用：
- 把较旧、重复、已经不太需要的工具结果替换成占位符

Anthropic 版本更复杂，因为：
- `tool_result` 是嵌在 message block 里的
- 还要根据 `tool_use_id` 找回原工具名和输入

OpenAI 版本更简单，因为：
- `tool` 结果是单独消息

Anthropic 版本的关键思路：
1. 遍历所有消息，收集可 snip 的 `tool_result`
2. 如果是 `read_file` 且读的是同一个文件，旧的那次优先 snip
3. 超过 `KEEP_RECENT_RESULTS` 的老结果也会被 snip
4. 把内容替换为：

```ts
SNIP_PLACEHOLDER
```

OpenAI 版本的关键思路：
- 收集所有 `role === "tool"` 的消息
- 除最近 N 条外，其余替换为占位符

这里要特别注意一个“代码层面的差异”：
- Anthropic 版会尽量识别“同一个文件被重复 `read_file`”这种情况，优先 snip 旧结果
- OpenAI 版当前实现没有回溯具体工具输入，只是简单按时间顺序保留最近 N 条

### 10.4 第三层：`microcompact*`

作用：
- 如果很久没有继续调用模型
- 就更激进地清理旧工具结果

触发条件：

```ts
if (!this.lastApiCallTime || (Date.now() - this.lastApiCallTime) < MICROCOMPACT_IDLE_MS) return;
```

意思是：
- 如果没有 API 调用记录，或者离上次调用还没过够 5 分钟，就不触发

触发后：
- 收集所有还没被清理的工具结果
- 保留最近 `KEEP_RECENT_RESULTS`
- 其余替换为：

```ts
"[Old result cleared]"
```

怎么理解这一层的用法：
- 它不是看“上下文快满没满”
- 而是看“距离上次 API 调用是不是已经过去很久”
- 如果缓存已经变冷，就把更旧的工具结果进一步清掉，给后续轮次腾空间

### 10.5 `findToolUseById(toolUseId)`

作用：
- 在 Anthropic 消息历史里，根据 `tool_use_id` 找到当时调用的是哪个工具、输入是什么

返回值：

```ts
{ name: string; input: any } | null
```

为什么需要它：
- 在 `snipStaleResultsAnthropic()` 里，仅有 `tool_result` 还不够
- 还要知道这个结果是哪个工具产生的，比如是不是 `read_file`

### 10.6 这一组的 TS / JS 语法重点

`Set`：
- 适合做“是否属于某个集合”的判断
- 这里用来记录可裁剪工具、待 snip 的索引

`Map<string, number[]>`：
- key 是文件路径
- value 是这个文件在结果数组中出现过的索引列表

`Array.isArray(msg.content)`：
- 类型收窄的一种常见手段
- 先确认真的是数组，再安全遍历

`as any[]`：
- 类型断言
- 告诉 TS 暂时按数组处理

`for ... of` 和 `for (let i = 0; ...)`
- 这里两种都用了
- 需要索引时更常用传统 `for`

---

## 11. 模块六：工具分发、Skill、Plan mode 帮助函数、Sub-agent

这一组包括：
- `executeToolCall`
- `executeSkillTool`
- `generatePlanFilePath`
- `buildPlanModePrompt`
- `executePlanModeTool`
- `clearHistoryKeepSystem`
- `executeAgentTool`

这一组的地位很重要：
- 它们把“工具调用”这件事和 Agent 主循环接起来
- 也把 skill、plan mode、sub-agent、MCP 这些高级能力组织起来

### 11.1 `executeToolCall(name, input)`

作用：
- 统一的工具调用分发入口

逻辑：

```ts
if (name === "enter_plan_mode" || name === "exit_plan_mode") return await this.executePlanModeTool(name);
if (name === "agent") return this.executeAgentTool(input);
if (name === "skill") return this.executeSkillTool(input);
if (this.mcpManager.isMcpTool(name)) return this.mcpManager.callTool(name, input);
return executeTool(name, input, this.readFileState);
```

怎么理解：
- `plan mode` 工具先走内部状态切换逻辑
- `agent` 工具会拉起一个真正的 sub-agent
- `skill` 工具会走 skill 分发
- 如果工具名属于 MCP server 提供的工具，就转给 `mcpManager.callTool(...)`
- 剩下的普通本地工具，才交给 `executeTool(...)`

这个函数相当于“工具分发器”。

这里还有一个容易忽略的变化：
- 普通工具现在不是简单 `executeTool(name, input)`
- 而是 `executeTool(name, input, this.readFileState)`

这表示：
- 工具执行层现在还能拿到“文件上次读取状态”
- 给后面的 read-before-edit 保护做辅助判断

### 11.2 `executeSkillTool(input)`

作用：
- 执行 skill 工具
- 支持两种模式：`inline` 和 `fork`

先通过动态导入拿到 skill 执行逻辑：

```ts
const { executeSkill } = await import("./skills.js");
```

再执行：

```ts
const result = executeSkill(input.skill_name, input.args || "");
```

如果 skill 不存在：

```ts
return `Unknown skill: ${input.skill_name}`;
```

#### `inline` 模式

如果 skill 只是想给主代理注入一段 prompt：

```ts
return `[Skill "${input.skill_name}" activated]\n\n${result.prompt}`;
```

也就是：
- 不开子代理
- 直接把 skill prompt 作为 tool result 返回给当前上下文
- 下一轮模型就能把这段 prompt 当成额外工作指令继续用

#### `fork` 模式

如果 skill 需要在隔离上下文里运行：

1. 根据 skill 允许的工具列表过滤工具
2. 打印子代理开始提示
3. `new Agent(...)` 创建 sub-agent
4. 调 `subAgent.runOnce(...)`
5. 把子代理 token 消耗累加到主代理
6. 返回子代理结果文本

这里的关键点：
- `fork` 模式真的会开一个新的 Agent 实例
- 但这个子代理的工具集、system prompt 可以和主代理不一样

更贴近代码地看，有两个分支：

1. 如果 skill 返回了 `allowedTools`
- 就从当前 `this.tools` 里只保留这些工具

2. 如果 skill 没限制工具
- 默认会把 `agent` 工具排除掉：

```ts
this.tools.filter(t => t.name !== "agent")
```

这样做的意图是：
- skill fork 默认不开启“再继续套娃开子代理”
- 先把 skill 子代理限制在一个更可控的范围里

另外这个 sub-agent 的关键配置也值得记一下：
- `customSystemPrompt` 用的是 `result.prompt`
- `permissionMode` 如果主代理当前在 `plan`，子代理也跟着 `plan`
- 否则 skill fork 子代理直接用 `bypassPermissions`

### 11.3 `generatePlanFilePath()`

作用：
- 生成一个 plan 文件路径

逻辑：
1. 目标目录是 `~/.claude/plans`
2. 如果目录不存在，就创建
3. 文件名是 `plan-${this.sessionId}.md`

一句话理解：
- 为当前 session 生成独立 plan 文件

### 11.4 `buildPlanModePrompt()`

作用：
- 返回一段 plan mode 专用说明文字

这段文字会追加到原始 system prompt 后面，告诉模型：
- 现在处于 plan mode
- 不能做普通编辑，也不能运行非只读工具
- 只能编辑 plan 文件
- 复杂任务可以用 `agent` 工具并指定 `type="plan"`
- 完成后必须调用 `exit_plan_mode`

这段函数的本质不是“执行 plan mode”，而是“生成 plan mode 的提示词文本”。

### 11.5 `executePlanModeTool(name)`

作用：
- 处理模型通过 tool 调用发起的：
- `enter_plan_mode`
- `exit_plan_mode`

它和 `togglePlanMode()` 的区别：
- `togglePlanMode()` 是 REPL 的 `/plan` 命令入口
- `executePlanModeTool()` 是模型工具调用入口
- 后者还负责 plan 文件读取、审批分支和退出后的后续执行准备

#### `enter_plan_mode`

如果已经是 plan mode：

```ts
return "Already in plan mode.";
```

否则：
1. 记录旧模式
2. 切到 `plan`
3. 生成 plan 文件路径
4. 修改 `systemPrompt`
5. 如果是 OpenAI-compatible，同步更新第一条 system message
6. 打印提示
7. 返回一段说明文字给模型

这一段和 `togglePlanMode()` 很像，但更偏“工具协议”：
- 不只是切状态
- 还会把 plan 文件路径和后续操作规则明确返回给模型

#### `exit_plan_mode`

这是整个 plan mode 流程中最复杂的一段。

大致逻辑：

1. 如果当前不在 plan mode，直接返回
2. 读取 plan 文件内容
3. 如果有 `planApprovalFn`，走交互审批流程
4. 根据用户选择决定：
- 继续 planning
- 批准并执行
- 批准但手动执行
- 批准并清空上下文再执行
5. 更新 `permissionMode`
6. 恢复 `systemPrompt`
7. 必要时清空上下文，但保留 system prompt
8. 把 plan 内容包装成返回文本交还给模型

审批结果在代码里对应 4 种 `choice`：
- `keep-planning`
- `execute`
- `manual-execute`
- `clear-and-execute`

这 4 个分支的真实行为可以直接记成：

1. `keep-planning`
- 不退出 plan mode
- 把用户反馈包装成字符串返回给模型
- 让模型继续改 plan，之后再次调用 `exit_plan_mode`

2. `execute`
- 退出 plan mode
- 把权限切到 `acceptEdits`
- 保留当前上下文
- 把“已批准 plan”作为 tool result 返回，让模型继续实现

3. `manual-execute`
- 退出 plan mode
- 权限恢复到进入 plan 之前的模式，通常是 `prePlanMode || "default"`
- 把 plan 内容返回出来，但不强制切到 `acceptEdits`

4. `clear-and-execute`
- 退出 plan mode
- 权限切到 `acceptEdits`
- 调 `clearHistoryKeepSystem()`
- 再把批准后的 plan 作为“新的起点”交回主循环继续执行

其中最特别的是：

```ts
this.contextCleared = true;
```

含义：
- 这不是立刻继续执行
- 而是给后面的 agent 主循环一个信号
- 告诉它“这次 context 被清空了，下一步要把 plan 作为新的 user message 注入”

这个信号在后面的主循环里会被消费：
- Anthropic 路径下，会把返回文本塞回 `anthropicMessages` 的新 `user` 消息
- OpenAI 路径下，也会把返回文本塞回新的 `user` 消息
- 然后中断这轮原本的 tool result 追加流程，避免旧上下文再混回来

如果没有 `planApprovalFn`：
- 比如子代理场景
- 就直接退出 plan mode，不走交互审批
- 权限恢复到 `prePlanMode || "default"`
- 然后把 plan 内容原样返回

### 11.6 `clearHistoryKeepSystem()`

作用：
- 清空消息历史
- 但保留 system prompt

为什么要有这个函数：
- plan 批准后可能想“清空上下文重新执行”
- 但 system prompt 不能丢

所以它和 `clearHistory()` 的区别是：
- `clearHistory()` 是 REPL 用户手动清空，会顺带重置 token 统计
- `clearHistoryKeepSystem()` 是内部辅助方法，只做和 plan 流程相关的最小清理

按现在的代码，它具体只做三件事：
- 清空 `anthropicMessages`
- 清空 `openaiMessages`
- 如果是 OpenAI-compatible，重新补回第一条 `system` message

另外它只把 `lastInputTokenCount` 归零：
- 不会清空 `totalInputTokens`
- 也不会清空 `totalOutputTokens`

### 11.7 `executeAgentTool(input)`

作用：
- 真正执行 `agent` 工具，也就是拉起一个 sub-agent

流程：
1. 读取子代理类型 `type`
2. 读取子任务描述 `description`
3. 读取子任务 prompt
4. 打印子代理开始提示
5. `getSubAgentConfig(type)` 取对应子代理配置
6. `new Agent(...)` 创建 sub-agent
7. `await subAgent.runOnce(prompt)`
8. 把 token 用量累加到主代理
9. 返回子代理文本结果

这里的关键设计：
- 子代理本质上还是 `Agent`
- 只是 system prompt、tools、permissionMode 不同

默认值也值得记一下：

```ts
const type = (input.type || "general") as SubAgentType;
const description = input.description || "sub-agent task";
const prompt = input.prompt || "";
```

也就是：
- 不传 `type` 时默认是 `general`
- 不传 `description` 和 `prompt` 也不会报错，只会用默认值

这个 sub-agent 的配置来源和 skill fork 也不完全一样：
- `executeAgentTool()` 用的是 `getSubAgentConfig(type)`
- 也就是每种子代理类型都有预定义的 `systemPrompt` 和 `tools`
- 而 `executeSkillTool()` 的 fork 模式更像“按 skill 运行时结果动态生成一个子代理”

权限模式上，当前实现是：
- 如果主代理在 `plan`，子代理也继续 `plan`
- 否则直接使用 `bypassPermissions`

可以理解为：
- 主代理会“递归地”创建更小的、隔离上下文的 Agent

### 11.8 这一组的 TS / JS 语法重点

`Record<string, any>`：
- 表示一个对象
- key 是字符串
- value 先宽松地按 `any` 处理

动态导入：

```ts
await import("./skills.js")
```

表示：
- 运行到这里时再加载模块
- 常用于减少循环依赖或延迟加载

过滤数组：

```ts
this.tools.filter(t => result.allowedTools!.includes(t.name))
```

这里的 `!` 是“非空断言”：
- 告诉 TS：`allowedTools` 在这里一定存在

模板字符串：

```ts
`plan-${this.sessionId}.md`
```

用于拼接字符串，比 `+` 更清晰。

---

## 12. 这一大段最该记住的“行为主线”

从 `togglePlanMode()` 到 `// ─── Anthropic backend ─` 之前，可以按下面这条主线记忆：

1. 先提供一些“外部注入”的回调能力
- `setConfirmFn`
- `setPlanApprovalFn`

2. 提供一些“当前状态查询”和“模式切换”能力
- `togglePlanMode`
- `getPermissionMode`
- `getTokenUsage`

3. 用 `chat()` 作为主入口，进入真正的一轮 Agent 执行

4. 为子代理提供 `runOnce()` 和 `emitText()` 这样的专用支持

5. 为 REPL 和会话管理提供辅助方法
- 清空历史
- 显示费用
- 恢复会话
- 自动保存

6. 为上下文控制提供两种压缩能力
- 摘要压缩
- 本地多层压缩

7. 用 `executeToolCall()` 把普通工具、skill、plan mode、sub-agent、MCP 工具统一接到工具系统上

也就是说，这一大段代码的作用不是“和模型聊天的细节实现”，而是：

**把 Agent 的中层能力都搭好，让后面的 Anthropic / OpenAI 后端主循环可以直接调用。**

---

## 13. 这一段常见易混点速记

### `togglePlanMode()` 和 `executePlanModeTool()` 的区别

- `togglePlanMode()`：用户在 REPL 输入 `/plan` 时调用
- `executePlanModeTool()`：模型调用工具 `enter_plan_mode` / `exit_plan_mode` 时调用

### `clearHistory()` 和 `clearHistoryKeepSystem()` 的区别

- `clearHistory()`：给 REPL 用户用，清历史并重置 token 统计
- `clearHistoryKeepSystem()`：给内部 plan 流程用，清历史但保留 system prompt

### `chat()` 和 `runOnce()` 的区别

- `chat()`：主对话入口，偏主代理使用
- `runOnce()`：子代理入口，返回文本和 token 结果

### `compactConversation()` 和 `runCompressionPipeline()` 的区别

- `compactConversation()`：调用模型生成摘要，属于较重压缩
- `runCompressionPipeline()`：只在本地消息数组上做裁剪、snip、microcompact，属于较轻压缩

### `executeToolCall()` 和 `executeTool()` 的区别

- `executeToolCall()`：Agent 类内部的统一分发入口
- `executeTool()`：更底层的普通工具执行器

---

## 14. 目前阅读到这里，最适合继续往下看的方式

继续读源码时，建议带着这几个问题往下看：

1. `chatAnthropic()` / `chatOpenAI()` 如何进入真正的模型循环
2. 模型返回 tool call 后，怎样接到这里的 `executeToolCall()`
3. `checkBudget()`、`checkAndCompact()`、`contextCleared` 这些状态，如何在主循环里生效
4. 子代理结果怎样回流到主代理上下文中

如果后面继续做笔记，可以接着按下面顺序写：
- Anthropic backend
- OpenAI-compatible backend
- confirm / permission 流程
- tools.ts 工具系统
