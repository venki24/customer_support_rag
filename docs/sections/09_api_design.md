# Section 9: API Design & Implementation

## WHY: API Design Matters for Multi-Tenant SaaS

A well-designed API is the contract between your frontend, third-party integrations, and your backend services. In a multi-tenant customer support RAG SaaS:

- **Scalability**: Clean API boundaries allow horizontal scaling of services
- **Multi-tenancy**: Proper tenant isolation at the API layer prevents data leakage
- **Developer Experience**: Consistent patterns reduce integration time
- **Versioning**: RESTful design enables backward-compatible evolution
- **Streaming**: Real-time chat responses require Server-Sent Events (SSE) for smooth UX
- **Security**: Authentication, authorization, and rate limiting must be API-first

Poor API design leads to tight coupling, security vulnerabilities, and frustrated developers. A thoughtful RESTful API with clear resource boundaries, proper HTTP semantics, and tenant isolation is foundational for a production-grade SaaS platform.

## WHAT: Complete API Endpoint Design

### Domain-Based Endpoint Organization

#### 1. Authentication Domain (`/api/v1/auth`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/signup` | Register new user with email/password |
| POST | `/auth/signin` | Login with email/password, returns access + refresh tokens |
| POST | `/auth/verify-otp` | Verify email OTP for 2FA |
| POST | `/auth/social/google` | OAuth login via Google |
| POST | `/auth/social/github` | OAuth login via GitHub |
| POST | `/auth/refresh` | Refresh access token using refresh token |
| POST | `/auth/logout` | Invalidate refresh token |
| POST | `/auth/forgot-password` | Request password reset email |
| POST | `/auth/reset-password` | Reset password with token |

#### 2. Organizations Domain (`/api/v1/organizations`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/organizations` | Create new organization (becomes tenant) |
| GET | `/organizations` | List user's organizations |
| GET | `/organizations/{org_id}` | Get organization details |
| PATCH | `/organizations/{org_id}` | Update organization settings |
| DELETE | `/organizations/{org_id}` | Delete organization (soft delete) |
| POST | `/organizations/{org_id}/members/invite` | Invite member via email |
| GET | `/organizations/{org_id}/members` | List organization members |
| DELETE | `/organizations/{org_id}/members/{user_id}` | Remove member |
| PATCH | `/organizations/{org_id}/members/{user_id}/role` | Update member role |

#### 3. Assistants Domain (`/api/v1/assistants`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/assistants` | Create AI assistant for organization |
| GET | `/assistants` | List assistants (filtered by org) |
| GET | `/assistants/{assistant_id}` | Get assistant details |
| PATCH | `/assistants/{assistant_id}` | Update assistant config (model, prompt, personality) |
| DELETE | `/assistants/{assistant_id}` | Delete assistant |
| POST | `/assistants/{assistant_id}/deploy` | Deploy assistant to production |
| GET | `/assistants/{assistant_id}/stats` | Get usage statistics |

#### 4. Knowledge Vault Domain (`/api/v1/knowledge`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/knowledge/sources/crawl` | Crawl URL(s) and extract content |
| POST | `/knowledge/sources/upload` | Upload file (PDF, DOCX, TXT, CSV) |
| POST | `/knowledge/sources/text` | Add raw text/markdown content |
| GET | `/knowledge/sources` | List all knowledge sources |
| GET | `/knowledge/sources/{source_id}` | Get source details |
| DELETE | `/knowledge/sources/{source_id}` | Delete source and its chunks |
| GET | `/knowledge/chunks` | List chunks with filters (pagination) |
| GET | `/knowledge/chunks/{chunk_id}` | Get specific chunk |
| PATCH | `/knowledge/chunks/{chunk_id}` | Edit chunk content |
| POST | `/knowledge/articles` | Create manual article |
| GET | `/knowledge/articles` | List articles |
| GET | `/knowledge/articles/{article_id}` | Get article content |
| PATCH | `/knowledge/articles/{article_id}` | Update article |
| DELETE | `/knowledge/articles/{article_id}` | Delete article |
| GET | `/knowledge/categories` | List categories |
| POST | `/knowledge/categories` | Create category |

#### 5. Chat Domain (`/api/v1/chat`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/chat/message` | Send message (SSE streaming response) |
| GET | `/chat/conversations` | List user's conversations |
| GET | `/chat/conversations/{conv_id}` | Get conversation history |
| DELETE | `/chat/conversations/{conv_id}` | Delete conversation |
| POST | `/chat/feedback` | Submit feedback (thumbs up/down) |
| POST | `/chat/conversations/{conv_id}/export` | Export conversation as PDF/JSON |

#### 6. Tickets Domain (`/api/v1/tickets`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/tickets` | Create support ticket |
| GET | `/tickets` | List tickets (with filters: status, priority, assignee) |
| GET | `/tickets/{ticket_id}` | Get ticket details |
| PATCH | `/tickets/{ticket_id}/status` | Update ticket status |
| PATCH | `/tickets/{ticket_id}/assign` | Assign ticket to agent |
| POST | `/tickets/{ticket_id}/comments` | Add comment to ticket |
| GET | `/tickets/{ticket_id}/comments` | List ticket comments |

