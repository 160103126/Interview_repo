# API Design — Coding Patterns

> Core patterns for building robust, scalable, and secure REST and GraphQL APIs.

---

## Table of Contents

- [Q1: RESTful Resource Routing](#q1-restful-resource-routing)
- [Q2: Pagination Implementation (Offset vs Cursor)](#q2-pagination-implementation-offset-vs-cursor)
- [Q3: Global Error Handling / Exception Middleware](#q3-global-error-handling--exception-middleware)
- [Q4: Idempotency Keys](#q4-idempotency-keys)
- [Q5: API Versioning Strategies](#q5-api-versioning-strategies)

---

### Q1: RESTful Resource Routing

**Problem:** Design a clean, standard RESTful API router for a nested resource (e.g., Comments on a Post) using FastAPI.

```python
from fastapi import APIRouter, HTTPException, status
from pydantic import BaseModel
from typing import List

# DTOs (Data Transfer Objects)
class CommentCreate(BaseModel):
    content: str

class CommentResponse(BaseModel):
    id: int
    post_id: int
    content: str

# Router for nested resource
router = APIRouter(prefix="/posts/{post_id}/comments", tags=["Comments"])

# 1. CREATE
@router.post("/", response_model=CommentResponse, status_code=status.HTTP_201_CREATED)
async def create_comment(post_id: int, comment: CommentCreate):
    # e.g., POST /posts/123/comments
    return {"id": 1, "post_id": post_id, "content": comment.content}

# 2. READ (List/Collection)
@router.get("/", response_model=List[CommentResponse])
async def list_comments(post_id: int, limit: int = 10, skip: int = 0):
    # e.g., GET /posts/123/comments?limit=10&skip=0
    return [{"id": 1, "post_id": post_id, "content": "Great post!"}]

# 3. READ (Single Item)
@router.get("/{comment_id}", response_model=CommentResponse)
async def get_comment(post_id: int, comment_id: int):
    # e.g., GET /posts/123/comments/456
    return {"id": comment_id, "post_id": post_id, "content": "Specific comment"}

# 4. UPDATE (Partial)
@router.patch("/{comment_id}", response_model=CommentResponse)
async def update_comment(post_id: int, comment_id: int, comment: CommentCreate):
    # e.g., PATCH /posts/123/comments/456
    return {"id": comment_id, "post_id": post_id, "content": comment.content}

# 5. DELETE
@router.delete("/{comment_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_comment(post_id: int, comment_id: int):
    # e.g., DELETE /posts/123/comments/456
    # Returns 204 No Content on success
    pass
```

**Key Points:**
- **Nouns, not verbs:** Use `/posts/{id}/comments` instead of `/getCommentsForPost`.
- **Plurals:** Always use plural nouns (`/users`, `/posts`).
- **HTTP Status Codes:** `201 Created` for POST, `204 No Content` for DELETE.

---

### Q2: Pagination Implementation (Offset vs Cursor)

**Problem:** Implement Cursor-based pagination, which is necessary for large datasets where offset pagination (`skip`, `limit`) becomes too slow.

```python
from fastapi import APIRouter, Query
from pydantic import BaseModel
from typing import List, Optional
import base64

class Item(BaseModel):
    id: int
    name: str

class CursorPage(BaseModel):
    items: List[Item]
    next_cursor: Optional[str]

router = APIRouter()

# Mock database sorted by ID
db = [{"id": i, "name": f"Item {i}"} for i in range(1, 100)]

def encode_cursor(item_id: int) -> str:
    return base64.b64encode(str(item_id).encode()).decode()

def decode_cursor(cursor: str) -> int:
    return int(base64.b64decode(cursor).decode())

@router.get("/items", response_model=CursorPage)
async def get_items_cursor(
    limit: int = Query(10, le=100), 
    cursor: Optional[str] = None
):
    # 1. Decode cursor to get the starting ID
    start_id = decode_cursor(cursor) if cursor else 0
    
    # 2. Query database: WHERE id > start_id ORDER BY id ASC LIMIT limit
    # (Mock implementation)
    results = [item for item in db if item["id"] > start_id][:limit]
    
    # 3. Generate next cursor based on the last item in results
    next_cursor = None
    if len(results) == limit:
        last_item_id = results[-1]["id"]
        next_cursor = encode_cursor(last_item_id)
        
    return {"items": results, "next_cursor": next_cursor}
```

**Key Points:**
- **Offset Pagination (`OFFSET 10000 LIMIT 10`):** The database must scan and skip the first 10,000 rows. Terrible performance on large tables.
- **Cursor Pagination (`WHERE id > 10000 LIMIT 10`):** The database uses an index to jump directly to ID 10000. $O(1)$ performance.
- Cursors are typically base64 encoded so clients treat them as opaque strings and don't try to guess or modify them.

---

### Q3: Global Error Handling / Exception Middleware

**Problem:** Ensure standard error formats (JSON) are returned even when the application crashes unexpectedly.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
import logging

app = FastAPI()
logger = logging.getLogger("api")

class CustomDomainException(Exception):
    def __init__(self, message: str, code: str):
        self.message = message
        self.code = code

# 1. Catch specific domain exceptions
@app.exception_handler(CustomDomainException)
async def domain_exception_handler(request: Request, exc: CustomDomainException):
    return JSONResponse(
        status_code=400,
        content={"error": {"code": exc.code, "message": exc.message}},
    )

# 2. Catch ALL unhandled exceptions (Global Fallback)
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    # Log the full stack trace internally
    logger.error(f"Unhandled error: {exc}", exc_info=True)
    
    # Return a generic 500 without leaking stack traces to the client
    return JSONResponse(
        status_code=500,
        content={"error": {"code": "INTERNAL_SERVER_ERROR", "message": "An unexpected error occurred."}},
    )

@app.get("/trigger-error")
async def trigger_error():
    raise CustomDomainException(message="Invalid transaction", code="TX_INVALID")
```

**Key Points:**
- **Never leak stack traces:** A raw 500 HTML page or a JSON response containing stack traces is a massive security vulnerability.
- **Structured Errors:** Always return errors in a predictable format (e.g., `{"error": {"code": "...", "message": "..."}}`).

---

### Q4: Idempotency Keys

**Problem:** Prevent double-charging a user if their network drops and they retry a payment request.

```python
from fastapi import FastAPI, Header, HTTPException
import redis.asyncio as redis

app = FastAPI()
redis_client = redis.Redis(host='localhost', port=6379)

@app.post("/payments")
async def process_payment(
    amount: float, 
    # Client generates a unique UUID for this specific transaction attempt
    idempotency_key: str = Header(...)
):
    cache_key = f"idempotency:payment:{idempotency_key}"
    
    # 1. Check if we've already processed this exact request
    existing_result = await redis_client.get(cache_key)
    if existing_result:
        # Return the cached successful response immediately
        return {"status": "success", "message": "Payment already processed", "data": existing_result}
        
    # 2. Acquire a lock to prevent race conditions from aggressive concurrent retries
    lock_acquired = await redis_client.set(f"lock:{cache_key}", "1", nx=True, ex=5)
    if not lock_acquired:
        raise HTTPException(status_code=409, detail="Request already in progress")
        
    try:
        # 3. Perform the actual (expensive/risky) operation
        transaction_id = "TX_" + idempotency_key[:8]
        result_data = f"Charged ${amount} (TxID: {transaction_id})"
        
        # 4. Save the result for future retries (store for 24 hours)
        await redis_client.set(cache_key, result_data, ex=86400)
        
        return {"status": "success", "data": result_data}
    finally:
        # Release lock
        await redis_client.delete(f"lock:{cache_key}")
```

**Key Points:**
- **Idempotency:** An operation is idempotent if making multiple identical requests has the same effect as making a single request. (GET, PUT, DELETE are naturally idempotent; POST is not).
- Clients (frontend/mobile) generate a UUID and send it in the `Idempotency-Key` header.
- Redis is used to store the key and the final response. If the client retries, they get the saved response instead of executing the payment again.

---

### Q5: API Versioning Strategies

**Problem:** You need to introduce a breaking change to an API endpoint without breaking legacy mobile clients.

**Strategy 1: URL Versioning (Most Common & Recommended)**
```python
# Clear, explicit, cacheable
app.include_router(v1_router, prefix="/api/v1")
app.include_router(v2_router, prefix="/api/v2")

# GET /api/v1/users
# GET /api/v2/users
```

**Strategy 2: Header Versioning (Cleaner URLs)**
```python
from fastapi import FastAPI, Header, HTTPException

@app.get("/api/users")
async def get_users(accept_version: str = Header(default="1.0")):
    if accept_version == "1.0":
        return {"users": ["Alice", "Bob"]} # Legacy flat array
    elif accept_version == "2.0":
        return {"data": [{"name": "Alice"}, {"name": "Bob"}]} # New nested format
    else:
        raise HTTPException(status_code=400, detail="Unsupported API version")
```

**Which to choose?**
- **URL Versioning** is the industry standard (Stripe, GitHub, Twitter). It is easiest for developers to understand, easiest to document in Swagger, and easiest to route at the Load Balancer level (e.g., routing `/v1` traffic to legacy servers).
- Always default to URL versioning unless you have a strict constraint forcing Header versioning.
