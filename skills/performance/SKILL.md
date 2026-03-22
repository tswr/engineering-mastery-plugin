---
name: performance
description: "Apply when optimizing for speed, throughput, or memory efficiency, writing benchmarks, or profiling applications. Trigger on mentions of benchmarking, profiling, cache misses, memory layout, hot paths, latency, throughput, SIMD, branch prediction, or 'why is this slow.' Language-specific tools and idioms are in references/."
---

# Performance Optimization Skill

Optimize what matters, measure before you change, and prove every optimization with evidence. The most common performance mistake is not writing slow code — it's optimizing the wrong code, or optimizing without measurement, or measuring incorrectly.

**Language-specific tools and idioms are in the `references/` directory.** Read the relevant file when working in a specific language:
- `references/cpp.md` — Google Benchmark, perf, compiler intrinsics, memory layout
- `references/python.md` — cProfile, timeit, memory_profiler, NumPy vectorization
- `references/rust.md` — criterion, flamegraph, #[inline], data layout
- `references/java.md` — JMH, async-profiler, JIT warmup, GC tuning

---

## 1. Measure First, Then Optimize

Never optimize based on intuition. Developers are consistently bad at guessing where time is spent. The code you think is slow is usually not the bottleneck. The actual bottleneck is usually somewhere you didn't expect.

**The workflow:**
1. **Define the goal.** What metric matters? Latency (p50, p99, p999)? Throughput (requests/sec)? Memory footprint? Startup time? A vague goal ("make it faster") leads to wasted effort.
2. **Measure the baseline.** Profile the system under realistic conditions. Record the numbers. This is your reference point.
3. **Identify the bottleneck.** Use profiling to find where time is actually spent. Don't assume — look at the data. The bottleneck is the single constraint that limits overall performance; improving anything else is waste.
4. **Form a hypothesis.** "Replacing this map lookup with a flat array will reduce cache misses in this hot loop." Be specific.
5. **Make one change.** Change one thing at a time. Multiple simultaneous changes make it impossible to attribute improvement.
6. **Measure again.** Compare against the baseline under identical conditions. Did it actually improve? By how much? Did it regress anything else?
7. **Keep or revert.** If the improvement is real and meaningful, keep it. If it's within noise or introduces complexity without payoff, revert.

**The cardinal sin:** optimizing without a profile. Every minute spent optimizing code that isn't on the hot path is a minute wasted — and worse, it often makes the code harder to read and maintain for zero performance gain.

---

## 2. Benchmarking Correctly

A benchmark that produces wrong numbers is worse than no benchmark, because it gives false confidence. Getting benchmarking right requires controlling for multiple sources of error.

### Warmup

Cold code behaves differently from warm code. CPU caches are empty, branch predictors are untrained, JIT compilers haven't kicked in, memory pages aren't mapped. Always include a warmup phase that exercises the code path before you start measuring. Discard warmup iterations from results.

### Measure Distributions, Not Averages

A single average hides the shape of the data. Measure enough iterations to capture the distribution, then report percentiles: p50 (median), p90, p99, and p99.9 for latency-sensitive work. A function with 1ms average and 500ms p99 has a very different character from one with 5ms average and 6ms p99.

### Control for Variance

**Sources of noise:** background processes, CPU frequency scaling, thermal throttling, garbage collection pauses, OS scheduling, other tenants on shared hardware, memory allocation patterns that vary across runs.

**Mitigations:** pin to a CPU core when possible, disable frequency scaling for benchmarks, close competing workloads, run enough iterations that statistical noise is small relative to the signal, report standard deviation alongside the mean.

**The significance test:** if an optimization shows a 3% improvement but your measurement variance is 5%, you haven't proven anything. The change must exceed the noise floor.

### Prevent Compiler Elimination