#### 7. Analytics Domain (`/api/v1/analytics`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/analytics/dashboard` | Get dashboard metrics (today, 7d, 30d) |
| GET | `/analytics/chat-logs` | Paginated chat logs with filters |
| GET | `/analytics/resolution-rate` | Resolution rate over time |
| GET | `/analytics/popular-questions` | Top N user questions |
| GET | `/analytics/assistant-performance` | Per-assistant metrics |
| GET | `/analytics/knowledge-gaps` | Questions with no good answers |

#### 8. Customization Domain (`/api/v1/customization`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/customization/theme` | Get current theme config |
| PUT | `/customization/theme` | Update theme (colors, logo, fonts) |
| GET | `/customization/settings` | Get assistant settings (greetings, fallback) |
| PUT | `/customization/settings` | Update assistant settings |
| POST | `/customization/suggested-questions` | Add suggested question |
| GET | `/customization/suggested-questions` | List suggested questions |
| DELETE | `/customization/suggested-questions/{id}` | Remove suggested question |

### API Design Principles

1. **Multi-Tenancy Enforcement**: Every request includes `X-Organization-ID` header or extracts `org_id` from JWT
2. **Versioning**: `/api/v1/` prefix allows breaking changes in `/api/v2/`
3. **RESTful Conventions**: Use HTTP verbs correctly (GET, POST, PATCH, DELETE)
4. **Pagination**: Use `skip` and `limit` query params with `total` in response
5. **Filtering**: Query params for filters (e.g., `/tickets?status=open&priority=high`)
6. **Hypermedia**: Include relevant links in responses (HATEOAS-lite)
7. **Rate Limiting**: Per-tenant, per-endpoint limits with 429 responses
8. **Error Consistency**: Structured JSON errors with `error_code`, `message`, `details`

## HOW: Implementation with FastAPI

### 1. Chat Endpoint with SSE Streaming

```python
# routes/chat_api.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.responses import StreamingResponse
from typing import AsyncIterator
import json
from datetime import datetime

from schemas.chat_schemas import ChatRequest, ChatMessage, ConversationResponse
from services.chat_service import ChatService
from services.auth_service import get_current_user, get_organization_id
from models.user import User
from utils.logging_utils import logger

router = APIRouter(prefix="/api/v1/chat", tags=["Chat"])


@router.post("/message", response_class=StreamingResponse)
async def send_message(
    request: ChatRequest,
    current_user: User = Depends(get_current_user),
    org_id: str = Depends(get_organization_id),
    chat_service: ChatService = Depends(),
):
    """
    Send a chat message and stream the assistant's response via SSE.

    Args:
        request: ChatRequest with message, conversation_id, assistant_id
        current_user: Authenticated user from JWT
        org_id: Organization ID from JWT or header
        chat_service: Injected ChatService

    Returns:
        StreamingResponse with Server-Sent Events

    SSE Event Types:
        - token: Streamed text token
        - sources: Retrieved knowledge sources (sent once)
        - metadata: Response metadata (tokens, latency)
        - error: Error occurred during generation
        - done: Stream complete
    """
    logger.info(
        "Chat message received",
        user_id=current_user.id,
        org_id=org_id,
        assistant_id=request.assistant_id,
        conversation_id=request.conversation_id,
    )

    async def event_generator() -> AsyncIterator[str]:
        """Generate SSE events for streaming response."""
        try:
            # Stream response from chat service
            async for event in chat_service.stream_chat_response(
                user_id=current_user.id,
                org_id=org_id,
                message=request.message,
                conversation_id=request.conversation_id,
                assistant_id=request.assistant_id,
                context=request.context,
            ):
                # Format as SSE event
                event_type = event.get("type", "token")
                data = json.dumps(event.get("data", ""))

                yield f"event: {event_type}\n"
                yield f"data: {data}\n\n"

        except Exception as e:
            logger.error(
                "Error streaming chat response",
                error=str(e),
                user_id=current_user.id,
                org_id=org_id,
            )
            # Send error event
            error_event = {
                "error_code": "STREAMING_ERROR",
                "message": "An error occurred while generating the response",
                "details": str(e),
            }
            yield f"event: error\n"
            yield f"data: {json.dumps(error_event)}\n\n"
        finally:
            # Send done event
            yield f"event: done\n"
            yield f"data: {json.dumps({'timestamp': datetime.utcnow().isoformat()})}\n\n"

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        },
    )


@router.get("/conversations", response_model=list[ConversationResponse])
async def list_conversations(
    skip: int = 0,
    limit: int = 20,
    current_user: User = Depends(get_current_user),
    org_id: str = Depends(get_organization_id),
    chat_service: ChatService = Depends(),
):
    """
    List user's conversations with pagination.

    Query Params:
        skip: Number of conversations to skip (default: 0)
        limit: Max conversations to return (default: 20, max: 100)
    """
    if limit > 100:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Limit cannot exceed 100",
        )

    conversations = await chat_service.get_user_conversations(
        user_id=current_user.id,
        org_id=org_id,
        skip=skip,
        limit=limit,
    )

    return conversations


@router.get("/conversations/{conversation_id}", response_model=ConversationResponse)
async def get_conversation(
    conversation_id: str,
    current_user: User = Depends(get_current_user),
    org_id: str = Depends(get_organization_id),
    chat_service: ChatService = Depends(),
):
    """Get full conversation history by ID."""
    conversation = await chat_service.get_conversation(
        conversation_id=conversation_id,
        user_id=current_user.id,
        org_id=org_id,
    )

    if not conversation:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Conversation not found",
        )

    return conversation


@router.post("/feedback")
async def submit_feedback(
    message_id: str,
    rating: int,  # 1 (thumbs down) or 5 (thumbs up)
    comment: str | None = None,
    current_user: User = Depends(get_current_user),
    org_id: str = Depends(get_organization_id),
    chat_service: ChatService = Depends(),
):
    """Submit feedback for a chat message."""
    if rating not in [1, 5]:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Rating must be 1 (negative) or 5 (positive)",
        )

    await chat_service.save_feedback(
        message_id=message_id,
        user_id=current_user.id,
        org_id=org_id,
        rating=rating,
        comment=comment,
    )

    return {"message": "Feedback submitted successfully"}
```

