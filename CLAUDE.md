# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A full-stack RAG (Retrieval-Augmented Generation) chatbot that answers questions about DeepLearning.AI course materials. Users query via a chat UI, the backend performs semantic search over course documents using ChromaDB, and Claude generates answers grounded in the retrieved content.

## Commands

```bash
# Install dependencies
uv sync

# Run the app (starts on http://localhost:8000)
./run.sh
# Or manually:
cd backend && uv run uvicorn app:app --reload --port 8000

# API docs available at http://localhost:8000/docs
```

There are no tests, linting, or formatting tools configured.

## Architecture

**Request flow:** Frontend → FastAPI (`app.py`) → `rag_system.py` → Claude API with tool use → `search_tools.py` → `vector_store.py` (ChromaDB) → results back to Claude → grounded response → Frontend

The key pattern is a **two-call tool-use loop**: Claude receives the query and available tools, decides to call `search_course_content` with search parameters, gets results back, then generates the final answer.

**Backend (`backend/`):**
- `app.py` — FastAPI app, serves frontend static files and two API endpoints (`POST /api/query`, `GET /api/courses`). Loads documents from `../docs` on startup.
- `rag_system.py` — Orchestrator that wires together all components. Entry point is `query(query, session_id)`.
- `ai_generator.py` — Claude API client. Handles the tool-use loop: send message → receive tool_use → execute tool → send results back → get final response. Model: `claude-sonnet-4-20250514`, temperature 0.
- `vector_store.py` — ChromaDB wrapper with two collections: `course_catalog` (metadata) and `course_content` (chunks). Supports filtering by course name and lesson number.
- `document_processor.py` — Parses course `.txt` files with a specific format (headers: `Course Title:`, `Course Instructor:`, `Lesson N:`). Chunks text at sentence boundaries (800 chars, 100 overlap).
- `search_tools.py` — Abstract `Tool` base class + `ToolManager`. `CourseSearchTool` wraps vector store search as a Claude tool definition.
- `session_manager.py` — In-memory conversation history per session (max 2 exchanges).
- `config.py` — All configuration constants. Key: `ANTHROPIC_API_KEY` loaded from `.env`.
- `models.py` — Pydantic models: `Course`, `Lesson`, `CourseChunk`.

**Frontend (`frontend/`):** Vanilla HTML/CSS/JS. Dark theme chat UI with sidebar showing courses and example questions. Renders assistant responses as markdown (marked.js). No build step.

**Documents (`docs/`):** Plain text course transcripts with structured headers parsed by `document_processor.py`.

## Environment

- Python 3.13+ required, uses `uv` package manager
- Requires `ANTHROPIC_API_KEY` in `.env` (copy from `.env.example`)
- ChromaDB persists to `backend/chroma_db/` (gitignored)
- Embedding model: `all-MiniLM-L6-v2` (downloaded on first run)
