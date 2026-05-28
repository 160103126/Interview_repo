# Testing & Pytest — Coding Patterns

> Practical implementation of Pytest for FastAPI and LLM applications. Focuses on Fixtures, Mocking API calls, and overriding FastAPI dependencies.

---

## Table of Contents

- [Q1: Basic Pytest Fixtures](#q1-basic-pytest-fixtures)
- [Q2: Mocking an External API (or LLM)](#q2-mocking-an-external-api-or-llm)
- [Q3: Testing FastAPI Endpoints (TestClient)](#q3-testing-fastapi-endpoints-testclient)
- [Q4: Overriding FastAPI Dependencies](#q4-overriding-fastapi-dependencies)

---

### Q1: Basic Pytest Fixtures

**Problem:** Create a reusable database connection for multiple tests using Pytest Fixtures.

```python
import pytest

class DummyDB:
    def connect(self): return "Connected"
    def close(self): return "Closed"

# The fixture provides the DB instance to any test that requests it
@pytest.fixture(scope="function")
def db_session():
    db = DummyDB()
    print("Setup: ", db.connect())
    
    yield db  # Pauses here, passes 'db' to the test
    
    print("Teardown: ", db.close())  # Resumes here after test finishes

def test_database_connection(db_session):
    # db_session is automatically injected by Pytest
    assert db_session is not None
```

#### 🧠 Architectural Walkthrough
- **Dependency Injection:** We don't call `db_session()`. We just put it in the test function arguments. Pytest automatically finds the fixture with that name, executes it up to the `yield`, passes the yielded value into the test, and then executes the teardown code after the test finishes (even if the test fails!).

---

### Q2: Mocking an External API (or LLM)

**Problem:** Test a function that calls OpenAI without actually spending money or needing internet access.

```python
import pytest
from unittest.mock import patch, MagicMock
from my_app.llm_service import generate_summary # Assume this calls OpenAI internally

def test_generate_summary_success():
    # Patch the exact location where `ChatOpenAI` is used in your code
    with patch("my_app.llm_service.ChatOpenAI") as MockLLM:
        # Create a fake response object
        mock_response = MagicMock()
        mock_response.content = "This is a mocked summary."
        
        # Configure the mock class to return our fake response when invoked
        mock_instance = MockLLM.return_value
        mock_instance.invoke.return_value = mock_response
        
        # Execute the actual function
        result = generate_summary("Read this long text...")
        
        # Assertions
        assert result == "This is a mocked summary."
        mock_instance.invoke.assert_called_once()
```

#### 🧠 Architectural Walkthrough
- **The Target:** We patch `"my_app.llm_service.ChatOpenAI"`, not `"langchain.ChatOpenAI"`. We must intercept the object in the specific namespace where our business logic is importing and executing it.
- **`MagicMock`:** A powerful object that tracks how it was called and allows you to deeply nest fake return values (e.g., `mock.invoke().content`).

---

### Q3: Testing FastAPI Endpoints (TestClient)

**Problem:** Test a FastAPI route without spinning up a live uvicorn server.

```python
from fastapi.testclient import TestClient
from my_app.main import app # Import your FastAPI instance

client = TestClient(app)

def test_read_main():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}

def test_create_item():
    response = client.post(
        "/items/",
        headers={"X-Token": "secret"},
        json={"id": "foo", "title": "Foo Item"},
    )
    assert response.status_code == 201
    assert response.json() == {"id": "foo", "title": "Foo Item"}
```

#### 🧠 Architectural Walkthrough
- **`TestClient`:** Based on the `httpx` library. It bypasses the network entirely and passes HTTP requests directly into the FastAPI application's ASGI interface. This makes endpoint testing incredibly fast.

---

### Q4: Overriding FastAPI Dependencies

**Problem:** Your FastAPI endpoint depends on a real PostgreSQL database. How do you test the endpoint using a mock database instead?

```python
from fastapi.testclient import TestClient
from my_app.main import app, get_db

# 1. Create a fake dependency
async def override_get_db():
    yield "Fake DB Session"

# 2. Tell FastAPI to swap the real dependency for the fake one
app.dependency_overrides[get_db] = override_get_db

client = TestClient(app)

def test_get_users_uses_fake_db():
    response = client.get("/users/")
    assert response.status_code == 200
    # The endpoint used our fake DB instead of connecting to Postgres!
    assert response.json() == {"db_status": "Fake DB Session"}
    
# 3. Clean up the override after the test
app.dependency_overrides.clear()
```

#### 🧠 Architectural Walkthrough
- **`dependency_overrides`:** This is the killer feature of FastAPI. Because FastAPI uses Dependency Injection (`Depends()`), the framework maintains a dictionary of all dependencies. We can swap them out at runtime during tests to inject mock databases, bypass authentication, or bypass rate limiters.
