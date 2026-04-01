# BlogAgent

An AI-powered blog generation system with a human-in-the-loop review workflow. Submit a topic → the agent researches, writes, and illustrates a full blog post → you approve or reject → approved blogs are auto-published to GitHub Pages.

**Live Blog Site:** [https://github.com/VikasPrajapati1998/Blogs](https://github.com/VikasPrajapati1998/Blogs)

---

## How It Works

```
Topic Input → Research (Tavily) → Orchestrator → Parallel Workers
    → Merge Content → Generate Images (Gemini) → Human Review
        → Approve → Publish to GitHub Pages
        → Reject  → Saved with reason
```

1. **Router** — decides if web research is needed, generates search queries
2. **Research** — parallel Tavily searches, deduplication
3. **Orchestrator** — breaks the blog into 5 parallel writing tasks
4. **Workers** — write each section concurrently via Ollama (llama3.2:3b)
5. **Merge** — assembles ~1200–1400 word blog
6. **Image Generation** — Gemini generates 3 contextual images per blog
7. **HITL** — LangGraph pauses; you approve/reject via UI or API
8. **Publish** — auto git commit + push to the Blogs repo

---

## Stack

| Layer | Tech |
|---|---|
| Agent | LangGraph + LangChain |
| LLM | Ollama (`llama3.2:3b`) |
| Research | Tavily Search API |
| Images | Google Gemini (`google-genai`) |
| Backend | FastAPI + asyncpg + SQLAlchemy |
| Database | PostgreSQL (blog store) + SQLite (LangGraph checkpointer) |
| Frontend | React + TypeScript + Vite + Tailwind CSS |
| Publishing | GitHub Pages via git automation |

---

## Project Structure

```
BlogAgent/
├── Backend/
│   ├── agent.py          # LangGraph workflow (router → research → workers → HITL)
│   ├── main.py           # FastAPI app + all endpoints
│   ├── database.py       # PostgreSQL models + async engine
│   ├── prompts.py        # All LLM prompts
│   ├── config.py         # Settings from .env
│   ├── github.py         # Git add/commit/push automation
│   ├── logger.py         # Structured logging
│   ├── current_datetime.py
│   ├── example.dotenv    # Copy this to .env and fill in keys
│   ├── requirements.txt
│   └── Blogs/            # Static blog site (pushed to GitHub Pages)
│       ├── index.html
│       ├── contents/
│       │   ├── html/     # Generated blog HTML files
│       │   ├── css/
│       │   ├── js/
│       │   └── images/   # AI-generated images
└── Frontend/
    ├── src/
    │   ├── components/   # UI components
    │   ├── hooks/        # usePolling, useDownload
    │   ├── api.ts
    │   └── App.tsx
    └── vite.config.ts
```

---

## Prerequisites

- Python 3.11.9
- Node.js 18+
- PostgreSQL (running locally or remote)
- [Ollama](https://ollama.com) with `llama3.2:3b` pulled
- API keys: Tavily, Google Gemini
- GitHub repo configured for the Blogs site

---

## Backend Setup

```bash
cd Backend

# 1. Copy and fill in environment variables
cp example.dotenv .env

# 2. Create virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install fastapi uvicorn asyncpg sqlalchemy langgraph \
            langchain-ollama langchain-tavily google-genai python-dotenv

# 4. Pull the local LLM
ollama pull llama3.2:3b

# 5. Start the server
uvicorn main:app --reload --port 8000
```

Swagger UI → [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)

### Required `.env` keys

```env
# Database
DATABASE_URL=postgresql+asyncpg://user:password@localhost/blogdb

# LLM
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=llama3.2:3b

# APIs
TAVILY_API_KEY=tvly-...
GEMINI_API_KEY=AIza...

# GitHub (for auto-publish)
GITHUB_REPO_PATH=./Blogs
GITHUB_REMOTE=origin
GITHUB_BRANCH=main
```

---

## Frontend Setup

```bash
cd Frontend

npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000)

> Vite proxies all `/blogs` and `/health` requests to `http://localhost:8000` — no CORS setup needed.

### Production Build

```bash
npm run build   # outputs to ../static/dist/
```

Add to `main.py` to serve the React app from FastAPI:

```python
from fastapi.staticfiles import StaticFiles
app.mount("/", StaticFiles(directory="static/dist", html=True), name="static")
```

---

## API Reference

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/blogs/generate` | Start generation for a topic |
| `GET` | `/blogs` | List all blogs |
| `GET` | `/blogs/{thread_id}` | Get single blog |
| `GET` | `/blogs/{thread_id}/status` | Poll workflow state |
| `POST` | `/blogs/approve` | Approve → publish + push to GitHub |
| `POST` | `/blogs/reject` | Reject with reason |
| `PUT` | `/blogs/{thread_id}` | Edit title or content |
| `DELETE` | `/blogs/{thread_id}` | Delete blog |

---

## Published Blogs

Approved blogs are automatically committed and pushed to the Blogs repository and served via GitHub Pages.

→ [https://github.com/VikasPrajapati1998/Blogs](https://github.com/VikasPrajapati1998/Blogs)

---

## License

[MIT](LICENSE)
