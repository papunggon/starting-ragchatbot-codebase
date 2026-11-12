# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A RAG chatbot that answers questions about course transcripts in `docs/`. FastAPI backend + ChromaDB vector store + Anthropic Claude (tool use), with a vanilla HTML/JS frontend served by the same FastAPI app.

## Commands

Uses `uv` (Python 3.13+). Requires `ANTHROPIC_API_KEY` in `.env` at the repo root (see `.env.example`).

```bash
uv sync                                               # install dependencies
./run.sh                                              # start server on http://localhost:8000
cd backend && uv run uvicorn app:app --reload --port 8000   # manual start
```

- Web UI: `http://localhost:8000`, API docs: `http://localhost:8000/docs`
- The server **must be started from `backend/`**: `app.py` uses relative paths `../docs`, `../frontend`, and `config.CHROMA_PATH = "./chroma_db"` (so the DB lives at `backend/chroma_db/`).
- No test suite, linter, or formatter is configured. `main.py` at the root is an unused stub.

## Architecture

Request flow for `POST /api/query` (`{query, session_id?}` → `{answer, sources, session_id}`):

1. `frontend/script.js` posts the query and stores the `session_id` returned on first response.
2. `backend/app.py` creates a session if needed and calls `RAGSystem.query()`.
3. `backend/rag_system.py` is the orchestrator that wires together all components. It fetches session history and calls `AIGenerator.generate_response()` with the tool definitions from `ToolManager`.
4. `backend/ai_generator.py` calls Claude with `tool_choice: auto`. If `stop_reason == "tool_use"`, it executes the tools via `ToolManager`, then makes **one** follow-up call *without tools* — so Claude gets at most one round of tool use per query. Conversation history is injected into the **system prompt** as text, not as message turns.
5. `backend/search_tools.py`: `CourseSearchTool` (`search_course_content`, params `query`, optional `course_name`, `lesson_number`) calls `VectorStore.search()`. New tools implement the `Tool` ABC and are registered with `ToolManager.register_tool()` in `RAGSystem.__init__`.
6. Sources reach the UI through side-channel state: the tool stores `last_sources` on itself during formatting; `RAGSystem` reads them via `tool_manager.get_last_sources()` after generation and then calls `reset_sources()`. This is shared mutable state on a singleton, not per-request.

### Vector store (`backend/vector_store.py`)

Two ChromaDB collections, both embedded with `all-MiniLM-L6-v2` via sentence-transformers:
- `course_catalog` — one doc per course; **ID is the course title**. Metadata includes instructor, course link, and `lessons_json` (serialized lesson list). Used for fuzzy course-name resolution: `_resolve_course_name()` does a 1-result semantic query so partial names like "MCP" map to the full title.
- `course_content` — text chunks with metadata `course_title`, `lesson_number`, `chunk_index`. `search()` resolves the course name, then builds a `where` filter (`$and` when both course and lesson are given).

### Ingestion

On FastAPI startup, `rag_system.add_course_folder("../docs", clear_existing=False)` processes every `.txt/.pdf/.docx` file (all read as UTF-8 text) and skips courses whose title already exists in the catalog. To force a re-index after changing parsing/chunking, delete `backend/chroma_db/` or call with `clear_existing=True`.

`backend/document_processor.py` expects this file format:
```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 0: <lesson title>
Lesson Link: <url>
<transcript text...>
Lesson 1: ...
```
Text is split by sentence into ~`CHUNK_SIZE` (800) char chunks with `CHUNK_OVERLAP` (100). Chunks get a context prefix (`"Lesson N content: ..."` or `"Course <title> Lesson N content: ..."`) before being stored.

### Config

All tunables live in the `Config` dataclass in `backend/config.py`: Claude model, embedding model, chunk size/overlap, `MAX_RESULTS` (5), `MAX_HISTORY` (2 exchanges kept per session), Chroma path. Sessions are in-memory only (`session_manager.py`) and are lost on restart.
