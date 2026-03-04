# Section 11: Background Jobs & Task Queue

## WHY: The Need for Asynchronous Processing

Heavy operations in a RAG system cannot block HTTP requests:

**Problems with Synchronous Processing**:
- Web crawling a 100-page documentation site takes 2-5 minutes
- Embedding generation for 1000 documents takes 30-60 seconds
- Large file parsing (50MB PDFs) takes 10-20 seconds
- HTTP requests would timeout (typically 30s limit)
- Server resources exhausted by long-running requests
- Poor user experience with blocked UI

**Why Background Jobs**:
- **Decoupled Execution**: API responds immediately with task ID, processing happens asynchronously
- **Resource Management**: Worker pools handle concurrency, preventing server overload
- **Reliability**: Automatic retries for transient failures (network issues, API rate limits)
- **Scalability**: Independent worker scaling based on queue depth
- **Scheduling**: Periodic maintenance tasks without cron complexity
- **Monitoring**: Centralized visibility into job status, failures, and performance

## WHAT: Celery + Redis Architecture

```
                                    ┌─────────────────────────────────┐
                                    │     FastAPI Application         │
                                    │  ┌──────────────────────────┐  │
   User Request ──────────────────> │  │  POST /knowledge/ingest  │  │
                                    │  │  - Validate input        │  │
                                    │  │  - Create task record    │  │
                                    │  │  - Enqueue Celery task   │  │
                                    │  │  - Return task_id        │  │
                                    │  └──────────┬───────────────┘  │
                                    └─────────────┼───────────────────┘
                                                  │
                                                  ▼
                        ┌─────────────────────────────────────────────┐
                        │           Redis (Message Broker)            │
                        │  ┌────────────┐  ┌────────────┐  ┌───────┐ │
                        │  │high_priority│  │  default   │  │  low  │ │
                        │  │   queue     │  │   queue    │  │ queue │ │
                        │  └────────────┘  └────────────┘  └───────┘ │
                        └─────────────────────────────────────────────┘
                                    │           │           │
                          ┌─────────┘           │           └─────────┐
                          ▼                     ▼                     ▼
            ┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────┐
            │  Celery Worker Pool  │ │  Celery Worker Pool  │ │Celery Worker │
            │  (High Priority)     │ │     (Default)        │ │    (Low)     │
            │  - 4 workers         │ │  - 8 workers         │ │  - 2 workers │
            │  - reindex_knowledge │ │  - ingest_url        │ │  - cleanup   │
            │  - generate_answer   │ │  - ingest_file       │ │  - analytics │
            └──────────┬───────────┘ └──────────┬───────────┘ └──────┬───────┘
                       │                        │                     │
                       └────────────┬───────────┴─────────────────────┘
                                    ▼
                        ┌───────────────────────┐
                        │  MongoDB (Job Store)  │
                        │  - Task status        │
                        │  - Progress tracking  │
                        │  - Error logs         │
                        │  - Result data        │
                        └───────────────────────┘
                                    │
                                    ▼
            User polls: GET /tasks/{task_id}/status → {status: "completed", progress: 100}
```

**Components**:
- **Celery**: Distributed task queue with worker management
- **Redis**: Message broker for task distribution and result backend
- **MongoDB**: Persistent storage for task metadata and status
- **Worker Pools**: Multiple workers per queue for concurrent processing
- **Beat Scheduler**: Periodic task execution (cron-like)

## HOW: Task Types and Implementation

### Task Categories

**1. Ingestion Tasks** (CPU/IO intensive):
- `ingest_url`: Crawl website, extract content, generate embeddings
- `ingest_file`: Parse PDF/DOCX/CSV, chunk content, store vectors
- `reindex_knowledge`: Rebuild vector database for tenant

**2. Analytics Tasks** (Scheduled):
- `generate_analytics`: Aggregate conversation metrics, compute statistics
- `detect_stale_knowledge`: Find outdated documents based on age and usage
- `compute_answer_quality`: Calculate CSAT scores, identify improvement areas

**3. Maintenance Tasks** (Scheduled):
- `cleanup_expired`: Remove expired sessions, old conversations
- `archive_old_data`: Move historical data to cold storage
- `prune_vector_db`: Clean up deleted document vectors

