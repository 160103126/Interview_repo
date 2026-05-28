# Testing & Pytest — Theory

> MAANG interviews for Backend/ML roles expect you to know how to write robust, deterministic tests for highly non-deterministic systems (like LLMs) and complex microservices.

---

## Table of Contents
1. [The Testing Pyramid](#1-the-testing-pyramid)
2. [Pytest Architecture](#2-pytest-architecture)
3. [Mocking & Patching](#3-mocking--patching)
4. [Testing LLMs (Non-Deterministic Code)](#4-testing-llms-non-deterministic-code)

---

### 1. The Testing Pyramid

1. **Unit Tests (Bottom - 70%):** Tests a single function or class in absolute isolation. Any database or external API call MUST be mocked. These are lightning fast.
2. **Integration Tests (Middle - 20%):** Tests the interaction between two or more components (e.g., your FastAPI endpoint talking to a real PostgreSQL test database).
3. **End-to-End / E2E Tests (Top - 10%):** Tests the entire system from the user's perspective (e.g., Selenium/Playwright clicking buttons on a UI). These are slow and brittle.

### 2. Pytest Architecture

* **Fixtures:** The backbone of Pytest. Unlike Python `unittest` which uses `setUp()` and `tearDown()`, Pytest uses dependency injection. A fixture provides setup data (like a DB connection) and injects it into any test function that asks for it.
* **Scope:** Fixtures can be scoped:
  * `function` (default): Setup/Teardown runs for *every* test.
  * `module`: Runs once per test file.
  * `session`: Runs once for the entire test suite (perfect for spinning up a Dockerized test database).

### 3. Mocking & Patching

* **What is Mocking?** Replacing a real object with a fake one that tracks how it was used and returns pre-programmed responses.
* **Why Patch?** If function A calls `requests.get()`, you cannot test function A without the internet. You use `@patch('requests.get')` to intercept the call.
* **The "Where to Patch" Rule:** You must patch the object *where it is used*, not where it is defined. If `my_app.service` imports `requests`, you patch `my_app.service.requests.get`, NOT `requests.get`.

### 4. Testing LLMs (Non-Deterministic Code)

Testing Generative AI is notoriously difficult because the output changes every time.
* **Level 1: Mock the LLM.** For unit testing your application logic, mock the `OpenAI` client to return a hardcoded string. This proves your chain routes correctly, but doesn't test the prompt quality.
* **Level 2: Deterministic Constraints.** Test if the LLM output conforms to a Pydantic schema or contains specific keywords.
* **Level 3: LLM-as-a-Judge.** For integration testing, run the LLM, and then use a *second* LLM (like GPT-4) to grade the output against a rubric. (See `Evaluation` section).
