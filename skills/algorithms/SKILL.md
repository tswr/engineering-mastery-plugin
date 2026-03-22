---
name: algorithms
description: "Apply when choosing data structures, analyzing complexity, or implementing algorithmic solutions: Big O analysis, arrays, trees, graphs, sorting, searching, dynamic programming, and greedy algorithms. Trigger on mentions of time complexity, space complexity, Big O, data structures, sorting, searching, dynamic programming, or graph algorithms. Language-specific idioms are in references/."
---

# Algorithms and Data Structures Skill

Apply algorithmic thinking to every design decision that involves data. The choice of data structure and algorithm determines whether a system handles ten users or ten million. Every working programmer needs fluency in complexity analysis, the core data structures, and the fundamental algorithm families — not as academic exercises, but as practical tools for building software that performs.

This skill covers the essential toolkit: analyzing performance with Big O, choosing the right data structure for the job, and applying the major algorithmic strategies (divide and conquer, dynamic programming, greedy). When you write or review code, think in terms of access patterns and growth rates, not just correctness.

The difference between a junior and senior engineer often comes down to data structure and algorithm fluency. A senior engineer recognizes that an O(n²) nested loop can become O(n) with a hash table, that a priority queue eliminates a manual sort, or that a problem has the structure of a shortest-path search. This recognition comes from knowing the toolkit and practicing its application.

> **Language-specific implementations:** See the `references/` directory for idiomatic usage of standard library data structures and algorithms in:
> - **C++** — STL containers, `<algorithm>`, iterator patterns
> - **Python** — built-in types, `collections`, `heapq`, `bisect`, `itertools`
> - **Rust** — `std::collections`, iterator chains, ownership-aware data structures
> - **Java** — Collections Framework, `java.util` structures, `Comparable`/`Comparator`

---

## 1. Complexity Analysis

**Understand asymptotic notation as the language of performance.** Big O (O) gives the upper bound — the worst an algorithm can do as input grows. Big Omega (Ω) gives the lower bound — the best any algorithm for a problem can do. Big Theta (Θ) gives a tight bound — both upper and lower. In practice, Big O dominates engineering conversations, but knowing all three prevents sloppy reasoning. O(n log n) sorting doesn't mean every sort takes n log n steps — it means it never takes more than that (up to a constant factor).

**Know the common complexity classes and what they feel like at scale.** O(1) — constant, independent of input size. O(log n) — grows very slowly; binary search on a billion elements takes ~30 steps. O(n) — linear; you touch each element once. O(n log n) — the sweet spot for sorting and many divide-and-conquer algorithms. O(n²) — quadratic; bearable for n < 10⁴, painful beyond that. O(2ⁿ) — exponential; only feasible for n < 20-25. Knowing these thresholds tells you instantly whether an approach is viable for your input size.

**Compute complexity for any code you write or review.** Walk through the loops. Nested loops over the same input typically mean O(n²). A loop that halves the input each iteration means O(log n). Recursive calls that branch into two subproblems of half size give O(n log n). For recursive algorithms, use the Master Theorem: if T(n) = aT(n/b) + O(nᵈ), the complexity depends on comparing log_b(a) to d. If you cannot state the time and space complexity of a function, you do not understand it well enough to ship it.

**Trade time for space and vice versa.** Many optimizations boil down to this tradeoff. Caching or precomputation spends memory to save time. In-place algorithms save memory at the cost of more computation or complexity. Make the tradeoff explicit and intentional, not accidental.

**Use amortized analysis for operations with variable cost.** Some operations are expensive occasionally but cheap on average. A dynamic array that doubles its capacity on overflow has O(n) cost for that one resize, but O(1) amortized cost per insertion across a sequence of insertions. Amortized cost is the correct way to reason about these patterns — not worst-case per operation. Other examples of amortized O(1) operations: splay tree access over a sequence of operations, and hash table insertions with periodic rehashing.

**Distinguish worst-case from average-case.** Worst-case guarantees matter for real-time systems, latency-sensitive services, and adversarial inputs. Average-case analysis matters for batch processing and typical workloads. Know which one your system needs. Quicksort is O(n²) worst-case but O(n log n) average — that worst-case may or may not matter depending on your context. Best-case analysis is rarely useful for decision-making — an algorithm that is fast on easy inputs is not necessarily a good algorithm.

