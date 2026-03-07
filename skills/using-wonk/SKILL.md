---
name: using-wonk
description: >
  This skill should be used when the agent needs to search code, find where a
  symbol is defined, trace references or call sites, explore import
  dependencies, list project structure, trace call graphs, analyze blast radius,
  detect changes, or explore execution flows — tasks like "find the definition
  of X", "who calls this function", "what does this file import", "search
  the codebase for pattern", "what's the blast radius of changing X", or
  "show me the execution flow". Wonk provides 22 MCP tools (wonk_search,
  wonk_sym, wonk_ref, wonk_sig, wonk_show, wonk_deps, wonk_rdeps,
  wonk_callers, wonk_callees, wonk_callpath, wonk_summary, wonk_flows,
  wonk_blast, wonk_changes, wonk_context, wonk_ask, wonk_cluster, wonk_impact,
  wonk_init, wonk_update, wonk_status, wonk_repos) that return ranked,
  deduplicated results optimized for LLM agents. Prefer wonk tools over Grep
  and Glob for code exploration in supported languages (TypeScript, JavaScript,
  Python, Rust, Go, Java, C, C++, Ruby, PHP, C#).
---

# Wonk — Structure-Aware Code Search

Wonk indexes codebases with Tree-sitter and returns ranked, grouped search
results. It eliminates token waste from hundreds of unranked grep lines by
classifying results into tiers: definitions first, then call sites, imports,
other matches, comments, and tests.

## When to Use Wonk

**Prefer wonk tools for all code search tasks.** Fall back to Grep/Glob only when:
- The project uses a language wonk does not support
- Searching non-code files (configs, docs, data files, logs)
- Wonk tools are not available in the current session

## Tool Selection Guide

Pick the right tool for the task:

| Task | Best Tool | NOT this |
|------|-----------|----------|
| "Find where X is defined" | `wonk_sym` | ~~wonk_search~~ |
| "Show me the source of X" | `wonk_show` (batch: `name="X,Y,Z"`) | ~~multiple wonk_show calls~~ |
| "Who calls X?" | `wonk_callers` (or `wonk_context` for everything) | ~~wonk_show → read body → manual trace~~ |
| "How does X reach Y?" | `wonk_callpath(from="X", to="Y")` | ~~chained wonk_show calls~~ |
| "How does this flow work?" | `wonk_flows(entry="main")` | ~~wonk_show → wonk_show → wonk_show~~ |
| "Everything about X" | `wonk_context` (ONE call: def + callers + callees + imports) | ~~wonk_sym + wonk_callers + wonk_callees + wonk_ref~~ |
| "Summarize module architecture" | `wonk_summary(path, depth=1)` — returns top-level symbols with doc comments + import edges | ~~Glob + multiple Read calls~~ |
| "List symbols in a file" | `wonk_summary(path)` — compact outline with doc comments | ~~Grep for definitions~~ |
| "Search for pattern" | `wonk_search` | Only use Grep for non-code files |
| "Check function signature" | `wonk_sig` (fastest, no bodies) | ~~wonk_show~~ |

## Efficient Patterns

**DO:** Use 2-3 targeted wonk calls and stop.
- `wonk_sym` → `wonk_show` → done
- `wonk_context` → done (replaces 4+ separate calls)
- `wonk_callpath(from, to)` → done (replaces 6+ show calls)
- `wonk_show(name="main,parse,validate")` → done (batch lookup)
- `wonk_summary(path="src/module", depth=1)` → done (architecture overview with symbols, doc comments + imports)

**DON'T:** Chain 5+ wonk_show calls to manually trace a call path.
Each successive call adds all previous results to context → quadratic token growth.

## Handling Empty Results

When a tool returns 0 results, it includes diagnostic hints:
- Similar symbol name suggestions
- Advice to remove file/kind filters for broader results
- Suggestion to try wonk_search for text-based matching

Follow the hints rather than retrying with slightly different parameters.

## Handling Truncation

When results are truncated, the response includes a `hint` field.
- **wonk_show auto-shallow:** Large container types (class, struct, trait, enum,
  interface) automatically fall back to shallow mode when their full body exceeds
  the budget. Results include `"auto_shallow": true` so you know the body was
  replaced with signature + child signatures.
- For other tools: `"Increase budget to see more results."`

If auto-shallow still doesn't fit, use `shallow:true` explicitly or increase the budget.

## Error Handling

- **Wonk tools unavailable:** Fall back to Grep/Glob. Do not attempt installation.
- **Unsupported language:** Fall back to Grep/Glob (see Supported Languages below).
- **No results:** Follow the diagnostic hints in the response. If still
  empty, fall back to Grep.
- **Index stale:** The session hook runs `wonk update` automatically at start.
  If files changed mid-session, use the `wonk_init` tool to refresh the index.

## Supported Languages

TypeScript (.ts), TSX (.tsx), JavaScript (.js, .jsx), Python (.py),
Rust (.rs), Go (.go), Java (.java), C (.c, .h), C++ (.cpp, .cc, .hpp),
Ruby (.rb), PHP (.php), C# (.cs).
