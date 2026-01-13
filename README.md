# greg — Knowledge Graph RAG (Microsoft GraphRAG 2.x)

Full-stack app for turning documents (PDF/DOCX/TXT/MD) into a **GraphRAG 2.x** index and then running **Graph RAG Q&A** + **graph visualization** over the extracted entities/relationships.

## Tech stack (what’s actually in this repo)

### Backend (`backend/`)
- **Runtime**: Python 3.x
- **API framework**: **FastAPI** + **Uvicorn**
- **GraphRAG**: **Microsoft `graphrag`** (`graphrag.api.build_index`, `graphrag.api.global_search`)
- **LLM/Embeddings providers**:
  - **Azure OpenAI** (via `openai` Python SDK `AsyncAzureOpenAI`)
  - **OpenAI / OpenAI-compatible** (via `openai` Python SDK `AsyncOpenAI`)
  - **Groq**: partially wired (OpenAI-compatible base URL) for **GraphRAG indexing**; Q&A path currently supports **only `openai` + `azure`**.
- **Document parsing**: `pypdf`, `python-docx`
- **Chunking**: `langchain-text-splitters` (`RecursiveCharacterTextSplitter`)
- **Data layer**: `pandas` + `pyarrow` (GraphRAG outputs are parquet files)
- **Config**: `pydantic-settings` (reads env vars), plus runtime updates via `/api/graph/config`

### Frontend (`frontend/`)
- **Build tooling**: **Vite** (dev server defaults to **port 8080**)
- **UI**: **React 18 + TypeScript**, **Tailwind CSS**, **shadcn/ui** (Radix primitives)
- **State**: **Zustand** (persisted API base URL + local credentials)
- **Data fetching**: **@tanstack/react-query**
- **Graph rendering**: **react-force-graph-2d**
- **Routing**: **react-router-dom**

## Architecture (current implementation)

1. **Upload** a document to the backend (`POST /api/graph/upload`).
2. Backend extracts text, builds a GraphRAG 2.x config, and runs **`build_index`**.
3. GraphRAG writes outputs (notably `entities.parquet`, `relationships.parquet`, …) to a per-`group_id` workspace under the OS temp directory.
4. **Search/Q&A** uses GraphRAG’s **`global_search`** + an LLM call to generate a final answer.
5. **Visualization** reads `entities.parquet` + `relationships.parquet` and returns a node/edge list consumed by the React graph view.

> Note: You’ll see **Neo4j** fields in config/UI, but the current backend codebase **does not use a Neo4j driver** and does not persist data to Neo4j (no `neo4j` dependency is installed). Graph state currently lives in the GraphRAG workspace on disk.

## Running locally

### Prereqs
- **Python 3.10+** (recommended) for the backend
- **Node.js 18+** + npm for the frontend

### 1) Start the backend (FastAPI)

From the repo root:

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt

# Set env vars (examples below), then:
python .\main.py
```

Backend defaults:
- **API**: `http://localhost:8000`
- **Swagger docs**: `http://localhost:8000/docs`
- **API base path**: `http://localhost:8000/api/graph`

### 2) Start the frontend (Vite)

From the repo root:

```powershell
cd frontend
npm install
npm run dev
```

Frontend defaults:
- **App**: `http://localhost:8080`
- **Backend base URL setting**: in the UI, set to `http://localhost:8000/api/graph` (default)

## Configuration

### Recommended: configure dynamically via API

The backend exposes a unified configuration endpoint:
- `GET /api/graph/config` — fetch current (masked) config
- `POST /api/graph/config` — update environment variables at runtime

Example:

```bash
curl -X POST http://localhost:8000/api/graph/config \
  -H "Content-Type: application/json" \
  -d '{
    "env_vars": {
      "LLM_PROVIDER": "openai",
      "EMBEDDING_PROVIDER": "openai",
      "OPENAI_API_KEY": "sk-...",
      "OPENAI_MODEL": "gpt-4o",
      "OPENAI_EMBEDDING_MODEL": "text-embedding-3-small"
    }
  }'
```

### Environment variables (backend)

Provider selection:
- **`LLM_PROVIDER`**: `openai` | `azure` *(indexing also has some `groq` wiring; Q&A does not)*
- **`EMBEDDING_PROVIDER`**: `openai` | `azure`

OpenAI:
- **`OPENAI_API_KEY`**
- `OPENAI_MODEL` (default: `gpt-4o`)
- `OPENAI_EMBEDDING_MODEL` (default: `text-embedding-3-small`)
- `OPENAI_BASE_URL` (optional; use for OpenAI-compatible gateways)

Azure OpenAI:
- **`AZURE_OPENAI_ENDPOINT`**
- **`AZURE_OPENAI_API_KEY`**
- `AZURE_OPENAI_API_VERSION` (default in code: `2024-02-01`)
- `AZURE_OPENAI_DEPLOYMENT_NAME` (default: `gpt-4o`)
- `AZURE_OPENAI_EMBEDDING_DEPLOYMENT`

Chunking:
- `CHUNK_SIZE` (default: `1000`)
- `CHUNK_OVERLAP` (default: `200`)

Server:
- `HOST` (default: `0.0.0.0`)
- `PORT` (default: `8000`)
- `DEBUG` (default: `true`)

## API surface (backend)

Core endpoints (base: `/api/graph`):
- **`POST /upload`**: multipart upload (`file`, optional `group_id`)
- **`POST /ask`**: question answering (`question`, optional `group_id`, `top_k`)
- **`GET /visualize`**: nodes/edges for the graph view (optional `group_id`, `limit`)
- **`GET /stats`**: basic counts derived from GraphRAG output
- **`POST /search`**: GraphRAG global search
- **`GET /health`**: checks GraphRAG availability (and returns a “neo4j_connected” flag for compatibility)
- **`GET /groups`**: list groups seen in the current process
- **`GET|POST /config`**: unified runtime config

Full details + request/response examples: see `backend/api.md`.

## Where the “graph” is stored

GraphRAG writes per-group workspaces under your OS temp directory:
- Workspace name pattern: `graphrag_{group_id}`
- Key output files used by the API:
  - `output/entities.parquet`
  - `output/relationships.parquet`
  - `output/communities.parquet`
  - `output/community_reports.parquet`

On Windows, the temp directory is typically `%TEMP%`.

## Project layout

```
backend/
  main.py                 # FastAPI app + CORS
  config.py               # env var config + runtime updates
  routers/graph_router.py # REST endpoints
  services/
    document_processor.py # PDF/DOCX/TXT/MD extraction
    graph_service.py      # GraphRAG build_index/global_search + parquet I/O
    rag_service.py        # LLM answer synthesis (OpenAI/Azure via openai SDK)
  models/schemas.py       # Pydantic request/response models

frontend/
  vite.config.ts          # dev server port 8080
  src/lib/api.ts          # typed API client (base URL configurable)
  src/stores/*            # Zustand stores (API URL + credentials)
  src/components/*        # chat + graph + settings UI
```

## Notes / troubleshooting

- **GraphRAG not available**: the backend lazy-imports `graphrag`; if install/import fails, `/health` will degrade and graph ops won’t work. Reinstall backend deps and ensure `pyarrow` works on your platform.
- **No persistence across server restarts**: group workspaces are stored under temp; restarting the backend can “forget” groups until you upload again.
- **Neo4j config**: currently present in config/UI but not used by the backend code in this repo.

