# Data Structures & Algorithms — Theory

> Covers Big-O analysis, data structure internals, algorithm paradigms, and space-time trade-offs. Essential for MAANG coding rounds.

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: What is Big-O notation? Explain the common complexity classes.

**Answer:**

Big-O describes the **upper bound** of an algorithm's growth rate as input size increases, ignoring constants and lower-order terms.

| Complexity | Name | Example | 1M inputs |
|-----------|------|---------|-----------|
| O(1) | Constant | Hash table lookup | Instant |
| O(log n) | Logarithmic | Binary search | ~20 ops |
| O(n) | Linear | Array scan | 1M ops |
| O(n log n) | Linearithmic | Merge sort | ~20M ops |
| O(n²) | Quadratic | Bubble sort | 1T ops (💀) |
| O(2ⁿ) | Exponential | Subsets generation | Heat death |
| O(n!) | Factorial | Permutations | Universe ends |

**Big-O vs Big-Ω vs Big-Θ:**
- **O(n)** — Upper bound (worst case at most)
- **Ω(n)** — Lower bound (best case at least)
- **Θ(n)** — Tight bound (both upper and lower)

```python
# Amortized analysis example: list.append()
# Worst case: O(n) — when resizing occurs
# Amortized: O(1) — resizing happens infrequently
# n appends cost O(n) total → O(1) per append on average
```

---

### Q2: What is the difference between an Array and a Linked List?

**Answer:**

| Feature | Array (Python list) | Linked List |
|---------|-------------------|-------------|
| Memory | Contiguous block | Scattered nodes |
| Access by index | O(1) — direct offset | O(n) — must traverse |
| Insert at beginning | O(n) — shift all | O(1) — update pointer |
| Insert at end | O(1) amortized | O(1) with tail pointer, O(n) without |
| Insert at middle | O(n) — shift elements | O(1) after finding node |
| Delete | O(n) — shift elements | O(1) after finding node |
| Cache performance | Excellent (spatial locality) | Poor (scattered memory) |
| Memory overhead | Low (just pointers to objects) | High (each node has next/prev pointers) |

**When to use Linked List:** Frequent insertions/deletions at arbitrary positions, implementing queues/stacks, when you can't predict size.

**When to use Array:** Random access needed, cache-friendly iteration, known or predictable size.

---

### Q3: Explain Stack and Queue data structures.

**Answer:**

**Stack — LIFO (Last In, First Out):**
```python
# Python implementation: use list (append/pop from end)
stack = []
stack.append(1)  # push — O(1)
stack.append(2)
stack.pop()      # pop — O(1), returns 2

# Or use collections.deque
from collections import deque
stack = deque()
stack.append(1)
stack.pop()
```

**Queue — FIFO (First In, First Out):**
```python
# DON'T use list (pop(0) is O(n)!)
# Use collections.deque
from collections import deque
queue = deque()
queue.append(1)     # enqueue — O(1)
queue.append(2)
queue.popleft()     # dequeue — O(1), returns 1

# Thread-safe: queue.Queue
import queue
q = queue.Queue()
q.put(1)
q.get()
```

**Applications:**
- **Stack:** Function call stack, undo/redo, bracket matching, DFS
- **Queue:** BFS, task scheduling, message queues, print spooling

---

### Q4: What is a Hash Table? How do collisions work?

**Answer:**

A hash table maps keys to values using a **hash function** to compute the index:

```
key → hash(key) → index = hash(key) % table_size → value
```

**Collision Resolution:**

| Strategy | Mechanism | Pros | Cons |
|----------|-----------|------|------|
| **Chaining** | Each bucket holds a linked list | Simple, works well with high load | Extra memory for pointers |
| **Open Addressing** | Probe for next empty slot | Cache-friendly, no extra pointers | Clustering, complex deletion |
| **Linear Probing** | Try index+1, index+2, ... | Simple | Primary clustering |
| **Quadratic Probing** | Try index+1², index+2², ... | Reduces clustering | Secondary clustering |
| **Double Hashing** | Use second hash function | Best distribution | More computation |

**Python's dict uses open addressing** with perturbation-based probing.

**Time complexity:**

| Operation | Average | Worst (all collisions) |
|-----------|---------|----------------------|
| Insert | O(1) | O(n) |
| Search | O(1) | O(n) |
| Delete | O(1) | O(n) |

**Load factor** = n_items / table_size. Python resizes at 2/3 (~0.667).

---

### Q5: What is a Binary Search Tree (BST)?

**Answer:**

A BST is a binary tree where for every node:
- All values in the **left subtree** are **less than** the node's value
- All values in the **right subtree** are **greater than** the node's value

