# Python Advanced & Internals — Interview Questions & Answers

> Deep-dive into CPython internals, metaclasses, descriptors, memory layout, bytecode, C extensions, and advanced patterns. Targeted at senior/MAANG-level interviews.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What happens internally when you write `x = 10` in Python?

**Answer:**

1. CPython creates an integer object `10` on the heap (or retrieves it from the small integer cache for -5 to 256)
2. The object has: `ob_refcnt` (reference count), `ob_type` (pointer to `int` type), and `ob_digit` (the value)
3. The name `x` is stored in the local/global namespace dictionary
4. The namespace dict maps the string `"x"` to a pointer to the integer object

```python
import sys
import ctypes

x = 10
print(id(x))              # Memory address of the int object
print(sys.getrefcount(x)) # Reference count (high for small ints due to caching)

# Small integer caching
a = 256
b = 256
print(a is b)  # True — cached

a = 257
b = 257
print(a is b)  # False (typically) — not cached
```

**Memory layout of a Python int (CPython):**
```
PyLongObject:
  ob_refcnt:  Py_ssize_t  (8 bytes)
  ob_type:    PyTypeObject* (8 bytes)
  ob_size:    Py_ssize_t  (8 bytes) — number of digits
  ob_digit[]: uint32_t[]  (variable) — the actual number
```

A simple `int(10)` takes **28 bytes** on a 64-bit system!

---

### Q2: What is the difference between CPython, PyPy, Jython, and Cython?

**Answer:**

| Implementation | Language | GIL | Key Feature |
|---------------|----------|-----|-------------|
| **CPython** | C | Yes | Reference implementation, most widely used |
| **PyPy** | Python (RPython) | Yes (but JIT helps) | JIT compiler — 4-10x faster for long-running code |
| **Jython** | Java | No (uses JVM threads) | Runs on JVM, can import Java classes |
| **IronPython** | C# | No (uses .NET threads) | Runs on .NET CLR |
| **Cython** | C/Python hybrid | Can release GIL | Compiles Python-like code to C for speed |
| **MicroPython** | C | Yes | For microcontrollers and embedded systems |
| **GraalPython** | Java (Truffle) | No | Part of GraalVM, polyglot support |

**CPython** is the default — when someone says "Python," they almost always mean CPython.

---

### Q3: How does Python's string interning work?

**Answer:**

Python **interns** (caches and reuses) certain strings to save memory and speed up comparisons:

```python
# Automatically interned: strings that look like identifiers
a = "hello"
b = "hello"
print(a is b)  # True — interned

# NOT interned: strings with spaces or special chars
a = "hello world"
b = "hello world"
print(a is b)  # False (usually)

# Force interning
import sys
a = sys.intern("hello world")
b = sys.intern("hello world")
print(a is b)  # True — explicitly interned
```

**Interning rules (CPython):**
- All strings that look like valid Python identifiers (alphanumeric + underscore)
- String constants determined at compile time
- Dictionary keys (for fast lookup)
- Attribute names

**Why it matters:** Interned string comparison is O(1) (pointer comparison) instead of O(n) (character-by-character).

---

### Q4: What is Python bytecode?

**Answer:**

Python source code is compiled to **bytecode** — an intermediate, platform-independent instruction set for the Python Virtual Machine (PVM):

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
# Output:
#   2           0 LOAD_FAST                0 (a)
#               2 LOAD_FAST                1 (b)
#               4 BINARY_ADD
#               6 RETURN_VALUE
```

**Bytecode compilation flow:**
```
Source (.py) → Lexer → Parser → AST → Compiler → Bytecode (.pyc) → PVM executes
```

```python
# Inspect the code object
code = add.__code__
print(code.co_consts)     # Constants used
print(code.co_varnames)   # Local variable names
print(code.co_code)       # Raw bytecode bytes
print(code.co_stacksize)  # Max stack depth needed

# .pyc files are cached bytecode (in __pycache__/)
# Format: magic_number + timestamp + marshalled code object
```

---

### Q5: What is `__dict__` and how does attribute lookup work?

**Answer:**

Every Python object (by default) has a `__dict__` — a dictionary storing its attributes:

```python
class Dog:
    species = "Canine"  # Class attribute → Dog.__dict__
    
    def __init__(self, name):
        self.name = name  # Instance attribute → dog.__dict__

dog = Dog("Rex")
print(dog.__dict__)       # {'name': 'Rex'}
print(Dog.__dict__)       # {'species': 'Canine', '__init__': ..., ...}
```

**Attribute lookup order (simplified MRO):**
1. **Data descriptors** on the type (class) — e.g., `@property` with setter
2. **Instance `__dict__`**
3. **Non-data descriptors** and other class attributes
4. `__getattr__` (if defined) — fallback

```python
class MyClass:
    @property
    def x(self):  # Data descriptor — step 1
        return 42

