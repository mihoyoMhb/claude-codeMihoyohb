# Claude Code From Scratch

[![GitHub stars](https://img.shields.io/github/stars/Windy3f3f3f3f/claude-code-from-scratch?style=social)](https://github.com/Windy3f3f3f3f/claude-code-from-scratch)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)](#)
[![Lines of Code](https://img.shields.io/badge/~3000_lines-minimal-green)](#)

> Build Claude Code from scratch, step by step

<p align="center">
  <a href="https://windy3f3f3f3f.github.io/claude-code-from-scratch/"><strong>📘 Read Tutorial Online →</strong></a>
  &nbsp;&nbsp;|&nbsp;&nbsp;
  <a href="./README.md">中文</a>
</p>

> 📖 **Want to understand the internals?** Companion project **[How Claude Code Works](https://github.com/Windy3f3f3f3f/how-claude-code-works)** — 12 deep-dive articles, 330K+ characters, source-level analysis of Claude Code's architecture

---

**Claude Code open-sourced 500K lines of TypeScript. Too much to read?**

This project recreates Claude Code's core architecture in **~3000 lines** — Agent Loop, tool system, 4-tier context compression, memory, skills, multi-agent — with each step comparing the real source to our simplified version.

This isn't a demo — it's a **step-by-step tutorial**. Follow along, write a few thousand lines of code yourself, and quickly grasp the essence of the best coding agent out there. No need to wade through hundreds of thousands of lines.

<video src="https://github.com/user-attachments/assets/4f6597e2-6ea3-45ae-8a6b-77662c4e9540" width="100%" autoplay loop muted playsinline></video>

## Step-by-Step Tutorial

11 chapters, from core loop to advanced capabilities. Each chapter includes real code + Claude Code source comparison. Follow along and build your own coding agent:

| Chapter | Content | Source Mapping |
|---------|---------|---------------|
| [1. Agent Loop](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/01-agent-loop) | Core loop: call LLM → execute tools → repeat | `agent.ts` ↔ `query.ts` |
| [2. Tool System](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/02-tools) | 8 tools: definition & implementation | `tools.ts` ↔ `Tool.ts` + 66 tools |
| [3. System Prompt](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/03-system-prompt) | Prompt engineering for a coding agent | `prompt.ts` ↔ `prompts.ts` |
| [4. Streaming](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/04-streaming) | Anthropic + OpenAI dual-backend streaming | `agent.ts` ↔ `api/claude.ts` |
| [5. Safety](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/05-safety) | Dangerous command detection + confirmation | `tools.ts` ↔ `permissions.ts` (52KB) |
| [6. Context](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/06-context) | Result truncation + auto-compaction | `agent.ts` ↔ `compact/` |
| [7. CLI & Sessions](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/07-cli-session) | REPL, Ctrl+C, session persistence | `cli.ts` ↔ `cli.tsx` |
| [8. Memory & Skills](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/08-memory-skills) | 4-type memory + skill template system | `memory.ts` + `skills.ts` |
| [9. Multi-Agent](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/09-multi-agent) | Sub-Agent fork-return architecture | `subagent.ts` ↔ `AgentTool/` |
| [10. Permission Rules](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/10-permission-rules) | Configurable allow/deny permission rules | `tools.ts` ↔ `permissions/` |
| [11. Comparison](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/11-whats-next) | Full comparison + extension ideas | Global |

## Quick Start

```bash
git clone https://github.com/Windy3f3f3f3f/claude-code-from-scratch.git
cd claude-code-from-scratch
npm install && npm run build
```

### API Configuration

Two backends supported, auto-detected via environment variables:

**Option 1: Anthropic Format (Recommended)**

```bash
export ANTHROPIC_API_KEY="sk-ant-xxx"
# Optional: use a proxy
export ANTHROPIC_BASE_URL="https://aihubmix.com"
```

**Option 2: OpenAI-Compatible Format**

```bash
export OPENAI_API_KEY="sk-xxx"
export OPENAI_BASE_URL="https://api.openai.com/v1"
```

**Option 3: Local Ollama (OpenAI-Compatible)**

```bash
export OPENAI_BASE_URL="http://127.0.0.1:11434/v1"
export OPENAI_API_KEY="ollama"
export MINI_CLAUDE_MODEL="pielee/qwen3-4b-thinking-2507_q8"
```

Or use the helper script:

```bash
bash ./scripts/run_with_ollama.sh
```

Default model is `claude-opus-4-6`. Customize via env var or CLI flag:

```bash
export MINI_CLAUDE_MODEL="claude-sonnet-4-6"    # env var
npm start -- --model gpt-4o                      # CLI flag (higher priority)
```

### Run

**TypeScript**

```bash
npm start                    # Interactive REPL mode (recommended)
npm start -- --resume        # Resume last session
npm start -- --yolo          # Skip safety confirmations
npm start -- --plan          # Plan mode: analyze only, no modifications
npm start -- --accept-edits  # Auto-approve file edits
npm start -- --dont-ask      # CI mode: auto-deny confirmable actions
npm start -- --max-cost 0.50 # Cost limit (USD)
npm start -- --max-turns 20  # Turn limit
```

**Python**

```bash
mini-claude-py               # Interactive REPL mode (recommended)
mini-claude-py --resume      # Resume last session
mini-claude-py --yolo        # Skip safety confirmations
mini-claude-py --plan        # Plan mode: analyze only, no modifications
mini-claude-py --accept-edits # Auto-approve file edits
mini-claude-py --dont-ask    # CI mode: auto-deny confirmable actions
mini-claude-py --max-cost 0.50 # Cost limit (USD)
mini-claude-py --max-turns 20  # Turn limit
```

Install globally to use from any directory:

**TypeScript**

```bash
npm link                     # Global install
cd ~/your-project
mini-claude                  # Launch directly
```

**Python**

```bash
cd python
pip install -e .             # Global install (editable mode)
cd ~/your-project
mini-claude-py               # Launch directly
```

### REPL Commands

| Command | Function |
|---------|----------|
| `/clear` | Clear conversation history |
| `/cost` | Show cumulative token usage and cost |
| `/compact` | Manually trigger conversation compaction |
| `/memory` | List saved memories |
| `/skills` | List available skills |
| `/<skill>` | Invoke a registered skill (e.g. `/commit`) |

## Comparison with Claude Code

| Aspect | Claude Code | Mini Claude Code |
|--------|------------|-----------------|
| Purpose | Production coding agent | Educational / minimal |
| Tools | 66+ built-in | 8 tools (6 core + skill + agent) |
| Context | 4-level compression pipeline | 4-tier compression (budget + snip + microcompact + auto-compact) |
| Permissions | 7-layer + AST analysis | 5 modes + rule config + regex detection |
| Edit Validation | 14-step pipeline | Quote normalization + uniqueness + diff output |
| Memory | 4 types + semantic recall | 4 types + keyword recall |
| Skills | 6 sources + inline/fork | 2 sources + inline/fork |
| Multi-Agent | Sub-Agent + Custom + Coordinator + Swarm | Sub-Agent (3 built-in + custom agents) |
| Budget Control | USD/turns/abort | USD + turn limits |
| Code Size | 500k+ lines | ~3000 lines |

## Core Capabilities

- **Agent Loop**: Automatically calls tools, processes results, iterates until done
- **8 Tools**: Read, write, edit (quote normalization + diff output); search files/content; execute commands; skills; sub-agents
- **Streaming**: Real-time character-by-character output, Anthropic + OpenAI backends
- **4-Tier Context Compression**: Budget trimming → stale snip → microcompact → auto-compact, zero API cost progressive space reclaim
- **5 Permission Modes**: default / plan / acceptEdits / bypassPermissions / dontAsk
- **Memory System**: 4 types (user/feedback/project/reference) with cross-session persistence
- **Skills System**: Load reusable prompt templates, supports inline injection and fork sub-agent execution
- **Multi-Agent**: Sub-Agent fork-return pattern (3 built-in + `.claude/agents/` custom types)
- **Permission Rules**: Configurable allow/deny rules in `.claude/settings.json`, 16 dangerous command patterns (incl. Windows)
- **Extended Thinking**: Anthropic extended thinking support (`--thinking`), adaptive/enabled/disabled modes
- **Budget Control**: `--max-cost` USD limit + `--max-turns` turn limit, auto-stop on exceed
- **Session Persistence**: Auto-save conversations, `--resume` to restore
- **Cross-Platform**: Windows / macOS / Linux, auto-detects shell (PowerShell / bash / zsh)
- **Error Recovery**: Exponential backoff + random jitter retry (max 3 attempts), graceful Ctrl+C

## Project Structure

```
src/
├── agent.ts        # Agent loop: streaming, 4-tier compress, budget     (1064 lines)
├── tools.ts        # Tools: 8 tools + 5 perm modes + quote fix + diff  (667 lines)
├── cli.ts          # CLI entry: args, REPL, budget flags                (336 lines)
├── memory.ts       # Memory system: 4 types + file storage + recall     (205 lines)
├── ui.ts           # Terminal output: colors, formatting, sub-agent     (187 lines)
├── skills.ts       # Skills system: discovery + inline/fork modes       (175 lines)
├── subagent.ts     # Sub-agent: 3 built-in + custom agent discovery     (172 lines)
├── system-prompt.md # System prompt template                            (81 lines)
├── prompt.ts       # System prompt: template + memory/skill/agent       (76 lines)
├── session.ts      # Session persistence: save/load/list                (63 lines)
├── frontmatter.ts  # Shared YAML frontmatter parser                     (41 lines)
                                                          Total: ~3067 lines
```

## Related Projects

- **[how-claude-code-works](https://github.com/Windy3f3f3f3f/how-claude-code-works)** — Deep dive into Claude Code's architecture (12 articles, 330K+ characters)

## License

MIT
