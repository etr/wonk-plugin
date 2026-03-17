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

## Hard Rules (read these BEFORE making any call)

1. **NEVER Read a file that wonk already returned.** `wonk show` returns
   complete source code — opening the same file with Read wastes a call and
   doubles context. If wonk showed the code, you have it. Use it.
2. **NEVER pipe wonk output** through `| head`, `| grep`, `| awk`, or any
   filter. `--budget N` already limits output size — you do not need `| head`
   as a safety net. Piping destroys ranked output and often triggers
   unnecessary Read fallbacks. Run wonk bare: `wonk show X --budget 4000`
3. **NEVER use `2>/dev/null` or `2>&1 |`**. Stderr has diagnostics you need.
   Merging stderr into stdout then piping through head loses error messages
   and causes blind retries.
4. **NEVER guess file paths.** If you don't know what file contains a symbol,
   run `wonk sym X` first — it tells you the file and line. Then use that
   exact path in follow-up calls.
5. **If step 1 returned what you need — stop.** Don't make a second call
   "to be thorough." Answer from what you have.

## Step 2 Selection (when you need a second call)

After your first wonk call, pick the right second call — do NOT fall back to
Read/Grep/Glob:

| Step 1 returned | You still need | Step 2 |
|----------------|---------------|--------|
| Definition of a trait/interface | Who implements it | `wonk ref X` |
| Definition of a function | What it calls | `wonk callees X --depth 3` |
| Definition of a function | Who calls it | `wonk callers X` |
| Summary of a directory | Source of a specific symbol | `wonk show SymbolName` |
| Callers/callees list | Source of one caller | `wonk show callerName` |
| File path from `wonk sym` | Source code | `wonk show X --file path` |
| Truncated source (`... N more lines`) | Rest of the source | `wonk show X --page 2` (NOT Read) |

## Mode Selection

Check `.claude/wonk.local.md` in the project root for the configured mode.
Read the YAML frontmatter `mode` field.

- If the file does not exist or has no `mode` field → **CLI mode** (default)
- `mode: cli` → **CLI mode**
- `mode: mcp` → **MCP mode** (wonk MCP tools are available as `wonk_*`)

## When to Use Wonk

**ALWAYS try wonk first for code search tasks.** Only fall back to Grep/Glob/Read
if wonk returns no results or the situation matches one of these exceptions:
- The project uses a language wonk does not support
- Searching non-code files (configs, docs, data files, logs)
- Wonk is not installed

## Tool Selection Guide

**Match your question to the right recipe. This is the most important decision.**

| Question pattern | CLI Command | MCP Tool |
|-----------------|-------------|----------|
| "How does X handle/process Y?" | `wonk callees X --depth 3` | `wonk_callees(depth=3)` |
| "How does X reach Y?" | `wonk callpath X Y` | `wonk_callpath` |
| "What is X? Show me X" | `wonk show X` (batch: `X,Y,Z`; class method: `Class.method`) | `wonk_show` |
| "Everything about X" | `wonk context X` (def + callers + callees in ONE call) | `wonk_context` |
| "Who calls X?" (need caller names) | `wonk callers X` (gives caller NAMES + files) | `wonk_callers` |
| "What references/imports X?" (need call sites) | `wonk ref X` (gives call site lines) | `wonk_ref` |
| "What does this module contain?" | `wonk summary path --depth 1` | `wonk_summary` |
| "List symbols in a file" | `wonk show --file path --shallow` | `wonk_show(file=)` |
| "Find where X is defined" | `wonk sym X` | `wonk_sym` |
| "Check function signature" | `wonk sig X` | `wonk_sig` |
| "Search for keyword/regex" | `wonk search pattern` | `wonk_search` |
| "Semantic / NL search" | `wonk ask "query"` | `wonk_ask` |

**IMPORTANT**: `wonk search` is ONLY for literal keyword/regex search when you don't
have a specific function or class name. If the question mentions a function name
(e.g. "how does httpx handle redirects" → the function is `_send_handling_redirects`),
use `wonk callees` or `wonk show`, NOT `wonk search "redirect"`. Broad searches
return hundreds of matches and waste tokens.

## CLI Mode (Default)

Run wonk commands via the **Bash** tool. Always use `--format toon -q` for
compact, LLM-friendly output without stderr hints.

### CLI Reference

