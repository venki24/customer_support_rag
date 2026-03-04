# Section 5: Environment Setup & Docker Compose

## Overview

This section provides a complete, production-ready local development environment using Docker Compose. You'll understand why containerization matters for RAG systems, what components are required, and how to configure everything for seamless development.

---

## Why: The Case for Containerized Development

### Multi-Service Complexity
A Customer Support RAG SaaS requires orchestrating 6+ interconnected services: FastAPI application, MongoDB (document storage), Redis (caching/queue), Qdrant (vector database), Celery worker (async processing), and Celery beat (scheduling). Managing these manually across different environments leads to "works on my machine" problems.

### Dependency Isolation
Vector databases like Qdrant have specific version requirements. Sentence transformers need CUDA libraries for GPU acceleration. Docker ensures every developer and CI/CD pipeline runs identical environments with pinned dependencies.

### Infrastructure Parity
Your production Kubernetes cluster uses containerized services. Docker Compose mirrors this architecture locally, reducing deployment surprises. Health checks, restart policies, and volume mounts in development translate directly to production manifests.

### Rapid Onboarding
New engineers run `docker-compose up` and have a fully functional RAG system in 5 minutes, including pre-configured databases, vector stores, and queue workers.

---

## What: Architecture Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Compose Network                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   FastAPI   │───▶│   MongoDB   │    │    Redis    │     │
│  │  (app:8000) │    │   (27017)   │    │   (6379)    │     │
│  └──────┬──────┘    └─────────────┘    └──────┬──────┘     │
│         │                                       │            │
│         │           ┌─────────────┐            │            │
│         ├──────────▶│   Qdrant    │            │            │
│         │           │   (6333)    │            │            │
│         │           └─────────────┘            │            │
│         │                                       │            │
│  ┌──────▼──────┐                        ┌──────▼──────┐    │
│  │   Celery    │◀───────────────────────│   Celery    │    │
│  │   Worker    │                        │    Beat     │    │
│  └─────────────┘                        └─────────────┘    │
│                                                               │
│  Volumes: mongodb_data, qdrant_data, redis_data             │
└─────────────────────────────────────────────────────────────┘
```

**Services Breakdown**:
1. **FastAPI App**: Main API server handling HTTP requests, authentication, document ingestion orchestration
2. **MongoDB**: Stores user accounts, workspaces, document metadata, conversation history
3. **Redis**: Session cache, Celery task queue/results, rate limiting counters
4. **Qdrant**: Vector database storing document embeddings (MiniLM-L6-v2, 384 dimensions)
5. **Celery Worker**: Background tasks (document parsing, embedding generation, batch processing)
6. **Celery Beat**: Scheduled jobs (workspace analytics, cleanup tasks, health monitoring)

---

## How: Complete Environment Setup

### 1. Docker Compose Configuration

**File: `docker-compose.yml`**

```yaml
version: '3.8'

services:
  # FastAPI Application
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    container_name: rag_saas_api
    ports:
      - "8000:8000"
    environment:
      - ENV=development
      - RELOAD=true
    env_file:
      - .env
    volumes:
      # Mount source code for hot reload
      - ./app:/app/app
      - ./tests:/app/tests
      # Prevent overwriting node_modules if needed
      - /app/__pycache__
    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_healthy
      qdrant:
        condition: service_started
    networks:
      - rag_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  # MongoDB Database
  mongodb:
    image: mongo:7.0
    container_name: rag_saas_mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD}
      MONGO_INITDB_DATABASE: ${MONGO_DBNAME}
    volumes:
      - mongodb_data:/data/db
      - mongodb_config:/data/configdb
      # Optional: initialization scripts
      - ./scripts/mongo-init.js:/docker-entrypoint-initdb.d/mongo-init.js:ro
    networks:
      - rag_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  # Redis Cache & Queue
  redis:
    image: redis:7.2-alpine
    container_name: rag_saas_redis
    ports:
      - "6379:6379"
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - rag_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 10s

  # Qdrant Vector Database
  qdrant:
    image: qdrant/qdrant:v1.7.4
    container_name: rag_saas_qdrant
    ports:
      - "6333:6333"  # HTTP API
      - "6334:6334"  # gRPC API (optional)
    environment:
      - QDRANT__SERVICE__GRPC_PORT=6334
      - QDRANT__SERVICE__HTTP_PORT=6333
      # Disable telemetry for local dev
      - QDRANT__TELEMETRY_DISABLED=true
    volumes:
      - qdrant_data:/qdrant/storage
    networks:
      - rag_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:6333/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s

  # Celery Worker
  celery_worker:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    container_name: rag_saas_celery_worker
    environment:
      - ENV=development
      - C_FORCE_ROOT=true  # Allow running as root (dev only)
    env_file:
      - .env
    volumes:
      - ./app:/app/app
      # Shared volume for temporary file processing
      - celery_temp:/tmp/celery
    depends_on:
      redis:
        condition: service_healthy
      mongodb:
        condition: service_healthy
      qdrant:
        condition: service_started
    networks:
      - rag_network
    restart: unless-stopped
    command: celery -A app.tasks.celery_app worker --loglevel=info --concurrency=4 --max-tasks-per-child=50

  # Celery Beat Scheduler
  celery_beat:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    container_name: rag_saas_celery_beat
    environment:
      - ENV=development
      - C_FORCE_ROOT=true
    env_file:
      - .env
    volumes:
      - ./app:/app/app
      # Persist schedule database
      - celery_beat_schedule:/app/celerybeat-schedule
    depends_on:
      redis:
        condition: service_healthy
    networks:
      - rag_network
    restart: unless-stopped
    command: celery -A app.tasks.celery_app beat --loglevel=info --scheduler=redis_scheduler.schedulers:RedisScheduler

