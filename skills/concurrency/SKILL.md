---
name: concurrency
description: "Apply when writing concurrent, parallel, or distributed code: thread safety, synchronization, async patterns, distributed system reliability, consistency models, and resilience patterns. Trigger on mentions of threads, async, locks, distributed systems, retries, circuit breakers, consensus, eventual consistency, or message passing. Language-specific idioms are in references/."
---

# Concurrency and Distributed Systems Skill

Concurrency and distribution are where software meets physics. A single-threaded, single-machine program runs in a comfortable fiction: one thing happens after another, memory is consistent, and failure is all-or-nothing. Concurrency breaks the first assumption. Distribution breaks all three.

The consequences are severe. Concurrency bugs — data races, deadlocks, atomicity violations — are non-deterministic, rarely surface in testing, and corrupt state silently. Distributed failures — network partitions, message duplication, partial outages — are not exceptional but routine. Code that ignores these realities works in development and fails in production under load, at 3 AM, in ways the logs don't explain.

This skill covers both domains because they share a core problem: coordinating independent agents that operate without a single, global view of the world. The principles below are grounded in the foundational literature — Goetz's *Java Concurrency in Practice*, Kleppmann's *Designing Data-Intensive Applications*, Nygard's *Release It!*, Herlihy and Shavit's *The Art of Multiprocessor Programming*, and the distributed systems papers of Lamport, Deutsch, and others.

**Language-specific idioms and examples are in the `references/` directory.** Read the relevant file when generating code:
- `references/cpp.md` — C++ idioms (std::thread, atomics, futures, RAII locks)
- `references/python.md` — Python idioms (asyncio, threading, multiprocessing, GIL considerations)
- `references/rust.md` — Rust idioms (ownership-based thread safety, Send/Sync, tokio, channels)
- `references/java.md` — Java idioms (java.util.concurrent, synchronized, CompletableFuture, virtual threads)

---

## 1. Shared Mutable State is the Root of All Evil

The fundamental problem of concurrent programming is coordinating access to shared mutable state. As Goetz establishes in *Java Concurrency in Practice*, if multiple threads access the same mutable data without proper synchronization, the program is broken — even if it appears to work most of the time.

**Three strategies, in order of preference.** Goetz identifies three approaches to making concurrent code safe, and the ordering matters. First, confinement: don't share the data at all. Each thread owns its own data and never exposes it to others. This is the simplest and most reliable strategy because there is nothing to coordinate. Second, immutability: share the data freely, but never mutate it. Immutable objects are inherently thread-safe — no synchronization required, no possibility of data races, no need for defensive copies. Third, synchronization: share and mutate, but coordinate access through locks or other primitives. This is the most complex and error-prone strategy, and the one where most concurrency bugs live. Prefer these strategies in exactly this order, because each successive option is harder to get right and harder to reason about.

**Message passing achieves confinement.** Channels, actors, and message queues are mechanisms for transferring data between threads or processes without sharing it. When thread A sends a value to thread B through a channel, ownership transfers — A no longer accesses the value. This eliminates shared state by design rather than by discipline.

The actor model, originally described by Hewitt and refined by Agha, takes this further: each actor encapsulates private state and communicates exclusively through asynchronous messages. In languages with ownership semantics (like Rust), the compiler can enforce that sent values are no longer accessible by the sender. In languages without this enforcement, the discipline must be maintained by convention — once you send a value, treat it as gone.

**When you must share mutable state, minimize it.** Isolate the shared data into the smallest possible structure, protect it with a single synchronization mechanism, and keep the rest of the system confined or immutable. The less shared mutable state exists, the smaller the surface area for concurrency bugs. Goetz calls this the "split lock" strategy — rather than one big lock protecting a large object, use fine-grained locks on independent pieces of state. But beware: fine-grained locking introduces the risk of deadlocks and requires careful lock ordering discipline (see Principle 2).

---

## 2. Synchronization Primitives

