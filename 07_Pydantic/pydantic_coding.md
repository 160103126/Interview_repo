# Pydantic — Coding Patterns

> Pydantic is the standard data validation library for modern Python (and FastAPI). These coding patterns cover advanced validation, serialization, and settings management for production systems.

---

## Table of Contents

- [Q1: Complex Nested Models and Validation](#q1-complex-nested-models-and-validation)
- [Q2: Custom Validators (Before and After)](#q2-custom-validators-before-and-after)
- [Q3: Computed Fields and Model Properties](#q3-computed-fields-and-model-properties)
- [Q4: Pydantic Settings Management](#q4-pydantic-settings-management)
- [Q5: Efficient Serialization (model_dump)](#q5-efficient-serialization-model_dump)

---

### Q1: Complex Nested Models and Validation

**Problem:** Create a robust user registration model that includes nested address information, strict email validation, and password strength requirements using Pydantic v2.

```python
from pydantic import BaseModel, EmailStr, Field, field_validator
from typing import List, Optional
import re

class Address(BaseModel):
    street: str = Field(..., min_length=5, max_length=100)
    city: str = Field(..., min_length=2)
    zip_code: str = Field(..., pattern=r"^\d{5}(-\d{4})?$")
    country: str = Field(default="USA")

class UserRegistration(BaseModel):
    username: str = Field(..., min_length=3, max_length=30)
    email: EmailStr
    password: str = Field(..., min_length=8, description="Must contain at least one special character")
    addresses: List[Address] = Field(min_length=1)  # Must have at least one address
    tags: List[str] = Field(default_factory=list)

    @field_validator('password')
    @classmethod
    def validate_password_strength(cls, v: str) -> str:
        if not re.search(r"[!@#$%^&*(),.?\":{}|<>]", v):
            raise ValueError('Password must contain at least one special character')
        if not any(char.isdigit() for char in v):
            raise ValueError('Password must contain at least one number')
        return v
```

**Key Points:**
- `EmailStr` requires the `pydantic[email]` extra package and provides RFC-compliant email validation.
- `Field(...)` explicitly marks a field as required and allows adding constraints (`min_length`, `pattern`).
- `default_factory=list` is required for mutable defaults like lists/dicts to prevent shared state across instances.

---

### Q2: Custom Validators (Before and After)

**Problem:** Demonstrate `mode="before"` validators for cleaning data before Pydantic parses it, and `model_validator` for validating relationships across multiple fields.

```python
from pydantic import BaseModel, field_validator, model_validator
from typing import Optional
from datetime import date

class EventBooking(BaseModel):
    event_name: str
    start_date: date
    end_date: date
    attendee_name: str

    # Clean data BEFORE Pydantic tries to parse it
    @field_validator("attendee_name", mode="before")
    @classmethod
    def clean_attendee_name(cls, v: str) -> str:
        if isinstance(v, str):
            return v.strip().title()
        return v

    # Validate relationships across multiple fields
    @model_validator(mode="after")
    def validate_dates(self) -> "EventBooking":
        if self.end_date < self.start_date:
            raise ValueError("end_date cannot be before start_date")
        return self
```

**Key Points:**
- `mode="before"` intercepts the raw input data. It is excellent for sanitizing inputs (stripping whitespace, normalizing strings) before strict type checking occurs.
- `@model_validator(mode="after")` operates on the fully parsed model instance, allowing you to compare fields (e.g., ensuring an end date follows a start date).

---

### Q3: Computed Fields and Model Properties

**Problem:** You have a model with a `first_name` and `last_name`. You want a `full_name` field that is automatically computed, included in the JSON output, and accurately reflected in the OpenAPI schema.

```python
from pydantic import BaseModel, computed_field

class UserProfile(BaseModel):
    first_name: str
    last_name: str
    age: int

    @computed_field
    @property
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"

    @computed_field(return_type=bool)
    @property
    def is_adult(self):
        return self.age >= 18
```

**Key Points:**
- In Pydantic v1, standard `@property` methods were not serialized into JSON by default.
- In Pydantic v2, `@computed_field` ensures the property is included in `model_dump()`, `model_dump_json()`, and the generated JSON Schema.

---

### Q4: Pydantic Settings Management

**Problem:** Load application configurations from environment variables and a `.env` file, prioritizing environment variables.

```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import PostgresDsn, SecretStr

class AppSettings(BaseSettings):
    # Model configuration for .env loading
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False
    )

    app_name: str = "My FastAPI App"
    environment: str = "development"
    
    # Validates valid Postgres connection strings
    database_url: PostgresDsn
    
    # SecretStr prevents accidental logging/printing of sensitive data
    api_key: SecretStr
    
    redis_port: int = 6379

# Instantiating automatically reads from ENV and .env
# settings = AppSettings()
# print(settings.api_key.get_secret_value()) # How to read a SecretStr
```

**Key Points:**
- Requires the `pydantic-settings` package (separated from core Pydantic in v2).
- `SecretStr` overrides the `__str__` and `__repr__` methods to output `**********`, preventing accidental leaks in logs. You must explicitly call `.get_secret_value()` to use the key.

---

### Q5: Efficient Serialization (model_dump)

**Problem:** Serialize a Pydantic model to a dictionary, excluding unset values and sensitive fields.

```python
from pydantic import BaseModel, Field

class Employee(BaseModel):
    id: int
    name: str
    department: str = "Engineering"
    salary: float = Field(exclude=True)  # Never include in standard dumps
    nickname: str | None = None

emp = Employee(id=1, name="Alice", salary=120000.0)

# 1. Standard dump (salary is excluded via Field config)
print(emp.model_dump())
# {'id': 1, 'name': 'Alice', 'department': 'Engineering', 'nickname': None}

# 2. Exclude unset values (defaults and None values that weren't explicitly provided)
print(emp.model_dump(exclude_unset=True))
# {'id': 1, 'name': 'Alice'}

# 3. Exclude defaults (removes 'department' because it matches the default)
print(emp.model_dump(exclude_defaults=True))
# {'id': 1, 'name': 'Alice', 'nickname': None}

# 4. Explicitly include/exclude at dump time
print(emp.model_dump(include={'name', 'department'}))
# {'name': 'Alice', 'department': 'Engineering'}
```

**Key Points:**
- In Pydantic v2, `dict()` and `json()` are deprecated in favor of `model_dump()` and `model_dump_json()`.
- `exclude_unset=True` is crucial for creating `PATCH` (partial update) API endpoints, ensuring you only update fields the user actually sent in the payload.
