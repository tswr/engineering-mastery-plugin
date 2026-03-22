# Java Algorithms Reference

Idiomatic Java patterns for algorithmic problem-solving. The `java.util` Collections Framework provides battle-tested implementations: `HashMap`, `TreeMap`, `HashSet`, `TreeSet`, `PriorityQueue`, `ArrayDeque`, and `ArrayList`. The `Arrays` and `Collections` utility classes offer sorting and binary search. Streams add functional-style processing when readability matters more than micro-optimization.

---

## Arrays, Strings, and Hash Tables

```java
import java.util.*;

// HashMap: O(1) average lookup — two-sum pattern
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (seen.containsKey(complement))
            return new int[] { seen.get(complement), i };
        seen.put(nums[i], i);
    }
    throw new IllegalArgumentException("no solution");
}

// Frequency counting with merge — increment or insert in one call
Map<String, Integer> freq = new HashMap<>();
for (String word : words)
    freq.merge(word, 1, Integer::sum);

// HashSet: add returns true if element was new
Set<String> visited = new HashSet<>();
if (visited.add(node)) { /* first visit only */ }

// Prefix sums — precompute O(n), query O(1)
int[] prefix = new int[nums.length + 1];
for (int i = 0; i < nums.length; i++)
    prefix[i + 1] = prefix[i] + nums[i];
// sum of nums[l..r) == prefix[r] - prefix[l]
```

## Trees and Graphs

```java
// BFS using ArrayDeque — O(V + E), preferred over LinkedList for queues
public int bfsShortest(Map<Integer, List<Integer>> graph, int start, int end) {
    Deque<int[]> queue = new ArrayDeque<>();
    Set<Integer> visited = new HashSet<>();
    queue.offer(new int[]{start, 0});
    visited.add(start);
    while (!queue.isEmpty()) {
        int[] curr = queue.poll();
        if (curr[0] == end) return curr[1];
        for (int nb : graph.getOrDefault(curr[0], List.of())) {
            if (visited.add(nb))
                queue.offer(new int[]{nb, curr[1] + 1});
        }
    }
    return -1;
}

// Dijkstra — O((V + E) log V), PriorityQueue as min-heap
public int[] dijkstra(List<List<int[]>> adj, int src) {
    int[] dist = new int[adj.size()];
    Arrays.fill(dist, Integer.MAX_VALUE);
    var pq = new PriorityQueue<int[]>(Comparator.comparingInt(a -> a[0]));
    dist[src] = 0;
    pq.offer(new int[]{0, src});
    while (!pq.isEmpty()) {
        int[] top = pq.poll();
        if (top[0] > dist[top[1]]) continue;  // stale entry
        for (int[] e : adj.get(top[1]))
            if (dist[top[1]] + e[1] < dist[e[0]])
                pq.offer(new int[]{dist[e[0]] = dist[top[1]] + e[1], e[0]});
    }
    return dist;
}
```

## Sorting and Searching

```java
// Arrays.sort — dual-pivot quicksort for primitives, TimSort for objects
int[] vals = {5, 2, 8, 1, 9};
Arrays.sort(vals);  // in-place, ascending

// Custom comparator — sort intervals by end time
int[][] intervals = {{1,4}, {2,3}, {0,6}};
Arrays.sort(intervals, Comparator.comparingInt(a -> a[1]));

// Multiple sort keys
players.sort(Comparator.comparingInt(Player::getScore).reversed()
                       .thenComparing(Player::getName));

// Binary search — O(log n), requires sorted input
int idx = Arrays.binarySearch(sorted, 12);
// If not found: returns -(insertion point) - 1
int pos = Collections.binarySearch(sortedList, 12);  // for Lists

// TreeMap — red-black tree, O(log n), sorted keys with range queries
TreeMap<Integer, String> tree = new TreeMap<>();
tree.put(5, "five"); tree.put(2, "two"); tree.put(8, "eight");
tree.ceilingEntry(3);   // 5->"five" (smallest key >= 3)
tree.floorEntry(7);     // 5->"five" (largest key <= 7)
tree.subMap(3, true, 8, false);  // keys in [3, 8)
```

## Dynamic Programming

```java
// Top-down with computeIfAbsent — memoize in one call
private Map<Integer, Long> memo = new HashMap<>();
public long fib(int n) {
    if (n < 2) return n;
    return memo.computeIfAbsent(n, k -> fib(k - 1) + fib(k - 2));
}

// Bottom-up: 0/1 knapsack, O(n*W) time, O(W) space
public int knapsack01(int[] weights, int[] values, int capacity) {
    int[] dp = new int[capacity + 1];
    for (int i = 0; i < weights.length; i++)
        for (int c = capacity; c >= weights[i]; c--)  // right-to-left
            dp[c] = Math.max(dp[c], dp[c - weights[i]] + values[i]);
    return dp[capacity];
}

// Edit distance — O(m*n) time, O(n) space
public int editDistance(String s, String t) {
    int m = s.length(), n = t.length();
    int[] prev = new int[n + 1], curr = new int[n + 1];
    for (int j = 0; j <= n; j++) prev[j] = j;
    for (int i = 1; i <= m; i++) {
        curr[0] = i;
        for (int j = 1; j <= n; j++) {
            if (s.charAt(i-1) == t.charAt(j-1)) curr[j] = prev[j-1];
            else curr[j] = 1 + Math.min(prev[j], Math.min(curr[j-1], prev[j-1]));
        }
        int[] tmp = prev; prev = curr; curr = tmp;
    }
    return prev[n];
}
```

## Choosing Data Structures

```java
// PriorityQueue — min-heap by default, O(log n) offer/poll, O(1) peek
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());

// ArrayDeque — O(1) both ends, use as stack or queue
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.push(2); stack.pop();  // 2 (LIFO)
Deque<Integer> queue = new ArrayDeque<>();
queue.offer(1); queue.offer(2); queue.poll();  // 1 (FIFO)

// TreeSet — sorted, O(log n), range queries
TreeSet<Integer> sorted = new TreeSet<>(List.of(5, 2, 8, 1, 9));
sorted.ceiling(3);  // 5 — smallest >= 3
sorted.floor(7);    // 5 — largest <= 7
sorted.subSet(2, true, 8, true);  // {2, 5, 8}

// LinkedHashMap — insertion order + O(1) lookup, LRU cache pattern
class LRUCache extends LinkedHashMap<Integer, Integer> {
    private final int capacity;
    LRUCache(int cap) { super(cap, 0.75f, true); this.capacity = cap; }
    protected boolean removeEldestEntry(Map.Entry<Integer,Integer> e) { return size() > capacity; }
}

// Streams: functional-style grouping + sorting
List<String> topWords = words.stream()
    .collect(Collectors.groupingBy(w -> w, Collectors.counting()))
    .entrySet().stream()
    .sorted(Map.Entry.<String,Long>comparingByValue().reversed())
    .limit(10).map(Map.Entry::getKey).collect(Collectors.toList());
```
