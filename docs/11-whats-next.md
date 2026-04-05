# 11. 架构对比与下一步

## 完整架构对比

| 组件 | Claude Code | mini-claude | 差异 |
|------|------------|-------------|------|
| **Agent Loop** | 7 种 continue reason | 只检查 tool_use | 简化循环控制 |
| **工具数量** | 66+ 工具 | 10 个工具（6 核心 + skill + agent + 2 plan mode） | 去掉特化工具 |
| **工具执行** | 并发执行多个工具调用 | 串行逐个执行 | 简化并发控制 |
| **API 后端** | Anthropic only | Anthropic + OpenAI 兼容 | 多了 OpenAI |
| **System Prompt** | static/dynamic 分界 + API 缓存 | 无缓存优化 | 去掉缓存 |
| **权限系统** | 7 层 + AST 分析 + 8 级规则源 | 5 模式 + 规则配置 + 正则 + 确认 | 层次对齐 |
| **上下文管理** | 4 级压缩流水线 | 4 层（budget + snip + microcompact + 摘要） | 架构对齐 |
| **记忆系统** | 4 类型 + 语义召回 + MEMORY.md 索引 | 4 类型 + 关键词召回 + MEMORY.md | 去掉语义匹配 |
| **技能系统** | 6 源 + 懒加载 + inline/fork | 2 源 + 预加载 + inline/fork | 去掉高级加载 |
| **多 Agent** | Sub-Agent + 自定义 + Coordinator + Swarm | Sub-Agent（3 内置 + 自定义） | 去掉 Coordinator/Swarm |
| **预算控制** | USD/轮次/abort 三维预算 | USD + 轮次限制 | 去掉 abort signal |
| **编辑验证** | 14 步流水线 | 引号容错 + 唯一性 + diff 输出 | 保留核心步骤 |

## 文件映射表

| mini-claude (TypeScript) | mini-claude (Python) | Claude Code 源码 | 说明 |
|------------|------------|-------------------|------|
| `src/agent.ts` | `python/mini_claude/agent.py` | `src/query.ts` + `src/QueryEngine.ts` | Agent 循环 + 会话管理 |
| `src/tools.ts` | `python/mini_claude/tools.py` | `src/Tool.ts` + `src/tools/` (66 个目录) | 工具定义与执行 |
| `src/prompt.ts` | `python/mini_claude/prompt.py` | `src/constants/prompts.ts` + `src/utils/claudemd.ts` | Prompt 构造 |
| `src/cli.ts` | `python/mini_claude/__main__.py` | `src/entrypoints/cli.tsx` + `src/commands/` | 入口与命令 |
| `src/ui.ts` | `python/mini_claude/ui.py` | `src/components/` (React/Ink 组件) | UI 渲染 |
| `src/session.ts` | `python/mini_claude/session.py` | `src/utils/sessionStorage.ts` + `src/history.ts` | 会话持久化 |
| `src/memory.ts` | `python/mini_claude/memory.py` | `src/utils/memory.ts` + 系统 prompt 注入 | 记忆系统 |
| `src/skills.ts` | `python/mini_claude/skills.py` | `src/utils/skills.ts` + `src/tools/SkillTool/` | 技能系统 |
| `src/subagent.ts` | `python/mini_claude/subagent.py` | `src/tools/AgentTool/` (built-in types) | 子 Agent 类型配置 |

## 我们没实现的

### Hooks（钩子系统）

Claude Code 有 25 种 hook 事件、6 种 hook 类型，可在工具执行前后插入自定义逻辑——拦截危险操作、记录审计日志、自动运行 lint 检查。它是 Claude Code 从"工具"变成"平台"的关键机制。

我们没实现的原因：核心挑战不在于"调一个函数"，而在于 hook 的发现与加载、错误隔离、stdin/stdout JSON 数据协议。这些工程细节约 500-800 行，但对理解 agent 原理没有帮助。

### Coordinator / Swarm 多 Agent 模式

我们实现了 Sub-Agent（fork-return）。Claude Code 还有两种模式：**Coordinator** 把大任务拆分给多个专业 Agent，**Swarm** 让多个 Agent 对等通信、并行探索。两种模式解决的是单 Agent 上下文不够时的任务分解问题。

没实现的原因：核心挑战是任务分解准确性和 Agent 间通信协议设计，更多是 prompt engineering 问题而非代码架构问题。实现本身不复杂，但要真正好用需要大量 prompt 调优。

