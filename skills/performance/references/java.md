# Java Performance Reference

Java profiling tools, JMH benchmarks, JIT warmup, GC tuning, and optimization patterns. Java's performance model is shaped by the JIT compiler and garbage collector — understanding their behavior is essential for reliable optimization.

---

## Profiling

```bash
# async-profiler: low-overhead sampling profiler
# Produces flamegraphs, supports CPU, allocation, and lock profiling
# https://github.com/async-profiler/async-profiler

# CPU profiling (attach to running JVM)
./asprof -d 30 -f flame.html <pid>

# Allocation profiling (find where objects are created)
./asprof -d 30 -e alloc -f alloc_flame.html <pid>

# Lock contention profiling
./asprof -d 30 -e lock -f lock_flame.html <pid>

# From within the JVM (programmatic)
# AsyncProfiler.getInstance().start(Events.CPU, 1_000_000);
# AsyncProfiler.getInstance().stop();
# AsyncProfiler.getInstance().dumpFlat(50);


# JFR (Java Flight Recorder) — built into the JDK, zero-config
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr MyApp
# Analyze with JDK Mission Control (jmc)

# JFR from command line on running process
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# GC logging (essential for understanding GC pauses)
java -Xlog:gc*:file=gc.log:time,uptime,level,tags MyApp
# Analyze with GCEasy (gceasy.io) or GCViewer
```

## JMH Benchmarks

```java
// In build.gradle:
// dependencies {
//     testImplementation 'org.openjdk.jmh:jmh-core:1.37'
//     testAnnotationProcessor 'org.openjdk.jmh:jmh-generator-annprocess:1.37'
// }

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)      // JIT warmup: critical for Java
@Measurement(iterations = 10, time = 1)
@Fork(2)                                // separate JVM forks for isolation
@State(Scope.Benchmark)
public class SortBenchmark {

    @Param({"100", "10000", "1000000"})
    private int size;

    private int[] data;

    @Setup(Level.Invocation)  // fresh data for each invocation
    public void setup() {
        data = ThreadLocalRandom.current().ints(size).toArray();
    }

    @Benchmark
    public void arraySort(Blackhole bh) {
        Arrays.sort(data);
        bh.consume(data);  // prevent dead code elimination
    }
}

// Comparing implementations
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
public class LookupBenchmark {

    private Map<Integer, String> hashMap;
    private int[] sortedArray;
    private int target;

    @Setup
    public void setup() {
        hashMap = new HashMap<>();
        for (int i = 0; i < 10_000; i++) {
            hashMap.put(i, "val" + i);
        }
        sortedArray = IntStream.range(0, 10_000).toArray();
        target = 5000;
    }

    @Benchmark
    public String hashMapLookup() {
        return hashMap.get(target);
    }

    @Benchmark
    public int binarySearchLookup() {
        return Arrays.binarySearch(sortedArray, target);
    }
}

// Run from command line:
// java -jar benchmarks.jar -rf json -rff results.json

// CRITICAL: Never benchmark in a main() method without JMH.
// The JIT compiler behaves differently in microbenchmarks vs real workloads.
// JMH handles: warmup, deoptimization guards, dead code elimination,
// fork isolation, and constant folding prevention.
```

## JIT Warmup and Tiered Compilation

```java
// The JIT compiler optimizes code based on runtime profiling.
// Code that runs thousands of times gets compiled and optimized.
// First runs are interpreted (slow), later runs are compiled (fast).

// JMH handles this with @Warmup iterations, but in production:
// - Expect slow startup / first requests
// - Don't measure performance during warmup
// - For latency-critical services, consider warming up endpoints
//   with synthetic traffic before accepting real traffic

// JIT compilation thresholds:
// -XX:CompileThreshold=10000 (default for server, method invocations before compile)
// -XX:+PrintCompilation       (log what the JIT compiles)
// -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining (see inlining decisions)

// Inlining: keep hot methods small (<325 bytes bytecode default)
// -XX:MaxInlineSize=35        (default for cold methods)
// -XX:FreqInlineSize=325      (default for hot methods)
// Methods too large to inline miss significant optimization opportunities
```

## Avoiding Allocation on Hot Paths

```java
// Object allocation is cheap in Java (~10ns), but GC pressure adds up.
// On hot paths, reducing allocation reduces GC pauses.

// Pre-size collections
var results = new ArrayList<Result>(items.size());  // single backing array
for (var item : items) {
    results.add(process(item));
}

// Reuse StringBuilder across iterations
final var sb = new StringBuilder(256);
for (var item : items) {
    sb.setLength(0);  // reuse buffer, no reallocation
    sb.append("processed: ").append(item.name());
    emit(sb.toString());
}

// Avoid autoboxing in hot loops
// BAD:
Map<Integer, Integer> map = new HashMap<>();  // Integer autoboxing
for (int i = 0; i < n; i++) {
    map.put(i, i * 2);  // boxes both ints → 2 allocations per iteration
}

// GOOD: use primitive-specialized collections (Eclipse Collections, HPPC)
// IntIntHashMap map = new IntIntHashMap();
// map.put(i, i * 2);  // no boxing

// Avoid String concatenation in loops
// BAD: creates new String each iteration
String result = "";
for (var word : words) {
    result += word + " ";  // O(n²) total, allocation each iteration
}

// GOOD:
String result = String.join(" ", words);  // single pass


// Use records and value-based classes for small immutable data
// JVM may optimize these to avoid heap allocation in the future (Valhalla)
record Point(double x, double y) {}
```