**Understand expected-case for randomized algorithms.** Randomized algorithms (like randomized quicksort or randomized quickselect) use random choices to avoid worst-case inputs. Their expected time is computed over the random choices of the algorithm, not over the distribution of inputs — this is a stronger guarantee than average-case, which assumes inputs follow some distribution.

**Do back-of-the-envelope estimates before coding.** Estimate the input size, multiply by the complexity, and check whether it finishes in time. An O(n²) algorithm on 10⁶ elements means ~10¹² operations — that won't finish in a second. An O(n log n) algorithm on the same input means ~2×10⁷ operations — that will. This five-second calculation prevents hours of wasted implementation. A modern CPU executes roughly 10⁸ to 10⁹ simple operations per second — use this as your baseline for estimation.

**Don't confuse Big O with actual performance.** Big O describes growth rate, not absolute speed. An O(n) algorithm with a constant factor of 1000 is slower than an O(n log n) algorithm with a constant factor of 1 for all practical input sizes. Big O tells you which algorithm wins as n grows large — for small n, constants and cache behavior dominate. Use Big O for design decisions, profiling for final tuning.

---

## 2. Arrays, Strings, and Hash Tables

**Arrays are the foundational data structure.** O(1) random access by index, contiguous memory layout, and excellent cache locality make arrays the default choice when you need sequential or index-based access. Most other data structures are built on top of arrays internally.

**Dynamic arrays solve the fixed-size problem.** Vectors, ArrayLists, and Python lists grow by doubling capacity when full, giving O(1) amortized append. The key insight is that doubling (rather than growing by a fixed amount) makes the total cost of n insertions O(n) rather than O(n²). Shrinking policy matters too — some implementations shrink when the array is one-quarter full (not one-half) to avoid thrashing when alternating between insert and delete at the boundary.

**Strings are arrays of characters with their own idioms.** Most string problems reduce to array problems, but string-specific techniques matter: hashing substrings (Rabin-Karp) enables O(n) average-case pattern matching, the KMP algorithm achieves O(n) worst-case pattern matching by precomputing a failure function, and suffix arrays enable efficient substring search across large texts. For simpler problems, treating strings as character arrays and applying two-pointer or sliding window techniques is usually sufficient.

**Hash tables are the most practically useful data structure.** O(1) average-case lookup, insertion, and deletion by key. When you need to check membership, count occurrences, group items, or deduplicate — a hash table is almost always the right first choice. Understand the tradeoff: hash tables sacrifice ordering and have O(n) worst-case when collisions degrade.

**Understand hash table internals to use them effectively.** Chaining handles collisions with linked lists at each bucket; open addressing probes for the next empty slot. Both work, but load factor management is critical for performance. A good hash function distributes keys uniformly; a bad one clusters keys and destroys performance. When the load factor exceeds a threshold (typically 0.7-0.75), the table resizes and rehashes all entries — an O(n) operation that is O(1) amortized.

**Apply the two-pointer technique for sorted arrays and sequences.** Place one pointer at the start and one at the end, then move them inward based on a condition. This turns many O(n²) brute-force scans into O(n) single-pass solutions. Classic applications: finding pairs that sum to a target, removing duplicates in place, partitioning. The technique also works with two pointers moving in the same direction at different speeds (fast/slow pointers) — useful for cycle detection in linked lists and finding the middle of a list in one pass.

**Use the sliding window for contiguous subarray and substring problems.** Maintain a window defined by two boundaries, expanding and contracting as you scan. This handles problems like "longest substring without repeating characters" or "maximum sum subarray of size k" in O(n) instead of the brute-force O(n²) or O(n³). There are two variants: fixed-size windows (move both boundaries together) and variable-size windows (expand until a condition breaks, then contract). The variable-size variant often uses a hash table to track window contents.

**Precompute prefix sums for range queries.** When you need the sum of any subarray repeatedly, build a prefix sum array once in O(n), then answer each range query in O(1) by subtraction. This pattern generalizes to prefix products, prefix XOR, and 2D prefix sums for matrix range queries. The core insight is preprocessing: spend O(n) time once to answer unlimited queries in O(1) each. This preprocessing mindset applies broadly — whenever you answer the same type of question many times over static data, precompute.

