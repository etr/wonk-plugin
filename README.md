# wonk-plugin

Claude Code plugin for [wonk](https://github.com/etr/wonk) — structure-aware
code search that cuts LLM token burn.

## What it does

This plugin connects wonk to Claude Code so the agent can search your codebase
using ranked, deduplicated results instead of raw grep output. It provides:

- **MCP server** — registers `wonk mcp serve` so all 9 wonk tools
  (`wonk_search`, `wonk_sym`, `wonk_ref`, `wonk_sig`, `wonk_ls`, `wonk_deps`,
  `wonk_rdeps`, `wonk_status`, `wonk_init`) appear as native tools
- **Skill** — teaches the agent when to prefer wonk over Grep/Glob and how to
  handle errors and unsupported languages
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

```sh
claude plugin add etr/wonk-plugin
```

## Supported languages

TypeScript, JavaScript, Python, Rust, Go, Java, C, C++, Ruby, PHP.

## License

MIT