When confinement and immutability are insufficient and shared mutable state is unavoidable, synchronization primitives coordinate access. Using them correctly requires understanding their semantics, costs, and failure modes.

**Mutexes and intrinsic locks** provide mutual exclusion — only one thread can hold the lock at a time. As Goetz describes, every access to shared mutable state (both reads and writes) must be guarded by the same lock. Locking only the writes is insufficient because readers can observe partially-written, inconsistent state. This is one of the most common concurrency mistakes: developers assume that read-only access is safe without synchronization, but without a memory barrier (which locks provide), a reader may see stale or partially-updated values due to CPU caching and compiler reordering.

**Read-write locks** allow multiple concurrent readers but exclusive writers. Use them when reads heavily dominate writes and the critical section is long enough that mutex contention is measurable. For short critical sections, a plain mutex is often faster because read-write locks have higher bookkeeping overhead. Read-write locks also introduce the risk of writer starvation — if readers continuously hold the lock, a waiting writer may never get access. Most implementations offer a fairness policy to mitigate this, but fairness comes at the cost of throughput.

**Condition variables** enable threads to wait for a specific state change rather than spinning. A thread that needs a queue to be non-empty, for example, waits on a condition variable rather than polling in a loop. Always recheck the condition in a loop after waking — spurious wakeups are permitted by most implementations, and another thread may have consumed the state between the signal and the wakeup. The standard pattern is: acquire lock, check condition in a while loop, wait if condition is false (which atomically releases the lock and suspends the thread), then proceed when the condition is true and the lock is reacquired.

**Atomics** provide lock-free operations on single variables — compare-and-swap, fetch-and-add, and similar primitives. They are appropriate for simple counters, flags, and single-pointer updates. They are not appropriate for protecting multi-variable invariants, where a mutex is necessary. When using atomics, you must understand memory ordering — acquire/release semantics, sequential consistency, and relaxed ordering. The default (sequential consistency) is the safest but most expensive; weaker orderings improve performance but are extremely easy to misuse. Use the default unless profiling demands otherwise and you fully understand the implications.

**Lock ordering prevents deadlocks.** As Goetz emphasizes, deadlocks occur when threads acquire multiple locks in inconsistent orders. Thread A holds lock 1 and waits for lock 2; thread B holds lock 2 and waits for lock 1 — neither can make progress. The solution is a global ordering: assign a logical order to all locks and always acquire them in ascending order. Never acquire a lock while holding a lock that is later in the ordering. Document the ordering explicitly so future maintainers respect it.

**Minimize lock duration.** Hold locks for the minimum duration possible — and never perform I/O, network calls, or unbounded computation while holding a lock. A lock held during a network call means that every other thread waiting for that lock is now blocked for the duration of the network round-trip, including any retries or timeouts. Compute what you need, acquire the lock, update the shared state, release the lock.

**Lock-free and wait-free algorithms.** Herlihy and Shavit describe algorithms that avoid locks entirely, using atomic compare-and-swap operations to make progress. A lock-free algorithm guarantees that at least one thread makes progress in a finite number of steps, even if other threads are stalled. A wait-free algorithm guarantees that every thread completes in a bounded number of steps. These are stronger progress guarantees than locks provide (where a thread holding a lock can stall all waiters).

However, lock-free algorithms are rarely needed outside library code — they are difficult to design, difficult to verify, and easy to get subtly wrong. The ABA problem, memory reclamation, and memory ordering subtleties make hand-written lock-free code a minefield. Use proven implementations from your language's standard library rather than inventing your own.

---

## 3. Concurrency Patterns

Raw synchronization primitives are the assembly language of concurrency — powerful but error-prone. Higher-level patterns compose them into structures that are easier to reason about, harder to misuse, and well-understood enough that their failure modes are documented. As Goetz describes, most concurrent programs can be built from a small number of well-known patterns.

**Producer-consumer.** One or more producers generate work items and place them on a bounded queue; one or more consumers remove items and process them. The bounded queue provides natural backpressure — when the queue is full, producers block until consumers catch up. This decouples the rate of production from the rate of consumption and is the backbone of most concurrent data pipelines.

