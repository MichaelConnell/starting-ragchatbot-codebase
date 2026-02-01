# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Running the Application
```bash
./run.sh                                    # Start server (recommended)
cd backend && uv run uvicorn app:app --reload --port 8000  # Manual start
```

Access points:
- Web Interface: http://localhost:8000
- API Docs: http://localhost:8000/docs

### Dependencies
```bash
uv sync                                     # Install/update dependencies
```

### Environment Setup
Create `.env` in root with:
```
ANTHROPIC_API_KEY=your-key-here
```

### Adding Course Documents
Place `.txt`, `.pdf`, or `.docx` files in `docs/` folder. The app loads them automatically on startup.

Expected document format:
```
Course Title: [title]
Course Link: [url]
Course Instructor: [instructor]

Lesson 0: [lesson title]
Lesson Link: [optional url]
[lesson content...]

Lesson 1: [lesson title]
[content...]
```

## Architecture

### Core Pattern: Agentic Tool-Based RAG

This system uses an **agentic loop** where Claude decides when to search, rather than preprocessing every query. This is fundamentally different from traditional RAG:

**Traditional RAG**: Query → Embed → Search → Inject Results → Generate
**This System**: Query → Claude Decides → (Optional) Search → Generate

#### The Two-Call Pattern

Every tool-based query makes **two sequential API calls** to Claude ([ai_generator.py:43-135](backend/ai_generator.py#L43-L135)):

1. **First Call**: Claude receives the user query with available tools and decides whether to use `search_course_content`
   - If general knowledge: Returns answer directly (stop_reason: "end_turn")
   - If course-specific: Returns tool use request (stop_reason: "tool_use")

2. **Tool Execution**: If tool requested, `ToolManager.execute_tool()` runs the search

3. **Second Call**: Claude receives tool results and synthesizes final answer
   - Messages: [user_query, assistant_tool_use, user_tool_result]
   - No tools available in this call (already executed)
   - Returns final text response

This pattern is in `_handle_tool_execution()` - understanding this flow is critical for debugging or modifying AI behavior.

### Component Architecture

```
RAGSystem (rag_system.py) - Main orchestrator
├── DocumentProcessor - Parses course files into chunks
├── VectorStore - Manages two ChromaDB collections
│   ├── course_catalog - Course metadata for fuzzy name matching
│   └── course_content - Chunked course material for semantic search
├── AIGenerator - Handles Claude API with tool loop
├── SessionManager - Conversation history (last 2 exchanges)
└── ToolManager → CourseSearchTool → VectorStore
```

### ChromaDB Two-Collection Strategy

The system maintains **two separate collections** for different purposes ([vector_store.py:51-52](backend/vector_store.py#L51-L52)):

1. **`course_catalog`**: Course-level metadata
   - ID: Course title (e.g., "Building Agentic RAG with Claude")
   - Document: Course title (for embedding)
   - Metadata: Instructor, link, lessons JSON
   - Purpose: Fuzzy course name resolution ("MCP" → full title)

2. **`course_content`**: Chunked lesson content
   - ID: `{course_title}_{chunk_index}`
   - Document: Enriched chunk text with context prefix
   - Metadata: course_title, lesson_number, chunk_index
   - Purpose: Semantic content search

**Why two collections?** When a user searches with a partial course name like "MCP", the system:
1. Queries `course_catalog` to resolve "MCP" → "Building Agentic RAG with Claude"
2. Uses resolved title to filter `course_content` search
3. Returns only chunks from that course

See `VectorStore.search()` and `_resolve_course_name()` for implementation.

### Document Processing Pipeline

Chunks are enriched with context **before** embedding ([document_processor.py:184-234](backend/document_processor.py#L184-L234)):

- First chunk of lesson: `"Lesson 5 content: {text}"`
- Last lesson chunks: `"Course {title} Lesson {num} content: {text}"`

This context is embedded with the chunk, improving semantic search accuracy. The chunk_text() method uses sentence-based splitting (not fixed-size) with intelligent overlap to preserve semantic boundaries.

### Session Management and Context

Conversation history is **injected into the system prompt**, not as messages ([ai_generator.py:61-64](backend/ai_generator.py#L61-L64)):

```python
system_content = f"{SYSTEM_PROMPT}\n\nPrevious conversation:\n{history}"
```

This approach:
- Keeps the actual messages array clean (only current query)
- Allows Claude to see context without affecting tool use patterns
- Limited to last 2 exchanges (4 messages) per `config.MAX_HISTORY`

### Source Attribution Side-Channel

Sources are tracked **separately** from the AI conversation flow:

1. `CourseSearchTool` stores sources in `self.last_sources` during formatting
2. `ToolManager.get_last_sources()` retrieves them after AI response
3. Sources returned to frontend for UI display
4. `tool_manager.reset_sources()` cleans up for next query

Sources are NOT passed to Claude - they're purely for frontend citation display.

## Configuration

All settings in [backend/config.py](backend/config.py):

- `CHUNK_SIZE: 800` - Characters per chunk (sentence-based)
- `CHUNK_OVERLAP: 100` - Overlap between chunks
- `MAX_RESULTS: 5` - Vector search results returned
- `MAX_HISTORY: 2` - Conversation exchanges to remember
- `ANTHROPIC_MODEL` - Currently uses `claude-sonnet-4-20250514`
- `EMBEDDING_MODEL` - Uses `all-MiniLM-L6-v2` (384 dimensions)

## Key Implementation Details

### Tool Definition
Tools are defined in [search_tools.py](backend/search_tools.py) using Anthropic's tool schema. To add a new tool:
1. Create class inheriting from `Tool`
2. Implement `get_tool_definition()` returning Anthropic schema
3. Implement `execute(**kwargs)` with actual functionality
4. Register with `tool_manager.register_tool(your_tool)`

### ChromaDB Persistence
ChromaDB stores data in `./chroma_db` (relative to backend/). Data persists across restarts. To rebuild:
```python
rag_system.add_course_folder("../docs", clear_existing=True)
```

### Course Deduplication
On startup, the system checks existing course titles before adding documents. If a course with the same title exists, it skips re-processing. This prevents duplicate embeddings on restart.

### Error Handling in Search
`SearchResults` dataclass ([vector_store.py:8-32](backend/vector_store.py#L8-L32)) uses `error` field for graceful failures. Tool execution returns error messages as strings, which Claude can interpret and communicate to users naturally.