### 2. Knowledge Vault Endpoints

```python
# routes/knowledge_api.py
from fastapi import APIRouter, Depends, HTTPException, UploadFile, File, status
from typing import List

from schemas.knowledge_schemas import (
    CrawlRequest,
    SourceResponse,
    ChunkResponse,
    ArticleCreate,
    ArticleResponse,
    PaginatedResponse,
)
from services.knowledge_service import KnowledgeService
from services.auth_service import get_current_user, get_organization_id
from models.user import User
from utils.logging_utils import logger

router = APIRouter(prefix="/api/v1/knowledge", tags=["Knowledge"])


@router.post("/sources/crawl", response_model=SourceResponse, status_code=status.HTTP_201_CREATED)
async def crawl_urls(
    request: CrawlRequest,
    current_user: User = Depends(get_current_user),
    org_id: str = Depends(get_organization_id),
    knowledge_service: KnowledgeService = Depends(),
):
    """
    Crawl one or more URLs to extract content for knowledge base.

    Process:
        1. Validate URLs and access permissions
        2. Enqueue crawl job to Celery
        3. Return source ID for status tracking

    Args:
        request: CrawlRequest with urls, max_depth, max_pages
    """
    logger.info(
        "Crawl request received",
        user_id=current_user.id,
        org_id=org_id,
        url_count=len(request.urls),
    )

    source = await knowledge_service.create_crawl_source(
        org_id=org_id,
        urls=request.urls,
        max_depth=request.max_depth,
        max_pages=request.max_pages,
        created_by=current_user.id,
    )

    return source


@router.post("/sources/upload", response_model=SourceResponse, status_code=status.HTTP_201_CREATED)
async def upload_file(
    file: UploadFile = File(...),
    current_user: User = Depends(get_current_user),
    org_id: str = Depends(get_organization_id),
    knowledge_service: KnowledgeService = Depends(),
):
    """
    Upload file (PDF, DOCX, TXT, CSV) to knowledge base.

    Supported formats:
        - PDF: Extracted via PyPDF2
        - DOCX: Extracted via python-docx
        - TXT/MD: Direct text extraction
        - CSV: Structured data extraction

    Max file size: 50MB
    """
    # Validate file type
    allowed_types = [
        "application/pdf",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        "text/plain",
        "text/markdown",
        "text/csv",
    ]

    if file.content_type not in allowed_types:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=f"Unsupported file type: {file.content_type}",
        )

    # Check file size (50MB limit)
    contents = await file.read()
    if len(contents) > 50 * 1024 * 1024:
        raise HTTPException(
            status_code=status.HTTP_413_REQUEST_ENTITY_TOO_LARGE,
            detail="File size exceeds 50MB limit",
        )

    logger.info(
        "File upload received",
        user_id=current_user.id,
        org_id=org_id,
        filename=file.filename,
        content_type=file.content_type,
        size_bytes=len(contents),
    )

    source = await knowledge_service.create_file_source(
        org_id=org_id,
        filename=file.filename,
        content_type=file.content_type,
        file_data=contents,
        created_by=current_user.id,
    )

    return source


@router.post("/sources/text", response_model=SourceResponse, status_code=status.HTTP_201_CREATED)
async def add_text(
    title: str,
    content: str,
    current_user: User = Depends(get_current_user),
    org_id: str = Depends(get_organization_id),
    knowledge_service: KnowledgeService = Depends(),
):
    """Add raw text/markdown content to knowledge base."""
    logger.info(
        "Text source added",
        user_id=current_user.id,
        org_id=org_id,
        title=title,
    )

    source = await knowledge_service.create_text_source(
        org_id=org_id,
        title=title,
        content=content,
        created_by=current_user.id,
    )

    return source


@router.get("/sources", response_model=PaginatedResponse[SourceResponse])
async def list_sources(
    skip: int = 0,
    limit: int = 20,
    source_type: str | None = None,
    org_id: str = Depends(get_organization_id),
    knowledge_service: KnowledgeService = Depends(),
):
    """
    List knowledge sources with pagination and filtering.

    Query Params:
        skip: Offset for pagination
        limit: Max items per page (max: 100)
        source_type: Filter by type (crawl, upload, text, article)
    """
    if limit > 100:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Limit cannot exceed 100",
        )

    result = await knowledge_service.list_sources(
        org_id=org_id,
        skip=skip,
        limit=limit,
        source_type=source_type,
    )

    return result


@router.get("/sources/{source_id}", response_model=SourceResponse)
async def get_source(
    source_id: str,
    org_id: str = Depends(get_organization_id),
    knowledge_service: KnowledgeService = Depends(),
):
    """Get source details including processing status and stats."""
    source = await knowledge_service.get_source(
        source_id=source_id,
        org_id=org_id,
    )

    if not source:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Source not found",
        )

    return source


@router.delete("/sources/{source_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_source(
    source_id: str,
    current_user: User = Depends(get_current_user),
    org_id: str = Depends(get_organization_id),
    knowledge_service: KnowledgeService = Depends(),
):
    """
    Delete source and all associated chunks from knowledge base.

    This will:
        1. Delete source metadata from MongoDB
        2. Delete all chunks from Qdrant vector database
        3. Invalidate cached embeddings in Redis
    """
    await knowledge_service.delete_source(
        source_id=source_id,
        org_id=org_id,
        deleted_by=current_user.id,
    )

    return None


@router.get("/chunks", response_model=PaginatedResponse[ChunkResponse])
async def list_chunks(
    skip: int = 0,
    limit: int = 50,
    source_id: str | None = None,
    org_id: str = Depends(get_organization_id),
    knowledge_service: KnowledgeService = Depends(),
):
    """List chunks with optional filtering by source."""
    result = await knowledge_service.list_chunks(
        org_id=org_id,
        skip=skip,
        limit=limit,
        source_id=source_id,
    )

    return result
```