```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13
```

```python
class BSTNode:
    def __init__(self, val):
        self.val = val
        self.left = None
        self.right = None

def insert(root, val):
    if not root:
        return BSTNode(val)
    if val < root.val:
        root.left = insert(root.left, val)
    else:
        root.right = insert(root.right, val)
    return root

def search(root, val):
    if not root or root.val == val:
        return root
    if val < root.val:
        return search(root.left, val)
    return search(root.right, val)
```

| Operation | Average (balanced) | Worst (skewed) |
|-----------|-------------------|----------------|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(n) |
| Delete | O(log n) | O(n) |

**Self-balancing BSTs** (AVL, Red-Black) guarantee O(log n) worst case.

---

### Q6: What is a Heap? Explain Min-Heap and Max-Heap.

**Answer:**

A **heap** is a complete binary tree satisfying the heap property:
- **Min-Heap:** Parent ≤ children (smallest at root)
- **Max-Heap:** Parent ≥ children (largest at root)

```python
import heapq

# Python's heapq is a MIN-HEAP
min_heap = []
heapq.heappush(min_heap, 3)
heapq.heappush(min_heap, 1)
heapq.heappush(min_heap, 4)
heapq.heappush(min_heap, 1)

print(heapq.heappop(min_heap))  # 1 (smallest)
print(heapq.heappop(min_heap))  # 1

# For MAX-HEAP: negate values
max_heap = []
heapq.heappush(max_heap, -3)
heapq.heappush(max_heap, -1)
print(-heapq.heappop(max_heap))  # 3 (largest)

# heapq utilities
nums = [5, 3, 8, 1, 9, 2]
heapq.heapify(nums)            # In-place, O(n)
print(heapq.nlargest(3, nums))  # [9, 8, 5]
print(heapq.nsmallest(3, nums)) # [1, 2, 3]
```

| Operation | Complexity |
|-----------|-----------|
| Insert (push) | O(log n) |
| Extract min/max (pop) | O(log n) |
| Peek min/max | O(1) |
| Build heap (heapify) | O(n) |

**Applications:** Priority queues, Dijkstra's algorithm, median finding, K-th largest element, task scheduling.

---

### Q7: What is the difference between BFS and DFS?

**Answer:**

| Feature | BFS (Breadth-First) | DFS (Depth-First) |
|---------|--------------------|--------------------|
| Data structure | Queue | Stack (or recursion) |
| Strategy | Level by level | Go deep first |
| Space | O(width) — can be O(n) | O(depth) — O(log n) for balanced trees |
| Shortest path? | ✅ (unweighted graphs) | ❌ |
| Complete? | ✅ (finds solution if exists) | ❌ (can get stuck in infinite branches) |

```python
from collections import deque

# BFS
def bfs(graph, start):
    visited = set()
    queue = deque([start])
    visited.add(start)
    result = []
    
    while queue:
        node = queue.popleft()
        result.append(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return result

# DFS (iterative)
def dfs(graph, start):
    visited = set()
    stack = [start]
    result = []
    
    while stack:
        node = stack.pop()
        if node not in visited:
            visited.add(node)
            result.append(node)
            for neighbor in reversed(graph[node]):
                if neighbor not in visited:
                    stack.append(neighbor)
    return result

# DFS (recursive)
def dfs_recursive(graph, node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs_recursive(graph, neighbor, visited)
    return visited
```

---

### Q8: Explain sorting algorithms — which to use when?

**Answer:**

| Algorithm | Best | Average | Worst | Space | Stable? | Use Case |
|-----------|------|---------|-------|-------|---------|----------|
| **Bubble Sort** | O(n) | O(n²) | O(n²) | O(1) | ✅ | Educational only |
| **Selection Sort** | O(n²) | O(n²) | O(n²) | O(1) | ❌ | Small arrays, minimize swaps |
| **Insertion Sort** | O(n) | O(n²) | O(n²) | O(1) | ✅ | Nearly sorted data, small n |
| **Merge Sort** | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ | Guaranteed O(n log n), linked lists |
| **Quick Sort** | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ | General purpose, cache-friendly |
| **Heap Sort** | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ | In-place, guaranteed O(n log n) |
| **Timsort** | O(n) | O(n log n) | O(n log n) | O(n) | ✅ | Python's default — hybrid merge+insertion |
| **Counting Sort** | O(n+k) | O(n+k) | O(n+k) | O(k) | ✅ | Small range integers |
| **Radix Sort** | O(d·n) | O(d·n) | O(d·n) | O(n+k) | ✅ | Fixed-length integers/strings |

