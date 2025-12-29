# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Retrieval-Augmented Generation (RAG) chatbot system that enables semantic search over course materials. It combines FastAPI (backend), vanilla JavaScript (frontend), ChromaDB (vector store), and Anthropic's Claude API (AI generation with tool calling).

## Development Commands

### Running the Application

```bash
# Quick start
./run.sh

# Manual start
cd backend && uv run uvicorn app:app --reload --port 8000
```

Access points:
- Web UI: http://localhost:8000
- API docs: http://localhost:8000/docs

### Environment Setup

```bash
# Install dependencies
uv sync

# Required .env file in root:
ANTHROPIC_API_KEY=your_key_here
```

### Working with Dependencies

This project uses `uv` as the package manager. Dependencies are defined in `pyproject.toml`:

```bash
# Add a new dependency
uv add package-name

# Update dependencies
uv sync

# Run Python scripts
uv run python script.py
```

## Architecture Overview

### Request Flow Architecture

The system processes queries through a multi-stage pipeline:

1. **Frontend** (frontend/script.js) → POST /api/query with `{query, session_id}`
2. **FastAPI** (backend/app.py) → Routes to RAG system
3. **RAG Orchestrator** (backend/rag_system.py) → Coordinates all components
4. **AI Generator** (backend/ai_generator.py) → Makes 2 Claude API calls:
   - First call: Claude decides whether to use search tool
   - If tool_use: ToolManager executes search
   - Second call: Claude generates final answer with search results
5. **Tool Execution** (backend/search_tools.py) → CourseSearchTool performs semantic search
6. **Vector Store** (backend/vector_store.py) → ChromaDB semantic search with:
   - Course name resolution (fuzzy matching via embeddings)
   - Content search with filters
7. **Response Assembly** → RAG system collects sources, updates conversation history
8. **Frontend Rendering** → Displays markdown-formatted answer with collapsible sources

### Core Components Interaction

**RAGSystem** (`rag_system.py`) is the central orchestrator that coordinates:
- `DocumentProcessor`: Parses course docs into structured Course/Lesson objects and chunks text
- `VectorStore`: Manages two ChromaDB collections (catalog + content)
- `AIGenerator`: Handles Claude API with tool calling protocol
- `SessionManager`: Maintains conversation history (max 2 exchanges by default)
- `ToolManager` + `CourseSearchTool`: Implements search functionality as Claude tool

**Key architectural decisions:**
- Tool-based search (not retrieval-then-generate): Claude decides when to search
- Dual ChromaDB collections: `course_catalog` for metadata/course names, `course_content` for text chunks
- Semantic course name matching: "MCP" can match "Introduction to MCP Servers"
- Session-based conversation memory with configurable history depth

### Vector Store Design

The `VectorStore` uses two ChromaDB collections with different purposes:

1. **course_catalog**: Stores course metadata for semantic course name matching
   - Documents: Course titles
   - Metadata: instructor, course_link, lessons_json, lesson_count
   - IDs: Course title (serves as unique identifier)

2. **course_content**: Stores chunked course text for content search
   - Documents: Text chunks (800 chars, 100 char overlap)
   - Metadata: course_title, lesson_number, chunk_index
   - IDs: `{course_title}_{chunk_index}`

Search flow: Query → Resolve course name (if provided) → Build filter → Search content → Return top 5 results

### Configuration System

All configuration is centralized in `backend/config.py`:
- `ANTHROPIC_MODEL`: claude-sonnet-4-20250514
- `EMBEDDING_MODEL`: all-MiniLM-L6-v2 (384-dim embeddings)
- `CHUNK_SIZE`: 800 characters
- `CHUNK_OVERLAP`: 100 characters
- `MAX_RESULTS`: 5 search results
- `MAX_HISTORY`: 2 conversation exchanges
- `CHROMA_PATH`: ./chroma_db

These values affect system behavior. For example, changing `MAX_HISTORY` impacts context window size for Claude.

## Course Document Format

Course documents must follow this structure (see `docs/` for examples):

```
Course Title: [title]
Course Link: [url]
Course Instructor: [name]

Lesson 0: [title]
Lesson Link: [url]
[lesson content...]

Lesson 1: [title]
Lesson Link: [url]
[lesson content...]
```

The `DocumentProcessor` parses this format to extract:
- Course metadata (title, instructor, link)
- Lesson structure (number, title, link)
- Content chunks with lesson attribution

## Important Implementation Details

### Tool Calling Protocol

The AI Generator implements a two-step Claude API interaction:

1. **First call**: Claude receives user query + tool definitions, decides whether to use `search_course_content`
2. **If tool_use**: Execute tool, collect results
3. **Second call**: Send tool results back to Claude, get synthesized answer

This is implemented in `ai_generator.py:_handle_tool_execution()`. The tool results are formatted as structured messages following Anthropic's tool calling protocol.

### Session Management

Sessions are created on first query and persist across requests. The `SessionManager`:
- Stores conversation as list of Message objects (role + content)
- Maintains last N exchanges (configurable via MAX_HISTORY)
- Formats history as string for Claude's system prompt
- Each session has unique ID: `session_{counter}`

### Frontend-Backend Contract

The `/api/query` endpoint expects:
```json
{
  "query": "string",
  "session_id": "string | null"
}
```

Returns:
```json
{
  "answer": "string (markdown formatted)",
  "sources": ["Course - Lesson N", ...],
  "session_id": "string"
}
```

Sources are tracked in `CourseSearchTool.last_sources` during search execution and retrieved by RAG system after AI generation.

### Startup Behavior

On application startup (`app.py:startup_event`):
1. Checks for `../docs` folder
2. Calls `rag_system.add_course_folder()`
3. Processes each .txt/.pdf/.docx file
4. Skips courses already in ChromaDB (by title)
5. Adds new courses to both collections

This means ChromaDB persists between restarts. To rebuild index, delete `./chroma_db` directory.

## Common Patterns

### Adding a New Tool

1. Create tool class inheriting from `Tool` in `search_tools.py`
2. Implement `get_tool_definition()` returning Anthropic tool schema
3. Implement `execute(**kwargs)` with tool logic
4. Register in RAG system: `self.tool_manager.register_tool(YourTool(dependencies))`

### Modifying Search Behavior

Search logic is in `vector_store.py:search()`. The method:
1. Resolves course name to exact title (via semantic search on catalog)
2. Builds ChromaDB filter dict (supports course_title, lesson_number, or both)
3. Performs semantic search on content collection
4. Returns SearchResults dataclass

To add new filter parameters, extend `_build_filter()` and update tool definition.

### Changing AI Behavior

AI instructions are in `ai_generator.py:SYSTEM_PROMPT`. Key directives:
- Use search tool only for course-specific questions
- Maximum one search per query
- Provide brief, concise, educational responses
- No meta-commentary about search process

Modify this prompt to change AI behavior without code changes.