---

## 3. Linked Lists, Stacks, and Queues

**Linked lists trade random access for insertion flexibility.** O(1) insertion and deletion at a known position (given a pointer), but O(n) search and poor cache locality. In practice, prefer arrays unless you specifically need stable pointers, stable iterators during mutation, or O(1) splicing of sublists. Most of the time, arrays with shifting or gap buffers outperform linked lists due to cache effects.

**Know the linked list variants and when each applies.** Singly-linked lists support O(1) insertion/deletion at the head only. Doubly-linked lists add O(1) deletion given a node reference at the cost of extra memory per node. Circular linked lists connect the tail back to the head — useful for round-robin scheduling. The most common practical use of linked lists is as the collision-chain structure inside hash tables and as the internal structure of LRU caches (doubly-linked list + hash table).

**Stacks and queues are restricted-access structures — the restriction is the feature.** A stack (LIFO) enforces reverse-order processing: function call management, undo operations, expression parsing, DFS traversal. A queue (FIFO) enforces order-preserving processing: BFS traversal, task scheduling, buffering between producer and consumer. When you reach for a stack or queue, you are encoding an invariant about processing order. If you need both stack and queue behavior, consider a deque.

**Use monotonic stacks for "next greater/smaller element" problems.** Maintain a stack where elements are always in increasing (or decreasing) order. When a new element violates the order, pop elements and record the relationship. This solves problems like "for each element, find the next larger element" in O(n) instead of O(n²).

**Use deques for sliding window extrema.** A double-ended queue lets you push and pop from both ends in O(1). This makes it possible to track the maximum or minimum in a sliding window in O(n) total, by maintaining a monotonic deque of candidates.

**Implement stacks and queues with arrays, not linked lists, unless you have a reason.** Array-backed stacks and circular-buffer queues have better cache performance and lower memory overhead than linked-list implementations. Most standard libraries implement them this way. Use linked-list-backed versions only when you need guaranteed O(1) operations without amortization or need to share structure between multiple versions of the collection.

---

## 4. Trees and Graphs

**Trees are connected, acyclic graphs.** They model hierarchical relationships: file systems, organizational charts, parse trees, decision trees. Binary search trees (BSTs) provide O(log n) search, insertion, and deletion — but only when balanced. An unbalanced BST degrades to a linked list with O(n) operations.

**Balanced BSTs guarantee O(log n) through structural invariants.** AVL trees maintain strict balance (height difference of at most 1 between subtrees) and are slightly faster for lookups. Red-black trees maintain a weaker balance invariant and are faster for insertions and deletions. B-trees generalize to multiple keys per node, optimizing for disk I/O — they power databases and file systems. In practice, you rarely implement a balanced BST yourself — use your language's standard library sorted map or set, which uses a red-black tree or B-tree internally.

**Understand graph representations and their tradeoffs.** An adjacency list uses O(V + E) space and is efficient for sparse graphs (most real-world graphs). An adjacency matrix uses O(V²) space but provides O(1) edge lookup — use it for dense graphs or when you need to check edge existence frequently. The choice of representation affects both memory usage and the complexity of graph operations. For weighted graphs, adjacency lists store (neighbor, weight) pairs; adjacency matrices store the weight in each cell (with infinity or a sentinel for non-edges).

**Heaps provide efficient priority access.** A binary heap gives O(1) access to the minimum (or maximum) element, O(log n) insertion, and O(log n) extraction. Use a heap whenever you need to repeatedly find and remove the highest-priority item: priority queues, task schedulers, finding the k largest/smallest elements, merge k sorted lists. A heap can be built from an unsorted array in O(n) — faster than inserting n elements one at a time. This matters when you have all elements upfront.

**Tries (prefix trees) excel at string matching.** Each node represents a character, and paths from root to leaf spell out stored strings. Lookup, insertion, and prefix search all take O(m) where m is the string length, independent of how many strings are stored. Use tries for autocomplete, spell checking, and IP routing tables. The tradeoff is memory: a naive trie can use significant space due to pointers at each node. Compressed tries (Patricia tries, radix trees) merge chains of single-child nodes to reduce space overhead.

