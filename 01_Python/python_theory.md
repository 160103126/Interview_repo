# Python Theory — Interview Questions & Answers

> Covers core Python concepts, OOP, memory management, GIL, decorators, generators, context managers, type hints, and more. Difficulty: Simple → Medium → Hard (MAANG-level).

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What are the key differences between Python 2 and Python 3?

**Answer:**

| Feature | Python 2 | Python 3 |
|---------|----------|----------|
| Print | `print "hello"` (statement) | `print("hello")` (function) |
| Integer Division | `5/2 = 2` | `5/2 = 2.5`, `5//2 = 2` |
| Unicode | ASCII by default, `u"..."` for Unicode | Unicode by default |
| `range()` | Returns a list | Returns an iterator (lazy) |
| `xrange()` | Exists | Removed (range is lazy) |
| `input()` | `raw_input()` for strings | `input()` always returns string |
| Exceptions | `except Exception, e:` | `except Exception as e:` |
| `dict.keys()` | Returns list | Returns view object |

Python 2 reached End of Life on January 1, 2020. All new projects must use Python 3.12+.

---

### Q2: What are Python's built-in data types?

**Answer:**

Python has the following built-in data types:

| Category | Types | Mutable? |
|----------|-------|----------|
| **Numeric** | `int`, `float`, `complex`, `bool` | ❌ Immutable |
| **Sequence** | `list` | ✅ Mutable |
| | `tuple` | ❌ Immutable |
| | `range` | ❌ Immutable |
| **Text** | `str` | ❌ Immutable |
| **Binary** | `bytes` | ❌ Immutable |
| | `bytearray` | ✅ Mutable |
| | `memoryview` | ✅ Mutable |
| **Set** | `set` | ✅ Mutable |
| | `frozenset` | ❌ Immutable |
| **Mapping** | `dict` | ✅ Mutable |
| **None** | `NoneType` | ❌ Immutable |

Key insight: **Immutable objects can be used as dictionary keys and set elements** because their hash values don't change.

```python
# Valid — immutable keys
d = {(1, 2): "tuple key", "hello": "str key", 42: "int key"}

# Invalid — mutable keys
d = {[1, 2]: "list key"}  # TypeError: unhashable type: 'list'
```

---

### Q3: Explain the difference between `is` and `==`.

**Answer:**

- `==` checks **value equality** — do two objects have the same value?
- `is` checks **identity** — are two variables pointing to the **same object in memory**?

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)   # True — same values
print(a is b)   # False — different objects in memory
print(a is c)   # True — same object (c is an alias)

# Verify with id()
print(id(a), id(b))  # Different memory addresses
print(id(a), id(c))  # Same memory address
```

**Gotcha — Integer Caching (Interning):**

CPython caches small integers (-5 to 256) and short strings. This means `is` can return `True` for small integers even though you'd expect separate objects:

```python
x = 256
y = 256
print(x is y)  # True — cached/interned

x = 257
y = 257
print(x is y)  # False — not cached (in most contexts)
```

**Best Practice:** Always use `==` for value comparison. Use `is` only for `None` checks: `if x is None`.

---

### Q4: What are mutable and immutable objects? Why does it matter?

**Answer:**

**Mutable** objects can be changed in-place after creation. **Immutable** objects cannot — any "modification" creates a new object.

```python
# Mutable — list
lst = [1, 2, 3]
lst.append(4)        # Modifies in-place, same object
print(id(lst))       # Same id before and after

# Immutable — string
s = "hello"
s_new = s + " world"  # Creates a NEW string object
print(id(s), id(s_new))  # Different ids
```

**Why it matters:**

1. **Function arguments:** Mutable defaults are shared across calls (a famous Python gotcha):
```python
# BUG: Default list shared across calls
def add_item(item, lst=[]):
    lst.append(item)
    return lst

print(add_item(1))  # [1]
print(add_item(2))  # [1, 2] — unexpected!

# FIX: Use None sentinel
def add_item(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```

2. **Dictionary keys:** Only immutable (hashable) objects can be keys.
3. **Thread safety:** Immutable objects are inherently thread-safe.
4. **Performance:** Immutable objects can be interned/cached by the interpreter.

---

### Q5: What is the difference between a list and a tuple?

**Answer:**

| Feature | `list` | `tuple` |
|---------|--------|---------|
| Mutability | Mutable | Immutable |
| Syntax | `[1, 2, 3]` | `(1, 2, 3)` |
| Performance | Slower (overhead for mutation support) | Faster (fixed size) |
| Memory | More memory (over-allocates for growth) | Less memory |
| Hashable | No | Yes (if elements are hashable) |
| Use case | Collections that change | Fixed records, dict keys |

```python
import sys

lst = [1, 2, 3, 4, 5]
tup = (1, 2, 3, 4, 5)

print(sys.getsizeof(lst))  # 104 bytes (typically)
print(sys.getsizeof(tup))  # 80 bytes — 23% less memory
```

**Named tuples** give you the immutability of tuples with named access:
```python
from collections import namedtuple

Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 4)
print(p.x, p.y)  # 3 4
```

---

### Q6: How does Python's `pass`, `continue`, and `break` differ?

**Answer:**

| Keyword | Purpose | Scope |
|---------|---------|-------|
| `pass` | Does nothing — placeholder for empty blocks | Any block |
| `continue` | Skip the **rest of current iteration**, go to next | Loops only |
| `break` | **Exit the entire loop** immediately | Loops only |

```python
for i in range(5):
    if i == 0:
        pass         # Does nothing, loop continues normally
    if i == 2:
        continue     # Skips printing 2
    if i == 4:
        break        # Exits loop entirely, 4 never printed
    print(i)

# Output: 1, 3
```

---

### Q7: What are `*args` and `**kwargs`?

**Answer:**

They allow functions to accept a **variable number of arguments**:

- `*args` — collects extra **positional** arguments into a `tuple`
- `**kwargs` — collects extra **keyword** arguments into a `dict`

```python
def demo(*args, **kwargs):
    print(f"args = {args}")      # tuple
    print(f"kwargs = {kwargs}")  # dict

demo(1, 2, 3, name="Alice", age=30)
# args = (1, 2, 3)
# kwargs = {'name': 'Alice', 'age': 30}
```

**Order of parameters in a function signature:**
```python
def func(positional, /, normal, *args, keyword_only, **kwargs):
    pass

# 1. positional-only (before /)
# 2. normal (positional or keyword)
# 3. *args
# 4. keyword-only (after *args)
# 5. **kwargs
```

**Unpacking use case:**
```python
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add(*nums))  # 6 — unpacks list into positional args

config = {'a': 1, 'b': 2, 'c': 3}
print(add(**config))  # 6 — unpacks dict into keyword args
```

---

### Q8: Explain Python's string formatting approaches.

**Answer:**

Python offers 4 string formatting methods (use f-strings for new code):

```python
name = "Alice"
age = 30
pi = 3.14159

# 1. % formatting (old-style, avoid)
print("Name: %s, Age: %d" % (name, age))

# 2. str.format()
print("Name: {}, Age: {}".format(name, age))
print("Name: {name}, Age: {age}".format(name=name, age=age))

# 3. f-strings (Python 3.6+) — PREFERRED
print(f"Name: {name}, Age: {age}")
print(f"Pi to 2 decimals: {pi:.2f}")
print(f"{'centered':^20}")  # Center-aligned in 20 chars

# 4. Template strings (for untrusted input)
from string import Template
t = Template("Name: $name")
print(t.substitute(name=name))
```

**f-string advanced features (Python 3.12+):**
```python
# Debugging (Python 3.8+)
x = 42
print(f"{x = }")  # x = 42

# Expressions
print(f"{2 ** 10 = }")  # 2 ** 10 = 1024

# Format specs
print(f"{1000000:,}")     # 1,000,000
print(f"{0.856:.1%}")     # 85.6%
print(f"{42:08b}")        # 00101010 (binary, 8 chars)
```

---

### Q9: What is a dictionary comprehension? How does it differ from list comprehension?

**Answer:**

**Comprehensions** are concise ways to create collections:

```python
# List comprehension — creates a list
squares = [x**2 for x in range(5)]
# [0, 1, 4, 9, 16]