### MCP（Model Context Protocol）

MCP 让 agent 运行时动态加载外部工具——连接数据库、Slack、JIRA 等，声明一个服务器地址即可，不用改源码。这是 agent 工具来源的插件化架构。

没实现的原因：涉及子进程管理、JSON-RPC 通信、工具 schema 动态转换、生命周期管理，约 400-600 行。教程重点是让读者在 `tools.ts` 直接看到每个工具的实现——直接比间接更有教学价值。

### LSP 集成

LSP 让 agent 在编辑文件后毫秒级获得类型错误反馈，而不需要等完整的编译/测试周期。在大型项目中，这能把修复一个 bug 所需的循环次数减少 30-50%。

没实现的原因：需要管理 LSP 服务器进程、实现客户端协议（初始化握手、能力协商、增量同步），1000+ 行且依赖对 LSP 协议的深入理解。通过 shell 命令（`tsc --noEmit`、`python -m py_compile`）获得错误反馈，对教程场景已经足够。

### 并发工具执行

Claude Code 的 `StreamingToolExecutor` 对只读工具（`read_file`、`grep_search`）并行执行，在模型一次读取 3-5 个文件时速度提升 2-3 倍。每个工具有 `isConcurrencySafe()` 方法声明自己是否可以安全并行。

没实现的原因：`Promise.all()` 只有一行，但工具类型系统和并发错误处理约 200 行，且与我们的 JSON + switch/case 设计不兼容。下方"扩展方向"给出了一个简化版实现。

### Prompt Caching

Anthropic API 支持缓存系统提示词——Claude Code 把不变的部分（角色定义、工具规范）放前面，变化的部分（git 状态、当前文件）放后面，缓存命中可将输入 token 成本降低 90%。

没实现的原因：代码改动极小（20-30 行），但需要仔细设计提示词分区策略。如果你的 agent 要上线，这应该是第一个加上的优化。

### Bash AST 安全分析

Claude Code 用 tree-sitter 解析 shell 命令的 AST，进行 23 项静态安全检查，能分析出管道组合中的危险命令——这是纯正则做不到的。

没实现的原因：tree-sitter 是 C/C++ 原生库，需要 `node-gyp` 编译环境，环境障碍太高。正则匹配覆盖了 80% 的常见危险模式，教程场景风险可接受。

## 渐进式增强路线图

### 第一阶段：性能与成本优化（1-2 天）

| 增强项 | 解决的问题 | 预计代码量 |
|--------|-----------|-----------|
| Prompt Caching | 重复发送系统提示词浪费 token | ~30 行 |
| 并发工具执行 | 多个只读工具串行等待 | ~100 行 |

**Prompt Caching** 是投入产出比最高的优化：给系统提示词的静态部分加上 `cache_control: { type: "ephemeral" }` 标记，多轮对话中节省 50%+ 的输入 token 成本。

### 第二阶段：可扩展性（3-5 天）

| 增强项 | 解决的问题 | 预计代码量 |
|--------|-----------|-----------|
| Hook 系统 | 定制 agent 行为需要改源码 | ~300 行 |
| MCP 工具支持 | 添加新工具需要改 tools.ts | ~500 行 |
| Tool 类型系统 | switch/case 不能扩展到 20+ 工具 | ~200 行 |

核心转变是**从硬编码到插件化**。当前 switch/case 在 10 个工具时没问题，但超过 20 个就需要引入 Tool 接口（或 Python 的 Protocol/ABC），让每个工具成为独立模块。

### 第三阶段：可靠性与安全（1-2 周）

| 增强项 | 解决的问题 | 预计代码量 |
|--------|-----------|-----------|
| 7 种错误恢复策略 | 当前遇到错误直接崩溃 | ~400 行 |
| Bash AST 安全分析 | 正则匹配漏检复杂危险命令 | ~600 行 |
| 语义记忆召回 | 关键词匹配召回准确率低 | ~150 行 |

Claude Code 的 `query.ts` 有 1728 行，大部分是边缘情况处理：Prompt Too Long 时自动压缩重试、API 过载时指数退避、工具失败时把错误反馈给模型让它自修复。

### 第四阶段：高级 Agent 能力（2-4 周）

