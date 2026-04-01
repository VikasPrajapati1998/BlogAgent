# BlogAgent — Backend

> FastAPI + LangGraph backend powering the BlogAgent AI blog generation system.  
> Handles multi-agent orchestration, human-in-the-loop approval, AI image generation, PostgreSQL persistence, and automated GitHub publishing.

---

## Table of Contents

- [Overview](#overview)
- [Agent Workflow](#agent-workflow)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Environment Variables](#environment-variables)
- [Installation](#installation)
- [Database Setup](#database-setup)
- [Running the Server](#running-the-server)
- [API Reference](#api-reference)
- [GitHub Auto-Publishing](#github-auto-publishing)
- [Logging](#logging)
- [Configuration Reference](#configuration-reference)

---

## Overview

The backend is a **FastAPI** application that exposes a REST API for blog management and drives a **LangGraph** multi-agent workflow. When a topic is submitted, the workflow autonomously:

1. Classifies the topic as `open_book` (web-researched) or `closed_book` (LLM knowledge)
2. Fires parallel **Tavily** web searches to gather evidence
3. Plans a structured blog outline via an orchestrator agent
4. Dispatches parallel worker agents — one per section — to write content concurrently
5. Generates contextual images with **Google Gemini** and embeds them into the blog HTML
6. Pauses at a **Human-in-the-Loop (HITL)** checkpoint for review
7. On approval — publishes the blog files and pushes them to GitHub automatically
8. On rejection — records the reason in PostgreSQL and terminates cleanly

---

## Agent Workflow

```
[POST /blogs/generate]
        │
        ▼
  ┌─────────────┐
  │   Router    │  ← Classifies topic, generates search queries
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │  Research   │  ← Parallel Tavily searches (up to 5 queries)
  └──────┬──────┘
         │
         ▼
  ┌──────────────────┐
  │  Orchestrator    │  ← Plans blog: sections + word targets
  └──────┬───────────┘
         │
         ▼
  ┌──────────────────┐
  │   Dispatcher     │  ← Fans out tasks to parallel workers
  └──────┬───────────┘
         │
    ┌────┴────┐
    │ Workers │  ← Each worker writes one section (concurrent)
    └────┬────┘
         │
         ▼
  ┌──────────────────┐
  │  Merge Content   │  ← Combines sections into full draft
  └──────┬───────────┘
         │
         ▼
  ┌──────────────────┐
  │  Decide Images   │  ← Selects sections that need images
  └──────┬───────────┘
         │
         ▼
  ┌────────────────────────┐
  │ Generate & Place Images│  ← Google Gemini → PNG → embed in HTML
  └──────┬─────────────────┘
         │
         ▼
  ┌──────────────────┐
  │ HITL Checkpoint  │  ← Workflow pauses; status → PENDING
  └──────┬───────────┘
         │
    ┌────┴────────────────┐
    │                     │
  Approved             Rejected
    │                     │
    ▼                     ▼
Publish + Git push   Save reason → END
```

LangGraph checkpoints every state transition to `blog_hitl.db` (SQLite), making the workflow fully resumable after the human approval step.

---

## Tech Stack

| Component | Library / Tool |
|---|---|
| Web framework | [FastAPI](https://fastapi.tiangolo.com/) |
| ASGI server | [Uvicorn](https://www.uvicorn.org/) |
| Agent workflow | [LangGraph](https://langchain-ai.github.io/langgraph/) |
| Local LLM | [LangChain Ollama](https://python.langchain.com/docs/integrations/llms/ollama/) (`llama3.2:3b`) |
| Web research | [LangChain Tavily](https://python.langchain.com/docs/integrations/tools/tavily_search/) |
| Image generation | [Google Generative AI — Gemini](https://ai.google.dev/) |
| Database ORM | [SQLAlchemy](https://docs.sqlalchemy.org/) + [asyncpg](https://magicstack.github.io/asyncpg/) |
| HITL checkpointer | SQLite via LangGraph `SqliteSaver` |
| Git automation | Python `subprocess` via `github.py` |
| Config management | `python-dotenv` + Pydantic `BaseSettings` |

---

## Project Structure

```
Backend/
├── agent.py              # Full LangGraph workflow definition (all nodes + edges)
├── main.py               # FastAPI application — routes, lifespan, static mount
├── database.py           # SQLAlchemy async engine, models, CRUD helpers
├── config.py             # Pydantic BaseSettings — loads from .env
├── prompts.py            # All LLM prompt templates (router, orchestrator, workers, etc.)
├── logger.py             # Structured logger — console + rotating file handler
├── github.py             # Git add / commit / push automation
├── current_datetime.py   # IST-aware datetime utility used inside prompts
├── requirements.txt      # Pinned Python dependencies
├── example.dotenv        # Template for the .env file
├── blog_hitl.db          # LangGraph SQLite checkpointer (auto-created)
└── Blogs/                # Static output — also a standalone Git repo
    ├── index.html        # Blog listing page (auto-updated on approval)
    ├── style.css
    ├── script.js
    └── contents/
        ├── html/         # Per-blog HTML file
        ├── css/          # Per-blog stylesheet
        ├── js/           # Per-blog JavaScript
        └── images/       # AI-generated PNG images per blog
```

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Python | **3.11.9** | Exact version recommended |
| [Ollama](https://ollama.com/) | Latest | Local LLM inference engine |
| PostgreSQL | 14+ | Primary metadata store |
| Git | Any | Required for GitHub auto-push |

**Install and pull the Ollama model before starting the server:**

```bash
# Install Ollama from https://ollama.com, then:
ollama pull llama3.2:3b

# Verify
ollama list
```

---

## Environment Variables

Copy the template and populate all values:

```bash
cp example.dotenv .env
```

```env
# ── PostgreSQL ──────────────────────────────────────────────────────────────
DATABASE_URL=postgresql+asyncpg://username:password@localhost:5432/blogagent

# ── Ollama (Local LLM) ──────────────────────────────────────────────────────
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=llama3.2:3b

# ── Tavily (Web Research) ───────────────────────────────────────────────────
TAVILY_API_KEY=tvly-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# ── Google Gemini (Image Generation) ───────────────────────────────────────
GOOGLE_API_KEY=AIzaSy-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# ── GitHub Auto-Push ────────────────────────────────────────────────────────
GITHUB_REPO_PATH=./Blogs
GITHUB_REMOTE=origin
GITHUB_BRANCH=main
```

**Where to get API keys:**

| Key | Source |
|---|---|
| `TAVILY_API_KEY` | [app.tavily.com](https://app.tavily.com) — free tier available |
| `GOOGLE_API_KEY` | [aistudio.google.com](https://aistudio.google.com) — free tier available |

---

## Installation

```bash
# 1. Navigate to the backend directory
cd Backend

# 2. Create a virtual environment with Python 3.11.9
python -m venv venv

# 3. Activate the virtual environment
# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

# 4. Install dependencies
pip install fastapi uvicorn asyncpg sqlalchemy langgraph langchain-ollama langchain-tavily google-genai python-dotenv

# Or install from the pinned requirements file (recommended)
pip install -r requirements.txt
```

---

## Database Setup

The backend uses **PostgreSQL** for blog metadata and **SQLite** for LangGraph workflow state.

**Step 1 — Create the PostgreSQL database:**

```sql
CREATE DATABASE blogagent;
```

**Step 2 — The application handles the rest.**

On startup, SQLAlchemy auto-creates all required tables in PostgreSQL. The LangGraph checkpointer creates `blog_hitl.db` in the working directory automatically — no manual migration needed.

---

## Running the Server

**Start Ollama first (in a separate terminal):**
```bash
ollama serve
```

**Start the FastAPI server:**
```bash
# Development — with auto-reload
uvicorn main:app --reload --port 8000

# Production
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 1
```

> Use `--workers 1` in production. The LangGraph checkpointer is not process-safe across multiple workers.

**Expected startup output:**
```
INFO  [STARTUP] Static site mounted: .../Blogs → /site
INFO  [STARTUP] PostgreSQL tables ready.
INFO  [STARTUP] LangGraph workflow compiled. Checkpointer DB: 'blog_hitl.db'
```

**Swagger UI (interactive API docs):**
```
http://127.0.0.1:8000/docs
```

**ReDoc:**
```
http://127.0.0.1:8000/redoc
```

---

## API Reference

### Generate a Blog

```http
POST /blogs/generate
Content-Type: application/json

{
  "topic": "The Future of Quantum Computing"
}
```

Starts the LangGraph workflow in a background thread. Returns a `thread_id` immediately — use it to poll status.

**Response:**
```json
{
  "thread_id": "61229d73-5d01-45cf-aa14-98c7dfea1fc5",
  "status": "RUNNING"
}
```

---

### List All Blogs

```http
GET /blogs
GET /blogs?status=PENDING
```

Optional `status` filter: `RUNNING`, `PENDING`, `PUBLISHED`, `REJECTED`.

---

### Get a Single Blog

```http
GET /blogs/{thread_id}
```

Returns full blog metadata including title, content, status, and timestamps.

---

### Poll Workflow Status

```http
GET /blogs/{thread_id}/status
```

Returns the current LangGraph state. Frontend polls this every 4 seconds while `status = RUNNING`.

---

### Approve a Blog

```http
POST /blogs/approve
Content-Type: application/json

{
  "thread_id": "61229d73-5d01-45cf-aa14-98c7dfea1fc5"
}
```

Resumes the paused workflow past the HITL checkpoint. Triggers blog file generation and GitHub push. Sets status to `PUBLISHED`.

---

### Reject a Blog

```http
POST /blogs/reject
Content-Type: application/json

{
  "thread_id": "61229d73-5d01-45cf-aa14-98c7dfea1fc5",
  "reason": "Content is factually inaccurate."
}
```

Terminates the workflow. Records the rejection reason in PostgreSQL. Sets status to `REJECTED`.

---

### Edit a Blog

```http
PUT /blogs/{thread_id}
Content-Type: application/json

{
  "title": "Updated Title",
  "content": "Updated markdown content..."
}
```

Both fields are optional — send only the fields you want to update.

---

### Delete a Blog

```http
DELETE /blogs/{thread_id}
```

Permanently removes the blog record from PostgreSQL. Does not delete already-pushed GitHub files.

---

### Health Check

```http
GET /health
```

Returns `200 OK` if the server is running. Used by the frontend on startup.

---

## GitHub Auto-Publishing

When a blog is approved, `github.py` runs the following Git commands inside the `Blogs/` directory:

```bash
git add .
git commit -m "Blog DD-MM-YYYY-HH-MM-SS"
git push origin main
```

The `Blogs/` folder is a standalone Git repository that must be connected to your GitHub remote once before auto-push works.

**One-time remote setup:**

```bash
cd Backend/Blogs
git init
git remote add origin https://github.com/<your-username>/<your-repo>.git
git branch -M main
git push -u origin main
```

After this, every approval automatically commits and pushes the new blog's HTML, CSS, JS, and images to the remote, triggering GitHub Pages deployment.

**Commit log example:**
```
[main 34fcb3d] Blog 01-04-2026-10-10-51
 9 files changed, 656 insertions(+)
 create mode 100644 contents/css/The_Value_of_a_Nobel_Prize.css
 create mode 100644 contents/html/The_Value_of_a_Nobel_Prize.html
 create mode 100644 contents/images/The_Value_of_a_Nobel_Prize/...
```

---

## Logging

Logs are written to both **stdout** and a timestamped file under `logs/`.

**Log file naming:**
```
logs/backend-log_YYYYMMDD_HHMMSS.log
```

**Log format:**
```
2026-04-01 10:07:29 | INFO    | BlogApp.Agent   | [WORKER] Task 2 done — ~286w (dev=4.7%, 64.59s)
2026-04-01 10:10:25 | WARNING | BlogApp.Agent   | [IMGPLACE] Image 1 attempt 1/3 FAILED: No image content returned
2026-04-01 10:10:51 | INFO    | BlogApp.GitHub  | Git automation completed successfully
```

**Log levels:**

| Level | Emitted by |
|---|---|
| `INFO` | Workflow node entry/exit, API request handling, GitHub push events, status transitions |
| `DEBUG` | Individual Tavily search queries, Git subprocess commands, image file save paths |
| `WARNING` | Worker word count deviation > 20%, image generation retry attempts |
| `ERROR` | Unhandled exceptions, external API failures |

---

## Configuration Reference

All settings live in `config.py` and are loaded from `.env` via Pydantic `BaseSettings`.

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | — | Async PostgreSQL connection string (required) |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama server endpoint |
| `OLLAMA_MODEL` | `llama3.2:3b` | Model used for all LLM calls |
| `TAVILY_API_KEY` | — | Tavily search API key (required) |
| `GOOGLE_API_KEY` | — | Google Gemini API key for image generation (required) |
| `GITHUB_REPO_PATH` | `./Blogs` | Path to the Blogs git repository |
| `GITHUB_REMOTE` | `origin` | Git remote name |
| `GITHUB_BRANCH` | `main` | Branch to push to |
| `MAX_RESEARCH_QUERIES` | `5` | Max parallel Tavily queries per topic |
| `MAX_ORCHESTRATOR_RETRIES` | `3` | Max plan retries before workflow fails |
| `IMAGE_RETRY_COUNT` | `3` | Max Gemini image generation retries per image |
| `LANGGRAPH_CHECKPOINTER_DB` | `blog_hitl.db` | SQLite file path for LangGraph state |


## Quick Tech
- Use `python 3.11.9`
- Create `venv`
- Install: `pip install fastapi uvicorn asyncpg sqlalchemy langgraph langchain-ollama langchain-tavily google-genai python-dotenv`
- Run: `uvicorn main:app --reload` or `uvicorn main:app --reload --port 8000`
- Swagger: `http://127.0.0.1:8000/docs/` 