### 3. Pydantic Schemas

```python
# schemas/chat_schemas.py
from pydantic import BaseModel, Field
from typing import Optional, List, Dict, Any
from datetime import datetime


class ChatRequest(BaseModel):
    """Request schema for chat message."""
    message: str = Field(..., min_length=1, max_length=4000, description="User message")
    conversation_id: Optional[str] = Field(None, description="Existing conversation ID")
    assistant_id: str = Field(..., description="Assistant ID to use")
    context: Optional[Dict[str, Any]] = Field(None, description="Additional context metadata")

    class Config:
        json_schema_extra = {
            "example": {
                "message": "How do I reset my password?",
                "conversation_id": "conv_abc123",
                "assistant_id": "asst_xyz789",
                "context": {"user_tier": "premium", "page": "/settings"},
            }
        }


class ChatMessage(BaseModel):
    """Individual chat message in conversation."""
    id: str
    role: str = Field(..., description="user or assistant")
    content: str
    timestamp: datetime
    metadata: Optional[Dict[str, Any]] = None
    sources: Optional[List[Dict[str, Any]]] = None


class ConversationResponse(BaseModel):
    """Conversation with full message history."""
    id: str
    user_id: str
    org_id: str
    assistant_id: str
    messages: List[ChatMessage]
    created_at: datetime
    updated_at: datetime
    metadata: Optional[Dict[str, Any]] = None


# schemas/knowledge_schemas.py
from pydantic import BaseModel, Field, HttpUrl
from typing import List, Optional, Generic, TypeVar
from datetime import datetime
from enum import Enum


class SourceType(str, Enum):
    """Knowledge source types."""
    CRAWL = "crawl"
    UPLOAD = "upload"
    TEXT = "text"
    ARTICLE = "article"


class SourceStatus(str, Enum):
    """Processing status for sources."""
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"


class CrawlRequest(BaseModel):
    """Request to crawl URLs."""
    urls: List[HttpUrl] = Field(..., min_length=1, max_length=10)
    max_depth: int = Field(2, ge=1, le=5, description="Max crawl depth")
    max_pages: int = Field(100, ge=1, le=1000, description="Max pages to crawl")

    class Config:
        json_schema_extra = {
            "example": {
                "urls": ["https://docs.example.com/getting-started"],
                "max_depth": 2,
                "max_pages": 50,
            }
        }


class SourceResponse(BaseModel):
    """Knowledge source metadata."""
    id: str
    org_id: str
    source_type: SourceType
    title: str
    status: SourceStatus
    chunk_count: int = 0
    created_at: datetime
    created_by: str
    metadata: Optional[Dict[str, Any]] = None

    class Config:
        from_attributes = True


class ChunkResponse(BaseModel):
    """Text chunk with metadata."""
    id: str
    source_id: str
    content: str
    embedding_id: str
    metadata: Dict[str, Any]
    created_at: datetime


class ArticleCreate(BaseModel):
    """Create manual knowledge base article."""
    title: str = Field(..., min_length=1, max_length=200)
    content: str = Field(..., min_length=1)
    category: Optional[str] = None
    tags: List[str] = Field(default_factory=list, max_length=10)


class ArticleResponse(BaseModel):
    """Knowledge base article."""
    id: str
    org_id: str
    title: str
    content: str
    category: Optional[str]
    tags: List[str]
    created_at: datetime
    updated_at: datetime
    created_by: str


T = TypeVar("T")


class PaginatedResponse(BaseModel, Generic[T]):
    """Generic paginated response."""
    items: List[T]
    total: int
    skip: int
    limit: int
    has_more: bool

    @classmethod
    def create(cls, items: List[T], total: int, skip: int, limit: int):
        """Helper to create paginated response."""
        return cls(
            items=items,
            total=total,
            skip=skip,
            limit=limit,
            has_more=(skip + len(items)) < total,
        )
```

