# Design Patterns in Python

> Comprehensive guide to gang-of-four design patterns implemented in Pythonic ways, avoiding Java-isms and utilizing Python's dynamic features.

---

## Table of Contents

- [🟢 Creational Patterns](#-creational-patterns)
- [🟡 Structural Patterns](#-structural-patterns)
- [🔴 Behavioral Patterns](#-behavioral-patterns)
- [⚡ Python-Specific Patterns](#-python-specific-patterns)

---

## 🟢 Creational Patterns

### Q1: Implement the Singleton pattern. What are the pros and cons?

**Answer:**

Singleton ensures a class has only one instance and provides a global point of access.

**Method 1: Using `__new__` (Standard way)**
```python
class Singleton:
    _instance = None
    
    def __new__(cls, *args, **kwargs):
        if not cls._instance:
            cls._instance = super().__new__(cls)
        return cls._instance

s1 = Singleton()
s2 = Singleton()
print(s1 is s2)  # True
```

**Method 2: Using a Metaclass (Cleaner for multiple singletons)**
```python
class SingletonMeta(type):
    _instances = {}
    
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    pass
```

**Method 3: The Pythonic Way (Module-level Singleton)**
```python
# database.py
class Database:
    def connect(self): pass

# Instantiate once at module level
db = Database()

# main.py
# from database import db
# db is guaranteed to be a singleton because modules are only imported once
```

**Pros:** Controlled access, lazy initialization.
**Cons:** Global state, hard to unit test (hidden dependencies), violates Single Responsibility Principle.

---

### Q2: Explain the Factory Method pattern.

**Answer:**

Factory Method provides an interface for creating objects in a superclass, but allows subclasses to alter the type of objects that will be created.

```python
from abc import ABC, abstractmethod

# 1. Product Interface
class Document(ABC):
    @abstractmethod
    def show(self): pass

# 2. Concrete Products
class PDFDocument(Document):
    def show(self): return "Showing PDF"

class WordDocument(Document):
    def show(self): return "Showing Word Doc"

# 3. Creator
class DocumentCreator(ABC):
    @abstractmethod
    def create_document(self) -> Document:
        pass
    
    def render(self):
        # Creator uses the factory method
        doc = self.create_document()
        return f"Rendering: {doc.show()}"

# 4. Concrete Creators
class PDFCreator(DocumentCreator):
    def create_document(self) -> Document:
        return PDFDocument()

class WordCreator(DocumentCreator):
    def create_document(self) -> Document:
        return WordDocument()

creator = PDFCreator()
print(creator.render())  # Rendering: Showing PDF
```

**When to use:** When you don't know beforehand the exact types of objects your code should work with.

---

### Q3: How do you implement the Builder pattern?

**Answer:**

Builder lets you construct complex objects step by step. It allows you to produce different types and representations of an object using the same construction code.

```python
class Query:
    def __init__(self):
        self.select = ""
        self.from_tbl = ""
        self.where_cond = ""
    
    def __str__(self):
        return f"SELECT {self.select} FROM {self.from_tbl} WHERE {self.where_cond}"

class QueryBuilder:
    def __init__(self):
        self.query = Query()
    
    def select(self, fields: str):
        self.query.select = fields
        return self  # Return self for method chaining
    
    def from_table(self, table: str):
        self.query.from_tbl = table
        return self
    
    def where(self, condition: str):
        self.query.where_cond = condition
        return self
    
    def build(self) -> Query:
        return self.query

# Usage with method chaining
q = (QueryBuilder()
     .select("id, name")
     .from_table("users")
     .where("age > 18")
     .build())

print(q)  # SELECT id, name FROM users WHERE age > 18
```

**When to use:** When constructing an object requires many steps, many parameters (preventing "Telescoping Constructor" anti-pattern), or when you want method chaining.

---

## 🟡 Structural Patterns

### Q4: Implement the Adapter pattern.

**Answer:**

Adapter allows objects with incompatible interfaces to collaborate.

```python
# Old incompatible system
class EuropeanSocket:
    def voltage(self): return 230
    def live(self): return 1
    def neutral(self): return -1

# Target Interface expected by client
class USASocket:
    def voltage(self): return 110

# The Adapter
class SocketAdapter(USASocket):
    def __init__(self, socket: EuropeanSocket):
        self._socket = socket
    
    def voltage(self):
        # Convert voltage
        return 110

def use_usa_appliance(socket: USASocket):
    if socket.voltage() > 110:
        print("Boom! Appliance destroyed.")
    else:
        print("Appliance working perfectly.")

euro = EuropeanSocket()
adapter = SocketAdapter(euro)
use_usa_appliance(adapter)  # Appliance working perfectly.
```

**When to use:** Integrating third-party libraries or legacy code without changing their source code.

---

### Q5: What is the Decorator pattern and how does it relate to Python's `@decorator`?

**Answer:**

The GOF Decorator pattern attaches new behaviors to objects dynamically. Python's `@decorator` syntax is a built-in language feature that implements this pattern elegantly for functions and classes.

**GOF Object Decorator:**
```python
class Coffee:
    def cost(self): return 5
    def desc(self): return "Simple Coffee"

class MilkDecorator(Coffee):
    def __init__(self, coffee: Coffee):
        self._coffee = coffee
    def cost(self): return self._coffee.cost() + 2
    def desc(self): return self._coffee.desc() + ", Milk"

my_coffee = MilkDecorator(Coffee())
print(my_coffee.cost())  # 7
```

**Pythonic Function Decorator:**
```python
import functools

def log_execution(func):
    @functools.wraps(func)  # Preserves metadata (__name__, __doc__)
    def wrapper(*args, **kwargs):
        print(f"Executing {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished {func.__name__}")
        return result
    return wrapper

@log_execution
def process_data():
    pass
```

**When to use:** To add responsibilities to individual objects/functions without subclassing.

---

### Q6: Implement the Facade pattern.

**Answer:**

Facade provides a simplified interface to a complex subsystem.

```python
# Complex Subsystem parts
class CPU:
    def freeze(self): pass
    def jump(self, position): pass
    def execute(self): pass

class Memory:
    def load(self, position, data): pass

class HardDrive:
    def read(self, lba, size): return "boot_sector"

# The Facade
class ComputerFacade:
    def __init__(self):
        self.cpu = CPU()
        self.memory = Memory()
        self.hd = HardDrive()
    
    def start(self):
        """Simplifies the startup process for the client."""
        self.cpu.freeze()
        self.memory.load(0, self.hd.read(0, 1024))
        self.cpu.jump(0)
        self.cpu.execute()

# Client code
computer = ComputerFacade()
computer.start()
```

**When to use:** To provide a simple entry point to a complex subsystem (e.g., hiding a massive third-party API behind a simple interface).

---

## 🔴 Behavioral Patterns

### Q7: Implement the Observer pattern (Pub-Sub).

**Answer:**

Observer lets you define a subscription mechanism to notify multiple objects about any events that happen to the object they're observing.

```python
class Subject:
    def __init__(self):
        self._observers = []
        self._state = None
    
    def attach(self, observer):
        self._observers.append(observer)
    
    def detach(self, observer):
        self._observers.remove(observer)
    
    def notify(self):
        for observer in self._observers:
            observer.update(self._state)
    
    @property
    def state(self):
        return self._state
    
    @state.setter
    def state(self, value):
        self._state = value
        self.notify()

class Observer:
    def __init__(self, name):
        self.name = name
    
    def update(self, state):
        print(f"Observer {self.name} received state: {state}")

subject = Subject()
obs1 = Observer("A")
obs2 = Observer("B")

subject.attach(obs1)
subject.attach(obs2)

subject.state = "Active"
# Observer A received state: Active
# Observer B received state: Active
```

**When to use:** When changes to one object require changing others, and you don't know how many objects need to be changed (e.g., event listeners, UI data binding).

---

### Q8: Explain the Strategy pattern. How does Python simplify it?

**Answer:**

Strategy defines a family of algorithms, encapsulates each one, and makes them interchangeable.

**Standard OOP way:**
```python
from abc import ABC, abstractmethod

class PaymentStrategy(ABC):
    @abstractmethod
    def pay(self, amount: int): pass

class CreditCardPayment(PaymentStrategy):
    def pay(self, amount: int):
        print(f"Paid {amount} using Credit Card")

class ShoppingCart:
    def __init__(self, strategy: PaymentStrategy):
        self.strategy = strategy
    
    def checkout(self, amount):
        self.strategy.pay(amount)
```

**The Pythonic Way (using first-class functions):**
Because Python treats functions as first-class objects, you don't need classes for strategies!

```python
def credit_card_strategy(amount: int):
    print(f"Paid {amount} using Credit Card")

def paypal_strategy(amount: int):
    print(f"Paid {amount} using PayPal")

class ShoppingCart:
    def __init__(self, strategy_func):
        self.strategy_func = strategy_func
    
    def checkout(self, amount):
        self.strategy_func(amount)

cart = ShoppingCart(paypal_strategy)
cart.checkout(100)
```

**When to use:** When you have multiple algorithms for a specific task and want to switch between them at runtime.

---

### Q9: Implement the Command pattern.

**Answer:**

Command turns a request into a stand-alone object containing all information about the request.

```python
from abc import ABC, abstractmethod

# 1. The Command interface
class Command(ABC):
    @abstractmethod
    def execute(self): pass
    @abstractmethod
    def undo(self): pass

# 2. Receiver (does the actual work)
class Light:
    def turn_on(self): print("Light is ON")
    def turn_off(self): print("Light is OFF")

# 3. Concrete Commands
class TurnOnLightCommand(Command):
    def __init__(self, light: Light):
        self.light = light
    def execute(self):
        self.light.turn_on()
    def undo(self):
        self.light.turn_off()

# 4. Invoker (asks the command to carry out the request)
class RemoteControl:
    def __init__(self):
        self.history = []
    
    def execute_command(self, command: Command):
        command.execute()
        self.history.append(command)
    
    def undo_last(self):
        if self.history:
            cmd = self.history.pop()
            cmd.undo()

# Usage
light = Light()
cmd = TurnOnLightCommand(light)
remote = RemoteControl()

remote.execute_command(cmd)  # Light is ON
remote.undo_last()           # Light is OFF
```

**When to use:** GUI buttons, multi-level undo/redo operations, queuing tasks, macro recording.

---

## ⚡ Python-Specific Patterns

### Q10: What is Dependency Injection and how is it used in Python (e.g., FastAPI)?

**Answer:**

Dependency Injection (DI) is passing dependencies to an object/function rather than having it create them itself. This makes code highly testable.

**Without DI:**
```python
class UserService:
    def __init__(self):
        # Hardcoded dependency! Hard to mock.
        self.db = MySQLDatabase()
    
    def get_user(self, id):
        return self.db.query(id)
```

**With DI:**
```python
class UserService:
    def __init__(self, db_connection):
        # Dependency is injected. Easy to pass MockDB for testing.
        self.db = db_connection
```

**FastAPI's DI System:**
FastAPI has a first-class DI system that resolves dependencies recursively before routing.

```python
from fastapi import Depends, FastAPI

app = FastAPI()

# The dependency
def get_db_session():
    db = DatabaseSession()
    try:
        yield db
    finally:
        db.close()

# The consumer (FastAPI injects the dependency automatically)
@app.get("/users/{id}")
def read_user(id: int, db: DatabaseSession = Depends(get_db_session)):
    return db.query_user(id)
```

---

### Q11: Explain the Context Manager pattern (`with` statement).

**Answer:**

Context Managers encapsulate resource acquisition and release, guaranteeing cleanup even if exceptions occur. It replaces the `try...finally` block.

**Class-based approach:**
```python
class DatabaseConnection:
    def __enter__(self):
        print("Connecting to DB...")
        self.conn = "DB_CONN"
        return self.conn
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Closing DB connection...")
        if exc_type:
            print(f"Exception occurred: {exc_val}")
        return False  # False = propagate exception, True = swallow exception

with DatabaseConnection() as conn:
    print(f"Using {conn}")
```

**Generator-based approach (using contextlib):**
```python
from contextlib import contextmanager

@contextmanager
def database_connection():
    print("Connecting to DB...")
    conn = "DB_CONN"
    try:
        yield conn
    finally:
        print("Closing DB connection...")

with database_connection() as conn:
    print(f"Using {conn}")
```

---

### Q12: What is the Borg pattern (Monostate)? How does it differ from Singleton?

**Answer:**

The Borg pattern (coined by Alex Martelli) ensures that all instances of a class share the same state, rather than restricting the class to a single instance.

```python
class Borg:
    _shared_state = {}  # Shared dictionary for all instances
    
    def __init__(self):
        # Assign the instance dict to the shared dict
        self.__dict__ = self._shared_state

b1 = Borg()
b2 = Borg()

b1.val = "Shared"
print(b2.val)  # "Shared"

print(b1 is b2)  # False! They are different objects sharing state.
```

**Singleton vs Borg:**
- **Singleton:** Focuses on *object identity* (only one object exists).
- **Borg:** Focuses on *object state* (many objects exist, but they all share the exact same state). Borg is often considered more Pythonic because it avoids overriding `__new__` and subclassing issues.

---

*End of Design Patterns — 12 comprehensive questions on standard GOF and Python-specific patterns.*