**Python uses Timsort:** Hybrid of merge sort + insertion sort. Exploits existing sorted runs in data.

```python
# Python sorting
arr = [3, 1, 4, 1, 5, 9, 2, 6]
sorted_arr = sorted(arr)      # New list — Timsort
arr.sort()                     # In-place — Timsort
arr.sort(key=lambda x: -x)    # Descending
```

---

### Q9: What is a Graph? Explain representations.

**Answer:**

A graph G = (V, E) consists of vertices V and edges E.

**Representations:**

```python
# 1. Adjacency List (most common — space efficient for sparse graphs)
graph = {
    'A': ['B', 'C'],
    'B': ['A', 'D', 'E'],
    'C': ['A', 'F'],
    'D': ['B'],
    'E': ['B', 'F'],
    'F': ['C', 'E']
}

# 2. Adjacency Matrix (good for dense graphs, O(1) edge lookup)
#     A  B  C  D  E  F
# A [[0, 1, 1, 0, 0, 0],
# B  [1, 0, 0, 1, 1, 0],
# C  [1, 0, 0, 0, 0, 1],
# D  [0, 1, 0, 0, 0, 0],
# E  [0, 1, 0, 0, 0, 1],
# F  [0, 0, 1, 0, 1, 0]]

# 3. Edge List
edges = [('A','B'), ('A','C'), ('B','D'), ('B','E'), ('C','F'), ('E','F')]
```

| Representation | Space | Edge lookup | Add edge | Iterate neighbors |
|---------------|-------|-------------|----------|-------------------|
| Adjacency List | O(V+E) | O(degree) | O(1) | O(degree) |
| Adjacency Matrix | O(V²) | O(1) | O(1) | O(V) |
| Edge List | O(E) | O(E) | O(1) | O(E) |

---

### Q10: What is Dynamic Programming? Explain with an example.

**Answer:**

**Dynamic Programming (DP)** solves problems by breaking them into overlapping subproblems and storing results to avoid recomputation.

**Two approaches:**

```python
# Example: Fibonacci

# 1. Top-down (Memoization) — recursive + cache
def fib_memo(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib_memo(n - 1) + fib_memo(n - 2)
    return memo[n]

# 2. Bottom-up (Tabulation) — iterative + table
def fib_tab(n):
    if n <= 1:
        return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]
    return dp[n]

# 3. Space-optimized — O(1) space
def fib_opt(n):
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b
```

**When to use DP:**
1. **Optimal substructure** — optimal solution uses optimal solutions to subproblems
2. **Overlapping subproblems** — same subproblems are solved multiple times

---

## 🟡 Medium (Intermediate)

### Q11: Explain the two-pointer technique with examples.

**Answer:**

Two pointers move through the data structure (usually in opposite directions or at different speeds) to solve problems efficiently.

**Pattern 1 — Opposite ends (sorted array):**
```python
def two_sum_sorted(nums: list[int], target: int) -> list[int]:
    left, right = 0, len(nums) - 1
    while left < right:
        total = nums[left] + nums[right]
        if total == target:
            return [left, right]
        elif total < target:
            left += 1
        else:
            right -= 1
    return []
```

**Pattern 2 — Slow/fast (linked list cycle detection):**
```python
def has_cycle(head) -> bool:
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False
```

**Pattern 3 — Same direction (remove duplicates):**
```python
def remove_duplicates(nums: list[int]) -> int:
    if not nums:
        return 0
    write = 1
    for read in range(1, len(nums)):
        if nums[read] != nums[read - 1]:
            nums[write] = nums[read]
            write += 1
    return write
```

---

### Q12: Explain the sliding window technique.

**Answer:**

Sliding window maintains a window (subarray/substring) that expands or shrinks to find optimal results.

**Fixed-size window:**
```python
def max_sum_subarray(nums: list[int], k: int) -> int:
    """Maximum sum of subarray of size k."""
    window_sum = sum(nums[:k])
    max_sum = window_sum
    
    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]  # Slide window
        max_sum = max(max_sum, window_sum)
    
    return max_sum
# Time: O(n), Space: O(1)
```

**Variable-size window:**
```python
def longest_substring_k_distinct(s: str, k: int) -> int:
    """Longest substring with at most k distinct characters."""
    from collections import defaultdict
    
    char_count = defaultdict(int)
    left = max_len = 0
    
    for right in range(len(s)):
        char_count[s[right]] += 1
        
        while len(char_count) > k:  # Shrink window
            char_count[s[left]] -= 1
            if char_count[s[left]] == 0:
                del char_count[s[left]]
            left += 1
        
        max_len = max(max_len, right - left + 1)
    
    return max_len
# Time: O(n), Space: O(k)
```