## Data Structure Choices

```java
// ArrayList vs LinkedList: almost always ArrayList
// ArrayList: contiguous memory, cache-friendly iteration, O(1) random access
// LinkedList: pointer chasing, cache-hostile, higher per-element overhead
// Use LinkedList only for: frequent insertion/removal at known positions via iterator

// HashMap: default choice for key-value lookup
// Load factor 0.75 is good for most cases
// Pre-size to avoid rehashing: new HashMap<>(expectedSize * 4 / 3 + 1)

// EnumMap / EnumSet: when keys are enum values
// Backed by arrays, extremely fast, zero boxing
var statusCounts = new EnumMap<Status, Integer>(Status.class);

// Arrays for primitive-heavy, fixed-size work
// int[] is far more cache-friendly than List<Integer>
int[] values = new int[n];  // contiguous, no boxing, no object headers
```

## Concurrency Performance

```java
// Virtual threads (Java 21+): for I/O-bound concurrency
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var futures = items.stream()
        .map(item -> executor.submit(() -> fetchFromNetwork(item)))
        .toList();
    var results = futures.stream()
        .map(f -> f.get())
        .toList();
}
// Millions of virtual threads are fine — they're cheap


// Parallel streams for CPU-bound data parallelism
long total = items.parallelStream()
    .filter(Item::isActive)
    .mapToLong(Item::value)
    .sum();
// Only for CPU-bound work on large datasets — overhead is real for small inputs


// ConcurrentHashMap for shared state
var cache = new ConcurrentHashMap<Key, Value>();
cache.computeIfAbsent(key, k -> expensiveCompute(k));  // atomic, no external lock

// Striped counters for high-contention counting
// java.util.concurrent.atomic.LongAdder is better than AtomicLong under contention
var counter = new LongAdder();
counter.increment();         // thread-local accumulation
long total = counter.sum();  // merge when needed


// Avoid synchronized on hot paths — use lock-free structures
// ConcurrentLinkedQueue, ConcurrentSkipListMap, etc.
// If you must lock, minimize the critical section:

// BAD: lock held during I/O
synchronized(lock) {
    var data = readFromDatabase();  // holds lock during I/O
    cache.put(key, data);
}

// GOOD: lock only protects shared state
var data = readFromDatabase();   // no lock during I/O
synchronized(lock) {
    cache.put(key, data);        // minimal critical section
}
```

## GC Tuning Basics

```bash
# See which GC is active
java -XX:+PrintFlagsFinal -version | grep UseG1

# G1GC (default since Java 9): good general-purpose collector
java -XX:+UseG1GC -XX:MaxGCPauseMillis=200 MyApp

# ZGC (Java 17+): ultra-low latency (<1ms pauses), higher throughput cost
java -XX:+UseZGC MyApp

# Shenandoah: similar to ZGC, available in OpenJDK
java -XX:+UseShenandoahGC MyApp

# Heap sizing: don't over-allocate (wastes memory, longer GC cycles)
java -Xms4g -Xmx4g MyApp  # fixed heap avoids resize pauses

# Key principle: reduce allocation rate on hot paths first,
# then tune GC parameters. Tuning GC without reducing allocation
# is treating symptoms, not causes.
```

## JMH Pitfalls to Avoid

```java
// PITFALL 1: Constant folding — JIT precomputes constant expressions
// BAD:
@Benchmark
public int badBench() {
    return 42 * 17;  // JIT computes this at compile time: measures nothing
}
// GOOD: use @State fields that JMH prevents from being constant-folded

// PITFALL 2: Dead code elimination — JIT removes unused results
// BAD:
@Benchmark
public void badBench() {
    compute();  // result discarded — JIT may eliminate the call
}
// GOOD:
@Benchmark
public int goodBench() {
    return compute();  // JMH captures the result
}
// OR:
@Benchmark
public void goodBench(Blackhole bh) {
    bh.consume(compute());
}

// PITFALL 3: Loop optimization — JIT may optimize loop differently
// BAD:
@Benchmark
public long badBench() {
    long sum = 0;
    for (int i = 0; i < 1000; i++) {
        sum += i;  // JIT may replace with closed-form formula
    }
    return sum;
}
// Let JMH control the iteration — don't write your own measurement loops

// PITFALL 4: Measuring in main() without JMH
// The JVM optimizes differently in single-method benchmarks vs real apps.
// Always use JMH. Always.
```
