# LatticeAI Monorepo

This repository contains two standalone projects that can run independently or together:

1. `LatticeAI/`  
   LLM gateway with API keys, routing, rate limiting, and usage logging.
2. `CitadelRag/`  
   Multi-tenant RAG platform with ingestion, retrieval, evaluation, and a web UI.

When run together, CitadelRag uses LatticeAI as its OpenAI-compatible gateway for chat and embeddings.

## Prerequisites

- Docker + Docker Compose
- Optional for local dev (without Docker): Python 3.11, Node 20

## Repo Layout

- `LatticeAI/`  
  Lattice gateway API, UI, Docker config.
- `CitadelRag/`  
  RAG API, worker, UI, Docker config.

## One-Time Setup

Create the shared Docker network used for cross-stack communication:

```bash
docker network create shared-ai-net
```

## Run Individually

### LatticeAI Only

1. Create the environment file:

```bash
cd LatticeAI
cp .env.example .env
```

2. Start services:

```bash
docker compose -f docker/docker-compose.yml up -d
```

3. Access:
- API: `http://localhost:8000`
- UI: `http://localhost:5173`

### CitadelRag Only

1. Create the environment file:

```bash
cd CitadelRag
cp env.example .env
```

2. Start services:

```bash
docker compose -f docker/docker-compose.yml up -d
```

3. Access:
- API: `http://localhost:8001`
- UI: `http://localhost:5174`

## Run Together

Run both stacks and connect CitadelRag to LatticeAI.

### 1) Start LatticeAI

```bash
cd LatticeAI
docker compose -f docker/docker-compose.yml up -d
```

### 2) Configure CitadelRag to use LatticeAI

Edit `CitadelRag/.env`:

```bash
LLM_PROVIDER=custom
EMBED_PROVIDER=custom
CUSTOM_LLM_BASE_URL=http://lattice-gateway:8000/v1
CUSTOM_EMBED_BASE_URL=http://lattice-gateway:8000/v1
LATTICE_API_KEY=your_lattice_api_key_here
VITE_API_URL=http://localhost:8001/api
```

Notes:
- `lattice-gateway` is the Docker network alias for the LatticeAI gateway.
- Generate Lattice API keys from the Lattice UI or API.

### 3) Start CitadelRag

```bash
cd CitadelRag
docker compose -f docker/docker-compose.yml up -d
```

### 4) Verify

- CitadelRag UI: `http://localhost:5174`
- Run a query in CitadelRag and confirm it succeeds.
- LatticeAI should show usage logs for those requests.

## Ports

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
- MinIO: `9000` (API), `9001` (Console)

## How to Add Another Project

1. Create a new folder at the repo root:
   - Example: `NewProject/`
2. Add a `docker-compose.yml` inside the new project.
3. Decide if the new project must talk to other stacks:
   - If yes, attach the relevant service to `shared-ai-net`.
   - Give the service a stable alias for DNS resolution.
4. Avoid port collisions:
   - Use unused host ports.
5. Update this root README:
   - Add the new project to the list.
   - Add run instructions and ports.

## Troubleshooting

1. `port is already allocated`
   - Change host ports in the relevant compose file.
2. `network shared-ai-net not found`
   - Run `docker network create shared-ai-net`.
3. `401 Invalid API key` in CitadelRag
   - Check `LATTICE_API_KEY` in `CitadelRag/.env`.
4. `Connection refused to lattice-gateway`
   - Ensure LatticeAI is running and on `shared-ai-net`.