---

### Q13: What is a Trie? When would you use it?

**Answer:**

A **Trie** (prefix tree) is a tree-like data structure for storing strings where each node represents a character:

```
        root
       / | \
      a  b  c
     /   |
    p    a
   / \   |
  p   r  t  ← "bat"
  |
  l
  |
  e  ← "apple"
```

| Operation | Trie | Hash Set | Sorted Array |
|-----------|------|----------|--------------|
| Search | O(L) | O(L) avg | O(L log n) |
| Insert | O(L) | O(L) avg | O(n) |
| Prefix search | O(P) | O(n·P) | O(log n + k) |
| Autocomplete | O(P + k) | O(n) | O(log n + k) |

Where L = string length, P = prefix length, k = results count.

**Use cases:** Autocomplete, spell checking, IP routing, word games, prefix-based filtering.

---

### Q14: Explain Dijkstra's algorithm.

**Answer:**

Dijkstra finds the **shortest path from a source to all other vertices** in a weighted graph with non-negative edges.

```python
import heapq

def dijkstra(graph: dict, start: str) -> dict:
    """
    graph: {node: [(neighbor, weight), ...]}
    Returns: {node: shortest_distance}
    """
    distances = {node: float('inf') for node in graph}
    distances[start] = 0
    pq = [(0, start)]  # (distance, node)
    visited = set()
    
    while pq:
        dist, node = heapq.heappop(pq)
        
        if node in visited:
            continue
        visited.add(node)
        
        for neighbor, weight in graph[node]:
            new_dist = dist + weight
            if new_dist < distances[neighbor]:
                distances[neighbor] = new_dist
                heapq.heappush(pq, (new_dist, neighbor))
    
    return distances

graph = {
    'A': [('B', 1), ('C', 4)],
    'B': [('C', 2), ('D', 5)],
    'C': [('D', 1)],
    'D': []
}
print(dijkstra(graph, 'A'))  # {'A': 0, 'B': 1, 'C': 3, 'D': 4}
# Time: O((V + E) log V), Space: O(V)
```

**Limitations:** Cannot handle negative edge weights (use Bellman-Ford instead).

---

### Q15: What is a Union-Find (Disjoint Set Union) data structure?

**Answer:**

Union-Find tracks elements partitioned into disjoint sets, supporting:
- **Find:** Which set does an element belong to?
- **Union:** Merge two sets

```python
class UnionFind:
    def __init__(self, n: int):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.count = n  # Number of connected components
    
    def find(self, x: int) -> int:
        """Find root with path compression."""
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]
    
    def union(self, x: int, y: int) -> bool:
        """Union by rank. Returns False if already connected."""
        px, py = self.find(x), self.find(y)
        if px == py:
            return False
        if self.rank[px] < self.rank[py]:
            px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]:
            self.rank[px] += 1
        self.count -= 1
        return True
    
    def connected(self, x: int, y: int) -> bool:
        return self.find(x) == self.find(y)

# With path compression + union by rank:
# Find: O(α(n)) ≈ O(1) amortized (inverse Ackermann)
# Union: O(α(n)) ≈ O(1) amortized
```

**Use cases:** Kruskal's MST, connected components, cycle detection, social network groups.

---

### Q16: Explain topological sort and when it's used.

**Answer:**

**Topological sort** orders vertices in a DAG (Directed Acyclic Graph) such that for every edge u→v, u comes before v.

```python
from collections import deque, defaultdict

def topological_sort_bfs(graph: dict) -> list:
    """Kahn's algorithm (BFS-based)."""
    in_degree = defaultdict(int)
    for node in graph:
        for neighbor in graph[node]:
            in_degree[neighbor] += 1
    
    queue = deque([n for n in graph if in_degree[n] == 0])
    result = []
    
    while queue:
        node = queue.popleft()
        result.append(node)
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)
    
    if len(result) != len(graph):
        raise ValueError("Graph has a cycle!")
    return result

# DFS-based
def topological_sort_dfs(graph: dict) -> list:
    visited = set()
    stack = []
    
    def dfs(node):
        visited.add(node)
        for neighbor in graph.get(node, []):
            if neighbor not in visited:
                dfs(neighbor)
        stack.append(node)  # Post-order
    
    for node in graph:
        if node not in visited:
            dfs(node)
    
    return stack[::-1]  # Reverse post-order

# Time: O(V + E), Space: O(V)
```

**Use cases:** Build systems (make), course prerequisites, task scheduling, dependency resolution.