# Dict comprehension — creates a dict
square_map = {x: x**2 for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# Set comprehension — creates a set
unique_lengths = {len(word) for word in ["hello", "world", "hi"]}
# {2, 5}

# Generator expression — creates a lazy iterator (NOT a tuple comprehension)
gen = (x**2 for x in range(5))
# <generator object>
```

**Nested comprehension:**
```python
# Flatten a 2D list
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = [num for row in matrix for num in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]

# With conditionals
even_squares = {x: x**2 for x in range(10) if x % 2 == 0}
# {0: 0, 2: 4, 4: 16, 6: 36, 8: 64}
```

---

### Q10: What is the difference between `deepcopy` and `shallow copy`?

**Answer:**

```python
import copy

original = [[1, 2, 3], [4, 5, 6]]

# Shallow copy — new outer list, but inner lists are SHARED
shallow = copy.copy(original)
shallow[0][0] = 999
print(original[0][0])  # 999 — inner list modified!

# Deep copy — completely independent clone at all levels
original = [[1, 2, 3], [4, 5, 6]]
deep = copy.deepcopy(original)
deep[0][0] = 999
print(original[0][0])  # 1 — original untouched
```

**Ways to create shallow copies:**
```python
# All of these create SHALLOW copies:
lst2 = lst[:]           # Slice
lst2 = list(lst)        # Constructor
lst2 = lst.copy()       # .copy() method
lst2 = copy.copy(lst)   # copy module

# For dicts:
d2 = d.copy()
d2 = dict(d)
d2 = {**d}              # Unpacking
```

**When to use deepcopy:** When your data structure has nested mutable objects (lists of lists, dicts of lists, etc.) and you need a fully independent copy.

---

### Q11: Explain Python's scope rules (LEGB).

**Answer:**

Python resolves variable names using the **LEGB rule**, searching in this order:

1. **L — Local**: Inside the current function
2. **E — Enclosing**: Inside enclosing (outer) functions (closures)
3. **G — Global**: At the module level
4. **B — Built-in**: Python's built-in namespace (`print`, `len`, etc.)

```python
x = "global"  # Global scope

def outer():
    x = "enclosing"  # Enclosing scope
    
    def inner():
        x = "local"  # Local scope
        print(x)     # "local" — found in L
    
    inner()
    print(x)         # "enclosing" — found in E

outer()
print(x)             # "global" — found in G
print(len)           # <built-in function len> — found in B
```

**Modifying outer scopes:**
```python
x = 0

def modify_global():
    global x      # Declare intent to modify global
    x = 42

def outer():
    y = 0
    def modify_enclosing():
        nonlocal y  # Declare intent to modify enclosing
        y = 42
    modify_enclosing()
    print(y)        # 42
```

---

### Q12: What are Python's falsy values?

**Answer:**

Objects that evaluate to `False` in a boolean context:

| Type | Falsy Value |
|------|-------------|
| `bool` | `False` |
| `NoneType` | `None` |
| `int` | `0` |
| `float` | `0.0` |
| `complex` | `0j` |
| `str` | `""` (empty string) |
| `list` | `[]` (empty list) |
| `tuple` | `()` (empty tuple) |
| `dict` | `{}` (empty dict) |
| `set` | `set()` (empty set) |
| `range` | `range(0)` (empty range) |
| Custom | Objects with `__bool__()` returning `False` or `__len__()` returning `0` |

```python
# Pythonic way to check for empty collections
if not my_list:       # Better than: if len(my_list) == 0
    print("empty")

if my_string:         # Better than: if my_string != ""
    print("not empty")

# But be careful with explicit None checks:
if x is None:         # Use 'is', not 'if not x' (0 and "" are not None)
    print("x is None")
```

---

### Q13: What is a lambda function?

**Answer:**

A **lambda** is an anonymous, single-expression function:

```python
# Regular function
def square(x):
    return x ** 2

# Equivalent lambda
square = lambda x: x ** 2

# Common use cases — short callbacks
names = ["Charlie", "Alice", "Bob"]
sorted_names = sorted(names, key=lambda name: len(name))
# ['Bob', 'Alice', 'Charlie']

# With map and filter
numbers = [1, 2, 3, 4, 5]
evens = list(filter(lambda x: x % 2 == 0, numbers))     # [2, 4]
doubled = list(map(lambda x: x * 2, numbers))            # [2, 4, 6, 8, 10]
```

**Limitations:** Lambdas can only contain a single expression (no statements, no assignments, no multi-line logic). Use regular `def` for anything complex.

**Anti-pattern:** Don't assign a lambda to a variable — use `def` instead:
```python
# Bad
square = lambda x: x ** 2

# Good
def square(x):
    return x ** 2
```

---

### Q14: What are Python exceptions? How does exception handling work?

**Answer:**

Exceptions are errors detected during execution. Python uses `try/except/else/finally`:

```python
try:
    result = 10 / 0
except ZeroDivisionError as e:
    print(f"Error: {e}")           # Catches specific exception
except (TypeError, ValueError):
    print("Type or Value error")   # Catch multiple types
except Exception as e:
    print(f"Unexpected: {e}")      # Catch-all (avoid in production)
else:
    print("No errors occurred")    # Runs ONLY if no exception
finally:
    print("Always runs")           # Cleanup — always executes
```

**Exception hierarchy (key classes):**
```
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   └── OverflowError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── OSError
    │   └── FileNotFoundError
    ├── TypeError
    ├── ValueError
    ├── AttributeError
    ├── StopIteration
    └── RuntimeError
```

**Custom exceptions:**
```python
class InsufficientFundsError(Exception):
    def __init__(self, balance, amount):
        self.balance = balance
        self.amount = amount
        super().__init__(
            f"Cannot withdraw ${amount}. Balance: ${balance}"
        )

# Usage
raise InsufficientFundsError(balance=100, amount=150)
```

---

### Q15: How does Python's `for...else` work?

**Answer:**

The `else` block in a `for` loop runs **only if the loop completes without hitting a `break`**:

```python
# Searching for a value
def find_prime_factor(n):
    for i in range(2, n):
        if n % i == 0:
            print(f"Found factor: {i}")
            break
    else:
        # This runs if NO break was hit — meaning no factor was found
        print(f"{n} is prime!")

find_prime_factor(7)   # "7 is prime!"
find_prime_factor(12)  # "Found factor: 2"
```

This also works with `while...else`. Think of `else` as "no break."

---

### Q16: What is the difference between `@staticmethod` and `@classmethod`?

**Answer:**

```python
class MyClass:
    class_var = "I belong to the class"
    
    def instance_method(self):
        """Has access to instance (self) and class."""
        return f"Instance: {self}"
    
    @classmethod
    def class_method(cls):
        """Has access to class (cls), NOT instance."""
        return f"Class: {cls.class_var}"
    
    @staticmethod
    def static_method():
        """No access to instance or class. Just a namespaced function."""
        return "I'm just a regular function in a class namespace"
```

| Feature | Instance Method | `@classmethod` | `@staticmethod` |
|---------|----------------|-----------------|-----------------|
| First arg | `self` (instance) | `cls` (class) | None |
| Access instance? | ✅ | ❌ | ❌ |
| Access class? | ✅ (via `self.__class__`) | ✅ | ❌ |
| Called on instance? | ✅ | ✅ | ✅ |
| Called on class? | ❌ | ✅ | ✅ |
| Use case | Modify instance state | Factory methods, alternate constructors | Utility functions |

**Factory method pattern with `@classmethod`:**
```python
class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day
    
    @classmethod
    def from_string(cls, date_string):
        """Alternate constructor from string."""
        year, month, day = map(int, date_string.split('-'))
        return cls(year, month, day)  # Works with subclasses too!
    
    @classmethod
    def today(cls):
        import datetime
        d = datetime.date.today()
        return cls(d.year, d.month, d.day)

d = Date.from_string("2024-01-15")
```

---

### Q17: Explain Python's `with` statement and context managers.

**Answer:**

The `with` statement ensures **resource cleanup** by automatically calling `__enter__` and `__exit__`:

```python
# Without context manager (risky — file may not close on exception)
f = open("data.txt")
try:
    data = f.read()
finally:
    f.close()

# With context manager (guaranteed cleanup)
with open("data.txt") as f:
    data = f.read()
# f.close() is called automatically, even if an exception occurs
```

**How it works under the hood:**
```python
class ManagedFile:
    def __init__(self, filename):
        self.filename = filename
    
    def __enter__(self):
        self.file = open(self.filename, 'r')
        return self.file  # This is what gets assigned to 'as' variable
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file.close()
        return False  # Return True to suppress the exception

with ManagedFile("data.txt") as f:
    data = f.read()
```

**Simpler approach with `contextlib`:**
```python
from contextlib import contextmanager

@contextmanager
def managed_file(filename):
    f = open(filename, 'r')
    try:
        yield f         # Everything before yield = __enter__
    finally:
        f.close()       # Everything after yield = __exit__
```

---

### Q18: What are decorators in Python?

**Answer:**

A **decorator** is a function that wraps another function to extend its behavior without modifying it:

```python
import functools
import time

def timer(func):
    @functools.wraps(func)  # Preserves original function's metadata
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer  # Equivalent to: slow_function = timer(slow_function)
def slow_function():
    time.sleep(1)
    return "done"

slow_function()  # Prints: slow_function took 1.0012s
```

**Why `@functools.wraps`?** Without it, `slow_function.__name__` would be `"wrapper"` instead of `"slow_function"`, breaking introspection and debugging.

**Decorator with arguments:**
```python
def repeat(n):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(n):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)  # repeat(3) returns decorator, which wraps greet
def greet(name):
    print(f"Hello, {name}!")