Goetz identifies this as one of the most fundamental and widely applicable concurrency patterns. The key design decisions are the queue's capacity (which determines how much burst traffic can be absorbed), the blocking policy (block, drop, or reject when full), and the number of consumers (which determines processing throughput). An unbounded queue is almost never correct — it is simply a deferred out-of-memory error.

**Thread pool and executor.** Creating a new thread for each unit of work is wasteful and dangerous — threads are expensive to create, and unbounded thread creation under load leads to resource exhaustion. Use a thread pool that maintains a fixed number of worker threads drawing from a shared work queue. As Goetz describes, executors decouple task submission from task execution, allowing the same code to run on different threading policies without modification.

Size the pool carefully. Too few threads and you underutilize CPU; too many and you pay for context switching and memory overhead. For CPU-bound work, a pool sized to the number of available cores is a reasonable starting point. For I/O-bound work, larger pools may be appropriate because threads spend most of their time blocked. In either case, use a bounded work queue to prevent unbounded memory growth when work arrives faster than it can be processed.

**Fan-out/fan-in.** Distribute independent work items to multiple workers in parallel (fan-out), then collect and combine the results (fan-in). The key discipline is ensuring that results are combined safely — typically through a thread-safe collection or a dedicated aggregator — and that errors in individual workers don't silently disappear. Decide upfront what happens when one worker fails: should the entire operation fail (fail-fast), should remaining results be returned (best-effort), or should the failed item be retried? The error-handling strategy must be explicit, not accidental.

**Pipeline.** Chain processing stages, each running concurrently, connected by queues. Data flows through the stages in order. Each stage can be scaled independently by adding more workers. This pattern is particularly useful for stream processing, ETL, and any workload where different stages have different throughput characteristics. The throughput of the pipeline is limited by its slowest stage — identify the bottleneck and scale that stage first.

**Actor model.** Each actor is an independent entity with private state that processes messages sequentially from its inbox. Actors communicate only by sending asynchronous messages to other actors. Because each actor processes one message at a time, there is no concurrent access to its state and no need for locks. The model eliminates shared mutable state by construction.

The tradeoff is that actors introduce their own complexity. Actors can deadlock if they synchronously wait for each other. Debugging message flows across many actors is nontrivial — a request may traverse dozens of actors before producing a result, and tracing that path requires explicit correlation IDs. Mailbox overflow is a concern when actors receive messages faster than they can process them. Despite these tradeoffs, the actor model scales well and is the foundation of highly concurrent systems like telecommunications switches and distributed databases.

---

## 4. Async and Non-Blocking I/O

Threads are expensive when most of their time is spent waiting for I/O. A web server handling 10,000 concurrent connections with one thread per connection needs 10,000 threads — each consuming stack memory, each adding scheduling overhead. Async programming multiplexes many I/O-bound tasks onto fewer threads, trading programming model simplicity for resource efficiency. The same 10,000 connections can be served by a handful of threads if the I/O is non-blocking.

**Event loops** multiplex I/O across a single thread or small thread pool. When a task initiates an I/O operation, it yields control back to the event loop, which schedules another ready task. The I/O operation completes in the background, and the event loop resumes the waiting task when the result is available. This model handles thousands of concurrent connections with minimal thread overhead. The critical rule of event-loop programming: never block the event loop. A single blocking call — a synchronous file read, a CPU-intensive computation, a sleep — stalls every other task waiting for the loop. Offload blocking work to a dedicated thread pool.

**Futures and promises** represent a value that will be available later. They provide a composable interface for async operations — you can chain transformations, combine multiple futures, and handle errors without nested callbacks. The key discipline is that futures must be awaited or otherwise consumed; a future that is created but never awaited represents a silently lost operation whose errors vanish silently. When combining multiple futures, use the appropriate combinator for the semantics you need: wait for all (and fail if any fails), wait for all (collecting individual successes and failures), or wait for the first to complete.

