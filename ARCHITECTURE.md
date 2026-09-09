# Bountiful Architecture

Bountiful is a local-first AI agent platform. The user clones this
repo, names their agent, and runs it. The agent has persistent memory,
tool access, and inference through local models or the Anthropic API.

This document covers the full ecosystem — every repo, every package,
how they connect, and why they're built the way they are.


## Repos

| Repo | What it is |
|------|------------|
| [bountiful](https://github.com/CablepunkPress/bountiful) | Agent template. Configuration, shim scripts, this document. |
| [basic-bot](https://github.com/CablepunkPress/basic-bot) | Engine. Chat loop, memory, tools, provider abstraction. |
| [basic-ui](https://github.com/CablepunkPress/basic-ui) | Reference Flask interface and local launch orchestration. |
| [extend-a-bot](https://github.com/CablepunkPress/extend-a-bot) | Plugin tool groups. |


## Agent Assembly

### Design Principles

**The directory is the agent.** An agent's identity, personality, and
tools all live in its repository directory. Clone the template, name
the directory, build and run the agent.

**The database is the agent's memory.** The SQLite file at
`~/.{id}/{id}.db` holds every conversation, summary, and vector
embedding the agent has ever produced. Back it up.

**dashboard.json is the source of truth.** The `id` field drives the
database filename, dot-directory path, keyring service, log prefix,
and UI title.

**Shared infrastructure, isolated data.** All agents share llama.cpp
and the embedding model at `~/.bountiful/`. Each agent's data is
isolated in its own dot-directory.

**Shim scripts stay stable.** Three of the four scripts are thin shims
that delegate to the engine or UI packages. Logic lives behind pip so
engine updates deliver new behavior without touching the agent repo.

### Directory Structure

The cloned bountiful repo becomes the agent's working directory:

| File | What it does |
|------|--------------|
| `dashboard.json` | Agent identity. `id` names everything; `name` is the display name. |
| `persona.md` | The agent's personality. `{{ name }}` is replaced at load time. |
| `config.json` | Agent settings. Flask port and future overrides. Missing fields use engine defaults. |
| `pyproject.toml` | Pins engine and UI versions. `packages = []` — dependencies only. |
| `build.py` | First run: names the agent, removes `.github/`, updates pyproject.toml. Every run: venv, pip install, delegates shared infrastructure to the engine (`python -m basic_bot.infra`). Idempotent. Standalone — runs before the engine exists. |
| `run.py` | Shim. Bounces into the venv, then calls `basic_ui.launch.launch(ROOT)`. |
| `add_tools.py` | Shim. Bounces into the venv, then calls `basic_bot.setup.tools.run(ROOT, args)`. Installs tool groups from extend-a-bot, including pip dependencies. `--update` refreshes code while preserving `_config.py` files. |
| `add_secrets.py` | Shim. Bounces into the venv, then calls `basic_bot.setup.secrets.run(ROOT)`. Prompts for API keys declared by the agent and its tool manifests. |
| `tools/` | Plugin tool groups. Each subdirectory is a group; `_` files are shared modules, everything else with a `TOOL` dict and `handler` is a tool. |


Two dot-directories on the filesystem:

`~/.bountiful/`   # Shared infrastructure (all agents)
`~/.{id}/`        # Agent-specific data


## Script Architecture

Three of the four scripts share the same pattern: detect whether
they're running inside the venv, bounce into it if not, then make a
single import call into the engine or UI package. The venv bounce is
stdlib-only code that never changes.

`build.py` is the exception — it runs before the venv exists. Its
pre-engine responsibilities (naming ceremony, venv creation, pip
install) must stay standalone. After pip install succeeds, it delegates
shared infrastructure setup to the engine via
`python -m basic_bot.infrastructure`, which handles prerequisite checks,
llama.cpp compilation, and model downloads.

## Install Order

python build.py          # name, venv, pip install, infrastructure
python add_tools.py ...  # optional: install tool groups (requires venv)
python add_secrets.py    # store API keys in keyring
python run.py            # launch the agent


`build.py` must run first — everything else requires the venv it
creates.

## Naming Ceremony

On first run, `build.py` detects the sentinel `"my-agent"` in
`dashboard.json`:

1. Derives agent ID from the directory name (lowercased)
2. Derives display name (title case, hyphens/underscores to spaces)
3. Prompts for confirmation; user can override both
4. Checks for dot-directory collision (`~/.{name}/` already exists)
5. Writes `dashboard.json` (id, name) and `pyproject.toml` (name)
6. Removes `.github/` directory (upstream funding metadata)

Runs once — the sentinel check prevents re-triggering.

## Assembly Flow

`run.py` calls `basic_ui.launch.launch(agent_path)`, which:

1. Loads the agent's API key from keyring into the environment
2. Reads `config.json` for agent settings (Flask port)
3. Starts llama-server for embeddings if the embedding provider is local
4. Calls `basic_ui.server.create_local_app(agent_path)`, which calls
   `basic_bot.factory.create_runtime(agent_path)`
5. Runs the Flask app
6. Tears down servers on exit (Ctrl+C)

The factory reads dashboard.json and persona.md, builds the SQLite
store, discovers tools, instantiates the inference provider, and returns
a `BotRuntime`. basic-ui wraps the runtime in Flask routes.

Every agent inherits the engine's three-tier memory (sliding window,
rolling summary, RAG archive), two belt tools (`search_archive`,
`recall_message`), and the inference provider abstraction. See
basic-bot's ARCHITECTURE.md for engine internals.

## Tool Groups

Each group in `tools/` carries a `tool.json` manifest declaring its
pip dependencies (installed by `add_tools.py`), keyring secrets
(prompted by `add_secrets.py`), and user config files (protected during
`--update`). Groups are self-contained — copy one to another agent and
only `_config.py` values and secrets need setting up.


## Engine Core

The engine lives in `basic-bot/basic_bot/`.

### The Factory

`factory.py` — `create_runtime(agent_path)` takes one argument and
derives everything:

1. Reads `dashboard.json` — the `id` field becomes the agent identity
2. Configures logging with the agent ID as prefix
3. Reads `persona.md`, substituting `{{ name }}`
4. Loads `context/*.md` files alphabetically (underscore-prefixed
   files excluded), appends engine's `capabilities.md`
5. Builds the storage backend — SQLite at `~/.{id}/{id}.db`
6. Discovers tools — belt tools from the engine, plugins from
   `agent_path/tools/`
7. Builds both providers:
   - `_build_chat_provider(config)` — LocalProvider or ClaudeProvider
     based on `inference_provider` in config.toml
   - `_build_summary_provider(config)` — always LocalProvider,
     defaults to the chat model if `summary_model` is absent
8. Returns a `BotRuntime` dataclass

The factory builds runtimes, not web applications. Flask wraps the
runtime in routes. The engine stays framework-agnostic.

**Identity flows from `dashboard.json`.** The `id` names the database,
the dot-directory, the keyring service, and the log prefix. One
declaration, everything derived.

### Chat Orchestration

`chat.py` is provider-agnostic. It loads memory context, builds the
system prompt (including dynamic tool count and tool names in the
MODEL section), calls `runtime.chat_provider.chat()`, and handles
the tool execution loop. Tool calls and results stay in the engine's
internal format; the provider translates at the boundary.

### System Prompt Structure

Markdown headings that models navigate by section name:

```
# PERSONA        (from persona.md)
# CAPABILITIES   (engine-owned, from instructions/capabilities.md)
# MODEL          (injected — model, family, effort, thinking,
                  tool count, sorted tool names)
# MEMORY         (injected — rolling summary, window position)
```

The MODEL section includes explicit tool count and sorted tool names
because small models cannot reliably count their own tool definitions
or identify themselves from tool schemas alone.


## Memory System

Three tiers so no gap exists between what the agent remembers and
what it can retrieve.

### Tiers

**Short-term (sliding window)** — recent messages verbatim. Holds
between `WINDOW_FLOOR` (20) and `WINDOW_CEILING` (40) messages.
The window oscillates between floor and ceiling in a sawtooth
pattern. Expected behavior, not a bug.

**Intermediate-term (rolling summary)** — compressed narrative of
everything past the window. Preserves durable facts cheaply. Updated
at each fold by the dedicated summary model.

**Long-term (RAG archive)** — turn pairs embedded as vectors, stored
permanently, searchable via `search_archive`.

### Fold Lifecycle

When the window hits `WINDOW_CEILING`, a fold triggers. In sync mode
(default), the fold completes before the chat response returns:

1. **RAG embedding** — the oldest messages in the window are paired
   and embedded via the embedding server, stored as vectors
2. **Summary generation** — the existing summary and the folded
   messages are sent to the summary provider, which produces an
   updated rolling summary
3. **Boundary advance** — the window boundary moves forward

**Ordering constraint:** RAG embeds first. If RAG fails, summary
does not fire and the boundary does not advance. No message passes
the boundary without a vector.

Files: `basic-bot/basic_bot/fold.py` (lifecycle orchestration),
`basic-bot/basic_bot/summary.py` (prompt construction and provider
call), `basic-bot/basic_bot/rag.py` (turn pairing and vector
storage).

### Summary Model

Summary runs through the `summary_provider` on the runtime — a
separate provider slot from `chat_provider`. Both can point at the
same server or different servers. Summary always runs locally with
thinking disabled.

Summary sampling parameters are baked into the summary code path,
not shared with the chat parameter lookup. Summary does not use
presence penalty (no prior assistant messages to parrot), and uses
non-thinking temperature values for coherent prose.

Every fold logs: `thinking=%s, %d chars, model=%s` — confirms
thinking state, tracks summary length, verifies the correct model.

### Summary Hardening

The summary is generated from raw transcripts, which can contain
text that looks like instructions. Two defenses: XML fencing around
the transcript and a summarizer system prompt that forbids following
transcript instructions. The summary prompt also includes a
supersession instruction: when new information supersedes old, replace
the old — do not preserve both versions of a changed fact.

### Sequence Numbers

Every message gets a permanent sequence number, shown to the model
as `<!-- seq:N -->` HTML comments. HTML comments were chosen because
Haiku echoed bracket formats back into replies; comments read as
non-content metadata. The engine strips echoed annotations at reply
extraction (inside the provider) and at context build.

### Retrieval Tools

- **`search_archive`** (`basic-bot/basic_bot/tool_belt/search_archive.py`)
  — semantic vector search over the RAG archive
- **`recall_message`** (`basic-bot/basic_bot/tool_belt/recall_message.py`)
  — deterministic lookup by seq number (±2 context), seq range, or
  date range. Capped at 50 results.

Complementary modes: semantic search fails when you know the exact
turn; deterministic lookup fails when you only remember the gist.


## Provider Abstraction

The engine never imports provider-specific SDKs. All inference goes
through `InferenceProvider`, a protocol defined in
`basic-bot/basic_bot/providers/protocol.py`.

### Provider Contract

```python
class InferenceProvider(Protocol):
    def get_models(self) -> dict[str, ModelInfo]: ...
    def get_default_model(self) -> str: ...
    def get_fallback_model(self) -> str: ...
    def chat(self, *, messages, system, tools,
             model_id, effort, thinking) -> ChatResponse: ...
```

Providers return `ChatResponse` (text, model_used, thinking,
tool_calls) and `ToolCall` (name, input, id) — engine-owned types.

### Model Taxonomy

Every model carries a `ModelInfo` dataclass with:

- **Host** — `"api"` or `"local"`
- **Provider** — `"Anthropic"`, `"Alibaba"`, etc.
- **Family** — `"Claude"`, `"Qwen"`, etc.
- Plus: `id`, `display_name`, `rank`, `effort_levels`, `thinking_type`

The UI can slice models by host, provider, family, or flat ranked
list without restructuring code.

### ClaudeProvider

`basic-bot/basic_bot/providers/claude.py` — wraps the Anthropic SDK.
Owns the Claude model catalog, token limits, API kwargs, and response
parsing. File named `claude.py` not `anthropic.py` because the latter
would shadow the pip package.

### LocalProvider

`basic-bot/basic_bot/providers/local.py` — talks to llama-server's
OpenAI-compatible `/v1/chat/completions` endpoint via stdlib `urllib`.
One server, one model — the model is chosen at construction and is
the only entry `get_models()` returns.

Handles: alias echo from `--alias`, thinking toggle via
`chat_template_kwargs`, tool calling in OpenAI format, sequence
annotation stripping, and `finish_reason == "length"` truncation
warnings.

### Provider Split

`BotRuntime` has two provider slots:

```python
chat_provider: InferenceProvider    # User-switchable
summary_provider: InferenceProvider # Infrastructure, set by config
```

The factory builds both. `_build_chat_provider` reads
`inference_provider` from config.toml to decide LocalProvider vs
ClaudeProvider. `_build_summary_provider` always builds a
LocalProvider, defaulting to the chat model if `summary_model` is
absent from config (convention over configuration).

Two LocalProvider instances pointing at the same server is valid —
they're stateless HTTP clients.

### Sampling Parameters

Per-family, per-thinking-mode defaults in `SAMPLING_DEFAULTS`:

```python
SAMPLING_DEFAULTS = {
    "qwen": {
        "thinking": {
            "temperature": 0.6, "top_p": 0.95, "top_k": 20,
            "min_p": 0.0, "presence_penalty": 1.5,
        },
        "non_thinking": {
            "temperature": 0.7, "top_p": 0.8, "top_k": 20,
            "min_p": 0.0, "presence_penalty": 1.5,
        },
    },
}
```

Values from Qwen3 GGUF recommendations for quantized models.
`presence_penalty=1.5` prevents repetition ruts in multi-turn chat.
Applied automatically based on model family and thinking state.
Users never see or configure these.


## Infrastructure

Shared Bountiful infrastructure at `~/.bountiful/`, managed by
`basic-bot/basic_bot/infrastructure/`.

### Package Contents

infrastructure/
    __init__.py
    __main__.py       # Entry point for `python -m basic_bot.infrastructure`
    llamacpp.py       # Clone and compile llama.cpp from source
    models.py         # Model catalog — filenames, URLs, launch args
    server.py         # llama-server lifecycle — start, stop, health check



### llama.cpp Build

`llamacpp.py` clones and compiles llama.cpp from source. Includes
prerequisite checks for git, cmake, and a C++ compiler. Platform
detection for build flags:

- Linux with CUDA: `-DGGML_CUDA=ON` (requires CUDA toolkit installed)
- macOS: Metal auto-detected by cmake, no flags needed
- Fallback: CPU-only build

CUDA build requires `-j 4` (not `-j $(nproc)`) — nvcc segfaults
under memory pressure with too many parallel jobs.

### Model Catalog

`models.py` contains:

- `EMBEDDING_MODEL_FILE` and `download_embedding_model()` for the
  embedding model
- `INFERENCE_MODELS` catalog with per-model filename, download URL,
  and launch args
- `get_inference_model_path(model_id)` and
  `get_inference_launch_args(model_id)` for server startup

Current embedding model: Qwen3-Embedding-0.6B Q8_0 GGUF, self-hosted
at CablepunkPress/Qwen3-Embedding-0.6B-GGUF on Hugging Face with
the `tokenizer.ggml.add_eos_token` fix.

### Server Lifecycle

`server.py` manages llama-server processes:

- `ensure_embedding(embedding_url)` — CPU-only, hides GPU via
  `CUDA_VISIBLE_DEVICES=""`, `--n-gpu-layers 0`
- `ensure_inference(model_id, inference_url)` — GPU, launch args
  from model catalog, `--alias` set to catalog ID
- Shared helpers: `_check_port`, `_start_and_wait`
- Separate log files: `llama-embedding.log`, `llama-inference.log`

<!-- NOTE: Server lifecycle is being redesigned for on-demand
     sequential mode. server.py will gain start/stop lifecycle
     management, and launch orchestration will support both
     sequential (one server at a time) and concurrent (all servers
     at boot) modes, controlled by config.toml. -->

### Deployment Topology

Two llama-server processes, one CUDA-enabled binary:

| Port  | Role      | Hardware | Model |
|-------|-----------|----------|-------|
| 11444 | Embedding | CPU-only | Qwen3-Embedding-0.6B Q8_0 |
| 11445 | Inference | GPU      | Configured in config.toml |

The embedding server uses `CUDA_VISIBLE_DEVICES=""` to hide the GPU
entirely. Without this, the CUDA-enabled binary tries to allocate
compute buffers on GPU even for embeddings, failing with OOM when
the inference model is loaded.

<!-- NOTE: A third port (11446) is planned for the dedicated summary
     server when hardware supports concurrent mode. In sequential
     mode, only one server runs at a time. -->


## Tool System

`basic-bot/basic_bot/tools.py` assembles the registry from two
sources:

### Belt Tools

`basic-bot/basic_bot/tool_belt/` — ship with the engine, always
loaded. These are the agent's memory access: `search_archive` and
`recall_message`.

### Plugin Tools (Box Tools)

`agent_path/tools/` — discovered from the filesystem. Each
subdirectory is a tool group. Files starting with `_` load as
shared modules; every other `.py` file exposing a `TOOL` dict and
a `handler` callable registers as a tool.

Plugin tools require no Python package, no `__init__.py`. The
directory is the installation. Tool groups can be copied between
agents or installed from extend-a-bot.

Shared modules are removed from `sys.modules` after each group
loads, so same-named `_config.py` files in different groups never
collide. `TOOL_BOX_ENABLED` env var disables all plugin tools
without removing them.

### Tool Handler Convention

`handler(context, **tool_input)` — the context dict carries the
store and user ID. No base classes, no decorators.

Tool descriptions bake in org-specific values at import time via
f-strings — the model sees the resolved string, not a template.

### extend-a-bot

`extend-a-bot/` is a repository of plugin tool groups. It is not
a pip package. Agents pull tool groups via `add_tools.py`, which
downloads a tarball from GitHub and extracts the requested directory.

Each group carries a `tool.json` manifest declaring pip dependencies,
keyring secrets, and user config files. `--update` overwrites code
while preserving files listed in the manifest's `config` section.

```
extend-a-bot/
└── github/
├── tool.json # Manifest — dependencies, secrets, config
├── _config.py # User-editable values
├── _auth.py # Shared auth module
├── create_branch.py # Tool file (TOOL dict + handler)
├── ... # 11 more GitHub tools
└── read_file.py
```


### Tool Trust

Draft tools are read-only by design. Write tools require UI
confirmation gates. `_config.py` (user-editable) and `_auth.py`
(maintainer-owned) are separated so updates don't overwrite
user configuration.


## UI

The reference chat interface lives in `basic-ui/`.

### Files

| File | What it does |
|------|--------------|
| `launch.py` | `launch(agent_path)` — orchestrates local launch: secrets, servers, Flask app, teardown |
| `server.py` | `create_local_app(agent_path)` — calls the factory, builds Flask routes |
| `templates/index.html` | Agent name injected via Jinja2. Markdown rendered client-side. |
| `static/css/styles.css` | Dark-mode styles, responsive layout |
| `static/js/chat.js` | Message handling, model selection, history loading |

<!-- NOTE: server.py will be renamed to app.py to avoid collision
     with infrastructure/server.py. chat.js will be replaced by
     HTMX in the upcoming UI rewrite. -->

### Design Decisions

- **No dependency on basic-bot at the package level.** Imports happen
  at runtime. The agent repo depends on both and wires them together.
- **Deferred imports in launch.py.** All imports inside `launch()`
  because the Anthropic client reads `ANTHROPIC_API_KEY` at import
  time.
- **Models come from the provider.** The `/models` endpoint reads
  `runtime.chat_provider.get_models()` with rank-based ordering.
- **Templates and static files ship inside the pip package.**
  Paths resolve via `Path(__file__).parent`.


## Configuration

### config.toml (Agent Level)

Lives in the agent directory. Self-documenting with comments.
TOML is stdlib (`tomllib` in Python 3.11+).

```toml
# Agent Configuration
flask_port = 11555
inference_provider = "local"    # "local" or "claude"
default_model = "qwen3-8b-q4_k_m"
max_tokens = 4096
```

**Convention over configuration.** Identity decisions must be
explicit (`inference_provider`). Infrastructure has or-guard
defaults (`summary_model` defaults to the chat model if absent).
The user only configures what they want to change.

### config.py (Engine Level)

`basic-bot/basic_bot/config.py` — constants with env-var overrides:

| Constant | Default | Purpose |
|----------|---------|---------|
| `WINDOW_FLOOR` | 20 | Minimum window size after fold |
| `WINDOW_CEILING` | 40 | Message count triggering fold |
| `FOLD_MODE` | `"sync"` | `"sync"` or `"async"` |
| `EMBEDDING_PROVIDER` | `"local"` | `"local"` or `"vertex"` |
| `STORAGE_BACKEND` | `"sqlite"` | `"sqlite"` or `"firestore"` |
| `EMBEDDING_URL` | `http://localhost:11444` | Embedding server |
| `TOOL_BOX_ENABLED` | `true` | Kill switch for plugin tools |

### Context Directory

`context/` in the agent directory. Arbitrary `.md` files dropped in,
loaded alphabetically into the system prompt between persona and
capabilities. Files starting with `_` are excluded (convention for
READMEs or notes that shouldn't enter the prompt).


## Storage and Embedding

### Storage

`MessageStore` protocol in `basic-bot/basic_bot/store.py` covers
conversation state, message CRUD, summaries, and vectors.

- **`SQLiteMessageStore`** (`store_sqlite.py`) — local default. WAL
  mode, four tables, vectors as `struct.pack` blobs, cosine
  similarity in Python. Stdlib only.
- **`FirestoreMessageStore`** (`store_firestore.py`) — cloud.

Selected by `STORAGE_BACKEND` env var. Imports are deferred so
local installs never pull in google.cloud.

**The database file is the agent's memory.** SQLite makes it
portable — migration is copying the file.

### Embedding

`EmbeddingProvider` protocol in `basic-bot/basic_bot/embeddings.py`.

- **`LocalEmbedder`** — HTTP to llama-server on localhost (default)
- **`VertexEmbedder`** — Vertex AI (cloud)

Qwen3 task prefixes: queries get an instruct prefix, documents pass
through bare. Applied inside the embedder.

**Embedding spaces are model-specific.** Swapping models requires
re-embedding everything — `scripts/reembed.py` exists for this.

Current model: Qwen3-Embedding-0.6B Q8_0 (639MB, 1024 dims, 32K
context), CPU-only on port 11444.


## Scripts

Maintenance utilities in `basic-bot/scripts/`, run manually:

- **`reembed.py`** — re-embed all vectors (required after model swap)
- **`rebuild_summary.py`** — regenerate the rolling summary
- **`backfill_rag.py`** / **`backfill_seq.py`** — legacy migration
- **`test_fold.py`** / **`test_fold_lifecycle.py`** / **`test_rag.py`**
  — manual verification


## Design Rationale

Collected decisions that might look arbitrary:

- **Factory returns a runtime, not an app.** Web frameworks are a
  deployment concern. The engine stays framework-agnostic.
- **Provider abstraction at the boundary.** The engine uses its own
  message format. Providers translate at the edge. Adding a provider
  means one file, no engine changes.
- **Three-level model taxonomy.** Supports mixed local+API
  deployments in one registry.
- **Filesystem tool loading.** Plugin tools are user-owned files.
  No package structure means drag-and-drop installation.
- **`~/.{id}/{id}.db` derived from dashboard.json.** One identity
  declaration drives everything.
- **HTML comments for seq numbers.** Haiku echoed bracket formats;
  comments read as non-content metadata.
- **RAG blocks summary on failure.** No message passes the boundary
  without a vector.
- **`search_archive` not `search_memory`.** Haiku pattern-matched
  "search your memory" to a literal tool name.
- **`struct.pack` for vectors.** IEEE 754 binary beats JSON for
  high-dimensional floats in SQLite.
- **Brute-force cosine similarity.** At personal scale (thousands
  of vectors), Python brute force is fast enough.
- **Provider split (chat + summary).** Chat is the variable layer
  (swap models, switch to API). Summary is permanent infrastructure
  (one model, proven correct, never touched). The engine doesn't
  care if both providers point at the same server or different ones.
- **Summary model is pure transformer.** Hybrid SSM/attention
  architectures compress context into lossy state, destroying
  existing summaries instead of folding into them. Pure transformers
  handle cross-context attention correctly.
- **Convention over configuration.** `summary_model` has an or-guard
  default. Absent from config.toml, the engine picks the right value.
- **`claude.py` not `anthropic.py`.** Avoids shadowing the pip package.
- **Setup and infrastructure in the engine package.** Shim scripts in
  the agent repo stay stable. Logic lives behind pip so updates
  deliver new behavior to every agent.


## Files Trees

bountiful/
├── .github/
│   └── FUNDING.yml
├── .gitignore
├── ARCHITECTURE.md
├── LICENSE
├── README.md
├── add_secrets.py
├── add_tools.py
├── build.py
├── config.toml
├── dashboard.json
├── persona.md
├── pyproject.toml
├── run.py
└── tools/
    └── README.md


basic-bot/
├── .github/
│   └── FUNDING.yml
├── .gitignore
├── ARCHITECTURE.md
├── LICENSE
├── README.md
├── pyproject.toml
├── basic_bot/
│   ├── __init__.py
│   ├── __main__.py
│   ├── chat.py
│   ├── config.py
│   ├── diagnostics.py
│   ├── embeddings.py
│   ├── factory.py
│   ├── fold.py
│   ├── memory.py
│   ├── profile.py
│   ├── rag.py
│   ├── runtime.py
│   ├── secrets_env.py
│   ├── store.py
│   ├── store_sqlite.py
│   ├── summary.py
│   ├── tools.py
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   ├── llamacpp.py
│   │   ├── orchestration.py
│   │   └── server.py
│   ├── instructions/
│   │   └── capabilities.md
│   ├── profiles/
│   │   └── nvidia_12gb.toml
│   ├── providers/
│   │   ├── __init__.py
│   │   ├── claude.py
│   │   ├── local.py
│   │   └── protocol.py
│   ├── setup/
│   │   ├── __init__.py
│   │   ├── secrets.py
│   │   └── tools.py
│   └── tool_belt/
│       ├── __init__.py
│       ├── recall_message.py
│       └── search_archive.py
└── scripts/
    ├── backfill_rag.py
    ├── backfill_seq.py
    ├── rebuild_summary.py
    ├── reembed.py
    ├── test_fold.py
    ├── test_fold_lifecycle.py
    └── test_rag.py


basic-ui/
├── .github/
│   └── FUNDING.yml
├── ARCHITECTURE.md
├── LICENSE
├── README.md
├── pyproject.toml
└── basic_ui/
    ├── __init__.py
    ├── app.py
    ├── config.py
    ├── launch.py
    ├── static/
    │   ├── css/
    │   │   └── styles.css
    │   └── js/
    │       ├── chat.js
    │       ├── globals.d.ts
    │       └── jsconfig.json
    └── templates/
        └── index.html


extend-a-bot/
├── .github/
│   └── FUNDING.yml
├── .gitignore
├── ARCHITECTURE.md
├── LICENSE
├── README.md
├── version.json
└── github/
    ├── _auth.py
    ├── _config.py
    ├── create_branch.py
    ├── create_or_update_file.py
    ├── create_pull_request.py
    ├── delete_branch.py
    ├── delete_file.py
    ├── get_commit_history.py
    ├── get_repo_info.py
    ├── list_branches.py
    ├── list_repo_contents.py
    ├── list_repos.py
    ├── merge_pull_request.py
    ├── read_file.py
    └── tool.json