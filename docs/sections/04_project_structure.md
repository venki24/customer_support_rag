# Section 4: Project Structure & Setup

## Overview

This section provides a complete walkthrough of how to structure a production-ready FastAPI application for a Customer Support RAG SaaS. We cover the directory layout, key files, configuration management, dependency injection, and development workflow.

**WHY this structure matters:**
- Clear separation of concerns makes the codebase maintainable as it grows
- Modular design enables independent testing and development of features
- Configuration management prevents hardcoded values and supports multiple environments
- Dependency injection makes code testable and loosely coupled

---

## Complete Directory Structure

```
customer-support-rag/
│
├── app/                              # Main application package
│   ├── __init__.py
│   ├── main.py                       # FastAPI application entry point
│   ├── config.py                     # Pydantic settings for configuration
│   ├── dependencies.py               # Dependency injection setup
│   │
│   ├── middleware/                   # Custom middleware components
│   │   ├── __init__.py
│   │   ├── auth.py                   # JWT authentication middleware
│   │   ├── tenant.py                 # Tenant isolation middleware
│   │   ├── rate_limit.py             # Rate limiting middleware
│   │   └── logging.py                # Request/response logging
│   │
│   ├── models/                       # Database models (MongoDB/Beanie)
│   │   ├── __init__.py
│   │   ├── organization.py           # Organization/tenant entity
│   │   ├── user.py                   # User and authentication
│   │   ├── assistant.py              # AI assistant configuration
│   │   ├── knowledge.py              # Knowledge base documents
│   │   ├── conversation.py           # Chat conversations and messages
│   │   ├── ticket.py                 # Support ticket integration
│   │   └── analytics.py              # Usage analytics and metrics
│   │
│   ├── schemas/                      # Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── auth.py                   # Login, token, registration schemas
│   │   ├── chat.py                   # Chat request/response schemas
│   │   ├── knowledge.py              # Knowledge CRUD schemas
│   │   ├── assistant.py              # Assistant configuration schemas
│   │   └── common.py                 # Shared schemas (pagination, errors)
│   │
│   ├── routes/                       # API route handlers
│   │   ├── __init__.py
│   │   ├── auth.py                   # POST /auth/login, /auth/register
│   │   ├── chat.py                   # POST /chat, GET /chat/{id}
│   │   ├── knowledge.py              # CRUD for knowledge bases
│   │   ├── assistant.py              # CRUD for assistants
│   │   ├── tickets.py                # Ticket management endpoints
│   │   ├── analytics.py              # Analytics and reporting
│   │   └── health.py                 # Health check endpoints
│   │
│   ├── services/                     # Business logic layer
│   │   ├── __init__.py
│   │   ├── auth_service.py           # User authentication and JWT
│   │   ├── chat_service.py           # Main chat orchestration
│   │   ├── knowledge_service.py      # Knowledge management
│   │   ├── assistant_service.py      # Assistant configuration
│   │   │
│   │   ├── ingestion/                # Knowledge ingestion pipeline
│   │   │   ├── __init__.py
│   │   │   ├── ingestion_service.py  # Main ingestion orchestrator
│   │   │   ├── strategies.py         # Strategy pattern for sources
│   │   │   ├── parsers.py            # Content parsers (PDF, HTML, etc.)
│   │   │   └── validators.py         # Content validation
│   │   │
│   │   ├── retrieval/                # Retrieval pipeline
│   │   │   ├── __init__.py
│   │   │   ├── retrieval_pipeline.py # Main retrieval orchestrator
│   │   │   ├── embeddings.py         # Embedding generation
│   │   │   ├── vector_search.py      # Qdrant vector search
│   │   │   └── reranker.py           # Result re-ranking
│   │   │
│   │   └── generation/               # Response generation pipeline
│   │       ├── __init__.py
│   │       ├── generation_pipeline.py # Main generation orchestrator
│   │       ├── prompt_builder.py     # Prompt construction
│   │       ├── gpt_client.py         # OpenAI GPT client
│   │       └── circuit_breaker.py    # Circuit breaker implementation
│   │
│   ├── repositories/                 # Data access layer (Repository pattern)
│   │   ├── __init__.py
│   │   ├── organization_repo.py
│   │   ├── user_repo.py
│   │   ├── assistant_repo.py
│   │   ├── knowledge_repo.py
│   │   ├── conversation_repo.py
│   │   └── vector_repo.py            # Qdrant operations
│   │
│   ├── workers/                      # Celery background workers
│   │   ├── __init__.py
│   │   ├── celery_app.py             # Celery configuration
│   │   ├── ingestion_tasks.py        # Document ingestion tasks
│   │   ├── scheduled_tasks.py        # Periodic tasks (analytics, cleanup)
│   │   └── webhook_tasks.py          # External webhook processing
│   │
│   ├── utils/                        # Utility functions and helpers
│   │   ├── __init__.py
│   │   ├── chunking.py               # Text chunking strategies
│   │   ├── text_cleaner.py           # Text preprocessing
│   │   ├── token_counter.py          # Token counting for LLMs
│   │   ├── embeddings.py             # Embedding utilities
│   │   ├── cache.py                  # Redis caching helpers
│   │   └── logger.py                 # Structured logging setup
│   │
│   └── exceptions/                   # Custom exception classes
│       ├── __init__.py
│       ├── auth_exceptions.py
│       ├── knowledge_exceptions.py
│       └── rag_exceptions.py
│
├── tests/                            # Test suite
│   ├── __init__.py
│   ├── conftest.py                   # Pytest fixtures and configuration
│   ├── unit/                         # Unit tests
│   │   ├── services/
│   │   ├── repositories/
│   │   └── utils/
│   ├── integration/                  # Integration tests
│   │   ├── test_chat_flow.py
│   │   ├── test_ingestion_flow.py
│   │   └── test_auth_flow.py
│   └── e2e/                          # End-to-end tests
│       └── test_full_workflow.py
│
├── scripts/                          # Utility scripts
│   ├── init_db.py                    # Initialize database and collections
│   ├── seed_data.py                  # Seed test data
│   ├── migrate.py                    # Database migrations
│   └── backup.py                     # Backup utilities
│
├── docker/                           # Docker configuration
│   ├── Dockerfile                    # Main application Dockerfile
│   ├── Dockerfile.worker             # Celery worker Dockerfile
│   └── nginx.conf                    # Nginx reverse proxy config
│
├── alembic/                          # Database migrations (if using SQL)
│   ├── versions/
│   └── env.py
│
├── .env.example                      # Environment variable template
├── .env                              # Actual environment variables (gitignored)
├── .gitignore
├── docker-compose.yml                # Local development environment
├── docker-compose.prod.yml           # Production environment
├── requirements.txt                  # Python dependencies
├── requirements-dev.txt              # Development dependencies
├── pytest.ini                        # Pytest configuration
├── README.md                         # Project documentation
└── Makefile                          # Common commands
```