### 4. Dependency Injection Pattern

```python
# services/auth_service.py
from fastapi import Depends, HTTPException, status, Header
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import jwt, JWTError
from typing import Optional

from models.user import User
from database.mongodb import get_database
from config import settings
from utils.logging_utils import logger

security = HTTPBearer()


async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
) -> User:
    """
    Extract and validate JWT token from Authorization header.

    Returns authenticated User object.
    Raises 401 if token is invalid or expired.
    """
    token = credentials.credentials

    try:
        payload = jwt.decode(
            token,
            settings.JWT_SECRET,
            algorithms=[settings.JWT_ALGORITHM],
        )
        user_id: str = payload.get("sub")
        if user_id is None:
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED,
                detail="Invalid authentication token",
            )
    except JWTError as e:
        logger.warning("JWT decode failed", error=str(e))
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid or expired token",
        )

    # Fetch user from database
    db = await get_database()
    user = await db.users.find_one({"_id": user_id})
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="User not found",
        )

    return User(**user)


async def get_organization_id(
    x_organization_id: Optional[str] = Header(None),
    current_user: User = Depends(get_current_user),
) -> str:
    """
    Extract organization ID from header or user's default org.

    Multi-tenancy enforcement:
        1. Check X-Organization-ID header
        2. Fallback to user's default organization
        3. Validate user has access to the organization
    """
    org_id = x_organization_id or current_user.default_org_id

    if not org_id:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Organization ID required (X-Organization-ID header or default org)",
        )

    # Verify user has access to this organization
    if org_id not in current_user.organization_ids:
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Access denied to this organization",
        )

    return org_id


# Usage in service classes
from fastapi import Depends

class ChatService:
    """Chat service with dependency injection."""

    def __init__(
        self,
        db=Depends(get_database),
        cache=Depends(get_redis_client),
        qdrant_client=Depends(get_qdrant_client),
    ):
        self.db = db
        self.cache = cache
        self.qdrant = qdrant_client
```

### 5. Error Handling Middleware

```python
# middleware/error_handler.py
from fastapi import Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from starlette.exceptions import HTTPException as StarletteHTTPException
from typing import Union
import traceback
from uuid import uuid4

from utils.logging_utils import logger


async def error_handler_middleware(request: Request, call_next):
    """
    Global error handling middleware.

    Catches all exceptions and returns structured JSON errors.
    """
    trace_id = str(uuid4())
    request.state.trace_id = trace_id

    try:
        response = await call_next(request)
        return response
    except Exception as exc:
        logger.error(
            "Unhandled exception",
            trace_id=trace_id,
            path=request.url.path,
            method=request.method,
            error=str(exc),
            traceback=traceback.format_exc(),
        )

        return JSONResponse(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            content={
                "error_code": "INTERNAL_SERVER_ERROR",
                "message": "An internal error occurred",
                "trace_id": trace_id,
                "details": str(exc) if settings.DEBUG else None,
            },
        )


def register_exception_handlers(app):
    """Register custom exception handlers."""

    @app.exception_handler(StarletteHTTPException)
    async def http_exception_handler(request: Request, exc: StarletteHTTPException):
        """Handle HTTP exceptions with structured format."""
        trace_id = getattr(request.state, "trace_id", str(uuid4()))

        return JSONResponse(
            status_code=exc.status_code,
            content={
                "error_code": f"HTTP_{exc.status_code}",
                "message": exc.detail,
                "trace_id": trace_id,
            },
        )

    @app.exception_handler(RequestValidationError)
    async def validation_exception_handler(request: Request, exc: RequestValidationError):
        """Handle Pydantic validation errors."""
        trace_id = getattr(request.state, "trace_id", str(uuid4()))

        return JSONResponse(
            status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
            content={
                "error_code": "VALIDATION_ERROR",
                "message": "Request validation failed",
                "trace_id": trace_id,
                "details": exc.errors(),
            },
        )


# app.py - Register middleware
from fastapi import FastAPI
from middleware.error_handler import error_handler_middleware, register_exception_handlers

app = FastAPI()

# Add error handling
app.middleware("http")(error_handler_middleware)
register_exception_handlers(app)
```

### 6. Rate Limiting Middleware