**4. Real-time Tasks** (High Priority):
- `generate_answer`: RAG pipeline execution for chat queries
- `regenerate_embedding`: Single document re-embedding on update

### Celery Configuration

```python
# celery_app.py
from celery import Celery
from celery.schedules import crontab
from config import settings

celery_app = Celery(
    "customer_support_rag",
    broker=settings.REDIS_URL,  # Redis for task queue
    backend=settings.REDIS_URL,  # Redis for result storage
)

# Configuration
celery_app.conf.update(
    # Task routing: different queues for different priorities
    task_routes={
        "tasks.ingest_url": {"queue": "default"},
        "tasks.ingest_file": {"queue": "default"},
        "tasks.reindex_knowledge": {"queue": "high_priority"},
        "tasks.generate_answer": {"queue": "high_priority"},
        "tasks.generate_analytics": {"queue": "low"},
        "tasks.cleanup_expired": {"queue": "low"},
    },

    # Retry policy with exponential backoff
    task_annotations={
        "*": {
            "autoretry_for": (Exception,),
            "retry_kwargs": {"max_retries": 3},
            "retry_backoff": True,  # Exponential backoff
            "retry_backoff_max": 600,  # Max 10 minutes
            "retry_jitter": True,  # Add randomness to prevent thundering herd
        }
    },

    # Result expiration
    result_expires=3600,  # 1 hour

    # Task time limits
    task_soft_time_limit=300,  # 5 minutes soft limit (raises exception)
    task_time_limit=600,  # 10 minutes hard limit (kills worker)

    # Worker settings
    worker_prefetch_multiplier=1,  # Fetch one task at a time (better for long tasks)
    worker_max_tasks_per_child=1000,  # Restart worker after 1000 tasks (prevent memory leaks)

    # Serialization
    task_serializer="json",
    result_serializer="json",
    accept_content=["json"],

    # Timezone
    timezone="UTC",
    enable_utc=True,
)

# Celery Beat schedule for periodic tasks
celery_app.conf.beat_schedule = {
    "generate-daily-analytics": {
        "task": "tasks.generate_analytics",
        "schedule": crontab(hour=2, minute=0),  # 2 AM daily
        "kwargs": {"period": "daily"},
    },
    "detect-stale-knowledge-weekly": {
        "task": "tasks.detect_stale_knowledge",
        "schedule": crontab(day_of_week=1, hour=3, minute=0),  # Monday 3 AM
        "kwargs": {"threshold_days": 90},
    },
    "cleanup-expired-hourly": {
        "task": "tasks.cleanup_expired",
        "schedule": crontab(minute=0),  # Every hour
    },
    "prune-vector-db-daily": {
        "task": "tasks.prune_vector_db",
        "schedule": crontab(hour=4, minute=0),  # 4 AM daily
    },
}
```

### Job Status Tracking

```python
# models/job_status.py
from datetime import datetime
from enum import Enum
from typing import Optional, Dict, Any
from beanie import Document
from pydantic import Field

class TaskStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"
    RETRY = "retry"

class TaskType(str, Enum):
    INGEST_URL = "ingest_url"
    INGEST_FILE = "ingest_file"
    REINDEX_KNOWLEDGE = "reindex_knowledge"
    GENERATE_ANALYTICS = "generate_analytics"
    CLEANUP_EXPIRED = "cleanup_expired"
    DETECT_STALE = "detect_stale_knowledge"

class JobStatus(Document):
    task_id: str = Field(..., description="Celery task ID")
    tenant_id: str = Field(..., description="Tenant who owns this job")
    user_id: str = Field(..., description="User who initiated the job")

    type: TaskType = Field(..., description="Type of task")
    status: TaskStatus = Field(default=TaskStatus.PENDING)

    progress: int = Field(default=0, ge=0, le=100, description="Progress percentage")
    current_step: Optional[str] = Field(None, description="Current operation")

    result: Optional[Dict[str, Any]] = Field(None, description="Task result data")
    error: Optional[str] = Field(None, description="Error message if failed")

    created_at: datetime = Field(default_factory=datetime.utcnow)
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None

    retry_count: int = Field(default=0)

    # Metadata for specific task types
    metadata: Dict[str, Any] = Field(default_factory=dict)

    class Settings:
        name = "job_statuses"
        indexes = [
            "task_id",
            "tenant_id",
            [("tenant_id", 1), ("created_at", -1)],
            [("status", 1), ("created_at", -1)],
        ]
```

