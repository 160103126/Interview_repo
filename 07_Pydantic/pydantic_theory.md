# Pydantic V2 — Theory & Concepts

> Data validation and settings management using Python type annotations. Focus is heavily on Pydantic V2 internals and Rust-based core.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is Pydantic and why is it used?

**Answer:**

Pydantic is a data validation and settings management library for Python using standard Python type hints.

**Why use it?**
1. **Type Coercion:** It forces input data to match defined types (e.g., converts string `"123"` to integer `123`).
2. **Data Validation:** Ensures data meets strict constraints before it enters your application.
3. **Serialization/Deserialization:** Easily converts complex objects to/from JSON.
4. **IDE Support:** Because it uses standard type hints, IDEs (VSCode/PyCharm) provide excellent autocompletion and linting out of the box.

---

### Q2: How does Pydantic V2 differ from V1?

**Answer:**

Pydantic V2 was a massive rewrite from the ground up:

1. **`pydantic-core` (Rust):** The core validation logic was rewritten in Rust, making V2 **5x to 50x faster** than V1 (which was pure Python/Cython).
2. **Strict Mode:** V2 introduced a `strict=True` mode that disables type coercion (e.g., `"123"` will raise a validation error instead of converting to `123` if the type is `int`).
3. **Model namespace:** Removed `__` and `_` prefixed methods like `model.dict()` and replaced them with `model_dump()`, freeing up the namespace for user fields.

---

### Q3: How do you define a basic Pydantic Model?

**Answer:**

```python
from pydantic import BaseModel, EmailStr, Field

class User(BaseModel):
    id: int
    name: str = Field(..., max_length=50)  # ... means required
    email: EmailStr
    is_active: bool = True  # Default value makes it optional in input

# Deserialization (Validation)
user = User(id="1", name="Alice", email="alice@example.com")
print(user.id)  # Output: 1 (Coerced from string to int)

# Serialization
print(user.model_dump()) 
# {'id': 1, 'name': 'Alice', 'email': 'alice@example.com', 'is_active': True}
print(user.model_dump_json())
# '{"id":1,"name":"Alice","email":"alice@example.com","is_active":true}'
```

---

## 🟡 Medium (Intermediate)

### Q4: Explain the `Field` function in Pydantic. What is it used for?

**Answer:**

`Field` is used to customize and add metadata/constraints to model fields that cannot be expressed with type hints alone.

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    # Validation constraints
    price: float = Field(gt=0, description="Price must be greater than zero")
    tags: list[str] = Field(default_factory=list, max_length=10)
    
    # Aliasing (parsing camelCase JSON to snake_case Python)
    stock_count: int = Field(alias="stockCount", default=0)

# Input uses the alias
prod = Product(price=19.99, stockCount=100)
# Access uses the python attribute
print(prod.stock_count)  # 100
```
*Note: `default_factory` is crucial for mutable defaults like lists or dicts to avoid sharing state between instances.*

---

### Q5: How do you handle custom validation in Pydantic V2?

**Answer:**

Pydantic V2 provides `@field_validator` (for single fields) and `@model_validator` (for the whole model).

```python
from pydantic import BaseModel, field_validator, model_validator

class UserRegistration(BaseModel):
    username: str
    password: str
    confirm_password: str

    @field_validator('username')
    @classmethod  # Validators are class methods in V2
    def username_alphanumeric(cls, v: str) -> str:
        if not v.isalnum():
            raise ValueError('Username must be alphanumeric')
        return v

    @model_validator(mode='after')
    def check_passwords_match(self) -> 'UserRegistration':
        # mode='after' means this runs after individual fields are validated
        if self.password != self.confirm_password:
            raise ValueError('Passwords do not match')
        return self
```

---

### Q6: What is Pydantic `BaseSettings`? (Now `pydantic-settings`)

**Answer:**

It's a specialized model used for environment variable management. It automatically reads from OS environment variables and `.env` files.

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class AppConfig(BaseSettings):
    database_url: str
    api_key: str
    debug_mode: bool = False

    # V2 way to specify env file config
    model_config = SettingsConfigDict(env_file='.env', env_file_encoding='utf-8')

# It will automatically look for DATABASE_URL, API_KEY in the environment
config = AppConfig()
print(config.database_url)
```

---

## 🔴 Hard (Advanced / MAANG-level)

### Q7: Explain Pydantic V2's execution model. How does `pydantic-core` bridge Python and Rust?

**Answer:**

In Pydantic V1, building models and validating data happened entirely in Python space, resulting in significant overhead from function calls and type checking loops.

In **V2**:
1. When you define a `BaseModel` class in Python, Pydantic inspects the type annotations.
2. It generates a **CoreSchema** (a declarative dictionary describing exactly how validation should happen).
3. This `CoreSchema` is passed down to `pydantic-core` (the Rust extension).
4. Rust compiles this schema into a highly optimized **SchemaValidator** struct.
5. When you instantiate `User(id=1)`, Python hands the raw kwargs `{"id": 1}` to the Rust `SchemaValidator`.
6. Rust performs the validation, coercion, and error generation entirely natively, bypassing the Python interpreter.
7. Finally, Rust constructs the Python object and returns it.

This architecture means the validation loop runs at C/Rust speeds.

---

### Q8: What are `RootModel`s and when do you use them?

**Answer:**

A `RootModel` (replacing `__root__` from V1) is used when the JSON payload is not a dictionary/object, but a primitive type (like a raw list or string) that needs validation.

```python
from pydantic import RootModel

# Suppose the API expects a raw list of integers: [1, 2, 3]
# Standard BaseModel expects a dict: {"numbers": [1, 2, 3]}

class NumberList(RootModel[list[int]]):
    pass

data = NumberList([1, 2, 3])
print(data.root)  # [1, 2, 3]

# Deserializing from JSON array
data2 = NumberList.model_validate_json('[4, 5, 6]')
```

---

### Q9: How do you use `Annotated` with Pydantic V2 to create reusable types?

**Answer:**

Python 3.9+ introduced `typing.Annotated`, which allows attaching metadata to types. Pydantic V2 leverages this heavily to separate validation logic from model definition, making types highly reusable.

```python
from typing import Annotated
from pydantic import BaseModel, Field, StringConstraints

# Create a reusable custom type
# Must be 3 chars, uppercase, alphanumeric
CurrencyCode = Annotated[
    str, 
    StringConstraints(min_length=3, max_length=3, to_upper=True, pattern=r'^[A-Z]+$')
]

# Use it in multiple models without rewriting the constraints
class Transaction(BaseModel):
    amount: float
    currency: CurrencyCode

class BankAccount(BaseModel):
    balance: float
    base_currency: CurrencyCode

t = Transaction(amount=100.0, currency="usd")
print(t.currency)  # "USD" (coerced to upper)
```

---

*End of Pydantic Theory — 9 questions covering V2 updates, validation, execution architecture, and advanced type annotations.*
