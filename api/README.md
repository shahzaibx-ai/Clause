# Clause API

A FastAPI backend that answers customer questions from the Clause Shop handbook using RAG (Retrieval-Augmented Generation).

It loads the handbook PDF, splits it into chunks, embeds them into a Qdrant vector store, and then answers questions using only the most relevant chunks. By default everything runs locally with Ollama.

## Tech stack

- **FastAPI**: HTTP API
- **LangChain**: model and embedding setup (`init_chat_model`, `init_embeddings`)
- **Ollama**: local chat model (`gemma3:1b`) and embeddings (`nomic-embed-text`)
- **Qdrant**: vector store (embedded on disk by default, or a server)
- **pydantic-settings**: configuration from `.env`

## Project layout

```
api/
├── app/
│   ├── main.py            # FastAPI app and routes
│   ├── cli.py             # Ask a question from the terminal
│   ├── core/
│   │   └── config.py      # Settings, embeddings and Qdrant client
│   └── services/
│       ├── ingestion.py   # Load → chunk → embed → store
│       └── chat.py        # RAGService: retrieve → prompt → answer
├── files/
│   └── shop_handbook.pdf  # Source document
├── vector_stores/         # Local vector store (created by ingestion)
├── .env.example
└── pyproject.toml
```

## Prerequisites

- Python 3.13+
- [uv](https://docs.astral.sh/uv/)
- [Ollama](https://ollama.com/) running locally

Pull the models:

```bash
ollama pull gemma3:1b
ollama pull nomic-embed-text
```

## Setup

Run all commands from the `api/` folder.

```bash
cd api
uv sync
cp .env.example .env
```

## 1. Ingest the handbook

This builds the Qdrant collection from `files/shop_handbook.pdf`. Run it once, and again whenever the PDF or the chunking and embedding settings change. It deletes and rebuilds the collection each time.

```bash
uv run python -c "from app.services.ingestion import ingest; print(ingest(), 'chunks indexed')"
```

## 2. Run the API

```bash
uv run fastapi dev app/main.py
```

The API runs at http://127.0.0.1:8000. Interactive docs are at http://127.0.0.1:8000/docs.

### Endpoints

| Method | Path   | Description                          |
|--------|--------|--------------------------------------|
| GET    | `/`    | Health check: `{"status": "ok"}`     |
| POST   | `/ask` | Ask a question about the handbook    |

**Request**

```bash
curl -X POST http://127.0.0.1:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the return policy?"}'
```

`question` is required. Whitespace is stripped, and it must be 1–2000 characters long.

**Response**

```json
{
  "answer": "…",
  "sources": [
    { "source": "files/shop_handbook.pdf", "page": 2, "score": 0.7812 }
  ]
}
```

If nothing relevant is retrieved, `answer` says the handbook doesn't cover the question and `sources` is empty.

## Ask from the terminal

```bash
uv run python -m app.cli
```

This asks one question and prints the answer.

> **Note:** embedded Qdrant (`RAG_QDRANT_PATH`) can only be opened by one process at a time. Stop the API before running the CLI or ingestion, or point `RAG_QDRANT_URL` at a Qdrant server.

## Configuration

All settings are read from `api/.env`, and each one needs the `RAG_` prefix. See [`.env.example`](.env.example) for the full list.

| Variable               | Default                   | Purpose                                     |
|------------------------|---------------------------|---------------------------------------------|
| `RAG_CHAT_PROVIDER`    | `ollama`                  | Chat model provider                         |
| `RAG_CHAT_MODEL`       | `gemma3:1b`               | Chat model name                             |
| `RAG_CHAT_TEMPERATURE` | `0.1`                     | Sampling temperature                        |
| `RAG_EMBED_PROVIDER`   | `ollama`                  | Embedding provider                          |
| `RAG_EMBED_MODEL`      | `nomic-embed-text:latest` | Embedding model                             |
| `RAG_EMBED_DIM`        | `768`                     | Must match the embedding model's output size |
| `RAG_CHUNK_SIZE`       | `600`                     | Characters per chunk                        |
| `RAG_CHUNK_OVERLAP`    | `100`                     | Overlap between chunks                      |
| `RAG_TOP_K`            | `3`                       | Chunks retrieved per question               |
| `RAG_COLLECTION_NAME`  | `shop_handbook`           | Qdrant collection                           |
| `RAG_QDRANT_PATH`      | `./vector_stores`         | Local Qdrant storage                        |
| `RAG_QDRANT_URL`       | *(unset)*                 | Qdrant server URL (overrides the path)      |

To use another provider, such as Anthropic, OpenAI or Google, install its LangChain package and set the provider, model and API key as shown in `.env.example`. If you change the embedding model, update `RAG_EMBED_DIM` and run ingestion again.
