# SQLAlchemy 2.0+ — Theory & Concepts

> Modern SQLAlchemy paradigms, focusing on the 2.0 style syntax, asynchronous ORM, Declarative mapping, and the Session lifecycle.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is SQLAlchemy? Core vs ORM?

**Answer:**

SQLAlchemy is the premier SQL toolkit and Object-Relational Mapper (ORM) for Python.

It is split into two distinct APIs:
1. **Core:** A schema-centric, SQL Expression Language. You deal directly with tables, columns, and SQL syntax (like `select()`, `insert()`). It is fast and close to raw SQL.
2. **ORM (Object Relational Mapper):** A domain-centric view. You deal with Python classes (Models) and objects. The ORM translates object manipulations into Core SQL expressions.

*Note:* In SQLAlchemy 2.0, the ORM and Core APIs have been unified significantly, standardizing on the `select()` construct instead of the old ORM `session.query()`.

---

### Q2: How do you declare Models in SQLAlchemy 2.0?

**Answer:**

SQLAlchemy 2.0 heavily uses Python type hints for Declarative mapping (`Mapped[]` and `mapped_column()`).

```python
from sqlalchemy import String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

# 1. Base Class
class Base(DeclarativeBase):
    pass

# 2. Model Class
class User(Base):
    __tablename__ = "users"

    # Type hints define the column type and nullability
    id: Mapped[int] = mapped_column(primary_key=True)
    
    # Custom constraints via mapped_column
    username: Mapped[str] = mapped_column(String(30), unique=True)
    
    # Optional field (nullable=True is inferred from Optional)
    fullname: Mapped[str | None]
```

---

### Q3: What is the Engine and the Session?

**Answer:**

- **Engine:** The starting point. It manages a **Connection Pool** and the Dialect (which translates SQL to Postgres, MySQL, SQLite, etc.). It does *not* connect to the DB until an operation requires it.
- **Session:** The ORM's "workspace" for your objects. It manages the **Unit of Work** pattern. It tracks changes to objects and flushes them to the database in a single transaction when `commit()` is called.

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine("postgresql://user:pass@localhost/dbname")
SessionLocal = sessionmaker(bind=engine)

# Usage
with SessionLocal() as session:
    session.add(User(username="alice"))
    session.commit()
```

---

## 🟡 Medium (Intermediate)

### Q4: Explain the Unit of Work pattern and the Session Lifecycle.

**Answer:**

The `Session` tracks the state of every object attached to it.
When you modify an object, no SQL is sent to the database immediately.

**States:**
1. **Transient:** Object created in Python, not added to a session. (`user = User()`)
2. **Pending:** Added to session, but not yet flushed to the DB. (`session.add(user)`)
3. **Persistent:** Present in the session and has a corresponding row in the database.
4. **Detached:** Exists in Python, but the session that created it is closed.

**Flush vs Commit:**
- `session.flush()`: Sends the SQL statements (`INSERT`, `UPDATE`) to the database, but within the current transaction. (Useful if you need the auto-generated primary key ID before committing).
- `session.commit()`: Calls flush(), then actually commits the database transaction.

```mermaid
stateDiagram-v2
    [*] --> Transient: Object Instantiated
    Transient --> Pending: session.add(obj)
    Pending --> Persistent: session.flush() / commit()
    Persistent --> Deleted: session.delete(obj)
    Persistent --> Detached: session.close()
    Detached --> Persistent: session.add(obj)
```

---

### Q5: How do you query data in SQLAlchemy 2.0?

**Answer:**

The old `session.query(User).filter(...)` is considered legacy. The 2.0 style uses the `select` construct passed to `session.scalars()`.

```python
from sqlalchemy import select

with SessionLocal() as session:
    # 1. Build the statement
    stmt = select(User).where(User.username == "alice")
    
    # 2. Execute and get results
    # scalars() unpacks the row tuples so you get the actual User objects
    alice = session.scalars(stmt).first()
    
    # Get all users
    all_users = session.scalars(select(User)).all()
```

---

### Q6: How do you configure Relationships (One-to-Many)?

**Answer:**

Use `relationship()` and `ForeignKey`. In 2.0, use `Mapped[list[Child]]`.

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    
    # One-to-Many
    addresses: Mapped[list["Address"]] = relationship(back_populates="user")

class Address(Base):
    __tablename__ = "addresses"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    
    # Many-to-One
    user: Mapped["User"] = relationship(back_populates="addresses")
```

---

## 🔴 Hard (Advanced / MAANG-level)

### Q7: Explain Lazy Loading vs Eager Loading (N+1 Query Problem).

**Answer:**

**The N+1 Problem:**
```python
# 1 Query to get 100 users
users = session.scalars(select(User)).all()

for user in users:
    # 100 separate Queries executed here! (Lazy Loading)
    print(user.addresses) 
```
*Total queries: 1 (to get users) + 100 (to get addresses) = 101 queries.*

**Solution (Eager Loading):**
Tell SQLAlchemy to fetch the related data in the initial query.

```python
from sqlalchemy.orm import joinedload, selectinload

# Method 1: joinedload (Uses an SQL JOIN). Good for Many-to-One.
stmt = select(Address).options(joinedload(Address.user))

# Method 2: selectinload (Issues a second SELECT ... WHERE id IN (...)). 
# Best practice for One-to-Many/Many-to-Many.
stmt = select(User).options(selectinload(User.addresses))
```

---

### Q8: How does Asyncio work with SQLAlchemy 2.0?

**Answer:**

SQLAlchemy 1.4+ introduced an `asyncio` extension. You must use an async driver (like `asyncpg` for Postgres, not `psycopg2`).

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

# Note the +asyncpg in the URL
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/dbname")
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_user(username: str):
    async with AsyncSessionLocal() as session:
        stmt = select(User).where(User.username == username)
        
        # Must AWAIT the execution
        result = await session.execute(stmt)
        return result.scalars().first()
```

*Crucial Gotcha:* You cannot use Lazy Loading (`user.addresses`) in an async session, because lazy loading triggers an implicit synchronous SQL query, which blocks the event loop and raises a `MissingGreenlet` error. You **must** eagerly load relationships using `selectinload`.

---

### Q9: How do you handle massive inserts efficiently? (Bulk Operations)

**Answer:**

Using `session.add()` in a loop is extremely slow for thousands of rows due to ORM overhead.

**Method 1: ORM Bulk Insert (v2.0)**
```python
# Much faster than add(). Skips unit of work tracking.
session.execute(
    insert(User),
    [
        {"username": "user1", "fullname": "User One"},
        {"username": "user2", "fullname": "User Two"},
        # ... 10,000 more dicts
    ]
)
session.commit()
```

**Method 2: Returning IDs (PostgreSQL only)**
```python
stmt = insert(User).values([{"username": "u1"}, {"username": "u2"}]).returning(User.id)
result = session.scalars(stmt).all()
```

---

*End of SQLAlchemy Theory — 9 questions covering 2.0 syntax, type hints, Unit of Work, N+1 problem, and AsyncORM.*