greet("Alice")  # Prints "Hello, Alice!" three times
```

---

### Q19: What is the difference between `append()`, `extend()`, and `+=` for lists?

**Answer:**

```python
# append — adds ONE element (even if it's a list)
a = [1, 2, 3]
a.append([4, 5])
print(a)  # [1, 2, 3, [4, 5]]

# extend — adds each element from an iterable
b = [1, 2, 3]
b.extend([4, 5])
print(b)  # [1, 2, 3, 4, 5]

# += — same as extend for lists (but creates new object for tuples)
c = [1, 2, 3]
c += [4, 5]
print(c)  # [1, 2, 3, 4, 5]
```

**Subtle difference with `+=` for tuples:**
```python
t = (1, 2, 3)
id_before = id(t)
t += (4, 5)
print(id(t) == id_before)  # False — new tuple created!
```

---

### Q20: How does `enumerate()` work and why should you use it?

**Answer:**

`enumerate()` adds a counter to an iterable, returning `(index, value)` pairs:

```python
fruits = ["apple", "banana", "cherry"]

# Without enumerate (anti-pattern)
for i in range(len(fruits)):
    print(i, fruits[i])

# With enumerate (Pythonic)
for i, fruit in enumerate(fruits):
    print(i, fruit)

# Custom start index
for i, fruit in enumerate(fruits, start=1):
    print(i, fruit)  # 1 apple, 2 banana, 3 cherry
```

It's more Pythonic, less error-prone, and works with any iterable (not just sequences).

---

## 🟡 Medium (Intermediate)

### Q21: How does Python's memory management work?

**Answer:**

Python uses a **private heap** managed by the **Python Memory Manager**, which has several layers:

**1. Reference Counting (Primary mechanism):**
Every object has a reference count. When it drops to 0, memory is freed immediately.

```python
import sys

a = [1, 2, 3]
print(sys.getrefcount(a))  # 2 (a + getrefcount's reference)

b = a
print(sys.getrefcount(a))  # 3

del b
print(sys.getrefcount(a))  # 2
```

**2. Garbage Collection (Cycle detector):**
Reference counting can't handle circular references. Python's GC detects and collects reference cycles.

```python
import gc

class Node:
    def __init__(self):
        self.ref = None

# Circular reference — reference counting alone can't free this
a = Node()
b = Node()
a.ref = b
b.ref = a

del a
del b
# Objects still exist! GC's cycle detector will find and free them.
gc.collect()  # Force garbage collection
```

**3. Memory Pools (pymalloc):**
- For objects ≤ 512 bytes, Python uses its own allocator with memory pools
- Pools are organized into **arenas** (256 KB chunks)
- This avoids the overhead of calling the OS for every small allocation

**4. Interning:**
- Small integers (-5 to 256) are cached and reused
- Short strings meeting identifier rules are interned
- This saves memory and speeds up comparisons

---

### Q22: What are generators and how do they differ from regular functions?

**Answer:**

A **generator** is a function that uses `yield` to produce a sequence of values **lazily** — one at a time, on demand:

```python
# Regular function — creates entire list in memory
def get_squares_list(n):
    result = []
    for i in range(n):
        result.append(i ** 2)
    return result

# Generator — produces values one at a time
def get_squares_gen(n):
    for i in range(n):
        yield i ** 2  # Pauses here, resumes on next()

# Usage
gen = get_squares_gen(5)
print(next(gen))  # 0 — computes and pauses
print(next(gen))  # 1 — resumes and pauses
print(next(gen))  # 4

# Iterate over remaining
for sq in gen:
    print(sq)  # 9, 16
```

**Key differences:**

| Feature | Function | Generator |
|---------|----------|-----------|
| Keyword | `return` | `yield` |
| Execution | Runs to completion | Pauses at each `yield` |
| Memory | Stores entire result | Stores only current state |
| Return value | The result | A generator iterator object |
| Reusable | Call again for new result | Exhausted after one pass |

**Memory advantage (critical for large datasets):**
```python
import sys

# List: ~800 KB
big_list = [x for x in range(100_000)]
print(sys.getsizeof(big_list))  # ~824,456 bytes

# Generator: ~200 bytes regardless of size!
big_gen = (x for x in range(100_000))
print(sys.getsizeof(big_gen))   # ~200 bytes
```

**`yield from` — delegate to sub-generator:**
```python
def chain(*iterables):
    for it in iterables:
        yield from it  # Equivalent to: for item in it: yield item

list(chain([1, 2], [3, 4], [5]))  # [1, 2, 3, 4, 5]
```

---

### Q23: Explain Python's Global Interpreter Lock (GIL).

**Answer:**

The **GIL** is a mutex in CPython that allows only **one thread to execute Python bytecode at a time**, even on multi-core machines.

**Why it exists:**
- CPython's memory management (reference counting) is not thread-safe
- The GIL simplifies the C extension API
- Without it, every reference count operation would need a lock

**Impact:**
```python
import threading
import time

def cpu_bound(n):
    """CPU-intensive work."""
    while n > 0:
        n -= 1

# Single-threaded: ~4s
start = time.time()
cpu_bound(100_000_000)
cpu_bound(100_000_000)
print(f"Sequential: {time.time() - start:.2f}s")

# Multi-threaded: ALSO ~4s (GIL prevents true parallelism!)
start = time.time()
t1 = threading.Thread(target=cpu_bound, args=(100_000_000,))
t2 = threading.Thread(target=cpu_bound, args=(100_000_000,))
t1.start(); t2.start()
t1.join(); t2.join()
print(f"Threaded: {time.time() - start:.2f}s")
```

**When threads STILL help — I/O-bound work:**
```python
# The GIL is RELEASED during I/O operations (network, file, sleep)
# So threads ARE useful for I/O-bound tasks
import requests

def fetch(url):
    return requests.get(url)

# This WILL speed up with threads because GIL is released during HTTP calls
```

**Workarounds for CPU-bound parallelism:**
1. `multiprocessing` — separate processes, each with its own GIL
2. `concurrent.futures.ProcessPoolExecutor` — high-level multi-process API
3. C extensions (NumPy, etc.) — release the GIL during C-level computations
4. **Python 3.13+ Free-threaded mode** — experimental no-GIL CPython build

---

### Q24: What is the Method Resolution Order (MRO) in Python?

**Answer:**

**MRO** determines the order in which base classes are searched when calling a method. Python uses the **C3 Linearization** algorithm.

```python
class A:
    def who(self): return "A"

class B(A):
    def who(self): return "B"

class C(A):
    def who(self): return "C"

class D(B, C):
    pass

# MRO: D -> B -> C -> A -> object
print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)

d = D()
print(d.who())  # "B" — found in B first, per MRO
```

**`super()` follows MRO, not just the parent:**
```python
class A:
    def __init__(self):
        print("A init")

class B(A):
    def __init__(self):
        print("B init")
        super().__init__()  # Calls C.__init__(), NOT A.__init__()!

class C(A):
    def __init__(self):
        print("C init")
        super().__init__()

class D(B, C):
    def __init__(self):
        print("D init")
        super().__init__()

D()
# Output: D init → B init → C init → A init
# super() follows MRO: D -> B -> C -> A
```

---

### Q25: What are `__init__`, `__new__`, `__str__`, and `__repr__`?

**Answer:**

These are **dunder (magic/special) methods**:

```python
class Product:
    def __new__(cls, *args, **kwargs):
        """Called FIRST — creates the instance object.
        Rarely overridden unless implementing singletons or immutable types."""
        print("__new__ called")
        instance = super().__new__(cls)
        return instance
    
    def __init__(self, name, price):
        """Called SECOND — initializes the instance."""
        print("__init__ called")
        self.name = name
        self.price = price
    
    def __repr__(self):
        """Unambiguous string for DEVELOPERS (debugging, logging).
        Should ideally be valid Python to recreate the object."""
        return f"Product(name={self.name!r}, price={self.price})"
    
    def __str__(self):
        """Readable string for END USERS (print, f-strings)."""
        return f"{self.name}: ${self.price:.2f}"

p = Product("Widget", 9.99)
print(repr(p))  # Product(name='Widget', price=9.99)
print(str(p))   # Widget: $9.99
print(p)        # Uses __str__ — Widget: $9.99
```

**Key rule:** If you only implement one, implement `__repr__`. If `__str__` is missing, Python falls back to `__repr__`.

---

### Q26: How do Python closures work?

**Answer:**

A **closure** is an inner function that captures and remembers variables from its enclosing scope, even after the outer function has returned.

```python
def make_multiplier(factor):
    def multiplier(x):
        return x * factor  # 'factor' is captured from enclosing scope
    return multiplier

double = make_multiplier(2)
triple = make_multiplier(3)

print(double(5))   # 10
print(triple(5))   # 15

# The closure "remembers" factor
print(double.__closure__[0].cell_contents)  # 2
```

**Classic gotcha — late binding in loops:**
```python
# Bug: All functions capture the SAME variable 'i'
funcs = []
for i in range(3):
    funcs.append(lambda: i)

print([f() for f in funcs])  # [2, 2, 2] — all see i=2!

