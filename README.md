# Clause

A customer-support assistant for Clause Shop, built with RAG (Retrieval-Augmented Generation) on a local Ollama setup.

Ask a question and Clause looks it up in the shop handbook. It answers only from what it finds, and returns the sources it used.

## How it works

```
shop_handbook.pdf → chunk → embed → Qdrant
                                       │
question ──→ embed ──→ top-k search ───┘──→ LLM (handbook context only) ──→ answer + sources
```

## Project layout

| Folder       | What it holds                                              |
|--------------|------------------------------------------------------------|
| `api/`       | FastAPI RAG backend (LangChain + Ollama + Qdrant)          |
| `web/`       | Frontend (coming soon)                                     |
| `notebooks/` | Step-by-step learning notebooks (local only, not tracked)  |
| `playground/`| Standalone RAG experiments (local only, not tracked)       |

## Quick start

```bash
ollama pull gemma3:1b
ollama pull nomic-embed-text

cd api
uv sync
cp .env.example .env
uv run python -c "from app.services.ingestion import ingest; ingest()"
uv run fastapi dev app/main.py
```

Then open http://127.0.0.1:8000/docs and try `POST /ask`.

See [`api/README.md`](api/README.md) for endpoints, configuration and the CLI.