```python
# middleware/rate_limiter.py
from fastapi import Request, HTTPException, status
from typing import Callable
import time

from database.redis import get_redis_client
from utils.logging_utils import logger


class RateLimiter:
    """
    Token bucket rate limiter using Redis.

    Enforces per-tenant, per-endpoint rate limits.
    """

    def __init__(self, requests: int, window: int):
        """
        Args:
            requests: Max requests allowed in window
            window: Time window in seconds
        """
        self.requests = requests
        self.window = window

    def __call__(self, request: Request, call_next: Callable):
        """Rate limit middleware."""
        return self._check_rate_limit(request, call_next)

    async def _check_rate_limit(self, request: Request, call_next: Callable):
        """Check rate limit before processing request."""
        org_id = request.headers.get("X-Organization-ID")
        if not org_id:
            # No org_id, skip rate limiting (auth will fail later)
            return await call_next(request)

        # Build rate limit key
        endpoint = f"{request.method}:{request.url.path}"
        key = f"ratelimit:{org_id}:{endpoint}:{int(time.time() / self.window)}"

        redis = await get_redis_client()

        # Increment counter
        current = await redis.incr(key)
        if current == 1:
            # Set expiry on first request in window
            await redis.expire(key, self.window)

        # Check if over limit
        if current > self.requests:
            logger.warning(
                "Rate limit exceeded",
                org_id=org_id,
                endpoint=endpoint,
                current=current,
                limit=self.requests,
            )
            raise HTTPException(
                status_code=status.HTTP_429_TOO_MANY_REQUESTS,
                detail=f"Rate limit exceeded: {self.requests} requests per {self.window}s",
                headers={"Retry-After": str(self.window)},
            )

        # Add rate limit headers
        response = await call_next(request)
        response.headers["X-RateLimit-Limit"] = str(self.requests)
        response.headers["X-RateLimit-Remaining"] = str(max(0, self.requests - current))
        response.headers["X-RateLimit-Reset"] = str(int(time.time()) + self.window)

        return response
```

## Production Considerations

### 1. API Versioning Strategy

- Use URL path versioning (`/api/v1/`, `/api/v2/`)
- Maintain backward compatibility within major versions
- Deprecation headers: `X-API-Deprecation-Date`, `X-API-Sunset-Date`
- Version-specific routers in FastAPI

### 2. SSE vs WebSocket Decision Matrix