---

## Key Files Deep Dive

### 1. main.py - FastAPI Application Entry Point

**WHY:**
- Central configuration point for the entire application
- Lifespan management for startup/shutdown tasks
- Middleware registration in correct order
- Route registration with proper prefixes

**WHAT:**
```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
import time

from app.config import settings
from app.database import init_mongodb, close_mongodb, init_redis, close_redis
from app.dependencies import get_qdrant_client
from app.routes import (
    auth_router,
    chat_router,
    knowledge_router,
    assistant_router,
    tickets_router,
    analytics_router,
    health_router
)
from app.middleware.auth import auth_middleware
from app.middleware.tenant import tenant_isolation_middleware
from app.middleware.rate_limit import rate_limit_middleware
from app.utils.logger import logger

@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Manage application lifespan: startup and shutdown events.
    Replaces deprecated @app.on_event("startup") and @app.on_event("shutdown").
    """
    # STARTUP
    logger.info("Starting Customer Support RAG API...")

    # Initialize databases
    await init_mongodb()
    logger.info("MongoDB connected")

    await init_redis()
    logger.info("Redis connected")

    # Initialize Qdrant
    qdrant_client = get_qdrant_client()
    logger.info("Qdrant client initialized")

    # Initialize embedding model
    from app.utils.embeddings import EmbeddingService
    embedding_service = EmbeddingService()
    await embedding_service.load_model()
    logger.info("Embedding model loaded")

    logger.info("Application startup complete")

    yield  # Application runs here

    # SHUTDOWN
    logger.info("Shutting down Customer Support RAG API...")

    await close_mongodb()
    logger.info("MongoDB connection closed")

    await close_redis()
    logger.info("Redis connection closed")

    logger.info("Application shutdown complete")

# Initialize FastAPI app
app = FastAPI(
    title="Customer Support RAG API",
    description="AI-powered customer support with RAG capabilities",
    version="1.0.0",
    lifespan=lifespan,
    docs_url="/docs" if settings.ENVIRONMENT != "production" else None,
    redoc_url="/redoc" if settings.ENVIRONMENT != "production" else None,
)

# CORS Configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Custom Middleware (executes in reverse order of addition)
# Order: Request -> Rate Limit -> Tenant -> Auth -> Route Handler
@app.middleware("http")
async def add_request_id(request: Request, call_next):
    """Add unique request ID for tracing."""
    request_id = request.headers.get("X-Request-ID", f"req_{int(time.time() * 1000)}")
    request.state.request_id = request_id

    response = await call_next(request)
    response.headers["X-Request-ID"] = request_id
    return response

app.middleware("http")(rate_limit_middleware)
app.middleware("http")(tenant_isolation_middleware)
app.middleware("http")(auth_middleware)

# Global Exception Handler
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    logger.error(f"Unhandled exception: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal server error",
            "request_id": getattr(request.state, "request_id", None)
        }
    )

# Register Routes
app.include_router(health_router, prefix="/health", tags=["Health"])
app.include_router(auth_router, prefix="/api/v1/auth", tags=["Authentication"])
app.include_router(chat_router, prefix="/api/v1/chat", tags=["Chat"])
app.include_router(knowledge_router, prefix="/api/v1/knowledge", tags=["Knowledge"])
app.include_router(assistant_router, prefix="/api/v1/assistants", tags=["Assistants"])
app.include_router(tickets_router, prefix="/api/v1/tickets", tags=["Tickets"])
app.include_router(analytics_router, prefix="/api/v1/analytics", tags=["Analytics"])

@app.get("/")
async def root():
    return {
        "name": "Customer Support RAG API",
        "version": "1.0.0",
        "status": "running"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "app.main:app",
        host=settings.HOST,
        port=settings.PORT,
        reload=settings.ENVIRONMENT == "development",
        log_level="info"
    )
```

