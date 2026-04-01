# Claude Code From Scratch

[![GitHub stars](https://img.shields.io/github/stars/Windy3f3f3f3f/claude-code-from-scratch?style=social)](https://github.com/Windy3f3f3f3f/claude-code-from-scratch)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)](#)
[![Lines of Code](https://img.shields.io/badge/~1300_lines-minimal-green)](#)

> Build Claude Code from scratch, step by step

<p align="center">
  <a href="https://windy3f3f3f3f.github.io/claude-code-from-scratch/"><strong>📘 Read Tutorial Online →</strong></a>
  &nbsp;&nbsp;|&nbsp;&nbsp;
  <a href="./README.md">中文</a>
</p>

> 📖 **Want to understand the internals?** Companion project **[How Claude Code Works](https://github.com/Windy3f3f3f3f/how-claude-code-works)** — 12 deep-dive articles, 338K characters, source-level analysis of Claude Code's architecture

---

**Recreate Claude Code's core capabilities in ~1300 lines of TypeScript.** This isn't a demo — it's a step-by-step tutorial where each chapter compares Claude Code's real source with our simplified implementation, helping you truly understand how coding agents work.

<video src="https://github.com/Windy3f3f3f3f/claude-code-from-scratch/raw/main/demo.mp4" width="100%" autoplay loop muted playsinline></video>

## Step-by-Step Tutorial

8 chapters, from core loop to complete CLI. Each chapter includes real code + Claude Code source comparison:

| Chapter | Content | Source Mapping |
|---------|---------|---------------|
| [1. Agent Loop](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/01-agent-loop) | Core loop: call LLM → execute tools → repeat | `agent.ts` ↔ `query.ts` |
| [2. Tool System](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/02-tools) | 6 tools: definition & implementation | `tools.ts` ↔ `Tool.ts` + 66 tools |
| [3. System Prompt](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/03-system-prompt) | Prompt engineering for a coding agent | `prompt.ts` ↔ `prompts.ts` |
| [4. Streaming](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/04-streaming) | Anthropic + OpenAI dual-backend streaming | `agent.ts` ↔ `api/claude.ts` |
| [5. Safety](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/05-safety) | Dangerous command detection + confirmation | `tools.ts` ↔ `permissions.ts` (52KB) |
| [6. Context](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/06-context) | Result truncation + auto-compaction | `agent.ts` ↔ `compact/` |
| [7. CLI & Sessions](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/07-cli-session) | REPL, Ctrl+C, session persistence | `cli.ts` ↔ `cli.tsx` |
| [8. Comparison](https://windy3f3f3f3f.github.io/claude-code-from-scratch/#/docs/08-whats-next) | Full comparison + extension ideas | Global |

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

Default model is `claude-opus-4-6`. Customize via env var or CLI flag:

```bash
export MINI_CLAUDE_MODEL="claude-sonnet-4-6"    # env var
npm start -- --model gpt-4o                      # CLI flag (higher priority)
```

### Run

```bash
npm start                    # Interactive REPL mode (recommended)
npm start -- --resume        # Resume last session
npm start -- --yolo          # Skip safety confirmations
```

Install globally to use from any directory:

```bash
npm link                     # Global install
cd ~/your-project
mini-claude                  # Launch directly
```

### REPL Commands

| Command | Function |
|---------|----------|
| `/clear` | Clear conversation history |
| `/cost` | Show cumulative token usage and cost |
| `/compact` | Manually trigger conversation compaction |

## Comparison with Claude Code

| Aspect | Claude Code | Mini Claude Code |
|--------|------------|-----------------|
| Purpose | Production coding agent | Educational / minimal |
| Tools | 66+ built-in | 6 core tools |
| Context | 4-level compression | Token tracking + auto-compact |
| Streaming | Ink/React rendering | Native stream printing |
| Security | 5-layer permission system | Basic command confirmation |
| Code Size | 500k+ lines | ~1300 lines |

## Core Capabilities

- **Agent Loop**: Automatically calls tools, processes results, iterates until done
- **6 Core Tools**: Read, write, edit files; search files/content; execute commands
- **Streaming**: Real-time character-by-character output, Anthropic + OpenAI backends
- **Context Management**: Automatic token tracking with conversation compaction
- **Safe by Default**: Dangerous commands require confirmation; `--yolo` to skip
- **Session Persistence**: Auto-save conversations, `--resume` to restore
- **Error Recovery**: Exponential backoff retry on rate limits, graceful Ctrl+C

## Project Structure

```
src/
├── cli.ts      # CLI entry: args, REPL, Ctrl+C         (209 lines)
├── agent.ts    # Agent loop: streaming, retry, compact  (620 lines)
├── tools.ts    # Tool definitions: 6 tools + truncation (304 lines)
├── prompt.ts   # System prompt: template + env inject   (65 lines)
├── session.ts  # Session persistence: save/load/list    (63 lines)
└── ui.ts       # Terminal output: colors, formatting    (102 lines)
                                              Total: ~1300 lines
```

## Architecture

```
User Input
  │
  ▼
┌─────────────────────────────────────┐
│          Agent Loop                 │
│                                     │
│  Messages → API (stream) → Output  │
│       ▲                   │         │
│       │              ┌────┴───┐     │
│       │              │  Text  │     │
│       │              │ Tools  │     │
│       │              └────┬───┘     │
│       │                   │         │
│       │   ┌────────┐┌────▼───┐     │
│       │   │Truncate│←│Execute│     │
│       │   └────────┘└────┬───┘     │
│       │                   │         │
│       │   ┌───────────────▼───┐     │
│       └───│Token Track+Compact│     │
│           └───────────────────┘     │
└─────────────────────────────────────┘
  │
  ▼
Task Complete → Auto-save Session
```

## Related Projects

- **[how-claude-code-works](https://github.com/Windy3f3f3f3f/how-claude-code-works)** — Deep dive into Claude Code's architecture (12 articles, 338K characters)

## License

MIT