| Factor | SSE | WebSocket |
|--------|-----|-----------|
| **Direction** | Server → Client only | Bidirectional |
| **Protocol** | HTTP/1.1, HTTP/2 | Separate protocol (ws://) |
| **Reconnection** | Built-in auto-reconnect | Manual implementation |
| **Browser Support** | All modern browsers | All modern browsers |
| **Load Balancing** | Standard HTTP LB works | Requires sticky sessions |
| **Use Case** | Streaming responses, notifications | Real-time collaboration, gaming |

**For RAG chat streaming**: SSE is simpler and sufficient (one-way streaming from GPT to client).

### 3. Pagination Best Practices

```python
# Cursor-based pagination for large datasets
@router.get("/chat/messages")
async def list_messages(
    cursor: Optional[str] = None,  # Base64-encoded timestamp
    limit: int = 20,
):
    """Cursor pagination for infinite scroll."""
    # Decode cursor to get last timestamp
    last_timestamp = decode_cursor(cursor) if cursor else None

    # Query messages after cursor
    query = {"timestamp": {"$gt": last_timestamp}} if last_timestamp else {}
    messages = await db.messages.find(query).limit(limit + 1).to_list()

    # Check if more results exist
    has_more = len(messages) > limit
    if has_more:
        messages = messages[:limit]

    # Generate next cursor
    next_cursor = encode_cursor(messages[-1].timestamp) if messages else None

    return {
        "items": messages,
        "next_cursor": next_cursor,
        "has_more": has_more,
    }
```

### 4. Multi-Tenancy Enforcement Checklist

- [ ] Extract `org_id` from JWT or header in every endpoint
- [ ] Add `org_id` filter to all database queries
- [ ] Validate user has access to the organization
- [ ] Include `org_id` in Qdrant collection namespace
- [ ] Separate Redis keys by organization (`{org_id}:chat:...`)
- [ ] Apply per-tenant rate limits
- [ ] Audit log all cross-tenant access attempts

## Interview Q&A

### Q1: How would you design REST APIs for a multi-tenant SaaS platform?

**Answer:**

For multi-tenant SaaS, API design must enforce strict tenant isolation at every layer:

**1. Tenant Identification:**
- Extract `org_id` from JWT payload (set during authentication)
- Allow override via `X-Organization-ID` header for multi-org users
- Validate user has access to the requested organization

**2. Resource Scoping:**
- Every database query MUST include `org_id` filter
- Use compound indexes: `{org_id: 1, created_at: -1}`
- Vector database collections namespaced by `org_id`

**3. API Structure:**
- RESTful resource hierarchy: `/api/v1/{resource}`
- Use path parameters for IDs: `/assistants/{assistant_id}`
- Query params for filters: `/tickets?status=open`
- Pagination with `skip`, `limit`, and `total`

**4. Security:**
- JWT with short expiry (15min) + refresh tokens (7d)
- Per-tenant rate limiting using Redis
- Row-level security in queries (always filter by `org_id`)

**5. Versioning:**
- URL path versioning (`/v1/`, `/v2/`)
- Maintain backward compatibility within major versions
- Deprecation warnings in headers

**Example dependency injection for tenant isolation:**

```python
async def get_organization_id(
    x_organization_id: Optional[str] = Header(None),
    current_user: User = Depends(get_current_user),
) -> str:
    org_id = x_organization_id or current_user.default_org_id

    # Validate access
    if org_id not in current_user.organization_ids:
        raise HTTPException(status_code=403, detail="Access denied")

    return org_id
```

**Key principle:** Never trust client-provided tenant IDs without validating against the authenticated user's permissions.

### Q2: SSE vs WebSocket for streaming chat responses—which is better and why?

**Answer:**

**For RAG chat streaming, Server-Sent Events (SSE) is the better choice:**

**SSE Advantages:**
1. **Simpler Protocol:** Standard HTTP/1.1 or HTTP/2, no protocol upgrade needed
2. **Auto-Reconnection:** Browser's EventSource API automatically reconnects on disconnect
3. **Load Balancer Friendly:** Works with standard HTTP load balancers (no sticky sessions)
4. **One-Way Streaming:** Perfect for server → client streaming (GPT responses)
5. **Built-in Event Types:** Named events (`token`, `sources`, `error`, `done`)

**WebSocket Advantages:**
1. **Bidirectional:** Client can send data without new HTTP request
2. **Lower Latency:** No HTTP overhead per message
3. **Binary Data:** Efficient for non-text data

**Decision Matrix:**

| Use Case | Recommended |
|----------|-------------|
| Streaming GPT responses | **SSE** |
| Real-time collaborative editing | WebSocket |
| Live dashboards with updates | SSE |
| Multiplayer games | WebSocket |
| Chat with typing indicators | WebSocket |

**SSE Implementation:**

```python
async def event_generator():
    async for token in gpt_stream():
        yield f"event: token\ndata: {json.dumps({'text': token})}\n\n"
    yield f"event: done\ndata: {json.dumps({'timestamp': time.time()})}\n\n"

return StreamingResponse(
    event_generator(),
    media_type="text/event-stream",
    headers={"Cache-Control": "no-cache"},
)
```

**Client-side (JavaScript):**

```javascript
const eventSource = new EventSource('/api/v1/chat/message');

eventSource.addEventListener('token', (e) => {
    const data = JSON.parse(e.data);
    appendToken(data.text);
});

eventSource.addEventListener('done', (e) => {
    eventSource.close();
});
```

**Verdict:** Use SSE for one-way streaming (RAG responses), WebSocket only if you need bidirectional real-time communication.

### Q3: How do you handle pagination for large result sets in REST APIs?

**Answer:**

Two primary pagination strategies, each with trade-offs:

**1. Offset-Based Pagination (Skip/Limit):**

```python
@router.get("/tickets")
async def list_tickets(skip: int = 0, limit: int = 20):
    items = await db.tickets.find().skip(skip).limit(limit).to_list()
    total = await db.tickets.count_documents({})

    return {
        "items": items,
        "total": total,
        "skip": skip,
        "limit": limit,
        "has_more": (skip + len(items)) < total,
    }
```

**Pros:**
- Simple to implement
- Supports jumping to arbitrary pages
- Easy to show "Page X of Y"

**Cons:**
- Performance degrades with large offsets (skip=10000 slow)
- Inconsistent results if data changes between pages
- Total count query expensive for large collections

**2. Cursor-Based Pagination:**

```python
@router.get("/messages")
async def list_messages(
    cursor: Optional[str] = None,  # Encoded timestamp or ID
    limit: int = 20,
):
    last_id = decode_cursor(cursor) if cursor else None
    query = {"_id": {"$gt": last_id}} if last_id else {}

    items = await db.messages.find(query).sort("_id", 1).limit(limit + 1).to_list()
    has_more = len(items) > limit

    if has_more:
        items = items[:limit]

    next_cursor = encode_cursor(items[-1]._id) if items else None

    return {
        "items": items,
        "next_cursor": next_cursor,
        "has_more": has_more,
    }
```

**Pros:**
- Consistent performance (index-based lookup)
- Handles data changes gracefully
- No expensive count queries

**Cons:**
- Can't jump to arbitrary pages
- No "Page X of Y" display
- Cursor encoding adds complexity

**Best Practices:**

1. **Use cursor pagination for infinite scroll** (chat history, feeds)
2. **Use offset pagination for admin UIs** where page numbers matter
3. **Limit max page size** (e.g., 100 items) to prevent abuse
4. **Add indexes on sort fields** for performance
5. **Return `has_more` flag** to optimize client-side logic
6. **Consider keyset pagination** for time-series data (sorted by timestamp)

**Production tip:** For large collections (>1M docs), cursor pagination is essential to maintain <100ms response times.

### Q4: How do you implement proper error handling in FastAPI for production?

**Answer:**

Production-grade error handling requires multiple layers:

**1. Global Exception Middleware:**

```python
async def error_handler_middleware(request: Request, call_next):
    trace_id = str(uuid4())
    request.state.trace_id = trace_id

    try:
        return await call_next(request)
    except Exception as exc:
        logger.error("Unhandled exception", trace_id=trace_id, error=str(exc))

        return JSONResponse(
            status_code=500,
            content={
                "error_code": "INTERNAL_SERVER_ERROR",
                "message": "An internal error occurred",
                "trace_id": trace_id,
                "details": str(exc) if DEBUG else None,
            },
        )
```

**2. Custom Exception Handlers:**

```python
@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error_code": f"HTTP_{exc.status_code}",
            "message": exc.detail,
            "trace_id": request.state.trace_id,
        },
    )

@app.exception_handler(RequestValidationError)
async def validation_handler(request, exc):
    return JSONResponse(
        status_code=422,
        content={
            "error_code": "VALIDATION_ERROR",
            "message": "Request validation failed",
            "details": exc.errors(),
        },
    )
```

**3. Domain-Specific Exceptions:**

```python
class TenantQuotaExceeded(Exception):
    """Raised when tenant exceeds their quota."""
    pass

@app.exception_handler(TenantQuotaExceeded)
async def quota_handler(request, exc):
    return JSONResponse(
        status_code=429,
        content={
            "error_code": "QUOTA_EXCEEDED",
            "message": "Your organization has exceeded its quota",
            "details": {"limit": exc.limit, "usage": exc.usage},
        },
    )
```

**4. Structured Error Responses:**

```python
{
    "error_code": "RESOURCE_NOT_FOUND",
    "message": "The requested resource was not found",
    "trace_id": "abc-123-def-456",
    "details": {
        "resource_type": "assistant",
        "resource_id": "asst_xyz"
    }
}
```

**5. Error Codes Convention:**

- `VALIDATION_ERROR` - 422 Pydantic validation failures
- `UNAUTHORIZED` - 401 Missing/invalid JWT
- `FORBIDDEN` - 403 No permission for resource
- `RESOURCE_NOT_FOUND` - 404 Entity doesn't exist
- `QUOTA_EXCEEDED` - 429 Rate limit or plan quota
- `INTERNAL_SERVER_ERROR` - 500 Unhandled exceptions

**6. Structured Logging:**

```python
logger.error(
    "Failed to process chat message",
    trace_id=trace_id,
    user_id=user_id,
    org_id=org_id,
    error_type=type(exc).__name__,
    error_msg=str(exc),
    stack_trace=traceback.format_exc(),
)
```

**Production Checklist:**
- [ ] Never expose internal errors to clients in production
- [ ] Include `trace_id` in all error responses for debugging
- [ ] Log all 500 errors with full stack traces
- [ ] Return 400/422 for client errors with helpful messages
- [ ] Use 429 for rate limits with `Retry-After` header
- [ ] Monitor error rates by `error_code` in APM tool

### Q5: How do you implement dependency injection in FastAPI services?

**Answer:**

FastAPI's dependency injection system uses Python's type hints and `Depends()`:

**1. Basic Dependencies:**

```python
async def get_database():
    """Provide database connection."""
    client = AsyncIOMotorClient(settings.MONGO_URI)
    db = client[settings.DB_NAME]
    try:
        yield db
    finally:
        client.close()

@router.get("/users")
async def list_users(db=Depends(get_database)):
    users = await db.users.find().to_list(100)
    return users
```

**2. Authentication Dependencies:**

```python
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(HTTPBearer()),
) -> User:
    token = credentials.credentials
    payload = jwt.decode(token, settings.JWT_SECRET)
    user = await User.get(payload["sub"])
    return user

@router.get("/profile")
async def get_profile(current_user: User = Depends(get_current_user)):
    return current_user
```

**3. Service Layer Dependencies:**

```python
class ChatService:
    def __init__(
        self,
        db=Depends(get_database),
        cache=Depends(get_redis_client),
        qdrant=Depends(get_qdrant_client),
    ):
        self.db = db
        self.cache = cache
        self.qdrant = qdrant

    async def send_message(self, user_id: str, message: str):
        # Use self.db, self.cache, self.qdrant
        pass

@router.post("/chat")
async def send_message(
    request: ChatRequest,
    chat_service: ChatService = Depends(),
):
    return await chat_service.send_message(user_id, request.message)
```

**4. Chain Dependencies:**

```python
async def get_organization_id(
    current_user: User = Depends(get_current_user),
    x_org_id: str = Header(None),
) -> str:
    """Extract and validate org_id."""
    org_id = x_org_id or current_user.default_org_id

    if org_id not in current_user.organization_ids:
        raise HTTPException(403, "Access denied")

    return org_id

@router.get("/data")
async def get_data(
    org_id: str = Depends(get_organization_id),  # Depends on get_current_user
):
    return await db.data.find({"org_id": org_id}).to_list()
```

**5. Dependency Overrides (Testing):**

```python
# test_api.py
def mock_database():
    return MockDatabase()

app.dependency_overrides[get_database] = mock_database

client = TestClient(app)
response = client.get("/users")
```

**Benefits:**
1. **Reusability:** Share logic across endpoints
2. **Testability:** Override dependencies in tests
3. **Type Safety:** IDE autocomplete and type checking
4. **Automatic Validation:** FastAPI handles dependency errors
5. **Nested Dependencies:** Dependencies can depend on other dependencies

**Production Pattern:**

```python
# dependencies.py
async def get_chat_service(
    db=Depends(get_database),
    cache=Depends(get_redis_client),
    user=Depends(get_current_user),
    org_id=Depends(get_organization_id),
) -> ChatService:
    """Fully initialized ChatService with all deps."""
    return ChatService(db=db, cache=cache, user=user, org_id=org_id)

# routes/chat.py
@router.post("/message")
async def send_message(
    request: ChatRequest,
    service: ChatService = Depends(get_chat_service),
):
    return await service.process_message(request)
```

This pattern centralizes dependency wiring, making routes clean and testable.

---

**Next Steps:**
- Section 10: Implement multi-layer caching strategy with Redis
- Section 11: Design background job processing with Celery
- Section 12: Build monitoring and observability stack
