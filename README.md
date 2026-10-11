# SupportFlow AI

> Local-first, production-style **agentic customer support platform** — RAG, tool-calling, human escalation, ticketing, feedback, MLflow experiment tracking, and monitoring. Runs entirely on your laptop with **no paid APIs**.

SupportFlow AI is a portfolio-grade **GenAI + LLMOps** project targeting AI Engineer roles. It is **not a chatbot**: a LangGraph agent classifies intent, retrieves company docs via RAG, calls real business tools over PostgreSQL, validates its own answers, creates support tickets, escalates to humans when uncertain — all with full observability and experiment tracking.

---

## Architecture

```
Customer ──► Next.js Chat UI ──► FastAPI ──► LangGraph Agent
                                                │
                              ┌─────────────────┼─────────────────┐
                              │                 │                 │
                         RAG Node         Tool Node        Quality Check
                         (ChromaDB)       (7 DB tools)     (hallucination guard)
                              │                 │                 │
                              └─────────────────┼─────────────────┘
                                                │
                                    Response + Sources
                                                │
                              ┌─────────────────┼─────────────────┐
                              │                 │                 │
                        PostgreSQL          MLflow           Prometheus
                        (persist)          (tracking)        (metrics)
```

**Agent intents handled:** FAQ / policy, order status, subscription, refund eligibility, payment / billing, escalation requests, complaints, and unknown (graceful no-hallucination fallback).

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 15, TypeScript, Tailwind CSS |
| Backend | FastAPI, Python 3.12, Pydantic v2, SQLAlchemy 2.0 |
| Local LLM | Ollama — `mistral:7b` (chat), `nomic-embed-text` (embeddings) |
| Agent workflow | LangGraph |
| LLM orchestration | LangChain |
| Vector DB | ChromaDB (local, persistent) |
| Main DB | PostgreSQL 16 (Docker) |
| Experiment tracking | MLflow (local) |
| Evaluation | DeepEval — deterministic checks + LLM-as-judge via Ollama |
| Monitoring | Prometheus + Grafana |
| Testing | Pytest + Playwright |
| Deployment | Docker Compose (local) |

---

## Build Status

| Phase | Status | Description |
|---|---|---|
| 0 — Repo scaffolding | ✅ | Folder structure, `.gitignore`, `.env.example` |
| 1 — FastAPI backend | ✅ | `GET /health`, typed config (pydantic-settings), CORS, structured logging |
| 2 — PostgreSQL + Alembic | ✅ | 11 tables, SQLAlchemy 2.0 models, single migration |
| 3 — Seed data + KB docs | ✅ | 5 users, 5 orders, 3 subscriptions, 4 payments, 8 policy docs |
| 4 — ChromaDB + RAG | ✅ | 29 chunks across 8 docs; retrieval verified with source attribution |
| 5 — Ollama + LangChain chat | ✅ | `POST /api/chat`; RAG → LLM pipeline; 503 on Ollama down; 13 unit tests |
| 5.5 — Early chat frontend | ✅ | Next.js `/chat` page; suggested questions; source cards; loading/error states |
| 6 — Business tools | ✅ | 7 DB tools with Pydantic schemas; 25/25 tests; duplicate payment detection |
| 7 — LangGraph agent | ✅ | StateGraph; 8 intents; hallucination guard; 69/69 tests; routes `/api/chat` |
| 8 — Persistence + APIs | ✅ | Conversation/message/tool-call persistence; 8 REST endpoints; 95/95 tests |
| 9 — Admin dashboard | ✅ | `/admin`, `/admin/tickets/[id]`, `/admin/knowledge-base`, `/admin/conversations/[id]` |
| 10 — Customer chat upgrade | 🚧 | Multi-turn `conversation_id`, feedback buttons, ticket status card |
| 11 — MLflow tracking | ✅ | Logs every chat run: intent, route, latency, tool calls, escalation flags |
| 12 — DeepEval evaluation | ⬜ | Deterministic checks + LLM-as-judge; `/api/evaluations` endpoint |
| 13 — Prometheus + Grafana | ⬜ | `/metrics` endpoint; provisioned Grafana dashboard |
| 14 — Full Docker Compose | ⬜ | One-command local stack: postgres + backend + frontend + mlflow + monitoring |
| 15 — Automated tests | ⬜ | Pytest suite + Playwright E2E |
| 16 — Final polish | ⬜ | Full README, demo script, UI/UX pass, screenshots |