volumes:
  mongodb_data:
    driver: local
  mongodb_config:
    driver: local
  redis_data:
    driver: local
  qdrant_data:
    driver: local
  celery_temp:
    driver: local
  celery_beat_schedule:
    driver: local

networks:
  rag_network:
    driver: bridge
```

### 2. Environment Variables Template

**File: `.env.example`**

```bash
# ============================================
# ENVIRONMENT
# ============================================
ENV=development
DEBUG=true
LOG_LEVEL=INFO
APP_NAME=CustomerSupportRAGSaaS
APP_VERSION=1.0.0

# ============================================
# API SERVER
# ============================================
HOST=0.0.0.0
PORT=8000
RELOAD=true
WORKERS=4

# ============================================
# MONGODB DATABASE
# ============================================
MONGO_HOST=mongodb
MONGO_PORT=27017
MONGO_USER=rag_admin
MONGO_PASSWORD=secure_mongo_password_change_in_production
MONGO_DBNAME=rag_saas_db
# Full connection string (overrides individual params if set)
MONGO_CONN_STRING=mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${MONGO_HOST}:${MONGO_PORT}/${MONGO_DBNAME}?authSource=admin

# ============================================
# REDIS CACHE & QUEUE
# ============================================
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=secure_redis_password_change_in_production
REDIS_DB=0
REDIS_MAX_CONNECTIONS=50
# Session expiry (seconds)
REDIS_SESSION_TTL=86400
# Cache expiry for embeddings (seconds)
REDIS_CACHE_TTL=3600

# ============================================
# QDRANT VECTOR DATABASE
# ============================================
QDRANT_HOST=qdrant
QDRANT_PORT=6333
QDRANT_GRPC_PORT=6334
# Set to empty for no API key in local dev
QDRANT_API_KEY=
QDRANT_COLLECTION_NAME=support_docs
# Vector dimensions for MiniLM-L6-v2
QDRANT_VECTOR_SIZE=384
# Distance metric: Cosine, Euclid, Dot
QDRANT_DISTANCE_METRIC=Cosine

# ============================================
# OPENAI API (GPT)
# ============================================
OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
# Model for answer generation
OPENAI_MODEL=gpt-4o-mini
# Model for query rewriting
OPENAI_REWRITE_MODEL=gpt-4o-mini
# Max tokens for generation
OPENAI_MAX_TOKENS=1500
OPENAI_TEMPERATURE=0.3
# Request timeout (seconds)
OPENAI_TIMEOUT=30
# Retry configuration
OPENAI_MAX_RETRIES=3

# ============================================
# EMBEDDING MODEL (Sentence Transformers)
# ============================================
EMBEDDING_MODEL_NAME=sentence-transformers/all-MiniLM-L6-v2
EMBEDDING_DEVICE=cpu
# Set to 'cuda' for GPU acceleration
# EMBEDDING_DEVICE=cuda
EMBEDDING_BATCH_SIZE=32
# Max sequence length for embeddings
EMBEDDING_MAX_LENGTH=512

# ============================================
# CELERY TASK QUEUE
# ============================================
CELERY_BROKER_URL=redis://:${REDIS_PASSWORD}@${REDIS_HOST}:${REDIS_PORT}/1
CELERY_RESULT_BACKEND=redis://:${REDIS_PASSWORD}@${REDIS_HOST}:${REDIS_PORT}/2
CELERY_TASK_SERIALIZER=json
CELERY_RESULT_SERIALIZER=json
CELERY_ACCEPT_CONTENT=json
# Task time limits (seconds)
CELERY_TASK_TIME_LIMIT=600
CELERY_TASK_SOFT_TIME_LIMIT=540
# Worker concurrency
CELERY_WORKER_CONCURRENCY=4
CELERY_WORKER_MAX_TASKS_PER_CHILD=50

# ============================================
# JWT AUTHENTICATION
# ============================================
JWT_SECRET_KEY=super_secret_jwt_key_min_32_chars_change_in_production_use_secrets_generator
JWT_ALGORITHM=HS256
# Access token expiry (minutes)
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=60
# Refresh token expiry (days)
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

# ============================================
# RAG CONFIGURATION
# ============================================
# Number of chunks to retrieve
RAG_TOP_K=5
# Minimum similarity score (0-1)
RAG_SCORE_THRESHOLD=0.7
# Chunk size for document splitting (characters)
CHUNK_SIZE=800
CHUNK_OVERLAP=200
# Enable hybrid search (keyword + vector)
ENABLE_HYBRID_SEARCH=true
# Reranking model (optional)
RERANKER_MODEL=
# Enable query rewriting
ENABLE_QUERY_REWRITE=true