### Ingestion Task with Progress Tracking

```python
# tasks/ingestion_tasks.py
from celery import Task
from celery.signals import task_prerun, task_postrun, task_failure, task_retry
from typing import List, Dict, Any
import asyncio
from datetime import datetime

from celery_app import celery_app
from services.crawler_service import CrawlerService
from services.embedding_service import EmbeddingService
from services.knowledge_service import KnowledgeService
from models.job_status import JobStatus, TaskStatus

class BaseTaskWithProgress(Task):
    """Base task class with progress tracking"""

    async def update_progress(self, task_id: str, progress: int, current_step: str):
        """Update job status in MongoDB"""
        await JobStatus.find_one(JobStatus.task_id == task_id).update({
            "$set": {
                "progress": progress,
                "current_step": current_step,
                "status": TaskStatus.RUNNING,
            }
        })

@celery_app.task(bind=True, base=BaseTaskWithProgress, name="tasks.ingest_url")
def ingest_url_task(self, tenant_id: str, url: str, max_depth: int = 2) -> Dict[str, Any]:
    """
    Crawl a URL, extract content, generate embeddings, and store in knowledge base.

    Progress breakdown:
    - 0-30%: Crawling pages
    - 30-60%: Parsing and chunking content
    - 60-90%: Generating embeddings
    - 90-100%: Storing in vector database
    """
    loop = asyncio.get_event_loop()
    return loop.run_until_complete(
        _ingest_url_async(self, tenant_id, url, max_depth)
    )

async def _ingest_url_async(task, tenant_id: str, url: str, max_depth: int) -> Dict[str, Any]:
    task_id = task.request.id

    try:
        # Update status to running
        await JobStatus.find_one(JobStatus.task_id == task_id).update({
            "$set": {
                "status": TaskStatus.RUNNING,
                "started_at": datetime.utcnow(),
            }
        })

        # Step 1: Crawl pages (0-30%)
        await task.update_progress(task_id, 5, "Initializing crawler")
        crawler = CrawlerService(tenant_id)

        pages = []
        async for page in crawler.crawl(url, max_depth=max_depth):
            pages.append(page)
            progress = min(30, 5 + (len(pages) * 2))  # Increment progress
            await task.update_progress(task_id, progress, f"Crawled {len(pages)} pages")

        # Step 2: Parse and chunk content (30-60%)
        await task.update_progress(task_id, 30, "Parsing content")
        knowledge_service = KnowledgeService(tenant_id)
        chunks = []

        for i, page in enumerate(pages):
            page_chunks = await knowledge_service.chunk_content(
                page["content"],
                metadata={"url": page["url"], "title": page["title"]}
            )
            chunks.extend(page_chunks)
            progress = 30 + int((i + 1) / len(pages) * 30)
            await task.update_progress(task_id, progress, f"Parsed {i+1}/{len(pages)} pages")

        # Step 3: Generate embeddings (60-90%)
        await task.update_progress(task_id, 60, "Generating embeddings")
        embedding_service = EmbeddingService()

        batch_size = 50
        embedded_chunks = []
        for i in range(0, len(chunks), batch_size):
            batch = chunks[i:i+batch_size]
            texts = [chunk["content"] for chunk in batch]
            embeddings = await embedding_service.embed_batch(texts)

            for chunk, embedding in zip(batch, embeddings):
                chunk["embedding"] = embedding
                embedded_chunks.append(chunk)

            progress = 60 + int((i + batch_size) / len(chunks) * 30)
            await task.update_progress(task_id, min(90, progress),
                                      f"Embedded {len(embedded_chunks)}/{len(chunks)} chunks")

        # Step 4: Store in vector database (90-100%)
        await task.update_progress(task_id, 90, "Storing in knowledge base")
        document_ids = await knowledge_service.bulk_insert(embedded_chunks, source_url=url)

        await task.update_progress(task_id, 100, "Completed")

        # Mark as completed
        result = {
            "pages_crawled": len(pages),
            "chunks_created": len(chunks),
            "documents_stored": len(document_ids),
            "source_url": url,
        }

        await JobStatus.find_one(JobStatus.task_id == task_id).update({
            "$set": {
                "status": TaskStatus.COMPLETED,
                "progress": 100,
                "result": result,
                "completed_at": datetime.utcnow(),
            }
        })

        return result

    except Exception as e:
        # Mark as failed
        await JobStatus.find_one(JobStatus.task_id == task_id).update({
            "$set": {
                "status": TaskStatus.FAILED,
                "error": str(e),
                "completed_at": datetime.utcnow(),
            }
        })
        raise
```

