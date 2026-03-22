# C++ Algorithms Reference

Idiomatic C++ patterns for algorithmic problem-solving. The STL provides containers (`vector`, `unordered_map`, `map`, `set`, `priority_queue`, `deque`, `stack`, `queue`) and algorithm functions (`sort`, `lower_bound`, `nth_element`) that cover the vast majority of needs. Prefer STL over hand-rolled implementations — they are heavily optimized, handle edge cases, and use cache-friendly memory layouts.

---

## Arrays, Strings, and Hash Tables

```cpp
// Two-sum with unordered_map — O(n) average via O(1) hash lookups
std::pair<int,int> two_sum(const std::vector<int>& nums, int target) {
    std::unordered_map<int,int> seen;  // value -> index
    for (int i = 0; i < static_cast<int>(nums.size()); ++i) {
        auto it = seen.find(target - nums[i]);
        if (it != seen.end()) return {it->second, i};
        seen[nums[i]] = i;
    }
    throw std::runtime_error("no solution");
}

// Frequency counting — operator[] default-initializes int to 0
std::unordered_map<char, int> freq;
for (char c : text) ++freq[c];

// unordered_set for O(1) average membership testing
std::unordered_set<std::string> visited;
if (visited.insert(node).second) {  // .second is true if insertion happened
    // process node — first visit only
}

// Prefix sums — precompute once, query O(1)
std::vector<long long> prefix(nums.size() + 1, 0);
for (size_t i = 0; i < nums.size(); ++i)
    prefix[i + 1] = prefix[i] + nums[i];
// sum of nums[l..r) == prefix[r] - prefix[l]
```

## Trees and Graphs

```cpp
// BFS — shortest path in unweighted graph, O(V + E)
int bfs_shortest(const std::unordered_map<int, std::vector<int>>& graph,
                 int start, int end) {
    std::queue<std::pair<int,int>> q;
    std::unordered_set<int> visited;
    q.push({start, 0});
    visited.insert(start);
    while (!q.empty()) {
        auto [node, dist] = q.front();  // structured binding (C++17)
        q.pop();
        if (node == end) return dist;
        for (int nb : graph.at(node)) {
            if (visited.insert(nb).second)
                q.push({nb, dist + 1});
        }
    }
    return -1;
}

// Dijkstra — O((V + E) log V) with min-heap via greater<>
std::vector<int> dijkstra(const std::vector<std::vector<std::pair<int,int>>>& adj,
                          int src) {
    std::vector<int> dist(adj.size(), INT_MAX);
    std::priority_queue<std::pair<int,int>, std::vector<std::pair<int,int>>,
                        std::greater<>> pq;  // min-heap: smallest distance on top
    dist[src] = 0;
    pq.push({0, src});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;  // stale entry
        for (auto [v, w] : adj[u])
            if (dist[u] + w < dist[v])
                pq.push({dist[v] = dist[u] + w, v});
    }
    return dist;
}
```

## Sorting and Searching

```cpp
// std::sort — introsort, O(n log n) worst-case guaranteed
std::vector<int> vals = {5, 2, 8, 1, 9};
std::sort(vals.begin(), vals.end());                   // ascending
std::sort(vals.begin(), vals.end(), std::greater<>());  // descending

// Custom comparator — sort intervals by end time
struct Interval { int start, end; };
std::sort(intervals.begin(), intervals.end(),
          [](const Interval& a, const Interval& b) { return a.end < b.end; });

// lower_bound / upper_bound — binary search on sorted range, O(log n)
std::vector<int> sorted = {2, 5, 5, 8, 12, 16};
auto lo = std::lower_bound(sorted.begin(), sorted.end(), 5);  // first >= 5
auto hi = std::upper_bound(sorted.begin(), sorted.end(), 5);  // first > 5
int count_of_5 = static_cast<int>(hi - lo);  // 2

// nth_element — O(n) average, places k-th smallest at correct position
std::nth_element(data.begin(), data.begin() + 2, data.end());

// partial_sort — O(n log k), sort only the first k elements
std::partial_sort(data.begin(), data.begin() + 3, data.end());

// Structured bindings for map iteration (C++17)
for (const auto& [word, count] : ordered_freq) { /* sorted key order */ }
```

## Dynamic Programming

```cpp
// Top-down with memoization
std::unordered_map<int, long long> memo;
long long fib(int n) {
    if (n < 2) return n;
    auto it = memo.find(n);
    if (it != memo.end()) return it->second;
    return memo[n] = fib(n - 1) + fib(n - 2);
}

// Bottom-up: 0/1 knapsack, O(n*W) time, O(W) space
int knapsack_01(const std::vector<int>& w, const std::vector<int>& v, int cap) {
    std::vector<int> dp(cap + 1, 0);
    for (size_t i = 0; i < w.size(); ++i)
        for (int c = cap; c >= w[i]; --c)  // right-to-left: each item once
            dp[c] = std::max(dp[c], dp[c - w[i]] + v[i]);
    return dp[cap];
}

// Edit distance — O(m*n) time, O(n) space with row swapping
int edit_distance(const std::string& s, const std::string& t) {
    int m = s.size(), n = t.size();
    std::vector<int> prev(n + 1), curr(n + 1);
    for (int j = 0; j <= n; ++j) prev[j] = j;
    for (int i = 1; i <= m; ++i) {
        curr[0] = i;
        for (int j = 1; j <= n; ++j) {
            if (s[i-1] == t[j-1]) curr[j] = prev[j-1];
            else curr[j] = 1 + std::min({prev[j], curr[j-1], prev[j-1]});
        }
        std::swap(prev, curr);
    }
    return prev[n];
}
```

## Choosing Data Structures

```cpp
// std::array vs std::vector
std::array<int, 4> dirs = {0, 1, 0, -1};  // fixed size, stack-allocated, zero overhead
std::vector<int> dynamic;
dynamic.reserve(1000);  // pre-allocate to avoid repeated reallocations

// priority_queue — max-heap by default
std::priority_queue<int> max_heap;
max_heap.push(3); max_heap.push(1); max_heap.push(4);
max_heap.top();  // 4

// min-heap via greater<> comparator
std::priority_queue<int, std::vector<int>, std::greater<>> min_heap;

// std::set / std::map — red-black tree, O(log n), sorted, supports range queries
std::set<int> sorted_set = {3, 1, 4, 5};
auto it = sorted_set.lower_bound(3);  // iterator to first >= 3

// std::deque — O(1) push/pop at both ends, O(1) random access
std::deque<int> dq;
dq.push_back(1);
dq.push_front(0);
dq.pop_front();  // O(1)
```