# ============================================
# FILE PROCESSING
# ============================================
# Max file size for upload (bytes) - 10MB
MAX_FILE_SIZE=10485760
# Allowed file extensions
ALLOWED_EXTENSIONS=pdf,txt,docx,md,html
# Temporary file storage path
TEMP_FILE_PATH=/tmp/celery

# ============================================
# RATE LIMITING
# ============================================
# Requests per minute per user
RATE_LIMIT_PER_MINUTE=60
# Requests per hour per workspace
WORKSPACE_RATE_LIMIT_PER_HOUR=1000

# ============================================
# CORS CONFIGURATION
# ============================================
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://localhost:8000
CORS_ALLOW_CREDENTIALS=true
CORS_ALLOWED_METHODS=GET,POST,PUT,DELETE,PATCH
CORS_ALLOWED_HEADERS=*

# ============================================
# MONITORING & OBSERVABILITY
# ============================================
# Enable Sentry error tracking
ENABLE_SENTRY=false
SENTRY_DSN=
# Enable metrics export
ENABLE_METRICS=false
PROMETHEUS_PORT=9090
```

### 3. Python Dependencies

**File: `requirements.txt`**

```txt
# ============================================
# FASTAPI & WEB SERVER
# ============================================
fastapi==0.109.0
uvicorn[standard]==0.27.0
python-multipart==0.0.6
httpx==0.26.0

# ============================================
# DATABASE & ORM
# ============================================
# MongoDB async driver
motor==3.3.2
# Beanie ODM for MongoDB
beanie==1.24.0
# Pydantic for data validation
pydantic==2.5.3
pydantic-settings==2.1.0

# ============================================
# REDIS & CACHING
# ============================================
redis==5.0.1
hiredis==2.3.2

# ============================================
# CELERY TASK QUEUE
# ============================================
celery==5.3.6
celery-redbeat==2.1.1
# Flower for Celery monitoring (optional)
flower==2.0.1

# ============================================
# VECTOR DATABASE
# ============================================
qdrant-client==1.7.3

# ============================================
# AI & EMBEDDINGS
# ============================================
# OpenAI GPT API
openai==1.10.0
# Sentence transformers for embeddings
sentence-transformers==2.3.1
torch==2.1.2
# For CPU-only (smaller): torch==2.1.2+cpu
transformers==4.37.1

# ============================================
# DOCUMENT PROCESSING
# ============================================
# PDF parsing
pypdf2==3.0.1
pdfplumber==0.10.4
# Word documents
python-docx==1.1.0
# HTML parsing
beautifulsoup4==4.12.3
lxml==5.1.0
# Markdown
markdown==3.5.2
# Text splitting
langchain-text-splitters==0.0.1

# ============================================
# AUTHENTICATION & SECURITY
# ============================================
pyjwt[crypto]==2.8.0
passlib[bcrypt]==1.7.4
python-jose[cryptography]==3.3.0
cryptography==42.0.0

# ============================================
# UTILITIES
# ============================================
# Environment variable loading
python-dotenv==1.0.0
# Logging
loguru==0.7.2
# HTTP requests
requests==2.31.0
# Date/time utilities
python-dateutil==2.8.2
# UUID generation
uuid==1.30
# JSON handling
orjson==3.9.12

# ============================================
# DEVELOPMENT & TESTING
# ============================================
pytest==7.4.4
pytest-asyncio==0.23.3
pytest-mock==3.12.0
pytest-cov==4.1.0
httpx==0.26.0
mongomock==4.1.2
fakeredis==2.21.1

# ============================================
# CODE QUALITY
# ============================================
black==24.1.1
ruff==0.1.14
mypy==1.8.0
pre-commit==3.6.0

# ============================================
# MONITORING (OPTIONAL)
# ============================================
# sentry-sdk==1.40.0
# prometheus-client==0.19.0
```

### 4. Multi-Stage Dockerfile

**File: `Dockerfile`**

```dockerfile
# ============================================
# Stage 1: Base Image with Dependencies
# ============================================
FROM python:3.11-slim as base

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    gcc \
    g++ \
    git \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# ============================================
# Stage 2: Dependencies Installation
# ============================================
FROM base as dependencies

# Copy requirements first (better caching)
COPY requirements.txt .

# Install Python dependencies
RUN pip install --upgrade pip setuptools wheel && \
    pip install -r requirements.txt

# ============================================
# Stage 3: Development Image
# ============================================
FROM dependencies as development

# Copy application code
COPY ./app /app/app

# Create non-root user for development
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Default command (can be overridden in docker-compose)
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# ============================================
# Stage 4: Production Image (Optimized)
# ============================================
FROM dependencies as production

# Copy only necessary application code
COPY ./app /app/app

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Production command with multiple workers
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### 5. Configuration Management with Pydantic

**File: `app/config.py`**

