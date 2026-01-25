# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Multi-Source RAG + Text-to-SQL system: a FastAPI application combining document RAG (Retrieval-Augmented Generation) with Text-to-SQL capabilities. Features intelligent query routing, multi-level caching, and AWS Lambda deployment.

## Common Commands

### Development
```bash
# Install dependencies
pip install -r requirements.txt
# OR with UV (faster)
uv pip install -r requirements.txt

# Run locally
uvicorn app.main:app --reload

# Run with specific host/port
python -m app.main
```

### Testing & Linting
```bash
# Run tests
pytest tests/ -v

# Lint
ruff check app/

# Format check
black --check app/

# Format code
black app/
```

### Evaluation
```bash
# Run RAGAS evaluation (measures faithfulness and answer relevancy)
python evaluate.py
```

### Docker (Local Development)
```bash
docker-compose up -d
docker-compose logs -f
docker-compose down
```

### Deployment
Push to `main` branch triggers automatic deployment to AWS Lambda via GitHub Actions. The CI/CD pipeline:
1. Builds x86-64 Docker image from `Dockerfile.lambda`
2. Pushes to Amazon ECR
3. Updates Lambda function
4. Runs health checks

## Architecture

### Core Services (`app/services/`)

- **embedding_service.py**: OpenAI text-embedding-3-small (1536 dimensions) with Redis caching
- **vector_service.py**: Pinecone operations (upsert, search, delete) using gRPC
- **rag_service.py**: Retrieval + GPT-4 answer generation with source citations
- **sql_service.py**: Vanna 2.0 agent for Text-to-SQL with approval workflow
- **router_service.py**: Keyword-based query routing (SQL/DOCUMENTS/HYBRID using 60+ keywords)
- **document_service.py**: Document parsing (Docling primary, Unstructured fallback) and chunking
- **docling_service.py**: Context-aware parsing with HybridChunker for structure preservation

### Caching Architecture (Two-Tier)

1. **Query Cache** (`query_cache_service.py`): Redis-based, 5-10ms retrieval
   - RAG answers (1h TTL), SQL generation (24h TTL), SQL results (15min TTL), embeddings (7d TTL)

2. **Document Cache** (`cache_service.py`): S3/local storage, 100-200ms retrieval
   - SHA-256 content-based deduplication for chunks, embeddings, metadata

### Storage Backends (`app/services/`)
- **storage_backend.py**: Abstract interface for storage
- **s3_storage.py**: AWS S3 backend (production)
- **local_storage.py**: Local filesystem backend (development)

### Entry Points
- **app/main.py**: FastAPI app with 18 endpoints, service initialization
- **lambda_handler.py**: AWS Lambda entry point using Mangum
- **app/config.py**: Pydantic Settings for environment configuration

## Key Configuration (Environment Variables)

Required:
- `OPENAI_API_KEY`: For embeddings and LLM (RAG)
- `PINECONE_API_KEY`: For vector storage
- `DATABASE_URL`: PostgreSQL connection for Text-to-SQL

Optional:
- `UPSTASH_REDIS_URL`, `UPSTASH_REDIS_TOKEN`: Query caching (40-60% cost reduction)
- `OPIK_API_KEY`: LLM monitoring
- `STORAGE_BACKEND`: "s3" or "local" (default: "s3")
- `S3_CACHE_BUCKET`: S3 bucket for document cache

SQL Determinism (for consistent SQL generation):
- `VANNA_TEMPERATURE=0.0`: Fully deterministic
- `VANNA_TOP_P=0.1`, `VANNA_SEED=42`

## Query Routing Logic

The `QueryRouter` in `router_service.py` uses keyword analysis:
- **SQL**: Keywords like `count`, `total`, `sum`, `average`, `revenue`, `how many`, `list all`
- **DOCUMENTS**: Keywords like `what is`, `explain`, `define`, `policy`, `procedure`, `how to`
- **HYBRID**: Keywords like `and explain`, `show data and explain`

## API Endpoints Structure

- `/query`: Unified endpoint with intelligent routing (recommended)
- `/query/documents`: Direct RAG queries
- `/query/sql/generate`, `/query/sql/execute`, `/query/sql/pending`: SQL workflow
- `/upload`: Document upload with automatic chunking and embedding
- `/documents`: List uploaded documents
- `/cache/*`: Cache management (stats, clear)
- `/vectors/clear`: Clear Pinecone vectors
- `/health`, `/info`, `/stats`: System information

## Code Conventions

- Type hints on all functions
- Google-style docstrings
- Line length: 100 characters (configured in `pyproject.toml`)
- Services are initialized globally in `main.py:initialize_services()`
- Cache services are optional - the app gracefully degrades without them

## File Organization

```
app/
  main.py              # FastAPI app, endpoints, service initialization
  config.py            # Pydantic settings
  utils.py             # Validation, error handling utilities
  services/            # All business logic services
tests/
  test_storage_backends.py  # Storage backend tests
workflows/             # Architecture documentation (Mermaid diagrams)
data/
  sql/schema.sql       # Database schema
  generate_sample_data.py  # Sample data generator
```
