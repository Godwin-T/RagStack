# Technical Overview

This document describes the architecture, runtime flows, data stores, and operational details for the monorepo.

## Scope

Two independent projects live in this repository:

1. `LatticeAI/`  
   OpenAI-compatible LLM gateway with API key management, provider routing, rate limiting, and usage logging.
2. `CitadelRag/`  
   Multi-tenant RAG platform (API + worker + UI) with ingestion, vector search, and evaluation.

When run together, CitadelRag uses LatticeAI as its LLM and embeddings gateway.

## Repository Layout

- `LatticeAI/`  
  API service, UI, Docker config, migrations.
- `CitadelRag/`  
  API service, worker, UI, Docker config, storage.
- Root  
  Coordination docs (`README.md`, this file).

## Architecture Overview

Text diagram of the combined system:

1. User → CitadelRag UI → CitadelRag API  
2. CitadelRag API → LatticeAI Gateway  
3. LatticeAI Gateway → Provider (OpenAI / Groq / Anthropic)  
4. Responses → CitadelRag API → UI

CitadelRag worker performs background jobs (ingest, memory summaries, evals) and also calls LatticeAI for LLM or embeddings.

## LatticeAI Technical Overview

### Core Services

- FastAPI Gateway: `LatticeAI/app/main.py`
- Database: Postgres
- Cache/Rate limit store: Redis
- UI: React in `LatticeAI/ui`

### API Endpoints (Core)

- `/v1/chat/completions`
- `/v1/embeddings`
- `/keys` (API key CRUD)
- `/usage`, `/requests`, `/dashboard` (usage reporting)

### Auth Flow

- API key is sent as `Authorization: Bearer <key>`.
- Keys are hashed and verified server-side.
- Key validation also checks key status and owner user status.

### Provider Routing

- Providers are selected by configuration and/or request payload.
- Fallback is supported based on configured order and error type.
- Token usage is extracted from provider responses for billing and usage logs.

### Rate Limiting

- Redis-backed token bucket using Lua script.
- Per API key: `rl:<api_key_id>`
- Capacity = `RATE_LIMIT_RPM`

### Observability

- JSON logs via `structlog`.
- Optional OpenTelemetry tracing (FastAPI + HTTPX).

## CitadelRag Technical Overview

### Core Services

- FastAPI API: `CitadelRag/api`
- Worker: `CitadelRag/worker`
- UI: `CitadelRag/ui`
- Data Stores: Postgres, Redis, Qdrant, MinIO

### RAG Flow

1. Ingestion queues document jobs in Redis.
2. Worker processes, chunks, embeds, and stores vectors in Qdrant.
3. Query:
   - Embed query
   - Retrieve top chunks from Qdrant
   - Build prompt
   - Call LLM
   - Persist query + answer + citations

### Chat Architecture

CitadelRag exposes a chat-first interface that wraps the RAG query pipeline.

Flow:

1. UI creates a chat session per tenant.
2. Each user message is stored as a chat message.
3. The message is passed through the same RAG query pipeline (`/chat/messages` → `run_query`).
4. Assistant response is stored as a chat message with citations.

Key tables:

- `chat_sessions` (tenant_id, user_id, title)
- `chat_messages` (session_id, role, content, citations)

### Memory & Evaluation

- Memory summaries are generated after N turns.
- Eval runs score retrieval/answer quality.

## Cross-Stack Integration

CitadelRag uses LatticeAI as a custom provider:

- `LLM_PROVIDER=custom`
- `EMBED_PROVIDER=custom`
- `CUSTOM_LLM_BASE_URL=http://lattice-gateway:8000/v1`
- `CUSTOM_EMBED_BASE_URL=http://lattice-gateway:8000/v1`

Auth is passed via:

- `Authorization: Bearer <Lattice API key>`

Key storage is per user + tenant (stored in `user_settings.lattice_api_key`) and sent via override headers.

## Data Stores and Schemas

### LatticeAI (Postgres)

Key tables:

- `orgs`, `projects`, `users`, `org_memberships`
- `api_keys` (hashed keys + status)
- `request_logs` (provider, model, latency, tokens, cost)
- `model_pricing`

### CitadelRag (Postgres)

Key tables:

- `tenants`, `users`, `memberships`
- `documents`, `document_files`
- `chunks`, `embeddings`, `embedding_versions`
- `queries`, `answers`
- `user_settings` (includes `lattice_api_key`)
- `eval_sets`, `eval_runs`, `events`

## Environment Variables

### LatticeAI

Key variables:

- `DATABASE_URL`
- `REDIS_URL`
- `OPENAI_API_KEY`, `GROQ_API_KEY`, `ANTHROPIC_API_KEY`
- `RATE_LIMIT_RPM`
- `OTEL_ENABLED`, `OTEL_EXPORTER_OTLP_ENDPOINT`

### CitadelRag

Key variables:

- `DATABASE_URL`
- `REDIS_URL`
- `QDRANT_URL`
- `S3_ENDPOINT` + MinIO keys
- `LLM_PROVIDER`, `EMBED_PROVIDER`
- `CUSTOM_LLM_BASE_URL`, `CUSTOM_EMBED_BASE_URL`
- `LATTICE_API_KEY`

## Docker and Networking

Two compose files are used:

- `LatticeAI/docker/docker-compose.yml`
- `CitadelRag/docker/docker-compose.yml`

Cross-stack communication uses the external network:

```bash
docker network create shared-ai-net
```

LatticeAI gateway is aliased as `lattice-gateway` on that network.

### Ports

LatticeAI:

- API: `8000`
- UI: `5173`
- Postgres: `5432`
- Redis: `6379`

CitadelRag:

- API: `8001`
- UI: `5174`
- Postgres: `5433`
- Redis: `6380`
- Qdrant: `6333`
- MinIO: `9000` and `9001`

## Operational Workflows

### Migrations

LatticeAI:

- Alembic migrations from `LatticeAI/alembic`

CitadelRag:

- Alembic migrations from `CitadelRag/api/alembic`
- Run before API start in production

### API Key Lifecycle

- Keys are created in LatticeAI.
- Keys are stored in CitadelRag per user + tenant.
- Rotate keys by updating `user_settings.lattice_api_key`.

## Security Notes

- LatticeAI stores key hashes, not raw keys.
- CitadelRag stores the raw Lattice API key for downstream requests.
- Prompt content is not logged in LatticeAI by default.

## Troubleshooting

Common issues:

- Port conflicts: adjust host ports in compose files.
- Network not found: run `docker network create shared-ai-net`.
- 401 from LatticeAI: check `LATTICE_API_KEY` in CitadelRag settings.
- Connection refused to `lattice-gateway`: LatticeAI is not running or not attached to shared network.