**HOW to run:**
```bash
# Development with auto-reload
python app/main.py

# Production with Gunicorn + Uvicorn workers
gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
```

---

### 2. config.py - Pydantic Settings for Configuration

**WHY:**
- Type-safe configuration with validation
- Environment-specific settings (dev, staging, prod)
- Secrets management (never commit .env to git)
- Single source of truth for all settings

**WHAT:**
```python
# app/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import List, Optional
from functools import lru_cache

class Settings(BaseSettings):
    """
    Application settings loaded from environment variables.
    Uses Pydantic Settings for validation and type safety.
    """
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore"
    )

    # Application
    APP_NAME: str = "Customer Support RAG API"
    ENVIRONMENT: str = "development"  # development, staging, production
    HOST: str = "0.0.0.0"
    PORT: int = 8000
    DEBUG: bool = True
    ALLOWED_ORIGINS: List[str] = ["http://localhost:3000", "http://localhost:8000"]

    # Security
    SECRET_KEY: str  # For JWT signing
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7

    # MongoDB
    MONGO_HOST: str = "localhost"
    MONGO_PORT: int = 27017
    MONGO_USER: Optional[str] = None
    MONGO_PASSWORD: Optional[str] = None
    MONGO_DB_NAME: str = "customer_support_rag"
    MONGO_MAX_POOL_SIZE: int = 50
    MONGO_MIN_POOL_SIZE: int = 10

    @property
    def MONGO_URI(self) -> str:
        """Construct MongoDB connection URI."""
        if self.MONGO_USER and self.MONGO_PASSWORD:
            return (
                f"mongodb://{self.MONGO_USER}:{self.MONGO_PASSWORD}@"
                f"{self.MONGO_HOST}:{self.MONGO_PORT}/{self.MONGO_DB_NAME}"
            )
        return f"mongodb://{self.MONGO_HOST}:{self.MONGO_PORT}/{self.MONGO_DB_NAME}"

    # Redis
    REDIS_HOST: str = "localhost"
    REDIS_PORT: int = 6379
    REDIS_PASSWORD: Optional[str] = None
    REDIS_DB: int = 0
    REDIS_CACHE_TTL: int = 3600  # 1 hour

    @property
    def REDIS_URI(self) -> str:
        """Construct Redis connection URI."""
        if self.REDIS_PASSWORD:
            return f"redis://:{self.REDIS_PASSWORD}@{self.REDIS_HOST}:{self.REDIS_PORT}/{self.REDIS_DB}"
        return f"redis://{self.REDIS_HOST}:{self.REDIS_PORT}/{self.REDIS_DB}"

    # Qdrant Vector Database
    QDRANT_HOST: str = "localhost"
    QDRANT_PORT: int = 6333
    QDRANT_API_KEY: Optional[str] = None
    QDRANT_COLLECTION_PREFIX: str = "org"
    QDRANT_VECTOR_SIZE: int = 384  # MiniLM embedding dimension

    # OpenAI / LLM
    OPENAI_API_KEY: str
    OPENAI_MODEL: str = "gpt-4"
    OPENAI_MAX_TOKENS: int = 1000
    OPENAI_TEMPERATURE: float = 0.7
    OPENAI_TIMEOUT: int = 30  # seconds

    # Embeddings
    EMBEDDING_MODEL: str = "sentence-transformers/all-MiniLM-L6-v2"
    EMBEDDING_BATCH_SIZE: int = 32
    EMBEDDING_MAX_LENGTH: int = 512

    # RAG Configuration
    RAG_CHUNK_SIZE: int = 512
    RAG_CHUNK_OVERLAP: int = 50
    RAG_TOP_K: int = 20
    RAG_RERANK_TOP_K: int = 5
    RAG_MAX_CONTEXT_TOKENS: int = 3000

    # Celery
    CELERY_BROKER_URL: str = "redis://localhost:6379/1"
    CELERY_RESULT_BACKEND: str = "redis://localhost:6379/2"
    CELERY_TASK_ALWAYS_EAGER: bool = False  # Set True for synchronous testing

    # Rate Limiting
    RATE_LIMIT_ENABLED: bool = True
    RATE_LIMIT_REQUESTS_PER_MINUTE: int = 60
    RATE_LIMIT_BURST: int = 10

    # Logging
    LOG_LEVEL: str = "INFO"
    LOG_FORMAT: str = "json"  # json or text

    # Circuit Breaker
    CIRCUIT_BREAKER_FAILURE_THRESHOLD: int = 5
    CIRCUIT_BREAKER_RECOVERY_TIMEOUT: int = 60  # seconds

    # File Upload
    MAX_UPLOAD_SIZE_MB: int = 10
    ALLOWED_FILE_TYPES: List[str] = [".pdf", ".docx", ".txt", ".md", ".html"]

    class Config:
        case_sensitive = True

@lru_cache()
def get_settings() -> Settings:
    """
    Get cached settings instance.
    Uses lru_cache to ensure settings are loaded only once.
    """
    return Settings()

settings = get_settings()
```

