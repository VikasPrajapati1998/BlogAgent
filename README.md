# BlogAgent 🖊️

> An end-to-end, AI-powered blog generation system with a **Human-in-the-Loop (HITL)** approval workflow, autonomous research, parallel content writing, AI image generation, and automatic GitHub publishing.

[![Python](https://img.shields.io/badge/Python-3.11.9-blue?logo=python)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115+-green?logo=fastapi)](https://fastapi.tiangolo.com/)
[![React](https://img.shields.io/badge/React-18+-61DAFB?logo=react)](https://react.dev/)
[![LangGraph](https://img.shields.io/badge/LangGraph-HITL_Workflow-orange)](https://langchain-ai.github.io/langgraph/)
[![License](https://img.shields.io/badge/License-MIT-yellow)](LICENSE)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Agent Workflow](#agent-workflow)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Backend Setup](#backend-setup)
- [Frontend Setup](#frontend-setup)
- [Running the Application](#running-the-application)
- [API Reference](#api-reference)
- [Blog Publishing](#blog-publishing)
- [Configuration](#configuration)
- [Logging](#logging)
- [Features](#features)
- [License](#license)

---

## Overview

**BlogAgent** is a full-stack agentic application that automates the entire lifecycle of a blog post — from topic input to live GitHub Pages publication. It uses a **LangGraph**-powered multi-agent workflow that:

1. Routes topics to either open-book (web research) or closed-book (LLM knowledge) mode
2. Runs parallel Tavily web searches to gather evidence
3. Uses an orchestrator to plan a structured blog outline
4. Dispatches parallel worker agents to write each section concurrently
5. Generates and embeds relevant AI images (Google Gemini)
6. Pauses for **human review and approval** before publishing
7. On approval, commits the final blog to a GitHub repository and deploys it to GitHub Pages

Published blogs are live at: **[https://github.com/VikasPrajapati1998/Blogs](https://github.com/VikasPrajapati1998/Blogs)**

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    React Frontend                        │
│   (Topic Input → Live Status Polling → Review & HITL)   │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTP (Vite proxy → :8000)
┌──────────────────────────▼──────────────────────────────┐
│                  FastAPI Backend (:8000)                  │
│   /blogs/generate  /blogs/approve  /blogs/reject  etc.   │
└──────┬─────────────────────┬────────────────────────────┘
       │                     │
  PostgreSQL DB         LangGraph HITL
  (blog metadata)       Workflow Engine
                             │
            ┌────────────────┼────────────────┐
            │                │                │
         Router          Research          Workers
        (LLM-based)     (Tavily x5)      (Parallel)
            │                                 │
        Orchestrator ──── Plan ──────► Merge & Images
                                            │
                                     HITL Checkpoint
                                      (await human)
                                            │
                                    Approved? → GitHub Push
                                    Rejected? → Save reason
```

---

## Agent Workflow

The LangGraph workflow consists of the following sequential and parallel nodes:

| Node | Description |
|---|---|
| **Router** | Decides `open_book` vs `closed_book` mode and generates Tavily search queries |
| **Research** | Runs up to 5 parallel Tavily searches; deduplicates and filters evidence |
| **Orchestrator** | Plans a structured blog with titled sections and per-section word targets |
| **Dispatcher** | Fans out writing tasks to parallel worker agents |
| **Workers** | Each worker independently writes one blog section using the LLM |
| **Merge Content** | Combines all sections into a single coherent draft |
| **Decide Images** | Determines which sections need AI-generated images and where to place them |
| **Generate & Place Images** | Calls Google Gemini to generate images; embeds them into the HTML blog |
| **HITL Checkpoint** | Pauses the workflow; blog status set to `PENDING` in PostgreSQL |
| **Finalize Approved** | Publishes the blog to `Blogs/index.html`; pushes to GitHub |
| **Finalize Rejected** | Saves the rejection reason to the database; workflow ends |

---

## Tech Stack

**Backend**
- [Python 3.11.9](https://www.python.org/)
- [FastAPI](https://fastapi.tiangolo.com/) — REST API and lifecycle management
- [LangGraph](https://langchain-ai.github.io/langgraph/) — stateful multi-agent HITL workflow
- [LangChain Ollama](https://python.langchain.com/docs/integrations/llms/ollama/) — local LLM inference (`llama3.2:3b`)
- [LangChain Tavily](https://python.langchain.com/docs/integrations/tools/tavily_search/) — web research
- [Google Generative AI (Gemini)](https://ai.google.dev/) — AI image generation
- [asyncpg + SQLAlchemy](https://docs.sqlalchemy.org/) — async PostgreSQL ORM
- [SQLite](https://www.sqlite.org/) — LangGraph checkpointer (`blog_hitl.db`)
- [Uvicorn](https://www.uvicorn.org/) — ASGI server

**Frontend**
- [React 18](https://react.dev/) + [TypeScript](https://www.typescriptlang.org/)
- [Vite](https://vitejs.dev/) — dev server and bundler
- [Tailwind CSS](https://tailwindcss.com/) — utility-first styling

**Infrastructure**
- [GitHub Pages](https://pages.github.com/) — static blog hosting
- [GitHub Actions](https://github.com/features/actions) — CI/CD for blog site

---

## Project Structure

```
BlogAgent/
├── Backend/
│   ├── agent.py              # LangGraph multi-agent workflow (HITL)
│   ├── main.py               # FastAPI app — routes, startup, shutdown
│   ├── database.py           # SQLAlchemy models + async PostgreSQL helpers
│   ├── config.py             # Pydantic settings (loaded from .env)
│   ├── prompts.py            # All LLM prompt templates
│   ├── logger.py             # Structured logging setup
│   ├── github.py             # Git automation (add → commit → push)
│   ├── current_datetime.py   # IST datetime utility
│   ├── requirements.txt      # Python dependencies
│   ├── example.dotenv        # Example environment variable file
│   └── Blogs/                # Static blog output (served + pushed to GitHub)
│       ├── index.html        # Blog listing page
│       ├── style.css
│       ├── script.js
│       └── contents/
│           ├── html/         # Per-blog HTML files
│           ├── css/          # Per-blog CSS
│           ├── js/           # Per-blog JS
│           └── images/       # AI-generated blog images
│
└── Frontend/
    ├── src/
    │   ├── components/
    │   │   ├── GenerateForm.tsx      # Topic input form
    │   │   ├── BlogList.tsx          # Filterable, date-grouped blog list
    │   │   ├── BlogCard.tsx          # Single blog card
    │   │   ├── BlogViewer.tsx        # Full blog detail + live polling
    │   │   ├── ApprovalPanel.tsx     # Approve / reject with reason
    │   │   ├── DownloadMenu.tsx      # Export as MD / PDF / DOCX
    │   │   ├── EditModal.tsx         # In-place title and content editor
    │   │   ├── StatsBar.tsx          # Live dashboard counts
    │   │   ├── StatusBadge.tsx       # Coloured status pill
    │   │   └── AnimatedBackground.tsx
    │   ├── hooks/
    │   │   ├── usePolling.ts         # Auto-refresh while workflow runs
    │   │   └── useDownload.ts        # Download logic (MD / PDF / DOCX)
    │   ├── api.ts                    # All fetch calls to FastAPI
    │   ├── types.ts                  # Shared TypeScript types
    │   ├── App.tsx                   # Root layout and navigation
    │   └── main.tsx                  # React entry point
    ├── vite.config.ts
    ├── tailwind.config.js
    └── package.json
```

---

## Prerequisites

Ensure the following are installed and available on your system before proceeding.

| Requirement | Version | Notes |
|---|---|---|
| Python | 3.11.9 | Exact version recommended |
| Node.js | 18+ | Verify with `node -v` |
| npm | 9+ | Bundled with Node.js |
| [Ollama](https://ollama.com/) | Latest | Local LLM runner |
| PostgreSQL | 14+ | Running locally or remote |
| Git | Any | Required for GitHub auto-push |

**Pull the required Ollama model:**
```bash
ollama pull llama3.2:3b
```

---

## Environment Setup

Copy the example environment file and fill in your values:

```bash
cd Backend
cp example.dotenv .env
```

Edit `.env` with your credentials:

```env
# ── PostgreSQL ─────────────────────────────────────────
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/blogagent

# ── LLM (Ollama) ───────────────────────────────────────
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=llama3.2:3b

# ── Web Research ────────────────────────────────────────
TAVILY_API_KEY=tvly-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# ── Image Generation ────────────────────────────────────
GOOGLE_API_KEY=AIzaSy-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# ── GitHub Auto-Push ────────────────────────────────────
GITHUB_REPO_PATH=./Blogs          # Path to the Blogs git repo
GITHUB_REMOTE=origin
GITHUB_BRANCH=main
```

> **API Keys Required:**
> - **Tavily** — [Get a free key at app.tavily.com](https://app.tavily.com)
> - **Google Gemini** — [Get a key at aistudio.google.com](https://aistudio.google.com)

---

## Backend Setup

```bash
# 1. Navigate to the backend directory
cd Backend

# 2. Create a virtual environment
python -m venv venv

# 3. Activate the virtual environment
# On Windows:
venv\Scripts\activate
# On macOS / Linux:
source venv/bin/activate

# 4. Install dependencies
pip install fastapi uvicorn asyncpg sqlalchemy langgraph langchain-ollama langchain-tavily google-genai python-dotenv

# Or from the requirements file:
pip install -r requirements.txt
```

**PostgreSQL — create the database:**
```sql
CREATE DATABASE blogagent;
```
The application creates all required tables automatically on first startup via SQLAlchemy.

---

## Frontend Setup

```bash
# Navigate to the frontend directory
cd Frontend

# Install all dependencies
npm install
```

---

## Running the Application

### Step 1 — Start Ollama

```bash
ollama serve
```

Verify the model is ready:
```bash
ollama list
# Should show: llama3.2:3b
```

### Step 2 — Start the FastAPI Backend

```bash
cd Backend
uvicorn main:app --reload --port 8000
```

Expected startup output:
```
INFO  [STARTUP] Static site mounted: .../Blogs → /site
INFO  [STARTUP] PostgreSQL tables ready.
INFO  [STARTUP] LangGraph workflow compiled. Checkpointer DB: 'blog_hitl.db'
```

Interactive API docs: **http://127.0.0.1:8000/docs**

### Step 3 — Start the React Frontend

```bash
cd Frontend
npm run dev
```

Open **http://localhost:3000** in your browser.

> The Vite dev server automatically proxies all `/blogs` and `/health` requests to `http://localhost:8000` — no CORS configuration needed.

---

## API Reference

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/blogs/generate` | Start a new blog generation workflow |
| `GET` | `/blogs` | List all blogs (with optional status filter) |
| `GET` | `/blogs/{thread_id}` | Get a single blog by thread ID |
| `GET` | `/blogs/{thread_id}/status` | Poll the current workflow state |
| `POST` | `/blogs/approve` | Approve a pending blog → triggers GitHub publish |
| `POST` | `/blogs/reject` | Reject a pending blog with a reason |
| `PUT` | `/blogs/{thread_id}` | Edit blog title or content |
| `DELETE` | `/blogs/{thread_id}` | Permanently delete a blog |
| `GET` | `/health` | Health check endpoint |

**Example — generate a blog:**
```bash
curl -X POST http://localhost:8000/blogs/generate \
  -H "Content-Type: application/json" \
  -d '{"topic": "The Future of Quantum Computing"}'
```

**Example — approve a blog:**
```bash
curl -X POST http://localhost:8000/blogs/approve \
  -H "Content-Type: application/json" \
  -d '{"thread_id": "61229d73-5d01-45cf-aa14-98c7dfea1fc5"}'
```

---

## Blog Publishing

When a blog is approved, the workflow automatically:

1. Writes the blog HTML, CSS, and JS into `Backend/Blogs/contents/`
2. Updates `Backend/Blogs/index.html` with the new entry
3. Runs `git add . → git commit → git push origin main` inside the `Blogs/` directory

The `Blogs/` folder is a standalone Git repository connected to:
**[https://github.com/VikasPrajapati1998/Blogs](https://github.com/VikasPrajapati1998/Blogs)**

Published blogs are served via GitHub Pages and accessible publicly.

**To initialize the Blogs git remote (one-time setup):**
```bash
cd Backend/Blogs
git init
git remote add origin https://github.com/VikasPrajapati1998/Blogs.git
git branch -M main
git push -u origin main
```

---

## Configuration

All configuration is managed in `Backend/config.py` via Pydantic `BaseSettings`. Values are read from `.env` automatically. Key settings:

| Setting | Default | Description |
|---|---|---|
| `OLLAMA_MODEL` | `llama3.2:3b` | Ollama model used for all LLM calls |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama server address |
| `MAX_RESEARCH_QUERIES` | `5` | Number of parallel Tavily searches |
| `MAX_ORCHESTRATOR_RETRIES` | `3` | Blog plan retries before failure |
| `IMAGE_RETRY_COUNT` | `3` | Gemini image generation retry attempts |
| `LANGGRAPH_CHECKPOINTER_DB` | `blog_hitl.db` | SQLite file for LangGraph state |

---

## Logging

The backend uses a structured logger (`Backend/logger.py`) that writes to both the console and a timestamped log file under `logs/`.

Log format:
```
2026-04-01 10:07:29 | INFO | BlogApp.Agent | [WORKER] Task 2 done — ~286w (dev=4.7%, 64.59s)
```

Log levels used across the workflow:

| Level | Usage |
|---|---|
| `INFO` | Workflow node entry/exit, API calls, GitHub push events |
| `DEBUG` | Individual Tavily queries, Git commands, image save paths |
| `WARNING` | Word count deviations > 20%, image generation retries |
| `ERROR` | Unhandled exceptions, failed external API calls |

---

## Features

| Feature | Description |
|---|---|
| **Agentic Research** | Parallel Tavily web searches with LLM-based deduplication |
| **Multi-mode Routing** | `open_book` (web-grounded) vs `closed_book` (LLM knowledge) |
| **Parallel Writing** | Up to 5 worker agents write sections concurrently |
| **AI Image Generation** | Google Gemini generates contextual images per section |
| **HITL Workflow** | LangGraph interrupts the workflow at approval checkpoint |
| **Live Polling** | Frontend polls every 4s while the workflow is running |
| **Approve / Reject** | Review full blog before it goes live; reject with reason |
| **Edit & Rename** | Rewrite any section or change the title before approving |
| **GitHub Auto-Push** | Approved blogs commit and push to GitHub automatically |
| **Download** | Export any blog as Markdown, PDF, or DOCX |
| **Filter & Group** | Filter by status; blogs grouped by creation date |
| **Stats Dashboard** | Live counts: total / pending / published / rejected |

---

## Production Build

To build the frontend and serve it through FastAPI:

```bash
# Build the React app
cd Frontend
npm run build
# Output goes to ../static/dist/
```

Add this to `Backend/main.py` **after all API routes**:
```python
from fastapi.staticfiles import StaticFiles

app.mount("/", StaticFiles(directory="static/dist", html=True), name="static")
```

Now `http://localhost:8000` serves both the API and the React frontend from a single process.

---

## License

This project is licensed under the **MIT License**. See [LICENSE](LICENSE) for details.

---

<p align="center">Built with LangGraph · FastAPI · React · Ollama · Tavily · Google Gemini</p>
