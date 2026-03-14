# RagStack

RagStack is an AI systems workspace that combines two independent projects into a modular stack for building and experimenting with real-world AI infrastructure.

The repository integrates two core services:

- **LatticeAI** — an OpenAI-compatible LLM gateway that manages API keys, routing, rate limiting, and usage tracking.
- **CitadelRag** — a multi-tenant Retrieval-Augmented Generation (RAG) platform for document ingestion, retrieval, and AI-powered chat.

Each service can run independently, but when connected together they form a complete AI system where **CitadelRag uses LatticeAI as the gateway for model access**.

---

# Why RagStack

Modern AI products are rarely a single application. They are systems composed of multiple services:

- Model gateways  
- Retrieval systems  
- APIs  
- Infrastructure layers  
- Observability and cost monitoring  

RagStack explores how to build these **modular AI architectures** rather than treating AI as a single script calling a model.

This repository focuses on:

- AI system design  
- Service orchestration  
- RAG pipelines  
- Model gateway infrastructure  
- Multi-service development workflows  

---

# Repository Structure

```
RagStack/
│
├── LatticeAI/        # LLM gateway infrastructure
├── CitadelRag/       # Multi-tenant RAG platform
│
├── .gitmodules
├── README.md
└── TECHNICAL.md
```

Both projects are included as **git submodules** and remain independent repositories.

---

# Projects

## LatticeAI

LatticeAI is the **gateway layer** of the stack.

It acts as an OpenAI-compatible proxy that sits between applications and model providers.

Core responsibilities:

- API key authentication  
- Request routing  
- Rate limiting  
- Usage logging  
- Cost monitoring  
- Provider abstraction  

This architecture allows applications to interact with models through a **controlled infrastructure layer** rather than calling providers directly.

Example use cases:

- Centralized LLM access  
- Provider switching  
- Request monitoring  
- Multi-tenant usage tracking  

---

## CitadelRag

CitadelRag is the **application layer** built on top of the gateway.

It focuses on building a complete **Retrieval-Augmented Generation platform** capable of ingesting documents, retrieving relevant context, and generating grounded responses using LLMs.

Core capabilities include:

- Document ingestion pipelines  
- Document chunking and preprocessing  
- Embedding generation  
- Vector search  
- Context retrieval  
- AI chat interfaces  
- Multi-tenant document environments  

When connected to LatticeAI, CitadelRag routes its LLM and embedding requests through the gateway.

---

# System Architecture

The integrated system follows a layered architecture.

## AI Request Flow

```
User / Application
        ↓
    CitadelRag
        ↓
   LatticeAI Gateway
        ↓
LLM / Embedding Provider
```

## Retrieval Workflow

```
Documents
    ↓
Ingestion Pipeline
    ↓
Text Chunking
    ↓
Embedding Generation
    ↓
Vector Database
    ↓
Context Retrieval
    ↓
LLM Prompt Construction
    ↓
Grounded Response
```

This architecture separates:

- **Application logic (CitadelRag)**  
- **Model gateway infrastructure (LatticeAI)**  

---

# Running the Projects

Both projects can run independently or together.

## Prerequisites

- Docker  
- Docker Compose  
- Optional for development: Python 3.11 and Node.js  

---

# Initial Setup

Create the shared Docker network used for communication between services.

```
docker network create shared-ai-net
```

---

# Running LatticeAI

```
cd LatticeAI
cp .env.example .env
docker compose -f docker/docker-compose.yml up -d
```

Access services:

- API → http://localhost:8000  
- UI → http://localhost:5173  

---

# Running CitadelRag

```
cd CitadelRag
cp env.example .env
docker compose -f docker/docker-compose.yml up -d
```

Access services:

- API → http://localhost:8001  
- UI → http://localhost:5174  

---

# Running the Full Stack

### Step 1 — Start LatticeAI

```
cd LatticeAI
docker compose -f docker/docker-compose.yml up -d
```

### Step 2 — Configure CitadelRag

Edit `CitadelRag/.env`:

```
LLM_PROVIDER=custom
EMBED_PROVIDER=custom

CUSTOM_LLM_BASE_URL=http://lattice-gateway:8000/v1
CUSTOM_EMBED_BASE_URL=http://lattice-gateway:8000/v1

LATTICE_API_KEY=your_lattice_api_key_here

VITE_API_URL=http://localhost:8001/api
```

### Step 3 — Start CitadelRag

```
cd CitadelRag
docker compose -f docker/docker-compose.yml up -d
```

### Step 4 — Verify Integration

1. Open the CitadelRag UI  
2. Submit a query  
3. Confirm the request appears in LatticeAI usage logs  

---

# Ports

## LatticeAI

- API → 8000  
- UI → 5173  
- Postgres → 5432  
- Redis → 6379  

## CitadelRag

- API → 8001  
- UI → 5174  
- Postgres → 5433  
- Redis → 6380  
- Qdrant → 6333  
- MinIO API → 9000  
- MinIO Console → 9001  

---

# Engineering Focus

RagStack explores practical AI engineering topics such as:

- Modular AI system architecture  
- Retrieval-augmented generation pipelines  
- LLM gateway design  
- Model provider abstraction  
- Service orchestration with Docker  
- Infrastructure-aware AI development  

Rather than focusing purely on ML models, the goal is to explore **how modern AI products are built as distributed systems**.

---

# Future Improvements

Potential areas for expansion include:

- Retrieval evaluation pipelines  
- LLM response benchmarking  
- Tenant-aware analytics  
- Cost monitoring dashboards  
- Multi-cloud deployment strategies  
- Automated infrastructure provisioning  
- Improved developer workflows for multi-repo systems  

---

# Author

**Godwin**

GitHub  
https://github.com/Godwin-T

Portfolio  
[Add your portfolio link here]

---

# License

MIT