---

### Q17: What is a Balanced BST? Explain AVL vs Red-Black trees.

**Answer:**

| Feature | AVL Tree | Red-Black Tree |
|---------|----------|----------------|
| Balance | Strictly balanced (height diff ≤ 1) | Approximately balanced |
| Search | Faster (stricter balance) | Slightly slower |
| Insert/Delete | Slower (more rotations) | Faster (fewer rotations) |
| Rotations per operation | Up to O(log n) | At most 2-3 |
| Use case | Read-heavy workloads | Write-heavy workloads |
| Used in | Databases, some maps | Java TreeMap, Linux kernel, C++ std::map |

**Python's `sortedcontainers` library** provides efficient sorted collections:
```python
from sortedcontainers import SortedList, SortedDict, SortedSet

sl = SortedList([5, 1, 3])
sl.add(2)      # O(log n)
sl.remove(3)   # O(log n)
print(sl)      # SortedList([1, 2, 5])
```

---

### Q18: Explain the greedy algorithm paradigm.

**Answer:**

Greedy algorithms make the **locally optimal choice** at each step, hoping to find the global optimum. They work when the problem has:
1. **Greedy choice property** — local optimal leads to global optimal
2. **Optimal substructure** — optimal solution contains optimal sub-solutions

```python
# Example: Activity Selection (max non-overlapping activities)
def activity_selection(activities: list[tuple[int, int]]) -> list:
    """activities = [(start, end), ...]"""
    # Sort by end time (greedy choice: pick earliest-ending activity)
    activities.sort(key=lambda x: x[1])
    
    selected = [activities[0]]
    last_end = activities[0][1]
    
    for start, end in activities[1:]:
        if start >= last_end:
            selected.append((start, end))
            last_end = end
    
    return selected

# Example: Fractional Knapsack
def fractional_knapsack(items: list[tuple], capacity: float) -> float:
    """items = [(value, weight), ...], returns max value."""
    # Sort by value/weight ratio (greedy: best ratio first)
    items.sort(key=lambda x: x[0]/x[1], reverse=True)
    
    total_value = 0
    for value, weight in items:
        if capacity >= weight:
            total_value += value
            capacity -= weight
        else:
            total_value += value * (capacity / weight)
            break
    return total_value
```

**Greedy works for:** Activity selection, Huffman coding, Kruskal/Prim MST, Dijkstra's shortest path.
**Greedy DOESN'T work for:** 0/1 Knapsack, Longest path, Traveling salesman (use DP instead).

---

### Q19: What is backtracking? How does it differ from brute force?

**Answer:**

**Backtracking** is an optimized brute force that **prunes** invalid paths early. It builds solutions incrementally and abandons a path as soon as it determines it can't lead to a valid solution.

```python
# Example: N-Queens — place N queens on N×N board so no two attack each other
def solve_n_queens(n: int) -> list[list[str]]:
    results = []
    
    def is_valid(board, row, col):
        for r in range(row):
            if board[r] == col:  # Same column
                return False
            if abs(board[r] - col) == abs(r - row):  # Same diagonal
                return False
        return True
    
    def backtrack(board, row):
        if row == n:
            results.append(board[:])
            return
        
        for col in range(n):
            if is_valid(board, row, col):
                board[row] = col
                backtrack(board, row + 1)  # Explore
                board[row] = -1            # Backtrack (undo)
    
    backtrack([-1] * n, 0)
    return results

# Brute force: try all n^n placements
# Backtracking: prune invalid placements early → much faster
```

**Template:**
```python
def backtrack(state, choices):
    if is_solution(state):
        record(state)
        return
    
    for choice in choices:
        if is_valid(state, choice):
            make_choice(state, choice)
            backtrack(state, remaining_choices)
            undo_choice(state, choice)  # Backtrack
```

---

### Q20: Explain the concept of amortized analysis.

**Answer:**

Amortized analysis computes the **average cost per operation over a sequence** of operations, even though individual operations may be expensive.

**Example — Dynamic array (Python list) append:**

```python
# Most appends are O(1), but occasionally O(n) when resizing
# Resizing doubles the capacity: 1 → 2 → 4 → 8 → 16 → 32 → ...
# 
# For n appends:
# - Number of copies due to resizing: 1 + 2 + 4 + 8 + ... + n ≈ 2n
# - Total cost: n (regular appends) + 2n (copies) = 3n
# - Amortized cost per append: 3n / n = O(1)
```

**Three methods of amortized analysis:**
1. **Aggregate method:** Total cost / n operations
2. **Accounting method:** Assign "charges" to operations, save credits for future expensive ones
3. **Potential method:** Define a potential function that captures accumulated savings

