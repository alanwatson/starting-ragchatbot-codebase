# RAG Chatbot Query Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            FRONTEND (Browser)                                │
│                          frontend/script.js                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                    1. User types query & clicks send
                    2. Disable input, show loading dots
                                     │
                                     ▼
                    ┌────────────────────────────────┐
                    │  POST /api/query               │
                    │  {                             │
                    │    query: "What is MCP?",      │
                    │    session_id: "session_1"     │
                    │  }                             │
                    └────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FASTAPI ENDPOINT                                     │
│                          backend/app.py                                      │
│                                                                              │
│  @app.post("/api/query")                                                    │
│  async def query_documents(request: QueryRequest):                          │
│      session_id = request.session_id or create_session()                    │
│      answer, sources = rag_system.query(request.query, session_id)          │
│      return {answer, sources, session_id}                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RAG ORCHESTRATOR                                     │
│                        backend/rag_system.py                                 │
│                                                                              │
│  query(query, session_id):                                                  │
│    1. Get conversation history ──────────┐                                  │
│    2. Generate AI response with tools    │                                  │
│    3. Extract sources                    │                                  │
│    4. Update conversation history        │                                  │
└──────────────────────────────────────────┼──────────────────────────────────┘
                                           │                  │
                    ┌──────────────────────┘                  │
                    ▼                                         ▼
    ┌───────────────────────────────┐         ┌──────────────────────────────┐
    │   SESSION MANAGER             │         │     AI GENERATOR             │
    │ backend/session_manager.py    │         │  backend/ai_generator.py     │
    │                               │         │                              │
    │ - Store conversation history  │         │  generate_response():        │
    │ - Max 2 exchanges (4 msgs)    │         │    1. Build system prompt    │
    │ - Format for Claude context   │         │    2. Add conversation hist  │
    └───────────────────────────────┘         │    3. Call Claude API        │
                                              └──────────────────────────────┘
                                                             │
                                                             ▼
                                              ┌──────────────────────────────┐
                                              │   CLAUDE API (1st Call)      │
                                              │   Anthropic Claude Sonnet 4  │
                                              │                              │
                                              │  System: "You are an AI...   │
                                              │           Previous conv..."  │
                                              │  User: "What is MCP?"        │
                                              │  Tools: [search_course_...]  │
                                              │  Temperature: 0              │
                                              └──────────────────────────────┘
                                                             │
                                         ┌───────────────────┴───────────────────┐
                                         │                                       │
                              stop_reason: "tool_use"            stop_reason: "end_turn"
                                         │                                       │
                                         ▼                                       │
                      ┌─────────────────────────────────┐                       │
                      │  Claude wants to search:        │                       │
                      │  {                              │                       │
                      │    tool: "search_course_content"│                       │
                      │    query: "MCP",                │                       │
                      │    course_name: null            │                       │
                      │  }                              │                       │
                      └─────────────────────────────────┘                       │
                                         │                                       │
                                         ▼                                       │
                      ┌─────────────────────────────────┐                       │
                      │      TOOL MANAGER               │                       │
                      │   backend/search_tools.py       │                       │
                      │                                 │                       │
                      │  execute_tool(name, **params)   │                       │
                      │    ↓                            │                       │
                      │  CourseSearchTool.execute()     │                       │
                      └─────────────────────────────────┘                       │
                                         │                                       │
                                         ▼                                       │
                      ┌─────────────────────────────────────────────┐           │
                      │           VECTOR STORE                      │           │
                      │        backend/vector_store.py              │           │
                      │                                             │           │
                      │  search(query, course_name, lesson_number): │           │
                      │                                             │           │
                      │  Step 1: Resolve course name (if provided) │           │
                      │    ↓                                        │           │
                      │  ┌──────────────────────────────────┐      │           │
                      │  │ course_catalog.query()           │      │           │
                      │  │ - Semantic search for course     │      │           │
                      │  │ - "MCP" → "Intro to MCP Servers" │      │           │
                      │  └──────────────────────────────────┘      │           │
                      │    ↓                                        │           │
                      │  Step 2: Build filter                      │           │
                      │    {course_title: "...", lesson_number: N} │           │
                      │    ↓                                        │           │
                      │  Step 3: Search content                    │           │
                      │    ↓                                        │           │
                      │  ┌──────────────────────────────────┐      │           │
                      │  │ CHROMADB                         │      │           │
                      │  │                                  │      │           │
                      │  │ course_content.query():          │      │           │
                      │  │ - Embed query using              │      │           │
                      │  │   SentenceTransformer            │      │           │
                      │  │ - Cosine similarity search       │      │           │
                      │  │ - Apply filters                  │      │           │
                      │  │ - Return top 5 chunks            │      │           │
                      │  └──────────────────────────────────┘      │           │
                      │    ↓                                        │           │
                      │  Returns: SearchResults                    │           │
                      │    - documents: [chunk1, chunk2...]        │           │
                      │    - metadata: [meta1, meta2...]           │           │
                      │    - distances: [0.23, 0.45...]            │           │
                      └─────────────────────────────────────────────┘           │
                                         │                                       │
                                         ▼                                       │
                      ┌─────────────────────────────────┐                       │
                      │  Format Search Results          │                       │
                      │  backend/search_tools.py        │                       │
                      │                                 │                       │
                      │  "[Intro to MCP - Lesson 1]     │                       │
                      │   MCP stands for Model Context  │                       │
                      │   Protocol...                   │                       │
                      │                                 │                       │
                      │   [Intro to MCP - Lesson 2]     │                       │
                      │   MCP servers provide..."       │                       │
                      │                                 │                       │
                      │  Sources: ["Intro to MCP -      │                       │
                      │            Lesson 1", ...]      │                       │
                      └─────────────────────────────────┘                       │
                                         │                                       │
                                         ▼                                       │
                      ┌─────────────────────────────────┐                       │
                      │  Return Tool Results            │                       │
                      │  to AI Generator                │                       │
                      └─────────────────────────────────┘                       │
                                         │                                       │
                                         ▼                                       │
                      ┌──────────────────────────────────────────┐              │
                      │   CLAUDE API (2nd Call)                  │              │
                      │                                          │              │
                      │  Messages:                               │              │
                      │    [User: "What is MCP?"]                │              │
                      │    [Assistant: {tool_use}]               │              │
                      │    [User: {tool_results with content}]   │              │
                      │                                          │              │
                      │  Claude synthesizes answer from results  │              │
                      └──────────────────────────────────────────┘              │
                                         │                                       │
                                         └───────────────┬───────────────────────┘
                                                         │
                                                         ▼
                                         ┌───────────────────────────────┐
                                         │  Final Answer Generated       │
                                         │                               │
                                         │  "MCP (Model Context Protocol)│
                                         │   is a protocol that allows..."│
                                         └───────────────────────────────┘
                                                         │
                                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BACK TO RAG SYSTEM                                   │
│                        backend/rag_system.py                                 │
│                                                                              │
│  1. Get sources from tool_manager.get_last_sources()                        │
│  2. Update conversation history (add_exchange)                              │
│  3. Return (answer, sources)                                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FASTAPI RESPONSE                                     │
│                          backend/app.py                                      │
│                                                                              │
│  return QueryResponse(                                                      │
│      answer="MCP (Model Context Protocol)...",                              │
│      sources=["Intro to MCP - Lesson 1", "Intro to MCP - Lesson 2"],       │
│      session_id="session_1"                                                 │
│  )                                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
                          ┌──────────────────────────────┐
                          │  HTTP 200 OK                 │
                          │  Content-Type: application/  │
                          │               json           │
                          │  {                           │
                          │    "answer": "...",          │
                          │    "sources": [...],         │
                          │    "session_id": "session_1" │
                          │  }                           │
                          └──────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            FRONTEND RENDERING                                │
│                          frontend/script.js                                  │
│                                                                              │
│  1. Remove loading animation                                                │
│  2. Store session_id for next query                                         │
│  3. addMessage(data.answer, 'assistant', data.sources)                      │
│     - Convert markdown to HTML (marked.js)                                  │
│     - Create message bubble                                                 │
│     - Add collapsible sources section                                       │
│     - Append to chat, scroll to bottom                                      │
│  4. Re-enable input                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                         │
                                         ▼
                          ┌──────────────────────────────┐
                          │    USER SEES RESPONSE        │
                          │                              │
                          │  ┌────────────────────────┐  │
                          │  │ MCP (Model Context     │  │
                          │  │ Protocol) is a protocol│  │
                          │  │ that allows...         │  │
                          │  │                        │  │
                          │  │ ▶ Sources              │  │
                          │  └────────────────────────┘  │
                          └──────────────────────────────┘


═══════════════════════════════════════════════════════════════════════════════
                              KEY COMPONENTS
═══════════════════════════════════════════════════════════════════════════════

┌────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   ChromaDB     │  │ Sentence         │  │   Claude API     │
│   Collections  │  │ Transformers     │  │   (Sonnet 4)     │
├────────────────┤  ├──────────────────┤  ├──────────────────┤
│                │  │                  │  │                  │
│ course_catalog │  │ all-MiniLM-L6-v2 │  │ Tool Calling     │
│  - titles      │  │                  │  │ Temperature: 0   │
│  - instructors │  │ Converts text    │  │ Max tokens: 800  │
│  - metadata    │  │ to 384-dim       │  │                  │
│                │  │ embeddings       │  │ System prompt    │
│ course_content │  │                  │  │ + history        │
│  - chunks      │  │ Cosine similarity│  │                  │
│  - 800 chars   │  │ for search       │  │ 2-step process:  │
│  - 100 overlap │  │                  │  │  1. Tool decision│
│                │  │                  │  │  2. Final answer │
└────────────────┘  └──────────────────┘  └──────────────────┘


═══════════════════════════════════════════════════════════════════════════════
                            DATA FLOW SUMMARY
═══════════════════════════════════════════════════════════════════════════════

Query → FastAPI → RAG System → AI Generator → Claude (1st)
                                    ↓
                               Tool Decision
                                    ↓
                              Tool Manager
                                    ↓
                          CourseSearchTool
                                    ↓
                              Vector Store
                                    ↓
                  ChromaDB (semantic search)
                                    ↓
                          Format Results
                                    ↓
                       Claude (2nd) → Final Answer
                                    ↓
               RAG System (collect sources, update history)
                                    ↓
                    FastAPI Response → Frontend
                                    ↓
                          Display to User
```

## Flow Characteristics

**Synchronous Path**: Frontend → Backend → RAG → AI → (Tools if needed) → Response
**Session Persistence**: Session ID maintained across queries for conversation context
**Smart Routing**: Claude decides when to search vs. use general knowledge
**Semantic Search**: Fuzzy matching on course names and content similarity
**Source Tracking**: Sources captured during tool execution and returned to user
**Error Handling**: Try-catch at each layer with graceful degradation
