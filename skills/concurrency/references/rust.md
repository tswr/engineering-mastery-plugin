# Rust Concurrency Reference

Idiomatic Rust concurrent patterns. Ownership enforces thread safety at compile time: Send marks types safe to transfer between threads, Sync marks types safe to share by reference. Data races are impossible in safe Rust.

---

## Shared Mutable State

```rust
use std::sync::{Arc, Mutex, RwLock};
use std::thread;

// Arc<Mutex<T>>: shared ownership + mutual exclusion
let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];
for _ in 0..10 {
    let counter = Arc::clone(&counter);
    handles.push(thread::spawn(move || {
        let mut num = counter.lock().unwrap();  // RAII guard — unlocks on drop
        *num += 1;
    }));
}
for h in handles { h.join().unwrap(); }
assert_eq!(*counter.lock().unwrap(), 10);

// RwLock: multiple readers or one exclusive writer
let config = Arc::new(RwLock::new(HashMap::new()));
let val = config.read().unwrap().get("key").cloned();  // shared read lock
config.write().unwrap().insert("key".into(), "val".into());  // exclusive write

// Prefer confinement: move ownership into the thread
let data = vec![1, 2, 3];
let handle = thread::spawn(move || {
    data.iter().sum::<i32>()  // data moved here — caller cannot access it
});
```

## Synchronization Primitives

```rust
use std::sync::atomic::{AtomicBool, AtomicUsize, Ordering};
use std::sync::{Arc, Condvar, Mutex};

// Atomics: lock-free single-value operations
static REQUEST_COUNT: AtomicUsize = AtomicUsize::new(0);
static SHUTDOWN: AtomicBool = AtomicBool::new(false);

fn handle_request() {
    REQUEST_COUNT.fetch_add(1, Ordering::Relaxed);  // relaxed OK for counters
}
fn request_shutdown() {
    SHUTDOWN.store(true, Ordering::Release);  // pairs with Acquire load
}

// Condvar: wait for a condition (wait_while handles spurious wakeups)
let pair = Arc::new((Mutex::new(Vec::<i32>::new()), Condvar::new()));
let (lock, cvar) = &*pair;
lock.lock().unwrap().push(42);   // producer side
cvar.notify_one();
let mut q = lock.lock().unwrap(); // consumer side
q = cvar.wait_while(q, |q| q.is_empty()).unwrap();
```

## Concurrency Patterns

```rust
use std::sync::mpsc;
use std::thread;

// Channels: ownership transfer via message passing
let (tx, rx) = mpsc::channel();
for i in 0..5 {
    let tx = tx.clone();
    thread::spawn(move || { tx.send(format!("msg {i}")).unwrap(); });
}
drop(tx);  // drop original so rx iterator terminates
for msg in rx { println!("{msg}"); }

// sync_channel: bounded, provides backpressure
let (tx, rx) = mpsc::sync_channel(10);  // blocks sender when buffer full
thread::spawn(move || {
    for i in 0..100 { tx.send(i).unwrap(); }
});
for val in rx { process(val); }

// Rayon: data parallelism with work stealing
use rayon::prelude::*;
let total: f64 = data.par_iter()            // parallel iterator
    .map(|x| expensive_transform(*x))
    .sum();
data.par_sort();  // parallel sort, drop-in replacement
```

## Async and Non-Blocking I/O

```rust
use tokio;
use std::time::Duration;

// tokio::spawn + join! for concurrent async tasks
async fn fetch_both() -> (User, Orders) {
    let (user, orders) = tokio::join!(fetch_user(1), fetch_orders(1));
    (user.unwrap(), orders.unwrap())
}

// select!: race futures, cancel losers
async fn fetch_with_timeout() -> Result<Data, Error> {
    tokio::select! {
        result = fetch_data() => result,
        _ = tokio::time::sleep(Duration::from_secs(5)) => Err(Error::Timeout),
    }  // losing branch is dropped (cancelled) automatically
}

// Bounded async channel
let (tx, mut rx) = tokio::sync::mpsc::channel(100);
tokio::spawn(async move {
    for i in 0..50 { tx.send(i).await.unwrap(); }  // backpressure via await
});
while let Some(v) = rx.recv().await { process(v).await; }

// JoinSet: structured concurrency (tokio 1.21+)
use tokio::task::JoinSet;
let mut set = JoinSet::new();
for item in items { set.spawn(async move { process(item).await }); }
while let Some(res) = set.join_next().await { handle(res.unwrap()); }
// Remaining tasks cancelled on JoinSet drop
```

## Reliability Patterns

```rust
use std::time::Duration;
use rand::Rng;

async fn retry_with_backoff<F, Fut, T, E>(mut f: F, max_retries: u32) -> Result<T, E>
where F: FnMut() -> Fut, Fut: std::future::Future<Output = Result<T, E>> {
    for attempt in 0..=max_retries {
        match f().await {
            Ok(v) => return Ok(v),
            Err(e) if attempt == max_retries => return Err(e),
            Err(_) => {
                let delay = Duration::from_millis(100 * 2u64.pow(attempt));
                tokio::time::sleep(delay).await; // add jitter in production
            }
        }
    }
    unreachable!()
}

// Timeout on every external call
async fn call_service() -> Result<Response, Error> {
    tokio::time::timeout(Duration::from_secs(5), do_request())
        .await
        .map_err(|_| Error::Timeout)?
}
```

## Testing Concurrent Code

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Arc;

    #[test]
    fn test_counter_under_contention() {
        let counter = Arc::new(ThreadSafeCounter::new());
        let handles: Vec<_> = (0..50).map(|_| {
            let c = Arc::clone(&counter);
            std::thread::spawn(move || { for _ in 0..1000 { c.increment(); } })
        }).collect();
        for h in handles { h.join().unwrap(); }
        assert_eq!(counter.get(), 50_000);
    }

    #[tokio::test]
    async fn test_timeout() {
        let r = tokio::time::timeout(
            Duration::from_millis(100), tokio::time::sleep(Duration::from_secs(10))
        ).await;
        assert!(r.is_err());
    }
}

// ThreadSanitizer:  RUSTFLAGS="-Z sanitizer=thread" cargo +nightly test
// Miri (unsafe race detection):  cargo +nightly miri test
```
