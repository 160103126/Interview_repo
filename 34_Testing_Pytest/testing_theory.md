# Testing & Pytest — Theory & Concepts

> MAANG interviews for Backend/ML roles expect you to know how to write robust, deterministic tests for highly non-deterministic systems (like LLMs) and complex microservices.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: Explain the Testing Pyramid. What are the three layers?

**Answer:**

The Testing Pyramid is a visual strategy for balancing test types by their cost and speed.

1. **Unit Tests (Bottom — 70% of your tests):** Tests a single function or class in absolute isolation. Any database or external API call MUST be mocked. These are lightning fast (milliseconds) and cheap to maintain.
2. **Integration Tests (Middle — 20%):** Tests the interaction between two or more components (e.g., your FastAPI endpoint talking to a real PostgreSQL test database via SQLAlchemy). Slower than unit tests but validates that components work together.
3. **End-to-End / E2E Tests (Top — 10%):** Tests the entire system from the user's perspective (e.g., Selenium/Playwright clicking buttons on a UI, or hitting a fully deployed API). These are slow, brittle, and expensive.

**The Anti-Pattern (Ice Cream Cone):** Many teams write mostly E2E tests and very few unit tests. This results in extremely slow CI pipelines that take 45 minutes to run, discouraging developers from running tests at all.

---

### Q2: What are Pytest Fixtures? How do they differ from `unittest.setUp()`?

**Answer:**

Fixtures are the backbone of Pytest. They use **Dependency Injection** to provide test data, database connections, or mock objects to test functions.

```python
import pytest

@pytest.fixture
def sample_user():
    """Any test function can receive this by adding 'sample_user' as a parameter."""
    return {"name": "Alice", "email": "alice@example.com"}

def test_user_has_email(sample_user):
    assert "@" in sample_user["email"]
```

**How they differ from `unittest.setUp()`:**
- `setUp()` runs before *every* test in the class, even if that test doesn't need the setup data. Fixtures are only injected into tests that explicitly request them.
- Fixtures can be shared across files using `conftest.py`.
- Fixtures support `yield` for teardown logic (cleanup), which is cleaner than `tearDown()`.

---

### Q3: Explain Fixture Scopes in Pytest.

**Answer:**

Fixture scopes control how often a fixture is created and destroyed:

| Scope | Lifetime | Use Case |
|---|---|---|
| `function` (default) | Created and destroyed for *every* test function | Lightweight data, mocks |
| `class` | Once per test class | Shared state within a class |
| `module` | Once per test file (`.py`) | Database connection per file |
| `session` | Once for the entire test suite run | Spinning up a Dockerized test DB, expensive setup |

```python
@pytest.fixture(scope="session")
def db_engine():
    """Created once for the entire test run. Shared by ALL tests."""
    engine = create_engine("postgresql://test:test@localhost/testdb")
    yield engine
    engine.dispose()  # Teardown: runs after ALL tests complete
```

---

### Q4: What is `conftest.py`? Why is it important?

**Answer:**

`conftest.py` is a special file that Pytest discovers automatically. Fixtures defined in it are available to **all test files** in the same directory and subdirectories without needing to import them.

**Key Rules:**
- You can have multiple `conftest.py` files in nested directories for hierarchical fixture scoping.
- It is the standard place to put shared fixtures (like database sessions, test clients, auth tokens).
- Plugins and hooks can also be defined here.

```
tests/
├── conftest.py        # Fixtures available to ALL tests
├── unit/
│   ├── conftest.py    # Fixtures only for unit tests
│   └── test_models.py
└── integration/
    ├── conftest.py    # Fixtures only for integration tests
    └── test_api.py
```

---

## 🟡 Medium (Intermediate)

### Q5: Explain Mocking and Patching. What is the "Where to Patch" rule?

**Answer:**

- **Mocking:** Replacing a real object with a fake one (`MagicMock`) that tracks how it was used and returns pre-programmed responses. You test your code's *logic* without hitting real databases or APIs.
- **Patching:** Using `@patch()` or `with patch()` to temporarily replace an object during a test.

**The "Where to Patch" Rule (Critical for interviews):**
You must patch the object **where it is used**, NOT where it is defined.

```python
# my_app/service.py
import requests  # <-- Defined here

def get_weather(city):
    return requests.get(f"https://api.weather.com/{city}")

# tests/test_service.py
from unittest.mock import patch

# ❌ WRONG: Patching where it's defined
@patch("requests.get")

# ✅ CORRECT: Patching where it's used (in my_app.service)
@patch("my_app.service.requests.get")
def test_get_weather(mock_get):
    mock_get.return_value.status_code = 200
    result = get_weather("NYC")
    assert result.status_code == 200
    mock_get.assert_called_once()
```

---

### Q6: What is `pytest.mark.parametrize`? Why is it powerful?

**Answer:**

`parametrize` runs the **same test function multiple times** with different inputs and expected outputs. It replaces the need to write 10 nearly identical test functions.

```python
import pytest

@pytest.mark.parametrize("input_val, expected", [
    ("hello", 5),
    ("", 0),
    ("Python", 6),
    ("  ", 2),
])
def test_string_length(input_val, expected):
    assert len(input_val) == expected
```

**Why it's powerful:**
- Each parameter set is reported as a *separate* test case in the output (4 tests in the example above).
- If one fails, the others still run, giving you precise failure information.
- Excellent for testing edge cases (empty strings, None, negative numbers) systematically.

---

### Q7: How do you test for expected exceptions with `pytest.raises`?

