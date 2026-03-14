---
name: using-wonk
description: >
  This skill should be used when the agent needs to search code, find where a
  symbol is defined, trace references or call sites, explore import
  dependencies, list project structure, trace call graphs, analyze blast radius,
  detect changes, or explore execution flows — tasks like "find the definition
  of X", "who calls this function", "what does this file import", "search
  the codebase for pattern", "what's the blast radius of changing X", or
  "show me the execution flow". Wonk provides structure-aware code search via
  CLI commands (default) or MCP tools. Prefer wonk over Grep and Glob for code
  exploration in supported languages (TypeScript, JavaScript, Python, Rust, Go,
  Java, C, C++, Ruby, PHP, C#).
---

# Wonk — Structure-Aware Code Search

Wonk indexes codebases with Tree-sitter and returns ranked, grouped search
results. It eliminates token waste from hundreds of unranked grep lines by
classifying results into tiers: definitions first, then call sites, imports,
other matches, comments, and tests.

## Mode Selection

Check `.claude/wonk.local.md` in the project root for the configured mode.
Read the YAML frontmatter `mode` field.

- If the file does not exist or has no `mode` field → **CLI mode** (default)
- `mode: cli` → **CLI mode**
- `mode: mcp` → **MCP mode** (wonk MCP tools are available as `wonk_*`)

## When to Use Wonk

**Prefer wonk for all code search tasks.** Fall back to Grep/Glob only when:
- The project uses a language wonk does not support
- Searching non-code files (configs, docs, data files, logs)
- Wonk is not installed

## Tool Selection Guide

Pick the right command/tool for the task:

| Task | CLI Command | MCP Tool |
|------|-------------|----------|
| "Find where X is defined" | `wonk sym X` | `wonk_sym` |
| "Show me the source of X" | `wonk show X` (batch: `wonk show X,Y,Z`) | `wonk_show` |
| "Who calls X?" | `wonk callers X` | `wonk_callers` |
| "How does X reach Y?" | `wonk callpath X Y` | `wonk_callpath` |
| "How does this flow work?" | `wonk flows main` | `wonk_flows` |
| "Everything about X" | `wonk context X` (ONE call: def + callers + callees + imports) | `wonk_context` |
| "Summarize module architecture" | `wonk summary path --depth 1` | `wonk_summary` |
| "List symbols in a file" | `wonk summary path` | `wonk_summary` |
| "Search for keyword/regex" | `wonk search pattern` | `wonk_search` |
| "Semantic / NL search" | `wonk ask "query"` (requires embeddings) | `wonk_ask` |
| "List files referencing X" | `wonk ref X` | `wonk_ref` |
| "Check function signature" | `wonk sig X` (fastest, no bodies) | `wonk_sig` |

## CLI Mode (Default)

Run wonk commands via the **Bash** tool. Always use `--format toon -q` for
compact, LLM-friendly output without stderr hints.

### CLI Reference

```
wonk search <pattern> [--regex] [-i] [--raw] [--smart] [--semantic] [--budget N] [-- path1 path2]
wonk sym <name> [--kind <kind>] [--file <path>] [--exact] [--budget N]
wonk ref <name> [--budget N] [-- path1 path2]
wonk show <name> [--file <path>] [--kind <kind>] [--exact] [--shallow] [--budget N]
wonk sig <name> [--budget N]
wonk callers <name> [--depth N] [--min-confidence N] [--budget N]
wonk callees <name> [--depth N] [--min-confidence N] [--budget N]
wonk callpath <from> <to> [--min-confidence N] [--budget N]
wonk context <name> [--file <path>] [--kind <kind>] [--min-confidence N] [--budget N]
wonk summary <path> [--detail outline|rich] [--depth N] [--recursive] [--budget N]
wonk flows [entry] [--from <file>] [--depth N] [--branching N] [--min-confidence N] [--budget N]
wonk blast <symbol> [--direction upstream|downstream] [--depth N] [--include-tests] [--min-confidence N] [--budget N]
wonk changes [--scope unstaged|staged|all|compare] [--base <ref>] [--blast] [--flows] [--min-confidence N] [--budget N]
wonk deps <file> [--budget N]
wonk rdeps <file> [--budget N]
wonk ask "<query>" [--from <file>] [--to <file>] [--budget N]
wonk cluster <path> [--top N] [--budget N]
wonk impact <file> [--since <commit>] [--budget N]
wonk init [--local]
wonk update [--force]
wonk status
wonk repos list
```

### CLI Examples

```bash
# Find definition of a symbol
wonk sym parseConfig --format toon -q

# Show source code of multiple symbols
wonk show "parseConfig,validateInput,main" --format toon -q

# Full context for a symbol (definition + callers + callees + imports)
wonk context handleRequest --format toon -q --budget 4000

# Module architecture overview
wonk summary src/api --depth 1 --format toon -q

# Search for a pattern
wonk search "TODO|FIXME" --regex --format toon -q

# Call chain between two symbols
wonk callpath main handleRequest --format toon -q

# Blast radius of a change
wonk blast parseConfig --format toon -q

# Changed symbols with blast radius
wonk changes --blast --format toon -q
```

## MCP Mode

When `mode: mcp` is configured, wonk MCP tools are available as native tools
with the `wonk_` prefix. Use them directly — they accept structured parameters
and return JSON.

The 22 available MCP tools: `wonk_search`, `wonk_sym`, `wonk_ref`, `wonk_sig`,
`wonk_show`, `wonk_deps`, `wonk_rdeps`, `wonk_callers`, `wonk_callees`,
`wonk_callpath`, `wonk_summary`, `wonk_flows`, `wonk_blast`, `wonk_changes`,
`wonk_context`, `wonk_ask`, `wonk_cluster`, `wonk_impact`, `wonk_init`,
`wonk_update`, `wonk_status`, `wonk_repos`.

## Efficient Patterns

**DO:** Use 2-3 targeted calls and stop.
- `wonk sym` → `wonk show` → done
- `wonk context` → done (replaces 4+ separate calls)
- `wonk callpath from to` → done (replaces 6+ show calls)
- `wonk show X,Y,Z` → done (batch lookup)
- `wonk summary path --depth 1` → done (architecture overview)

**DON'T:** Chain 5+ show calls to manually trace a call path.
Each successive call adds all previous results to context → quadratic token growth.

## Handling Empty Results

When a command returns 0 results, it includes diagnostic hints:
- Similar symbol name suggestions
- Advice to remove file/kind filters for broader results
- Suggestion to try `wonk search` for text-based matching

Follow the hints rather than retrying with slightly different parameters.

## Handling Truncation

When results are truncated, the response includes a hint with
"Showing N of M" and truncation metadata.
- **wonk show auto-shallow:** Large container types (class, struct, trait, enum,
  interface) automatically fall back to shallow mode when their full body exceeds
  the budget. Results include `auto_shallow` so you know the body was replaced
  with signature + child signatures.
- **wonk show returns complete source** — you do not need to Read the same file afterward.
- Results are sorted by relevance; the most important matches are shown first.

## Error Handling

- **Wonk not installed:** Fall back to Grep/Glob. Do not attempt installation.
- **Unsupported language:** Fall back to Grep/Glob (see Supported Languages below).
- **No results:** Follow the diagnostic hints in the response. If still
  empty, fall back to Grep.
- **Index stale:** The session hook runs `wonk update` automatically at start.
  If files changed mid-session, run `wonk init` (CLI) or use `wonk_init` (MCP)
  to refresh the index.

## Supported Languages

TypeScript (.ts), TSX (.tsx), JavaScript (.js, .jsx), Python (.py),
Rust (.rs), Go (.go), Java (.java), C (.c, .h), C++ (.cpp, .cc, .hpp),
Ruby (.rb), PHP (.php), C# (.cs).
