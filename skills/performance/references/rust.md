# Rust Performance Reference

Rust profiling tools, criterion benchmarks, and optimization patterns. Rust's zero-cost abstractions and ownership model give you C-level performance with safety, but you still need to understand the hardware to get the most out of it.

---

## Profiling

```bash
# perf (Linux): CPU profiling
perf record -g --call-graph dwarf ./target/release/my_program
perf report

# Flamegraph (via cargo-flamegraph)
# cargo install flamegraph
cargo flamegraph --release
# Outputs flamegraph.svg

# For specific benchmarks:
cargo flamegraph --bench my_benchmark -- --bench

# valgrind/callgrind: instruction-level profiling
# cargo install cargo-valgrind
valgrind --tool=callgrind ./target/release/my_program
kcachegrind callgrind.out.*

# cachegrind: cache miss analysis
valgrind --tool=cachegrind ./target/release/my_program

# DHAT: heap profiling (allocation patterns)
valgrind --tool=dhat ./target/release/my_program
```

## Criterion Benchmarks

```rust
// In Cargo.toml:
// [dev-dependencies]
// criterion = { version = "0.5", features = ["html_reports"] }
//
// [[bench]]
// name = "my_benchmark"
// harness = false

use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};

// Basic benchmark
fn bench_sort(c: &mut Criterion) {
    let mut data: Vec<i32> = (0..10_000).rev().collect();

    c.bench_function("sort_10k", |b| {
        b.iter_batched(
            || data.clone(),               // setup: fresh unsorted data each iteration
            |mut d| d.sort(),              // measured: only the sort
            criterion::BatchSize::SmallInput,
        );
    });
}

// Prevent compiler elimination
fn bench_computation(c: &mut Criterion) {
    c.bench_function("fibonacci", |b| {
        b.iter(|| {
            black_box(fibonacci(black_box(20)))  // prevent dead code elimination
        });
    });
}

// Parameterized benchmarks: compare across sizes
fn bench_lookup(c: &mut Criterion) {
    let mut group = c.benchmark_group("lookup");

    for size in [100, 1_000, 10_000, 100_000] {
        let data: Vec<i32> = (0..size).collect();
        let target = size / 2;

        group.bench_with_input(
            BenchmarkId::new("binary_search", size),
            &size,
            |b, _| {
                b.iter(|| data.binary_search(&target));
            },
        );

        let set: std::collections::HashSet<i32> = data.iter().copied().collect();
        group.bench_with_input(
            BenchmarkId::new("hashset_contains", size),
            &size,
            |b, _| {
                b.iter(|| set.contains(&target));
            },
        );
    }
    group.finish();
}

// Compare two implementations
fn bench_compare(c: &mut Criterion) {
    let mut group = c.benchmark_group("string_concat");
    let words: Vec<String> = (0..1000).map(|i| format!("word{i}")).collect();

    group.bench_function("push_string", |b| {
        b.iter(|| {
            let mut s = String::new();
            for w in &words {
                s.push_str(w);
                s.push(' ');
            }
            black_box(s);
        });
    });

    group.bench_function("join", |b| {
        b.iter(|| {
            black_box(words.join(" "));
        });
    });

    group.finish();
}

criterion_group!(benches, bench_sort, bench_computation, bench_lookup, bench_compare);
criterion_main!(benches);
```

## Memory Layout

```rust
// Check struct sizes and alignment
use std::mem;

println!("Size: {}", mem::size_of::<MyStruct>());
println!("Align: {}", mem::align_of::<MyStruct>());

// Field ordering matters — Rust may reorder fields for optimal packing
// Use #[repr(C)] to force C layout when needed (e.g., FFI, cache line control)

// BAD: Array of Structures for iteration over subset of fields
struct Particle {
    position: [f32; 3],    // 12 bytes — what we need for physics
    color: [f32; 4],       // 16 bytes — unused in physics step
    normal: [f32; 3],      // 12 bytes — unused in physics step
    material_id: u32,      // 4 bytes  — unused in physics step
}
// 44 bytes per particle loaded into cache, only 12 used

// GOOD: Structure of Arrays
struct ParticleSystem {
    positions: Vec<[f32; 3]>,  // contiguous, cache-friendly iteration
    colors: Vec<[f32; 4]>,     // separate, loaded only when needed
    normals: Vec<[f32; 3]>,
    material_ids: Vec<u32>,
}


// Avoid enum bloat: large enums waste memory on small variants
// BAD:
enum Message {
    Tiny(u8),                  // needs 1 byte, but enum is sized to largest
    Huge([u8; 1024]),          // forces all Messages to be 1025+ bytes
}

// GOOD: box the large variant
enum Message {
    Tiny(u8),
    Huge(Box<[u8; 1024]>),    // only 8 bytes (pointer) in the enum
}


// Cache-line padding to prevent false sharing
#[repr(align(64))]
struct CacheAligned<T>(T);

let counters: Vec<CacheAligned<AtomicU64>> = (0..num_threads)
    .map(|_| CacheAligned(AtomicU64::new(0)))
    .collect();
```

