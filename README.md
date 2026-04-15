# CodeMind CLI

> Continuous, real-time codebase indexing via Git hooks — from cold start to instant sync.

CodeMind CLI is a lightweight, distributable Python package that sits directly inside a developer's repository and keeps the CodeMind knowledge graph perpetually up-to-date. Instead of re-indexing an entire codebase from scratch (which can take ~15 minutes), it watches Git's native lifecycle and pushes only the exact changed files to the CodeMind backend — reducing sync time to **~2 seconds per commit**.

---

## How It Works

```
git commit -m "fix bug"
       │
       ▼
  post-commit hook  (installed by codemind)
       │
       ▼
  codemind sync HEAD~1 HEAD
       │
       ├── diff_parser.py  →  detects modified / renamed / deleted files
       │
       ├── POST /api/sync/upsert  →  backend re-chunks & re-embeds changed files
       │
       └── POST /api/sync/delete  →  backend prunes stale nodes & vectors
```

The backend merges changes into Neo4j using Cypher `MERGE` statements and updates Qdrant vectors — no destructive full re-index.

---

## Architecture

The system is split into two decoupled components:

| Component | Responsibility |
|---|---|
| **CodeMind Server** | Hosts Neo4j + Qdrant, exposes REST/WebSocket APIs, manages auth |
| **CodeMind CLI** | Runs on the developer's machine, observes Git diffs, pushes deltas |

---

## Installation

```bash
pip install codemind-cli
```

Or install locally from source:

```bash
git clone https://github.com/your-org/codemind-cli
cd codemind-cli/cli
pip install -e .
```

---

## Quickstart

### 1. Authenticate

```bash
codemind login
```

Prompts for your CodeMind Auth Token and Server URL. Credentials are stored securely in `~/.codemind/credentials`.

### 2. Initialize a Repository

```bash
cd your-project/
codemind init
```

Checks whether this repository has been indexed before. If not, triggers a full initial ingest via `POST /api/ingest`.

### 3. Install the Git Hook

```bash
codemind install-hook
```

Writes an executable `post-commit` script into `.git/hooks/`. From this point on, every `git commit` automatically triggers an incremental sync — no manual intervention needed.

### 4. Manual Sync (Optional)

```bash
codemind sync
```

Manually parses `git diff` against the last synced commit and fires the incremental payload to the backend. Useful for CI pipelines or after rebases.

---

## CLI Commands

| Command | Description |
|---|---|
| `codemind login` | Save auth credentials to `~/.codemind/credentials` |
| `codemind init` | Initialize and fully index the current repository |
| `codemind sync [from] [to]` | Sync only changed files between two Git refs |
| `codemind install-hook` | Embed the `post-commit` hook into `.git/hooks/` |

---

## Backend API Endpoints

The CLI communicates with the CodeMind Server via two incremental sync endpoints:

### `POST /api/sync/upsert`
Accepts raw file contents for added or modified files. The backend parses, chunks, embeds, and merges them into the existing graph without touching unrelated nodes.

### `POST /api/sync/delete`
Accepts a list of deleted file paths. The backend locates all associated chunks and relationships in Neo4j and purges their vectors from Qdrant.

---

## Configuration: Fat Client vs. Thin Client

> **Decision Required**

Because the CLI pushes code directly to the backend, you must choose how AST chunking and embedding are handled:

**Option A — Fat Client (Distributed)**
- The CLI parses the AST and generates embeddings locally using **Ollama**.
- Zero API costs, fully distributed, works offline.
- Requires Ollama to be installed on the developer's machine.

**Option B — Thin Client (Server-side)**
- The CLI zips modified files and sends raw text to the server.
- Faster to build and deploy, simpler CLI codebase.
- Consumes server-side API keys (Gemini); adds network overhead.

Configure your preference in `~/.codemind/config.toml`:

```toml
[sync]
mode = "fat"   # or "thin"
ollama_model = "nomic-embed-text"
```

---

## Project Structure

```
codemind-cli/
├── cli/
│   ├── pyproject.toml          # Package configuration (Poetry / setuptools)
│   └── codemind/
│       ├── main.py             # CLI entry point (Typer / Click)
│       ├── diff_parser.py      # Git diff-tree parser for detecting file changes
│       └── hooks.py            # Git hook installer and post-commit script generator
└── backend/
    └── app/
        ├── codebase/
        │   └── sync.py         # Incremental upsert & delete API endpoints
        └── agents/
            └── graph_builder.py  # merge_graph() using Cypher MERGE
```

---

## Development & Testing

### Install in editable mode

```bash
cd cli/
pip install -e .
```

### Run tests

```bash
pytest tests/
```

### Integration test (end-to-end)

The test suite spins up a dummy Git repository, runs `codemind init`, applies a commit, and asserts that the Neo4j graph updates correctly using `MERGE` parameters:

```bash
pytest tests/integration/test_sync_flow.py -v
```

---

## Requirements

- Python 3.10+
- Git 2.x
- Access to a running CodeMind Server instance
- *(Fat Client only)* Ollama with a supported embedding model

---

## License

MIT © CodeMind Contributors
