# API Design & RESTful Principles

> Best practices for designing robust, scalable, and developer-friendly APIs. Covers REST semantics, versioning, pagination, and Webhooks vs Polling.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What are the core principles of REST (Representational State Transfer)?

**Answer:**

REST is an architectural style, not a strict protocol.

1. **Client-Server Separation:** The UI (client) and data storage (server) are independent.
2. **Statelessness:** Each request from client to server must contain *all* the information needed to understand the request. The server does not store session state (e.g., uses JWT tokens instead of server-side sessions).
3. **Cacheability:** Responses must define themselves as cacheable or not to prevent clients from reusing stale data.
4. **Uniform Interface:** Resources are identified in requests (URIs) and manipulated through standard HTTP methods.

---

### Q2: Explain HTTP Methods and their semantics (Idempotency).

**Answer:**

- **GET:** Retrieve a resource. **Idempotent** (Calling it 100 times has the same effect as 1 time). **Safe** (Does not modify data).
- **POST:** Create a new resource. **NOT Idempotent** (Calling it twice creates two resources).
- **PUT:** Replace a resource entirely. **Idempotent** (Replacing it with the same data 10 times is the same as 1 time).
- **PATCH:** Partially update a resource. **NOT strictly Idempotent** (e.g., `PATCH {"increment_view_count": 1}`), though it can be depending on implementation.
- **DELETE:** Delete a resource. **Idempotent** (Deleting it once removes it; subsequent calls return 404, but the end state is the same).

---

### Q3: How should you design resource URIs?

**Answer:**

- **Use Nouns, not Verbs:**
  - ❌ `POST /createUser`
  - ✅ `POST /users`
- **Use Plurals:**
  - ❌ `GET /user/123`
  - ✅ `GET /users/123`
- **Nesting for Relationships (up to 2 levels):**
  - ✅ `GET /users/123/orders` (Orders belonging to user 123)
  - ❌ `GET /users/123/orders/456/items/789` (Too deep. Just use `/orders/456/items`)

---

## 🟡 Medium (Intermediate)

### Q4: Explain the common HTTP Status Codes.

**Answer:**

- **2xx (Success):**
  - `200 OK`: Standard success.
  - `201 Created`: Resource successfully created (used for POST).
  - `204 No Content`: Success, but no body to return (used for DELETE).
- **3xx (Redirection):**
  - `301 Moved Permanently`.
  - `304 Not Modified`: Client's cached version is still valid.
- **4xx (Client Error):**
  - `400 Bad Request`: Malformed JSON or validation error.
  - `401 Unauthorized`: Missing or invalid authentication token.
  - `403 Forbidden`: Authenticated, but lacks permissions (admin only).
  - `404 Not Found`: Resource doesn't exist.
  - `429 Too Many Requests`: Rate limit exceeded.
- **5xx (Server Error):**
  - `500 Internal Server Error`: Your code crashed.
  - `502 Bad Gateway`: Proxy/Load Balancer issue.
  - `503 Service Unavailable`: Server overloaded or down for maintenance.

---

### Q5: How do you handle API Versioning?

**Answer:**

APIs evolve. If you change a response payload, you break existing mobile apps. You must version your API.

1. **URI Versioning (Most Common, Developer Friendly):**
   - `https://api.example.com/v1/users`
   - Easy to test in browser. Used by Stripe, Twitter.
2. **Header Versioning:**
   - `Accept: application/vnd.example.v1+json`
   - Cleaner URLs, but harder to test and debug using basic tools. Used by GitHub.
3. **Query Parameter:**
   - `https://api.example.com/users?version=1`
   - Rare, usually discouraged.

---

### Q6: What are the different ways to paginate data?

**Answer:**

**1. Offset/Limit Pagination:**
- `/users?limit=10&offset=20`
- *Pros:* Easy to implement (SQL `LIMIT 10 OFFSET 20`), supports jumping to a specific page.
- *Cons:* Terrible performance on large datasets (DB must scan and skip the first N rows). Data drift (if an item is inserted/deleted while paging, items might be skipped or duplicated).

**2. Cursor/Keyset Pagination (Best Practice for large data):**
- `/users?limit=10&after=user_id_456`
- *Pros:* Excellent performance (SQL `WHERE id > 456 LIMIT 10` uses an index). Immune to data drift.
- *Cons:* Cannot jump to "Page 50". Can only go "Next" or "Previous". Used by Facebook, Slack, Twitter.

---

## 🔴 Hard (Advanced / MAANG-level)

### Q7: Explain Webhooks vs Long Polling vs Server-Sent Events (SSE) vs WebSockets.

**Answer:**

How do you get real-time updates from a server?

1. **Short Polling:** Client asks server every 5 seconds "Any updates?". 
   - *Cons:* Terrible efficiency, 99% of requests return empty.
2. **Long Polling:** Client asks, server *holds the connection open* until there is data, then responds. Client immediately asks again.
   - *Pros:* Better than short polling. Works over old infrastructure.
3. **Server-Sent Events (SSE):** Unidirectional connection (Server -> Client). Client connects once, server streams events.
   - *Pros:* Standard HTTP, built-in reconnection, great for stock tickers or live feeds.
4. **WebSockets:** Bi-directional persistent TCP connection.
   - *Pros:* Low latency chat, multiplayer games. 
   - *Cons:* Hard to load balance, stateful connections.
5. **Webhooks (Server to Server):** Instead of Client asking Server A, Client gives Server A an HTTP URL. When data is ready, Server A makes a POST request to that URL. 
   - *Pros:* The most efficient way for two backend servers to communicate asynchronously (e.g., Stripe payment success callbacks).

---

### Q8: How do you design an asynchronous, long-running API endpoint?

**Answer:**

If generating a PDF report takes 30 seconds, a standard HTTP request will time out. You must design it asynchronously using the **Polling Pattern** or **Webhook Pattern**.

**The Polling Design:**
1. **POST `/reports`**: Client requests report generation.
2. Server immediately returns `202 Accepted` with a Location header: `Location: /reports/123/status`.
3. Server hands task to a background worker (Celery/Kafka).
4. Client polls `GET /reports/123/status`.
   - If not done, server returns `200 OK: {"status": "processing"}`.
   - If done, server returns `303 See Other` with `Location: /reports/123/download`.
5. Client makes `GET /reports/123/download` to get the file.

---

### Q9: What is GraphQL? How does it compare to REST?

**Answer:**

GraphQL is a query language developed by Facebook.

| Feature | REST | GraphQL |
|---------|------|---------|
| **Endpoints** | Multiple (`/users`, `/posts`) | Single (`/graphql`) |
| **Over-fetching** | Common (get entire user object just for name) | None (Client specifies exactly which fields they want) |
| **Under-fetching**| Common (requires multiple requests to get user + posts) | None (Nested queries fetch all related data in one request) |
| **Caching** | Easy (Leverages HTTP caching) | Hard (Everything is a POST request to the same URL) |
| **Complexity** | Simple | Complex server implementation (N+1 query problems are common) |

---

*End of API Design Theory — 9 questions covering REST semantics, pagination, versioning, async patterns, and GraphQL.*