obj = MyClass()
obj.__dict__['x'] = 99  # Instance dict — step 2
print(obj.x)  # 42 — data descriptor wins over instance dict!
```

---

### Q6: Explain the `__slots__` mechanism in detail.

**Answer:**

`__slots__` replaces the per-instance `__dict__` with a fixed-size struct:

```python
class SlottedPoint:
    __slots__ = ('x', 'y')
    
    def __init__(self, x, y):
        self.x = x
        self.y = y

# Internally, CPython creates descriptors for each slot
print(type(SlottedPoint.x))  # <class 'member_descriptor'>

p = SlottedPoint(1, 2)
# p.__dict__  # AttributeError — no __dict__!
# p.z = 3    # AttributeError — can't add attributes

# Memory savings
import sys
class Regular:
    def __init__(self, x, y):
        self.x = x; self.y = y

print(sys.getsizeof(Regular(1, 2)))       # ~48 bytes + __dict__ (~104 bytes)
print(sys.getsizeof(SlottedPoint(1, 2)))   # ~48 bytes (no __dict__)
```

**Inheritance with `__slots__`:**
```python
class Base:
    __slots__ = ('x',)

class Child(Base):
    __slots__ = ('y',)  # Only NEW slots — x is inherited
    
# If you forget __slots__ in Child, it gets a __dict__ anyway!
class BadChild(Base):
    pass  # Has __dict__ — defeats the purpose
```

**When NOT to use `__slots__`:**
- Classes that need dynamic attributes
- Classes using multiple inheritance with conflicting slots
- When you need `__dict__` for serialization (pickle, JSON)

---

### Q7: What are weak references and when do you use them?

**Answer:**

Weak references don't increase the reference count — they allow the referenced object to be garbage collected:

```python
import weakref

class ExpensiveObject:
    def __init__(self, name):
        self.name = name
    def __repr__(self):
        return f"ExpensiveObject({self.name!r})"

# Strong reference — keeps object alive
obj = ExpensiveObject("data")
strong_ref = obj  # refcount +1

# Weak reference — doesn't prevent GC
weak_ref = weakref.ref(obj)
print(weak_ref())  # ExpensiveObject('data') — object still alive

del obj, strong_ref
print(weak_ref())  # None — object was garbage collected!
```

**Practical use cases:**

```python
# 1. WeakValueDictionary — cache that doesn't prevent GC
cache = weakref.WeakValueDictionary()
obj = ExpensiveObject("cached")
cache["key"] = obj
del obj
# cache["key"] — KeyError! Object was collected.

# 2. Preventing circular references in tree structures
class TreeNode:
    def __init__(self, value, parent=None):
        self.value = value
        self.children = []
        self._parent = weakref.ref(parent) if parent else None
    
    @property
    def parent(self):
        return self._parent() if self._parent else None

# 3. Observer pattern — observers don't keep subjects alive
class EventManager:
    def __init__(self):
        self._listeners = weakref.WeakSet()
```

**Cannot create weak references to:** `int`, `str`, `tuple`, `list`, `dict`, `set`, `None`, `bool`.

---

## 🟡 Medium (Intermediate)

### Q8: Explain Python's memory allocator (pymalloc) in detail.

**Answer:**

CPython uses a **three-level memory architecture**:

```
Level 3:  Object-specific allocators (list, dict, int, etc.)
Level 2:  Python's pymalloc allocator (objects ≤ 512 bytes)
Level 1:  C's malloc/free (objects > 512 bytes)
Level 0:  OS virtual memory (mmap, brk)
```

**pymalloc's structure:**
- **Arenas:** 256 KB chunks from OS (the largest unit)
- **Pools:** 4 KB blocks within arenas (one per size class)
- **Blocks:** Individual allocations within pools (8, 16, 24, ... 512 bytes)

```python
# Size classes: blocks are allocated in multiples of 8 bytes
# Request 1-8 bytes → 8 byte block
# Request 9-16 bytes → 16 byte block
# Request 17-24 bytes → 24 byte block
# ...
# Request 505-512 bytes → 512 byte block
# Request > 512 bytes → uses malloc() directly
```

**Why pymalloc?**
- Reduces calls to OS malloc (expensive)
- Reduces memory fragmentation for small objects
- Most Python objects are small (ints, strings, tuples)

**Memory profiling:**
```python
import tracemalloc

tracemalloc.start()
# ... your code ...
current, peak = tracemalloc.get_traced_memory()
print(f"Current: {current / 1024:.1f} KB, Peak: {peak / 1024:.1f} KB")

snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('filename')[:5]:
    print(stat)
```

---

### Q9: How does Python's dictionary implementation work internally?

**Answer:**

Since Python 3.6, dicts use a **compact, ordered hash table** with separate dense and sparse arrays:

```
# Pre-3.6 implementation (unordered):
# Single hash table where each entry stores (hash, key, value)
# Lots of wasted space due to empty slots

# Python 3.6+ implementation (ordered, compact):
# Two arrays:

