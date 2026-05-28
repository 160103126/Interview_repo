# FastAPI — Theory & Concepts

> High-performance async web framework built on Starlette and Pydantic. Covers Dependency Injection, concurrency, middleware, and architecture.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What makes FastAPI "Fast"? (Architecture)

**Answer:**

FastAPI is fast because it stands on the shoulders of two giants:
1. **Starlette:** A lightweight, high-performance ASGI (Asynchronous Server Gateway Interface) framework/toolkit. It handles the web routing, requests, and responses asynchronously.
2. **Pydantic:** Handles the data validation and serialization. (Especially in V2, utilizing Rust for core validation).

Because it uses ASGI and `asyncio`, it can handle thousands of concurrent connections efficiently without blocking the thread, making it comparable in speed to NodeJS or Go for I/O bound tasks.

---

### Q2: How do you declare Path, Query, and Body parameters?

**Answer:**

FastAPI infers the location of the parameter based on how it's declared in the function signature.

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

# 1. PATH Parameter (in the URL path string)
# 2. QUERY Parameter (not in the path string, has a default/type)
# 3. BODY Parameter (uses a Pydantic model)

@app.post("/items/{item_id}")
async def create_item(
    item_id: int,           # Path parameter (matches URL {item_id})
    q: str = None,          # Query parameter (default makes it optional: ?q=test)
    item: Item = None       # Body parameter (Because it's a Pydantic model)
):
    return {"item_id": item_id, "q": q, "item_name": item.name}
```

---

### Q3: What is Dependency Injection (DI) in FastAPI?

**Answer:**

DI is a pattern where the framework provides the necessary objects/functions to your endpoint, rather than the endpoint creating them itself.

FastAPI uses the `Depends()` function.

```python
from fastapi import FastAPI, Depends

app = FastAPI()

# 1. Define the dependency (can be a function or class)
def pagination_params(skip: int = 0, limit: int = 10):
    return {"skip": skip, "limit": limit}

# 2. Inject it using Depends()
@app.get("/items/")
async def read_items(params: dict = Depends(pagination_params)):
    return params
```
**Benefits:**
- **Code reuse:** Use the same DB connection, auth logic, or pagination across 50 endpoints.
- **Testing:** Easy to override dependencies during unit testing (using `app.dependency_overrides`).

---

## 🟡 Medium (Intermediate)

### Q4: Explain `async def` vs `def` in FastAPI route handlers.

**Answer:**

FastAPI handles both smoothly, but they execute differently under the hood:

- **`async def`**: Runs directly on the main **asyncio event loop**. You *must* use this if you are using `await` inside the function (e.g., async DB driver, async HTTP client). If you put blocking sync code (like `time.sleep()`) in an `async def`, **you will freeze the entire server**.
  
- **`def` (Synchronous)**: FastAPI detects it is not async. To prevent blocking the main event loop, FastAPI automatically runs this function in an external **threadpool**. You use this for blocking I/O (like standard SQLAlchemy, `requests` library, or CPU heavy math).

*Rule of Thumb:* If you don't know, use `def`. If your library supports `await`, use `async def`.

---

### Q5: What is Middleware and how do you write one?

**Answer:**

Middleware is code that executes for **every single request** before it reaches your route handler, and for **every single response** before it returns to the client.

*Use cases:* CORS, logging, timing requests, injecting headers.

```python
from fastapi import FastAPI, Request
import time

app = FastAPI()

@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    start_time = time.time()
    
    # 1. Code here runs BEFORE the request reaches the endpoint
    response = await call_next(request)
    
    # 2. Code here runs AFTER the endpoint returns the response
    process_time = time.time() - start_time
    response.headers["X-Process-Time"] = str(process_time)
    
    return response
```

---

### Q6: How does FastAPI handle Authentication? (OAuth2 with Password Flow)

**Answer:**

FastAPI provides built-in security utilities (`fastapi.security`). The most common is `OAuth2PasswordBearer`.

```python
from fastapi import Depends, FastAPI, HTTPException
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()

# This tells FastAPI where the client should send the username/password
# to get a token. It also hooks into the auto-generated Swagger UI.
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

# This is a dependency. It extracts the "Authorization: Bearer <token>" header
@app.get("/users/me")
async def read_users_me(token: str = Depends(oauth2_scheme)):
    # Validate the JWT token here
    if token != "valid_secret_token":
        raise HTTPException(status_code=401, detail="Invalid token")
    return {"token": token}
```

---

## 🔴 Hard (Advanced / MAANG-level)

### Q7: Explain BackgroundTasks. How are they different from Celery?

**Answer:**

`BackgroundTasks` allows you to run a function asynchronously *after* returning a response to the client.

```python
from fastapi import BackgroundTasks, FastAPI

app = FastAPI()

def write_log(message: str):
    with open("log.txt", "a") as f:
        f.write(message)

@app.post("/send-notification/{email}")
async def send_notification(email: str, background_tasks: BackgroundTasks):
    # Returns 200 OK immediately, runs write_log in the background
    background_tasks.add_task(write_log, f"Sent email to {email}")
    return {"message": "Notification scheduled"}
```

**FastAPI BackgroundTasks vs Celery/RabbitMQ:**
- **FastAPI Tasks:** Run in the same memory space, in the same event loop/threadpool. If the FastAPI server crashes/restarts, the task is **lost permanently**. Good for trivial things (logging, simple emails).
- **Celery:** Runs in a separate worker process, backed by a persistent message broker (RabbitMQ/Redis). If a worker crashes, the task remains in the queue and is retried. Mandatory for heavy processing, reliable delivery, or scheduled cron jobs.

---

### Q8: How do you yield dependencies with teardown logic?

**Answer:**

Dependencies can use the `yield` keyword instead of `return`. This allows you to execute "teardown" code *after* the endpoint has finished processing the request (similar to a context manager).

*Crucial for Database connections to ensure they are closed!*

```python
from fastapi import Depends, FastAPI

def get_db():
    print("1. Opening DB Connection")
    db = "DatabaseConnectionObject"
    try:
        # Pauses here, passes 'db' to the endpoint
        yield db 
    finally:
        # Executes after the endpoint is finished (or if it crashes)
        print("3. Closing DB Connection")

app = FastAPI()

@app.get("/")
def read_root(db=Depends(get_db)):
    print("2. Endpoint executing")
    return {"status": "ok"}

# Output:
# 1. Opening DB Connection
# 2. Endpoint executing
# 3. Closing DB Connection
```

---

### Q9: What is ASGI vs WSGI? What servers do you use to run FastAPI in production?

**Answer:**

- **WSGI (Web Server Gateway Interface):** The old standard (Django, Flask, Gunicorn). It is strictly **synchronous**. It processes one request per thread/worker. If a request sleeps for 1 second, that worker does nothing else for 1 second.
- **ASGI (Asynchronous Server Gateway Interface):** The modern standard. It handles requests via an event loop, allowing a single worker to process thousands of concurrent I/O-bound requests.

**Production Deployment:**
You do not run FastAPI using `python main.py` in production. You need an ASGI server.
- **Uvicorn:** A fast ASGI server based on `uvloop`.
- **Gunicorn:** A process manager. 

*Best Practice:* Run Gunicorn as the process manager, and tell it to use Uvicorn worker classes. This gives you multiple processes (utilizing multiple CPU cores) where each process runs a highly efficient async event loop.
`gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker`

---

*End of FastAPI Theory — 9 questions covering architecture, DI, concurrency, and ASGI production deployments.*