| 增强项 | 解决的问题 | 预计代码量 |
|--------|-----------|-----------|
| Coordinator 模式 | 大任务超出单 Agent 上下文容量 | ~500 行 |
| Swarm 模式 | 探索性任务需要多路径并行 | ~600 行 |
| LSP 集成 | 类型错误只能通过编译发现 | ~1000 行 |
| ToolSearch 延迟加载 | 66+ 工具描述占满提示词 | ~200 行 |

## 扩展方向

### 1. 并发工具执行

标记只读工具为并发安全，一次返回多个工具调用时并行执行：

<!-- tabs:start -->
#### **TypeScript**
```typescript
const SAFE = new Set(["read_file", "list_files", "grep_search"]);
const [safe, unsafe] = partition(toolCalls, tc => SAFE.has(tc.name));
const safeResults = await Promise.all(safe.map(tc => executeTool(tc.name, tc.input)));
const unsafeResults = [];
for (const tc of unsafe) unsafeResults.push(await executeTool(tc.name, tc.input));
```
#### **Python**
```python
SAFE = {"read_file", "list_files", "grep_search"}
safe = [tc for tc in tool_calls if tc["name"] in SAFE]
unsafe = [tc for tc in tool_calls if tc["name"] not in SAFE]
safe_results = await asyncio.gather(*(execute_tool(tc["name"], tc["input"]) for tc in safe))
unsafe_results = [await execute_tool(tc["name"], tc["input"]) for tc in unsafe]
```
<!-- tabs:end -->

注意：合并结果时要按原始调用顺序排列，否则模型会混淆哪个结果对应哪个调用。

### 2. Hooks 系统

最简单的方案是 command hook——在 `executeTool` 前 spawn shell 子进程，通过 stdin JSON 传入工具信息，解析 stdout JSON 决定 allow/deny。

配置示例：
```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "run_shell", "command": "./hooks/pre-shell.sh" }
    ]
  }
}
```

核心逻辑：遍历匹配的 hook，spawn 子进程传 JSON，根据 `{"action": "allow"}` / `{"action": "deny", "reason": "..."}` 决定是否继续执行。约 300 行，最耗时的是子进程的超时和 crash 处理。

### 3. MCP 工具支持

<!-- tabs:start -->
#### **TypeScript**
```typescript
import { MCPClient } from "@anthropic-ai/mcp";
const client = new MCPClient("npx @modelcontextprotocol/server-filesystem /tmp");
const mcpTools = await client.listTools();
toolDefinitions.push(...mcpTools.map(convertToToolDef));
```
#### **Python**
```python
from anthropic_mcp import MCPClient
client = MCPClient("npx @modelcontextprotocol/server-filesystem /tmp")
mcp_tools = await client.list_tools()
tool_definitions.extend(convert_to_tool_def(t) for t in mcp_tools)
```
<!-- tabs:end -->

在 `executeTool` 的 `default` 分支检查是否是 MCP 工具并转发执行。使用现成 MCP SDK 约 200 行，自己实现 JSON-RPC 约 500 行。

### 4. 语义记忆召回

用小模型替代关键词匹配，从记忆清单中选出相关条目：

<!-- tabs:start -->
#### **TypeScript**
```typescript
async function semanticRecall(query: string): Promise<MemoryEntry[]> {
  const manifest = memories.map(m => `${m.filename}: ${m.description}`).join("\n");
  const response = await sideQuery(`Which of these memories are relevant to: ${query}\n${manifest}`);
  return parseRelevantFilenames(response);
}
```
#### **Python**
```python
async def semantic_recall(query: str) -> list[dict]:
    manifest = "\n".join(f"{m['filename']}: {m['description']}" for m in memories)
    response = await side_query(f"Which of these memories are relevant to: {query}\n{manifest}")
    return parse_relevant_filenames(response)
```
<!-- tabs:end -->

代价是每次会话开始多一次 haiku API 调用。约 150 行。

### 5. 错误自修复

把工具执行错误作为工具结果反馈给模型，而不是中断循环。模型经常能自己修复：路径拼错换路径、命令参数错了改参数。

```typescript
try {
  result = await executeToolImpl(name, input);
} catch (e) {
  result = `Error: ${e.message}\n\nPlease try a different approach.`;
}
// 把 result 作为 tool_result 返回给模型
```

约 50-80 行，但能显著提升 agent 实际可用性——这是 Claude Code 最聪明的设计之一。

## 核心洞察

**1. Agent 的本质是一个 while 循环**

```
while true:
    response = llm.call(messages)
    if no tool_calls in response: break
    for tool_call in response.tool_calls:
        result = execute(tool_call)
        messages.append(result)
```