# 1. Indices array (sparse) — maps hash to index in entries
indices = [None, None, 1, None, None, 0, None, 2]
#                      ^                ^        ^
#                    "b"->1         "a"->0    "c"->2

# 2. Entries array (dense, compact) — stores actual data in insertion order
entries = [
    (hash("a"), "a", 1),   # index 0
    (hash("b"), "b", 2),   # index 1
    (hash("c"), "c", 3),   # index 2
]
```

**Key operations:**

| Operation | Average | Worst |
|-----------|---------|-------|
| `d[key]` | O(1) | O(n) |
| `d[key] = val` | O(1) amortized | O(n) |
| `del d[key]` | O(1) | O(n) |
| `key in d` | O(1) | O(n) |
| Iteration | O(n) | O(n) |

**Hash collision resolution:** Python uses **open addressing** with **probing** (not chaining). When a collision occurs:
```python
# Perturbation-based probing
j = hash_value
perturb = hash_value
while slot_occupied:
    j = (5 * j + perturb + 1) % table_size
    perturb >>= 5  # perturb converges to 0
```

**Dict resizing:** When the table is 2/3 full, Python resizes (typically doubles):
```python
# Load factor threshold: 2/3 (0.667)
# When entries > 2/3 * table_size → resize to next power of 2
```

---

### Q10: How does CPython implement lists internally?

**Answer:**

A Python `list` is implemented as a **dynamic array** of pointers (not values):

```c
// CPython source (simplified)
typedef struct {
    PyObject ob_base;           // refcount + type pointer
    Py_ssize_t ob_size;         // number of elements (len)
    PyObject **ob_item;         // pointer to array of PyObject pointers
    Py_ssize_t allocated;       // total allocated slots
} PyListObject;
```

**Over-allocation strategy:**
```python
# When a list needs to grow, CPython over-allocates:
# new_allocated = (newsize >> 3) + (3 if newsize < 9 else 6) + newsize
# This gives: 0, 4, 8, 16, 24, 32, 40, 52, 64, 76, ...

import sys
lst = []
for i in range(20):
    lst.append(i)
    print(f"len={len(lst):2d}, size={sys.getsizeof(lst)} bytes")
```

**Operation complexity:**

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `lst[i]` | O(1) | Direct pointer array access |
| `lst.append(x)` | O(1) amortized | Occasional resize (O(n) copy) |
| `lst.insert(i, x)` | O(n) | Shifts all elements after i |
| `lst.pop()` | O(1) | Remove from end |
| `lst.pop(i)` | O(n) | Shifts elements |
| `x in lst` | O(n) | Linear scan |
| `lst.sort()` | O(n log n) | Timsort |
| `lst[i:j]` | O(j-i) | Creates new list |

---

### Q11: What is the `__new__` vs `__init__` distinction and when does it matter?

**Answer:**

`__new__` **creates** the instance, `__init__` **initializes** it. For most classes, you only need `__init__`. `__new__` matters for:

**1. Immutable types (can't modify after creation):**
```python
class UpperStr(str):
    def __new__(cls, value):
        # Must use __new__ because str is immutable
        # __init__ is too late — the string is already created
        return super().__new__(cls, value.upper())

s = UpperStr("hello")
print(s)  # "HELLO"
```

**2. Singleton pattern:**
```python
class Singleton:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

a = Singleton()
b = Singleton()
print(a is b)  # True
```

**3. Controlling instance creation:**
```python
class Cached:
    _cache = {}
    
    def __new__(cls, value):
        if value not in cls._cache:
            instance = super().__new__(cls)
            cls._cache[value] = instance
        return cls._cache[value]
    
    def __init__(self, value):
        self.value = value
```

**Call order:** `Cached(42)` → `Cached.__new__(Cached, 42)` → `Cached.__init__(instance, 42)`

---

### Q12: How do Python generators work at the bytecode level?

**Answer:**

When Python encounters `yield`, it compiles the function into a generator that manages its own **execution frame**:

```python
def gen_func():
    x = 1
    yield x
    x += 1
    yield x

import dis
dis.dis(gen_func)
# Key bytecodes:
#   LOAD_CONST 1
#   STORE_FAST x
#   LOAD_FAST x
#   YIELD_VALUE    ← saves frame state, returns value
#   RESUME         ← resumes from saved state
#   ...
#   YIELD_VALUE
```

**Internal state machine:**
```python
gen = gen_func()

# State: GEN_CREATED — not started yet
print(gen.gi_frame.f_lineno)  # Points to function start

next(gen)  # Executes until first yield
# State: GEN_SUSPENDED — paused at yield, frame preserved

# The generator preserves:
# - Local variables (gen.gi_frame.f_locals)
# - Instruction pointer (gen.gi_frame.f_lasti)
# - Stack state
# - Exception state

next(gen)  # Resumes, executes until second yield

try:
    next(gen)  # No more yields