**Answer:**

```python
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def test_divide_by_zero():
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)

def test_divide_normal():
    assert divide(10, 2) == 5.0
```

**Key Points:**
- `pytest.raises` acts as a context manager. If the expected exception is NOT raised inside the `with` block, the test **fails**.
- The `match` parameter accepts a regex to verify the error message content.
- This is critical for testing validation logic in Pydantic models and FastAPI endpoints.

---

### Q8: Explain `caplog`, `capsys`, and `tmp_path` built-in fixtures.

**Answer:**

Pytest provides powerful built-in fixtures for common testing needs:

**1. `capsys` — Capture stdout/stderr:**
```python
def test_print_output(capsys):
    print("hello world")
    captured = capsys.readouterr()
    assert captured.out == "hello world\n"
```

**2. `caplog` — Capture log messages:**
```python
import logging

def test_logging(caplog):
    logger = logging.getLogger("my_app")
    with caplog.at_level(logging.WARNING):
        logger.warning("disk space low")
    assert "disk space low" in caplog.text
```

**3. `tmp_path` — Temporary directory (auto-cleaned):**
```python
def test_file_creation(tmp_path):
    file = tmp_path / "output.txt"
    file.write_text("test data")
    assert file.read_text() == "test data"
    # Directory is automatically cleaned up after the test
```

---

## 🔴 Hard (Advanced / MAANG-level)

### Q9: How do you test async code with Pytest?

**Answer:**

Standard Pytest cannot run `async def` test functions. You need the `pytest-asyncio` plugin.

```python
import pytest
import httpx

@pytest.mark.asyncio
async def test_async_api_call():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://httpbin.org/get")
    assert response.status_code == 200

# For testing async fixtures:
@pytest.fixture
async def async_db_session():
    session = await create_async_session()
    yield session
    await session.close()

@pytest.mark.asyncio
async def test_create_user(async_db_session):
    user = await async_db_session.execute(select(User))
    assert user is not None
```

**Key Configuration (`pyproject.toml`):**
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"  # Automatically marks all async tests
```

---

### Q10: Explain Test Coverage. What is a good coverage target?

**Answer:**

**Test Coverage** measures the percentage of your source code that is executed during testing. The `pytest-cov` plugin generates coverage reports.

```bash
pytest --cov=my_app --cov-report=html tests/
```

**Coverage Metrics:**
- **Line Coverage:** Were all lines executed? (Most common)
- **Branch Coverage:** Were all `if/else` branches taken? (More rigorous — a function with an `if` and no `else` can have 100% line coverage but 50% branch coverage).

**Good Target:**
- **80-90% is excellent** for application code.
- **100% is a trap.** Chasing 100% leads to testing trivial getters/setters, creating maintenance burden without real value.
- Focus on **critical path coverage** — authentication, payment processing, data validation — not boilerplate.

---

### Q11: How do you test non-deterministic LLM outputs?

**Answer:**

Testing Generative AI is notoriously difficult because the output changes every time. Use a layered approach:

**Level 1: Mock the LLM (Unit Tests)**
For testing your application logic (routing, error handling), mock the LLM client to return a hardcoded string. This proves your chain routes correctly.

**Level 2: Deterministic Constraints (Integration Tests)**
Don't test the exact text. Test structural properties:
- Does the output conform to a Pydantic schema (JSON mode)?
- Does it contain required keywords?
- Is the response length within bounds?
- Does it refuse to answer out-of-scope questions?

**Level 3: LLM-as-a-Judge (Evaluation Tests)**
Run the actual LLM, then use a *second* LLM (like GPT-4) to grade the output against a rubric:
> "Score this response 1-5 on Helpfulness and Factual Accuracy. Output JSON."

**Level 4: Regression Test Suites (Golden Tests)**
Maintain a set of "golden" input-output pairs. After any prompt change, re-run the suite and compare outputs to catch regressions. Use semantic similarity (embedding distance) rather than exact string matching.

---

### Q12: Explain the `monkeypatch` fixture vs `unittest.mock.patch`. When to use which?

**Answer:**

Both replace objects during tests, but they have different philosophies:

**`monkeypatch` (Pytest-native):**
```python
def test_env_variable(monkeypatch):
    monkeypatch.setenv("API_KEY", "test-key-123")
    monkeypatch.setattr("my_app.config.DEBUG", True)
    # Changes are automatically reverted after the test
```

**`unittest.mock.patch`:**
```python
from unittest.mock import patch, MagicMock

@patch("my_app.service.openai_client")
def test_llm_call(mock_client):
    mock_client.chat.completions.create.return_value = MagicMock(
        choices=[MagicMock(message=MagicMock(content="Paris"))]
    )
    result = my_function()
    assert result == "Paris"
```

| Feature | `monkeypatch` | `unittest.mock.patch` |
|---|---|---|
| Best for | Environment vars, simple attribute replacement | Complex mock objects with call tracking |
| Call assertions | ❌ No `assert_called_once()` | ✅ Full call tracking (args, call count) |
| Scope | Pytest only | Works in unittest and pytest |
| Complexity | Simple | More powerful but verbose |

**Rule of thumb:** Use `monkeypatch` for simple replacements (env vars, configs). Use `mock.patch` when you need to assert *how* something was called.

---

*End of Testing & Pytest Theory — 12 comprehensive questions covering the Testing Pyramid, Fixtures, Mocking, Parametrize, Async testing, Coverage, LLM testing, and monkeypatch.*