```python
"""
Configuration management using Pydantic Settings.

This module loads configuration from environment variables with:
- Type validation and coercion
- Default values
- Computed fields
- Environment-specific settings
"""

from functools import lru_cache
from typing import Literal, Optional
from pydantic import Field, field_validator, computed_field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """
    Application settings with environment variable loading.

    Loads from .env file in development, environment variables in production.
    All settings have type validation and sensible defaults.
    """

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore"
    )

    # ========================================
    # ENVIRONMENT
    # ========================================
    env: Literal["development", "staging", "production"] = Field(
        default="development",
        description="Application environment"
    )
    debug: bool = Field(default=False, description="Debug mode")
    log_level: Literal["DEBUG", "INFO", "WARNING", "ERROR"] = Field(
        default="INFO",
        description="Logging level"
    )
    app_name: str = Field(default="CustomerSupportRAGSaaS", description="Application name")
    app_version: str = Field(default="1.0.0", description="Application version")

    # ========================================
    # API SERVER
    # ========================================
    host: str = Field(default="0.0.0.0", description="API host")
    port: int = Field(default=8000, description="API port")
    reload: bool = Field(default=False, description="Auto-reload on code changes")
    workers: int = Field(default=4, description="Number of uvicorn workers")

    # ========================================
    # MONGODB
    # ========================================
    mongo_host: str = Field(default="localhost", description="MongoDB host")
    mongo_port: int = Field(default=27017, description="MongoDB port")
    mongo_user: str = Field(default="admin", description="MongoDB username")
    mongo_password: str = Field(default="password", description="MongoDB password")
    mongo_dbname: str = Field(default="rag_saas_db", description="MongoDB database name")
    mongo_conn_string: Optional[str] = Field(default=None, description="Full MongoDB URI")

    @computed_field
    @property
    def mongodb_url(self) -> str:
        """Construct MongoDB connection URL."""
        if self.mongo_conn_string:
            return self.mongo_conn_string
        return (
            f"mongodb://{self.mongo_user}:{self.mongo_password}"
            f"@{self.mongo_host}:{self.mongo_port}/{self.mongo_dbname}"
            f"?authSource=admin"
        )

    # ========================================
    # REDIS
    # ========================================
    redis_host: str = Field(default="localhost", description="Redis host")
    redis_port: int = Field(default=6379, description="Redis port")
    redis_password: str = Field(default="", description="Redis password")
    redis_db: int = Field(default=0, description="Redis database number")
    redis_max_connections: int = Field(default=50, description="Redis connection pool size")
    redis_session_ttl: int = Field(default=86400, description="Session TTL (seconds)")
    redis_cache_ttl: int = Field(default=3600, description="Cache TTL (seconds)")

    @computed_field
    @property
    def redis_url(self) -> str:
        """Construct Redis connection URL."""
        if self.redis_password:
            return f"redis://:{self.redis_password}@{self.redis_host}:{self.redis_port}/{self.redis_db}"
        return f"redis://{self.redis_host}:{self.redis_port}/{self.redis_db}"

    # ========================================
    # QDRANT
    # ========================================
    qdrant_host: str = Field(default="localhost", description="Qdrant host")
    qdrant_port: int = Field(default=6333, description="Qdrant HTTP port")
    qdrant_grpc_port: int = Field(default=6334, description="Qdrant gRPC port")
    qdrant_api_key: Optional[str] = Field(default=None, description="Qdrant API key")
    qdrant_collection_name: str = Field(default="support_docs", description="Qdrant collection")
    qdrant_vector_size: int = Field(default=384, description="Vector dimensions")
    qdrant_distance_metric: Literal["Cosine", "Euclid", "Dot"] = Field(
        default="Cosine",
        description="Distance metric"
    )

    @computed_field
    @property
    def qdrant_url(self) -> str:
        """Construct Qdrant connection URL."""
        return f"http://{self.qdrant_host}:{self.qdrant_port}"

    # ========================================
    # OPENAI
    # ========================================
    openai_api_key: str = Field(..., description="OpenAI API key (required)")
    openai_model: str = Field(default="gpt-4o-mini", description="GPT model for answers")
    openai_rewrite_model: str = Field(default="gpt-4o-mini", description="GPT model for queries")
    openai_max_tokens: int = Field(default=1500, description="Max tokens per response")
    openai_temperature: float = Field(default=0.3, ge=0, le=2, description="Generation temperature")
    openai_timeout: int = Field(default=30, description="Request timeout (seconds)")
    openai_max_retries: int = Field(default=3, description="Max retry attempts")

    @field_validator("openai_temperature")
    @classmethod
    def validate_temperature(cls, v: float) -> float:
        """Ensure temperature is in valid range."""
        if not 0 <= v <= 2:
            raise ValueError("Temperature must be between 0 and 2")
        return v

    # ========================================
    # EMBEDDING MODEL
    # ========================================
    embedding_model_name: str = Field(
        default="sentence-transformers/all-MiniLM-L6-v2",
        description="Sentence transformer model"
    )
    embedding_device: Literal["cpu", "cuda"] = Field(default="cpu", description="Device for embeddings")
    embedding_batch_size: int = Field(default=32, description="Batch size for embeddings")
    embedding_max_length: int = Field(default=512, description="Max sequence length")

    # ========================================
    # CELERY
    # ========================================
    celery_broker_url: Optional[str] = Field(default=None, description="Celery broker URL")
    celery_result_backend: Optional[str] = Field(default=None, description="Celery result backend")
    celery_task_serializer: str = Field(default="json", description="Task serializer")
    celery_result_serializer: str = Field(default="json", description="Result serializer")
    celery_accept_content: list[str] = Field(default=["json"], description="Accepted content types")
    celery_task_time_limit: int = Field(default=600, description="Task hard timeout (seconds)")
    celery_task_soft_time_limit: int = Field(default=540, description="Task soft timeout (seconds)")
    celery_worker_concurrency: int = Field(default=4, description="Worker concurrency")
    celery_worker_max_tasks_per_child: int = Field(default=50, description="Tasks before worker restart")

    @computed_field
    @property
    def celery_broker_computed(self) -> str:
        """Compute Celery broker URL if not provided."""
        if self.celery_broker_url:
            return self.celery_broker_url
        # Use Redis DB 1 for broker
        return f"redis://:{self.redis_password}@{self.redis_host}:{self.redis_port}/1"

    @computed_field
    @property
    def celery_backend_computed(self) -> str:
        """Compute Celery result backend if not provided."""
        if self.celery_result_backend:
            return self.celery_result_backend
        # Use Redis DB 2 for results
        return f"redis://:{self.redis_password}@{self.redis_host}:{self.redis_port}/2"

    # ========================================
    # JWT AUTHENTICATION
    # ========================================
    jwt_secret_key: str = Field(..., description="JWT secret key (required)")
    jwt_algorithm: str = Field(default="HS256", description="JWT algorithm")
    jwt_access_token_expire_minutes: int = Field(default=60, description="Access token expiry")
    jwt_refresh_token_expire_days: int = Field(default=7, description="Refresh token expiry")

    @field_validator("jwt_secret_key")
    @classmethod
    def validate_jwt_secret(cls, v: str) -> str:
        """Ensure JWT secret is sufficiently long."""
        if len(v) < 32:
            raise ValueError("JWT secret must be at least 32 characters")
        return v

    # ========================================
    # RAG CONFIGURATION
    # ========================================
    rag_top_k: int = Field(default=5, ge=1, le=20, description="Number of chunks to retrieve")
    rag_score_threshold: float = Field(default=0.7, ge=0, le=1, description="Minimum similarity")
    chunk_size: int = Field(default=800, description="Chunk size (characters)")
    chunk_overlap: int = Field(default=200, description="Chunk overlap (characters)")
    enable_hybrid_search: bool = Field(default=True, description="Enable hybrid search")
    reranker_model: Optional[str] = Field(default=None, description="Reranking model")
    enable_query_rewrite: bool = Field(default=True, description="Enable query rewriting")

    # ========================================
    # FILE PROCESSING
    # ========================================
    max_file_size: int = Field(default=10485760, description="Max file size (bytes)")
    allowed_extensions: list[str] = Field(
        default=["pdf", "txt", "docx", "md", "html"],
        description="Allowed file types"
    )
    temp_file_path: str = Field(default="/tmp/celery", description="Temp file storage")

    # ========================================
    # RATE LIMITING
    # ========================================
    rate_limit_per_minute: int = Field(default=60, description="Requests per minute per user")
    workspace_rate_limit_per_hour: int = Field(default=1000, description="Requests per hour per workspace")

    # ========================================
    # CORS
    # ========================================
    cors_allowed_origins: list[str] = Field(
        default=["http://localhost:3000"],
        description="Allowed CORS origins"
    )
    cors_allow_credentials: bool = Field(default=True, description="Allow credentials")
    cors_allowed_methods: list[str] = Field(
        default=["GET", "POST", "PUT", "DELETE", "PATCH"],
        description="Allowed HTTP methods"
    )
    cors_allowed_headers: list[str] = Field(default=["*"], description="Allowed headers")

    # ========================================
    # MONITORING
    # ========================================
    enable_sentry: bool = Field(default=False, description="Enable Sentry")
    sentry_dsn: Optional[str] = Field(default=None, description="Sentry DSN")
    enable_metrics: bool = Field(default=False, description="Enable Prometheus metrics")
    prometheus_port: int = Field(default=9090, description="Prometheus port")

    # ========================================
    # COMPUTED PROPERTIES
    # ========================================
    @computed_field
    @property
    def is_production(self) -> bool:
        """Check if running in production."""
        return self.env == "production"

    @computed_field
    @property
    def is_development(self) -> bool:
        """Check if running in development."""
        return self.env == "development"


@lru_cache()
def get_settings() -> Settings:
    """
    Get cached settings instance.

    Uses lru_cache to ensure settings are loaded once and reused.
    Perfect for FastAPI dependency injection.

    Returns:
        Settings: Validated settings instance
    """
    return Settings()


# Convenience export
settings = get_settings()
```