# Fix: Default argument captures current value
funcs = []
for i in range(3):
    funcs.append(lambda i=i: i)

print([f() for f in funcs])  # [0, 1, 2] — correct!
```

---

### Q27: Explain the difference between `@property` and regular attributes.

**Answer:**

`@property` lets you define **computed attributes** with getter/setter logic while keeping clean attribute-access syntax:

```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius  # "Private" backing attribute
    
    @property
    def celsius(self):
        """Getter — called on attribute access."""
        return self._celsius
    
    @celsius.setter
    def celsius(self, value):
        """Setter — called on attribute assignment."""
        if value < -273.15:
            raise ValueError("Temperature below absolute zero!")
        self._celsius = value
    
    @property
    def fahrenheit(self):
        """Read-only computed property."""
        return self._celsius * 9/5 + 32

t = Temperature(100)
print(t.celsius)      # 100 — calls getter
print(t.fahrenheit)   # 212.0 — computed

t.celsius = 0         # Calls setter with validation
t.fahrenheit = 100    # AttributeError — no setter defined (read-only)
```

**When to use `@property`:**
- Add validation to attribute setting
- Compute derived values on the fly
- Maintain backward compatibility when changing from simple attributes to computed ones

---

### Q28: What is the difference between `__getattr__` and `__getattribute__`?

**Answer:**

```python
class Demo:
    def __init__(self):
        self.existing = "I exist"
    
    def __getattribute__(self, name):
        """Called for EVERY attribute access (existing or not).
        Be very careful — can cause infinite recursion!"""
        print(f"__getattribute__: {name}")
        return super().__getattribute__(name)
    
    def __getattr__(self, name):
        """Called ONLY when normal lookup fails (attribute doesn't exist).
        This is the 'fallback' method."""
        print(f"__getattr__: {name}")
        return f"Default value for {name}"

d = Demo()
print(d.existing)  # __getattribute__ → "I exist"
print(d.missing)   # __getattribute__ → fails → __getattr__ → "Default value for missing"
```

| Method | When called | Use case |
|--------|-------------|----------|
| `__getattribute__` | **Every** attribute access | Logging, proxying, access control |
| `__getattr__` | Only when attribute **not found** | Default values, dynamic attributes |

---

### Q29: How do you create an abstract base class in Python?

**Answer:**

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        """Subclasses MUST implement this."""
        pass
    
    @abstractmethod
    def perimeter(self) -> float:
        pass
    
    def describe(self):
        """Concrete method — inherited by subclasses."""
        return f"Shape with area={self.area():.2f}"

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius
    
    def area(self) -> float:
        import math
        return math.pi * self.radius ** 2
    
    def perimeter(self) -> float:
        import math
        return 2 * math.pi * self.radius

# shape = Shape()  # TypeError: Can't instantiate abstract class
circle = Circle(5)
print(circle.describe())  # "Shape with area=78.54"
```

**Abstract properties:**
```python
class Vehicle(ABC):
    @property
    @abstractmethod
    def fuel_type(self) -> str:
        pass

class ElectricCar(Vehicle):
    @property
    def fuel_type(self) -> str:
        return "Electric"
```

---

### Q30: What are `dataclasses` and when should you use them?

**Answer:**

`dataclasses` (Python 3.7+) auto-generate `__init__`, `__repr__`, `__eq__`, and more from class annotations:

```python
from dataclasses import dataclass, field

@dataclass
class Employee:
    name: str
    age: int
    department: str = "Engineering"  # Default value
    skills: list = field(default_factory=list)  # Mutable default
    _id: int = field(init=False, repr=False)    # Excluded from __init__ and __repr__
    
    def __post_init__(self):
        """Called after __init__ for custom logic."""
        self._id = hash(self.name)

# Auto-generated __init__, __repr__, __eq__
e = Employee("Alice", 30)
print(e)  # Employee(name='Alice', age=30, department='Engineering', skills=[])
```

**Frozen (immutable) dataclass:**
```python
@dataclass(frozen=True)
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
p.x = 3.0  # FrozenInstanceError!
```

**`dataclass` vs `NamedTuple` vs `Pydantic`:**

| Feature | `dataclass` | `NamedTuple` | Pydantic `BaseModel` |
|---------|-------------|--------------|---------------------|
| Mutable | ✅ (or frozen) | ❌ | ✅ |
| Validation | ❌ | ❌ | ✅ |
| Inheritance | ✅ | Limited | ✅ |
| Performance | Fast | Fastest | Slower (validation overhead) |
| Serialization | Manual | Manual | Built-in (JSON, dict) |

---

### Q31: What is duck typing?

**Answer:**

**"If it walks like a duck and quacks like a duck, it's a duck."**

Python doesn't check an object's type — it checks whether the object supports the required **behavior** (methods/attributes):

```python
class Duck:
    def quack(self): print("Quack!")
    def walk(self): print("Walking like a duck")

class Person:
    def quack(self): print("I'm quacking like a duck!")
    def walk(self): print("Walking like a person")

def make_it_quack(thing):
    # Doesn't care about the TYPE, only that it has a quack() method
    thing.quack()

make_it_quack(Duck())    # "Quack!"
make_it_quack(Person())  # "I'm quacking like a duck!"
```

**Protocols (Python 3.8+) — structural subtyping for type checkers:**
```python
from typing import Protocol

class Quackable(Protocol):
    def quack(self) -> None: ...

def make_it_quack(thing: Quackable) -> None:
    thing.quack()
# Type checker validates that the passed object has a quack() method
# No explicit inheritance needed — just matching the Protocol structure
```

---

### Q32: How does `__slots__` work?

**Answer:**

By default, Python stores instance attributes in a `__dict__` (a dictionary). `__slots__` replaces this with a fixed-size array, saving memory:

```python
class WithDict:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class WithSlots:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y

import sys
a = WithDict(1, 2)
b = WithSlots(1, 2)

print(sys.getsizeof(a.__dict__))  # ~104 bytes for the dict alone
# b has no __dict__!

# Can't add dynamic attributes with __slots__
b.z = 3  # AttributeError: 'WithSlots' object has no attribute 'z'
```

**Memory savings at scale:**
```python
# Creating 1 million objects
import tracemalloc
tracemalloc.start()

objects = [WithDict(i, i) for i in range(1_000_000)]
print(tracemalloc.get_traced_memory())  # ~200 MB

objects = [WithSlots(i, i) for i in range(1_000_000)]
print(tracemalloc.get_traced_memory())  # ~72 MB — 64% less!
```

**Use `__slots__` when:** You're creating many instances of the same class and want to save memory.

---

### Q33: What is the `collections` module? Name key data structures.

**Answer:**

```python
from collections import (
    defaultdict, Counter, OrderedDict, deque, namedtuple, ChainMap
)

# defaultdict — dict with default values for missing keys
word_count = defaultdict(int)
for word in "hello world hello".split():
    word_count[word] += 1  # No KeyError!
# {'hello': 2, 'world': 1}

# Counter — count elements
c = Counter("abracadabra")
print(c.most_common(3))  # [('a', 5), ('b', 2), ('r', 2)]

# deque — double-ended queue (O(1) append/pop from both ends)
dq = deque([1, 2, 3], maxlen=5)
dq.appendleft(0)    # O(1) — vs O(n) for list.insert(0, x)
dq.rotate(1)         # [3, 0, 1, 2]

# namedtuple — tuple with named fields
Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 4)
print(p.x, p.y)     # 3, 4

# ChainMap — combine multiple dicts (first match wins)
defaults = {'color': 'red', 'size': 'medium'}
user_prefs = {'color': 'blue'}
config = ChainMap(user_prefs, defaults)
print(config['color'])  # 'blue' — user_prefs checked first
print(config['size'])   # 'medium' — falls through to defaults
```

---

### Q34: How do you handle multiple inheritance in Python?

**Answer:**

Python supports multiple inheritance. The key challenges are method resolution and the **diamond problem**:

```python
class Loggable:
    def log(self):
        print(f"Logging: {self}")

class Serializable:
    def serialize(self):
        return str(self.__dict__)

class User(Loggable, Serializable):
    def __init__(self, name):
        self.name = name

u = User("Alice")
u.log()           # Works — from Loggable
u.serialize()     # Works — from Serializable
```

**Best practice — Use Mixins:**
```python
class TimestampMixin:
    """Mixin that adds timestamps to any class."""
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        from datetime import datetime
        self.created_at = datetime.now()
        self.updated_at = datetime.now()

class JSONMixin:
    """Mixin that adds JSON serialization."""
    def to_json(self):
        import json
        return json.dumps(self.__dict__, default=str)

class User(TimestampMixin, JSONMixin):
    def __init__(self, name, email):
        self.name = name
        self.email = email
        super().__init__()

u = User("Alice", "alice@example.com")
print(u.to_json())
print(u.created_at)
```

**Rules of thumb:**
1. Prefer **composition over inheritance** for complex cases
2. Keep mixins small and focused on a single concern
3. Always use `super()` to support cooperative multiple inheritance
4. Be aware of MRO (C3 linearization)

