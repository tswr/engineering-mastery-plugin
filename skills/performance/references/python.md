# Python Performance Reference

Python profiling tools, benchmarking, and optimization patterns. Python's performance model is fundamentally different from compiled languages — the interpreter overhead dominates, so the primary strategy is moving hot loops out of Python into C extensions, vectorized operations, or compiled alternatives.

---

## Profiling

```python
# cProfile: function-level CPU profiling
import cProfile
import pstats

cProfile.run('main()', 'profile_output')
stats = pstats.Stats('profile_output')
stats.sort_stats('cumulative')
stats.print_stats(20)  # top 20 functions by cumulative time

# Or from the command line
# python -m cProfile -s cumulative my_script.py


# line_profiler: line-by-line profiling of specific functions
# pip install line_profiler
# Decorate functions with @profile, then run:
# kernprof -l -v my_script.py

@profile
def process_data(data):
    filtered = [x for x in data if x > threshold]  # see time per line
    sorted_data = sorted(filtered)
    return aggregate(sorted_data)


# memory_profiler: track memory usage per line
# pip install memory_profiler
from memory_profiler import profile

@profile
def load_dataset(path):
    with open(path) as f:
        data = json.load(f)       # see memory delta per line
    processed = transform(data)
    return processed


# py-spy: sampling profiler that works on running processes
# pip install py-spy
# py-spy record -o flame.svg -- python my_script.py
# py-spy top --pid 12345  # attach to running process
```

## Benchmarking with timeit

```python
import timeit

# Benchmark a function: measures wall clock, handles warmup and repetition
result = timeit.timeit(
    stmt='sorted(data)',
    setup='import random; data = [random.randint(0, 10000) for _ in range(10000)]',
    number=1000
)
print(f"Total: {result:.3f}s, Per call: {result/1000*1000:.3f}ms")

# In IPython/Jupyter: %timeit magic
# %timeit sorted(data)
# Reports mean ± std dev, handles iteration count automatically


# For more rigorous benchmarking: pytest-benchmark
# pip install pytest-benchmark
import pytest

def test_sort_performance(benchmark):
    data = list(range(10000, 0, -1))
    result = benchmark(sorted, data)
    assert result == sorted(data)  # also verify correctness
# Reports: min, max, mean, stddev, median, IQR, rounds


# Manual benchmark with warmup and statistics
import time
import statistics

def benchmark(func, *args, warmup=10, iterations=100):
    # Warmup
    for _ in range(warmup):
        func(*args)

    # Measure
    times = []
    for _ in range(iterations):
        start = time.perf_counter_ns()
        func(*args)
        elapsed = time.perf_counter_ns() - start
        times.append(elapsed)

    times_ms = [t / 1_000_000 for t in times]
    return {
        "p50": statistics.median(times_ms),
        "p90": sorted(times_ms)[int(0.9 * len(times_ms))],
        "p99": sorted(times_ms)[int(0.99 * len(times_ms))],
        "mean": statistics.mean(times_ms),
        "stdev": statistics.stdev(times_ms),
    }
```

## Vectorization with NumPy

```python
import numpy as np

# BAD: Python loop — interpreter overhead per element
def distances_loop(points, origin):
    result = []
    for p in points:
        d = math.sqrt((p[0]-origin[0])**2 + (p[1]-origin[1])**2)
        result.append(d)
    return result

# GOOD: NumPy vectorized — single C call, SIMD-friendly
def distances_vectorized(points, origin):
    diff = points - origin          # broadcast subtraction
    return np.sqrt(np.sum(diff**2, axis=1))  # vectorized ops

# Speedup: typically 50-200x for large arrays


# BAD: conditional logic in Python loop
def threshold_filter_loop(data, threshold):
    result = []
    for x in data:
        if x > threshold:
            result.append(x)
    return result

# GOOD: boolean indexing
def threshold_filter_vectorized(data, threshold):
    return data[data > threshold]  # no Python loop


# Preallocate output arrays instead of building with append
# BAD:
results = []
for i in range(n):
    results.append(compute(i))

# GOOD:
results = np.empty(n)
for i in range(n):
    results[i] = compute(i)

# BEST: vectorize the entire computation
results = compute_vectorized(np.arange(n))
```

## Common Python Performance Patterns

```python
# Use built-in data structures — they're implemented in C
# dict/set lookups are O(1) and fast in practice
lookup_set = set(large_list)  # O(1) membership test
if item in lookup_set: ...    # vs O(n) for 'if item in large_list'


# List comprehensions > explicit loops (compiler-optimized bytecode)
# BAD:
result = []
for x in data:
    if predicate(x):
        result.append(transform(x))

# GOOD:
result = [transform(x) for x in data if predicate(x)]


# str.join > string concatenation in loops
# BAD: O(n²) — each += copies the entire string
s = ""
for word in words:
    s += word + " "

# GOOD: O(n) — single allocation
s = " ".join(words)


# Use collections for specialized needs
from collections import deque, defaultdict, Counter

# deque for O(1) append/pop from both ends (list.pop(0) is O(n))
queue = deque()
queue.append(item)
item = queue.popleft()  # O(1), not O(n)


# functools.lru_cache for memoization
from functools import lru_cache

@lru_cache(maxsize=1024)
def expensive_lookup(key):
    return database.query(key)


# __slots__ to reduce memory per instance
class Point:
    __slots__ = ('x', 'y', 'z')
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z
# ~40% less memory than a regular class with __dict__


# Generator expressions for large datasets (avoid materializing full list)
# BAD: materializes entire list in memory
total = sum([compute(x) for x in huge_dataset])

# GOOD: generator — processes one element at a time
total = sum(compute(x) for x in huge_dataset)
```

## When to Leave Python

```python
# For CPU-bound hot loops that can't be vectorized with NumPy:

# Option 1: Cython — compile Python-like code to C
# Option 2: Numba — JIT compile numerical Python
from numba import njit

@njit
def monte_carlo_pi(n):
    count = 0
    for i in range(n):
        x = random.random()
        y = random.random()
        if x*x + y*y <= 1.0:
            count += 1
    return 4.0 * count / n
# First call compiles; subsequent calls run at C speed

# Option 3: ctypes/cffi — call existing C/C++ libraries
# Option 4: multiprocessing — bypass the GIL for CPU parallelism
from multiprocessing import Pool

with Pool(processes=4) as pool:
    results = pool.map(cpu_bound_function, data_chunks)
```

## Memory Optimization

```python
# Check object sizes
import sys
sys.getsizeof([1, 2, 3])          # list: ~120 bytes
sys.getsizeof((1, 2, 3))          # tuple: ~72 bytes
sys.getsizeof({1: 'a', 2: 'b'})   # dict: ~232 bytes

# For large datasets: use numpy arrays instead of lists
# list of 1M ints: ~28 MB
# numpy array of 1M int32s: ~4 MB

# tracemalloc: find memory allocation hotspots
import tracemalloc

tracemalloc.start()
# ... run code ...
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('lineno')[:10]:
    print(stat)
```
