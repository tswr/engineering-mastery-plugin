# Java Concurrency Reference

Idiomatic Java concurrent patterns using java.util.concurrent (JUC). Java 21 introduces virtual threads (Project Loom) for lightweight thread-per-request concurrency without async/await complexity.

---

## Shared Mutable State

```java
// Prefer confinement: ThreadLocal gives each thread its own copy
private static final ThreadLocal<SimpleDateFormat> DATE_FMT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

// Immutable records are inherently thread-safe — no synchronization needed
public record PriceUpdate(String symbol, BigDecimal price, Instant timestamp) {}

// ConcurrentHashMap: thread-safe without locking the entire map
private final ConcurrentHashMap<String, AtomicLong> counters = new ConcurrentHashMap<>();

public void increment(String key) {
    counters.computeIfAbsent(key, k -> new AtomicLong()).incrementAndGet();
}

// When sharing mutable state, synchronize ALL access (reads and writes)
public class ThreadSafeCounter {
    private long count = 0;
    public synchronized void increment() { count++; }
    public synchronized long get() { return count; }
}
```

## Synchronization Primitives

```java
import java.util.concurrent.locks.*;

// ReentrantLock: always unlock in finally
private final ReentrantLock lock = new ReentrantLock();

public void update() {
    lock.lock();
    try { /* critical section */ }
    finally { lock.unlock(); }
}

// ReadWriteLock: concurrent readers, exclusive writers
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

public String read(String key) {
    rwLock.readLock().lock();
    try { return cache.get(key); }
    finally { rwLock.readLock().unlock(); }
}
public void write(String key, String value) {
    rwLock.writeLock().lock();
    try { cache.put(key, value); }
    finally { rwLock.writeLock().unlock(); }
}

// Deadlock prevention: consistent lock ordering by comparable key
public void transfer(Account a, Account b, long amount) {
    Account first = a.id() < b.id() ? a : b;
    Account second = a.id() < b.id() ? b : a;
    synchronized (first) { synchronized (second) {
        first.debit(amount); second.credit(amount);
    }}
}

// AtomicReference: lock-free single-value updates
private final AtomicReference<Config> config = new AtomicReference<>(Config.defaults());
public void updateConfig(UnaryOperator<Config> updater) {
    config.updateAndGet(updater);  // compare-and-swap loop internally
}
```

## Concurrency Patterns

```java
import java.util.concurrent.*;

// ExecutorService: fixed thread pool for task execution
ExecutorService exec = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
Future<String> future = exec.submit(() -> fetchData(url));
String result = future.get(10, TimeUnit.SECONDS);  // blocks with timeout

// Always shut down executors
exec.shutdown();
if (!exec.awaitTermination(60, TimeUnit.SECONDS)) exec.shutdownNow();

// BlockingQueue: producer-consumer with backpressure
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);
queue.put(task);                                  // blocks when full
Task t = queue.poll(5, TimeUnit.SECONDS);         // blocks with timeout

// CompletableFuture: composable async pipelines
CompletableFuture<Order> pipeline = CompletableFuture
    .supplyAsync(() -> fetchUser(userId), exec)
    .thenCompose(user -> fetchOrders(user.id()))
    .thenApply(orders -> orders.get(0))
    .orTimeout(10, TimeUnit.SECONDS)
    .exceptionally(ex -> { log.warn("failed", ex); return fallback(); });

// Combine independent futures
CompletableFuture.allOf(fetchUser(1), fetchOrders(1), fetchInventory()).join();
```

## Async and Non-Blocking I/O

```java
// Virtual threads (Java 21+): lightweight, JVM-managed
try (var exec = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = urls.stream()
        .map(url -> exec.submit(() -> fetch(url)))  // one virtual thread per task
        .toList();
    for (var f : futures) { process(f.get(30, TimeUnit.SECONDS)); }
}

// Structured concurrency (Java 21 preview): scoped task lifetimes
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<User> userTask = scope.fork(() -> fetchUser(id));
    Subtask<Orders> ordersTask = scope.fork(() -> fetchOrders(id));
    scope.join();            // wait for both
    scope.throwIfFailed();   // propagate first failure
    return new UserProfile(userTask.get(), ordersTask.get());
}  // if one fails, the other is cancelled automatically
```

## Reliability Patterns

```java
import java.time.Duration;
import java.util.concurrent.ThreadLocalRandom;

public <T> T retryWithBackoff(Callable<T> op, int maxRetries) throws Exception {
    var baseMs = 100L;
    for (int attempt = 0; attempt <= maxRetries; attempt++) {
        try { return op.call(); }
        catch (IOException | TimeoutException e) {
            if (attempt == maxRetries) throw e;
            long delay = Math.min(baseMs * (1L << attempt), 30_000);
            Thread.sleep(delay + ThreadLocalRandom.current().nextLong(delay / 2));
        }
    }
    throw new AssertionError("unreachable");
}

// Always set timeouts on HTTP clients
HttpClient client = HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(5)).build();
HttpResponse<String> resp = client.send(
    HttpRequest.newBuilder().uri(URI.create("https://api.example.com/data"))
        .timeout(Duration.ofSeconds(10)).build(),
    BodyHandlers.ofString());
```

## Testing Concurrent Code

```java
import org.junit.jupiter.api.Test;
import java.util.concurrent.*;
import static org.junit.jupiter.api.Assertions.*;

@Test
void stressTestConcurrentIncrements() throws Exception {
    var counter = new ThreadSafeCounter();
    int threads = 50, perThread = 10_000;
    var latch = new CountDownLatch(1);  // all threads start together
    try (var exec = Executors.newFixedThreadPool(threads)) {
        var futures = new ArrayList<Future<?>>();
        for (int i = 0; i < threads; i++)
            futures.add(exec.submit(() -> {
                latch.await();
                for (int j = 0; j < perThread; j++) counter.increment();
                return null;
            }));
        latch.countDown();
        for (var f : futures) f.get(30, TimeUnit.SECONDS);
    }
    assertEquals(threads * perThread, counter.get());
}

// Use jcstress for low-level concurrency correctness tests:
//   https://github.com/openjdk/jcstress
```
