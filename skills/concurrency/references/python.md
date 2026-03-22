# Python Concurrency Reference

Idiomatic Python concurrent patterns. The GIL limits threads to one executing Python bytecode at a time — use threads for I/O-bound work, multiprocessing for CPU-bound. asyncio provides cooperative concurrency for high-volume I/O.

---

## Shared Mutable State

```python
import threading

# threading.Lock for mutual exclusion — context manager ensures release
lock = threading.Lock()
shared_counter = 0

def increment():
    global shared_counter
    with lock:  # released even on exception
        shared_counter += 1

# Prefer confinement: threading.local gives each thread its own copy
thread_data = threading.local()

def worker():
    thread_data.conn = create_connection()  # thread-confined, no sharing
    thread_data.conn.execute("SELECT 1")
```

## Synchronization Primitives

```python
import threading

# Condition: wait for a predicate (handles spurious wakeups internally)
condition = threading.Condition()
buffer = []

def produce(item):
    with condition:
        buffer.append(item)
        condition.notify()

def consume():
    with condition:
        condition.wait_for(lambda: len(buffer) > 0, timeout=5.0)
        return buffer.pop(0) if buffer else None

# Semaphore: limit concurrent access
pool_sem = threading.Semaphore(value=5)

def access_resource():
    with pool_sem:  # blocks if 5 threads already inside
        do_work()
```

## Concurrency Patterns

```python
import queue
import concurrent.futures

# Producer-consumer with bounded queue (backpressure built in)
work_queue = queue.Queue(maxsize=100)

def producer():
    for item in generate_items():
        work_queue.put(item)  # blocks when full
    work_queue.put(None)  # sentinel

def consumer():
    while (item := work_queue.get()) is not None:
        process(item)
        work_queue.task_done()

# ThreadPoolExecutor for I/O-bound fan-out
def fetch_all(urls):
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        future_to_url = {executor.submit(fetch, url): url for url in urls}
        for future in concurrent.futures.as_completed(future_to_url):
            try:
                result = future.result(timeout=30)
            except Exception as exc:
                print(f"{future_to_url[future]} failed: {exc}")

# ProcessPoolExecutor for CPU-bound work (bypasses GIL)
def compute_all(datasets):
    with concurrent.futures.ProcessPoolExecutor() as executor:
        return list(executor.map(heavy_computation, datasets, timeout=120))
```

## Async and Non-Blocking I/O

```python
import asyncio

# Fan-out with gather
async def fetch_many(user_ids: list[int]) -> list[dict]:
    tasks = [fetch_user(uid) for uid in user_ids]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return [r for r in results if not isinstance(r, Exception)]

# Structured concurrency with TaskGroup (Python 3.11+)
async def process_batch(items: list[str]):
    async with asyncio.TaskGroup() as tg:
        # If any task raises, all others are cancelled
        for item in items:
            tg.create_task(process_item(item))

# Never block the event loop — offload blocking work
async def read_large_file(path: str) -> bytes:
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(None, _blocking_read, path)

# Bounded concurrency with asyncio.Semaphore
async def fetch_with_limit(urls: list[str], max_concurrent: int = 5):
    sem = asyncio.Semaphore(max_concurrent)
    async def limited_fetch(url):
        async with sem:
            return await fetch(url)
    return await asyncio.gather(*[limited_fetch(u) for u in urls])
```

## Reliability Patterns

```python
import asyncio, random

# Retry with exponential backoff and jitter
async def retry_with_backoff(coro_factory, max_retries=3, base_delay=1.0):
    for attempt in range(max_retries + 1):
        try:
            return await coro_factory()
        except (asyncio.TimeoutError, ConnectionError):
            if attempt == max_retries:
                raise
            delay = min(base_delay * (2 ** attempt), 30.0)
            await asyncio.sleep(delay + random.uniform(0, delay * 0.5))

# Timeout on every external call
async def call_service():
    try:
        return await asyncio.wait_for(external_api_call(), timeout=5.0)
    except asyncio.TimeoutError:
        return fallback_value()
```

## Testing Concurrent Code

```python
import concurrent.futures, unittest

class TestConcurrentCounter(unittest.TestCase):
    def test_counter_under_contention(self):
        counter = ThreadSafeCounter()
        num_threads, per_thread = 50, 1000
        with concurrent.futures.ThreadPoolExecutor(max_workers=num_threads) as ex:
            futures = [ex.submit(counter.increment_n, per_thread) for _ in range(num_threads)]
            for f in concurrent.futures.as_completed(futures):
                f.result()  # raises if the task raised
        self.assertEqual(counter.value, num_threads * per_thread)

# Async tests with pytest-asyncio
import pytest, asyncio

@pytest.mark.asyncio
async def test_timeout_enforced():
    with pytest.raises(asyncio.TimeoutError):
        await asyncio.wait_for(asyncio.sleep(10), timeout=0.1)
```