```
wonk search <pattern> [--regex] [-i] [--raw] [--smart] [--semantic] [--file <path>] [--budget N] [-- path1 path2]
wonk sym <name> [--kind <kind>] [--file <path>] [--exact] [--limit N] [--budget N]
wonk ref <name> [--output full|files] [--file <path>] [--budget N] [-- path1 path2]
wonk show <name[,name2,...]> [--file <path>] [--kind <kind>] [--exact] [--shallow] [--budget N]
wonk sig <name> [--budget N]
wonk callers <name> [--reference-file F] [--callers-file F] [--depth N] [--min-confidence N] [--budget N]
wonk callees <name> [--reference-file F] [--callees-file F] [--depth N] [--min-confidence N] [--budget N]
wonk callpath <from> <to> [--reference-file F] [--destination-file F] [--min-confidence N] [--budget N]
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

# Show source code of multiple symbols (batch)
wonk show "parseConfig,validateInput,main" --format toon -q

# Show source using qualified path (Python/JS dot or Rust ::)
wonk show Client.get --format toon -q
wonk show ScheduledIo::wake --format toon -q

# Full context for a symbol (definition + callers + callees + imports)
wonk context handleRequest --format toon -q --budget 4000

# Module architecture overview
wonk summary src/api --depth 1 --format toon -q

# Search for a pattern
wonk search "TODO|FIXME" --regex --format toon -q

# List just file paths that reference a symbol
wonk ref BaseTransport --output files --format toon -q

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

## Task Recipes (pick the right one, execute it, stop)

Each recipe is 1-2 calls. Do NOT chain 3+ calls — each adds all prior context.

### Complex multi-file traces → delegate to Agent

For questions requiring 3+ files (e.g. "How does error handling work?", "Trace
the request flow from server to handler"), spawn an **Explore subagent** that
uses wonk internally. The subagent's context stays isolated — only its summary
comes back to your main conversation. This is dramatically cheaper than making
5-10 wonk calls on the main thread.

```
Agent(subagent_type="Explore", prompt="Use `wonk callees X --depth 3` and
`wonk show Y` via Bash to trace how <topic> works. Summarize the flow.")
```

### "Find where X is defined / show me X"
```
wonk show X --budget 4000                    # source + location
wonk show X,Y,Z --budget 4000               # batch multiple symbols
wonk show Class.method --budget 4000         # qualified: Python/JS/TS class method
wonk show Mod::Type --budget 4000            # qualified: Rust path
```

### "What methods does class X have?"
For Python/JS/TS classes, **batch the method names with qualified paths**:
```
wonk show "X.get,X.post,X.send,X.close" --budget 4000
```
Or show all top-level symbols in the file containing X:
```
wonk show --file path/to/x.py --shallow --budget 4000
```
For JS prototype-based classes (not `class` syntax):
```
wonk search "X.prototype" --budget 4000
```
Do NOT use `wonk context X` for listing methods — context shows callers/callees,
not child method signatures.

### "Everything about X" (definition + who calls it + what it calls)
```
wonk context X --budget 4000                 # ONE call: def + callers + callees + imports
wonk context Class.method --budget 4000      # qualified path works here too
```

### "Who calls X?" / "What calls X?"
```
wonk callers X --budget 4000
wonk callers Class.method --budget 4000      # qualified path scopes to the right method
```

### "What does X call?" / "Trace flow from X"
```
wonk callees X --depth 3 --budget 4000       # forward call graph
wonk callees Class.get --depth 3 --budget 4000  # qualified path
```

### "How does A reach B?" (call chain)
```
wonk callpath A B --budget 4000              # shortest path A → ... → B
```

### "What references X?" / "What imports X?"
```
wonk ref X --budget 4000                     # all references with context
wonk ref X --output files --budget 4000      # just file paths (compact)
```

### "Summarize this module / directory"
```
wonk summary path/ --depth 1 --budget 8000   # depth 1 = immediate children
```
Use `--budget 8000` for summary (not 4000) — directory summaries include
descriptions and can be large. For very large directories, omit `--depth` to
get just the top-level metrics.

### "Search for keyword / pattern"
```
wonk search "pattern" --budget 4000          # ranked, definitions first
wonk search "handleError|onError" --regex    # regex patterns
```

### "How does X work?" (architecture / flow tracing)
Two-call pattern:
1. `wonk context X --budget 4000` — understand X's role (def + callers + callees)
2. `wonk callees X --depth 3 --budget 4000` — trace forward flow from X

Or for module-level architecture:
1. `wonk summary path/ --depth 1 --budget 8000` — module overview with file list

## Handling Empty Results

When a command returns 0 results, it includes diagnostic hints:
- Similar symbol name suggestions
- Advice to remove file/kind filters for broader results
- Suggestion to try `wonk search` for text-based matching

Follow the hints rather than retrying with slightly different parameters.

## Handling Truncation & Pagination

When results are truncated, the response includes "Showing N of M".
- Use `--page 2` to see the next page. Don't read all pages — only the minimum needed.
- **wonk show auto-shallow:** Large container types auto-fallback to shallow mode
  when their body exceeds budget. `auto_shallow` flag indicates this happened.
- **wonk show returns complete source** — you do not need to Read the same file afterward.
- **wonk summary** progressive: root summary always shown, children truncated at budget boundary.
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