except StopIteration:
    pass
# State: GEN_CLOSED — frame deallocated
```

**`yield from` optimization:**
```python
# yield from delegates to a sub-generator
# It handles: next(), send(), throw(), close() forwarding
def chain(*iterables):
    for it in iterables:
        yield from it  # Optimized C-level delegation
```

---

### Q13: Explain how `@property` works using the descriptor protocol.

**Answer:**

`property` is a **data descriptor** implemented in C. Here's a pure Python equivalent:

```python
class MyProperty:
    """Pure Python implementation of @property."""
    
    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
        self.__doc__ = doc or (fget.__doc__ if fget else None)
    
    def __get__(self, obj, objtype=None):
        if obj is None:
            return self  # Accessed from class, return descriptor itself
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)
    
    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)
    
    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)
    
    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel, self.__doc__)
    
    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel, self.__doc__)
    
    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel, self.__doc__)

class Circle:
    def __init__(self, radius):
        self._radius = radius
    
    @MyProperty
    def radius(self):
        return self._radius
    
    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius must be non-negative")
        self._radius = value

c = Circle(5)
print(c.radius)    # Calls MyProperty.__get__ → fget(c)
c.radius = 10      # Calls MyProperty.__set__ → fset(c, 10)
```

**Why data descriptor wins over instance `__dict__`:** Because CPython's `type.__getattribute__` checks for data descriptors on the class **before** checking the instance dict. This is why `@property` can intercept attribute access even when an instance has a same-named key in its `__dict__`.

---

### Q14: What are `__getattr__`, `__getattribute__`, `__setattr__`, and `__delattr__`?

**Answer:**

```python
class Proxy:
    """Demonstrates all attribute access hooks."""
    
    def __init__(self, target):
        # Must use object.__setattr__ to avoid infinite recursion
        object.__setattr__(self, '_target', target)
    
    def __getattribute__(self, name):
        """Called for EVERY attribute access."""
        print(f"__getattribute__({name})")
        if name == '_target':
            return object.__getattribute__(self, name)
        target = object.__getattribute__(self, '_target')
        return getattr(target, name)
    
    def __getattr__(self, name):
        """Called ONLY when __getattribute__ raises AttributeError."""
        print(f"__getattr__({name})")
        return f"<default for {name}>"
    
    def __setattr__(self, name, value):
        """Called for EVERY attribute assignment."""
        print(f"__setattr__({name}, {value})")
        object.__setattr__(self, name, value)
    
    def __delattr__(self, name):
        """Called for EVERY attribute deletion."""
        print(f"__delattr__({name})")
        object.__delattr__(self, name)
```

**Lookup chain:**
```
obj.attr
  → type(obj).__getattribute__(obj, 'attr')
    → Check data descriptors on type(obj) and its MRO
    → Check obj.__dict__
    → Check non-data descriptors on type(obj) and its MRO
    → If not found: type(obj).__getattr__(obj, 'attr')
    → If still not found: raise AttributeError
```

---

### Q15: How does Python's `import` system handle circular imports?

**Answer:**

```python
# When Python imports a module, it:
# 1. Creates a module object and adds it to sys.modules
# 2. Executes the module's code top-to-bottom
# 3. During execution, the module is PARTIALLY initialized

# module_a.py
import module_b  # module_a is partially initialized here

x = 10

def func_a():
    return module_b.y  # Works — accessed at call time, not import time

# module_b.py
import module_a  # Gets the PARTIALLY initialized module_a

y = 20

def func_b():
    return module_a.x  # Works IF module_a.x is defined by call time
```

**Why `from module_a import x` fails but `import module_a` works:**
```python
# This fails during circular import:
from module_a import x
# Because x doesn't exist yet in the partially-initialized module

# This works:
import module_a
# Because module_a exists in sys.modules (partially initialized)
# And x will be available by the time you access module_a.x at runtime
```

**Best practices to avoid circular imports:**
1. Import at function level (lazy import)
2. Use `TYPE_CHECKING` guard for type hints
3. Restructure to break the cycle
4. Use `import module` instead of `from module import name`

---

## 🔴 Hard (Advanced / MAANG-level)

### Q16: How does CPython's frame object work? What happens during a function call?

**Answer:**

Every function call creates a **frame object** that represents the execution state:

```python
import sys
import inspect

def outer():
    x = 10
    frame = sys._getframe()
    print(f"Function: {frame.f_code.co_name}")
    print(f"Locals: {frame.f_locals}")
    print(f"Line number: {frame.f_lineno}")
    print(f"Bytecode offset: {frame.f_lasti}")
    print(f"Caller: {frame.f_back.f_code.co_name}")
    inner()

def inner():
    # Walk the call stack
    frame = sys._getframe()
    while frame:
        print(f"  {frame.f_code.co_name} at line {frame.f_lineno}")
        frame = frame.f_back