**HOW to use:**
```python
# In any module
from app.config import settings

# Access settings
print(settings.MONGO_URI)
print(settings.RAG_CHUNK_SIZE)
```

---

### 3. dependencies.py - Dependency Injection Setup

**WHY:**
- Centralized dependency management
- Easy to mock for testing
- Lifecycle management (connection pooling)
- Loose coupling between components

**WHAT:**
```python
# app/dependencies.py
from typing import Annotated, Optional
from fastapi import Depends, HTTPException, status, Header
from motor.motor_asyncio import AsyncIOMotorClient, AsyncIOMotorDatabase
from redis.asyncio import Redis
from qdrant_client import QdrantClient
import jwt

from app.config import settings
from app.models.user import User
from app.repositories.user_repo import UserRepository

# ============================================================================
# Database Dependencies
# ============================================================================

# MongoDB Client (singleton)
_mongo_client: Optional[AsyncIOMotorClient] = None

async def get_mongo_client() -> AsyncIOMotorClient:
    """Get MongoDB client instance."""
    global _mongo_client
    if _mongo_client is None:
        _mongo_client = AsyncIOMotorClient(
            settings.MONGO_URI,
            maxPoolSize=settings.MONGO_MAX_POOL_SIZE,
            minPoolSize=settings.MONGO_MIN_POOL_SIZE
        )
    return _mongo_client

async def get_database(
    client: Annotated[AsyncIOMotorClient, Depends(get_mongo_client)]
) -> AsyncIOMotorDatabase:
    """Get MongoDB database instance."""
    return client[settings.MONGO_DB_NAME]

# Redis Client (singleton)
_redis_client: Optional[Redis] = None

async def get_redis_client() -> Redis:
    """Get Redis client instance."""
    global _redis_client
    if _redis_client is None:
        _redis_client = await Redis.from_url(
            settings.REDIS_URI,
            encoding="utf-8",
            decode_responses=True
        )
    return _redis_client

# Qdrant Client (singleton)
_qdrant_client: Optional[QdrantClient] = None

def get_qdrant_client() -> QdrantClient:
    """Get Qdrant client instance."""
    global _qdrant_client
    if _qdrant_client is None:
        _qdrant_client = QdrantClient(
            host=settings.QDRANT_HOST,
            port=settings.QDRANT_PORT,
            api_key=settings.QDRANT_API_KEY
        )
    return _qdrant_client

# ============================================================================
# Authentication Dependencies
# ============================================================================

async def get_current_user(
    authorization: Annotated[Optional[str], Header()] = None,
    db: Annotated[AsyncIOMotorDatabase, Depends(get_database)] = None
) -> User:
    """
    Extract and validate JWT token, return current user.
    Used as a dependency for protected routes.
    """
    if not authorization:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Authorization header missing"
        )

    try:
        # Extract token from "Bearer <token>"
        scheme, token = authorization.split()
        if scheme.lower() != "bearer":
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid authentication scheme"
            )

        # Decode JWT
        payload = jwt.decode(
            token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        user_id: str = payload.get("sub")
        if user_id is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid token payload"
            )

        # Fetch user from database
        user_repo = UserRepository(db)
        user = await user_repo.find_by_id(user_id)
        if user is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="User not found"
            )

        return user

    except jwt.ExpiredSignatureError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Token expired"
        )
    except jwt.InvalidTokenError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid token"
        )
    except ValueError:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid authorization header format"
        )

async def get_current_organization_id(
    current_user: Annotated[User, Depends(get_current_user)]
) -> str:
    """
    Extract organization_id from current user.
    Used for tenant isolation in queries.
    """
    if not current_user.organization_id:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="User not associated with any organization"
        )
    return current_user.organization_id

# ============================================================================
# Repository Dependencies
# ============================================================================

async def get_user_repository(
    db: Annotated[AsyncIOMotorDatabase, Depends(get_database)]
) -> UserRepository:
    """Get user repository instance."""
    return UserRepository(db)

# Add similar factory functions for other repositories...

# ============================================================================
# Service Dependencies
# ============================================================================

from app.services.chat_service import ChatService
from app.services.knowledge_service import KnowledgeService
from app.utils.embeddings import EmbeddingService

async def get_embedding_service() -> EmbeddingService:
    """Get embedding service instance (singleton)."""
    # This would be initialized in lifespan and stored globally
    from app.main import embedding_service
    return embedding_service

async def get_chat_service(
    db: Annotated[AsyncIOMotorDatabase, Depends(get_database)],
    redis: Annotated[Redis, Depends(get_redis_client)],
    qdrant: Annotated[QdrantClient, Depends(get_qdrant_client)],
    embedding_service: Annotated[EmbeddingService, Depends(get_embedding_service)]
) -> ChatService:
    """Get chat service instance with all dependencies."""
    return ChatService(
        db=db,
        redis=redis,
        qdrant=qdrant,
        embedding_service=embedding_service
    )

async def get_knowledge_service(
    db: Annotated[AsyncIOMotorDatabase, Depends(get_database)],
    qdrant: Annotated[QdrantClient, Depends(get_qdrant_client)]
) -> KnowledgeService:
    """Get knowledge service instance with all dependencies."""
    return KnowledgeService(db=db, qdrant=qdrant)
```