---

## Project Structure

```
supportflow/
├── docker-compose.yml          # Postgres 16 (port 5433) — docker compose up -d postgres
├── .env.example                # All config vars — copy to .env and edit
├── claude-docs/                # Execution plan, phase notes, session handoffs
├── data/
│   ├── knowledge_base/         # 8 CloudDesk policy docs (*.md)
│   └── chroma/                 # Persisted vector store (gitignored — regenerate with ingest)
├── infra/                      # Prometheus config, Grafana provisioning (Phase 13)
├── backend/
│   ├── requirements.txt
│   ├── alembic/                # DB migrations
│   └── app/
│       ├── main.py             # App factory, CORS, routers
│       ├── core/               # Config (pydantic-settings), structured logging
│       ├── db/                 # Session, DeclarativeBase, seed script
│       ├── models/             # 11 SQLAlchemy models
│       ├── schemas/            # Pydantic request/response schemas
│       ├── api/                # Routers: chat, conversations, tickets, feedback, KB, admin
│       ├── services/           # Business logic layer
│       ├── rag/                # Ingest + retriever (ChromaDB + Ollama embeddings)
│       ├── llm/                # ChatOllama wrapper, prompt templates
│       ├── tools/              # 7 business tools over Postgres
│       ├── agent/              # LangGraph StateGraph + intent classifier + nodes
│       ├── evaluation/         # DeepEval suite (Phase 12)
│       └── observability/      # MLflow tracking, Prometheus metrics
└── frontend/
    └── app/
        ├── chat/               # Customer chat page
        └── admin/              # Dashboard, tickets, KB, conversations
```

---

## Quick Start

### Prerequisites