## Avoiding Allocation

```rust
// Pre-allocate collections
let mut results = Vec::with_capacity(items.len());  // single allocation
for item in &items {
    results.push(process(item));
}

// Reuse buffers with clear()
let mut buffer = String::new();
for input in inputs {
    buffer.clear();  // reuses allocation
    write!(buffer, "processed: {}", input).unwrap();
    send(&buffer);
}

// SmallVec for collections that are usually small
// In Cargo.toml: smallvec = "1"
use smallvec::SmallVec;
let mut tags: SmallVec<[Tag; 4]> = SmallVec::new();
// Stack-allocated up to 4 elements, heap-allocated beyond that

// Cow for avoiding unnecessary clones
use std::borrow::Cow;

fn process(input: &str) -> Cow<str> {
    if input.contains("bad") {
        Cow::Owned(input.replace("bad", "good"))  // allocates only when needed
    } else {
        Cow::Borrowed(input)  // zero-cost: no allocation
    }
}

// Arena allocation for many short-lived objects
// In Cargo.toml: bumpalo = "3"
use bumpalo::Bump;
let arena = Bump::new();
let node1 = arena.alloc(TreeNode::new(1));
let node2 = arena.alloc(TreeNode::new(2));
// All nodes freed at once when arena is dropped — no per-node dealloc
```

## Iterator Chains and Zero-Cost Abstractions

```rust
// Rust iterators compile to the same code as manual loops
// This is NOT slower than a hand-written loop:
let total: i64 = items.iter()
    .filter(|item| item.is_active())
    .map(|item| item.value())
    .sum();

// Collect into specific types to control allocation
let results: Vec<_> = data.iter()
    .filter_map(|x| process(x).ok())
    .collect();  // single allocation, correct capacity

// Parallel iteration with rayon
// In Cargo.toml: rayon = "1"
use rayon::prelude::*;

let total: i64 = items.par_iter()  // parallel version — same API
    .filter(|item| item.is_active())
    .map(|item| item.value())
    .sum();
```

## Compiler Hints

```rust
// Inline hot functions
#[inline(always)]
fn hot_inner_loop_helper(x: f64) -> f64 {
    x * x + 2.0 * x + 1.0
}

// Don't inline cold functions (keep hot code compact in icache)
#[cold]
#[inline(never)]
fn handle_error(err: &Error) {
    eprintln!("Error: {err}");
}

// Likely/unlikely hints (nightly, or use the likely_stable crate)
if likely(result.is_ok()) {
    process(result.unwrap());
} else {
    handle_error();
}

// unsafe for performance-critical paths (with justification)
// Use get_unchecked only when bounds are proven and profiling shows impact
unsafe {
    let val = data.get_unchecked(index);  // skips bounds check
}
// ALWAYS: document why this is safe, benchmark to prove it matters
```

## Concurrency Performance

```rust
// Rayon for data parallelism (easiest, usually fastest)
use rayon::prelude::*;
let results: Vec<_> = inputs.par_iter().map(|x| process(x)).collect();

// Channels for pipeline parallelism
use std::sync::mpsc;
let (tx, rx) = mpsc::channel();
std::thread::spawn(move || {
    for item in produce_items() {
        tx.send(item).unwrap();
    }
});
for item in rx {
    consume(item);
}

// Atomic operations: use weakest sufficient ordering
use std::sync::atomic::{AtomicU64, Ordering};
counter.fetch_add(1, Ordering::Relaxed);     // counter: no ordering needed
flag.store(true, Ordering::Release);          // publish: release
if flag.load(Ordering::Acquire) { ... }       // consume: acquire

// RwLock for read-heavy shared state
use std::sync::RwLock;
let data = RwLock::new(shared_state);
// Readers (concurrent):
let guard = data.read().unwrap();
// Writer (exclusive):
let mut guard = data.write().unwrap();
```
