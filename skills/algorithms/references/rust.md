# Rust Algorithms Reference

Idiomatic Rust patterns for algorithmic problem-solving. The standard library provides `std::collections` (HashMap, BTreeMap, HashSet, BTreeSet, VecDeque, BinaryHeap) alongside powerful `Vec` and slice methods. Iterator combinators enable expressive, zero-cost abstractions for data transformation. Ownership semantics naturally prevent many categories of bugs in graph and tree algorithms.

---

## Arrays, Strings, and Hash Tables

```rust
use std::collections::{HashMap, HashSet, VecDeque, BinaryHeap, BTreeMap};

// HashMap with Entry API — update-or-insert without double lookup
fn word_frequencies(words: &[&str]) -> HashMap<&str, usize> {
    let mut freq = HashMap::new();
    for &word in words {
        *freq.entry(word).or_insert(0) += 1;
    }
    freq
}

// Two-sum with HashMap — O(n) via O(1) lookups
fn two_sum(nums: &[i32], target: i32) -> Option<(usize, usize)> {
    let mut seen: HashMap<i32, usize> = HashMap::new();
    for (i, &num) in nums.iter().enumerate() {
        if let Some(&j) = seen.get(&(target - num)) {
            return Some((j, i));
        }
        seen.insert(num, i);
    }
    None
}

// HashSet — insert returns false if already present
fn has_duplicates(nums: &[i32]) -> bool {
    let mut seen = HashSet::new();
    nums.iter().any(|num| !seen.insert(num))  // O(1) per check
}
```

## Trees and Graphs

```rust
use std::cmp::Reverse;

// BFS using VecDeque — O(V + E), pop_front is O(1)
fn bfs_shortest(graph: &HashMap<i32, Vec<i32>>, start: i32, end: i32) -> i32 {
    let mut queue = VecDeque::new();
    let mut visited = HashSet::new();
    queue.push_back((start, 0));
    visited.insert(start);
    while let Some((node, dist)) = queue.pop_front() {
        if node == end { return dist; }
        if let Some(neighbors) = graph.get(&node) {
            for &nb in neighbors {
                if visited.insert(nb) {  // returns true if new
                    queue.push_back((nb, dist + 1));
                }
            }
        }
    }
    -1
}

// Dijkstra — BinaryHeap is max-heap; Reverse wrapper gives min-heap
fn dijkstra(adj: &[Vec<(usize, u64)>], src: usize) -> Vec<u64> {
    let mut dist = vec![u64::MAX; adj.len()];
    let mut heap = BinaryHeap::new();
    dist[src] = 0;
    heap.push(Reverse((0u64, src)));
    while let Some(Reverse((d, u))) = heap.pop() {
        if d > dist[u] { continue; }  // stale entry
        for &(v, w) in &adj[u] {
            if dist[u] + w < dist[v] { dist[v] = dist[u] + w; heap.push(Reverse((dist[v], v))); }
        }
    }
    dist
}
```

## Sorting and Searching

```rust
// Vec::sort — stable, O(n log n), pattern-defeating quicksort
let mut vals = vec![5, 2, 8, 1, 9];
vals.sort();            // ascending, stable
vals.sort_unstable();   // faster, may reorder equal elements

// sort_by_key and sort_by for custom ordering
intervals.sort_by_key(|&(_, end)| end);  // sort by end time
players.sort_by(|a, b| b.score.cmp(&a.score).then(a.name.cmp(&b.name)));

// binary_search on sorted slice — O(log n)
let sorted = vec![2, 5, 8, 12, 16, 23, 38];
match sorted.binary_search(&12) {
    Ok(idx) => { /* found at idx */ }
    Err(idx) => { /* insert point */ }
}

// partition_point — find first index where predicate is false
let first_ge_10 = sorted.partition_point(|&x| x < 10);  // 3 (index of 12)

// BTreeMap — sorted keys, O(log n), supports range queries
let mut tree: BTreeMap<i32, String> = BTreeMap::new();
tree.insert(5, "five".into());
tree.insert(2, "two".into());
tree.insert(8, "eight".into());
for (k, v) in tree.range(3..=7) {  // keys in [3, 7]
    println!("{k}: {v}");
}

// windows and chunks on slices
let data = vec![1, 2, 3, 4, 5];
for w in data.windows(3) { /* overlapping: [1,2,3], [2,3,4], [3,4,5] */ }
for c in data.chunks(2)  { /* non-overlapping: [1,2], [3,4], [5] */ }
```

## Dynamic Programming

```rust
// Top-down with memoization via HashMap
fn fib(n: u64, memo: &mut HashMap<u64, u64>) -> u64 {
    if n < 2 { return n; }
    if let Some(&v) = memo.get(&n) { return v; }
    let result = fib(n - 1, memo) + fib(n - 2, memo);
    memo.insert(n, result);
    result
}

// Bottom-up: 0/1 knapsack, O(n*W) time, O(W) space
fn knapsack_01(weights: &[usize], values: &[i64], capacity: usize) -> i64 {
    let mut dp = vec![0i64; capacity + 1];
    for (&w, &v) in weights.iter().zip(values.iter()) {
        for c in (w..=capacity).rev() {  // right-to-left: each item once
            dp[c] = dp[c].max(dp[c - w] + v);
        }
    }
    dp[capacity]
}

// Longest increasing subsequence — O(n log n) with partition_point
fn lis_length(nums: &[i32]) -> usize {
    let mut tails: Vec<i32> = Vec::new();
    for &num in nums {
        let pos = tails.partition_point(|&x| x < num);
        if pos == tails.len() { tails.push(num); }
        else { tails[pos] = num; }
    }
    tails.len()
}

```

## Choosing Data Structures

```rust
// Iterator combinators — zero-cost abstractions, single-pass
let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
let sum_even_sq: i64 = nums.iter()
    .filter(|&&x| x % 2 == 0)
    .map(|&x| (x as i64) * (x as i64))
    .sum();  // 4 + 16 + 36 + 64 + 100 = 220

// collect into different container types via turbofish
let set: HashSet<i32> = nums.iter().copied().collect();
let deque: VecDeque<i32> = nums.iter().copied().collect();

// VecDeque — O(1) push/pop at both ends, ideal for sliding windows
let mut window: VecDeque<i32> = VecDeque::new();
for &val in &nums {
    window.push_back(val);
    if window.len() > 3 { window.pop_front(); }
}
```