所有的复杂性——权限、上下文管理、记忆、多 Agent——都是围绕这个循环的增强和防护。

**2. 提示词是最便宜的代码**

系统提示词里的一句话，效果等同于一个 if 语句，实现成本是 0 行代码。agent 开发中很多行为问题的最优解不是写更多代码，而是写更好的提示词——更灵活、更容易修改、非技术人员也能读懂。

**3. 工具设计决定能力上限**

让模型做它擅长的（理解意图、生成代码），让工具做模型不擅长的（精确字符串匹配、文件系统操作、进程管理）。`edit_file` 是典型：模型生成要替换的内容，工具负责在文件中精确定位和替换。

**4. 上下文管理是 agent 的"记忆力"**

上下文管理之于 agent，就像内存管理之于操作系统——用有限资源提供"无限"错觉。4 层压缩流水线让 agent 在有限窗口中保持对长对话的记忆。

**5. 安全不是事后补丁**

权限检查是 agent 循环的一个步骤，不是外挂的 middleware。没有任何工具可以绕过它。更重要的是 fail-closed 设计：新工具如果忘记声明权限级别，被自动当作"需要确认"处理——系统通过默认值保证安全。

**6. 从 3000 行到 50 万行的差距在于边缘情况**

Claude Code 多出来的代码大多是：各运行环境兼容性、网络和 API 不可靠性、用户输入多样性、企业级审计和访问控制。这些"无聊"的代码不会出现在架构图中，却是工具能否在真实世界可靠运行的关键。从原型到产品，80% 的距离在这里。

**7. LLM 与代码的协作边界**

构建 coding agent 最核心的能力：设计好 LLM 和代码之间的协作边界。哪些让 LLM 决定，哪些让代码决定——边界划得好，agent 既灵活又可靠。我们在教程里每个设计决策都体现了这个原则：模型决定"做什么"，代码确保"安全地做"。

## 交叉引用

想深入了解 Claude Code 各模块的设计原理？参考兄弟项目的详细文档：

| 主题 | 本教程 | how-claude-code-works |
|------|--------|----------------------|
| Agent 循环 | [Ch1: Agent Loop](docs/01-agent-loop.md) | [系统主循环](https://windy3f3f3f3f.github.io/how-claude-code-works/#/docs/02-agent-loop) |
| 工具系统 | [Ch2: 工具系统](docs/02-tools.md) | [工具系统](https://windy3f3f3f3f.github.io/how-claude-code-works/#/docs/04-tool-system) |
| 上下文管理 | [Ch6: 上下文管理](docs/06-context.md) | [上下文工程](https://windy3f3f3f3f.github.io/how-claude-code-works/#/docs/03-context-engineering) |
| 权限安全 | [Ch5: 权限与安全](docs/05-safety.md) | [权限与安全](https://windy3f3f3f3f.github.io/how-claude-code-works/#/docs/10-permission-security) |
| 记忆系统 | [Ch8: 记忆与技能](docs/08-memory-skills.md) | [记忆系统](https://windy3f3f3f3f.github.io/how-claude-code-works/#/docs/08-memory-system) |
| 多 Agent | [Ch9: 多 Agent](docs/09-multi-agent.md) | [多 Agent 架构](https://windy3f3f3f3f.github.io/how-claude-code-works/#/docs/07-multi-agent) |

---

## 结语

~3400 行代码（TS）/ ~2900 行（Python），11 个文件，覆盖了一个 coding agent 的核心组件和进阶能力：

**核心组件：** Agent Loop、工具系统（10 工具 + 引号容错 + diff 输出）、System Prompt（Markdown 模板 + 环境注入）、流式输出（Anthropic + OpenAI 双后端）、权限安全（5 模式 + 正则 + 确认）、上下文管理（4 层压缩）、CLI / 会话（REPL + JSON 持久化）

**进阶能力：** 记忆系统、技能系统、多 Agent（Sub-Agent + 3 内置类型 + 自定义）、权限规则（settings.json）、Plan Mode、预算控制

Claude Code 50 万行里的大量代码是边缘情况处理和企业级可靠性。但核心 agent 能力——理解用户意图 → 调用工具操作代码 → 迭代直到完成——就是这 ~3400 行的事。

现在你有了一个功能丰富的 coding agent，也理解了它背后每一行代码的设计意图。去扩展它吧。