---

### Q35: What are type hints and how do they work in Python?

**Answer:**

Type hints (PEP 484, Python 3.5+) add optional type annotations. They are **not enforced at runtime** but enable static analysis with tools like `mypy`:

```python
from typing import Optional, Union, Any

# Basic annotations
def greet(name: str, times: int = 1) -> str:
    return f"Hello, {name}! " * times

# Collections (Python 3.9+ — use built-in types)
def process(items: list[int]) -> dict[str, int]:
    return {"sum": sum(items), "count": len(items)}

# Optional (value or None)
def find_user(user_id: int) -> Optional[str]:  # Same as str | None
    pass

# Union (multiple types) — Python 3.10+ uses |
def parse(value: int | str) -> str:
    return str(value)

# Type aliases
type Vector = list[float]  # Python 3.12+ syntax
def scale(v: Vector, factor: float) -> Vector:
    return [x * factor for x in v]
```

**Advanced typing (Python 3.12+):**
```python
from typing import TypeVar, Generic, Protocol, TypedDict, Literal

# Generic classes
type T = TypeVar('T')

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []
    
    def push(self, item: T) -> None:
        self._items.append(item)
    
    def pop(self) -> T:
        return self._items.pop()

# TypedDict — typed dictionaries
class UserDict(TypedDict):
    name: str
    age: int
    email: str

# Literal — restrict to specific values
def set_mode(mode: Literal["read", "write", "append"]) -> None:
    pass
```

---

### Q36: What is `itertools` and what are its most useful functions?

**Answer:**

`itertools` provides memory-efficient tools for working with iterators:

```python
import itertools

# count — infinite counter
counter = itertools.count(start=10, step=2)
print([next(counter) for _ in range(5)])  # [10, 12, 14, 16, 18]

# cycle — repeat an iterable infinitely
colors = itertools.cycle(["red", "green", "blue"])
print([next(colors) for _ in range(5)])  # ['red', 'green', 'blue', 'red', 'green']

# chain — concatenate iterables
combined = itertools.chain([1, 2], [3, 4], [5])
print(list(combined))  # [1, 2, 3, 4, 5]

# product — Cartesian product
print(list(itertools.product("AB", "12")))
# [('A','1'), ('A','2'), ('B','1'), ('B','2')]

# combinations and permutations
print(list(itertools.combinations("ABC", 2)))  # [('A','B'), ('A','C'), ('B','C')]
print(list(itertools.permutations("ABC", 2)))  # [('A','B'), ('A','C'), ('B','A'), ...]

# groupby — group consecutive items
data = [("fruit", "apple"), ("fruit", "banana"), ("veg", "carrot")]
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# fruit [('fruit', 'apple'), ('fruit', 'banana')]
# veg [('veg', 'carrot')]

# islice — slice an iterator (can't use regular slicing on iterators)
print(list(itertools.islice(range(100), 5, 10)))  # [5, 6, 7, 8, 9]

# starmap — map with unpacking
print(list(itertools.starmap(pow, [(2, 3), (3, 2), (10, 3)])))  # [8, 9, 1000]
```

---

### Q37: How does Python's `functools` module help in interviews?

**Answer:**

```python
import functools

# 1. lru_cache — memoization (caching function results)
@functools.lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

print(fibonacci(100))  # Instant! Without cache: would take forever
print(fibonacci.cache_info())  # CacheInfo(hits=98, misses=101, ...)

# 2. cache — unlimited cache (Python 3.9+)
@functools.cache
def expensive_computation(x):
    return x ** x

# 3. partial — fix some arguments
def power(base, exp):
    return base ** exp

square = functools.partial(power, exp=2)
cube = functools.partial(power, exp=3)
print(square(5))  # 25
print(cube(5))    # 125

# 4. reduce — fold a sequence into a single value
product = functools.reduce(lambda a, b: a * b, [1, 2, 3, 4, 5])
print(product)  # 120

# 5. total_ordering — auto-generate comparison methods
@functools.total_ordering
class Student:
    def __init__(self, name, grade):
        self.name = name
        self.grade = grade
    
    def __eq__(self, other):
        return self.grade == other.grade
    
    def __lt__(self, other):
        return self.grade < other.grade
    # __le__, __gt__, __ge__ are auto-generated!
```

---

### Q38: What is the `walrus operator` (`:=`) and when should you use it?

**Answer:**

The walrus operator (Python 3.8+) **assigns a value to a variable as part of an expression**:

```python
# Without walrus — redundant call
data = input("Enter: ")
if data:
    process(data)

# With walrus — assign and test in one step
if (data := input("Enter: ")):
    process(data)

# Useful in while loops
while (line := file.readline()) != "":
    process(line)

# Useful in comprehensions
# Filter and transform in one pass (avoid computing twice)
results = [
    upper
    for name in names
    if (upper := name.upper()) startswith("A")
]

# Useful in regex matching
import re
if (match := re.search(r'\d+', text)):
    print(match.group())
```

**Don't overuse it** — readability matters. Use it when it eliminates genuine redundancy.

---

### Q39: Explain the `__call__` dunder method.

**Answer:**

`__call__` makes an instance **callable** — you can use it like a function:

```python
class Validator:
    def __init__(self, min_val, max_val):
        self.min_val = min_val
        self.max_val = max_val
    
    def __call__(self, value):
        if not self.min_val <= value <= self.max_val:
            raise ValueError(f"{value} not in [{self.min_val}, {self.max_val}]")
        return True

validate_age = Validator(0, 150)
validate_age(25)     # True — calling the instance like a function
validate_age(200)    # ValueError

# Check if something is callable
print(callable(validate_age))  # True
```

**Use cases:** Stateful functions, class-based decorators, strategy pattern.

---

### Q40: What is the difference between `map()`, `filter()`, and `reduce()`?

**Answer:**

```python
from functools import reduce

numbers = [1, 2, 3, 4, 5]

# map — apply function to each element
squares = list(map(lambda x: x**2, numbers))       # [1, 4, 9, 16, 25]

# filter — keep elements where function returns True
evens = list(filter(lambda x: x % 2 == 0, numbers)) # [2, 4]

# reduce — accumulate elements into single value
total = reduce(lambda acc, x: acc + x, numbers)      # 15
```

**Pythonic alternatives (preferred):**
```python
# Use comprehensions instead of map/filter
squares = [x**2 for x in numbers]            # Clearer than map
evens = [x for x in numbers if x % 2 == 0]  # Clearer than filter

# Use sum/max/min instead of reduce
total = sum(numbers)                          # Clearer than reduce
```

---

## 🔴 Hard (Advanced / MAANG-level)

### Q41: Explain Python's descriptor protocol.

**Answer:**

Descriptors are objects that define how attribute access works. They power `@property`, `@classmethod`, `@staticmethod`, and `__slots__`.

A descriptor is any object that implements at least one of:
- `__get__(self, obj, objtype=None)` — attribute access
- `__set__(self, obj, value)` — attribute assignment
- `__delete__(self, obj)` — attribute deletion

```python
class Validated:
    """A descriptor that validates attribute values."""
    
    def __init__(self, min_value=None, max_value=None):
        self.min_value = min_value
        self.max_value = max_value
    
    def __set_name__(self, owner, name):
        """Called when the descriptor is assigned to a class attribute."""
        self.name = name
        self.private_name = f"_{name}"
    
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self  # Accessing from the class
        return getattr(obj, self.private_name, None)
    
    def __set__(self, obj, value):
        if self.min_value is not None and value < self.min_value:
            raise ValueError(f"{self.name} must be >= {self.min_value}")
        if self.max_value is not None and value > self.max_value:
            raise ValueError(f"{self.name} must be <= {self.max_value}")
        setattr(obj, self.private_name, value)

class Product:
    price = Validated(min_value=0)
    quantity = Validated(min_value=0, max_value=1000)
    
    def __init__(self, name, price, quantity):
        self.name = name
        self.price = price        # Triggers Validated.__set__
        self.quantity = quantity

p = Product("Widget", 9.99, 100)
p.price = -5  # ValueError: price must be >= 0
```

**Descriptor types:**
- **Data descriptor:** Defines `__set__` and/or `__delete__` — takes priority over instance `__dict__`
- **Non-data descriptor:** Only defines `__get__` — instance `__dict__` takes priority

**Attribute lookup order:**
1. Data descriptor on the class
2. Instance `__dict__`
3. Non-data descriptor on the class

---

### Q42: How do metaclasses work in Python?

**Answer:**

A **metaclass** is the "class of a class." Just as classes create instances, metaclasses create classes. The default metaclass is `type`.

```python
# Regular class creation
class MyClass:
    pass

# Equivalent using type() metaclass directly
MyClass = type('MyClass', (object,), {'x': 42})
```

**Custom metaclass:**
```python
class SingletonMeta(type):
    """Metaclass that enforces the Singleton pattern."""
    _instances = {}
    
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Database(metaclass=SingletonMeta):
    def __init__(self):
        self.connection = "connected"

db1 = Database()
db2 = Database()
print(db1 is db2)  # True — same instance!
```