### Job Status Service

```python
# services/job_service.py
from typing import Optional, List
from datetime import datetime, timedelta

from models.job_status import JobStatus, TaskStatus, TaskType
from celery_app import celery_app

class JobService:
    def __init__(self, tenant_id: str):
        self.tenant_id = tenant_id

    async def create_job(
        self,
        task_id: str,
        user_id: str,
        task_type: TaskType,
        metadata: dict = None
    ) -> JobStatus:
        """Create a new job status record"""
        job = JobStatus(
            task_id=task_id,
            tenant_id=self.tenant_id,
            user_id=user_id,
            type=task_type,
            status=TaskStatus.PENDING,
            metadata=metadata or {},
        )
        await job.insert()
        return job

    async def get_job(self, task_id: str) -> Optional[JobStatus]:
        """Get job status by task ID"""
        return await JobStatus.find_one(
            JobStatus.task_id == task_id,
            JobStatus.tenant_id == self.tenant_id,
        )

    async def list_jobs(
        self,
        status: Optional[TaskStatus] = None,
        task_type: Optional[TaskType] = None,
        limit: int = 50,
        skip: int = 0,
    ) -> List[JobStatus]:
        """List jobs with filters"""
        query = {"tenant_id": self.tenant_id}

        if status:
            query["status"] = status
        if task_type:
            query["type"] = task_type

        return await JobStatus.find(query).sort("-created_at").skip(skip).limit(limit).to_list()

    async def cancel_job(self, task_id: str) -> bool:
        """Cancel a running job"""
        job = await self.get_job(task_id)
        if not job or job.status not in [TaskStatus.PENDING, TaskStatus.RUNNING]:
            return False

        # Revoke Celery task
        celery_app.control.revoke(task_id, terminate=True)

        # Update status
        await job.update({
            "$set": {
                "status": TaskStatus.CANCELLED,
                "completed_at": datetime.utcnow(),
            }
        })
        return True

    async def retry_failed_job(self, task_id: str) -> Optional[str]:
        """Retry a failed job"""
        job = await self.get_job(task_id)
        if not job or job.status != TaskStatus.FAILED:
            return None

        # Create new task with same parameters
        task = celery_app.tasks.get(f"tasks.{job.type}")
        if not task:
            return None

        result = task.apply_async(kwargs=job.metadata)

        # Create new job record
        new_job = await self.create_job(
            task_id=result.id,
            user_id=job.user_id,
            task_type=job.type,
            metadata=job.metadata,
        )

        return new_job.task_id
```

### API Integration

```python
# routes/knowledge_routes.py
from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
from pydantic import BaseModel, HttpUrl

from auth.jwt_middleware import get_current_user
from services.job_service import JobService
from models.job_status import TaskType
from tasks.ingestion_tasks import ingest_url_task

router = APIRouter(prefix="/knowledge", tags=["knowledge"])

class IngestURLRequest(BaseModel):
    url: HttpUrl
    max_depth: int = 2

@router.post("/ingest/url")
async def ingest_url(
    request: IngestURLRequest,
    current_user: dict = Depends(get_current_user),
):
    """
    Initiate URL ingestion task.
    Returns task_id for status polling.
    """
    tenant_id = current_user["tenant_id"]
    user_id = current_user["user_id"]

    # Enqueue Celery task
    result = ingest_url_task.apply_async(
        kwargs={
            "tenant_id": tenant_id,
            "url": str(request.url),
            "max_depth": request.max_depth,
        }
    )

    # Create job status record
    job_service = JobService(tenant_id)
    job = await job_service.create_job(
        task_id=result.id,
        user_id=user_id,
        task_type=TaskType.INGEST_URL,
        metadata={"url": str(request.url), "max_depth": request.max_depth},
    )

    return {
        "task_id": result.id,
        "status": "pending",
        "message": "Ingestion task queued successfully",
    }

@router.get("/tasks/{task_id}/status")
async def get_task_status(
    task_id: str,
    current_user: dict = Depends(get_current_user),
):
    """Get status of a background task"""
    tenant_id = current_user["tenant_id"]
    job_service = JobService(tenant_id)

    job = await job_service.get_job(task_id)
    if not job:
        raise HTTPException(status_code=404, detail="Task not found")

    return {
        "task_id": job.task_id,
        "type": job.type,
        "status": job.status,
        "progress": job.progress,
        "current_step": job.current_step,
        "result": job.result,
        "error": job.error,
        "created_at": job.created_at,
        "completed_at": job.completed_at,
    }
```