- Python 3.12+
- Node 20+
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (running)
- [Ollama](https://ollama.com) (running)

### 1. Clone and configure

```bash
git clone https://github.com/abdullahahsen05/supportflow-ai.git
cd supportflow-ai
cp .env.example .env
```

### 2. Pull Ollama models

```bash
ollama pull nomic-embed-text   # ~270 MB — embeddings
ollama pull mistral:7b         # ~4 GB  — chat LLM
```

### 3. Start PostgreSQL

```bash
docker compose up -d postgres
```

> Runs on port **5433** (avoids conflicts with any local Postgres on 5432).

### 4. Set up the backend

```powershell
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1          # Windows PowerShell
# source .venv/bin/activate          # macOS / Linux

pip install -r requirements.txt
alembic upgrade head                 # create all tables
python -m app.db.seed                # seed fake company data
python -m app.rag.ingest             # embed KB docs into ChromaDB
uvicorn app.main:app --reload        # start API on :8000
```

### 5. Set up the frontend

```bash
cd frontend
npm install
npm run dev          # start on :3000
```

### 6. Verify

| URL | What you see |
|---|---|
| `http://localhost:8000/health` | `{"status": "ok", ...}` |
| `http://localhost:8000/docs` | Swagger UI |
| `http://localhost:3000/chat` | Customer chat page |
| `http://localhost:3000/admin` | Admin dashboard |

---

## Business Tools (Phase 6)

The agent has access to 7 typed tools that query real PostgreSQL data:

| Tool | Description |
|---|---|
| `get_order_status` | Fetch order by number — status, tracking, estimated delivery |
| `get_user_profile` | Look up user by email — name, role, subscription |
| `get_subscription_status` | Active plan, billing period, status |
| `check_payment_history` | Recent transactions, amounts, statuses |
| `check_refund_eligibility` | Policy check: 14-day window, order status, prior refunds |
| `create_support_ticket` | Creates a DB ticket with category, priority, summary |
| `escalate_to_human` | Flags conversation for agent review, creates escalation ticket |

---

## Agent Workflow (Phase 7)

```
Input
  │
  ▼
Intent Classifier (keyword + LLM fallback)
  │
  ├──► FAQ / Policy    ──► RAG Retrieval ──► LLM Answer
  ├──► Order Status    ──► get_order_status tool
  ├──► Subscription    ──► get_subscription_status tool
  ├──► Refund          ──► check_refund_eligibility tool
  ├──► Payment/Billing ──► check_payment_history tool
  ├──► Complaint       ──► escalate_to_human + create_support_ticket
  ├──► Escalation Req  ──► escalate_to_human + create_support_ticket
  └──► Unknown         ──► Honest "I don't know" (no hallucination)
          │
          ▼
    Quality Check (strip hallucinated ticket IDs)
          │
          ▼
    Final Response + Sources + Ticket (if created)
```

---

## Demo Scenarios

After setup, try these in the chat UI:

```
"What is your refund policy?"
→ RAG answer from refund_policy.md with source attribution

"What is the status of order 1004?"
→ tool call to get_order_status → "Shipped, tracking: TRK-CD-1004"

"I was charged twice for $79.99 — TXN-AYE-001 and TXN-AYE-002"
→ check_payment_history detects duplicate → escalation ticket created

"I want to cancel my subscription"
→ subscription_policy.md retrieved → cancellation steps returned

"THIS IS RIDICULOUS I WANT TO SPEAK TO A HUMAN"
→ complaint intent → escalation + ticket created automatically
```

---

## Configuration

All configuration is loaded from `.env` (or environment variables). See `.env.example` for the full list.

| Variable | Default | Description |
|---|---|---|
| `APP_ENV` | `development` | Environment tag |
| `DATABASE_URL` | `postgresql+psycopg://...@localhost:5433/supportflow` | Postgres connection |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama server |
| `OLLAMA_CHAT_MODEL` | `mistral:7b` | Chat LLM |
| `OLLAMA_EMBED_MODEL` | `nomic-embed-text` | Embedding model |
| `FRONTEND_URL` | `http://localhost:3000` | CORS origin |

---

## Running Tests

```powershell
cd backend
# with .venv active
pytest tests/ -v
```

Current: **95 tests passing** across tools, agent routing, RAG retrieval, and API endpoints.

---

## MLflow Tracking

Every chat interaction creates a tracked MLflow run:

```powershell
# Start the MLflow UI
mlflow ui --port 5001

# Open http://localhost:5001
# Each run logs: intent, route, latency, tool calls, sources, escalation flags, feedback
```

---

## Seed Data (CloudDesk Inc.)

The seed script populates realistic fake company data:

| Table | Rows | Notable |
|---|---|---|
| `users` | 5 | 3 customers (Ayesha, Omar, Sara), 1 agent (Bob), 1 admin (Alice) |
| `orders` | 5 | Order **#1004** for Ayesha — shipped, TRK-CD-1004, $149.99 |
| `subscriptions` | 3 | Ayesha: Pro active; Omar: Starter active; Sara: Pro past_due |
| `payments` | 4 | **TXN-AYE-001 + TXN-AYE-002** — duplicate $79.99 charge demo |
| `knowledge_documents` | 8 | All indexed into ChromaDB (Phase 4) |

Knowledge base docs: `refund_policy`, `subscription_policy`, `shipping_policy`, `billing_policy`, `account_setup`, `troubleshooting`, `pricing`, `terms`.

---

## Roadmap

The project is built in 17 phases. See [`claude-docs/execution-plan.md`](claude-docs/execution-plan.md) for the full spec.

Phases completed: **0–9, 11** | In progress: **10** | Remaining: **12–16**

---

## Notes

- **Ollama must be running** before starting the backend (`ollama serve`)
- **ChromaDB is gitignored** — regenerate with `python -m app.rag.ingest` after cloning
- **Port 5433** is used for Docker Postgres (avoids conflict with any local Postgres on 5432)
- All LLM inference, embeddings, and evaluation run locally — no API keys required
