# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Application

```bash
# Quick start (from project root)
./run.sh

# Manual (must run from backend/ directory)
cd backend && uv run uvicorn app:app --reload --port 8000
```

Requires a `.env` file in the project root:
```
ANTHROPIC_API_KEY=your-key-here
```

App serves at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

## Dependencies

```bash
uv sync
```

Always use `uv` to manage all dependencies and run Python commands — never `pip` directly.

```bash
uv add <package>      # add a dependency
uv remove <package>   # remove a dependency
uv run <command>      # run a command in the project environment
```

No test suite or linter is configured in this project.

## Architecture

This is a RAG (Retrieval-Augmented Generation) chatbot that answers questions about course materials.

**Request flow:** Frontend chat UI → `POST /api/query` → `RAGSystem.query()` → Claude decides whether to call the `search_course_content` tool → `VectorStore.search()` runs semantic similarity search on ChromaDB → results fed back to Claude → final answer returned.

**Key design: tool-based RAG.** Rather than always retrieving context before calling Claude, the system gives Claude a search tool and lets it decide when to use it. `AIGenerator` handles the two-turn tool-use loop (first call may trigger tool use; second call gets the tool results and produces the final answer).

**Two ChromaDB collections:**
- `course_catalog` — course-level metadata (title, instructor, link, lessons list as JSON)
- `course_content` — chunked lesson text, filterable by `course_title` and `lesson_number`

**Embeddings:** `all-MiniLM-L6-v2` via SentenceTransformers, used for both storing and querying.

**Session history** is stored in-memory only (lost on restart). `SessionManager` caps history at `MAX_HISTORY * 2` messages (default: 4).

**The server must be started from the `backend/` directory** because `app.py` uses relative paths (`../docs`, `../frontend`) to locate the docs folder and frontend static files.

## Course Document Format

Files in `docs/` must follow this structure for `DocumentProcessor` to parse them correctly:

```
Course Title: <title>
Course Link: <url>
Course Instructor: <name>

Lesson 1: <lesson title>
Lesson Link: <url>
<lesson content>

Lesson 2: <lesson title>
...
```

Supported file types: `.txt`, `.pdf`, `.docx`. Documents are chunked into ~800-character overlapping segments (configurable in `config.py`).

## Configuration

All tunable parameters are in `backend/config.py`:

| Setting | Default | Description |
|---------|---------|-------------|
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Claude model |
| `EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | Sentence embedding model |
| `CHUNK_SIZE` | `800` | Characters per chunk |
| `CHUNK_OVERLAP` | `100` | Overlap between chunks |
| `MAX_RESULTS` | `5` | Max search results returned |
| `MAX_HISTORY` | `2` | Conversation exchanges to retain |
| `CHROMA_PATH` | `./chroma_db` | ChromaDB persistence location (relative to `backend/`) |

## Extending the System

**Adding a new tool:** Subclass `Tool` in `search_tools.py`, implement `get_tool_definition()` (Anthropic tool schema) and `execute()`, then register with `tool_manager.register_tool(your_tool)` in `RAGSystem.__init__()`.

**Adding a new API endpoint:** Add it to `backend/app.py`. The frontend is mounted last as a catch-all, so API routes must be defined before the static mount.