**HOW to use in routes:**
```python
# app/routes/chat.py
from fastapi import APIRouter, Depends
from app.dependencies import get_current_user, get_chat_service
from app.models.user import User
from app.services.chat_service import ChatService

router = APIRouter()

@router.post("/")
async def create_chat(
    message: str,
    current_user: Annotated[User, Depends(get_current_user)],
    chat_service: Annotated[ChatService, Depends(get_chat_service)]
):
    """Create a new chat message and get AI response."""
    response = await chat_service.handle_chat(
        user_id=current_user.id,
        organization_id=current_user.organization_id,
        message=message
    )
    return response
```

---

## Environment Configuration

### .env.example Template

```bash
# .env.example - Template for environment variables
# Copy to .env and fill in actual values

# Application
APP_NAME="Customer Support RAG API"
ENVIRONMENT="development"  # development, staging, production
HOST="0.0.0.0"
PORT=8000
DEBUG=true
ALLOWED_ORIGINS=["http://localhost:3000"]

# Security
SECRET_KEY="your-secret-key-here-min-32-chars"
ALGORITHM="HS256"
ACCESS_TOKEN_EXPIRE_MINUTES=30

# MongoDB
MONGO_HOST="localhost"
MONGO_PORT=27017
MONGO_USER="admin"
MONGO_PASSWORD="password"
MONGO_DB_NAME="customer_support_rag"

# Redis
REDIS_HOST="localhost"
REDIS_PORT=6379
REDIS_PASSWORD=""
REDIS_DB=0

# Qdrant
QDRANT_HOST="localhost"
QDRANT_PORT=6333
QDRANT_API_KEY=""

# OpenAI
OPENAI_API_KEY="sk-..."
OPENAI_MODEL="gpt-4"

# Embeddings
EMBEDDING_MODEL="sentence-transformers/all-MiniLM-L6-v2"

# RAG
RAG_CHUNK_SIZE=512
RAG_CHUNK_OVERLAP=50

# Celery
CELERY_BROKER_URL="redis://localhost:6379/1"
CELERY_RESULT_BACKEND="redis://localhost:6379/2"
```

