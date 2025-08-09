# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Starting the Application
```bash
# Quick start with shell script (from starting-ragchatbot-codebase/)
cd starting-ragchatbot-codebase
chmod +x run.sh
./run.sh

# Manual start from backend directory
cd starting-ragchatbot-codebase/backend
uv run uvicorn app:app --reload --port 8000
```

### Dependency Management
```bash
# Install/sync dependencies (from starting-ragchatbot-codebase/)
cd starting-ragchatbot-codebase
uv sync

# Add new dependencies
uv add <package-name>
```

### Environment Setup
- Create `.env` file in `starting-ragchatbot-codebase/` root with `ANTHROPIC_API_KEY=your_key_here`
- Application runs on `http://localhost:8000`
- API docs available at `http://localhost:8000/docs`
- ChromaDB data persisted in `./chroma_db` directory

### Working with Course Documents
```bash
# Add new course documents to docs/ folder
# Supported formats: .txt, .pdf, .docx
# Expected format:
# Course Title: [title]
# Course Link: [url] 
# Course Instructor: [instructor]
# 
# Lesson 0: Introduction
# [content]
```

## Architecture Overview

This is a **tool-based RAG system** where Claude AI autonomously decides when to search course content rather than always retrieving context. The system processes structured course documents and provides conversational access to educational materials.

### Core Data Flow

**Query Processing Chain:**
1. **Frontend** (`frontend/script.js`) → POST `/api/query` with user question + session_id
2. **FastAPI** (`app.py`) → calls `rag_system.query()` 
3. **RAGSystem** (`rag_system.py`) → calls `ai_generator.generate_response()` with available tools
4. **AIGenerator** (`ai_generator.py`) → Claude API call with tool definitions
5. **Claude decides** whether to use `search_course_content` tool based on query type
6. **If search needed**: `CourseSearchTool` → `VectorStore` → ChromaDB semantic search
7. **Response synthesis**: Claude combines search results into natural language answer
8. **Sources tracking**: UI displays which course/lesson content was referenced

### Key Components

**RAGSystem** (`rag_system.py`): Central orchestrator
- Manages tool registration and execution flow
- Handles session management and conversation history
- Coordinates between document processing, vector storage, and AI generation
- Returns structured responses with sources for UI display

**Document Processing Pipeline** (`document_processor.py`):
- Parses structured course format: Course Title/Link/Instructor → Lesson markers
- Chunks content at 800 chars with 100 char overlap at sentence boundaries
- Creates `CourseChunk` objects with metadata (course_title, lesson_number, chunk_index)
- Adds contextual prefixes: `"Course [title] Lesson [N] content: [chunk]"`

**Vector Storage** (`vector_store.py`):
- ChromaDB with sentence-transformers embeddings (`all-MiniLM-L6-v2`)
- Separate collections for course metadata and content chunks
- Supports semantic search with course_name and lesson_number filtering
- Returns `SearchResults` objects with documents, metadata, and similarity scores

**AI Tool System** (`search_tools.py`, `ai_generator.py`):
- `CourseSearchTool`: Implements search interface for Claude with parameter validation
- `ToolManager`: Registers tools and tracks sources for UI attribution
- Claude autonomously chooses whether to search based on query analysis
- Tool calls include query, optional course_name, and optional lesson_number filters

**Session Management** (`session_manager.py`):
- Maintains conversation history (max 2 exchanges by default)
- Creates unique session IDs for frontend state management
- Formats conversation context for Claude API calls

### Configuration System

**Centralized Settings** (`config.py`):
- Uses dataclass pattern with environment variable loading
- Claude model: `claude-sonnet-4-20250514`
- Embedding model: `all-MiniLM-L6-v2` 
- Chunk processing: 800 chars with 100 overlap
- Search limits: 5 max results, 2 conversation history exchanges
- ChromaDB persistence path: `./chroma_db`

### Key Architectural Patterns

**Tool-Based RAG vs Traditional RAG**:
- Traditional: Always retrieves context before generation
- This system: Claude decides when search is needed based on query type
- Reduces unnecessary searches for general knowledge questions
- Enables more targeted, context-aware searches

**Incremental Document Loading**:
- `add_course_folder()` checks existing course titles before processing
- Avoids re-processing documents on restart
- Supports hot-reload of new course materials

**Structured Course Format**:
- Enforces consistent document structure for reliable parsing
- Separates course metadata from lesson content
- Enables precise filtering by course name and lesson number

**Session-Based Context**:
- Maintains conversation continuity across queries
- Enables follow-up questions with proper context
- Limits history to prevent context window overflow

**Source Attribution**:
- Tracks which course/lesson content was used for each response
- Provides transparent sourcing in UI collapsible sections
- Enables users to verify and explore referenced materials
- Always use uv to run the server and do not use pip directly
- make sure to use uv to manage all dependencies