# C++ Performance Reference

C++ profiling tools, benchmark framework (Google Benchmark), compiler-level optimization, and memory layout patterns.

---

## Profiling with perf

```bash
# CPU profile: where is time spent?
perf record -g ./my_program
perf report

# Cache miss analysis
perf stat -e cache-misses,cache-references,L1-dcache-load-misses ./my_program

# Branch misprediction
perf stat -e branch-misses,branches ./my_program

# Hardware counters: full picture
perf stat -e cycles,instructions,cache-misses,branch-misses ./my_program

# Generate flamegraph
perf record -g ./my_program
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

## Google Benchmark

```cpp
#include <benchmark/benchmark.h>

// Basic benchmark
static void BM_VectorPushBack(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<int> v;
        for (int i = 0; i < state.range(0); ++i) {
            v.push_back(i);
        }
    }
}
BENCHMARK(BM_VectorPushBack)->Range(8, 1 << 20);

// With setup/teardown outside measurement
static void BM_Sort(benchmark::State& state) {
    const auto n = state.range(0);
    for (auto _ : state) {
        state.PauseTiming();
        auto data = generate_random_vector(n);  // not measured
        state.ResumeTiming();

        std::sort(data.begin(), data.end());     // measured
    }
    state.SetItemsProcessed(state.iterations() * n);
}
BENCHMARK(BM_Sort)->Range(1 << 10, 1 << 20);

// Prevent compiler from eliminating dead code
static void BM_Computation(benchmark::State& state) {
    for (auto _ : state) {
        auto result = expensive_computation();
        benchmark::DoNotOptimize(result);  // prevents elimination
    }
}

// Prevent reordering of memory operations
static void BM_MemoryAccess(benchmark::State& state) {
    std::vector<int> data(1000000);
    for (auto _ : state) {
        for (auto& d : data) {
            d += 1;
        }
        benchmark::ClobberMemory();  // forces memory to be written
    }
}

// Compare two implementations
static void BM_MapLookup(benchmark::State& state) {
    std::map<int, int> m;
    for (int i = 0; i < 10000; ++i) m[i] = i;

    for (auto _ : state) {
        auto it = m.find(5000);
        benchmark::DoNotOptimize(it);
    }
}

static void BM_FlatArrayLookup(benchmark::State& state) {
    std::vector<int> v(10000);
    std::iota(v.begin(), v.end(), 0);

    for (auto _ : state) {
        // Binary search on sorted contiguous array
        auto it = std::lower_bound(v.begin(), v.end(), 5000);
        benchmark::DoNotOptimize(it);
    }
}

BENCHMARK(BM_MapLookup);
BENCHMARK(BM_FlatArrayLookup);

BENCHMARK_MAIN();
```

## Memory Layout

```cpp
// BAD: Array of Structures — wastes cache when iterating over positions
struct Particle {
    float x, y, z;          // 12 bytes — what we need
    float color_r, color_g, color_b, color_a;  // 16 bytes — don't need
    float normal_x, normal_y, normal_z;        // 12 bytes — don't need
    int material_id;                            // 4 bytes  — don't need
};
// 44 bytes per particle, but position update only needs 12
std::vector<Particle> particles;  // cache loads all 44 bytes per particle

// GOOD: Structure of Arrays — only touch what you need
struct ParticleSystem {
    std::vector<float> x, y, z;              // positions: contiguous
    std::vector<float> cr, cg, cb, ca;       // colors: separate
    std::vector<float> nx, ny, nz;           // normals: separate
    std::vector<int> material_id;
};
// Position update iterates over contiguous floats only


// Cache-line-aware padding to prevent false sharing
struct alignas(64) ThreadLocalCounter {
    std::atomic<uint64_t> count{0};
    // Padding ensures each counter is on its own cache line
    char padding[64 - sizeof(std::atomic<uint64_t>)];
};

std::array<ThreadLocalCounter, 8> per_thread_counters;
// Each thread increments its own counter without false sharing