**Structured concurrency.** As described by Nathaniel Smith in "Notes on structured concurrency," concurrent tasks should be scoped to a parent task or scope. When the parent is cancelled or completes, all child tasks are cancelled and awaited. This prevents orphaned tasks — concurrent operations that outlive the code that spawned them, leaking resources or producing results that nobody consumes. Structured concurrency makes the lifetime of concurrent work as explicit and predictable as the lifetime of a local variable. Without it, fire-and-forget task spawning creates the concurrent equivalent of goto — unstructured control flow that is difficult to reason about, debug, or cancel cleanly.

**Backpressure.** When a consumer is slower than a producer, there are only three options: buffer unboundedly (eventually exhausting memory), drop data (losing work), or signal the producer to slow down (backpressure). Backpressure is almost always the correct choice.

Implement it through bounded queues (the simplest mechanism — when the queue is full, the producer blocks or receives an error), flow control protocols (like TCP's sliding window), or reactive streams frameworks that propagate demand signals upstream. A system without backpressure will eventually fail under load — the only question is when. The failure mode is insidious: the system works fine under normal load, buffers grow slowly under moderate overload, and then suddenly exhausts memory and crashes under sustained peak load.

---

## 5. The Fallacies of Distributed Computing

Moving from concurrency on a single machine to distribution across multiple machines introduces a qualitatively different class of problems. Peter Deutsch identified eight assumptions that developers implicitly make about distributed systems — all of which are false. These are not academic curiosities; they are the root cause of most distributed system failures.

**The eight fallacies.** The network is reliable. Latency is zero. Bandwidth is infinite. The network is secure. Topology doesn't change. There is one administrator. Transport cost is zero. The network is homogeneous. Every distributed system bug can be traced to code that assumed one of these. As Vitillo notes in *Understanding Distributed Systems*, these fallacies are especially dangerous because code that violates them works perfectly in development — on localhost, latency is near zero, the network is reliable, and there is one administrator. The failures only appear in production, under load, across data centers.

**Design for partial failure.** As Kleppmann emphasizes in *Designing Data-Intensive Applications*, any network call can fail, arrive late, arrive twice, or succeed on the remote side while the response is lost. Unlike a local function call, which either returns or throws, a network call has a third outcome: it can simply hang indefinitely, leaving the caller uncertain whether the operation succeeded. This is the fundamental difference between local and remote calls — you cannot distinguish between a slow response and no response.

Unlike a single machine, which either works or crashes completely, a distributed system can be partially broken — some nodes up, some down, some slow — in an enormous number of configurations. The number of possible failure modes grows combinatorially with the number of components. You cannot test them all. You can only design for the general case: any component may be unavailable at any time.

**Impossibility results constrain what you can build.** The Two Generals' Problem demonstrates that reliable agreement over an unreliable channel is impossible in the general case — no finite number of message exchanges can guarantee that both parties know they agree. The FLP impossibility result (Fischer, Lynch, Paterson, 1985) proves that in an asynchronous system where even one process can crash, no deterministic consensus algorithm can guarantee both safety (never making a wrong decision) and liveness (eventually making a decision).

These are not limitations of current technology — they are mathematical facts. Practical consensus protocols work around them by making additional assumptions. Lamport's Paxos and its successor Raft assume partial synchrony — that the network is eventually timely, even if temporarily asynchronous. Under this assumption, they guarantee safety always and liveness eventually. Understanding these tradeoffs prevents you from attempting to build something that is provably impossible.

**Accept uncertainty; design around it.** Rather than trying to eliminate distributed failure, design systems that behave acceptably when failures occur. This means timeouts on every call, retries with idempotency, circuit breakers on external dependencies, and graceful degradation when a component is unavailable. As Vitillo emphasizes, the goal is not to prevent failures — that is impossible — but to limit their blast radius and recover automatically when conditions improve.

---

## 6. Reliability Patterns

Knowing that distributed failures are inevitable is only the first step. The next step is building systems that degrade gracefully rather than catastrophically when those failures occur. The patterns in this section, drawn primarily from Nygard's *Release It!*, are the engineering response to the fallacies. They are not optimizations — they are survival mechanisms. A production system without these patterns will eventually suffer cascading failures where one slow dependency brings down the entire service.

**Timeouts.** Every outgoing network call must have a timeout. A call without a timeout is a latency bomb — if the remote service hangs, your thread hangs, your thread pool fills, and your entire service becomes unresponsive. Nygard is unequivocal: no timeout means no limit on how long you'll wait, and unbounded waits propagate failures across service boundaries.

Set timeouts based on the expected latency of the dependency plus a reasonable margin, not on arbitrary large values. A 30-second timeout on a call that normally completes in 50 milliseconds means you're willing to wait 600 times longer than expected before giving up — during which time you're holding a thread hostage. Consider separate connection timeouts (how long to wait for a TCP connection) and request timeouts (how long to wait for a response once connected).

**Retries with exponential backoff and jitter.** Transient failures — network blips, temporary overloads — resolve themselves if you retry after a delay. Exponential backoff increases the delay between retries (1s, 2s, 4s, 8s) to avoid hammering a recovering service. Jitter adds randomness to the delay so that many clients retrying simultaneously don't create a thundering herd that overloads the service again. Retries without backoff and jitter can convert a brief outage into a sustained one.

Cap the number of retries — unbounded retries under sustained failure will queue up requests until memory is exhausted. Only retry operations that are idempotent and that failed with a retriable error (network timeout, 503); retrying a 400 Bad Request is pointless and retrying a non-idempotent POST without an idempotency key risks creating duplicate records. Distinguish between errors that are worth retrying (timeouts, connection resets, server overload) and errors that indicate a permanent problem (authentication failure, invalid request, resource not found).

**Circuit breaker.** As described by Nygard, a circuit breaker tracks recent failures to an external dependency. It operates in three states. Closed: calls pass through normally and failures are counted. When failures exceed a threshold within a time window, the breaker transitions to open. Open: calls fail immediately without contacting the dependency, giving it time to recover and preventing resource exhaustion in the caller. After a cooldown period, the breaker transitions to half-open. Half-open: a single probe call is allowed through. If it succeeds, the breaker closes and normal operation resumes. If it fails, the breaker opens again.

The circuit breaker prevents cascading failures — a pattern where a slow or failing dependency causes the calling service to exhaust its thread pool waiting for responses, which in turn causes the calling service's own callers to time out, propagating the failure upstream through the entire system.

**Bulkhead.** Named after the watertight compartments in a ship's hull, a bulkhead isolates failure domains. Nygard describes partitioning thread pools, connection pools, or other resources so that a failing dependency can exhaust only its allocated partition, not the entire system.

A service that uses a single shared thread pool for all outgoing calls will become completely unresponsive when any single dependency hangs — all threads end up blocked waiting for the slow dependency, leaving none available for calls to healthy dependencies. A service with separate pools per dependency limits the blast radius: the slow dependency exhausts its own pool, but the rest of the service continues operating normally. Apply the same principle to connection pools, memory allocations, and any shared resource that a failing component could monopolize.

**Dead letter queue.** Messages that cannot be processed after a configured number of retries are routed to a dead letter queue for investigation rather than retried forever. This prevents a single poison message — a malformed event, an unexpected schema version, an edge case that triggers a bug — from blocking an entire processing pipeline indefinitely. The dead letter queue must be monitored and alerted on. Unprocessed messages represent lost work or undiagnosed bugs, and a growing dead letter queue is a signal that something is systematically wrong.

---

## 7. Consistency and Ordering

In a single-machine, single-threaded program, there is one copy of the data and it is always consistent. In a distributed system, data is replicated across nodes for availability and performance, and those replicas can diverge. You must choose what kind of consistency you need — because strong consistency everywhere is expensive and often unnecessary, while no consistency guarantees at all leads to user-visible anomalies.

**Strong consistency (linearizability)** means that every operation appears to take effect atomically at some point between its invocation and completion, and all nodes agree on the order. As Kleppmann describes, linearizability is the strongest guarantee and the easiest to reason about, but it requires coordination (typically consensus protocols like Paxos or Raft) and imposes latency and availability costs. Every linearizable operation must wait for a majority of replicas to acknowledge before returning, which means cross-datacenter latency on every write. Use it only where correctness requires it — financial balances, unique constraints, leader election.

**Eventual consistency** means that if no new writes occur, all replicas will eventually converge to the same value. As Kleppmann explains, this is a deliberately weak guarantee — it says nothing about how long convergence takes or what intermediate states readers might observe. The "eventually" in eventual consistency might be milliseconds or hours, depending on the system.

Many systems can tolerate eventual consistency: a social media feed, a product catalog, a metrics dashboard. The key is identifying which parts of your system can tolerate stale reads and which cannot. Often the answer is mixed: within the same application, user profile data may be eventually consistent while account balance must be strongly consistent. Design your data architecture to apply the appropriate consistency level per data type, not globally.

**Causal consistency** preserves cause-and-effect ordering. If operation A causally precedes operation B (B saw A's result), then every node observes A before B. Unrelated operations — those with no causal dependency — may appear in different orders on different nodes, and that is acceptable. Causal consistency is stronger than eventual but weaker (and cheaper) than linearizability, and is sufficient for many applications.

Lamport's happened-before relation is the formal foundation for causal ordering. Two events are causally related if one could have influenced the other — either they occurred in the same process (in program order), or one is the send of a message and the other is the corresponding receive. Vector clocks and their variants are the standard mechanism for tracking causal dependencies in a distributed system.

**Idempotency.** As Kleppmann emphasizes, idempotency is essential in any distributed system. Because messages can be duplicated — by retries, by at-least-once delivery, by network replays — every operation must produce the same result when applied multiple times. Design operations so that applying them once or ten times produces identical outcomes. Use idempotency keys (a unique identifier per logical operation, stored server-side to detect duplicates), conditional writes (only update if the current version matches), or content-based deduplication to achieve this. An operation that is not idempotent in a distributed system is a bug waiting for a retry to trigger it.

**Saga pattern.** Originally described by Garcia-Molina and Salem (1987), a saga is a sequence of local transactions where each step has a compensating action that undoes its effect. If step three of five fails, the saga executes compensating actions for steps two and one in reverse order. Sagas provide eventual atomicity for long-running, cross-service transactions without distributed locks. The tradeoff is complexity: compensating actions must be idempotent (they might be retried), they must handle the case where the forward action partially succeeded, and the saga coordinator itself must be durable — if it crashes mid-saga, it must be able to resume from where it left off.

**Event sourcing.** Instead of storing the current state, store the append-only sequence of events that produced it. The current state is a derived view, computed by replaying events. This provides a complete audit trail, enables temporal queries ("what was the state at time T?"), and supports rebuilding read models from the same event log. As Kleppmann discusses, event logs serve as a single source of truth from which multiple materialized views can be derived. The tradeoff is increased storage, the need for event schema evolution, and the complexity of eventual consistency between the event log and derived views. Event sourcing pairs naturally with idempotent consumers — since events may be replayed during recovery, consumers must handle reprocessing the same event without producing duplicate side effects.

---

## 8. Testing Concurrent and Distributed Code

Testing concurrent and distributed code is fundamentally harder than testing sequential code, because the bugs are non-deterministic. A test that passes a thousand times proves nothing if the bug manifests on the thousand-and-first run under slightly different timing. As Goetz warns, tests that "pass most of the time" are not passing — they are hiding bugs. Conventional unit testing — write input, check output — is necessary but insufficient. You need specialized techniques that explore the space of possible interleavings and failure scenarios.

**Stress testing.** Run concurrent code with many threads, high contention, and sustained load. Vary the number of threads, the ratio of readers to writers, and the sizes of shared data structures. Many concurrency bugs only manifest under contention that far exceeds typical development-machine loads. Run stress tests repeatedly — hundreds or thousands of iterations — because a single pass proves very little. Introduce artificial delays and yielding at critical points to increase the likelihood of exposing race conditions. As Goetz notes, the window for a race condition may be a single instruction wide — stress testing widens that window through sheer repetition.

**Data race detectors.** Tools like ThreadSanitizer (TSan) and Helgrind instrument memory accesses at runtime to detect data races — unsynchronized concurrent accesses where at least one is a write. These tools find races that are nearly impossible to catch through manual review or conventional testing. Run them in CI on every commit. A single data race is undefined behavior in C and C++, and a correctness violation in any language. The cost is a 5-15x slowdown at runtime, which is why they are run in CI and testing, not production. But the cost of not running them — shipping a data race to production — is orders of magnitude higher.

**Deterministic simulation testing.** As described in the context of FoundationDB and discussed by Kleppmann, deterministic simulation replaces the real network, disk, and clock with controlled, deterministic implementations. All sources of non-determinism — scheduling, network delays, failures — are driven by a seeded random number generator. This makes distributed tests fully reproducible: a failing test can be replayed with the same seed to reproduce the exact failure.

This technique is the gold standard for testing distributed systems, though it requires significant infrastructure investment — the entire I/O layer must be abstracted behind interfaces that can be swapped for deterministic implementations. The payoff is enormous: FoundationDB credits deterministic simulation with allowing them to find and fix bugs that would be virtually impossible to reproduce otherwise.

**Property-based testing for invariants.** Rather than testing specific scenarios, define the invariants that must hold (e.g., "the total balance across all accounts is constant," "the queue never contains duplicates") and generate random sequences of concurrent operations. Verify that invariants hold after every sequence. This explores the state space far more effectively than hand-written test cases. When a failing sequence is found, the framework shrinks it to the minimal reproducing case — often revealing the exact interleaving that violates the invariant.

**Chaos engineering.** Pioneered at Netflix, chaos engineering injects failures — network partitions, node crashes, latency spikes, disk errors — into production-like environments to discover weaknesses before they cause real outages. The practice is grounded in the observation that distributed systems are too complex to reason about analytically; the only way to build confidence is to break them deliberately and observe the behavior.

Start with the question "what happens when X fails?" and verify that the answer matches your expectations. As Nygard emphasizes in *Release It!*, failures in production will combine in ways you never anticipated — chaos engineering is how you discover those combinations before your users do. Begin with simple experiments (kill a single instance, add latency to one dependency) and increase scope as confidence grows. Always have a hypothesis, a way to measure the impact, and an abort mechanism to stop the experiment if it causes unexpected damage.

---

## Applying This Skill

When generating concurrent or distributed code, apply these defaults automatically:
1. Prefer confinement over immutability over synchronization, in that order
2. Use message passing or channels to transfer data between threads rather than shared memory
3. Protect all shared mutable state with a single, clearly identified synchronization mechanism
4. Hold locks for the minimum duration; never perform I/O under a lock
5. Acquire multiple locks in a consistent global order to prevent deadlocks
6. Use thread pools rather than spawning threads directly
7. Never block an event loop; offload blocking work to a separate thread pool
8. Set explicit timeouts on every outgoing network call
9. Implement retries with exponential backoff, jitter, and a retry cap
10. Make all distributed operations idempotent
11. Apply circuit breakers to external dependencies
12. Use structured concurrency to scope task lifetimes
13. Choose the weakest consistency model that satisfies correctness requirements

When reviewing concurrent or distributed code, verify these properties:
- Every access to shared mutable state is synchronized (or the data is confined/immutable)
- Locks are acquired in a consistent order across all code paths
- Every network call has a timeout
- Every retry loop has a maximum retry count and backoff
- Operations that may be retried are idempotent
- Task/thread lifetimes are bounded and scoped

When the target language is known, read the corresponding file in `references/` for language-specific idioms and examples.
