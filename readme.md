# Personal Coding Agent

Personal Coding Agent is a command-line, RAG-powered code assistant for asking questions about a local codebase. It indexes source files into a vector store, exposes a LangChain agent with retrieval, filesystem, terminal, and MCP tools, and persists conversation state through LangGraph's SQLite checkpointer.

The package entry point is configured in `pyproject.toml` as `educosys_claude = "educosys_claude.main:run"`.

---

## Table of Contents

1. [What the Project Does](#what-the-project-does)
2. [High-Level Architecture](#high-level-architecture)
3. [End-to-End Runtime Flow](#end-to-end-runtime-flow)
4. [Retrieval-Augmented Generation Flow](#retrieval-augmented-generation-flow)
5. [Directory and File Responsibilities](#directory-and-file-responsibilities)
6. [Configuration](#configuration)
7. [Memory and Sessions](#memory-and-sessions)
8. [Tools and MCP Integration](#tools-and-mcp-integration)
9. [How to Run](#how-to-run)
10. [Operational Notes](#operational-notes)

---

## What the Project Does

At a high level, Educosys Claude:

1. Loads environment variables from `.env` in `educosys_claude/main.py`.
2. Loads application settings from `educosys_claude/config.yaml` via `educosys_claude/config.py`.
3. Creates an LLM and embedding model using provider-specific factories in `educosys_claude/llm/factory.py`.
4. Parses and indexes the current working directory as a codebase using Tree-sitter plus vector storage.
5. Builds a LangChain agent with code-search, filesystem, terminal, and MCP tools.
6. Starts an interactive REPL where users can ask codebase questions with `/ask <question>`.
7. Routes each user question to the agent through `educosys_claude/agent/orchestrator.py`.
8. Persists conversation state by thread/session id through `AsyncSqliteSaver`.

---

## High-Level Architecture

```text
User terminal
    |
    v
educosys_claude/main.py
    |-- loads config and .env
    |-- initializes LLM/embedder
    |-- indexes current working directory
    |-- builds LangChain agent
    |-- starts REPL
    |
    v
LangChain agent
    |-- search_codebase tool
    |      |
    |      v
    |   Retriever factory -> Chroma/Qdrant/Hybrid retriever
    |
    |-- filesystem tools
    |-- terminal tools
    |-- external MCP tools
    |
    v
LLM provider
    |-- OpenAI or Anthropic chat model

Indexing side:

Current working directory
    |
    v
code_parser.py
    |-- Tree-sitter AST chunks for code files
    |-- sliding-window chunks for text/config files
    |
    v
Indexer factory
    |-- semantic Chroma
    |-- semantic Qdrant
    |-- hybrid Qdrant
    |
    v
Vector store
```

The main architectural pattern is a **factory-driven RAG agent**:

- `config.yaml` chooses providers and modes.
- Factory modules select the implementation at runtime.
- Indexer modules create the searchable knowledge base.
- Retriever modules query that knowledge base.
- Agent tools expose retrieval and system actions to the LLM.

---

## End-to-End Runtime Flow

### 1. Program startup

The executable path is `educosys_claude.main:run`, declared in `pyproject.toml` under `[tool.poetry.scripts]`.

`run()` in `educosys_claude/main.py` calls `asyncio.run(_run_async())`. `_run_async()` is the top-level async runner that prints the CLI banner, opens the SQLite checkpointer, initializes dependencies, and starts the input loop.

Relevant code:

- `educosys_claude/main.py:103` — `run()`
- `educosys_claude/main.py:55` — `_run_async()`
- `educosys_claude/main.py:37` — `initialize(checkpointer)`
- `educosys_claude/main.py:28` — `get_or_create_index()`

### 2. Configuration and environment loading

At import time, `educosys_claude/main.py` loads `.env` from the parent project directory. Static application settings are loaded by `load_config()` in `educosys_claude/config.py:5`, which reads `educosys_claude/config.yaml`.

The global `config` object is imported throughout the application by factories, indexers, retrievers, and memory modules.

### 3. Initialization

`initialize(checkpointer)` in `educosys_claude/main.py:37` performs the bootstrapping sequence:

1. Calls `get_llm()` from `educosys_claude/llm/factory.py:8`.
2. Calls `get_embedder()` from `educosys_claude/llm/factory.py:23`.
3. Calls `get_or_create_index()` in `educosys_claude/main.py:28` to index the current working directory.
4. Calls `build_agent(checkpointer)` from `educosys_claude/agent/factory.py:27`.
5. Gets or creates a session id through `get_current_session()` in `educosys_claude/memory/session.py:16`.

### 4. REPL command loop

After initialization, `_run_async()` waits for commands from `rich.prompt.Prompt`.

Supported commands are implemented directly in `educosys_claude/main.py`:

- `/ask <question>` — sends a question to the agent.
- `/show_index` — displays stored vector chunks.
- `/new_session` — creates a new conversation session.
- `/switch <session_id>` — resumes a known session.
- `/session` — prints the current session id.
- `/exit` or `/quit` — stops the app.

For `/ask`, the REPL calls `handle_query(agent, question, session_id)` from `educosys_claude/agent/orchestrator.py:7`.

### 5. Agent invocation

`handle_query()` builds a LangGraph/LangChain config object with the current `thread_id`, then calls:

```python
await agent.ainvoke({"messages": [{"role": "user", "content": question}]}, agent_config)
```

The thread id is what connects a user session to persisted checkpointer state.

Errors are caught in `handle_query()` and returned as `Error: ...` strings, rather than crashing the REPL.

---

## Retrieval-Augmented Generation Flow

The RAG pipeline has two sides: indexing and retrieval.

### A. Indexing flow

```text
main.py:get_or_create_index()
    |
    v
context/indexers/factory.py:get_indexer()
    |
    +--> semantic_chroma.index_codebase()
    +--> semantic_qdrant.index_codebase()
    +--> hybrid_qdrant.index_codebase()
            |
            v
        code_parser.get_source_files()
            |
            v
        code_parser.parse_file()
            |
            +--> Tree-sitter AST parsing for code files
            +--> sliding-window parsing for text/config files
            |
            v
        embeddings + vector store upsert
```

`get_indexer()` in `educosys_claude/context/indexers/factory.py:8` selects the implementation using:

- `config["rag"]["mode"]`
- `config["vector_store"]["provider"]`

If `rag.mode == "hybrid"` and `vector_store.provider == "qdrant"`, the hybrid Qdrant indexer is used. If the provider is Qdrant but not hybrid, the semantic Qdrant indexer is used. Otherwise, Chroma is used.

### B. Code parsing

`educosys_claude/context/indexers/code_parser.py` owns source discovery and chunk creation.

Important objects and functions:

- `EXTENSION_TO_LANGUAGE` at `code_parser.py:17` maps source extensions to Tree-sitter language names.
- `TEXT_EXTENSIONS` at `code_parser.py:40` defines text/config files that use sliding-window chunking.
- `CHUNK_SIZE` and `CHUNK_OVERLAP` at `code_parser.py:46-47` control fixed-size text chunking.
- `BLOCK_NODE_TYPES` at `code_parser.py:52` defines AST node types considered meaningful chunks.
- `parse_file()` at `code_parser.py:71` routes each file to AST parsing or text parsing.
- `_parse_with_treesitter()` at `code_parser.py:90` parses code files into Tree-sitter ASTs.
- `_walk()` at `code_parser.py:112` recursively walks AST nodes and extracts top-level functions/classes.
- `_extract_name()` at `code_parser.py:137` determines each chunk's name.
- `_sliding_window()` at `code_parser.py:145` creates overlapping chunks for text/config files.
- `get_source_files()` at `code_parser.py:172` recursively finds indexable files and skips generated or dependency directories.

The parser returns `ParsedChunk` objects containing:

- chunk name
- chunk type
- source file path
- start line
- end line
- chunk content

### C. Vector storage

There are three indexing implementations:

#### `semantic_chroma.py`

`index_codebase()` at `educosys_claude/context/indexers/semantic_chroma.py:13`:

1. Creates a Chroma persistent client using `config["chromadb"]["persist_dir"]`.
2. Gets or creates the configured collection.
3. Skips re-indexing if the collection already contains chunks.
4. Parses each source file.
5. Embeds each chunk using `get_embedder()`.
6. Upserts each chunk into Chroma with metadata.

`show_index()` at `semantic_chroma.py:64` prints every stored Chroma chunk using Rich.

#### `semantic_qdrant.py`

`index_codebase()` at `educosys_claude/context/indexers/semantic_qdrant.py:18`:

1. Reads `QDRANT_URL` and `QDRANT_API_KEY` from the environment.
2. Checks whether the target Qdrant collection already has points.
3. Parses files into chunks.
4. Wraps chunks as LangChain `Document` objects.
5. Creates or loads a `QdrantVectorStore`.

`show_index()` at `semantic_qdrant.py:86` scrolls through Qdrant points and prints metadata.

#### `hybrid_qdrant.py`

`hybrid_qdrant.py` supports dense, sparse, and hybrid retrieval modes.

Important code:

- `RETRIEVAL_MODE_MAP` at `hybrid_qdrant.py:16`
- `_get_retrieval_mode()` at `hybrid_qdrant.py:25`
- `index_codebase()` at `hybrid_qdrant.py:32`
- `show_index()` at `hybrid_qdrant.py:104`

Hybrid indexing uses:

- dense embeddings from `get_embedder()`
- sparse BM25-style embeddings via `FastEmbedSparse(model_name="Qdrant/bm25")`
- Qdrant retrieval modes from `langchain_qdrant.RetrievalMode`

### D. Retrieval flow

When the LLM uses the `search_codebase` tool, this flow happens:

```text
agent/tools.py:search_codebase(query)
    |
    v
context/retrievers/factory.py:get_retriever()
    |
    +--> semantic_chroma.retrieve()
    +--> semantic_qdrant.retrieve()
    +--> hybrid_qdrant.retrieve()
            |
            v
        vector store similarity search
            |
            v
        list of chunk dictionaries
            |
            v
        formatted tool output with file, lines, type, name, and code
```

`search_codebase()` in `educosys_claude/agent/tools.py:12` is decorated with `@tool`, so LangChain exposes it to the agent. It retrieves the top five chunks and formats them with file names, line numbers, chunk types, names, and content.

Retriever selection is handled by `get_retriever()` in `educosys_claude/context/retrievers/factory.py:10`, using the same config-driven decision pattern as indexing.

---

## Directory and File Responsibilities

### Project root

#### `pyproject.toml`

Defines package metadata, Python version, dependencies, and the CLI script. Notable dependencies include:

- `langchain`
- `langchain-openai`
- `langchain-anthropic`
- `langchain-huggingface`
- `chromadb`
- `langchain-qdrant`
- `tree-sitter`
- `tree-sitter-languages`
- `langgraph-checkpoint-sqlite`
- `langchain-mcp-adapters`
- `rich`
- `pydantic`
- `pyyaml`

#### `.env`

Stores runtime secrets and environment-specific values such as API keys, Qdrant URL/API key, GitHub token, and current working directory for MCP filesystem access. This file is loaded by `educosys_claude/main.py` and MCP config loading.

#### `.memory/`

Runtime directory used for SQLite checkpoint state and the `current_session` file. The application creates it automatically when needed.

---

## `educosys_claude/`

This is the main Python package.

### `educosys_claude/main.py`

Main CLI entry point.

Key functions:

- `get_or_create_index()` at `main.py:28`
- `initialize(checkpointer)` at `main.py:37`
- `_run_async()` at `main.py:55`
- `run()` at `main.py:103`

Responsibilities:

1. Loads `.env`.
2. Creates a Rich console.
3. Initializes the LLM, embedder, index, agent, and session.
4. Creates an `AsyncSqliteSaver` checkpointer using `get_checkpointer_db_path()`.
5. Starts the interactive command loop.
6. Routes `/ask` commands to `handle_query()`.
7. Routes `/show_index` to the selected index inspector.
8. Manages session commands.

The application indexes `Path.cwd()`, meaning the codebase being indexed is the directory from which the CLI is launched.

### `educosys_claude/config.py`

Contains `load_config()` at `config.py:5`.

Responsibilities:

- Reads `educosys_claude/config.yaml`.
- Parses YAML into a Python dictionary.
- Exposes the global `config` object to the rest of the package.

### `educosys_claude/config.yaml`

Central configuration file.

Major sections:

- `memory` — SQLite DB path and summarization settings.
- `rag` — retrieval mode, currently `semantic` or `hybrid`.
- `vector_store` — backend provider and retrieval mode.
- `llm` — chat model provider and model name.
- `embeddings` — embedding provider and model name.
- `chromadb` — Chroma persistence and collection name.
- `qdrant` — Qdrant collection name.
- `elasticsearch` — present, but no Elasticsearch implementation exists in the current codebase.

Note: `vector_store` appears twice in the YAML. YAML parsers generally keep the later duplicate key, so the later `vector_store: provider: qdrant` may override the earlier `vector_store` block.

---

## Agent Layer

### `educosys_claude/agent/factory.py`

Builds the LangChain agent.

Important code:

- `SYSTEM_PROMPT` at `agent/factory.py:21`
- `build_agent(checkpointer)` at `agent/factory.py:27`

Responsibilities:

1. Creates the configured LLM through `get_llm()`.
2. Loads external MCP tools through `get_educosys_mcp_tools()`.
3. Registers built-in tools:
   - `search_codebase`
   - `run_command`
   - `run_in_directory`
   - `read_file`
   - `write_file`
   - `append_file`
   - `list_directory`
   - `file_exists`
4. Adds MCP tools to the same tool list.
5. Calls `create_agent()` with the LLM, tools, system prompt, and checkpointer.

The system prompt instructs the agent to behave like a senior software engineer, use `search_codebase` before answering, and cite file/function/line references.

### `educosys_claude/agent/orchestrator.py`

Contains `handle_query()` at `agent/orchestrator.py:7`.

Responsibilities:

- Receives a user question and session/thread id.
- Builds the checkpointer config: `{"configurable": {"thread_id": thread_id}}`.
- Invokes the LangChain agent asynchronously.
- Returns the final assistant message content.
- Converts runtime exceptions into error strings for CLI display.

### `educosys_claude/agent/tools.py`

Defines the code-search tool exposed to the agent.

Key function:

- `search_codebase(query)` at `agent/tools.py:12`

Responsibilities:

1. Gets the correct retriever using `get_retriever()`.
2. Retrieves five relevant chunks.
3. Formats chunks into a string containing:
   - file path
   - line range
   - chunk type
   - chunk name
   - code content
4. Returns `No relevant code found.` if retrieval is empty.

---

## LLM Layer

### `educosys_claude/llm/factory.py`

Provider factory for chat models and embedding models.

Key functions:

- `get_llm()` at `llm/factory.py:8`
- `get_embedder()` at `llm/factory.py:23`

`get_llm()` behavior:

- If `config["llm"]["provider"] == "anthropic"`, returns `ChatAnthropic`.
- Otherwise, returns `ChatOpenAI`.

`get_embedder()` behavior:

- If `config["embeddings"]["provider"] == "huggingface"`, returns `HuggingFaceEmbeddings`.
- Otherwise, returns `OpenAIEmbeddings`.

---

## Context Indexing Layer

### `educosys_claude/context/indexers/factory.py`

Selects the indexer and index-inspector implementation.

Key functions:

- `get_indexer()` at `context/indexers/factory.py:8`
- `get_index_inspector()` at `context/indexers/factory.py:24`

Selection rules:

- Hybrid + Qdrant -> `hybrid_qdrant`
- Qdrant -> `semantic_qdrant`
- Anything else -> `semantic_chroma`

### `educosys_claude/context/indexers/code_parser.py`

Responsible for parsing a repository into semantic chunks.

It supports many programming languages through Tree-sitter and supports text/config files through sliding windows.

Important behavior:

- Skips directories like `.venv`, `venv`, `__pycache__`, `.git`, `node_modules`, `dist`, and `build` in `get_source_files()`.
- Stops descending into nested functions/classes once a top-level block is captured in `_walk()`.
- Falls back to sliding-window chunking if Tree-sitter produces no meaningful blocks.

### `educosys_claude/context/indexers/semantic_chroma.py`

Chroma-based semantic indexing.

Use this when the configured vector provider is not Qdrant.

Stores each chunk with:

- document text
- dense embedding
- source path
- chunk name
- chunk type
- start/end lines

### `educosys_claude/context/indexers/semantic_qdrant.py`

Qdrant-based dense semantic indexing.

Uses `QdrantVectorStore.from_documents()` for new indexes and `QdrantVectorStore.from_existing_collection()` for existing collections.

### `educosys_claude/context/indexers/hybrid_qdrant.py`

Qdrant-based hybrid indexing.

Adds sparse BM25-style embedding support using `FastEmbedSparse(model_name="Qdrant/bm25")`. Retrieval mode is controlled by `config["vector_store"]["retrieval_mode"]` and can be dense, sparse, or hybrid.

---

## Context Retrieval Layer

### `educosys_claude/context/retrievers/factory.py`

Contains `get_retriever()` at `context/retrievers/factory.py:10`.

It mirrors the indexer factory and returns the matching retrieval function for the configured RAG/vector-store mode.

### `educosys_claude/context/retrievers/semantic_chroma.py`

Contains `retrieve(query, k=5)` at `context/retrievers/semantic_chroma.py:10`.

Responsibilities:

1. Opens the configured Chroma collection.
2. Embeds the query using the configured embedder.
3. Calls `collection.query()`.
4. Converts Chroma results into normalized chunk dictionaries.

### `educosys_claude/context/retrievers/semantic_qdrant.py`

Contains `retrieve(query, k=5)` at `context/retrievers/semantic_qdrant.py:13`.

Responsibilities:

1. Opens an existing Qdrant collection.
2. Runs `similarity_search_with_score()`.
3. Converts LangChain `Document` results into normalized chunk dictionaries.

### `educosys_claude/context/retrievers/hybrid_qdrant.py`

Contains `retrieve(query, k=5)` at `context/retrievers/hybrid_qdrant.py:20`.

Responsibilities:

1. Reads the configured retrieval mode.
2. Opens a Qdrant vector store with dense and sparse embeddings.
3. Runs similarity search.
4. Returns normalized chunk dictionaries.

---

## Memory Layer

### `educosys_claude/memory/session.py`

Manages user-facing session ids.

Key functions:

- `_session_file()` at `memory/session.py:12`
- `get_current_session()` at `memory/session.py:16`
- `new_session()` at `memory/session.py:25`
- `switch_session(session_id)` at `memory/session.py:34`

The current session id is stored in a file named `current_session` inside the parent directory of `config["memory"]["db_path"]`.

### `educosys_claude/memory/short_term.py`

Manages LangGraph checkpoint storage and summarization middleware.

Key functions:

- `get_checkpointer_db_path()` at `memory/short_term.py:12`
- `get_checkpointer()` at `memory/short_term.py:19`
- `get_summarization_middleware()` at `memory/short_term.py:24`

`get_checkpointer_db_path()` creates the memory directory if needed and returns the SQLite database path. `get_summarization_middleware()` constructs `SummarizationMiddleware`, but the current `build_agent()` implementation does not pass that middleware into `create_agent()`.

---

## MCP Layer

### `educosys_claude/mcp/educosys_mcp_client.py`

Contains `get_educosys_mcp_tools()` at `mcp/educosys_mcp_client.py:9`.

Responsibilities:

1. Loads MCP server configs.
2. Creates `MultiServerMCPClient`.
3. Retrieves all remote MCP tools.
4. Returns them so `agent/factory.py` can add them to the agent.

### `educosys_claude/mcp/educosys_mcp_config.py`

Contains `load_educosys_mcp_configs()` at `mcp/educosys_mcp_config.py:14`.

Responsibilities:

- Reads `educosys_mcp_servers.json`.
- Replaces `${VAR}` placeholders with environment variable values.
- Returns the `mcp_servers` dictionary.

### `educosys_claude/educosys_mcp_servers.json`

Declares external MCP servers.

Configured servers:

1. `github`
   - Runs `npx -y @modelcontextprotocol/server-github`.
   - Receives `GITHUB_PERSONAL_ACCESS_TOKEN` from `${GITHUB_TOKEN}`.
2. `filesystem`
   - Runs `npx -y @modelcontextprotocol/server-filesystem ${CWD}`.
   - Exposes filesystem access rooted at `${CWD}`.

---

## Local Tool Layer

### `educosys_claude/tools/filesystem_tools.py`

LangChain tools for file operations.

Key tools:

- `read_file(file_path)` at `tools/filesystem_tools.py:9`
- `write_file(file_path, content)` at `tools/filesystem_tools.py:32`
- `append_file(file_path, content)` at `tools/filesystem_tools.py:51`
- `delete_file(file_path)` at `tools/filesystem_tools.py:70`
- `list_directory(directory)` at `tools/filesystem_tools.py:88`
- `file_exists(file_path)` at `tools/filesystem_tools.py:106`

Safety and validation:

- Empty paths are rejected.
- Missing files/directories return error strings.
- `read_file()` limits file size to 10 MB.
- `read_file()` expects UTF-8 text.
- Permission errors are caught and returned as tool output.

Note: `delete_file()` exists in this file but is not currently registered in `agent/factory.py`'s tool list.

### `educosys_claude/tools/terminal_tools.py`

LangChain tools for shell execution.

Key helpers and tools:

- `_is_blocked(command)` at `tools/terminal_tools.py:10`
- `_format_result(result)` at `tools/terminal_tools.py:14`
- `run_command(command)` at `tools/terminal_tools.py:26`
- `run_in_directory(command, directory)` at `tools/terminal_tools.py:48`

Safety and validation:

- Blocks dangerous command patterns such as `rm -rf /`, `mkfs`, `dd if=`, and fork-bomb syntax.
- Rejects empty commands.
- Times out after 30 seconds.
- Captures stdout, stderr, and exit code.

---

## Observability

### `educosys_claude/observability/logger.py`

Contains `get_logger(name)` at `observability/logger.py:11`.

Responsibilities:

- Configures root logging at `WARNING` to reduce noisy third-party logs.
- Returns module-specific loggers set to `DEBUG` for the application's own modules.

---

## Configuration

The main configuration lives in `educosys_claude/config.yaml`.

Example conceptual configuration:

```yaml
memory:
  db_path: .memory/memory.db
  summarize_at_tokens: 4000
  keep_last_messages: 20

rag:
  mode: semantic

vector_store:
  provider: qdrant
  retrieval_mode: hybrid

llm:
  provider: openai
  model: gpt-5.5

embeddings:
  provider: openai
  model: text-embedding-3-small

qdrant:
  collection_name: codebase3
```

Environment variables commonly required:

```env
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
QDRANT_URL=...
QDRANT_API_KEY=...
GITHUB_TOKEN=...
CWD=/path/to/allowed/filesystem/root
```

Only the variables needed by the selected providers must be set.

---

## Memory and Sessions

The application has two session concepts:

1. **Current session id file**
   - Managed by `educosys_claude/memory/session.py`.
   - Stores the active session id in `.memory/current_session` when using the default config.

2. **LangGraph checkpoint database**
   - Managed through `AsyncSqliteSaver` in `educosys_claude/main.py`.
   - Path comes from `config["memory"]["db_path"]`.
   - Stores conversation state keyed by `thread_id`.

When `/ask` is used, `handle_query()` sends the current session id as the LangGraph `thread_id`. This allows separate conversations to be resumed using `/switch <session_id>`.

---

## Tools and MCP Integration

The agent receives three categories of tools:

### 1. RAG tool

- `search_codebase()` from `educosys_claude/agent/tools.py`

This is the primary code-understanding tool.

### 2. Local tools

From `educosys_claude/tools/filesystem_tools.py`:

- read file
- write file
- append file
- list directory
- check existence

From `educosys_claude/tools/terminal_tools.py`:

- run shell command
- run shell command in a directory

### 3. MCP tools

Loaded dynamically from `educosys_claude/educosys_mcp_servers.json` through:

- `load_educosys_mcp_configs()`
- `get_educosys_mcp_tools()`

These tools extend the agent with external capabilities such as GitHub operations and filesystem access through standard MCP servers.

---

## How to Run

### 1. Install dependencies

Using Poetry:

```bash
poetry install
```

### 2. Configure environment

Create or update `.env` with the required provider keys. For OpenAI + Qdrant, for example:

```env
OPENAI_API_KEY=your_openai_key
QDRANT_URL=your_qdrant_url
QDRANT_API_KEY=your_qdrant_api_key
GITHUB_TOKEN=your_github_token
CWD=/absolute/path/to/project
```

### 3. Start the CLI

```bash
poetry run educosys_claude
```

or:

```bash
python -m educosys_claude.main
```

### 4. Ask questions

```text
/ask explain the architecture of this project
/ask where is the retriever selected?
/ask how does indexing work?
/show_index
/session
/new_session
/switch <session_id>
/exit
```

---

## Operational Notes

1. **The indexed repository is `Path.cwd()`**
   - `get_or_create_index()` uses the current working directory, so run the CLI from the codebase you want to index.

2. **Existing indexes are reused**
   - Chroma and Qdrant indexers skip re-indexing if the target collection already contains chunks.
   - If code changes are not reflected, clear the vector collection or use a new collection name.

3. **Config drives implementation selection**
   - Indexers and retrievers are both selected by `rag.mode` and `vector_store.provider`.

4. **MCP depends on Node/npm tools**
   - The configured MCP servers use `npx`, so Node.js must be available.

5. **Elasticsearch is configured but not implemented**
   - `config.yaml` contains an `elasticsearch` block, but there are no Elasticsearch indexer/retriever files in the current codebase.

6. **Summarization middleware exists but is not wired into the agent**
   - `get_summarization_middleware()` exists in `memory/short_term.py`, but `build_agent()` currently passes only the checkpointer into `create_agent()`.

---

## Summary

Educosys Claude is organized around a clean RAG-agent architecture:

- `main.py` owns the CLI lifecycle.
- `config.py` and `config.yaml` own runtime settings.
- `llm/factory.py` owns provider-specific model creation.
- `context/indexers/*` build the codebase index.
- `context/retrievers/*` search that index.
- `agent/*` builds and invokes the LangChain agent.
- `memory/*` persists sessions and conversation checkpoints.
- `tools/*` exposes local filesystem and terminal capabilities.
- `mcp/*` loads external MCP tools.
- `observability/logger.py` centralizes logging behavior.

Together, these modules create an interactive assistant that can inspect, retrieve, reason about, and operate on a local codebase.