// Small Buffer Optimization: avoid heap allocation for small data
// std::string already does this (typically 15-22 bytes inline)
// For custom types:
template <typename T, size_t InlineCapacity = 8>
class SmallVector {
    size_t size_ = 0;
    union {
        T inline_storage_[InlineCapacity];
        struct {
            T* data;
            size_t capacity;
        } heap_;
    };
    // Use inline storage for small sizes, heap for large
};
```

## Avoiding Allocation in Hot Paths

```cpp
// BAD: allocating in a loop
for (const auto& item : items) {
    auto result = std::make_unique<Result>(process(item));  // heap alloc per iteration
    results.push_back(std::move(result));
}

// GOOD: pre-allocate
results.reserve(items.size());
for (const auto& item : items) {
    results.emplace_back(process(item));  // no reallocation
}

// GOOD: reuse buffers across calls
class Processor {
    std::string buffer_;  // reused across process() calls
public:
    void process(std::string_view input) {
        buffer_.clear();  // no deallocation — capacity is retained
        buffer_.append(input);
        // ... use buffer_ ...
    }
};

// GOOD: stack allocation for bounded-size temporary data
void process_batch(std::span<const Item> items) {
    assert(items.size() <= 256);
    std::array<Result, 256> results;  // stack-allocated, no heap
    // ...
}
```

## Branch Prediction

```cpp
// Sort data before branching on it when feasible
// BAD: unpredictable branches on random data
int sum = 0;
for (int x : unsorted_data) {
    if (x >= threshold) sum += x;  // ~50% taken — unpredictable
}

// GOOD: sort first, branch becomes predictable
std::sort(data.begin(), data.end());
int sum = 0;
for (int x : data) {
    if (x >= threshold) sum += x;  // first N skip, rest take — predictable
}

// GOOD: branchless alternative for tight loops
int sum = 0;
for (int x : data) {
    sum += (x >= threshold) * x;  // no branch at all
}

// Use [[likely]] and [[unlikely]] hints (C++20)
if (result.has_value()) [[likely]] {
    process(*result);
} else [[unlikely]] {
    handle_error();
}
```

## Compiler Hints

```cpp
// Force inlining on hot functions
[[gnu::always_inline]] inline void hot_function() { ... }

// Prevent inlining on cold functions (keep hot code compact in icache)
[[gnu::noinline]] void cold_error_handler() { ... }

// Restrict aliasing for auto-vectorization
void add_arrays(float* __restrict__ out,
                const float* __restrict__ a,
                const float* __restrict__ b,
                size_t n) {
    for (size_t i = 0; i < n; ++i) {
        out[i] = a[i] + b[i];  // compiler can vectorize with SIMD
    }
}

// Prefetch for known access patterns
for (size_t i = 0; i < n; ++i) {
    __builtin_prefetch(&data[i + 16], 0, 1);  // prefetch 16 ahead
    process(data[i]);
}
```

## Concurrency Performance

```cpp
// Lock-free atomic for simple counters
std::atomic<uint64_t> counter{0};
counter.fetch_add(1, std::memory_order_relaxed);  // weakest sufficient ordering

// Avoid lock contention: partition work
void parallel_sum(std::span<const int> data, int num_threads) {
    std::vector<int64_t> partial(num_threads, 0);
    // Each thread works on its own partition — no synchronization needed
    auto chunk_size = data.size() / num_threads;
    std::vector<std::thread> threads;
    for (int t = 0; t < num_threads; ++t) {
        threads.emplace_back([&, t] {
            auto begin = t * chunk_size;
            auto end = (t == num_threads - 1) ? data.size() : begin + chunk_size;
            for (auto i = begin; i < end; ++i) {
                partial[t] += data[i];  // thread-local accumulation
            }
        });
    }
    for (auto& t : threads) t.join();
    auto total = std::accumulate(partial.begin(), partial.end(), int64_t{0});
}

// Read-copy-update for read-heavy shared state
// std::shared_mutex for read-heavy, write-rare patterns
std::shared_mutex rw_lock;

// Readers (concurrent):
{
    std::shared_lock lock{rw_lock};
    // ... read shared data ...
}

// Writer (exclusive):
{
    std::unique_lock lock{rw_lock};
    // ... modify shared data ...
}
```
