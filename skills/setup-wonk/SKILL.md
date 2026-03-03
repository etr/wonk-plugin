---
name: setup-wonk
description: >
  This skill should be used when the user asks to install wonk, set up wonk,
  update wonk, configure wonk, enable semantic search, install Ollama for wonk,
  or troubleshoot a missing wonk binary — requests like "install wonk",
  "setup wonk", "configure wonk", "update wonk", "set up code search",
  "enable semantic search", or "wonk is not found".
---

# Wonk Setup — Install, Update, Dependencies & Configuration

Walk the user through a complete wonk setup in three steps: installing or
updating the wonk binary, setting up optional dependencies (Ollama for semantic
features), and configuring wonk options.

## Step 1: Check, Install, or Update Wonk

1. Run `command -v wonk` to check if wonk is on PATH.

### If wonk is found

1. Run `wonk --version` and note the installed version.
2. Fetch the latest release version:
   ```
   curl -s https://api.github.com/repos/etr/wonk/releases/latest
   ```
   Extract the `tag_name` field (strip the leading `v` to compare).
3. If the installed version matches the latest, tell the user wonk is
   up-to-date and skip to Step 2.
4. If a newer version is available, use **AskUserQuestion** to ask:
   - Question: "Wonk **<current>** is installed. Version **<latest>** is
     available. Would you like to update?"
   - Options: "Yes, update to <latest>" / "No, keep current version"
5. If the user chooses to update, run the install script:
   ```
   curl -fsSL https://raw.githubusercontent.com/etr/wonk/main/install.sh | sh
   ```
   Then verify with `wonk --version` and report the new version.

### If wonk is NOT found

1. Use **AskUserQuestion** to ask:
   - Question: "Where should wonk be installed?"
   - Options: "/usr/local/bin (default)" / "~/.local/bin" / "Custom location"
2. Run the install script with the chosen location:
   ```sh
   # Default:
   curl -fsSL https://raw.githubusercontent.com/etr/wonk/main/install.sh | sh

   # Custom:
   WONK_INSTALL=<path> curl -fsSL https://raw.githubusercontent.com/etr/wonk/main/install.sh | sh
   ```
3. Verify with `wonk --version`. If the command is not found, check whether
   the install directory is on the user's PATH and advise them to add it.

### Error handling

- **Permission denied:** Suggest re-running with `sudo`.
- **Unsupported platform:** Report the detected OS/architecture. Suggest
  `cargo install wonk` as a fallback if Rust is available.
- **curl not found:** Suggest installing curl first, or using
  `cargo install wonk`.

## Step 2: Optional Dependencies

After wonk is confirmed installed, use **AskUserQuestion** to ask:

- Question: "Wonk supports semantic search (`wonk ask`), clustering
  (`wonk cluster`), and impact analysis (`wonk impact`) when Ollama is
  available. Would you like to set up Ollama?"
- Options: "Yes, set up Ollama" / "No, skip"

### If the user chooses to set up Ollama

1. Run `command -v ollama` to check if Ollama is installed.
2. If not found, install it:
   ```
   curl -fsSL https://ollama.ai/install.sh | sh
   ```
   If this fails, tell the user to visit https://ollama.com/download for
   platform-specific instructions.
3. Check that Ollama is running:
   ```
   ollama list
   ```
   If this fails with a connection error, ask the user to start Ollama
   (`ollama serve` in another terminal, or launch the Ollama app) and wait
   for confirmation before continuing.
4. Pull the required models:
   ```
   ollama pull nomic-embed-text
   ollama pull llama3.2:3b
   ```
   - **nomic-embed-text** — embeddings for semantic search, clustering, and
     index initialization.
   - **llama3.2:3b** — AI-generated summaries.
5. Verify both models appear in `ollama list`. Report success or any errors.

### If the user skips

Tell them: "Wonk will work for structural search without Ollama. You can set
up Ollama later by running this skill again."

## Step 3: Configuration

Use **AskUserQuestion** to ask:

- Question: "Would you like to configure wonk? I can set up a config file with
  options like file size limits, ignore patterns, output format, and LLM
  settings."
- Options: "Yes, configure wonk" / "No, use defaults"

### If the user chooses to configure

1. Ask where to create the config file:
   - Options: "Global (`~/.wonk/config.toml`) — all repositories" /
     "Project (`.wonk/config.toml`) — this repo only"

2. Walk through config sections. For each, use **AskUserQuestion** to ask
   whether the user wants to customize it. Only include sections the user
   changes.

   **Indexing:**
   - `max_file_size_kb` — max file size to index (default: 1024 KiB)
   - `additional_extensions` — extra extensions beyond built-in set
     (e.g. `.proto`, `.graphql`)

   **Ignore patterns:**
   - `patterns` — glob patterns to exclude (e.g. `vendor/**`,
     `*.generated.go`)

   **Output:**
   - `default_format` — `"grep"` (default), `"json"`, or `"toon"`

   **LLM (only if Ollama was set up in Step 2):**
   - `model` — Ollama model for summaries (default: `llama3.2:3b`)
   - `generate_url` — Ollama endpoint (default:
     `http://localhost:11434/api/generate`; change for remote Ollama)

   **Daemon:**
   - `debounce_ms` — file-change debounce interval (default: 500)

3. Build the TOML config with only non-default values. The full template for
   reference:
   ```toml
   [daemon]
   debounce_ms = 500

   [index]
   max_file_size_kb = 1024
   additional_extensions = []

   [output]
   default_format = "grep"
   color = "auto"

   [ignore]
   patterns = []

   [llm]
   model = "llama3.2:3b"
   generate_url = "http://localhost:11434/api/generate"

   [search]
   rrf_k = 60.0
   ```

4. Create the parent directory if needed (`mkdir -p`) and write the file.
   Show the user what was written.

### If the user skips

Tell them: "Wonk will use built-in defaults. You can add a config later at
`~/.wonk/config.toml` (global) or `.wonk/config.toml` (per-project)."

## Final Summary

Print a summary of everything that was set up:
- Wonk version installed (or updated from → to)
- Ollama status and available models (or skipped)
- Config file path and what was configured (or using defaults)
- Suggest running `wonk init` in a repository to build the initial index
