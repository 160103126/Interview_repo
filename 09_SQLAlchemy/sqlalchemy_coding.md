# SQLAlchemy 2.0 — Coding Patterns

> Practical implementation of modern SQLAlchemy 2.0. Focuses on Asynchronous Sessions, Declarative `Mapped` typing, 1:N / M:N relationships, and solving the notorious N+1 Query Problem using `selectinload` and `joinedload`.

---

## Table of Contents

- [Q1: Modern Async Engine & Session Setup](#q1-modern-async-engine--session-setup)
- [Q2: Declarative Models (Mapped Types & 1:N Relationships)](#q2-declarative-models-mapped-types--1n-relationships)
- [Q3: Many-to-Many (M:N) Relationships](#q3-many-to-many-mn-relationships)
- [Q4: Solving the N+1 Query Problem](#q4-solving-the-n1-query-problem)
- [Q5: Basic CRUD Operations (Async)](#q5-basic-crud-operations-async)

---

### Q1: Modern Async Engine & Session Setup

**Problem:** Configure an asynchronous SQLAlchemy engine and session factory suitable for FastAPI.

```python
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import declarative_base

DATABASE_URL = "postgresql+asyncpg://user:password@localhost/dbname"

# 1. Create the Async Engine
engine = create_async_engine(
    DATABASE_URL,
    echo=True, # Logs generated SQL to stdout
    pool_size=10,
    max_overflow=20
)

# 2. Create the Session Factory
AsyncSessionLocal = async_sessionmaker(
    bind=engine,
    class_=AsyncSession,
    expire_on_commit=False # Crucial for async so attributes aren't lost after commit
)

# 3. Base class for declarative models
Base = declarative_base()

# 4. FastAPI Dependency
async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

#### 🧠 Architectural Walkthrough
- **`asyncpg` Driver:** You must explicitly use `+asyncpg` in the connection string because the default `psycopg2` is synchronous and blocks the entire Python event loop.
- **`expire_on_commit=False`:** In synchronous SQLAlchemy, committing a session marks objects as "expired". Accessing their attributes later triggers an automatic database query. In async SQLAlchemy, implicit I/O is forbidden (it raises a `MissingGreenlet` exception). Setting this to False prevents the crash.

---

### Q2: Declarative Models (Mapped Types & 1:N Relationships)

**Problem:** Create a User and Post model representing a One-to-Many relationship using SQLAlchemy 2.0 strict typing.

```python
from sqlalchemy import String, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship
from typing import List

class User(Base):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    username: Mapped[str] = mapped_column(String(50), unique=True)
    
    # 1:N Relationship
    posts: Mapped[List["Post"]] = relationship(back_populates="author", cascade="all, delete-orphan")

class Post(Base):
    __tablename__ = "posts"
    
    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    title: Mapped[str] = mapped_column(String(100))
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    
    # Back-reference
    author: Mapped["User"] = relationship(back_populates="posts")
```

#### 🧠 Architectural Walkthrough
- **SQLAlchemy 2.0 `Mapped[]`:** This replaces the old `Column(String)` syntax. It provides strict Python typing, allowing tools like mypy and IDEs to offer autocomplete and type checking natively.
- **`cascade="all, delete-orphan"`:** If a User is deleted, all associated Posts are automatically deleted by the ORM, maintaining referential integrity.

---

### Q3: Many-to-Many (M:N) Relationships

**Problem:** Create a Many-to-Many relationship between `Student` and `Course`.

```python
from sqlalchemy import Table, Column, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship
from typing import List

# Association Table (No declarative class needed for simple association)
student_course_table = Table(
    "student_course",
    Base.metadata,
    Column("student_id", ForeignKey("students.id"), primary_key=True),
    Column("course_id", ForeignKey("courses.id"), primary_key=True),
)

class Student(Base):
    __tablename__ = "students"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(50))
    
    courses: Mapped[List["Course"]] = relationship(
        secondary=student_course_table, back_populates="students"
    )

class Course(Base):
    __tablename__ = "courses"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(50))
    
    students: Mapped[List["Student"]] = relationship(
        secondary=student_course_table, back_populates="courses"
    )
```

#### 🧠 Architectural Walkthrough
- **The Association Table:** Relational databases cannot natively represent Many-to-Many connections. We must create a junction table that holds the foreign keys of both entities. We use the raw `Table` construct because this table holds no data of its own other than the keys.
- **`secondary`:** We pass the `student_course_table` into the `relationship` so SQLAlchemy knows which bridge table to use when performing JOINs.

---

### Q4: Solving the N+1 Query Problem

**Problem:** Fetch all Users and their Posts efficiently without causing the N+1 query problem.

```python
from sqlalchemy import select
from sqlalchemy.orm import selectinload, joinedload

async def get_users_with_posts_n_plus_1(session: AsyncSession):
    # BAD: Triggers N+1 problem if you try to access user.posts later
    result = await session.execute(select(User))
    return result.scalars().all()

async def get_users_with_posts_joined(session: AsyncSession):
    # GOOD for N:1 (Many-to-One)
    # Uses SQL LEFT OUTER JOIN (Brings back massive duplicate rows)
    stmt = select(User).options(joinedload(User.posts))
    result = await session.execute(stmt)
    return result.unique().scalars().all()

async def get_users_with_posts_selectin(session: AsyncSession):
    # BEST for 1:N (One-to-Many)
    # Uses a second SQL query with an IN clause: SELECT * FROM posts WHERE user_id IN (...)
    stmt = select(User).options(selectinload(User.posts))
    result = await session.execute(stmt)
    return result.scalars().all()
```

#### 🧠 Architectural Walkthrough
- **The N+1 Problem:** If you load 100 users, and then loop through them accessing `user.posts`, SQLAlchemy will issue 1 query for the users, and then 100 separate queries for the posts. This cripples database performance.
- **`joinedload`:** Issues a single `JOIN` query. However, if a user has 1000 posts, the User data is duplicated 1000 times in the SQL return payload, wasting network bandwidth. (Use this for Many-to-One, e.g., fetching a Post and its Author).
- **`selectinload`:** Issues exactly 2 queries. Query 1: Fetch users. Query 2: `SELECT * FROM posts WHERE user_id IN (1, 2, 3...)`. This is the most efficient method for 1:N and M:N relationships.

---

### Q5: Basic CRUD Operations (Async)

**Problem:** Perform an Async Insert, Update, and Delete.

```python
from sqlalchemy import select, update, delete

async def create_user(session: AsyncSession, username: str):
    new_user = User(username=username)
    session.add(new_user)
    await session.commit()
    await session.refresh(new_user) # Retrieves generated ID
    return new_user

async def update_username(session: AsyncSession, user_id: int, new_username: str):
    # Bulk Update (Faster than fetching and modifying)
    stmt = update(User).where(User.id == user_id).values(username=new_username)
    await session.execute(stmt)
    await session.commit()

async def delete_user(session: AsyncSession, user_id: int):
    stmt = delete(User).where(User.id == user_id)
    await session.execute(stmt)
    await session.commit()
```

#### 🧠 Architectural Walkthrough
- **Bulk Updates/Deletes:** Calling `update()` directly is much faster than `user = await session.get(User, id); user.username = "new"; await session.commit()`, as it avoids pulling the data into Python memory entirely.
