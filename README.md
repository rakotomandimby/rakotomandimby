# AI-Powered Development Ecosystem I am author of

## Table of Contents

- [code-ai.nvim & code-ai-agent](#ai-powered-development-ecosystem-code-ainvim--code-ai-agent)
  - [Architecture Overview](#architecture-overview)
  - [How They Work Together](#how-they-work-together)
    - [1. Dual Execution Modes with Dynamic Threshold Routing](#1-dual-execution-modes-with-dynamic-threshold-routing)
    - [2. Chunked State-Machine Ingestion Protocol](#2-chunked-state-machine-ingestion-protocol)
    - [3. Wire-Identical Conversation Modeling](#3-wire-identical-conversation-modeling)
    - [4. Simultaneous Multi-Provider Orchestration](#4-simultaneous-multi-provider-orchestration)
    - [5. Normalized Reasoning and Effort Levels](#5-normalized-reasoning-and-effort-levels)
    - [6. Curated Project Context & Instruction Hierarchies](#6-curated-project-context--instruction-hierarchies)
    - [7. Local Prompt Engineering (Ollama)](#7-local-prompt-engineering-ollama)
    - [8. Observability, History & Remote Publishing](#8-observability-history--remote-publishing)
  - [What This Brings to Developers Configuring Them](#what-this-brings-to-developers-configuring-them)
- [lgaicm — LazyGit AI Commit Message](#lgaicm--lazygit-ai-commit-message)
  - [lgaicm Quick Start](#lgaicm-quick-start)
  - [lgaicm Features](#lgaicm-features)
  - [Developer Value: What Configuration Brings You](#developer-value-what-configuration-brings-you)
  - [lgaicm Usage](#lgaicm-usage)
  - [lgaicm Configuration](#lgaicm-configuration)
  - [Conventional Commit Types](#conventional-commit-types)
  - [How lgaicm Works](#how-lgaicm-works)
  - [LazyGit Integration Details](#lazygit-integration-details)
  - [lgaicm Troubleshooting & Debugging](#lgaicm-troubleshooting--debugging)

---

# AI-Powered Development Ecosystem: `code-ai.nvim` & `code-ai-agent`

An advanced, developer-first AI pair-programming ecosystem for Neovim. By combining a lean, asynchronous Neovim Plug-in ([`code-ai.nvim`](https://github.com/code-ai-nvim/code-ai.nvim)) with a dedicated Node.js/TypeScript microservices backend ([`code-ai-agent`](https://github.com/code-ai-nvim/code-ai-agent)), this setup bridges high-context codebase reasoning and editor responsiveness.

---

## Architecture Overview

The ecosystem is decoupled into two cooperating layers:

1. **`code-ai.nvim` (The Neovim Plug-in)**:
   - Written in Lua.
   - Manages visual selections, buffers, floating markdown windows, project discovery, and custom prompt commands.
   - Decides autonomously whether to query providers directly or offload heavy workloads to external agent servers.

2. **`code-ai-agent` (The Backend Monorepo)**:
   - Built with Nx, TypeScript, Express, and SQLite.
   - Hosts provider-specific microservices:
     - `openai-agent` (targeting OpenAI's Responses API for GPT-5/Codex models).
     - `googleai-agent` (targeting Google AI / Vertex Gemini models).
     - `anthropic-agent` (targeting Claude models with up to 1M context beta support).
   - Manages SQLite persistence, state resetting, large payload streaming, and upstream provider communication.

---

## How They Work Together

### 1. Dual Execution Modes with Dynamic Threshold Routing
- **Standalone Mode**: For fast, small queries or single functions, Neovim queries provider APIs directly via `plenary.curl`. No external daemon is required for lightweight use.
- **Agent Mode**: When project context exceeds `agent_size_threshold_bytes` (default: 100 KB) or when forced via prompt configuration (`query_mode = "agent"`), Neovim shifts execution to the external agent.
- **Transparent Selection**: Developers do not need to micromanage modes—the system dynamically determines whether a query is light or heavy based on the total size of scanned project files.

### 2. Chunked State-Machine Ingestion Protocol
Large codebases can choke Neovim if serialized into massive JSON strings inside the Lua runtime. When operating in Agent Mode, `code-ai.nvim` and `code-ai-agent` communicate via a structured, sequential HTTP state machine:
1. **API Key Handshake**: `POST /` with `{ type: "api key", text: "..." }` — The agent cleans existing database state to prepare an isolated session.
2. **System Instructions**: `POST /` with `{ type: "system instructions", text: "..." }` — Stores project and role prompts.
3. **Model Selection**: `POST /` with `{ type: "model", text: "..." }` — Configures the target model and reasoning effort.
4. **File Chunks**: `POST /` with `{ type: "file", filename: "...", content: "..." }` — Each project file matched by `.ai-scanned-files` is streamed individually into SQLite (with a 50MB body limit per chunk).
5. **Prompt Trigger**: `POST /` with `{ type: "prompt", text: "..." }` — The agent gathers the SQLite records, reconstructs an alternating multi-turn conversation, dispatches the request to the upstream AI provider, and returns the response.

### 3. Wire-Identical Conversation Modeling
Both applications share the exact same prompt structure:
- Repository files are converted into an alternating user/assistant dialogue:
  - `User`: *"I need your help on this project."*
  - `Assistant`: *"Please provide the content of the `<filename>` file."*
  - `User`: *"Here is the content of the `<filename>` file: ```...```"*
  - `Assistant`: *"What would you like to do next?"*
  - `User`: *`<User Prompt>`*
- This structure gives the LLMs strong conversational grounding and improves code comprehension compared to naive single-prompt dumps.

### 4. Simultaneous Multi-Provider Orchestration
- Supports **Anthropic Claude**, **Google Gemini**, and **OpenAI GPT-5/Codex** concurrently.
- Running a single command (e.g. `:AIExplain` or `:AIRefactor`) can query all three providers in parallel.
- The results are displayed together in a floating markdown popup (`${anthropic_output}`, `${googleai_output}`, `${openai_output}` or concatenated `${output}`), allowing developers to compare answers from different models side-by-side.

### 5. Normalized Reasoning and Effort Levels
Both applications normalize reasoning/thinking effort across providers using a uniform `:effort` suffix on model names:
- Syntax: `claude-sonnet-5:high`, `gemini-3.5-flash:medium`, `gpt-4o:low`, or `model:none`.
- Uniform mapping across providers:
  - **Anthropic**: Maps to `output_config.effort` (`low`, `medium`, `high`, `xhigh`, `max`) or `thinking = { type = "between_tools" }` for `:none`.
  - **Google AI**: Maps to `thinkingConfig.thinkingLevel` (`minimal`, `low`, `medium`, `high`), where `:none` resolves to the most cost-effective tier (`low` for Gemini 3.7+, `minimal` otherwise).
  - **OpenAI**: Maps to `reasoning.effort` (`none`, `low`, `medium`, `high`, `xhigh`, `max`) through the Responses API.

### 6. Curated Project Context & Instruction Hierarchies
- **`.ai-scanned-files`**: Uses Git-style include/exclude glob syntax (e.g., `+**/*.lua`, `-**/node_modules/**`) to selectively include project files while filtering out build artifacts and lockfiles.
- **`.ai-system-instructions.md`**: Project-level custom instructions merged with bundled, localized system prompts (`common-system-instructions_en.md`, `common-system-instructions_fr.md`).

### 7. Local Prompt Engineering (Ollama)
- Features `:AIRewordPrompt`, connecting to a local Ollama instance (e.g., `llama3.2`) to refine and expand raw prompts before submitting them to cloud models.

### 8. Observability, History & Remote Publishing
- **History Tracking**: Keeps the 15 most recent conversation turns locally in `.ai-history/`.
- **Token Accounting**: Formats and displays token usage (e.g., `12.5k in, 1.2k out`).
- **Telemetry Ingestion**: Optionally sends input lines, input size, and token usage to a centralized ingestion endpoint.
- **Cloud Publishing**: Supports uploading rendered answers to an external markdown blog or team paste service via `upload_url`.

---

## What This Brings to Developers Configuring Them

Investing the time to set up and configure both `code-ai.nvim` and `code-ai-agent` provides tangible workflow advantages:

### 1. Uncompromised Editor Performance
- Running large context queries directly inside Neovim typically causes editor stutter, freezing, and memory bloat when serializing large JSON payloads.
- By delegating heavy context ingestion to `code-ai-agent`, Neovim remains fast, non-blocking, and responsive.

### 2. Multi-Model Consensus Without Browser Context Switching
- Developers no longer need to switch tabs between ChatGPT, Claude, and Gemini web interfaces.
- Visual selections are piped directly into all three engines simultaneously, allowing developers to choose the best solution immediately within Neovim.

### 3. Precise Cost, Latency, and Reasoning Control
- Through model suffix notation (`:low`, `:medium`, `:high`, `:none`), developers can control reasoning depth directly from Neovim keymaps—using lower reasoning tiers for boilerplate edits and high/maximum reasoning for complex algorithms.

### 4. Deterministic Project Context Without Context Overflow
- Rather than blindly reading entire repositories or relying on lossy vector searches, `.ai-scanned-files` gives developers explicit control over which files form the prompt context.
- Skips unreadable and empty files automatically, avoiding unexpected API rejections.

### 5. Flexible Deployment Topologies
- `code-ai-agent` can run locally on `localhost` or be hosted on a remote workstation/server with access to internal infrastructure.
- Decouples API credential management and token usage tracking from the local editor setup.

---

# `lgaicm` — LazyGit AI Commit Message

**Generate AI-powered conventional commit messages directly in LazyGit.** Press a hotkey, select a commit type, choose from AI-generated suggestions, and commit—all without leaving LazyGit.

---

## lgaicm Quick Start

### What You Need
- [LazyGit](https://github.com/jesseduffield/lazygit) installed
- Google AI API key (for Gemini)
- `bash`, `git`, `curl`, `jq`, and `mktemp` (standard in `coreutils`)

### Setup in 3 Steps

**1. Install `lgaicm`:**
```bash
git clone https://github.com/rakotomandimby/lgaicm.git
cd lgaicm
chmod +x lgaicm
ln -s "$(pwd)/lgaicm" /usr/local/bin/lgaicm
```

**2. Set your Google AI API key:**
```bash
export GOOGLEAI_API_KEY="your-google-ai-api-key-here"
# Add to ~/.bashrc or ~/.zshrc for persistence
```

**3. Add to LazyGit config** (`~/.config/lazygit/config.yml`):

Copy the content of [`lgaicm/lazygit-config.yml`](./lgaicm/lazygit-config.yml) and append it to your LazyGit configuration file.

**That's it!** Stage your changes in LazyGit and press `Ctrl+A` to generate and select AI-powered commit messages.

---

## lgaicm Features

- 🤖 **AI-Powered**: Leverages Google Gemini models to analyze git diff stats and patches to generate meaningful commit messages.
- 📝 **Conventional Commits**: Enforces conventional commit formatting with mandatory type prefixing and scope inference.
- 🎯 **Type Selection**: Choose from allowed conventional commit types (`feat`, `fix`, `chore`, `docs`, `style`, `refactor`, `perf`, `test`, `ci`, `build`).
- 🔧 **Highly Configurable**: Customize model, endpoint URL, timeouts, suggestion counts, diff truncation limits, subject length, and debug logs via environment variables.
- ⚡ **Fast Integration**: Seamlessly integrates with LazyGit's `customCommands` and `menuFromCommand`.
- 🎨 **Multiple Suggestions**: Generates a configurable number of distinct commit message options (default 5–7).
- 📄 **Multi-line Safe**: Produces comprehensive commit messages with subject lines and detailed multi-line bodies covering changes, risks, and regression testing tips.

---

## Developer Value: What Configuration Brings You

`lgaicm` is intentionally configurable at every layer. Here is a breakdown of what each configuration dimension gives back to you as a developer.

### Zero Context-Switching

By plugging into LazyGit's `customCommands` system, the entire workflow—staging, type-selection, suggestion review, and commit—happens inside the same TUI you already use. You never have to drop to a shell prompt, think of a commit message yourself, or tab-switch to a browser.

**What the configuration enables:** the `lazygit-config.yml` file wires up a two-step menu interaction (`Ctrl+A` → type menu → AI suggestion menu) using LazyGit's `menu` and `menuFromCommand` prompt types. Once appended to `~/.config/lazygit/config.yml`, this is entirely transparent.

### Consistent Conventional Commits, Enforced

The allowed commit types (`feat`, `fix`, `chore`, `docs`, `style`, `refactor`, `perf`, `test`, `ci`, `build`) are validated both at the shell level and passed as hard constraints into the AI prompt. The subject normalization logic additionally:

- Replaces any AI-generated type prefix with the one _you_ explicitly selected.
- Strips trailing periods from subject lines.
- Trims whitespace and enforces character limits.

**What this means for teams:** commit history stays uniform across contributors, regardless of who generated the message or which AI model was called. Code review, changelogs, and semantic versioning tooling (e.g., `semantic-release`) all benefit from this consistency automatically.

### Rich Multi-line Commit Bodies

The AI prompt instructs the model to produce commit bodies that cover:

- **What changed and why** — not just file names.
- **All scope areas** touched (multi-scope changes are explicitly listed, not silently merged into one line).
- **Behavior changes** and **notable risks**.
- **Regression testing hints** — concrete suggestions of what to verify after the change.
- **Code snippets** where useful, indented 4 spaces to distinguish from prose.
- Text wrapped at ~72 characters, matching `git log` conventions.

### Flexible AI Backend

`lgaicm` is not hard-wired to a single endpoint. The URL resolution system supports three patterns:

| Pattern | When to use |
|---|---|
| Default Vertex AI base URL (no config needed) | Standard Google AI Studio / Vertex projects |
| `{model}` placeholder in `LGAICM_API_URL` | Custom routing, reverse proxies |
| Fully qualified `:generateContent` URL | Fine-grained control (specific model versions, regions) |

Switch model tiers instantly for a different cost/quality tradeoff:
```bash
export LGAICM_MODEL="gemini-2.5-pro"         # higher quality, slower
export LGAICM_MODEL="gemini-2.0-flash-lite"  # faster, cheaper
```

### Tunable Suggestion Quality vs. Speed

| Variable | What it controls | Developer benefit |
|---|---|---|
| `LGAICM_MIN_SUGGESTIONS` | Minimum candidates the AI must produce | Never get fewer suggestions than you want |
| `LGAICM_MAX_SUGGESTIONS` | Maximum candidates requested | Control token cost and latency |
| `LGAICM_CURL_TIMEOUT` | Per-request HTTP timeout | Fail fast on slow networks or raise for large diffs |

If `LGAICM_MIN_SUGGESTIONS` is accidentally set higher than `LGAICM_MAX_SUGGESTIONS`, the script automatically clamps min down to max, so misconfiguration never causes a crash.

### Token Budget Control

Large diffs can exceed API input limits and slow responses noticeably. `lgaicm` gives you two independent byte-cap controls:

| Variable | Covers | Default |
|---|---|---|
| `LGAICM_MAX_STAT_BYTES` | `git diff --cached --stat` (file list summary) | 60 000 bytes |
| `LGAICM_MAX_DIFF_BYTES` | `git diff --cached` (full patch content) | 200 000 bytes |

When a diff exceeds its limit, the script truncates it cleanly and appends a `[Truncated to N bytes]` marker so the AI still knows the context was cut.

### Debug Transparency

Every significant internal step is logged with timestamps to `LGAICM_DEBUG_LOG` (default `/tmp/lgaicm_debug.log`):

- The fully resolved endpoint URL (with the API key redacted).
- System instructions and user prompt sent to the model.
- The raw JSON API response.
- Extracted candidate text and finish reasons.
- Session directory paths and file names.

Real-time monitoring:
```bash
tail -f /tmp/lgaicm_debug.log
```

### Automatic Session Cleanup

Each `suggest` invocation creates an isolated temporary directory (`/tmp/lgaicm.XXXXXX/`) marked with a `.lgaicm_session` sentinel file. The `commit` subcommand reads the selected message file, then removes the entire temp directory only after a successful commit—no stale files accumulate between sessions.

---

## lgaicm Usage

### In LazyGit (Recommended)

1. Stage your changes in LazyGit (`space` on modified files).
2. Press `Ctrl+A` (or your configured keybinding).
3. Select the commit type from the prompt menu.
4. Review the AI-generated commit message suggestions in the submenu.
5. Select your preferred message to commit automatically.

### Command Line

```bash
# Generate suggestions (requires staged changes)
lgaicm suggest --type feat
lgaicm suggest -t fix

# Commit using a selected suggestion file
lgaicm commit --file /tmp/lgaicm.XXXXXX/msg-001.txt
lgaicm commit -f /tmp/lgaicm.XXXXXX/msg-001.txt

# Show usage and available options
lgaicm --help
```

The `suggest` subcommand outputs lines in the delimiter format expected by LazyGit:
```text
<subject> <===> <path-to-message-file>
```

The `commit` subcommand creates the commit using `git commit -F <path>` and cleans up the temporary session directory.

---

## lgaicm Configuration

`lgaicm` is configured via environment variables. You can set them in your shell configuration file (`~/.bashrc`, `~/.zshrc`, etc.):

### API Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `GOOGLEAI_API_KEY` | *(required)* | Your Google AI API key |
| `LGAICM_MODEL` | `gemini-3.5-flash-lite` | Gemini model name |
| `LGAICM_API_URL` | `https://aiplatform.googleapis.com/v1/publishers/google/models` | Base Vertex AI endpoint or generateContent endpoint URL |
| `LGAICM_CURL_TIMEOUT` | `45` | API request timeout in seconds |

#### Endpoint URL Resolution Rules

`lgaicm` resolves `LGAICM_API_URL` using the following rules:

1. **Default / Base URL**: `lgaicm` automatically appends `/<LGAICM_MODEL>:generateContent`.
2. **Template Substitution**: Any `{model}` or `{MODEL}` placeholder is substituted with the value of `LGAICM_MODEL`.
3. **Explicit Endpoint**: If `LGAICM_API_URL` already contains `:generateContent`, it is used as-is.

### Output & Diff Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `LGAICM_MIN_SUGGESTIONS` | `5` | Minimum number of commit message suggestions to request |
| `LGAICM_MAX_SUGGESTIONS` | `7` | Maximum number of commit message suggestions to request |
| `LGAICM_MAX_SUBJECT_LENGTH` | `500` | Maximum character length for normalized commit subject lines |
| `LGAICM_MAX_STAT_BYTES` | `60000` | Maximum byte limit for `git diff --cached --stat` before truncation |
| `LGAICM_MAX_DIFF_BYTES` | `200000` | Maximum byte limit for `git diff --cached` patch before truncation |
| `LGAICM_DEBUG_LOG` | `/tmp/lgaicm_debug.log` | Target file path for detailed debug logging |

### Example Configuration

```bash
# ~/.bashrc or ~/.zshrc

# Required
export GOOGLEAI_API_KEY="your-google-ai-api-key"

# Optional customizations
export LGAICM_MODEL="gemini-3.5-flash-lite"
export LGAICM_MIN_SUGGESTIONS="3"
export LGAICM_MAX_SUGGESTIONS="5"
export LGAICM_CURL_TIMEOUT="60"
export LGAICM_MAX_STAT_BYTES="60000"
export LGAICM_MAX_DIFF_BYTES="200000"
export LGAICM_MAX_SUBJECT_LENGTH="100"
export LGAICM_DEBUG_LOG="/tmp/lgaicm_debug.log"
```

---

## Conventional Commit Types

| Type | Description |
|------|-------------|
| `feat` | A new feature |
| `fix` | A bug fix |
| `chore` | Maintenance, tooling, or non-user-facing changes |
| `docs` | Documentation only changes |
| `style` | Formatting, missing semi-colons, whitespace, etc. (no code behavior change) |
| `refactor` | Code restructuring without fixing a bug or adding a feature |
| `perf` | Performance improvement changes |
| `test` | Adding or updating tests |
| `ci` | CI/CD configuration files and scripts |
| `build` | Build system or external dependencies changes |

---

## How lgaicm Works

### Suggest Phase (`lgaicm suggest --type <type>`)

1. **Prerequisite Verification**: Checks for `git`, `curl`, `jq`, `mktemp`, a valid git repository, staged changes, and `GOOGLEAI_API_KEY`.
2. **Diff Collection & Safety Truncation**: Extracts `git diff --cached --stat` and `git diff --cached`, truncating according to `LGAICM_MAX_STAT_BYTES` and `LGAICM_MAX_DIFF_BYTES`.
3. **Structured Prompt Generation**: Builds a prompt enforcing JSON output (`[{ "subject": "...", "body": "..." }]`), conventional commit rules, subject line wrapping, and multi-paragraph commit bodies covering what changed, why, behavior changes, risks, and regression testing steps.
4. **API Request**: Calls the Gemini `generateContent` endpoint using `curl` with `responseMimeType: "application/json"`.
5. **JSON Extraction & Subject Normalization**: Extracts candidate text, parses the JSON array, ensures the type prefix is present, strips trailing periods, and trims whitespace.
6. **Temporary File Storage**: Writes each message to an isolated temporary session folder and emits `<subject> <===> <path>` lines for LazyGit.

### Commit Phase (`lgaicm commit --file <path>`)

1. **Commit Execution**: Reads the full multi-line commit message from the file using `git commit -F <file>`.
2. **Session Cleanup**: Identifies the parent session directory via the `.lgaicm_session` marker and removes the temporary directory upon completion.

---

## LazyGit Integration Details

The provided [`lgaicm/lazygit-config.yml`](./lgaicm/lazygit-config.yml) wires up a two-step interactive menu using LazyGit's `customCommands`:

```yaml
customCommands:
  - key: "<c-a>"              # Ctrl+A triggers the whole flow
    context: "global"         # Available from any LazyGit panel
    loadingText: "Generating commit messages..."
    prompts:
      - type: "menu"          # Step 1: user picks a conventional commit type
        key: "Type"
        title: "Type of change"
        options: [ feat, fix, chore, docs, style, refactor, perf, test, ci, build ]

      - type: "menuFromCommand"   # Step 2: lgaicm runs and populates the menu
        title: "AI Generated Commit Messages"
        key: "CommitFile"
        command: "lgaicm suggest --type {{.Form.Type}}"
        filter: '^(?P<label>.*?) <===> (?P<file>.*)$'
        valueFormat: '{{.file}}'
        labelFormat: '{{.label}}'

    command: "lgaicm commit --file {{.Form.CommitFile | quote}}"
```

- `menuFromCommand` runs `lgaicm suggest` and parses its `<label> <===> <file>` output using a regex filter.
- `{{.Form.CommitFile | quote}}` safely quotes the selected temp file path, making paths with spaces safe.
- The `global` context makes the command reachable from any LazyGit panel.
- To change the keybinding, replace `"<c-a>"` with your preferred key.

---

## lgaicm Troubleshooting & Debugging

### Debug Logs

```bash
tail -f /tmp/lgaicm_debug.log
```

### Common Errors

- **`Error: GOOGLEAI_API_KEY environment variable is not set.`**  
  Export your API key before launching LazyGit: `export GOOGLEAI_API_KEY="your-api-key"`

- **`Error: No staged changes detected.`**  
  Stage the intended changes with `git add <files>` or press `space` in LazyGit.

- **`Error: Not inside a git repository.`**  
  Ensure you are executing the command within a valid Git working tree.

- **`Error: Required command 'jq' is not available in PATH.`**  
  Install `jq`: `sudo apt-get install jq` / `brew install jq` / `sudo dnf install jq`

- **API timeout errors**  
  Increase the curl timeout: `export LGAICM_CURL_TIMEOUT="90"`
