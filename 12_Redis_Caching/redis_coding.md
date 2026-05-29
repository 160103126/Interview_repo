# Redis and Caching — Coding Patterns

> Foundational patterns for integrating Redis into Python applications for caching, rate limiting, and pub/sub mechanisms.

---

## Table of Contents

- [Q1: Redis Connection and Basic CRUD](#q1-redis-connection-and-basic-crud)
- [Q2: Implement a Decorator-based Cache](#q2-implement-a-decorator-based-cache)
- [Q3: Sliding Window Rate Limiter](#q3-sliding-window-rate-limiter)
- [Q4: Redis Pub/Sub Implementation](#q4-redis-pubsub-implementation)
- [Q5: Distributed Lock using Redis](#q5-distributed-lock-using-redis)

---

### Q1: Redis Connection and Basic CRUD

**Problem:** Establish an asynchronous connection to Redis using `redis-py` and perform basic operations with expiration.

```python
import asyncio
import redis.asyncio as redis

async def redis_basics():
    # Establish connection pool
    redis_client = redis.Redis(
        host='localhost', 
        port=6379, 
        decode_responses=True # Automatically decodes bytes to strings
    )
    
    try:
        # 1. SET with expiration (TTL)
        # ex=60 means expire in 60 seconds
        await redis_client.set("user:101:session", "active", ex=60)
        
        # 2. GET
        val = await redis_client.get("user:101:session")
        print(f"Value: {val}") # Output: Value: active
        
        # 3. Check TTL
        ttl = await redis_client.ttl("user:101:session")
        print(f"Time to live: {ttl} seconds")
        
        # 4. HASH operations (storing objects/dicts)
        user_data = {"name": "Alice", "role": "admin"}
        await redis_client.hset("user:102:profile", mapping=user_data)
        
        # Retrieve hash
        retrieved_hash = await redis_client.hgetall("user:102:profile")
        print(f"Hash: {retrieved_hash}")
        
    finally:
        # Gracefully close connection pool
        await redis_client.aclose()

# asyncio.run(redis_basics())
```

**Key Points:**
- Always use `redis.asyncio` for modern FastAPI/async applications to prevent blocking the event loop.
- `decode_responses=True` is highly recommended for text data to avoid manual `.decode('utf-8')` calls on every read.

---

### Q2: Implement a Decorator-based Cache

**Problem:** Write a reusable Python decorator that caches the output of an async function in Redis, using the function's arguments to generate the cache key.

```python
import json
import hashlib
from functools import wraps
import redis.asyncio as redis

# Assuming a global redis_client exists in a real app
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def cache_response(ttl_seconds: int = 300):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # 1. Generate a unique cache key based on function name and arguments
            key_data = json.dumps({"args": args, "kwargs": kwargs}, sort_keys=True)
            key_hash = hashlib.md5(key_data.encode()).hexdigest()
            cache_key = f"cache:{func.__name__}:{key_hash}"
            
            # 2. Check if data exists in cache
            cached_result = await redis_client.get(cache_key)
            if cached_result:
                print(f"CACHE HIT for {cache_key}")
                return json.loads(cached_result)
                
            # 3. CACHE MISS: Execute the actual function
            print(f"CACHE MISS for {cache_key}")
            result = await func(*args, **kwargs)
            
            # 4. Store the result in Redis with TTL
            await redis_client.set(
                cache_key, 
                json.dumps(result), 
                ex=ttl_seconds
            )
            
            return result
        return wrapper
    return decorator

# Usage Example:
@cache_response(ttl_seconds=60)
async def fetch_database_records(user_id: int):
    # Simulate slow database query
    await asyncio.sleep(2)
    return {"user_id": user_id, "data": "expensive_query_result"}
```

**Key Points:**
- **Cache Invalidation:** This implementation uses TTL (Time-To-Live). The data automatically expires.
- **Serialization:** We use `json.dumps()` because Redis stores strings/bytes, not native Python dictionaries.

---

### Q3: Sliding Window Rate Limiter

**Problem:** Implement a sliding window rate limiter in Redis to prevent API abuse.

```python
import time
import redis.asyncio as redis

class RateLimiter:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def is_allowed(self, user_id: str, limit: int, window_seconds: int) -> bool:
        """
        Returns True if the user is allowed to make a request, False if rate limited.
        """
        key = f"ratelimit:{user_id}"
        now = time.time()
        
        # Use a Redis pipeline for atomicity
        async with self.redis.pipeline(transaction=True) as pipe:
            # 1. Add current request timestamp to the sorted set
            pipe.zadd(key, {str(now): now})
            
            # 2. Remove timestamps older than the window
            window_start = now - window_seconds
            pipe.zremrangebyscore(key, 0, window_start)
            
            # 3. Count total requests in the current window
            pipe.zcard(key)
            
            # 4. Set TTL on the key to clean up inactive users
            pipe.expire(key, window_seconds)
            
            # Execute pipeline
            results = await pipe.execute()
            
            # results[2] contains the output of zcard (the request count)
            current_requests = results[2]
            
            return current_requests <= limit

# Usage:
# limiter = RateLimiter(redis_client)
# allowed = await limiter.is_allowed("user_123", limit=10, window_seconds=60)
# if not allowed:
#     raise HTTPException(status_code=429, detail="Too Many Requests")
```

**Key Points:**
- Uses Redis **Sorted Sets (`ZSET`)**. The score is the timestamp.
- **Pipelines (`pipe.execute()`)** are critical. They send all commands to Redis in a single network round-trip and execute them atomically, preventing race conditions if the user makes rapid concurrent requests.

---

### Q4: Redis Pub/Sub Implementation

**Problem:** Implement a Publish/Subscribe mechanism for real-time messaging (e.g., chat app or event notification).

```python
import asyncio
import redis.asyncio as redis

async def publisher(redis_client: redis.Redis, channel: str):
    for i in range(3):
        await asyncio.sleep(1)
        message = f"Event update {i}"
        print(f"Publishing: {message}")
        await redis_client.publish(channel, message)

async def subscriber(redis_client: redis.Redis, channel: str):
    pubsub = redis_client.pubsub()
    await pubsub.subscribe(channel)
    print(f"Subscribed to {channel}")
    
    # Listen for messages
    async for message in pubsub.listen():
        if message['type'] == 'message':
            print(f"Received via PubSub: {message['data']}")
            # Break condition for this example
            if "2" in message['data']:
                break

async def main():
    redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
    
    # Run publisher and subscriber concurrently
    await asyncio.gather(
        subscriber(redis_client, "system_events"),
        publisher(redis_client, "system_events")
    )
    
    await redis_client.aclose()

# asyncio.run(main())
```

**Key Points:**
- **Fire and Forget:** Unlike message queues (RabbitMQ, Kafka), Redis Pub/Sub does NOT persist messages. If a subscriber is offline when a message is published, that message is lost forever.
- Use Pub/Sub for ephemeral real-time data (chat rooms, live dashboard updates). Use Redis Streams or Celery if you need guaranteed delivery.

---

### Q5: Distributed Lock using Redis

**Problem:** You have multiple worker instances pulling from a database, and you need to ensure only one instance executes a critical task at a time (Distributed Locking).

```python
import asyncio
import uuid
import redis.asyncio as redis

class DistributedLock:
    def __init__(self, redis_client: redis.Redis, lock_name: str, expire_seconds: int = 10):
        self.redis = redis_client
        self.lock_name = f"lock:{lock_name}"
        self.expire_seconds = expire_seconds
        self.lock_id = str(uuid.uuid4()) # Unique identifier for THIS process

    async def acquire(self) -> bool:
        # NX=True: Only set the key if it does NOT already exist
        # EX: Set expiration to prevent infinite deadlocks if process crashes
        acquired = await self.redis.set(
            self.lock_name, 
            self.lock_id, 
            nx=True, 
            ex=self.expire_seconds
        )
        return bool(acquired)

    async def release(self):
        # Lua script ensures atomic deletion ONLY IF we are the lock owner
        # Prevents Process A from deleting Process B's lock if A's lock expired
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        await self.redis.eval(lua_script, 1, self.lock_name, self.lock_id)

async def critical_section(worker_name: str):
    redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
    lock = DistributedLock(redis_client, "daily_report_generation")
    
    if await lock.acquire():
        try:
            print(f"{worker_name} acquired the lock! Executing task...")
            await asyncio.sleep(2) # Simulate work
        finally:
            await lock.release()
            print(f"{worker_name} released the lock.")
    else:
        print(f"{worker_name} failed to acquire lock. Task is already running elsewhere.")
        
    await redis_client.aclose()
```

**Key Points:**
- **`nx=True` (Set if Not eXists):** This is the heart of the lock. Redis guarantees this operation is atomic.
- **Expiration (`ex`):** Critical. If the server crashes while holding the lock, it will naturally expire, preventing a system-wide deadlock.
- **Lua Script for Release:** You must check that you still own the lock before deleting it. A Lua script executes this read-and-delete atomically inside Redis.
