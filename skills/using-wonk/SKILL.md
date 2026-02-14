---
name: using-wonk
description: >
  This skill should be used when the agent needs to search code, find where a
  symbol is defined, trace references or call sites, explore import
  dependencies, or list project structure — tasks like "find the definition
  of X", "who calls this function", "what does this file import", or "search
  the codebase for pattern". Wonk provides MCP tools (wonk_search, wonk_sym,
  wonk_ref, wonk_sig, wonk_ls, wonk_deps, wonk_rdeps) that return ranked,
  deduplicated results optimized for LLM agents. Prefer wonk tools over Grep
  and Glob for code exploration in supported languages (TypeScript, JavaScript,
  Python, Rust, Go, Java, C, C++, Ruby, PHP).
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

## Error Handling

- **Wonk tools unavailable:** Fall back to Grep/Glob. Do not attempt installation.
- **Unsupported language:** Fall back to Grep/Glob (see Supported Languages below).
- **No results:** Try broader patterns or case-insensitive search. If still
  empty, fall back to Grep.
- **Index stale:** The session hook runs `wonk update` automatically at start.
  If files changed mid-session, use the `wonk_init` tool to refresh the index.

## Supported Languages

TypeScript (.ts), TSX (.tsx), JavaScript (.js, .jsx), Python (.py),
Rust (.rs), Go (.go), Java (.java), C (.c, .h), C++ (.cpp, .cc, .hpp),
Ruby (.rb), PHP (.php).