---

## 🔴 Hard (Advanced / MAANG-level)

### Q21: Explain segment trees and their applications.

**Answer:**

A **segment tree** is a binary tree for efficient range queries and point updates on an array:

```python
class SegmentTree:
    """Range Sum Query with point updates."""
    
    def __init__(self, nums: list[int]):
        self.n = len(nums)
        self.tree = [0] * (4 * self.n)
        self._build(nums, 0, 0, self.n - 1)
    
    def _build(self, nums, node, start, end):
        if start == end:
            self.tree[node] = nums[start]
            return
        mid = (start + end) // 2
        self._build(nums, 2*node+1, start, mid)
        self._build(nums, 2*node+2, mid+1, end)
        self.tree[node] = self.tree[2*node+1] + self.tree[2*node+2]
    
    def update(self, idx: int, val: int):
        self._update(0, 0, self.n-1, idx, val)
    
    def _update(self, node, start, end, idx, val):
        if start == end:
            self.tree[node] = val
            return
        mid = (start + end) // 2
        if idx <= mid:
            self._update(2*node+1, start, mid, idx, val)
        else:
            self._update(2*node+2, mid+1, end, idx, val)
        self.tree[node] = self.tree[2*node+1] + self.tree[2*node+2]
    
    def query(self, l: int, r: int) -> int:
        return self._query(0, 0, self.n-1, l, r)
    
    def _query(self, node, start, end, l, r):
        if r < start or end < l:
            return 0
        if l <= start and end <= r:
            return self.tree[node]
        mid = (start + end) // 2
        return (self._query(2*node+1, start, mid, l, r) +
                self._query(2*node+2, mid+1, end, l, r))

# Build: O(n), Query: O(log n), Update: O(log n)
```

**Applications:** Range sum/min/max queries, interval scheduling, computational geometry, lazy propagation for range updates.

---

### Q22: What is the difference between Kruskal's and Prim's algorithms for MST?

**Answer:**

Both find the **Minimum Spanning Tree** but use different approaches:

| Feature | Kruskal's | Prim's |
|---------|-----------|--------|
| Strategy | Sort edges, add smallest that doesn't create cycle | Grow tree from a vertex, add nearest vertex |
| Data structure | Union-Find | Min-Heap + adjacency list |
| Complexity | O(E log E) | O(E log V) with binary heap |
| Better for | Sparse graphs (E ≈ V) | Dense graphs (E ≈ V²) |

```python
# Kruskal's Algorithm
def kruskal(n: int, edges: list[tuple]) -> list:
    """edges = [(weight, u, v)]"""
    edges.sort()  # Sort by weight
    uf = UnionFind(n)
    mst = []
    
    for weight, u, v in edges:
        if uf.union(u, v):  # No cycle
            mst.append((weight, u, v))
            if len(mst) == n - 1:
                break
    return mst

# Prim's Algorithm
def prim(graph: dict, start: str) -> list:
    """graph = {node: [(weight, neighbor), ...]}"""
    visited = set()
    mst = []
    heap = [(0, start, None)]  # (weight, node, parent)
    
    while heap and len(visited) < len(graph):
        weight, node, parent = heapq.heappop(heap)
        if node in visited:
            continue
        visited.add(node)
        if parent is not None:
            mst.append((weight, parent, node))
        for w, neighbor in graph[node]:
            if neighbor not in visited:
                heapq.heappush(heap, (w, neighbor, node))
    return mst
```

---

### Q23: Explain the Knuth-Morris-Pratt (KMP) string matching algorithm.

**Answer:**

KMP finds a pattern in text in O(n+m) time by precomputing a **failure function** (partial match table) to avoid re-scanning characters:

```python
def kmp_search(text: str, pattern: str) -> list[int]:
    """Find all occurrences of pattern in text."""
    # Build failure function (longest proper prefix that is also suffix)
    def build_lps(pattern):
        lps = [0] * len(pattern)
        length = 0
        i = 1
        while i < len(pattern):
            if pattern[i] == pattern[length]:
                length += 1
                lps[i] = length
                i += 1
            else:
                if length != 0:
                    length = lps[length - 1]
                else:
                    lps[i] = 0
                    i += 1
        return lps
    
    lps = build_lps(pattern)
    results = []
    i = j = 0  # i = text index, j = pattern index
    
    while i < len(text):
        if text[i] == pattern[j]:
            i += 1
            j += 1
        
        if j == len(pattern):
            results.append(i - j)
            j = lps[j - 1]
        elif i < len(text) and text[i] != pattern[j]:
            if j != 0:
                j = lps[j - 1]  # Skip based on failure function
            else:
                i += 1
    
    return results

print(kmp_search("AABAACAADAABAABA", "AABA"))  # [0, 9, 12]
# Time: O(n + m), Space: O(m)
```