**Practical metaclass — auto-registering subclasses:**
```python
class PluginMeta(type):
    registry = {}
    
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if bases:  # Don't register the base class itself
            PluginMeta.registry[name] = cls
        return cls

class Plugin(metaclass=PluginMeta):
    pass

class PDFPlugin(Plugin):
    pass

class CSVPlugin(Plugin):
    pass

print(PluginMeta.registry)
# {'PDFPlugin': <class 'PDFPlugin'>, 'CSVPlugin': <class 'CSVPlugin'>}
```

**When to use metaclasses:** Almost never. Prefer decorators or `__init_subclass__` (Python 3.6+) for most use cases:

```python
class Plugin:
    registry = {}
    
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        Plugin.registry[cls.__name__] = cls

class PDFPlugin(Plugin):
    pass
# Same effect, much simpler!
```

---

### Q43: Explain Python's garbage collection in depth — how does the cycle detector work?

**Answer:**

Python's GC has **three generations** for cycle detection:

**Generation 0:** Newly created objects  
**Generation 1:** Survived one GC cycle  
**Generation 2:** Survived two+ GC cycles (long-lived objects)

```python
import gc

# View thresholds: (gen0_threshold, gen1_threshold, gen2_threshold)
print(gc.get_threshold())  # (700, 10, 10) by default

# Meaning:
# - Gen 0 collected when 700 allocations - deallocations happen
# - Gen 1 collected every 10 Gen 0 collections
# - Gen 2 collected every 10 Gen 1 collections
```

**How cycle detection works:**
1. For each object in a generation, temporarily decrement the reference count for each reference it holds to another object in the same generation
2. Objects with a remaining count > 0 are **reachable** (referenced from outside the generation)
3. Objects with count = 0 are **unreachable** — part of a cycle — and can be freed

**`__del__` and GC issues:**
```python
class Resource:
    def __del__(self):
        print("Cleanup!")  # Destructor — called when object is GC'd

# Problem: Objects with __del__ in reference cycles used to be uncollectable
# (before Python 3.4). Now they're handled, but __del__ order is undefined.
# Prefer context managers or weak references for cleanup.
```

**Weak references — avoid reference cycles:**
```python
import weakref

class Cache:
    def __init__(self):
        self._data = weakref.WeakValueDictionary()
    
    def get(self, key):
        return self._data.get(key)
    
    def set(self, key, value):
        self._data[key] = value
# Values are automatically removed when no strong references remain
```

---

### Q44: How does `asyncio` work under the hood?

**Answer:**

`asyncio` uses a **single-threaded event loop** with **cooperative multitasking**:

```python
import asyncio

async def fetch_data(url: str) -> str:
    """Coroutine — suspends at await points."""
    print(f"Start fetching {url}")
    await asyncio.sleep(1)          # Suspends, yields control to event loop
    print(f"Done fetching {url}")
    return f"Data from {url}"

async def main():
    # Run concurrently with gather
    results = await asyncio.gather(
        fetch_data("url1"),
        fetch_data("url2"),
        fetch_data("url3"),
    )
    # All 3 complete in ~1 second total (not 3 seconds)!
    print(results)

asyncio.run(main())
```

**How it works internally:**

1. `async def` creates a **coroutine function** that returns a coroutine object
2. `await` suspends the coroutine and returns control to the event loop
3. The event loop runs other ready coroutines while waiting
4. When the awaited operation completes, the coroutine resumes

**Key components:**
```python
# Event loop — the central scheduler
loop = asyncio.get_event_loop()

# Tasks — wrap coroutines for scheduling
task = asyncio.create_task(fetch_data("url1"))

# Futures — represent eventual results
future = asyncio.Future()
future.set_result(42)

# Queues — async producer-consumer pattern
queue = asyncio.Queue(maxsize=10)
await queue.put(item)
item = await queue.get()
```

**`async for` and `async with`:**
```python
class AsyncCounter:
    def __init__(self, stop):
        self.current = 0
        self.stop = stop
    
    def __aiter__(self):
        return self
    
    async def __anext__(self):
        if self.current >= self.stop:
            raise StopAsyncIteration
        await asyncio.sleep(0.1)
        self.current += 1
        return self.current

async def main():
    async for num in AsyncCounter(5):
        print(num)
```

---

### Q45: What is the difference between concurrency and parallelism in Python?

**Answer:**

| Concept | Concurrency | Parallelism |
|---------|-------------|-------------|
| Definition | Multiple tasks **in progress** at once | Multiple tasks **executing** at once |
| Analogy | One cook juggling multiple dishes | Multiple cooks, one dish each |
| Python tool | `threading`, `asyncio` | `multiprocessing` |
| GIL impact | Limited by GIL for CPU work | Bypasses GIL (separate processes) |
| Best for | I/O-bound (network, file, DB) | CPU-bound (math, ML training) |

```python
import asyncio
import multiprocessing
import threading

# Concurrency (I/O-bound) — asyncio
async def io_bound():
    tasks = [asyncio.create_task(fetch(url)) for url in urls]
    return await asyncio.gather(*tasks)

# Concurrency (I/O-bound) — threading
with concurrent.futures.ThreadPoolExecutor(max_workers=10) as pool:
    results = list(pool.map(fetch, urls))

# Parallelism (CPU-bound) — multiprocessing
with concurrent.futures.ProcessPoolExecutor(max_workers=4) as pool:
    results = list(pool.map(cpu_heavy, data_chunks))
```

**Decision tree:**
```
Is your task I/O-bound or CPU-bound?
├── I/O-bound (network, files, DB)
│   ├── Many connections → asyncio
│   └── Few connections → threading
└── CPU-bound (computation)
    └── multiprocessing (or C extension)
```

---

### Q46: How do you implement the iterator protocol in Python?

**Answer:**

An iterator implements `__iter__()` and `__next__()`:

```python
class FibonacciIterator:
    """Infinite Fibonacci sequence iterator."""
    
    def __init__(self):
        self.a, self.b = 0, 1
    
    def __iter__(self):
        return self  # Iterator returns itself
    
    def __next__(self):
        result = self.a
        self.a, self.b = self.b, self.a + self.b
        return result

fib = FibonacciIterator()
for i, num in enumerate(fib):
    if i >= 10:
        break
    print(num, end=" ")  # 0 1 1 2 3 5 8 13 21 34
```

**Iterable vs Iterator:**
- **Iterable:** Has `__iter__()` that returns an iterator (e.g., `list`, `str`)
- **Iterator:** Has `__iter__()` (returns self) AND `__next__()` (produces values)

```python
my_list = [1, 2, 3]           # Iterable (not an iterator)
my_iter = iter(my_list)        # Iterator (created from iterable)
print(next(my_iter))           # 1
print(next(my_iter))           # 2
```

---

### Q47: Explain `__init_subclass__` and how it replaces metaclasses.

**Answer:**

`__init_subclass__` (Python 3.6+) is called whenever a class is subclassed, providing a simpler alternative to metaclasses:

```python
class Serializable:
    _registry = {}
    
    def __init_subclass__(cls, format_type=None, **kwargs):
        super().__init_subclass__(**kwargs)
        if format_type:
            Serializable._registry[format_type] = cls
    
    @classmethod
    def get_serializer(cls, format_type):
        return cls._registry.get(format_type)

class JSONSerializer(Serializable, format_type="json"):
    def serialize(self, data):
        import json
        return json.dumps(data)

class XMLSerializer(Serializable, format_type="xml"):
    def serialize(self, data):
        return f"<data>{data}</data>"

# Auto-registered!
serializer_cls = Serializable.get_serializer("json")
print(serializer_cls().serialize({"key": "value"}))
```

**Validation use case:**
```python
class Model:
    required_fields: list[str] = []
    
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        if not hasattr(cls, 'required_fields') or not cls.required_fields:
            raise TypeError(f"{cls.__name__} must define 'required_fields'")

class User(Model):
    required_fields = ["name", "email"]  # OK

class BadModel(Model):
    pass  # TypeError!
```

---

### Q48: What is `match/case` (structural pattern matching) in Python 3.10+?

**Answer:**

Structural pattern matching goes far beyond a simple switch statement — it can **destructure** and **match patterns**:

```python
# Basic matching
def http_status(status):
    match status:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500 | 502 | 503:  # OR patterns
            return "Server Error"
        case _:                 # Wildcard (default)
            return "Unknown"

# Destructuring sequences
def process_command(command):
    match command.split():
        case ["quit"]:
            quit()
        case ["go", direction]:
            move(direction)
        case ["pick", "up", item]:
            pickup(item)
        case ["drop", *items]:     # Capture remaining
            drop(items)

# Class pattern matching
class Point:
    __match_args__ = ("x", "y")  # Define positional match order
    def __init__(self, x, y):
        self.x = x
        self.y = y

def locate(point):
    match point:
        case Point(x=0, y=0):
            return "Origin"
        case Point(x=0, y=y):
            return f"Y-axis at {y}"
        case Point(x=x, y=0):
            return f"X-axis at {x}"
        case Point(x, y) if x == y:   # Guard clause
            return f"Diagonal at {x}"
        case Point(x, y):
            return f"Point({x}, {y})"

# Mapping patterns
def process_config(config):
    match config:
        case {"type": "database", "host": host, "port": int(port)}:
            connect(host, port)
        case {"type": "cache", "backend": "redis", **rest}:
            setup_redis(**rest)
```