outer()
```

**Frame object structure (CPython):**
```c
typedef struct _frame {
    PyObject_HEAD
    struct _frame *f_back;     // Previous frame (caller)
    PyCodeObject *f_code;      // Code object being executed
    PyObject *f_builtins;      // Built-in namespace
    PyObject *f_globals;       // Global namespace
    PyObject *f_locals;        // Local namespace (may be dict or array)
    PyObject **f_valuestack;   // Evaluation stack
    int f_lasti;               // Last bytecode instruction index
    int f_lineno;              // Current line number
    // ... more fields
} PyFrameObject;
```

**Python 3.11+ optimization:** Frames are no longer heap-allocated for most calls. They use a **frame stack** in the interpreter, only creating heap frames when needed (e.g., `sys._getframe()`).

---

### Q17: Explain Python's garbage collector generations in depth.

**Answer:**

```python
import gc

# Three generations with configurable thresholds
print(gc.get_threshold())  # (700, 10, 10)
# Gen 0: collected when allocations - deallocations >= 700
# Gen 1: collected every 10 Gen 0 collections
# Gen 2: collected every 10 Gen 1 collections

# View generation statistics
print(gc.get_stats())
# [{'collections': 85, 'collected': 480, 'uncollectable': 0},
#  {'collections': 7, 'collected': 36, 'uncollectable': 0},
#  {'collections': 1, 'collected': 0, 'uncollectable': 0}]
```

**The cycle detection algorithm (tricolor marking variation):**

1. **Compute tentative reference count:** For each object in the generation, subtract internal references (references from other objects in the same generation). This gives the "external reference count."

2. **Identify roots:** Objects with external refcount > 0 are reachable from outside the generation → they are "roots."

3. **Trace from roots:** DFS/BFS from roots, marking everything reachable as "alive."

4. **Collect unreachable:** Everything not marked as alive is part of a reference cycle → can be freed.

```python
# Objects with __del__ (finalizers) — special handling
class HasFinalizer:
    def __del__(self):
        print("Cleaning up!")

# Python 3.4+: Objects with __del__ in cycles CAN be collected
# But the order of __del__ calls is undefined
# The gc module calls finalizers in arbitrary order

# Monitoring GC
gc.set_debug(gc.DEBUG_STATS | gc.DEBUG_LEAK)

# Callbacks when GC runs
def gc_callback(phase, info):
    if phase == "start":
        print(f"GC starting gen {info['generation']}")
    else:
        print(f"GC done: collected {info['collected']}")

gc.callbacks.append(gc_callback)
```

---

### Q18: How does `async/await` work at the bytecode level?

**Answer:**

```python
async def fetch_data():
    result = await get_from_db()
    return result

import dis
dis.dis(fetch_data)
# Key bytecodes:
#   RETURN_GENERATOR     ← creates the coroutine object
#   POP_TOP
#   RESUME 0
#   ...
#   CALL                 ← calls get_from_db()
#   GET_AWAITABLE        ← ensures result is awaitable
#   LOAD_CONST None
#   SEND                 ← sends to awaitable's __next__/send
#   YIELD_VALUE          ← suspends coroutine, returns to event loop
#   RESUME 1             ← resumes when awaitable completes
```

**Under the hood:**
```python
# async def creates a coroutine function
# Calling it returns a coroutine object
coro = fetch_data()

# Coroutine objects have:
# - cr_frame: current execution frame
# - cr_code: code object
# - cr_origin: traceback origin
# - cr_await: the object being awaited (or None)

# The event loop drives coroutines:
# 1. loop.run_until_complete(coro)
# 2. Event loop calls coro.send(None) to start
# 3. Coroutine runs until YIELD_VALUE (at an await point)
# 4. Event loop registers for I/O completion
# 5. When I/O completes, event loop calls coro.send(result)
# 6. Coroutine resumes from RESUME
```

---

### Q19: What are `__class_getitem__` and `__mro_entries__`?

**Answer:**

```python
# __class_getitem__ — enables subscript syntax on classes
class MyList:
    def __class_getitem__(cls, item):
        """MyList[int] calls this."""
        print(f"Subscripted with: {item}")
        # In stdlib, this creates a GenericAlias
        return f"MyList[{item.__name__}]"

print(MyList[int])  # "MyList[int]"

# This is how list[int], dict[str, int] work in Python 3.9+
# They call list.__class_getitem__(int)
```

```python
# __mro_entries__ — customize MRO for non-type bases
# Used by Generic[T] to insert itself into the MRO correctly

from typing import Generic, TypeVar
T = TypeVar('T')

class Stack(Generic[T]):
    """Generic[T] uses __mro_entries__ to properly participate in MRO."""
    pass

print(Stack.__mro__)
# (<class 'Stack'>, <class 'typing.Generic'>, <class 'object'>)
```

---

### Q20: How does Python 3.12+ `type` statement work?

**Answer:**

Python 3.12 introduced the `type` statement for creating type aliases with **lazy evaluation**:

```python
# Python 3.12+ type statement
type Vector = list[float]
type Matrix[T] = list[list[T]]  # Generic alias with type params