---

### Q24: What are B-Trees and B+ Trees? Why are they used in databases?

**Answer:**

**B-Tree:** Self-balancing tree optimized for disk access. Each node can have multiple keys and children.

| Feature | B-Tree | B+ Tree |
|---------|--------|---------|
| Data storage | In all nodes | Only in leaf nodes |
| Leaf linking | No | Linked list of leaves |
| Range queries | Slow (must traverse tree) | Fast (follow leaf pointers) |
| Fanout | Lower | Higher (internal nodes have more keys) |
| Used in | General purpose | Database indexes (PostgreSQL, MySQL) |

**Why B-Trees for databases:**
1. **Minimize disk I/O:** Large fanout (100s of children per node) → tree is very shallow (3-4 levels for millions of records)
2. **Node size = disk page:** Each node fits in one disk read (4-16 KB)
3. **Range queries efficient:** B+ tree leaf linked list enables sequential scanning
4. **Always balanced:** O(log_B n) for all operations, where B is the branching factor

---

### Q25: Explain the A* search algorithm.

**Answer:**

A* is an informed search algorithm that combines Dijkstra's (actual cost) with a heuristic (estimated remaining cost):

```python
import heapq

def a_star(graph, start, goal, heuristic):
    """
    graph: {node: [(neighbor, cost), ...]}
    heuristic: {node: estimated_distance_to_goal}
    f(n) = g(n) + h(n)
    """
    open_set = [(heuristic[start], 0, start, [start])]  # (f, g, node, path)
    visited = set()
    
    while open_set:
        f, g, current, path = heapq.heappop(open_set)
        
        if current == goal:
            return path, g
        
        if current in visited:
            continue
        visited.add(current)
        
        for neighbor, cost in graph[current]:
            if neighbor not in visited:
                new_g = g + cost
                new_f = new_g + heuristic[neighbor]
                heapq.heappush(open_set, (new_f, new_g, neighbor, path + [neighbor]))
    
    return None, float('inf')

# A* guarantees optimal path if heuristic is:
# - Admissible: h(n) ≤ actual cost (never overestimates)
# - Consistent: h(n) ≤ cost(n, n') + h(n') (triangle inequality)
```

**A* is optimal and complete** if heuristic is admissible. It's used in GPS navigation, game pathfinding, robotics, and puzzle solving.

---

### Q26: What is a Bloom Filter?

**Answer:**

A **Bloom filter** is a space-efficient probabilistic data structure that tests set membership:
- **No false negatives:** If it says "not in set," the element is definitely not in the set
- **Possible false positives:** If it says "maybe in set," the element might not be in the set

```python
import hashlib

class BloomFilter:
    def __init__(self, size: int, num_hashes: int):
        self.size = size
        self.num_hashes = num_hashes
        self.bit_array = [False] * size
    
    def _hashes(self, item: str) -> list[int]:
        """Generate multiple hash values."""
        results = []
        for i in range(self.num_hashes):
            h = hashlib.md5(f"{item}_{i}".encode()).hexdigest()
            results.append(int(h, 16) % self.size)
        return results
    
    def add(self, item: str) -> None:
        for idx in self._hashes(item):
            self.bit_array[idx] = True
    
    def might_contain(self, item: str) -> bool:
        return all(self.bit_array[idx] for idx in self._hashes(item))

bf = BloomFilter(size=1000, num_hashes=3)
bf.add("apple")
bf.add("banana")
print(bf.might_contain("apple"))   # True (definitely)
print(bf.might_contain("cherry"))  # False (definitely not) or True (false positive)
```

**Applications:** Database query optimization, cache filtering, spam detection, network routers.

---

### Q27: Explain the concept of consistent hashing.

**Answer:**

Consistent hashing distributes data across nodes in a way that **minimizes redistribution** when nodes are added or removed.

**Traditional hashing:** `server = hash(key) % n_servers` — Adding/removing a server reassigns ~all keys.

**Consistent hashing:** Map both keys and servers to a circle (0 to 2^32-1). Each key is assigned to the **next server clockwise** on the circle.

