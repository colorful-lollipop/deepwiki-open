# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DeepWiki-Open is an AI-powered wiki generator for code repositories. It analyzes GitHub/GitLab/Bitbucket repositories and generates interactive documentation with visual diagrams using RAG (Retrieval Augmented Generation).

**Architecture:**
- **Backend**: Python FastAPI (`api/`) - handles repo cloning, embeddings, RAG, and AI generation
- **Frontend**: Next.js (`src/`) - provides wiki UI with Mermaid diagram rendering
- **Model Clients**: Modular client architecture supporting multiple LLM providers (Google, OpenAI, OpenRouter, Bedrock, Azure, Ollama, Dashscope)
- **Data Pipeline**: Local storage of repos, embeddings (FAISS), and wiki cache in `~/.adalflow/`

## Development Commands

### Backend (Python/FastAPI)

```bash
# Install Python dependencies (requires poetry >= 2.0.1)
python -m pip install poetry==2.0.1 && poetry install -C api

# Start API server (default port 8001)
python -m api.main

# Set environment before running (create .env file first)
export PORT=8001
export LOG_LEVEL=DEBUG
export LOG_FILE_PATH=./api/logs/debug.log
python -m api.main
```

### Frontend (Next.js)

```bash
# Install JavaScript dependencies
npm install
# or
yarn install

# Start dev server (port 3000 with turbopack)
npm run dev
# or
yarn dev

# Build for production
npm run build

# Run production build
npm run start

# Lint code
npm run lint
```

### Docker Development

```bash
# Build and run with docker-compose
docker-compose up

# Run container directly
docker run -p 8001:8001 -p 3000:3000 \
  -v ~/.adalflow:/root/.adalflow \
  --env-file .env \
  ghcr.io/asyncfuncai/deepwiki-open:latest
```

## Code Architecture

### Backend Components (`api/`)

**Entry Point**: `main.py` - initializes uvicorn server with auto-reload in development

**Core API**: `api.py` - FastAPI application with WebSocket and HTTP endpoints

**Model Clients** (Provider Abstraction Layer):
- `openai_client.py` - OpenAI-compatible API client with batch embedding support (max 64 items per batch for ZhipuAI compatibility)
- `google_embedder_client.py` - Google AI embeddings
- `openrouter_client.py` - OpenRouter multi-model access
- `bedrock_client.py` - AWS Bedrock integration
- `azureai_client.py` - Azure OpenAI
- `dashscope_client.py` - Alibaba Dashscope
- All clients inherit from `adalflow.core.model_client.ModelClient`

**RAG Pipeline** (`rag.py`):
- `RAG` class manages retrieval-augmented generation
- `Memory` class maintains conversation history
- Uses FAISS retriever with embeddings from configured provider
- Supports multiple LLM providers through unified interface

**Data Pipeline** (`data_pipeline.py`):
- `download_repo()` - clones GitHub/GitLab/Bitbucket repos with token auth
- `DatabaseManager` - creates embeddings with `ToEmbeddings` pipeline
- `count_tokens()` - token counting with tiktoken (provider-specific encoding)
- `get_file_content()` - extracts and filters code files
- Text splitter uses configurable chunk sizes from `api/config/embedder.json`

**Wiki Generation** (`websocket_wiki.py`):
- `handle_websocket_chat()` - streaming wiki generation over WebSocket
- Supports custom file filters (included/excluded dirs and files)
- Multi-language support via `api/config/lang.json`

**Configuration** (`config.py` + `api/config/`):
- `generator.json` - model providers and parameters per provider
- `embedder.json` - embedding models and retriever settings
- `repo.json` - file filters and repo size limits
- Override with `DEEPWIKI_CONFIG_DIR` environment variable

### Frontend Components (`src/`)

**Next.js App Router Structure**:
- `src/app/page.tsx` - main landing page
- `src/app/[owner]/[repo]/page.tsx` - repository wiki view
- Uses React 19 and Next.js 15.3 with Turbopack

**Key Features**:
- Mermaid diagram rendering for code visualization
- Markdown rendering with syntax highlighting
- Multi-language i18n via next-intl

### Data Flow

1. **Repository Cloning**: `download_repo()` → stores in `~/.adalflow/repos/`
2. **Embedding Creation**: `DatabaseManager.create_documents_and_embeddings()` → FAISS index in `~/.adalflow/databases/`
3. **Wiki Generation**: WebSocket → RAG retrieval → LLM generation → cached in `~/.adalflow/wikicache/`
4. **Query**: User question → RAG retrieves relevant code → LLM generates response with context

## Environment Variables

**Required for basic operation:**
```
GOOGLE_API_KEY=sk-...     # For Google Gemini models
OPENAI_API_KEY=sk-...     # For embeddings (if using default OpenAI embedder)
```

**Provider-specific keys:**
```
OPENROUTER_API_KEY=...    # For OpenRouter models
AZURE_OPENAI_API_KEY=...  # For Azure OpenAI
AZURE_OPENAI_ENDPOINT=...
AZURE_OPENAI_VERSION=...
AWS_ACCESS_KEY_ID=...     # For Bedrock
AWS_SECRET_ACCESS_KEY=...
```

**Configuration:**
```
PORT=8001                          # API server port
DEEPWIKI_EMBEDDER_TYPE=openai      # openai|google|ollama|bedrock
OPENAI_BASE_URL=...                # Custom OpenAI endpoint
DEEPWIKI_CONFIG_DIR=...            # Custom config directory
LOG_LEVEL=INFO                     # DEBUG|INFO|WARNING|ERROR
LOG_FILE_PATH=./api/logs/app.log   # Log file location
DEEPWIKI_AUTH_MODE=true            # Enable authorization
DEEPWIKI_AUTH_CODE=secret          # Auth code when enabled
OLLAMA_HOST=http://localhost:11434 # Ollama server address
```

## Common Issues and Solutions

### Empty Embedding Vectors
- **Symptom**: "Document X has empty embedding vector, skipping"
- **Cause**: Batch processing with incorrect index management after merging multiple embedding API responses
- **Fix**: The `OpenAIClient.call()` and `acall()` methods now properly adjust global indices when merging batches from ZhipuAI (64-item limit)

### Model Client Configuration
- Model clients are loaded dynamically based on `provider` parameter in requests
- Provider defaults to "google" but can be overridden in UI
- Custom models can be entered if `supportsCustomModel: true` in `generator.json`

### File Filtering
- Use `excluded_dirs`, `excluded_files`, `included_dirs`, `included_files` in API requests
- Default filters defined in `api/config/repo.json`
- Useful for excluding tests, docs, or generated code from embeddings

### Development Server Reloading
- The `api/main.py` patches `watchfiles.watch()` to exclude logs directory from auto-reload
- Set `NODE_ENV=production` to disable hot-reload
- Logs directory is always excluded to prevent infinite reload loops

### Batch Embedding Limits
- ZhipuAI (via OpenAI-compatible endpoints) limits to 64 items per batch
- The `OpenAIClient` automatically splits large inputs into batches and merges responses
- Other providers may have different limits (check their documentation)

## Testing

Backend tests use pytest:
```bash
cd api
poetry run pytest
```

Frontend tests (if present):
```bash
npm test
# or
yarn test
```