**BFS explores level by level — use it for shortest paths in unweighted graphs.** BFS visits all nodes at distance d before visiting any node at distance d+1. This guarantees that the first time you reach a node, you reached it via the shortest path (in terms of edge count). BFS also detects whether a graph is bipartite. The time complexity is O(V + E) — every vertex and edge is processed exactly once. BFS uses a queue and requires O(V) extra space for the visited set and the queue itself.

**DFS explores as deep as possible — use it for structure and connectivity.** DFS naturally reveals connected components, cycle detection, topological ordering, and strongly connected components. It uses a stack (explicitly or via recursion) and classifies edges as tree, back, forward, or cross edges — each classification answers a different structural question. Time complexity is O(V + E), same as BFS. Use iterative DFS with an explicit stack for large graphs to avoid stack overflow from deep recursion.

**Learn the shortest path algorithms and when each applies.** Dijkstra's algorithm handles graphs with non-negative edge weights in O((V + E) log V) with a priority queue. Bellman-Ford handles negative weights (and detects negative cycles) in O(VE). BFS handles unweighted graphs in O(V + E). For all-pairs shortest paths, Floyd-Warshall runs in O(V³) using dynamic programming — simple to implement and practical for small, dense graphs. Choosing the wrong algorithm for the weight structure gives wrong answers, not just slow ones.