# Equivalent to (but lazily evaluated):
# Vector = TypeAliasType("Vector", list[float])

# The alias is lazy — the RHS is not evaluated until accessed
type Recursive = list["Recursive"]  # Forward reference works!

# Type parameter syntax (PEP 695)
def first[T](lst: list[T]) -> T:
    return lst[0]

class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []
    
    def push(self, item: T) -> None:
        self._items.append(item)
    
    def pop(self) -> T:
        return self._items.pop()

# Before 3.12:
from typing import TypeVar
T = TypeVar('T')
class OldStack(Generic[T]):
    ...
```

---

### Q21: Explain Python's eval loop and the specializing adaptive interpreter (Python 3.11+).

**Answer:**

Python 3.11 introduced the **specializing adaptive interpreter** that speeds up frequently executed bytecode:

```python
# Python 3.11+ uses ADAPTIVE bytecodes
# After a few executions, generic opcodes are replaced with specialized ones:

# LOAD_ATTR → LOAD_ATTR_INSTANCE_VALUE (for instance attribute access)
# LOAD_ATTR → LOAD_ATTR_MODULE (for module attribute access)
# BINARY_OP  → BINARY_OP_ADD_INT (for int + int)
# BINARY_OP  → BINARY_OP_ADD_FLOAT (for float + float)
# COMPARE_OP → COMPARE_OP_INT (for int comparisons)

# If the specialization fails (wrong type), it falls back to generic
# and increments a counter. After too many failures, it gives up.
```

**Python 3.13 additions:**
```python
# Copy-and-patch JIT compiler (experimental)
# Compiles hot bytecode to machine code
# Enabled with: python -X jit

# Free-threaded mode (no GIL)
# python -X gil=0
```

---

### Q22: How do you write C extensions for Python?

**Answer:**

```c
// example.c — Python C extension
#include <Python.h>

// The C function
static PyObject* fast_add(PyObject* self, PyObject* args) {
    long a, b;
    if (!PyArg_ParseTuple(args, "ll", &a, &b))
        return NULL;
    return PyLong_FromLong(a + b);
}

// Method table
static PyMethodDef ExampleMethods[] = {
    {"fast_add", fast_add, METH_VARARGS, "Add two integers quickly."},
    {NULL, NULL, 0, NULL}  // Sentinel
};

// Module definition
static struct PyModuleDef examplemodule = {
    PyModuleDef_HEAD_INIT,
    "example",              // Module name
    "Fast math operations", // Docstring
    -1,                     // State size (-1 = global state)
    ExampleMethods          // Method table
};

// Module initialization function
PyMODINIT_FUNC PyInit_example(void) {
    return PyModule_Create(&examplemodule);
}
```

**Modern alternatives to C extensions:**
```python
# 1. ctypes — call C libraries directly
import ctypes
libc = ctypes.CDLL("libc.so.6")  # or msvcrt on Windows

# 2. cffi — C Foreign Function Interface
from cffi import FFI
ffi = FFI()
ffi.cdef("int printf(const char *format, ...);")
C = ffi.dlopen(None)

# 3. Cython — Python-like syntax compiled to C
# example.pyx
# def fast_add(long a, long b):
#     return a + b

# 4. pybind11 — C++ bindings
# The most popular modern choice for C++ extensions
```

---

### Q23: What is the `__subclasshook__` method and how does it enable virtual subclasses?

**Answer:**

`__subclasshook__` allows an ABC to customize `isinstance()` and `issubclass()` checks:

```python
from abc import ABCMeta, abstractmethod

class Printable(metaclass=ABCMeta):
    @abstractmethod
    def __str__(self):
        pass
    
    @classmethod
    def __subclasshook__(cls, C):
        """Any class with __str__ is considered a Printable."""
        if cls is Printable:
            if any("__str__" in B.__dict__ for B in C.__mro__):
                return True
        return NotImplemented

# No explicit inheritance!
class MyClass:
    def __str__(self):
        return "I'm printable"

print(isinstance(MyClass(), Printable))  # True
print(issubclass(MyClass, Printable))     # True
```

**How `collections.abc.Iterable` uses this:**
```python
from collections.abc import Iterable

# Any class with __iter__ is considered Iterable
class MyIterable:
    def __iter__(self):
        return iter([1, 2, 3])

print(isinstance(MyIterable(), Iterable))  # True — via __subclasshook__
```

---

### Q24: Explain Python's `__prepare__` in metaclasses and its use in ordered class definitions.

**Answer:**

`__prepare__` (PEP 3115) returns the namespace dict used while executing the class body. By default, it returns a regular dict. You can return a custom mapping:

```python
class OrderedMeta(type):
    @classmethod
    def __prepare__(mcs, name, bases):
        """Return a custom namespace that tracks definition order."""
        from collections import OrderedDict
        return OrderedDict()
    
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, dict(namespace))
        cls._field_order = [
            key for key in namespace
            if not key.startswith('_')
        ]
        return cls

