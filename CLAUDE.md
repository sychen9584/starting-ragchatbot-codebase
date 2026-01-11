# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Always use `uv` for all dependency management and running the server. Do not use `pip` directly.**

```bash
# Install dependencies
uv sync

# Run the application
./run.sh
# Or manually:
cd backend && uv run uvicorn app:app --reload --port 8000

# Access points
# Web UI: http://localhost:8000
# API Docs: http://localhost:8000/docs
```

## Environment Setup

Requires a `.env` file in the root directory:
```
ANTHROPIC_API_KEY=your-key-here
```

## Architecture

This is a full-stack RAG (Retrieval-Augmented Generation) system for querying course materials.

### Query Flow

```
Frontend (script.js)
    │ POST /api/query {query, session_id}
    ▼
FastAPI (app.py)
    │
    ▼
RAGSystem (rag_system.py) ─── orchestrates all components
    │
    ├── SessionManager: conversation history (max 2 turns)
    │
    ├── AIGenerator (ai_generator.py)
    │   └── Claude API with tool calling
    │       └── If tool_use → execute tool → second API call with results
    │
    └── ToolManager (search_tools.py)
        └── CourseSearchTool
            └── VectorStore.search()
                └── ChromaDB semantic search
```

### Key Components

| File | Purpose |
|------|---------|
| `backend/rag_system.py` | Main orchestrator - coordinates document processing, search, and AI generation |
| `backend/ai_generator.py` | Claude API integration with tool calling loop |
| `backend/vector_store.py` | ChromaDB wrapper with two collections: `course_catalog` (metadata) and `course_content` (chunks) |
| `backend/document_processor.py` | Parses course files into chunks (800 chars, 100 overlap) |
| `backend/search_tools.py` | Tool definitions for Claude - `CourseSearchTool` with query/course_name/lesson_number params |
| `backend/config.py` | Central configuration (model names, chunk sizes, paths) |

### Data Models (models.py)

- `Course`: title, course_link, instructor, lessons[]
- `Lesson`: lesson_number, title, lesson_link
- `CourseChunk`: content, course_title, lesson_number, chunk_index

### Document Format

Course documents in `docs/` follow this structure:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[content...]

Lesson 1: [title]
...
```

### ChromaDB Collections

- `course_catalog`: Course metadata for semantic course name resolution
- `course_content`: Chunked lesson content with embeddings (all-MiniLM-L6-v2)

### Tool Calling Pattern

Claude receives `search_course_content` tool. When invoked:
1. First API call returns `stop_reason: "tool_use"`
2. Backend executes `CourseSearchTool.execute()`
3. Second API call with tool results (no tools) generates final answer