---

### Q49: Explain Python's `__prepare__`, `__new__`, and `__init__` in metaclass context.

**Answer:**

When a class is created, three methods are called on the metaclass:

```python
class OrderedMeta(type):
    @classmethod
    def __prepare__(mcs, name, bases):
        """Step 1: Return the namespace dict for the class body.
        Called BEFORE the class body executes."""
        print(f"__prepare__({name})")
        return {}  # Could return OrderedDict to track definition order
    
    def __new__(mcs, name, bases, namespace):
        """Step 2: Create the class object.
        Called AFTER the class body has executed."""
        print(f"__new__({name}, namespace keys: {list(namespace.keys())})")
        return super().__new__(mcs, name, bases, namespace)
    
    def __init__(cls, name, bases, namespace):
        """Step 3: Initialize the class object.
        Called after __new__ returns the class."""
        print(f"__init__({name})")
        super().__init__(name, bases, namespace)

class MyClass(metaclass=OrderedMeta):
    x = 1
    y = 2
    def method(self): pass

# Output:
# __prepare__(MyClass)
# __new__(MyClass, namespace keys: ['__module__', '__qualname__', 'x', 'y', 'method'])
# __init__(MyClass)
```

---

### Q50: What are Python's `__enter__` and `__exit__` in async context managers?

**Answer:**

Async context managers use `__aenter__` and `__aexit__`:

```python
class AsyncDBConnection:
    async def __aenter__(self):
        self.conn = await create_connection()
        return self.conn
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.conn.close()
        return False  # Don't suppress exceptions

# Usage
async with AsyncDBConnection() as conn:
    await conn.execute("SELECT 1")
```

**With `contextlib`:**
```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_db():
    conn = await create_connection()
    try:
        yield conn
    finally:
        await conn.close()

async with get_db() as conn:
    await conn.execute("SELECT 1")
```

---

### Q51: How does CPython's reference counting interact with the GIL?

**Answer:**

Reference counting is **not thread-safe** without the GIL. Every `Py_INCREF`/`Py_DECREF` is a non-atomic read-modify-write:

```c
// CPython internals (simplified)
#define Py_INCREF(op) (++(op)->ob_refcnt)
#define Py_DECREF(op) \
    if (--(op)->ob_refcnt == 0) \
        _Py_Dealloc(op)
```

Without the GIL, two threads could simultaneously:
1. Read the same reference count (say, 2)
2. Both decrement to 1
3. Object is never freed (memory leak) or freed while still referenced (crash)

**Python 3.13+ Free-threaded (no-GIL) approach:**
- Uses **biased reference counting**: the creating thread uses a fast, uncontested counter; other threads use a slower atomic counter
- Uses **deferred reference counting** for certain global objects
- Adds per-object locks where needed
- Currently experimental, with 10-40% single-threaded slowdown

---

### Q52: What is `__class_getitem__` and how does it enable generic types?

**Answer:**

`__class_getitem__` (Python 3.7+) allows classes to be subscriptable, enabling `MyClass[int]` syntax:

```python
class Matrix:
    def __class_getitem__(cls, item):
        """Called when you do Matrix[int] or Matrix[float]."""
        # item = int, float, etc.
        return type(f'Matrix[{item.__name__}]', (cls,), {'dtype': item})

IntMatrix = Matrix[int]
print(IntMatrix.dtype)  # <class 'int'>
```

This is how `list[int]`, `dict[str, int]`, etc. work in Python 3.9+.

---

### Q53: How do you profile and optimize Python code?

**Answer:**

```python
# 1. timeit — micro-benchmarks
import timeit
timeit.timeit('sum(range(100))', number=100000)

# 2. cProfile — function-level profiling
import cProfile
cProfile.run('my_function()')

# 3. line_profiler — line-by-line profiling
# pip install line_profiler
# @profile decorator, then: kernprof -l -v script.py

# 4. memory_profiler — memory usage
# pip install memory_profiler
# @profile decorator, then: python -m memory_profiler script.py

# 5. tracemalloc — trace memory allocations
import tracemalloc
tracemalloc.start()
# ... your code ...
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('lineno')[:10]:
    print(stat)
```

**Common optimization strategies:**
1. Use built-in functions (`sum`, `map`, `any`) — implemented in C
2. Use `collections` (deque, defaultdict) — specialized data structures
3. Use generators for large datasets — avoid loading everything into memory
4. Use `__slots__` for many small objects
5. Use `numpy` for numerical operations — vectorized C operations
6. Use `@functools.lru_cache` for expensive repeated computations
7. Prefer string `join()` over `+` concatenation in loops

---

### Q54: Explain Python's data model — what happens when you write `a + b`?

**Answer:**

Python's data model defines how operators, built-ins, and syntax map to dunder methods:

```python
# When you write: a + b
# Python does:
# 1. Try a.__add__(b)
# 2. If NotImplemented, try b.__radd__(a)
# 3. If still NotImplemented, raise TypeError

class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    def __add__(self, other):
        if isinstance(other, Vector):
            return Vector(self.x + other.x, self.y + other.y)
        return NotImplemented  # Let Python try other.__radd__
    
    def __radd__(self, other):
        """Called when other + self and other doesn't know how."""
        return self.__add__(other)
    
    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)
    
    def __rmul__(self, scalar):
        return self.__mul__(scalar)
    
    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

v1 = Vector(1, 2)
v2 = Vector(3, 4)
print(v1 + v2)     # Vector(4, 6) — calls v1.__add__(v2)
print(3 * v1)       # Vector(3, 6) — int.__mul__ fails → calls v1.__rmul__(3)
```

**Key operator mappings:**

| Expression | Dunder Method |
|-----------|---------------|
| `a + b` | `__add__` / `__radd__` |
| `a - b` | `__sub__` / `__rsub__` |
| `a * b` | `__mul__` / `__rmul__` |
| `a[key]` | `__getitem__` |
| `a[key] = val` | `__setitem__` |
| `len(a)` | `__len__` |
| `str(a)` | `__str__` |
| `repr(a)` | `__repr__` |
| `bool(a)` | `__bool__` |
| `hash(a)` | `__hash__` |
| `a == b` | `__eq__` |
| `a < b` | `__lt__` |
| `a in b` | `__contains__` |
| `for x in a` | `__iter__` |
| `a()` | `__call__` |

---

### Q55: What are the differences between `ProcessPoolExecutor` and `ThreadPoolExecutor`?

**Answer:**

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import time

def cpu_bound(n):
    return sum(i * i for i in range(n))

def io_bound(url):
    import urllib.request
    return urllib.request.urlopen(url).read()

# ThreadPoolExecutor — best for I/O-bound tasks
# Shares memory, lightweight, affected by GIL
with ThreadPoolExecutor(max_workers=10) as executor:
    futures = [executor.submit(io_bound, url) for url in urls]
    results = [f.result() for f in futures]

# ProcessPoolExecutor — best for CPU-bound tasks
# Separate memory, heavier, bypasses GIL
with ProcessPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(cpu_bound, n) for n in large_numbers]
    results = [f.result() for f in futures]
```

| Feature | ThreadPoolExecutor | ProcessPoolExecutor |
|---------|-------------------|---------------------|
| GIL | Limited by GIL | Bypasses GIL |
| Memory | Shared memory | Separate memory (serialized via pickle) |
| Overhead | Low (lightweight) | High (process creation) |
| Communication | Direct memory access | IPC (pickle serialization) |
| Best for | I/O-bound | CPU-bound |
| Max workers default | `min(32, os.cpu_count() + 4)` | `os.cpu_count()` |

---

### Q56: How would you implement a thread-safe Singleton in Python?

**Answer:**

```python
import threading

class Singleton:
    _instance = None
    _lock = threading.Lock()
    
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            with cls._lock:
                # Double-checked locking
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(self, value=None):
        # __init__ is called every time, so guard against re-init
        if not hasattr(self, '_initialized'):
            self.value = value
            self._initialized = True

# Simpler approach using module-level (Python modules are singletons)
# config.py
class _Config:
    def __init__(self):
        self.db_url = "..."

config = _Config()  # Module-level singleton

# Even simpler — use a metaclass (see Q42)
```

---

### Q57: Explain how Python handles name mangling and "private" attributes.

**Answer:**

Python has **no true private attributes**. It uses conventions and name mangling:

```python
class MyClass:
    def __init__(self):
        self.public = "anyone can access"      # Public
        self._protected = "convention: internal use"  # Protected (convention)
        self.__private = "name mangled"         # Name-mangled