class Record(metaclass=OrderedMeta):
    name = "string"
    age = "integer"
    email = "string"

print(Record._field_order)  # ['name', 'age', 'email'] — guaranteed order
```

**Use case — Enum-like class:**
```python
class AutoNumberMeta(type):
    @classmethod
    def __prepare__(mcs, name, bases):
        class AutoDict(dict):
            def __init__(self):
                super().__init__()
                self._counter = 0
            
            def __missing__(self, key):
                if key.startswith('_'):
                    raise KeyError(key)
                self._counter += 1
                self[key] = self._counter
                return self._counter
        
        return AutoDict()

class Color(metaclass=AutoNumberMeta):
    RED       # Auto-assigned 1
    GREEN     # Auto-assigned 2
    BLUE      # Auto-assigned 3

print(Color.RED, Color.GREEN, Color.BLUE)  # 1 2 3
```

---

### Q25: How does CPython's `peephole optimizer` work?

**Answer:**

CPython applies **compile-time optimizations** to bytecode:

```python
# 1. Constant folding
x = 2 * 3 * 4  # Compiled as x = 24 (computed at compile time)

import dis
dis.dis(compile("x = 2 * 3 * 4", "", "exec"))
#   LOAD_CONST  24
#   STORE_NAME  x

# 2. Dead code elimination
if False:
    print("never runs")  # Removed at compile time

# 3. Tuple of constants → frozenset for 'in' checks
if x in (1, 2, 3):  # Compiled as: if x in frozenset({1, 2, 3})
    pass

# 4. String concatenation of literals
s = "hello" + " " + "world"  # Compiled as: s = "hello world"

# 5. Boolean simplification
if not (a and b):  # May be optimized
    pass
```

**Python 3.13+ AST optimizer** performs more aggressive optimizations before bytecode generation.

---

### Q26: What is `sys.settrace` and how do debuggers work?

**Answer:**

`sys.settrace` installs a trace function called for every line, call, return, and exception:

```python
import sys

def trace_calls(frame, event, arg):
    """Trace function — called by CPython at various events."""
    if event == "call":
        print(f"CALL: {frame.f_code.co_name} at {frame.f_code.co_filename}:{frame.f_lineno}")
    elif event == "line":
        print(f"LINE: {frame.f_lineno} in {frame.f_code.co_name}")
    elif event == "return":
        print(f"RETURN: {arg} from {frame.f_code.co_name}")
    elif event == "exception":
        exc_type, exc_value, exc_tb = arg
        print(f"EXCEPTION: {exc_type.__name__}: {exc_value}")
    return trace_calls  # Return itself to continue tracing

sys.settrace(trace_calls)

def add(a, b):
    result = a + b
    return result

add(1, 2)
sys.settrace(None)  # Disable tracing
```

**How debuggers like pdb/PyCharm use this:**
1. Install trace function via `sys.settrace`
2. At breakpoints, pause execution and enter interactive mode
3. Use frame object to inspect/modify local variables
4. Use `frame.f_lineno` for step-over/step-into
5. Python 3.12+ added `sys.monitoring` (PEP 669) — more efficient

---

### Q27: How does `copy.deepcopy` handle complex object graphs?

**Answer:**

`deepcopy` uses a **memo dictionary** to handle shared references and cycles:

```python
import copy

class Node:
    def __init__(self, value, children=None):
        self.value = value
        self.children = children or []

# Create a graph with shared references
shared = Node("shared")
root = Node("root", [shared, shared])
root.children[0].children.append(root)  # Circular reference!

# deepcopy handles this correctly
root_copy = copy.deepcopy(root)

# Shared reference preserved (not duplicated)
assert root_copy.children[0] is root_copy.children[1]

# Circular reference preserved (not infinite loop)
assert root_copy.children[0].children[0] is root_copy
```

**Custom `__deepcopy__`:**
```python
class Resource:
    def __init__(self, name, connection):
        self.name = name
        self.connection = connection  # Don't copy connections!
    
    def __deepcopy__(self, memo):
        """Custom deep copy — share the connection."""
        new = Resource.__new__(Resource)
        memo[id(self)] = new  # Register in memo FIRST (handles cycles)
        new.name = copy.deepcopy(self.name, memo)
        new.connection = self.connection  # Share, don't copy
        return new
```

**The algorithm:**
1. Check if object is in `memo` → return existing copy (handles cycles)
2. Check for `__deepcopy__` method → use custom logic
3. Check for `__copy__` (for shallow copy with `copy.copy`)
4. For built-in types: use type-specific copier
5. For others: create new instance, add to memo, then deepcopy all attributes

---

### Q28: What changed in Python 3.13's free-threaded mode?

**Answer:**

Python 3.13 introduced an **experimental** no-GIL build:

```python
# Build with: ./configure --disable-gil
# Run with: python -X gil=0