### 6. Local Development Setup Guide

#### Step 1: Clone & Environment Setup

```bash
# Clone repository
git clone https://github.com/your-org/customer-support-rag-saas.git
cd customer-support-rag-saas

# Create .env from template
cp .env.example .env

# Edit .env with your values (CRITICAL: Set OPENAI_API_KEY and JWT_SECRET_KEY)
nano .env
```

#### Step 2: Start All Services

```bash
# Build and start all containers
docker-compose up --build

# Or run in detached mode
docker-compose up -d --build

# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f app
docker-compose logs -f celery_worker
```

#### Step 3: Verify Services

```bash
# Check all services are healthy
docker-compose ps

# Test API health
curl http://localhost:8000/health

# Test Qdrant
curl http://localhost:6333/health

# Test MongoDB
docker-compose exec mongodb mongosh --eval "db.adminCommand('ping')"

# Test Redis
docker-compose exec redis redis-cli -a your_password ping
```

#### Step 4: Initialize Database & Collections

```bash
# Run database initialization script
docker-compose exec app python -m app.scripts.init_db

# Create Qdrant collection
docker-compose exec app python -m app.scripts.init_qdrant

# Create admin user (optional)
docker-compose exec app python -m app.scripts.create_admin_user
```

#### Step 5: Run Tests