---

## Docker Compose Setup

### docker-compose.yml for Local Development

**WHY:**
- One command to start all services
- Consistent environment across team members
- Isolated networking and volumes

**WHAT:**
```yaml
# docker-compose.yml
version: '3.8'

services:
  # MongoDB
  mongodb:
    image: mongo:7
    container_name: rag_mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
      MONGO_INITDB_DATABASE: customer_support_rag
    volumes:
      - mongodb_data:/data/db
    networks:
      - rag_network

  # Redis
  redis:
    image: redis:7-alpine
    container_name: rag_redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - rag_network

  # Qdrant Vector Database
  qdrant:
    image: qdrant/qdrant:latest
    container_name: rag_qdrant
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_data:/qdrant/storage
    networks:
      - rag_network

  # FastAPI Application
  api:
    build:
      context: .
      dockerfile: docker/Dockerfile
    container_name: rag_api
    ports:
      - "8000:8000"
    environment:
      - MONGO_HOST=mongodb
      - REDIS_HOST=redis
      - QDRANT_HOST=qdrant
      - CELERY_BROKER_URL=redis://redis:6379/1
    env_file:
      - .env
    volumes:
      - ./app:/app/app
      - ./logs:/app/logs
    depends_on:
      - mongodb
      - redis
      - qdrant
    networks:
      - rag_network
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  # Celery Worker
  worker:
    build:
      context: .
      dockerfile: docker/Dockerfile.worker
    container_name: rag_worker
    environment:
      - MONGO_HOST=mongodb
      - REDIS_HOST=redis
      - QDRANT_HOST=qdrant
      - CELERY_BROKER_URL=redis://redis:6379/1
    env_file:
      - .env
    volumes:
      - ./app:/app/app
    depends_on:
      - redis
      - mongodb
      - qdrant
    networks:
      - rag_network
    command: celery -A app.workers.celery_app worker --loglevel=info --concurrency=4

  # Celery Beat (Scheduler)
  beat:
    build:
      context: .
      dockerfile: docker/Dockerfile.worker
    container_name: rag_beat
    environment:
      - REDIS_HOST=redis
      - CELERY_BROKER_URL=redis://redis:6379/1
    env_file:
      - .env
    volumes:
      - ./app:/app/app
    depends_on:
      - redis
    networks:
      - rag_network
    command: celery -A app.workers.celery_app beat --loglevel=info

volumes:
  mongodb_data:
  redis_data:
  qdrant_data:

networks:
  rag_network:
    driver: bridge
```