## Production Tips

### 1. Monitoring with Flower

```bash
# Install Flower
pip install flower

# Run Flower dashboard
celery -A celery_app flower --port=5555
```

Access dashboard at `http://localhost:5555` for:
- Active/completed/failed task counts
- Worker status and resource usage
- Task execution timelines
- Real-time monitoring

### 2. Dead Letter Queue

```python
# celery_app.py - add to configuration
celery_app.conf.update(
    task_reject_on_worker_lost=True,
    task_acks_late=True,  # Acknowledge after task completion, not before

    # Dead letter queue for failed tasks
    task_routes={
        "tasks.*": {
            "queue": "default",
            "routing_key": "task.default",
        },
    },
)

# Create separate queue for failed tasks
@celery_app.task(bind=True)
def handle_failed_task(self, task_id: str, error: str):
    """Move failed task to dead letter queue for manual inspection"""
    # Log to monitoring system
    # Send alert
    # Store in separate collection for analysis
    pass
```

### 3. Result Expiry and Cleanup

```python
# Celery config
result_expires=3600  # 1 hour

# Periodic cleanup task
@celery_app.task
def cleanup_old_results():
    """Remove expired task results from Redis"""
    # Celery handles this automatically if result_expires is set
    # Additional cleanup for MongoDB job records
    cutoff = datetime.utcnow() - timedelta(days=30)
    await JobStatus.find(
        JobStatus.completed_at < cutoff,
        JobStatus.status.in_([TaskStatus.COMPLETED, TaskStatus.FAILED])
    ).delete()
```

### 4. Worker Autoscaling

```bash
# Start workers with autoscaling
celery -A celery_app worker --autoscale=10,3  # Max 10, min 3 workers

# Separate workers for different queues
celery -A celery_app worker -Q high_priority --autoscale=5,2 -n worker_high@%h
celery -A celery_app worker -Q default --autoscale=10,4 -n worker_default@%h
celery -A celery_app worker -Q low --autoscale=3,1 -n worker_low@%h
```

### 5. Error Tracking

```python
from celery.signals import task_failure
import sentry_sdk

@task_failure.connect
def handle_task_failure(sender=None, task_id=None, exception=None, **kwargs):
    """Capture task failures in Sentry"""
    sentry_sdk.capture_exception(exception)

    # Update job status
    asyncio.run(update_job_on_failure(task_id, str(exception)))
```

## Interview Q&A

**Q1: Why use a task queue like Celery instead of Python background threads or asyncio tasks?**

**A**: Several critical reasons:

1. **Persistence**: Threads die if the server restarts. Celery tasks persist in Redis and resume after restart.
2. **Distribution**: Threads are limited to one process. Celery distributes tasks across multiple worker machines.
3. **Monitoring**: No built-in monitoring for threads. Celery provides task status, history, and Flower dashboard.
4. **Retry Logic**: Manual implementation for threads. Celery has automatic retry with exponential backoff.
5. **Resource Limits**: Threads share process memory. Celery workers have separate memory spaces and can be killed/restarted independently.
6. **Priority Queues**: Difficult with threads. Celery has built-in queue routing and prioritization.

For our RAG system, ingestion tasks might take 5-10 minutes. If the server restarts mid-task with threads, all progress is lost. With Celery, tasks resume after restart.

**Q2: How do you handle failed tasks and prevent infinite retries?**

**A**: Multi-layered approach:

```python
# 1. Automatic retry with exponential backoff
@celery_app.task(
    autoretry_for=(NetworkError, RateLimitError),  # Only retry transient errors
    retry_kwargs={"max_retries": 3},
    retry_backoff=True,  # 2^retry * base_delay
    retry_backoff_max=600,  # Max 10 minutes between retries
)

# 2. Track retry count in job status
async def _task_with_retry_tracking():
    job = await JobStatus.find_one(JobStatus.task_id == task_id)
    job.retry_count += 1
    await job.save()

# 3. Dead letter queue for max retries exceeded
if job.retry_count >= MAX_RETRIES:
    await move_to_dead_letter_queue(job)
    await send_alert_to_admin(job)

# 4. Manual retry endpoint for admins
@router.post("/tasks/{task_id}/retry")
async def manual_retry(task_id: str, admin_user: dict = Depends(require_admin)):
    # Human decision to retry after investigating failure
    pass
```

**Q3: How do you prevent duplicate task execution for scheduled jobs?**

**A**: Use Celery's task ID and database locking:

```python
from celery import Task
from redis import Redis
import hashlib

class IdempotentTask(Task):
    """Prevent duplicate execution using Redis lock"""

    def apply_async(self, args=None, kwargs=None, **options):
        # Generate unique task key based on task name and arguments
        task_key = hashlib.md5(
            f"{self.name}:{str(args)}:{str(kwargs)}".encode()
        ).hexdigest()

        redis = Redis.from_url(settings.REDIS_URL)
        lock_key = f"task_lock:{task_key}"

        # Try to acquire lock with 10-minute expiry
        if redis.set(lock_key, "1", ex=600, nx=True):
            # Lock acquired, execute task
            result = super().apply_async(args, kwargs, **options)
            return result
        else:
            # Lock exists, task already running
            return None

@celery_app.task(base=IdempotentTask, name="tasks.generate_analytics")
def generate_analytics(tenant_id: str, period: str):
    # This task won't execute if already running for same tenant/period
    pass
```

**Q4: How do you implement task prioritization for different tenant tiers?**

**A**: Combination of queue routing and priority levels:

```python
# 1. Route tasks based on tenant tier
def route_task_for_tenant(tenant_id: str, task_name: str):
    tenant = await Tenant.get(tenant_id)

    if tenant.tier == "enterprise":
        return {"queue": "high_priority", "priority": 9}
    elif tenant.tier == "pro":
        return {"queue": "default", "priority": 5}
    else:
        return {"queue": "low", "priority": 1}

# 2. Apply routing when enqueueing
result = ingest_url_task.apply_async(
    kwargs={"tenant_id": tenant_id, "url": url},
    **route_task_for_tenant(tenant_id, "ingest_url")
)

# 3. Worker configuration
# Start more workers for high priority queue
celery -A celery_app worker -Q high_priority --autoscale=10,5
celery -A celery_app worker -Q default --autoscale=5,2
celery -A celery_app worker -Q low --autoscale=2,1
```

**Q5: How do you handle long-running tasks that exceed time limits?**

**A**: Checkpointing and task chunking:

```python
@celery_app.task(
    soft_time_limit=300,  # Raise exception after 5 minutes
    time_limit=360,  # Kill worker after 6 minutes
)
def long_running_task(tenant_id: str, checkpoint_data: dict = None):
    try:
        # Resume from checkpoint if provided
        if checkpoint_data:
            start_index = checkpoint_data["last_processed_index"]
        else:
            start_index = 0

        items = get_items_to_process(tenant_id)

        for i in range(start_index, len(items)):
            process_item(items[i])

            # Save checkpoint every 100 items
            if i % 100 == 0:
                save_checkpoint(task_id, {"last_processed_index": i})

    except SoftTimeLimitExceeded:
        # Soft time limit reached, enqueue continuation task
        checkpoint = {"last_processed_index": i}
        long_running_task.apply_async(
            kwargs={"tenant_id": tenant_id, "checkpoint_data": checkpoint}
        )
        raise  # Let Celery know task didn't complete
```

Alternatively, break into smaller tasks:

```python
@celery_app.task
def process_large_dataset(tenant_id: str):
    # Break into chunks
    chunks = chunk_data(tenant_id, chunk_size=100)

    # Create subtask for each chunk
    job = group(
        process_chunk.s(tenant_id, chunk) for chunk in chunks
    )

    # Execute in parallel and aggregate results
    result = job.apply_async()
    return result.get()  # Wait for all subtasks
```