```bash
# Run full test suite
docker-compose exec app pytest

# Run with coverage
docker-compose exec app pytest --cov=app --cov-report=html

# Run specific test file
docker-compose exec app pytest tests/test_embedding_service.py -v
```

#### Step 6: Development Workflow

```bash
# Hot reload is enabled - just edit code and save
# Logs will show reload events

# Access Celery Flower monitoring (optional)
docker-compose exec celery_worker celery -A app.tasks.celery_app flower --port=5555
# Visit http://localhost:5555

# Access MongoDB with GUI (optional - install MongoDB Compass)
# Connection string: mongodb://rag_admin:your_password@localhost:27017/rag_saas_db

# Rebuild specific service
docker-compose up -d --build app

# Stop all services
docker-compose down

# Stop and remove volumes (CAUTION: deletes data)
docker-compose down -v
```

#### Common Development Commands

```bash
# Execute command in app container
docker-compose exec app python -m app.scripts.seed_data

# Run Python shell with app context
docker-compose exec app python

# Restart Celery worker (after task changes)
docker-compose restart celery_worker

# View resource usage
docker stats

# Clean up unused Docker resources
docker system prune -a --volumes
```

---

## Interview Q&A

### Q1: Why use Docker Compose for local development instead of installing services natively?

**Answer**: Docker Compose solves the "dependency hell" problem inherent in multi-service RAG architectures. Here's why it's superior for local development:

**Service Orchestration**: A RAG SaaS requires 6+ interconnected services (FastAPI, MongoDB, Redis, Qdrant, Celery worker, Celery beat). Docker Compose defines all dependencies, startup order (via `depends_on` with health checks), and networking in one declarative file. Without it, developers manually start services, handle port conflicts, and debug connectivity issues.

**Environment Consistency**: Qdrant 1.7.4 works differently than 1.6.x. MongoDB 7.0 has different authentication than 5.x. Docker pins exact versions (`image: mongo:7.0`, `image: qdrant/qdrant:v1.7.4`), ensuring your local environment matches staging and production. This eliminates "works on my machine" bugs.

**Onboarding Speed**: New engineers run `docker-compose up` and have a working RAG system in 5 minutes. Without Docker, they'd spend hours installing MongoDB, configuring Redis, compiling sentence transformers with correct CUDA libraries, and debugging path issues.

**Isolation**: Docker networks (`rag_network`) isolate services. You can run multiple projects simultaneously without port conflicts (map different host ports). Native installations pollute your system and cause conflicts.

**Production Parity**: Your production Kubernetes cluster uses containers. Docker Compose mirrors this architecture locally. Health checks (`healthcheck`), restart policies (`restart: unless-stopped`), and resource limits translate directly to production manifests.

**Trade-offs**: Docker adds ~500MB overhead per project and slower filesystem I/O on macOS/Windows (solved with named volumes for data). But for multi-service systems, the benefits vastly outweigh costs.

---

### Q2: How do you manage configuration across development, staging, and production environments?

**Answer**: Configuration management is critical for RAG systems due to sensitive credentials (OpenAI API keys, JWT secrets) and environment-specific settings (GPU vs CPU embeddings, connection strings). Here's a robust strategy:

**Pydantic Settings with Environment Variables**: Use `pydantic-settings` BaseSettings class (see `config.py`). It loads from environment variables with type validation, defaults, and computed fields. Benefits:
- **Type Safety**: `port: int` ensures you don't accidentally pass a string. `temperature: float = Field(ge=0, le=2)` validates ranges.
- **Computed Fields**: `@computed_field` auto-generates `mongodb_url` from individual `MONGO_HOST`, `MONGO_PORT`, etc., reducing duplication.
- **Validation**: `@field_validator` ensures JWT secrets are ≥32 chars, preventing weak security in production.
- **Caching**: `@lru_cache()` loads settings once, avoiding repeated file I/O.