**HOW to use:**
```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f api

# Stop all services
docker-compose down

# Rebuild after code changes
docker-compose up -d --build
```

---

## Makefile for Common Commands

```makefile
# Makefile
.PHONY: install test run docker-up docker-down clean

install:
	pip install -r requirements.txt

install-dev:
	pip install -r requirements-dev.txt

test:
	pytest tests/ -v --cov=app --cov-report=html

test-unit:
	pytest tests/unit/ -v

test-integration:
	pytest tests/integration/ -v

run:
	python app/main.py

run-worker:
	celery -A app.workers.celery_app worker --loglevel=info

run-beat:
	celery -A app.workers.celery_app beat --loglevel=info

docker-up:
	docker-compose up -d

docker-down:
	docker-compose down

docker-logs:
	docker-compose logs -f

docker-rebuild:
	docker-compose up -d --build

clean:
	find . -type d -name __pycache__ -exec rm -rf {} +
	find . -type f -name "*.pyc" -delete
	rm -rf .pytest_cache htmlcov .coverage

lint:
	black app/ tests/
	isort app/ tests/
	flake8 app/ tests/

migrate:
	python scripts/migrate.py

seed:
	python scripts/seed_data.py

init-db:
	python scripts/init_db.py
```

---

## Interview Questions & Answers

### Q1: How do you structure a FastAPI project for scalability?

**A:** I structure FastAPI projects using a layered architecture:

1. **Routes Layer**: Thin handlers that validate input via Pydantic schemas, extract auth context, and delegate to services. No business logic here.

2. **Services Layer**: Core business logic organized by domain (chat, knowledge, auth). Services orchestrate multiple repositories and external APIs.

3. **Repository Layer**: Data access abstraction using the Repository pattern. Each model has a repository with methods like `find_by_id`, `create`, `update`. This makes it easy to switch databases or write tests with mocks.

4. **Models vs Schemas**: Models are database entities (MongoDB documents), Schemas are request/response DTOs. This separation prevents leaking database internals to the API.

5. **Dependency Injection**: All dependencies (database connections, services) are injected via FastAPI's `Depends()`. This makes code testable and enables lifecycle management (connection pooling).

6. **Configuration**: All settings in `config.py` using Pydantic Settings. Type-safe, validated, environment-specific. Never hardcode values.

This structure scales because each layer can be developed, tested, and scaled independently.

### Q2: Explain your dependency injection strategy in FastAPI.

**A:** I use FastAPI's built-in dependency injection system with a centralized `dependencies.py` file:

1. **Singleton Dependencies**: Database clients (MongoDB, Redis, Qdrant) are created once and reused. I use module-level variables with `lru_cache` or global state in the lifespan function.

2. **Per-Request Dependencies**: Services are created per request but reuse singleton database clients. For example, `ChatService` depends on `get_database`, `get_redis_client`, and `get_qdrant_client`.

3. **Dependency Chain**: I compose dependencies. `get_current_user` depends on `get_database`, and `get_current_organization_id` depends on `get_current_user`. FastAPI resolves the entire chain automatically.