**Topological sort orders tasks with dependencies.** Given a directed acyclic graph (DAG), topological sort produces a linear ordering where every edge goes from earlier to later. Use it for build systems, course prerequisites, task scheduling — any scenario with "do X before Y" constraints. If the graph has a cycle, no topological order exists, and the algorithm detects this. Two approaches: DFS-based (reverse postorder) and BFS-based (Kahn's algorithm, using in-degree counts). Both run in O(V + E). Kahn's algorithm naturally detects cycles — if the sorted output contains fewer than V vertices, the graph has a cycle.

**Minimum spanning tree connects all vertices with minimum total edge weight.** Kruskal's algorithm sorts edges by weight and greedily adds edges that don't form cycles (using a union-find data structure for cycle detection). Prim's algorithm grows the tree from a starting vertex, always adding the cheapest edge that connects a new vertex (using a priority queue). Both run in O(E log V) with appropriate data structures. Use MST for network design, clustering, and approximation algorithms.

**Recognize problems that are graph problems in disguise.** Many real-world problems map to graphs even when they don't look like graph problems at first glance. A maze is a graph (cells are vertices, passages are edges). Word ladders are graphs (words are vertices, single-letter changes are edges). Social networks, dependency resolution, state machines, web crawling — all are graph problems. When you can model a problem as a graph, you gain access to the entire algorithmic toolkit for graphs: BFS, DFS, Dijkstra, topological sort, and more.

---

## 5. Sorting and Searching

**Comparison-based sorting has a proven Ω(n log n) lower bound.** No comparison-based sort can do better than n log n comparisons in the worst case. This is a mathematical fact derived from decision tree analysis — there are n! possible orderings, and each comparison eliminates at most half the remaining possibilities, requiring at least log₂(n!) ≈ n log n comparisons. Any claim of a faster general-purpose comparison sort is wrong.

**Quicksort is the practical default.** O(n log n) average time with O(log n) extra space (for the recursion stack). Its cache behavior is excellent due to sequential access patterns. The weakness is O(n²) worst-case on already-sorted or adversarial input — mitigate with randomized pivot selection or median-of-three. Most standard library sort implementations use a hybrid approach (introsort: quicksort + heapsort fallback + insertion sort for small partitions) to get quicksort's practical speed with O(n log n) worst-case guarantee.

**Mergesort guarantees O(n log n) and stability.** It always takes O(n log n) regardless of input, and it preserves the relative order of equal elements (stability). The cost is O(n) extra space. Use mergesort when worst-case guarantees matter or when stability is required (sorting records by multiple keys in sequence). Mergesort is also the natural choice for sorting linked lists, where its O(n) space overhead disappears because the merge operation works in-place on linked nodes.

**Heapsort provides O(n log n) worst-case with O(1) extra space.** It builds a heap in O(n), then extracts elements one at a time. Heapsort is not stable and has worse cache behavior than quicksort, so it is rarely the primary sort algorithm. Its value is as a fallback guarantee — introsort switches to heapsort when quicksort's recursion depth suggests worst-case behavior.

**Non-comparison sorts break the lower bound for restricted inputs.** Counting sort, radix sort, and bucket sort achieve O(n) time when input values fall within a known, bounded range (typically integers). Counting sort works by tallying occurrences — it requires O(k) extra space where k is the range of values. Radix sort processes digits from least significant to most significant, using a stable sort (typically counting sort) at each digit — its complexity is O(d × (n + k)) where d is the number of digits. Bucket sort distributes elements into buckets and sorts each bucket — it achieves O(n) when elements are uniformly distributed. These are not general-purpose — they require assumptions about the input domain.

**Binary search finds elements in sorted arrays in O(log n).** The concept is simple but the implementation is notoriously error-prone. The critical decisions are: inclusive or exclusive bounds? Check middle element first or check termination first? Off-by-one errors in binary search have caused bugs in production code for decades. Define your loop invariant precisely, and verify that the search space shrinks on every iteration. Binary search generalizes beyond arrays — any monotonic predicate can be binary-searched to find the transition point. Use this for problems like "find the minimum value that satisfies a condition."

**Quickselect finds the k-th smallest element in O(n) average time.** Based on the partitioning step of quicksort, but only recurses into one half. This is strictly better than sorting when you only need one order statistic. Worst-case is O(n²), but randomized pivot selection makes this unlikely.

**Use insertion sort for small arrays.** Despite being O(n²), insertion sort has tiny constant factors and excellent cache behavior. For arrays smaller than ~20-50 elements, it outperforms quicksort and mergesort. This is why production sort implementations switch to insertion sort for small partitions. Insertion sort is also the best choice for nearly-sorted data — it runs in O(n) when each element is at most k positions from its sorted position.

**Prefer your language's built-in sort.** Standard library sorts are heavily optimized, extensively tested, and handle edge cases you won't think of. Write a custom sort only when you need a specialized comparison, a non-comparison sort for performance, or a sorting network for parallel hardware. In all other cases, use the standard sort with a custom comparator.

---

## 6. Dynamic Programming

**DP applies when a problem has two properties: optimal substructure and overlapping subproblems.** Optimal substructure means the optimal solution contains optimal solutions to subproblems — you can build the best overall answer from the best answers to smaller instances. Overlapping subproblems means the same subproblems are solved repeatedly in a naive recursive approach — a recursive Fibonacci call tree recomputes fib(3) exponentially many times. If only one property holds, DP is not the right tool — divide and conquer handles optimal substructure without overlap (each subproblem is solved once, as in mergesort), and memoization without optimal substructure is just caching.

**The hardest step is defining the state.** The state captures what information you need to make the next decision. The dimensions of your DP table are determined by the state. A poorly chosen state leads to either incorrect results or exponential blowup. Ask: "What do I need to know at this point to make an optimal choice?" The answer defines your state variables. Common state dimensions include: position in a sequence, remaining capacity, subset of items considered, and previous choice made. Adding a dimension multiplies the state space — keep the state as lean as possible.

**Start top-down with memoization, then convert to bottom-up if needed.** Write the recursive solution first — get the recurrence relation right. Add a cache (memoization) to avoid recomputing subproblems. This top-down approach is easier to reason about and debug. It also has the advantage of only computing subproblems that are actually needed, which matters when the state space is sparse. Convert to bottom-up (tabulation) only if you need to optimize space, eliminate recursion overhead, or the recursion depth exceeds stack limits. Don't try to write bottom-up directly — the recurrence is harder to get right without the recursive structure as a guide.

**Optimize space by keeping only what you need.** Many DP problems depend only on the previous row or the previous few states. A 2D table can often be reduced to two 1D arrays, or even a single array updated in place. A 1D table can sometimes be reduced to a few variables. This optimization can change O(n²) space to O(n) or O(n) to O(1). The Fibonacci sequence is the simplest example: the naive DP table is O(n), but since each value depends only on the previous two, O(1) space suffices.

**Recognize the classic DP patterns.** Longest common subsequence, edit distance, knapsack (0/1 and unbounded), longest increasing subsequence, matrix chain multiplication, coin change — these are the foundational problems. They are not just textbook exercises — they appear as subproblems in real systems. Edit distance powers spell checkers and diff tools. LCS drives version control merge algorithms. The knapsack pattern appears in resource allocation, budget optimization, and scheduling. Recognizing the pattern is faster than deriving from scratch.

**Understand when DP is overkill.** Not every optimization problem requires DP. If the problem has the greedy choice property, a greedy algorithm is simpler and faster. If the problem has no overlapping subproblems, plain recursion or divide and conquer suffices. DP adds complexity to the implementation — use it only when the problem structure demands it. A useful heuristic: if the brute-force recursion recalculates the same results (you can verify by logging recursive calls), DP will help. If each recursive call is unique, DP adds nothing.

---

## 7. Greedy Algorithms

**Greedy algorithms make the locally optimal choice at each step.** At every decision point, pick the option that looks best right now, without reconsidering past choices or considering future consequences. This produces fast, simple algorithms — often O(n log n) dominated by an initial sort. The simplicity of greedy algorithms is their greatest advantage — they are easy to implement, easy to debug, and easy to reason about correctness (once the greedy choice property is established).

**Greedy works only when two properties hold: the greedy choice property and optimal substructure.** The greedy choice property means that a locally optimal choice can always be extended to a globally optimal solution. Optimal substructure means the remaining subproblem after making the greedy choice is itself an optimization problem of the same form. If either property fails, greedy gives wrong answers.

**Prove the greedy choice property before trusting a greedy solution.** Many problems look greedy but are not. The coin change problem with standard denominations (1, 5, 10, 25) works greedily, but with arbitrary denominations (1, 3, 4) it does not — greedy gives 4+1+1 for target 6, but optimal is 3+3. The difference between a correct greedy and an incorrect one is a proof, not intuition. The standard proof technique is an exchange argument: assume an optimal solution that does not make the greedy choice, then show you can swap in the greedy choice without worsening the solution.

**Know the classic greedy problems.** Interval scheduling (sort by end time, select non-overlapping intervals) is the canonical example — the proof by exchange argument is clean and instructive. Huffman coding builds an optimal prefix-free code by always merging the two least-frequent symbols. Minimum spanning tree algorithms (Kruskal's sorts edges by weight and adds non-cycle-forming edges; Prim's grows from a vertex by always adding the cheapest crossing edge) are both greedy. Dijkstra's shortest path is greedy — it always processes the closest unvisited vertex. Fractional knapsack (take fractions of items by value-to-weight ratio) is greedy; 0/1 knapsack (take or leave whole items) is not — this distinction illustrates how subtle the boundary between greedy and DP can be.

**Greedy is often the right first approach to try.** It is simple to implement, easy to analyze, and fast to run. When it works, it is usually the best solution. When it doesn't work, the attempt often reveals the problem structure that guides you toward DP or another technique.

**Distinguish greedy from DP by checking re-evaluation.** If the optimal solution at step k can be determined without reconsidering choices made at steps 1 through k-1, the problem is likely greedy. If the optimal solution at step k depends on the specific choices made earlier (not just that they were optimal), you need DP. This is the practical test for whether greedy applies.

---

## 8. Choosing the Right Data Structure

**The choice of data structure is often more important than the choice of algorithm.** The right data structure can reduce an O(n²) algorithm to O(n log n) or O(n). The wrong one forces you into workarounds that obscure the code and kill performance. Before designing an algorithm, design the data. Many algorithmic breakthroughs are really data structure breakthroughs — the algorithm becomes obvious once the data is organized correctly.

**Identify your access pattern first.** What operations dominate? The answer determines the structure:

- Random access by index → array
- Lookup by key → hash table
- Maintaining sorted order → balanced BST or sorted array
- Finding the minimum/maximum → heap
- First-in-first-out processing → queue
- Last-in-first-out processing → stack
- Prefix-based string search → trie
- Grouping elements into mergeable sets → disjoint-set (union-find)
- Range queries on static data → segment tree or prefix sums
- Insertion order with fast lookup → linked hash map

**Consider the full operation profile, not just one operation.** A hash table has O(1) lookup but no ordering. A balanced BST has O(log n) lookup but supports range queries, successor/predecessor, and ordered iteration. If you need both fast lookup and sorted traversal, a BST or skip list may serve better despite the slower individual lookup. Similarly, a heap gives O(1) min access but O(n) search — if you also need to find arbitrary elements, augment with a hash table index or choose a different structure entirely.

**Understand the union-find (disjoint set) data structure.** When the problem involves grouping elements into sets and merging sets, union-find with path compression and union by rank provides nearly O(1) amortized operations (technically O(α(n)), where α is the inverse Ackermann function). This powers Kruskal's MST algorithm, connected component tracking in dynamic graphs, and equivalence class computations.

**Account for constant factors and cache behavior.** Big O ignores constants, but real hardware does not. An O(n) linked list traversal can be slower than an O(n log n) array sort for moderate n due to cache misses. Arrays and hash tables with open addressing have excellent cache locality because they store data contiguously in memory. Trees and linked lists scatter nodes across the heap, causing cache misses on nearly every pointer dereference. For performance-critical code with moderate input sizes, data structure layout matters as much as asymptotic complexity. This is why array-based structures dominate in practice despite the theoretical advantages of pointer-based alternatives.

**Use the right data structure to simplify the algorithm.** If you find yourself writing complex bookkeeping code, you may be fighting the data structure instead of working with it. A priority queue eliminates manual "find the minimum" loops. A set eliminates manual duplicate checking. A stack eliminates manual depth tracking. A sorted map eliminates manual binary search maintenance. When the algorithm feels forced, reconsider the data structure — the code should feel natural, not like you are working against your tools.

**Know the standard library offerings in your language.** Every major language provides hash maps, dynamic arrays, sorted sets/maps, priority queues, and deques in its standard library. These implementations are heavily optimized and battle-tested — they handle edge cases around resizing, hashing, rebalancing, and memory allocation that handwritten versions typically miss. Reach for them before writing your own. Implementing a data structure yourself is justified only when the standard library version doesn't meet a specific requirement (custom memory allocation, lock-free access, specialized iteration order). Check the `references/` directory for language-specific guidance on which standard library structures to use and when.

**When uncertain, prototype and measure.** Asymptotic analysis tells you how algorithms scale, but constant factors, cache behavior, and real-world input distributions can shift the crossover point. For performance-critical decisions between two viable approaches, benchmark with realistic data. Let Big O guide your initial choice, then let measurement confirm it.

---

## Applying This Skill

When writing code, analyze the complexity of your solution before considering it done. State the time and space complexity explicitly — in comments or documentation for non-trivial functions. When choosing a data structure, match it to your access pattern — do not default to an array or list for everything. When solving optimization problems, consider whether greedy, DP, or divide and conquer applies, and verify the required properties before committing.

During code review, flag O(n²) or worse algorithms operating on unbounded input. Question data structure choices that do not match the dominant access pattern. Ask for complexity analysis on any non-trivial function. Watch for hidden quadratic behavior: string concatenation in loops, repeated list searches, nested iterations over the same collection, and calling O(n) operations inside O(n) loops.

When performance matters, profile first — but algorithmic improvement almost always outperforms micro-optimization. Moving from O(n²) to O(n log n) dwarfs any constant-factor tuning. Fix the algorithm before tuning the code. A better data structure or algorithm is the highest-leverage performance optimization available.

When solving a new problem, follow this sequence: understand the constraints (input size, time limit), consider brute force first (to understand the problem), identify the bottleneck (what makes brute force slow), apply the right technique (hash table for lookup, sorting for order, DP for overlapping subproblems, greedy for local-to-global optimality), and verify the complexity meets the constraints before implementing.

When faced with a problem you haven't seen before, decompose it. Most complex algorithmic problems combine two or three basic techniques — a sort followed by binary search, a graph traversal that uses a heap, a DP solution where each state is computed using a hash table lookup. Mastering the individual building blocks lets you assemble solutions to novel problems.