```python
import hashlib
from bisect import bisect_right

class ConsistentHash:
    def __init__(self, nodes: list[str], replicas: int = 100):
        self.replicas = replicas
        self.ring = []
        self.node_map = {}
        
        for node in nodes:
            self.add_node(node)
    
    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def add_node(self, node: str):
        for i in range(self.replicas):
            virtual_key = f"{node}:{i}"
            h = self._hash(virtual_key)
            self.ring.append(h)
            self.node_map[h] = node
        self.ring.sort()
    
    def remove_node(self, node: str):
        for i in range(self.replicas):
            h = self._hash(f"{node}:{i}")
            self.ring.remove(h)
            del self.node_map[h]
    
    def get_node(self, key: str) -> str:
        h = self._hash(key)
        idx = bisect_right(self.ring, h) % len(self.ring)
        return self.node_map[self.ring[idx]]

# When a node is added/removed, only K/N keys are redistributed
# (K = total keys, N = total nodes)
```

**Used in:** DynamoDB, Cassandra, Memcached, CDNs, load balancers.

---

### Q28: What is a Skip List?

**Answer:**

A **skip list** is a probabilistic data structure that provides O(log n) search, insertion, and deletion — like a balanced BST but simpler to implement.

```
Level 3: HEAD ──────────────────────────────────→ 50 ──→ NIL
Level 2: HEAD ──────────→ 20 ─────────────────→ 50 ──→ NIL
Level 1: HEAD ──→ 10 ──→ 20 ──→ 30 ──→ 40 ──→ 50 ──→ NIL
Level 0: HEAD ──→ 10 ──→ 20 ──→ 30 ──→ 40 ──→ 50 ──→ NIL
```

Each element exists at level 0. With probability p (typically 0.5), it's "promoted" to the next level. This creates a layered structure where upper levels act as "express lanes."

**Used in:** Redis sorted sets (ZSET), LevelDB, MemSQL.

| Operation | Average | Space |
|-----------|---------|-------|
| Search | O(log n) | O(n) |
| Insert | O(log n) | O(log n) per element |
| Delete | O(log n) | — |

---

### Q29: Explain the difference between Bellman-Ford and Dijkstra's algorithms.

**Answer:**

| Feature | Dijkstra's | Bellman-Ford |
|---------|-----------|--------------|
| Edge weights | Non-negative only | Any (including negative) |
| Negative cycles | Cannot detect | ✅ Detects |
| Complexity | O((V+E) log V) | O(V·E) |
| Algorithm type | Greedy | Dynamic programming |
| Use case | GPS, network routing | Currency arbitrage, graph with negative weights |

```python
def bellman_ford(n: int, edges: list, source: int) -> dict:
    """edges = [(u, v, weight)]"""
    dist = {i: float('inf') for i in range(n)}
    dist[source] = 0
    
    # Relax all edges V-1 times
    for _ in range(n - 1):
        for u, v, w in edges:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
    
    # Check for negative cycles (V-th relaxation)
    for u, v, w in edges:
        if dist[u] + w < dist[v]:
            raise ValueError("Negative cycle detected!")
    
    return dist
```

---

### Q30: What is the time complexity of common Python operations?

**Answer:**

**List operations:**
| Operation | Complexity |
|-----------|-----------|
| `lst[i]` | O(1) |
| `lst.append(x)` | O(1) amortized |
| `lst.pop()` | O(1) |
| `lst.pop(0)` / `lst.insert(0, x)` | O(n) |
| `lst.extend(iterable)` | O(k) |
| `x in lst` | O(n) |
| `lst.sort()` | O(n log n) |
| `lst[i:j]` | O(j-i) |
| `lst.remove(x)` | O(n) |
| `min(lst)` / `max(lst)` | O(n) |

**Dict operations:**
| Operation | Average | Worst |
|-----------|---------|-------|
| `d[key]` | O(1) | O(n) |
| `d[key] = val` | O(1) | O(n) |
| `key in d` | O(1) | O(n) |
| `del d[key]` | O(1) | O(n) |
| Iteration | O(n) | O(n) |

**Set operations:**
| Operation | Average | Worst |
|-----------|---------|-------|
| `x in s` | O(1) | O(n) |
| `s.add(x)` | O(1) | O(n) |
| `s1 \| s2` (union) | O(len(s1)+len(s2)) | — |
| `s1 & s2` (intersection) | O(min(len(s1),len(s2))) | — |
| `s1 - s2` (difference) | O(len(s1)) | — |

**Deque operations:**
| Operation | Complexity |
|-----------|-----------|
| `deque.append(x)` | O(1) |
| `deque.appendleft(x)` | O(1) |
| `deque.pop()` | O(1) |
| `deque.popleft()` | O(1) |
| `deque[i]` | O(n) |

---

*End of DSA Theory — 30 questions covering data structures, algorithms, and complexity analysis.*