obj = MyClass()
print(obj.public)      # Works
print(obj._protected)  # Works (convention only, not enforced)
# print(obj.__private) # AttributeError!
print(obj._MyClass__private)  # Works! Name mangling: _ClassName__attr
```

**Name mangling rules:**
- Applies to identifiers with **two or more leading underscores** and **at most one trailing underscore**
- `__var` → `_ClassName__var`
- `__var__` → NOT mangled (dunder methods)
- Purpose: Prevent accidental overriding in subclasses, NOT security

---

### Q58: What is `typing.Protocol` and how does it enable structural subtyping?

**Answer:**

`Protocol` (Python 3.8+) defines **structural subtyping** — a class satisfies a Protocol if it has the right methods, **without explicit inheritance**:

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> None: ...
    def resize(self, factor: float) -> None: ...

# No inheritance needed! Just implement the methods.
class Circle:
    def draw(self) -> None:
        print("Drawing circle")
    
    def resize(self, factor: float) -> None:
        self.radius *= factor

class Square:
    def draw(self) -> None:
        print("Drawing square")
    
    def resize(self, factor: float) -> None:
        self.side *= factor

def render(shape: Drawable) -> None:
    shape.draw()  # Type checker knows this is valid

render(Circle())  # ✅ Valid — Circle matches Drawable's structure
render(Square())  # ✅ Valid — Square matches Drawable's structure

# Runtime checking (with @runtime_checkable)
print(isinstance(Circle(), Drawable))  # True
```

**Protocol vs ABC:**

| Feature | `Protocol` | `ABC` |
|---------|-----------|-------|
| Subtyping | Structural (duck typing) | Nominal (explicit inheritance) |
| Inheritance required | ❌ | ✅ |
| Runtime checking | With `@runtime_checkable` | Built-in |
| Use case | Type checking, third-party compatibility | Enforced contracts |

---

### Q59: How do you handle circular imports in Python?

**Answer:**

Circular imports occur when module A imports module B, and module B imports module A:

```python
# module_a.py
from module_b import func_b  # ImportError if module_b also imports from module_a

def func_a():
    return func_b()

# module_b.py
from module_a import func_a  # Circular!

def func_b():
    return func_a()
```

**Solutions:**

1. **Restructure code** — Move shared code to a third module
2. **Import inside functions** (lazy import):
```python
def func_b():
    from module_a import func_a  # Import when needed, not at module load
    return func_a()
```

3. **Use `TYPE_CHECKING` for type hints:**
```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from module_a import ClassA  # Only imported during type checking, not runtime

def process(obj: "ClassA") -> None:
    pass
```

4. **Import the module, not the name:**
```python
import module_a  # Import the module
module_a.func_a()  # Access later
```

---

### Q60: What is the `__import__` system and how does Python's import machinery work?

**Answer:**

When you write `import foo`, Python:

1. Checks `sys.modules` (cache of already-imported modules)
2. If not cached, finds the module using **finders** in `sys.meta_path`
3. Loads the module using the appropriate **loader**
4. Caches in `sys.modules`
5. Binds the name in the calling module's namespace

```python
import sys

# sys.modules — cache of all imported modules
print('json' in sys.modules)  # False
import json
print('json' in sys.modules)  # True

# sys.path — where Python looks for modules
print(sys.path)  # ['', '/usr/lib/python3.12', ...]

# sys.meta_path — finders (ordered)
print(sys.meta_path)
# [BuiltinImporter, FrozenImporter, PathFinder]

# Custom importer
class CustomFinder:
    @classmethod
    def find_module(cls, name, path=None):
        if name == "virtual_module":
            return CustomLoader()
        return None

class CustomLoader:
    def load_module(self, name):
        import types
        mod = types.ModuleType(name)
        mod.hello = lambda: "Hello from virtual module!"
        sys.modules[name] = mod
        return mod

sys.meta_path.insert(0, CustomFinder)
import virtual_module
print(virtual_module.hello())  # "Hello from virtual module!"
```

---

### Q61: Explain `contextvars` and how they differ from thread-local storage.

**Answer:**

`contextvars` (Python 3.7+) provide **context-local storage** that works with both threads AND asyncio (unlike `threading.local`):

```python
import contextvars
import asyncio

# Create a context variable
request_id = contextvars.ContextVar('request_id', default='unknown')

async def handle_request(rid: str):
    request_id.set(rid)
    await process()

async def process():
    # Each coroutine has its own context — no cross-contamination
    print(f"Processing request: {request_id.get()}")

async def main():
    # These run concurrently but each has its own request_id
    await asyncio.gather(
        handle_request("req-1"),
        handle_request("req-2"),
        handle_request("req-3"),
    )

asyncio.run(main())
# Processing request: req-1
# Processing request: req-2
# Processing request: req-3
```

**`threading.local` vs `contextvars`:**

| Feature | `threading.local` | `contextvars` |
|---------|-------------------|---------------|
| Thread-safe | ✅ | ✅ |
| Async-safe | ❌ (shared across coroutines in same thread) | ✅ |
| Copy on task creation | ❌ | ✅ |
| Use case | Thread-only code | Modern async + threaded code |

---

### Q62: What is PEP 20 (The Zen of Python) and how does it guide code design?

**Answer:**

```python
import this  # Prints The Zen of Python
```

Key principles and their practical application:

| Principle | Practical Application |
|-----------|----------------------|
| **Beautiful is better than ugly** | Clean formatting, meaningful names |
| **Explicit is better than implicit** | No hidden side effects, clear data flow |
| **Simple is better than complex** | Prefer straightforward solutions |
| **Flat is better than nested** | Early returns, guard clauses, avoid deep nesting |
| **Errors should never pass silently** | Don't use bare `except:`, always handle errors explicitly |
| **There should be one obvious way to do it** | Follow idioms (f-strings, comprehensions, context managers) |

```python
# Flat is better than nested
# Bad
def process(data):
    if data:
        if data.is_valid():
            if data.has_permission():
                return do_work(data)

# Good — guard clauses
def process(data):
    if not data:
        return None
    if not data.is_valid():
        raise ValueError("Invalid data")
    if not data.has_permission():
        raise PermissionError("No permission")
    return do_work(data)
```

---

### Q63: How would you make a Python class hashable?

**Answer:**

An object is hashable if it has a `__hash__()` method and an `__eq__()` method, and its hash doesn't change over its lifetime:

```python
class Employee:
    def __init__(self, emp_id: int, name: str):
        self.emp_id = emp_id
        self.name = name
    
    def __eq__(self, other):
        if not isinstance(other, Employee):
            return NotImplemented
        return self.emp_id == other.emp_id
    
    def __hash__(self):
        return hash(self.emp_id)

# Now can be used in sets and as dict keys
e1 = Employee(1, "Alice")
e2 = Employee(1, "Alice")
print(e1 == e2)           # True
print({e1, e2})           # One element — they're "equal"
print({e1: "engineer"})   # Valid dict key
```

**Rules:**
- If you define `__eq__`, you MUST define `__hash__` (or set `__hash__ = None` to make unhashable)
- Objects that compare equal MUST have the same hash
- Hash values should never change while in a set/dict
- Use `@dataclass(frozen=True)` for automatic `__hash__`

---

### Q64: What is `typing.TypeGuard` and how does it help with type narrowing?

**Answer:**

`TypeGuard` (Python 3.10+) tells type checkers that a function narrows a type:

```python
from typing import TypeGuard

def is_string_list(val: list[object]) -> TypeGuard[list[str]]:
    """If this returns True, type checkers know val is list[str]."""
    return all(isinstance(x, str) for x in val)

def process(data: list[object]) -> None:
    if is_string_list(data):
        # Type checker knows data is list[str] here
        print(data[0].upper())  # No type error!
    else:
        # data is still list[object] here
        pass
```

**Python 3.12+ — `TypeIs` (more precise narrowing):**
```python
from typing import TypeIs

def is_str(val: object) -> TypeIs[str]:
    return isinstance(val, str)
```

---

### Q65: Explain the `__init__.py` file and Python package structure.

**Answer:**

`__init__.py` marks a directory as a Python package and controls what gets exported:

```
mypackage/
├── __init__.py          # Package initializer
├── module_a.py
├── module_b.py
└── subpackage/
    ├── __init__.py
    └── module_c.py
```

```python
# mypackage/__init__.py
from .module_a import ClassA
from .module_b import function_b

__all__ = ['ClassA', 'function_b']  # Controls: from mypackage import *

# Version and metadata
__version__ = "1.0.0"

# Package-level initialization
print("mypackage loaded!")  # Runs on import
```

**Namespace packages (Python 3.3+):** Directories WITHOUT `__init__.py` can still be packages. This allows splitting a package across multiple directories.

```python
# Regular package — needs __init__.py
import mypackage.module_a

# Namespace package — no __init__.py needed
# Can span multiple directories on sys.path
```

---

*End of Python Theory — 65 questions covering fundamentals through MAANG-level topics.*
