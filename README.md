# Multi-Agent AI

A self-hosted enterprise AI platform built on a multi-agent LangGraph pipeline. It answers questions from your documents, keeps memory across sessions, builds a knowledge graph, processes meeting recordings, browses the web and talks back by voice. Everything runs on free, open-source tools — no paid API keys required.

![CI](https://github.com/debasish218/multi-agent-ai/actions/workflows/ci.yml/badge.svg)

<!-- Screenshots: save images in docs/screenshots/ with these names -->
![Dashboard](docs/screenshots/dashboard.png)

## Features

- **Multi-agent chat** — Planner → Researcher → Executor → Critic pipeline (LangGraph) with streamed answers and a quality gate that retries weak responses.
- **Adaptive retrieval** — routes each query to vector search (Qdrant), keyword search (Elasticsearch), graph lookup (Neo4j) or web search (DuckDuckGo).
- **Document Q&A** — upload PDFs; they are chunked, embedded and indexed automatically.
- **Layered memory** — short-term (Redis) and long-term semantic memory (Qdrant) that persists across sessions.
- **Knowledge graph** — entities and relationships extracted from documents into Neo4j, with an interactive graph view.
- **Meeting intelligence** — transcribes recordings with faster-whisper and extracts summaries, decisions and action items.
- **Browser agent** — headless Chromium (Playwright) that navigates sites and extracts information on its own.
- **Voice AI** — real-time voice conversation over WebSocket (faster-whisper → LLM → edge-tts).
- **Evaluation & observability** — LLM-as-judge quality metrics, live agent status, Prometheus/Grafana dashboards and optional OpenTelemetry tracing.

## Screenshots

| Chat | Documents |
|------|-----------|
| ![Chat](docs/screenshots/chat.png) | ![Documents](docs/screenshots/documents.png) |

| Knowledge Graph | Voice |
|-----------------|-------|
| ![Knowledge Graph](docs/screenshots/knowledge-graph.png) | ![Voice](docs/screenshots/voice.png) |

| Browser Agent | Evaluation |
|---------------|------------|
| ![Browser Agent](docs/screenshots/browser.png) | ![Evaluation](docs/screenshots/eval.png) |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, Vite, TypeScript, Tailwind CSS, Zustand |
| API | FastAPI (REST, SSE, WebSocket) |
| Agents | LangGraph |
| LLM & embeddings | Ollama (`llama3.1`, `nomic-embed-text`) |
| Databases | PostgreSQL, Redis, Qdrant, Neo4j, Elasticsearch |
| Speech | faster-whisper (STT), edge-tts (TTS) |
| Browser automation | Playwright |
| Observability | Prometheus, Grafana, OpenTelemetry, Jaeger, LangSmith (optional) |

## Architecture

```
React SPA ──HTTP / SSE / WebSocket──▶ FastAPI (:8000)
                                        │
   /api/chat/stream ── LangGraph ───────┤  Planner → Researcher → Executor → Critic → Memory
                                        │              │
                                        │              ├─ Vector   → Qdrant
                                        │              ├─ Keyword  → Elasticsearch
                                        │              ├─ Graph    → Neo4j
                                        │              └─ Web      → DuckDuckGo
   /api/documents/*  ───────────────────┤  PDF → chunks → Qdrant + Elasticsearch + Neo4j
   /api/meetings/*   ───────────────────┤  faster-whisper + Ollama → PostgreSQL
   /api/browser/*    ───────────────────┤  Playwright + Ollama
   /api/eval/*       ───────────────────┤  LLM-as-judge metrics
   /ws/voice         ───────────────────┘  faster-whisper → Ollama → edge-tts
```

## Getting Started

### Prerequisites

- Docker and Docker Compose
- ~16 GB RAM (Ollama and Elasticsearch are memory-hungry)
- For local development: Python 3.11+, Node.js 20+ and [Ollama](https://ollama.com)

### Option A — Docker Compose

```bash
git clone https://github.com/debasish218/multi-agent-ai.git
cd multi-agent-ai
cp .env.example .env          # change SECRET_KEY at minimum

docker compose up -d --build

# Pull the models (first run only)
docker compose exec ollama ollama pull llama3.1
docker compose exec ollama ollama pull nomic-embed-text
```

- App: http://localhost
- API docs: http://localhost:8000/docs

### Option B — Local development

```bash
# 1. Infrastructure
docker compose up -d postgres redis qdrant neo4j elasticsearch

# 2. Models
ollama pull llama3.1
ollama pull nomic-embed-text

# 3. Backend
cd backend
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/playwright install chromium
.venv/bin/alembic upgrade head
./start_dev.sh                # backend on http://localhost:8000

# 4. Frontend (new terminal)
cd frontend
npm install
npm run dev                   # http://localhost:5173
```

> `docker-compose.override.yml` maps PostgreSQL to host port **5433** — adjust `POSTGRES_URL` accordingly when running the backend outside Docker.

### Create a user

```bash
curl -X POST http://localhost:8000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","email":"admin@demo.com","password":"admin123"}'
```

### Observability (optional)

```bash
docker compose -f docker-compose.observability.yml up -d
```

Grafana: http://localhost:3001 · Jaeger: http://localhost:16686

## Configuration

All settings are documented in [.env.example](.env.example). Key ones:

| Variable | Purpose |
|----------|---------|
| `SECRET_KEY` | JWT signing key |
| `POSTGRES_URL`, `REDIS_URL` | Database connections |
| `QDRANT_HOST`, `ELASTICSEARCH_URL` | Search backends |
| `OLLAMA_HOST`, `OLLAMA_PORT` | Local LLM server |
| `WHISPER_MODEL` | faster-whisper model size |
| `LANGSMITH_API_KEY`, `OTEL_EXPORTER_OTLP_ENDPOINT` | Optional tracing |

## Project Structure

```
├── backend/            FastAPI app, LangGraph agents, services, Alembic migrations
├── frontend/           React + Vite single-page app
├── observability/      Prometheus config and Grafana dashboards
├── deployment/         Railway and Vercel configs
├── docs/               Detailed project guide
└── .github/workflows/  CI (lint, type check, build) and deploy
```

## Documentation

- [Project Guide](docs/PROJECT_GUIDE.md) — component workflows, full API reference and frontend pages
- [Implementation Tracker](IMPLEMENTATION.md) — phase-by-phase build plan
- [Deployment Guide](deployment/README.md) — Docker, Railway and Vercel
