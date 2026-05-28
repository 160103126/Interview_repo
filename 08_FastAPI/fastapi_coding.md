# FastAPI — Coding Patterns

> Practical implementation patterns for FastAPI, focusing on real-world scenarios, dependency injection, middleware, error handling, background tasks, file uploads, and websockets. Every solution includes a deep architectural explanation and complexity/trade-off analysis.

---

## Table of Contents

- [Q1: Basic CRUD API (In-Memory)](#q1-basic-crud-api-in-memory)
- [Q2: Path vs Query Parameters](#q2-path-vs-query-parameters)
- [Q3: Custom HTTP Status Codes & Headers](#q3-custom-http-status-codes--headers)
- [Q4: Global Exception Handling](#q4-global-exception-handling)
- [Q5: File Uploads & Form Data](#q5-file-uploads--form-data)
- [Q6: Dependency Injection (Database Sessions)](#q6-dependency-injection-database-sessions)
- [Q7: Authentication (JWT Bearer Token)](#q7-authentication-jwt-bearer-token)
- [Q8: Background Tasks](#q8-background-tasks)
- [Q9: Middleware Configuration](#q9-middleware-configuration)
- [Q10: WebSockets](#q10-websockets)
- [Q11: Sub-Applications (Mounting)](#q11-sub-applications-mounting)
- [Q12: APIRouter Modularization](#q12-apirouter-modularization)

---

### Q1: Basic CRUD API (In-Memory)

**Problem:** Create a basic CRUD API for a "Task" resource using in-memory storage.

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List

app = FastAPI()

class Task(BaseModel):
    id: int
    title: str
    completed: bool = False

tasks_db: List[Task] = []

@app.post("/tasks/", response_model=Task, status_code=201)
def create_task(task: Task):
    if any(t.id == task.id for t in tasks_db):
        raise HTTPException(status_code=400, detail="Task already exists")
    tasks_db.append(task)
    return task

@app.get("/tasks/", response_model=List[Task])
def get_tasks():
    return tasks_db

@app.get("/tasks/{task_id}", response_model=Task)
def get_task(task_id: int):
    for task in tasks_db:
        if task.id == task_id:
            return task
    raise HTTPException(status_code=404, detail="Task not found")
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Pydantic Validation:** The `Task` class extends `BaseModel`. FastAPI automatically uses this to validate incoming JSON payloads and generate OpenAPI (Swagger) documentation.
- **Response Models:** Defining `response_model=Task` filters out internal data that shouldn't be exposed and explicitly casts the return types to match the JSON schema.

#### ⏱️ Complexity
- **Time Complexity:** $O(N)$ for insertion check and retrieval (list iteration).
- **Trade-off:** In-memory lists wipe on restart and fail in multi-worker Gunicorn environments.

---

### Q2: Path vs Query Parameters

**Problem:** Define an endpoint that uses both a path parameter and a query parameter.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str | None = None):
    if q:
        return {"item_id": item_id, "q": q}
    return {"item_id": item_id}
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **FastAPI Magic:** FastAPI infers that `item_id` is a **Path Parameter** because it appears in the decorator `@app.get("/items/{item_id}")`. It infers that `q` is a **Query Parameter** (e.g., `?q=search`) because it is in the function signature but *not* in the decorator string.

#### ⏱️ Complexity
- **Time/Space Complexity:** $O(1)$

---

### Q3: Custom HTTP Status Codes & Headers

**Problem:** How do you return a custom HTTP status code and custom headers dynamically?

```python
from fastapi import FastAPI, Response, status

app = FastAPI()

@app.post("/custom-response/")
def custom_response(response: Response):
    response.status_code = status.HTTP_202_ACCEPTED
    response.headers["X-Custom-Header"] = "My-Value"
    return {"message": "Accepted and processing"}
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **The `Response` Object:** By declaring a parameter of type `Response`, FastAPI passes the HTTP response object into your function *before* it is sent to the user, allowing dynamic mutation of headers and status codes based on internal logic.

#### ⏱️ Complexity
- **Time/Space Complexity:** $O(1)$

---

### Q4: Global Exception Handling

**Problem:** Implement Global Exception Handling to standardize all error responses.

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

app = FastAPI()

class CustomAppException(Exception):
    def __init__(self, message: str, error_code: str):
        self.message = message
        self.error_code = error_code

@app.exception_handler(CustomAppException)
async def custom_exception_handler(request: Request, exc: CustomAppException):
    return JSONResponse(
        status_code=400,
        content={"error_code": exc.error_code, "message": exc.message},
    )

@app.get("/trigger-error")
def trigger_error():
    raise CustomAppException(message="Insufficient funds", error_code="ERR_001")
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Standardizing Payloads:** A global exception handler acts as middleware. It catches custom exceptions raised deep in your business logic and formats them into a strict, predictable JSON schema, saving frontend developers from handling disparate error shapes.

#### ⏱️ Complexity
- **Time/Space Complexity:** $O(1)$

---

### Q5: File Uploads & Form Data

**Problem:** Accept a file upload along with form data.

```python
from fastapi import FastAPI, File, UploadFile, Form

app = FastAPI()

@app.post("/upload/")
async def upload_file(description: str = Form(...), file: UploadFile = File(...)):
    contents = await file.read() # Reads asynchronously
    return {"filename": file.filename, "description": description}
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **`UploadFile` vs `bytes`:** `UploadFile` uses Python's `SpooledTemporaryFile`. It stores data in memory up to 1MB, then transparently saves the rest to disk. Using `bytes` forces the entire file into RAM, crashing the server on large uploads (OOM).

#### ⏱️ Complexity
- **Time Complexity:** $O(N)$ where N is file size (I/O bound).
- **Space Complexity:** $O(1)$ RAM (spooled to disk).

---

### Q6: Dependency Injection (Database Sessions)

**Problem:** Inject a database session into an endpoint using FastAPI's `Depends`.

```python
from fastapi import FastAPI, Depends

app = FastAPI()

# Dummy DB
def get_db():
    db = "Database Connection"
    try:
        yield db
    finally:
        print("Closing Database Connection")

@app.get("/users/")
def get_users(db: str = Depends(get_db)):
    return {"db_session": db, "users": ["Alice", "Bob"]}
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Inversion of Control:** `Depends()` tells FastAPI to execute `get_db()` before running the route. 
- **Generators in Dependencies:** The `yield` statement pauses the dependency. The route executes. Once the route returns, FastAPI resumes `get_db()` at the `finally` block, ensuring DB connections are securely closed even if the route crashes.

#### ⏱️ Complexity
- **Time/Space Complexity:** $O(1)$ overhead.

---

### Q7: Authentication (JWT Bearer Token)

**Problem:** Protect an endpoint using OAuth2 with Password (and Bearer tokens).

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def get_current_user(token: str = Depends(oauth2_scheme)):
    if token != "fake-super-secret-token":
        raise HTTPException(status_code=401, detail="Invalid token")
    return {"username": "admin"}

@app.get("/secure-data/")
def read_secure_data(user: dict = Depends(get_current_user)):
    return {"data": "Top Secret", "user": user["username"]}
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Swagger UI Integration:** `OAuth2PasswordBearer` automatically injects the "Authorize" button into your auto-generated `/docs`. It extracts the `Authorization: Bearer <token>` header from incoming requests and passes the raw string to the dependency.

#### ⏱️ Complexity
- **Time Complexity:** $O(1)$ (Real JWT decoding takes $O(\text{token length})$ cryptographic time).

---

### Q8: Background Tasks

**Problem:** Send an email in the background without blocking the HTTP response.

```python
from fastapi import FastAPI, BackgroundTasks
import time

app = FastAPI()

def write_notification(email: str, message=""):
    time.sleep(2)  # Simulating a slow network call
    print(f"Sent email to {email}")

@app.post("/send-notification/{email}")
def send_notification(email: str, background_tasks: BackgroundTasks):
    background_tasks.add_task(write_notification, email, message="Hello")
    return {"message": "Notification scheduled in the background"}
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Non-blocking Execution:** The route returns `{"message": ...}` instantly (in ~2ms). Only *after* the HTTP response is dispatched does FastAPI execute the queued background tasks in the same event loop.
- **Trade-off:** This is lightweight but lacks persistence. If the server crashes during `time.sleep(2)`, the email is lost forever. For critical tasks, use Celery or Redis Queue.

#### ⏱️ Complexity
- **Time Complexity:** $O(1)$ HTTP response time.

---

### Q9: Middleware Configuration

**Problem:** Add a middleware that calculates the time taken to process each request.

```python
from fastapi import FastAPI, Request
import time

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    return response
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **The Onion Architecture:** Middleware wraps the entire application. `call_next(request)` plunges the request deep into routers, dependencies, and endpoints. The returned `response` bubbles back up, allowing you to intercept it and append universal headers (like CORS or Execution Time).

#### ⏱️ Complexity
- **Time Complexity:** $O(1)$ overhead.

---

### Q10: WebSockets

**Problem:** Implement a WebSocket endpoint that broadcasts messages to all clients.

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import List

app = FastAPI()

class ConnectionManager:
    def __init__(self):
        self.active: List[WebSocket] = []

    async def connect(self, ws: WebSocket):
        await ws.accept()
        self.active.append(ws)

    def disconnect(self, ws: WebSocket):
        self.active.remove(ws)

    async def broadcast(self, msg: str):
        for connection in self.active:
            await connection.send_text(msg)

manager = ConnectionManager()

@app.websocket("/ws/{client_id}")
async def websocket_endpoint(websocket: WebSocket, client_id: str):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            await manager.broadcast(f"Client {client_id}: {data}")
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Persistent Connections:** Unlike REST, WebSockets keep the TCP connection open indefinitely. The `while True:` loop waits asynchronously (`await`) for incoming text, blocking only that specific connection context without blocking the server.
- **Trade-off:** An in-memory connection array (`self.active`) fails in multi-pod Kubernetes setups. Production requires a Redis Pub/Sub backplane.

#### ⏱️ Complexity
- **Time Complexity:** $O(N)$ for broadcast (N is active connections).

---

### Q11: Sub-Applications (Mounting)

**Problem:** Mount two independent FastAPI apps together for versioning (V1 and V2).

```python
from fastapi import FastAPI

app = FastAPI(title="Main")
app_v1 = FastAPI(title="V1 API")
app_v2 = FastAPI(title="V2 API")

@app_v1.get("/users")
def get_users_v1(): return {"version": "v1"}

@app_v2.get("/users")
def get_users_v2(): return {"version": "v2"}

app.mount("/v1", app_v1)
app.mount("/v2", app_v2)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Complete Isolation:** Unlike `APIRouter` (which shares middleware), sub-mounting creates entirely independent ASGI apps. V2 can have strictly different authentication middleware and exception handlers than V1 without bleeding context.

#### ⏱️ Complexity
- **Time/Space Complexity:** $O(1)$ overhead.

---

### Q12: APIRouter Modularization

**Problem:** Split a massive `main.py` into separate routing modules.

```python
# users.py
from fastapi import APIRouter
router = APIRouter(prefix="/users", tags=["users"])

@router.get("/")
def get_users(): return ["Alice", "Bob"]

# main.py
from fastapi import FastAPI
# from users import router as users_router

app = FastAPI()
app.include_router(router)
```

#### 🧠 Algorithmic Walkthrough & Explanation
- **Microservice Structure:** The `APIRouter` acts as a "mini-FastAPI" instance. By using `prefix="/users"`, you avoid repeating `/users` in every route decorator within the module, keeping the code DRY.

#### ⏱️ Complexity
- **Time/Space Complexity:** $O(1)$ overhead.

---

*End of FastAPI Coding — 12 practical patterns spanning CRUD, Global Exceptions, Sub-mounting, Dependency Injection, and WebSockets.*
