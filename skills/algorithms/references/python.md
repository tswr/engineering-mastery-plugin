# Python Algorithms Reference

Idiomatic Python patterns for algorithmic problem-solving. Python's standard library provides `collections`, `heapq`, `bisect`, `itertools`, and `functools` — covering most common data structures and algorithms without third-party dependencies. Combined with built-in `dict`, `set`, and list comprehensions, most algorithmic patterns can be expressed concisely.

---

## Arrays, Strings, and Hash Tables

```python
from collections import defaultdict, Counter

# Counter: frequency counting in one call — O(n)
freq = Counter(["apple", "banana", "apple", "cherry"])
freq.most_common(2)  # [("apple", 2), ("banana", 1)]

# defaultdict: auto-initialize missing keys — avoids KeyError guards
graph = defaultdict(list)
for u, v in edges:
    graph[u].append(v)

# Two-sum with dict — O(n) via O(1) hash lookups
def two_sum(nums: list[int], target: int) -> tuple[int, int]:
    seen: dict[int, int] = {}  # value -> index
    for i, num in enumerate(nums):
        if (target - num) in seen:
            return (seen[target - num], i)
        seen[num] = i
    raise ValueError("no solution")

# set/frozenset: O(1) average membership testing
visited: set[str] = set()
STOP_WORDS = frozenset({"the", "a", "an", "is"})  # immutable, hashable
```

## Trees and Graphs

```python
from collections import deque

# BFS using deque — O(V + E), popleft is O(1)
def bfs_shortest(graph: dict[str, list[str]], start: str, end: str) -> int:
    queue = deque([(start, 0)])
    visited = {start}
    while queue:
        node, dist = queue.popleft()
        if node == end:
            return dist
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append((neighbor, dist + 1))
    return -1

# Topological sort — Kahn's algorithm with deque and in-degree counts
def topological_sort(graph: dict[str, list[str]]) -> list[str]:
    in_degree: dict[str, int] = defaultdict(int)
    for node in graph:
        for neighbor in graph[node]:
            in_degree[neighbor] += 1
    queue = deque(n for n in graph if in_degree[n] == 0)
    result: list[str] = []
    while queue:
        node = queue.popleft()
        result.append(node)
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)
    if len(result) != len(graph):
        raise ValueError("cycle detected")
    return result
```

## Sorting and Searching

```python
import bisect
import heapq

# bisect: binary search on sorted lists — O(log n)
sorted_vals = [2, 5, 8, 12, 16, 23, 38]
bisect.bisect_left(sorted_vals, 12)   # 3 — index of first >= 12
bisect.bisect_right(sorted_vals, 12)  # 4 — index of first > 12
bisect.insort(sorted_vals, 15)        # insert maintaining sort — O(n) shift

# heapq: min-heap backed by a list — O(log n) push/pop
tasks: list[tuple[int, str]] = []
heapq.heappush(tasks, (3, "low"))
heapq.heappush(tasks, (1, "high"))
heapq.heappop(tasks)  # (1, "high") — smallest first
# For max-heap, negate: heappush(tasks, (-priority, item))

# heapq.nlargest / nsmallest — O(n log k), better than sort for small k
top_3 = heapq.nlargest(3, items, key=lambda x: x.score)

# Merge k sorted lists — O(n log k)
merged = list(heapq.merge([1, 4, 7], [2, 5, 8], [3, 6, 9]))

# Custom sort: Python's Timsort is O(n log n), stable
intervals = [(1, 4), (2, 3), (0, 6)]
intervals.sort(key=lambda x: x[1])  # sort by end time
```

## Dynamic Programming

```python
from functools import lru_cache

# Top-down with lru_cache — automatic memoization
@lru_cache(maxsize=None)
def fib(n: int) -> int:
    if n < 2:
        return n
    return fib(n - 1) + fib(n - 2)  # repeated subproblems cached

# Edit distance — memoized recursion
@lru_cache(maxsize=None)
def edit_distance(s: str, t: str, i: int, j: int) -> int:
    if i == 0: return j
    if j == 0: return i
    if s[i - 1] == t[j - 1]:
        return edit_distance(s, t, i - 1, j - 1)
    return 1 + min(edit_distance(s, t, i - 1, j),
                   edit_distance(s, t, i, j - 1),
                   edit_distance(s, t, i - 1, j - 1))

# Bottom-up with space optimization — O(W) space instead of O(n*W)
def knapsack_01(weights: list[int], values: list[int], capacity: int) -> int:
    dp = [0] * (capacity + 1)
    for w, v in zip(weights, values):
        for c in range(capacity, w - 1, -1):  # right-to-left: use each item once
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[capacity]
```

## Choosing Data Structures

```python
from collections import OrderedDict
from itertools import combinations, permutations, product

# OrderedDict — O(1) move_to_end for LRU cache pattern
class LRUCache:
    def __init__(self, capacity: int) -> None:
        self._cache: OrderedDict[str, int] = OrderedDict()
        self._capacity = capacity
    def get(self, key: str) -> int:
        if key not in self._cache: return -1
        self._cache.move_to_end(key)  # O(1) — mark as recently used
        return self._cache[key]
    def put(self, key: str, value: int) -> None:
        if key in self._cache: self._cache.move_to_end(key)
        self._cache[key] = value
        if len(self._cache) > self._capacity:
            self._cache.popitem(last=False)  # evict LRU — O(1)

# itertools for combinatorial generation
list(combinations([1, 2, 3], 2))   # [(1,2), (1,3), (2,3)]
list(permutations([1, 2, 3], 2))   # [(1,2), (1,3), (2,1), ...]
list(product("AB", repeat=2))      # [('A','A'), ('A','B'), ('B','A'), ('B','B')]
```