**Environment-Specific .env Files**:
- Development: `.env` with local service hostnames (`MONGO_HOST=mongodb`, `REDIS_HOST=redis` - Docker Compose service names)
- Staging/Production: Environment variables injected by Kubernetes secrets/ConfigMaps. **Never commit .env to Git**.
- Template: `.env.example` documents all required variables without sensitive values.

**Secrets Management**:
- Development: `.env` (gitignored)
- Production: Kubernetes secrets or AWS Secrets Manager. Fetch at runtime with `boto3`:
  ```python
  if settings.is_production:
      settings.openai_api_key = get_secret("prod/openai-key")
  ```

**Environment Detection**: `ENV=development|staging|production` toggles behaviors:
- Development: `DEBUG=true`, hot reload, verbose logging
- Production: `DEBUG=false`, multiple workers, Sentry error tracking

**Configuration Validation**: At startup, `Settings()` raises `ValidationError` if required fields are missing (e.g., `OPENAI_API_KEY`). This fails fast instead of cryptic runtime errors.

**Example**: `EMBEDDING_DEVICE=cpu` in development (no GPU), `EMBEDDING_DEVICE=cuda` in production (GPU acceleration). Pydantic validates it's a valid Literal type.

**Anti-patterns to Avoid**:
- Hard-coding credentials in code
- Using `config.json` (merge conflict hell with teams)
- Different config loading logic across environments

---

### Q3: Explain the health check configuration in docker-compose.yml. Why is it important for a RAG system?

**Answer**: Health checks (`healthcheck`) in Docker Compose determine when a service is truly ready, not just running. For RAG systems with complex startup dependencies, they're critical:

**Dependency Ordering**: `depends_on` with `condition: service_healthy` ensures the FastAPI app doesn't start until MongoDB, Redis, and Qdrant are operational. Without health checks, the app crashes trying to connect to MongoDB that's still initializing.

**Example - MongoDB Health Check**:
```yaml
healthcheck:
  test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
  interval: 10s       # Check every 10 seconds
  timeout: 5s         # Fail if check takes >5s
  retries: 5          # Try 5 times before marking unhealthy
  start_period: 20s   # Grace period (don't fail during first 20s)
```
This executes `mongosh --eval "db.adminCommand('ping')"` inside the container. MongoDB returns a successful response only after authentication is ready and databases are mounted.

**Why It Matters for RAG**:
1. **Vector Database Initialization**: Qdrant takes 10-30s to load collections from disk. If the app queries Qdrant during initialization, you get cryptic 500 errors. Health check (`curl http://localhost:6333/health`) ensures Qdrant's HTTP API is responsive.

2. **Embedding Service Warm-up**: Sentence transformers load a ~400MB model on first use. If a user query hits the app before the model is loaded, the request times out. A custom health check can verify the model is ready:
   ```python
   @app.get("/health")
   async def health_check():
       await embedding_service.get_embeddings(["test"])  # Triggers model load
       return {"status": "healthy"}
   ```

3. **Celery Worker Readiness**: The worker must connect to Redis (queue) and Qdrant (embedding storage) before processing tasks. Without health checks, tasks fail immediately on startup.

**Orchestration Flow**:
1. `docker-compose up` starts all services in parallel
2. MongoDB health check passes after 20s
3. Redis health check passes after 10s
4. Qdrant starts (no health check, uses `service_started`)
5. FastAPI app starts only after MongoDB and Redis are healthy
6. Celery worker starts after Redis and MongoDB are healthy

**Monitoring**: `docker-compose ps` shows health status. Production Kubernetes uses liveness/readiness probes with identical logic.