4. **Testing**: In tests, I override dependencies with mocks:
```python
app.dependency_overrides[get_database] = lambda: mock_db
```

This makes testing isolated and fast without real databases.

5. **Type Annotations**: Using `Annotated[Type, Depends(func)]` provides better IDE support and clarity.

The benefits: testability, loose coupling, lifecycle management, and clear dependency graphs.

### Q3: How do you manage configuration across different environments?

**A:** I use Pydantic Settings with a `.env` file approach:

1. **Settings Class**: All configuration in `app/config.py` as a Pydantic Settings class. This provides type validation, defaults, and computed properties (like constructing `MONGO_URI` from components).

2. **Environment Files**:
   - `.env.example`: Template committed to git
   - `.env`: Actual secrets, gitignored
   - `.env.production`: Production overrides (loaded via environment variable `ENV_FILE`)

3. **Environment-Specific Values**: The `ENVIRONMENT` variable (dev/staging/prod) controls behavior like enabling debug mode or API docs.

4. **Secrets Management**: In production, I use cloud secret managers (AWS Secrets Manager, Azure Key Vault) and inject them as environment variables via Kubernetes or Docker.

5. **Validation**: Pydantic validates all settings on startup. If a required value is missing (like `OPENAI_API_KEY`), the app fails fast with a clear error.

6. **Caching**: Settings are cached using `@lru_cache` so they're loaded once, not on every import.

This approach prevents hardcoded values, supports multiple environments, and ensures type safety.

### Q4: Walk through your project's startup and shutdown lifecycle.

**A:** I use FastAPI's `lifespan` context manager (replaces deprecated startup/shutdown events):

**Startup sequence:**
1. **Database Connections**: Initialize MongoDB client with connection pool settings. This creates a persistent connection pool that all requests share.

2. **Redis Connection**: Initialize Redis client for caching and Celery queue.

3. **Qdrant Client**: Initialize vector database client.

4. **Embedding Model**: Load the sentence-transformer model into memory. This is expensive, so we do it once at startup.

5. **Health Check**: Ping all services to ensure they're reachable. Fail fast if any are down.

**Shutdown sequence:**
1. **Close Connections**: Gracefully close MongoDB and Redis connections, flushing any pending writes.

2. **Cancel Background Tasks**: If using `asyncio.create_task`, cancel them gracefully.

3. **Cleanup Resources**: Delete temp files, flush logs.

This ensures clean startup/shutdown and prevents connection leaks. In Kubernetes, this integrates with readiness/liveness probes.

### Q5: How do you organize tests for a FastAPI project?

**A:** I use a three-layer test structure:

1. **Unit Tests** (`tests/unit/`): Test individual functions/classes in isolation with mocked dependencies. Fast (milliseconds), high coverage.
   - Example: Test chunking logic in `utils/chunking.py` with string inputs
   - Mock all external dependencies (database, API calls)

2. **Integration Tests** (`tests/integration/`): Test interactions between components with real dependencies (test database, Redis).
   - Example: Test full ingestion flow from file upload to Qdrant storage
   - Use `pytest-asyncio` for async tests
   - Use fixtures to set up test data

3. **E2E Tests** (`tests/e2e/`): Test complete workflows from API request to response.
   - Example: POST to `/knowledge`, verify status, then query via `/chat`
   - Use FastAPI's `TestClient` for HTTP requests
   - Run against docker-compose test environment

**Key testing patterns:**
- `conftest.py`: Shared fixtures (mock database, test client, test users)
- `pytest.ini`: Configure test discovery, async support, coverage
- Dependency overrides: Replace real services with mocks
- Database cleanup: Ensure each test starts with a clean state

This structure gives confidence at all levels while keeping tests fast and maintainable.

---

## Summary

This project structure provides:

- **Clarity**: Each directory has a single responsibility
- **Scalability**: Layered architecture allows independent component scaling
- **Testability**: Dependency injection and repository pattern enable easy mocking
- **Maintainability**: Consistent patterns and configuration management
- **Production-Ready**: Docker support, health checks, graceful shutdown

The next section will cover the data models and database schema design.
