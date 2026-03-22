# C++ Concurrency Reference

Idiomatic C++ concurrent patterns. The standard library provides threads, mutexes, atomics, and futures since C++11. RAII lock guards prevent forgetting to unlock. C++20 adds jthread with cooperative cancellation, latches, barriers, and semaphores.

---

## Shared Mutable State

```cpp
#include <mutex>
#include <shared_mutex>

// Prefer confinement: thread_local gives each thread its own copy
thread_local std::vector<int> scratch_buffer;

// Mutex + lock_guard: RAII ensures unlock even on exception
class ThreadSafeCounter {
    mutable std::mutex mtx_;
    int count_ = 0;
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx_);
        ++count_;
    }
    int get() const {
        std::lock_guard<std::mutex> lock(mtx_);  // reads need sync too
        return count_;
    }
};

// shared_mutex: multiple concurrent readers, exclusive writers
class Config {
    mutable std::shared_mutex mtx_;
    std::unordered_map<std::string, std::string> data_;
public:
    std::string get(const std::string& key) const {
        std::shared_lock lock(mtx_);  // multiple readers allowed
        auto it = data_.find(key);
        return it != data_.end() ? it->second : "";
    }
    void set(const std::string& key, const std::string& value) {
        std::unique_lock lock(mtx_);  // exclusive writer
        data_[key] = value;
    }
};
```

## Synchronization Primitives

```cpp
#include <condition_variable>
#include <atomic>
#include <mutex>
#include <queue>

// Condition variable: bounded producer-consumer queue
template <typename T>
class BoundedQueue {
    std::mutex mtx_;
    std::condition_variable not_full_, not_empty_;
    std::queue<T> queue_;
    size_t capacity_;
public:
    explicit BoundedQueue(size_t cap) : capacity_(cap) {}
    void push(T item) {
        std::unique_lock lock(mtx_);
        not_full_.wait(lock, [this] { return queue_.size() < capacity_; });
        queue_.push(std::move(item));
        not_empty_.notify_one();
    }
    T pop() {
        std::unique_lock lock(mtx_);
        not_empty_.wait(lock, [this] { return !queue_.empty(); });
        T item = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one();
        return item;
    }
};

// scoped_lock: deadlock-free acquisition of multiple mutexes
std::mutex mtx_a, mtx_b;
void safe_transfer() {
    std::scoped_lock lock(mtx_a, mtx_b);  // acquires both without deadlock risk
}

// Atomics: lock-free single-value operations
std::atomic<int> request_count{0};
std::atomic<bool> shutdown_flag{false};

void handle_request() {
    request_count.fetch_add(1, std::memory_order_relaxed);  // relaxed OK for counters
}
void request_shutdown() {
    shutdown_flag.store(true, std::memory_order_release);  // pairs with acquire load
}
```

## Concurrency Patterns

```cpp
#include <thread>
#include <future>
#include <vector>

// std::async: simple fan-out returning futures
auto fut = std::async(std::launch::async, [] { return heavy_computation(); });
int value = fut.get();  // blocks until ready, rethrows exceptions

// Fan-out/fan-in: launch multiple tasks, collect results
std::vector<std::future<int>> futures;
for (const auto& item : work_items) {
    futures.push_back(std::async(std::launch::async, process, item));
}
for (auto& f : futures) {
    results.push_back(f.get());
}

// promise/future: explicit result passing between threads
std::promise<std::string> p;
std::future<std::string> f = p.get_future();
std::thread t([&p] {
    try { p.set_value(compute()); }
    catch (...) { p.set_exception(std::current_exception()); }
});
auto result = f.get();  // blocks, rethrows if set_exception was called
t.join();

// jthread (C++20): auto-joining, cooperative cancellation
std::jthread worker([](std::stop_token stop) {
    while (!stop.stop_requested()) {
        process_next_item();
    }
});
// destructor requests stop and joins — no leaked threads
```

## Reliability Patterns

```cpp
#include <chrono>
#include <random>
#include <thread>

// Retry with exponential backoff and jitter
template <typename F>
auto retry_with_backoff(F&& func, int max_retries = 3) -> decltype(func()) {
    std::mt19937 rng(std::random_device{}());
    auto base = std::chrono::milliseconds(100);
    for (int attempt = 0; attempt <= max_retries; ++attempt) {
        try { return func(); }
        catch (const TransientError&) {
            if (attempt == max_retries) throw;
            auto delay = base * (1 << attempt);
            std::uniform_int_distribution<int> jitter(0, delay.count() / 2);
            std::this_thread::sleep_for(delay + std::chrono::milliseconds(jitter(rng)));
        }
    }
    throw std::logic_error("unreachable");
}
```

## Testing Concurrent Code

```cpp
#include <thread>
#include <vector>
#include <cassert>

void stress_test_counter() {
    ThreadSafeCounter counter;
    constexpr int num_threads = 50, increments = 10000;
    {
        std::vector<std::jthread> threads;
        for (int i = 0; i < num_threads; ++i)
            threads.emplace_back([&] { for (int j = 0; j < increments; ++j) counter.increment(); });
    }  // jthread destructors join here
    assert(counter.get() == num_threads * increments);
}

// Compile with ThreadSanitizer to detect data races:
//   g++ -fsanitize=thread -g -O1 my_test.cpp
//   clang++ -fsanitize=thread -g -O1 my_test.cpp
```