**Best Practices**:
- Use lightweight checks (avoid heavy queries)
- Set realistic `start_period` (don't fail during initialization)
- Log check failures (`curl -f` returns non-zero on failure, visible in logs)

---

### Q4: How do you handle Celery task development and debugging in Docker Compose?

**Answer**: Celery debugging is notoriously tricky because tasks run in background workers with separate logs. Docker Compose requires specific strategies:

**Log Aggregation**:
```bash
# View worker logs in real-time
docker-compose logs -f celery_worker

# View both app and worker logs
docker-compose logs -f app celery_worker

# Follow all logs with timestamps
docker-compose logs -f --timestamps
```

**Task Visibility with Flower**: Add Celery Flower as a service for web-based task monitoring:
```yaml
celery_flower:
  build: .
  command: celery -A app.tasks.celery_app flower --port=5555
  ports:
    - "5555:5555"
  depends_on:
    - redis
```
Access `http://localhost:5555` to see task queue, active tasks, success/failure rates, and full tracebacks.

**Shared Volumes for File Processing**: Document parsing tasks write temporary files. Share volumes between app and worker:
```yaml
volumes:
  - celery_temp:/tmp/celery
```
Both services access the same filesystem path, preventing "file not found" errors.

**Interactive Debugging**: Celery doesn't support `breakpoint()` because it's a daemon. Instead:
1. **Synchronous Testing**: Call tasks directly without `.delay()`:
   ```python
   # In tests or shell
   result = embed_document_task(doc_id="test123")  # Synchronous
   ```
2. **Celery Shell**: Execute tasks with `docker-compose exec`:
   ```bash
   docker-compose exec celery_worker python
   >>> from app.tasks import embed_document_task
   >>> embed_document_task.delay("doc123")
   <AsyncResult: task-id>
   ```

**Task Retry Configuration**: RAG tasks often fail (OpenAI rate limits, transient Qdrant errors). Configure retries in task definition:
```python
@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def embed_document_task(self, doc_id: str):
    try:
        # Generate embeddings
        pass
    except Exception as exc:
        # Exponential backoff
        self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
```

**Hot Reload for Task Changes**: Celery workers don't auto-reload. After editing task code:
```bash
docker-compose restart celery_worker
```
For frequent changes, reduce `--max-tasks-per-child=5` (worker restarts after 5 tasks, picking up new code).

**Environment Consistency**: Both app and worker must use identical code. Mount source code in both services:
```yaml
app:
  volumes:
    - ./app:/app/app
celery_worker:
  volumes:
    - ./app:/app/app  # Same mount
```

**Debugging Serialization Errors**: Celery serializes arguments to JSON. Complex objects (Pydantic models) fail. Use:
```python
# Convert Pydantic model to dict
task.delay(**document.model_dump())
```

**Task Timeout Handling**: Embedding large PDFs can exceed task time limits. Configure in `.env`:
```bash
CELERY_TASK_TIME_LIMIT=600        # Hard kill after 10 minutes
CELERY_TASK_SOFT_TIME_LIMIT=540   # Warning at 9 minutes
```
Tasks receive `SoftTimeLimitExceeded` exception to cleanup gracefully before hard kill.

**Redis Inspection**: Check task queue status:
```bash
docker-compose exec redis redis-cli -a your_password
> KEYS celery*                  # View Celery keys
> LLEN celery                   # Queue length
> LRANGE celery 0 10            # View queued tasks
```

---

### Q5: What are the security considerations for this docker-compose.yml configuration?

**Answer**: The provided `docker-compose.yml` is optimized for local development. Production deployments require hardening:

**Secrets Management**:
- **Issue**: `.env` file contains plaintext passwords (`MONGO_PASSWORD`, `REDIS_PASSWORD`, `OPENAI_API_KEY`)
- **Fix**: Use Docker secrets or Kubernetes secrets. Never commit `.env`:
  ```bash
  echo ".env" >> .gitignore
  ```
- **Production**: Mount secrets from external vault:
  ```yaml
  secrets:
    mongo_password:
      external: true
  services:
    mongodb:
      secrets:
        - mongo_password
  ```

**Non-Root Users**: The Dockerfile creates `appuser` and runs as UID 1000. This prevents privilege escalation if the container is compromised. **Never run containers as root in production**.

**Network Isolation**:
- **Current**: All services on `rag_network` bridge network, can communicate freely
- **Improved**: Separate networks for different security zones:
  ```yaml
  networks:
    frontend:  # Only app service
    backend:   # App, MongoDB, Redis, Qdrant
  app:
    networks:
      - frontend
      - backend
  mongodb:
    networks:
      - backend  # Not exposed to frontend
  ```

**Port Exposure**:
- **Issue**: `ports: - "27017:27017"` exposes MongoDB to host. Attackers on your network can access the database.
- **Fix**: Only expose services that need external access (app:8000). Remove port mappings for MongoDB, Redis, Qdrant (they communicate via internal Docker network).

**Resource Limits**: Prevent DOS attacks by limiting resources:
```yaml
app:
  deploy:
    resources:
      limits:
        cpus: '2.0'
        memory: 2G
      reservations:
        cpus: '0.5'
        memory: 512M
```

**Image Vulnerabilities**: Use specific tags (`mongo:7.0`, not `mongo:latest`) and scan for CVEs:
```bash
docker scan rag_saas_api:latest
```

**CORS Configuration**: `.env` allows `http://localhost:3000`. Production should restrict to your actual frontend domain:
```bash
CORS_ALLOWED_ORIGINS=https://app.yourdomain.com
```

**Redis Authentication**: Current setup uses `--requirepass`. Production should use ACLs (Access Control Lists) with role-based permissions:
```bash
# Redis 6+ ACLs
redis-cli ACL SETUSER celery_user on >celery_password ~celery* +@all
```

**TLS/SSL**: Internal service communication is unencrypted. Production should use TLS:
- MongoDB: Enable TLS with certificates
- Redis: `redis-server --tls-port 6379 --tls-cert-file cert.pem`
- Qdrant: Configure HTTPS

**Environment Variable Injection**: Using `env_file: .env` exposes all variables to containers. Follow least privilege - only inject necessary variables per service.

**Health Check Security**: The MongoDB health check uses `mongosh --eval "db.adminCommand('ping')"`. Ensure the container's user has minimal permissions (not `admin` role).

**Logging Sensitive Data**: Ensure Celery and FastAPI logs don't leak API keys or user data. Configure loguru with redaction:
```python
logger.add("app.log", filter=lambda r: redact_secrets(r))
```

**Development vs Production**: This docker-compose.yml is for **local development only**. Production should use:
- Kubernetes with secrets management
- Managed services (MongoDB Atlas, Redis Cloud)
- API gateways with authentication
- Network policies isolating pods