Optimizing compilers will remove code whose results are never observed. If your benchmark computes a value and never uses it, the compiler may delete the entire computation. Use your language's benchmark framework's mechanism for preventing this (benchmark::DoNotOptimize, std::hint::black_box, JMH's Blackhole, etc.). Never use `volatile` or printing as a substitute — they introduce their own overhead and distort measurements.

### Benchmark Representative Workloads

A microbenchmark on a 10-element array tells you nothing about behavior on a 10-million-element dataset. Cache effects, memory allocation patterns, branch predictor behavior, and algorithmic complexity all change with scale. Benchmark at the scale and data distribution you'll see in production.

---

## 3. The USE Method

For every resource in the system (CPU, memory, disk, network, locks), check three things:

**Utilization** — what fraction of the resource's capacity is being consumed? A CPU at 95% utilization has little headroom. Memory at 80% of physical capacity may be triggering page reclamation.

**Saturation** — is work queuing because the resource is fully utilized? CPU run queue depth, disk I/O queue depth, thread pool queue depth. Saturation means latency is growing even if throughput is steady.

**Errors** — is the resource producing errors that force retries or fallbacks? Network packet drops, disk I/O errors, OOM kills, timeout failures.

**Why this matters:** the USE method systematically identifies the bottleneck instead of guessing. Walk through every resource, check all three metrics, and the bottleneck reveals itself. This is far more productive than staring at code and speculating.

---

## 4. The Memory Hierarchy

In modern systems, memory access patterns dominate performance more than instruction count for most workloads. The speed gap between CPU registers and main memory is roughly 200x. The cache hierarchy exists to bridge this gap, but only if your access patterns cooperate.

**Approximate latencies (order of magnitude):**
- L1 cache: ~1 ns (4 cycles)
- L2 cache: ~4 ns (12 cycles)
- L3 cache: ~12 ns (40 cycles)
- Main memory (DRAM): ~60-100 ns (200+ cycles)
- SSD random read: ~10-100 μs
- Network round-trip (same datacenter): ~500 μs
- HDD random read: ~5-10 ms

### Spatial Locality

Access memory in contiguous, sequential patterns. When you access one byte, the CPU fetches an entire cache line (typically 64 bytes). If your next access is within that same cache line, it's essentially free. If it's in a random location, you pay the full memory latency.

**Practical consequence:** arrays and flat structures that you iterate linearly are cache-friendly. Linked lists, trees with heap-allocated nodes, and hash maps with pointer chasing are cache-hostile. A linear scan of a contiguous array often beats a theoretically superior O(log n) tree lookup for small-to-medium datasets because the constant factor of cache misses dwarfs the algorithmic advantage.

### Temporal Locality

Access the same data again while it's still in cache. If you process a data item, do all the work on it before moving on. Don't make one pass to read, a second pass to transform, and a third pass to write — each pass evicts the previous pass's cache contents.

### Data-Oriented Design

Organize data for how it's accessed, not for how it's conceptually related. If you iterate over a million objects but only touch two of their ten fields, those two fields should be contiguous in memory (Structure of Arrays), not interleaved with eight unused fields (Array of Structures).

**Array of Structures (AoS):** `[{x,y,z,color,normal,...}, {x,y,z,color,normal,...}, ...]`
Cache loads all fields even if you only need x,y,z. Wastes bandwidth.

**Structure of Arrays (SoA):** `{xs: [...], ys: [...], zs: [...], colors: [...], ...}`
Iterating over just positions touches only contiguous position data. Cache-efficient.

This doesn't mean always use SoA — it means understand your access patterns and lay out data accordingly.

---

## 5. Algorithmic Optimization

The highest-leverage optimization is almost always algorithmic. No amount of cache tuning makes an O(n²) algorithm competitive with an O(n log n) one at scale.

**Check algorithmic complexity first.** Before profiling cache behavior or instruction counts, verify that the algorithm is appropriate for the data size. An O(n²) loop hiding inside an O(n) outer loop creates O(n³) that may not be obvious from code review.

**Watch for hidden quadratics.** String concatenation in a loop (each concat copies the full string), repeated linear searches in a list, nested loops where the inner collection grows with the outer. These are the most common performance bugs in production code.

**Right data structure for the access pattern.** Don't just pick the "fastest" data structure — pick the one that matches how you access the data. A hash map has O(1) lookup but terrible cache behavior for iteration. A sorted array has O(log n) lookup with binary search but excellent cache behavior for scans. The right choice depends on the ratio of lookups to iterations in your actual workload.

**Amortize expensive operations.** If an operation is expensive, do it less often. Batch writes instead of writing one at a time. Precompute results that are used repeatedly. Cache derived values that are expensive to recompute (but invalidate them correctly — stale caches are bugs).

---

## 6. Concurrency and Parallelism

**Amdahl's Law.** The maximum speedup from parallelization is limited by the fraction of work that must remain sequential. If 10% of the workload is inherently serial, the maximum speedup from infinite parallelism is 10x. Before parallelizing, estimate the serial fraction — if it's large, parallelize the serial part first or redesign the algorithm.

**Contention is the enemy.** Shared mutable state that requires synchronization (locks, atomics) becomes a bottleneck under concurrency. The more threads contend for the same lock, the worse the scaling. Solutions: reduce the critical section size, use lock-free structures where appropriate, partition data so threads operate on independent subsets, or eliminate sharing entirely.

**False sharing.** When two threads write to different variables that happen to reside on the same cache line, the CPU invalidates the entire line on every write, forcing expensive cache coherence traffic. This can make concurrent code slower than single-threaded. Fix by padding or aligning data so each thread's working set occupies its own cache line(s).

**Batch over coordinate.** When possible, divide work into independent batches that require no synchronization during execution and a single merge at the end. This is more scalable than fine-grained locking, easier to reason about, and avoids contention entirely during the work phase.

---

## 7. Common Optimization Patterns

**Hot path / cold path separation.** Identify the code path executed most frequently and optimize it aggressively. Move error handling, logging, and rare-case logic to separate functions so the hot path stays compact and stays in the instruction cache.

**Avoid unnecessary allocation.** Memory allocation and deallocation are expensive (especially under concurrency when the allocator's internal locks are contended). Reuse buffers, pre-allocate collections at the expected size, use object pools for high-frequency short-lived objects, and prefer stack allocation over heap allocation when the lifetime is limited to the scope.

**Branch prediction.** Modern CPUs predict which way branches will go and speculatively execute ahead. When prediction fails, the pipeline stalls. Organize code so the common case falls through (don't make the common case take the branch), sort data before branching on it when feasible, and consider branchless alternatives for tight inner loops where the branch is unpredictable.

**Lazy computation.** Don't compute what you don't need. If only 10% of results are ever accessed, compute on demand rather than eagerly. But beware the inverse: if you know you'll need a result, computing it eagerly may be cheaper than the overhead of lazy evaluation machinery.

**Batching I/O.** Individual I/O operations have high fixed costs (system call overhead, network round-trip, disk seek). Batch multiple operations into a single call: bulk reads, write-behind buffers, scatter-gather I/O, pipelined requests. The difference between 1000 individual writes and 1 batch write of 1000 items can be orders of magnitude.

---

## 8. Database Performance

Database queries are the most common performance bottleneck in application code. A single slow query on a hot path can dominate overall latency.

**Understand query plans.** Use `EXPLAIN` (or your database's equivalent) to see how the database executes your query. Look for sequential scans on large tables, nested loop joins on unindexed columns, and sort operations that spill to disk. The query plan is the ground truth — don't guess at query performance.

**Index design.** Indexes accelerate reads at the cost of slower writes and more storage. Index columns that appear in WHERE, JOIN, and ORDER BY clauses on frequently executed queries. Composite indexes must match the query's column order (leftmost prefix rule). Don't blindly index everything — unused indexes are pure overhead.

**N+1 queries.** The most common database performance anti-pattern: fetching a list of N items, then issuing one query per item to fetch related data. Result: N+1 round-trips instead of 1-2. Fix with JOINs, batch loading, or ORM eager-loading. Detect by logging query counts per request during development.

**Connection management.** Database connections are expensive to establish. Use connection pools sized to match your concurrency level. Too few connections → requests queue; too many → database overloaded. Monitor pool utilization as a key metric.

---

## 9. When Not to Optimize

**Premature optimization.** If the code isn't on a hot path, isn't part of a latency-critical request, and isn't consuming disproportionate resources, don't optimize it. Clear, maintainable code that's "fast enough" is better than clever, fragile code that's 20% faster on a path executed once per hour.

**Optimization that harms readability.** Every optimization has a maintenance cost. If the optimization makes the code significantly harder to understand, it must come with a proportionally significant performance improvement. A 2% improvement that triples the code complexity is almost never worth it.

**Optimization without evidence.** If you can't show a profile that identifies the bottleneck and a benchmark that proves the improvement, the optimization is speculative. Speculative optimizations accumulate as complexity debt.

**The threshold:** optimize when profiling shows a specific bottleneck, the benchmark proves the improvement exceeds noise, and the performance gain justifies the complexity cost.

---

## Applying This Skill

When optimizing:
1. Define the performance goal (metric + target)
2. Profile under realistic conditions to find the bottleneck
3. Apply the USE method to identify resource constraints
4. Check algorithmic complexity before micro-optimizing
5. Consider memory layout and cache behavior for data-intensive code
6. Make one change, benchmark, keep or revert
7. Document why the optimization exists (future readers will wonder)

When writing benchmarks:
1. Include warmup, discard warmup data
2. Measure enough iterations for statistical significance
3. Report percentiles (p50, p90, p99), not just averages
4. Prevent compiler elimination of dead code
5. Benchmark at representative scale and data distribution
6. Control for system noise — report variance

When reviewing performance-sensitive code:
- Is there a profile showing this is actually hot?
- Does the optimization have a benchmark proving it helps?
- Is the algorithmic complexity appropriate for the data size?
- Are memory access patterns cache-friendly?
- Is the complexity cost justified by the measured improvement?

When the target language is known, read the corresponding file in `references/` for language-specific tools and idioms.
