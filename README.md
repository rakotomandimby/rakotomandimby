# I am the author of the AI-Powered Development Ecosystem: `code-ai.nvim` & `code-ai-agent`

An advanced, developer-first AI pair-programming ecosystem for Neovim. By combining a lean, asynchronous Neovim frontend ([`code-ai.nvim`](./code-ai.nvim)) with a dedicated Node.js/TypeScript microservices backend ([`code-ai-agent`](./code-ai-agent)), this setup bridges high-context codebase reasoning and editor responsiveness.

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