import sys
print(sys._is_gil_enabled())  # False in free-threaded mode
```

**Key changes to enable no-GIL:**

1. **Biased reference counting:**
   - Object creator uses a fast, thread-local counter
   - Other threads use slower atomic operations
   - Merges happen during GC

2. **Per-object locks:**
   - Critical sections use per-object mutexes instead of the GIL
   - Most operations are lock-free using atomic operations

3. **Deferred reference counting:**
   - Some frequently-shared objects (modules, types) use deferred RC
   - Their refcount is not tracked until GC

4. **Immortal objects:**
   - `None`, `True`, `False`, small ints are "immortal"
   - Their refcount is never modified (avoids contention)

5. **Thread-safe collections:**
   - `dict`, `list` operations are made thread-safe with fine-grained locking

**Impact:**
- True parallelism for CPU-bound threaded code
- ~10-40% single-threaded slowdown (overhead of atomic ops)
- Some C extensions need updates for thread safety
- Still experimental — not recommended for production

---

### Q29: Explain Python's descriptor protocol — non-data vs data descriptors in detail.

**Answer:**

```python
class NonDataDescriptor:
    """Only implements __get__ — instance dict takes priority."""
    def __get__(self, obj, objtype=None):
        print("Non-data descriptor __get__")
        return 42

class DataDescriptor:
    """Implements __get__ AND __set__ — takes priority over instance dict."""
    def __get__(self, obj, objtype=None):
        print("Data descriptor __get__")
        return 42
    
    def __set__(self, obj, value):
        print(f"Data descriptor __set__({value})")

class MyClass:
    non_data = NonDataDescriptor()
    data = DataDescriptor()

obj = MyClass()

# Non-data descriptor — instance dict WINS
obj.__dict__['non_data'] = 99
print(obj.non_data)  # 99 — from instance dict, NOT descriptor

# Data descriptor — descriptor WINS
obj.__dict__['data'] = 99
print(obj.data)  # 42 — from descriptor, NOT instance dict
```

**Complete attribute lookup algorithm (`type.__getattribute__`):**
```python
def object_getattribute(obj, name):
    """Simplified CPython attribute lookup."""
    # 1. Get the object's type (class)
    obj_type = type(obj)
    
    # 2. Look up name in the type's MRO
    descriptor = None
    for base in obj_type.__mro__:
        if name in base.__dict__:
            descriptor = base.__dict__[name]
            break
    
    # 3. Check if it's a DATA descriptor (has __set__ or __delete__)
    if descriptor is not None:
        descriptor_get = getattr(type(descriptor), '__get__', None)
        descriptor_set = getattr(type(descriptor), '__set__', None)
        descriptor_delete = getattr(type(descriptor), '__delete__', None)
        
        if descriptor_set is not None or descriptor_delete is not None:
            # DATA descriptor — highest priority
            if descriptor_get:
                return descriptor_get(descriptor, obj, obj_type)
    
    # 4. Check instance __dict__
    if hasattr(obj, '__dict__') and name in obj.__dict__:
        return obj.__dict__[name]
    
    # 5. Non-data descriptor or plain class attribute
    if descriptor is not None:
        descriptor_get = getattr(type(descriptor), '__get__', None)
        if descriptor_get:
            return descriptor_get(descriptor, obj, obj_type)
        return descriptor
    
    # 6. Not found
    raise AttributeError(name)
```

---

### Q30: How does Python implement closures at the bytecode level?

**Answer:**

Closures use **cell objects** to share variables between scopes:

```python
def outer(x):
    def inner():
        return x  # 'x' is a "free variable" captured by cell
    return inner

import dis

# Outer function bytecode
dis.dis(outer)
# MAKE_CELL         x     ← creates a cell object for x
# ...
# LOAD_CLOSURE      x     ← loads the cell
# BUILD_TUPLE       1     ← packs cells into a tuple
# MAKE_FUNCTION            ← creates function with closure

closure = outer(42)

# Inspect the closure
print(closure.__closure__)          # (<cell at 0x...>,)
print(closure.__closure__[0].cell_contents)  # 42
print(closure.__code__.co_freevars)  # ('x',) — captured variable names

# Inner function bytecode
dis.dis(closure)
# LOAD_DEREF  x   ← loads value from cell object
# RETURN_VALUE
```

**Cell objects enable shared mutable state:**
```python
def counter():
    count = 0
    def increment():
        nonlocal count
        count += 1  # Modifies the SAME cell object
        return count
    def get():
        return count  # Reads from the SAME cell object
    return increment, get

inc, get = counter()
inc(); inc(); inc()
print(get())  # 3 — both functions share the same cell
```

---

*End of Python Advanced & Internals — 30 questions covering CPython internals, bytecode, memory management, and advanced features.*
