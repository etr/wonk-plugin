# wonk-plugin

Claude Code plugin for [wonk](https://github.com/etr/wonk) — structure-aware
code search that cuts LLM token burn.

## What it does

This plugin connects wonk to Claude Code so the agent can search your codebase
using ranked, deduplicated results instead of raw grep output. It provides:

- **CLI mode (default)** — runs `wonk` commands via the Bash tool. No background
  process needed.
- **MCP mode (optional)** — registers `wonk mcp serve` so all 22 wonk tools
  appear as native tools. Requires a persistent MCP server process.
- **Skill** — teaches the agent when to prefer wonk over Grep/Glob and how to
  pick the right wonk command for each task
- **Session hook** — runs `wonk update -q` on session start to keep the index
  fresh

## Prerequisites

Install wonk:

```sh
curl -fsSL https://raw.githubusercontent.com/etr/wonk/main/install.sh | sh
```

See the [wonk repo](https://github.com/etr/wonk) for other installation
methods.

## Install the plugin

Via the [Groundwork Marketplace](https://github.com/etr/groundwork-marketplace):

```sh
claude plugin marketplace add https://github.com/etr/groundwork-marketplace
claude plugin add wonk
```

Or directly:

```sh
claude plugin add https://github.com/etr/wonk-plugin
```

## Configuration

After installing, run the setup skill (`/wonk:setup-wonk`) to choose between
CLI and MCP modes. The preference is stored in `.claude/wonk.local.md`:

```yaml
---
mode: cli   # or "mcp"
---
```

CLI mode is recommended and used by default when no configuration exists.

## Supported languages

TypeScript, JavaScript, Python, Rust, Go, Java, C, C++, Ruby, PHP, C#.

## License

MIT
