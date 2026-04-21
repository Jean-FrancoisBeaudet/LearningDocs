# Algorithmic Patterns — Deep Dive

A practical, pattern-first catalogue of the algorithm templates that cover the vast majority of interview and competitive-programming problems. Each section follows the same template:

- **When to reach for it** — problem signals and constraints
- **Key insight** — the invariant that makes it work
- **Template** — reusable skeleton code (Python)
- **Canonical example** — one well-known problem, annotated
- **Complexity** — tight time and space bounds
- **Common pitfalls** — traps to watch for
- **Related** — cross-pattern links

> All code is Python 3, optimized for clarity over cleverness. Adapt to C++/Java/C# for performance-critical or interview-specific contexts.

## Table of Contents

1. [Two Pointers](#1-two-pointers)
2. [Sliding Window](#2-sliding-window)
3. [Fast & Slow Pointers](#3-fast--slow-pointers)
4. [Prefix Sums & Difference Arrays](#4-prefix-sums--difference-arrays)
5. [Binary Search (and Binary Search on Answer)](#5-binary-search-and-binary-search-on-answer)
6. [Monotonic Stack / Monotonic Queue](#6-monotonic-stack--monotonic-queue)
7. [Hashing & Frequency Maps](#7-hashing--frequency-maps)
8. [Heap / Priority Queue](#8-heap--priority-queue)
9. [Backtracking](#9-backtracking)
10. [Divide & Conquer](#10-divide--conquer)
11. [Greedy & Interval Scheduling](#11-greedy--interval-scheduling)
12. [Graph Traversal — BFS / DFS](#12-graph-traversal--bfs--dfs)
13. [Shortest Paths & Minimum Spanning Trees](#13-shortest-paths--minimum-spanning-trees)
14. [Union-Find (Disjoint Set Union)](#14-union-find-disjoint-set-union)
15. [Dynamic Programming](#15-dynamic-programming)
16. [Bit Manipulation](#16-bit-manipulation)

---

## 1. Two Pointers

**When to reach for it**
- Input is a sorted array or can be sorted cheaply.
- You're searching for a pair, triplet, or partition that satisfies a relation (sum, difference, squared sum…).
- You need to merge or compare two sequences in linear time.

**Key insight**
Movement of each pointer is *monotonic* (only forward, or only inward). Each step provably discards a whole class of candidate answers, so total work is O(n) instead of O(n²).

**Template**

```python
def two_pointers(arr, target):
    arr.sort()
    left, right = 0, len(arr) - 1
    while left < right:
        s = arr[left] + arr[right]
        if s == target:
            return (left, right)
        if s < target:
            left += 1        # need a bigger sum
        else:
            right -= 1       # need a smaller sum
    return None
```

**Canonical example — 3Sum**

Find all unique triplets that sum to 0.

```python
def three_sum(nums):
    nums.sort()
    result = []
    n = len(nums)
    for i in range(n - 2):
        if nums[i] > 0:
            break                                 # no positive triple can sum to 0
        if i > 0 and nums[i] == nums[i - 1]:
            continue                              # skip duplicate anchors
        left, right = i + 1, n - 1
        while left < right:
            s = nums[i] + nums[left] + nums[right]
            if s == 0:
                result.append([nums[i], nums[left], nums[right]])
                left += 1
                right -= 1
                while left < right and nums[left] == nums[left - 1]:
                    left += 1
                while left < right and nums[right] == nums[right + 1]:
                    right -= 1
            elif s < 0:
                left += 1
            else:
                right -= 1
    return result
```

**Complexity** — Time O(n²), Space O(1) extra (ignoring output).

**Common pitfalls**
- Forgetting to de-duplicate both anchor and inner pointers.
- Using two pointers on *unsorted* data without justification.
- Moving only one pointer when a match is found (infinite loops on duplicates).

**Related** — Sliding Window, Fast & Slow Pointers, Binary Search.

---

## 2. Sliding Window

**When to reach for it**
- Asking for the longest/shortest/best contiguous subarray or substring.
- Constraint is on a running property of the window: sum, distinct-count, frequency cap, product.
- Input is an array or string, not an arbitrary set.

**Key insight**
Maintain a window `[left, right]` and a running state. When the state violates the constraint, shrink from the left until it's valid again. Both pointers move at most n times → O(n) total.

**Template — variable-size window**

```python
def longest_valid_window(arr, is_valid):
    left = 0
    best = 0
    state = {}                                    # whatever you need to track
    for right, x in enumerate(arr):
        add_to_state(state, x)
        while not is_valid(state):
            remove_from_state(state, arr[left])
            left += 1
        best = max(best, right - left + 1)
    return best
```

**Canonical example — Longest substring without repeating characters**

```python
def length_of_longest_substring(s):
    last_seen = {}
    left = 0
    best = 0
    for right, ch in enumerate(s):
        if ch in last_seen and last_seen[ch] >= left:
            left = last_seen[ch] + 1              # jump past the previous occurrence
        last_seen[ch] = right
        best = max(best, right - left + 1)
    return best
```

**Complexity** — Time O(n), Space O(k) where k = alphabet/distinct elements.

**Common pitfalls**
- Using fixed-window logic for a variable-window problem (or vice-versa).
- Off-by-one on window length: `right - left + 1` vs `right - left`.
- Forgetting to update running state on *both* expansion and contraction.

**Related** — Two Pointers, Hashing & Frequency Maps, Monotonic Queue (for sliding-window max).

---

## 3. Fast & Slow Pointers

**When to reach for it**
- Linked-list problems: cycle detection, cycle entry, middle node, palindrome.
- Sequence-function cycle detection (Floyd, Brent) — e.g., "Find the duplicate number" with O(1) space.

**Key insight**
Two pointers moving at different speeds on the same cyclic structure must eventually meet, and the meeting point has algebraic properties you can exploit (Floyd's cycle-finding theorem).

**Template**

```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False
```

**Canonical example — Start of the cycle (Linked List Cycle II)**

```python
def detect_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            # Phase 2: reset one pointer; both move 1 step.
            entry = head
            while entry is not slow:
                entry = entry.next
                slow = slow.next
            return entry
    return None
```

**Complexity** — Time O(n), Space O(1).

**Common pitfalls**
- Not checking `fast` *and* `fast.next` before advancing two steps (NPE on odd-length lists).
- Confusing "meeting point" with "cycle start" — they're different nodes; the second-phase reset is essential.

**Related** — Two Pointers, Graph Traversal (functional graphs).

---

## 4. Prefix Sums & Difference Arrays

**When to reach for it**
- Many range-sum queries on a *static* array → prefix sums.
- Many range-update operations followed by a single read pass → difference array.
- 2D grid sum queries on a static grid → 2D prefix.
- "Subarray sum equals k" style problems — combine prefix sums with a hash map.

**Key insight**
Precompute cumulative sums once so each range query becomes O(1) subtraction. Difference arrays flip the duality: O(1) range update, O(n) final read.

**Template — 1D prefix sum**

```python
def build_prefix(arr):
    prefix = [0] * (len(arr) + 1)
    for i, x in enumerate(arr):
        prefix[i + 1] = prefix[i] + x
    return prefix

def range_sum(prefix, l, r):                      # inclusive [l, r]
    return prefix[r + 1] - prefix[l]
```

**Canonical example — Subarray sum equals k**

```python
from collections import defaultdict

def subarray_sum(nums, k):
    count = 0
    running = 0
    seen = defaultdict(int)
    seen[0] = 1                                   # empty prefix
    for x in nums:
        running += x
        count += seen[running - k]                # any earlier prefix whose value = running - k
        seen[running] += 1
    return count
```

**Complexity** — Build O(n), query O(1). Space O(n).

**Common pitfalls**
- Off-by-one on prefix indexing — keep `prefix[0] = 0` and use `prefix[i+1] - prefix[l]`.
- Forgetting to seed `seen[0] = 1` in the hash-map variant (misses subarrays starting at index 0).
- Using prefix sums on a mutable array without a segment tree / BIT.

**Related** — Hashing & Frequency Maps, Segment Tree / BIT (for dynamic versions), Sliding Window.

---

## 5. Binary Search (and Binary Search on Answer)

**When to reach for it**
- Sorted array search (classic) or a search space with a *monotonic predicate* (advanced).
- "Minimum X such that predicate(X) is true" / "largest feasible value" optimization problems.
- Rotated sorted arrays, search in a 2D matrix, first/last occurrence.

**Key insight**
If `feasible(x)` is monotonic — false, false, …, false, true, true, … — you can find the boundary in O(log range) regardless of whether the search space is an array index or a real-valued answer.

**Template — lower_bound (smallest index whose value ≥ target)**

```python
def lower_bound(arr, target):
    lo, hi = 0, len(arr)                          # half-open [lo, hi)
    while lo < hi:
        mid = (lo + hi) // 2
        if arr[mid] < target:
            lo = mid + 1
        else:
            hi = mid
    return lo                                     # may equal len(arr) if no such element
```

**Template — binary search on answer**

```python
def min_feasible(lo, hi, feasible):
    while lo < hi:
        mid = (lo + hi) // 2
        if feasible(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo
```

**Canonical example — Koko Eating Bananas**

Find the minimum eating speed `k` to finish all piles within `h` hours.

```python
from math import ceil

def min_eating_speed(piles, h):
    def feasible(k):
        return sum(ceil(p / k) for p in piles) <= h
    lo, hi = 1, max(piles)
    while lo < hi:
        mid = (lo + hi) // 2
        if feasible(mid):
            hi = mid
        else:
            lo = mid + 1
    return lo
```

**Complexity** — Time O(n log max_answer), Space O(1).

**Common pitfalls**
- Integer overflow on `(lo + hi) // 2` in languages with fixed-width ints — use `lo + (hi - lo) // 2`.
- Mixing half-open and closed intervals — pick one style and stick with it.
- Non-monotonic predicate → binary search silently returns garbage; verify monotonicity first.

**Related** — Two Pointers, Divide & Conquer, Greedy.

---

## 6. Monotonic Stack / Monotonic Queue

**When to reach for it**
- "Next/previous greater/smaller element" queries.
- Sliding-window maximum / minimum in O(n).
- Histogram problems (largest rectangle, trapping rain water).
- Stock span, daily temperatures, maximum of subarray mins.

**Key insight**
Maintain a stack/deque whose elements are in monotonic order. When a new element breaks the order, pop until it's restored — each popped element has found its "answer." Every element is pushed and popped at most once → O(n).

**Template — next greater element**

```python
def next_greater(arr):
    n = len(arr)
    result = [-1] * n
    stack = []                                    # stack of indices, values decreasing
    for i, x in enumerate(arr):
        while stack and arr[stack[-1]] < x:
            result[stack.pop()] = x
        stack.append(i)
    return result
```

**Canonical example — Largest rectangle in histogram**

```python
def largest_rectangle(heights):
    stack = []                                    # indices of increasing heights
    best = 0
    heights = heights + [0]                       # sentinel flushes the stack at the end
    for i, h in enumerate(heights):
        while stack and heights[stack[-1]] > h:
            top = stack.pop()
            left = stack[-1] if stack else -1
            width = i - left - 1
            best = max(best, heights[top] * width)
        stack.append(i)
    return best
```

**Complexity** — Time O(n), Space O(n).

**Common pitfalls**
- Storing values instead of indices — you usually need the index to compute distances.
- Forgetting the sentinel element at the end; the stack holds unresolved candidates.
- Using a monotonic **stack** when a monotonic **deque** is needed (sliding window requires pops from both ends).

**Related** — Sliding Window, Two Pointers, Divide & Conquer.

---

## 7. Hashing & Frequency Maps

**When to reach for it**
- Constant-time lookup, insertion, deletion.
- Counting occurrences, grouping anagrams, deduplication.
- Complements (`target - x`) in two-sum-style problems.
- Caching / memoization keyed on arbitrary values.

**Key insight**
Trade O(n) extra space for amortized O(1) operations. When paired with another pattern (prefix sums, sliding window), hashing often turns an O(n²) into O(n).

**Template — group by computed key**

```python
from collections import defaultdict

def group_by_key(items, key_fn):
    buckets = defaultdict(list)
    for x in items:
        buckets[key_fn(x)].append(x)
    return buckets
```

**Canonical example — Group anagrams**

```python
from collections import defaultdict

def group_anagrams(strs):
    buckets = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))                    # or a 26-length tuple of counts for O(n*k)
        buckets[key].append(s)
    return list(buckets.values())
```

**Complexity** — Depends on key construction. Sorted-string keys: O(n · k log k). Count-tuple keys: O(n · k).

**Common pitfalls**
- Using a mutable object (list/dict) as a dict key — Python requires hashable keys; convert to tuple/frozenset.
- Worst-case collision attacks on hash maps can degrade to O(n) per operation — rarely matters in interviews, but know it.
- Iterating a dict while mutating it — snapshot keys first: `for k in list(d):`.

**Related** — Sliding Window, Prefix Sums, Two Pointers.

---

## 8. Heap / Priority Queue

**When to reach for it**
- Repeatedly extract the smallest/largest element of a changing set.
- Top-K problems, k-way merge, scheduling, Dijkstra.
- Streaming median (two heaps).

**Key insight**
Binary heap gives O(log n) push/pop, O(1) peek. For top-K with K ≪ N, maintain a *bounded* min-heap of size K rather than sorting everything.

**Template — top-K largest**

```python
import heapq

def top_k_largest(nums, k):
    heap = []                                     # min-heap of size k
    for x in nums:
        if len(heap) < k:
            heapq.heappush(heap, x)
        elif x > heap[0]:
            heapq.heapreplace(heap, x)
    return heap                                   # contains the k largest in arbitrary order
```

**Canonical example — Merge K sorted lists**

```python
import heapq

def merge_k_lists(lists):
    heap = []
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst[0], i, 0))  # (value, list_index, element_index)
    merged = []
    while heap:
        val, li, ei = heapq.heappop(heap)
        merged.append(val)
        if ei + 1 < len(lists[li]):
            heapq.heappush(heap, (lists[li][ei + 1], li, ei + 1))
    return merged
```

**Complexity** — Top-K: O(n log k). K-way merge of N total elements across K lists: O(N log K).

**Common pitfalls**
- Python's `heapq` is a **min-heap** only. For a max-heap, push negated values or wrap in a comparator class.
- Tuple comparison falls through to later elements — adding a tiebreaker index avoids comparing non-orderable payloads.
- "Stale" heap entries after updates (common in Dijkstra): lazily skip popped entries that no longer match current distance.

**Related** — Shortest Paths (Dijkstra), Greedy, Sorting.

---

## 9. Backtracking

**When to reach for it**
- Enumerate all solutions: permutations, combinations, subsets, partitions.
- Constraint satisfaction: N-queens, Sudoku, word-search, graph coloring.
- Problem size is small (n ≤ ~20) and brute-force with pruning is fine.

**Key insight**
Depth-first exploration of a decision tree. *Choose → recurse → unchoose.* Prune aggressively whenever a partial solution can't lead to a valid final one.

**Template**

```python
def backtrack(path, choices, result):
    if is_solution(path):
        result.append(path.copy())
        return
    for c in choices:
        if not is_valid(path, c):
            continue                              # prune
        path.append(c)
        backtrack(path, next_choices(choices, c), result)
        path.pop()
```

**Canonical example — Permutations**

```python
def permutations(nums):
    result = []
    used = [False] * len(nums)
    path = []

    def dfs():
        if len(path) == len(nums):
            result.append(path.copy())
            return
        for i, x in enumerate(nums):
            if used[i]:
                continue
            used[i] = True
            path.append(x)
            dfs()
            path.pop()
            used[i] = False

    dfs()
    return result
```

**Complexity** — Typically O(branches^depth) in the worst case; pruning is what makes it practical. Permutations: O(n · n!).

**Common pitfalls**
- Appending `path` (reference) to results instead of `path.copy()` — all results end up identical (the final state).
- Forgetting to undo the choice after recursion.
- Missing a prune opportunity — always ask "can this partial solution still win?"
- Mutating the input; prefer explicit state (bitmask, used array) over in-place swaps if it doesn't hurt perf.

**Related** — DFS, Dynamic Programming (with memoization on overlapping subproblems), Bit Manipulation (for `used` masks).

---

## 10. Divide & Conquer

**When to reach for it**
- Problem naturally splits into independent halves (sort, search, closest pair).
- Parallelizable / recursive structure.
- Inversions counting, matrix multiplication (Strassen), FFT.

**Key insight**
Divide into independent subproblems, solve recursively, combine. The *combine* step is where the cleverness usually lives (think merge-step of merge sort).

**Template**

```python
def divide_and_conquer(arr, lo, hi):
    if hi - lo <= 1:
        return base_case(arr, lo, hi)
    mid = (lo + hi) // 2
    left = divide_and_conquer(arr, lo, mid)
    right = divide_and_conquer(arr, mid, hi)
    return combine(left, right)
```

**Canonical example — Count inversions (merge sort variant)**

```python
def count_inversions(arr):
    def sort_count(a):
        if len(a) <= 1:
            return a, 0
        mid = len(a) // 2
        left, inv_l = sort_count(a[:mid])
        right, inv_r = sort_count(a[mid:])
        merged, inv_m = merge(left, right)
        return merged, inv_l + inv_r + inv_m

    def merge(left, right):
        merged, inv, i, j = [], 0, 0, 0
        while i < len(left) and j < len(right):
            if left[i] <= right[j]:
                merged.append(left[i]); i += 1
            else:
                merged.append(right[j]); j += 1
                inv += len(left) - i              # every remaining left element is > right[j]
        merged.extend(left[i:]); merged.extend(right[j:])
        return merged, inv

    return sort_count(arr)[1]
```

**Complexity** — Merge-sort style: O(n log n) time, O(n) space. Generally given by Master theorem.

**Common pitfalls**
- Copying slices (`arr[:mid]`) in Python is O(n) — pass indices instead if hot path.
- Recursion depth on n=10⁶ can hit Python's 1000-frame default limit; raise with `sys.setrecursionlimit` or iterate.
- Ignoring the non-recursion base case overhead (lots of tiny merges is slower than one `sorted()` in CPython).

**Related** — Binary Search, Dynamic Programming, Sorting.

---

## 11. Greedy & Interval Scheduling

**When to reach for it**
- Locally optimal choice provably leads to a globally optimal solution.
- Scheduling, coin change (with canonical denominations), Huffman coding, activity selection.
- Sweep-line problems — sort events by time, process in order.

**Key insight**
Prove (or be told) the **greedy-choice property** and **optimal substructure**. Without a proof, greedy is a gamble — often wrong (e.g., generalized coin change).

**Template — interval scheduling by earliest end time**

```python
def max_non_overlapping(intervals):
    intervals.sort(key=lambda x: x[1])            # sort by end
    count = 0
    end = float("-inf")
    for s, e in intervals:
        if s >= end:
            count += 1
            end = e
    return count
```

**Canonical example — Minimum number of meeting rooms**

```python
import heapq

def min_meeting_rooms(intervals):
    intervals.sort(key=lambda x: x[0])            # sort by start
    rooms = []                                    # min-heap of end times
    for s, e in intervals:
        if rooms and rooms[0] <= s:
            heapq.heapreplace(rooms, e)           # reuse the earliest-freed room
        else:
            heapq.heappush(rooms, e)
    return len(rooms)
```

**Complexity** — O(n log n) for sorting; sweep line is O(n) after sort.

**Common pitfalls**
- Applying greedy without proof — always check a counterexample.
- Sorting by the wrong key (start vs end vs length).
- Edge-touching intervals: decide whether `[1,3]` and `[3,5]` overlap; constraints vary by problem.

**Related** — Heap / Priority Queue, Sorting, Dynamic Programming (when greedy fails).

---

## 12. Graph Traversal — BFS / DFS

**When to reach for it**
- Connectivity, reachability, components.
- Shortest path in **unweighted** graphs → BFS.
- Topological ordering, cycle detection → DFS.
- Flood fill, bipartite check, maze shortest path.

**Key insight**
BFS explores by distance (queue, FIFO). DFS explores by depth (stack or recursion). Both are O(V + E) with an adjacency list and a visited set.

**Template — BFS on a grid**

```python
from collections import deque

def bfs_grid(grid, start):
    rows, cols = len(grid), len(grid[0])
    visited = {start}
    queue = deque([(start, 0)])                   # (position, distance)
    while queue:
        (r, c), d = queue.popleft()
        for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and (nr, nc) not in visited and grid[nr][nc] != "#":
                visited.add((nr, nc))
                queue.append(((nr, nc), d + 1))
    return visited
```

**Canonical example — Topological sort (Kahn's algorithm)**

```python
from collections import deque, defaultdict

def topo_sort(n, edges):
    graph = defaultdict(list)
    in_degree = [0] * n
    for u, v in edges:
        graph[u].append(v)
        in_degree[v] += 1
    queue = deque(i for i in range(n) if in_degree[i] == 0)
    order = []
    while queue:
        u = queue.popleft()
        order.append(u)
        for v in graph[u]:
            in_degree[v] -= 1
            if in_degree[v] == 0:
                queue.append(v)
    return order if len(order) == n else []       # empty → cycle
</pre>
```

**Complexity** — O(V + E) time and space.

**Common pitfalls**
- Marking visited *when popping* instead of *when enqueuing* — same node enqueued many times in BFS.
- DFS recursion blowing the stack on a 10⁵-node path graph.
- Treating an undirected edge as one directed edge (forgetting the reverse).
- Using BFS for shortest paths on a **weighted** graph — you need Dijkstra.

**Related** — Union-Find, Shortest Paths, DP on DAGs.

---

## 13. Shortest Paths & Minimum Spanning Trees

**When to reach for it**
- Weighted graph, need shortest distance(s) → Dijkstra (non-negative weights), Bellman-Ford (handles negatives, detects negative cycles), Floyd-Warshall (all-pairs, small V), 0-1 BFS (weights ∈ {0,1}).
- Connect all nodes at minimum cost → Kruskal or Prim (MST).

**Key insight**
- **Dijkstra**: greedy over a min-heap, relies on non-negativity so popped distances are final.
- **Bellman-Ford**: relax every edge V−1 times; a V-th relaxation proves a negative cycle.
- **Kruskal**: sort edges, add the smallest that doesn't form a cycle (uses union-find).
- **Prim**: grow a tree by repeatedly adding the cheapest crossing edge (uses a heap).

**Template — Dijkstra**

```python
import heapq
from collections import defaultdict

def dijkstra(n, edges, src):
    graph = defaultdict(list)
    for u, v, w in edges:
        graph[u].append((v, w))
    dist = [float("inf")] * n
    dist[src] = 0
    heap = [(0, src)]
    while heap:
        d, u = heapq.heappop(heap)
        if d > dist[u]:
            continue                              # stale entry
        for v, w in graph[u]:
            nd = d + w
            if nd < dist[v]:
                dist[v] = nd
                heapq.heappush(heap, (nd, v))
    return dist
```

**Canonical example — Kruskal's MST**

```python
def kruskal(n, edges):
    parent = list(range(n))
    def find(x):
        while parent[x] != x:
            parent[x] = parent[parent[x]]         # path compression
            x = parent[x]
        return x
    def union(a, b):
        ra, rb = find(a), find(b)
        if ra == rb: return False
        parent[ra] = rb
        return True
    total = 0
    for w, u, v in sorted(edges, key=lambda e: e[0]):
        if union(u, v):
            total += w
    return total
```

**Complexity**
- Dijkstra with binary heap: O((V + E) log V).
- Bellman-Ford: O(V · E).
- Floyd-Warshall: O(V³).
- Kruskal: O(E log E); Prim with heap: O((V + E) log V).

**Common pitfalls**
- Running Dijkstra on a graph with negative edges — silent wrong answer.
- Not skipping stale heap entries (`if d > dist[u]: continue`).
- Forgetting that MST is undefined on a disconnected graph (→ minimum spanning forest).

**Related** — Union-Find, Heap / Priority Queue, BFS.

---

## 14. Union-Find (Disjoint Set Union)

**When to reach for it**
- Maintaining dynamic connectivity: "are u and v in the same component?"
- Kruskal's MST, offline connectivity queries, detecting cycles in an undirected graph.
- Grouping equivalences (e.g., accounts merge, redundant-connection problems).

**Key insight**
With *path compression* + *union by rank/size*, amortized cost per operation is essentially O(α(n)) — inverse Ackermann, effectively constant.

**Template**

```python
class DSU:
    def __init__(self, n):
        self.parent = list(range(n))
        self.size = [1] * n

    def find(self, x):
        while self.parent[x] != x:
            self.parent[x] = self.parent[self.parent[x]]   # path compression (halving)
            x = self.parent[x]
        return x

    def union(self, a, b):
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return False
        if self.size[ra] < self.size[rb]:
            ra, rb = rb, ra
        self.parent[rb] = ra
        self.size[ra] += self.size[rb]
        return True
```

**Canonical example — Number of connected components**

```python
def count_components(n, edges):
    dsu = DSU(n)
    components = n
    for u, v in edges:
        if dsu.union(u, v):
            components -= 1
    return components
```

**Complexity** — Near O(1) amortized per op. Total O((V + E) · α(V)).

**Common pitfalls**
- Omitting union-by-size/rank — degrades to O(log n) or worse per op without it.
- Recursive `find` on huge inputs — iterative path compression is safer in Python.
- Forgetting that DSU is for **undirected** connectivity; directed reachability needs different tools (SCC).

**Related** — Graph Traversal, Shortest Paths & MST, Offline Query Processing.

---

## 15. Dynamic Programming

Dynamic programming is a **family** of patterns rather than a single technique. The common thread: solutions to subproblems overlap and can be cached. Design steps:

1. **Define the state** — what exactly does `dp[i]` (or `dp[i][j]`, …) represent?
2. **Write the transition** — how is `dp[i]` built from smaller states?
3. **Identify the base cases**.
4. **Pick order**: top-down memoization (recursive + cache) or bottom-up tabulation.
5. **Optimize space** when only the last row/column of the table is needed.

### 15.1 Linear (1D) DP

Canonical problems: climbing stairs, house robber, maximum subarray (Kadane's).

```python
def rob(nums):
    prev, curr = 0, 0                             # dp[i-2], dp[i-1]
    for x in nums:
        prev, curr = curr, max(curr, prev + x)
    return curr
```

**Complexity** — O(n) time, O(1) space after rolling-variable optimization.

### 15.2 Knapsack Family

- **0/1 knapsack**: each item used at most once. Iterate weights in *reverse* in the 1D form.
- **Unbounded knapsack**: unlimited copies. Iterate weights *forward*.

```python
def knapsack_01(weights, values, capacity):
    dp = [0] * (capacity + 1)
    for w, v in zip(weights, values):
        for c in range(capacity, w - 1, -1):      # reverse: each item once
            dp[c] = max(dp[c], dp[c - w] + v)
    return dp[capacity]
```

**Complexity** — O(n · capacity).

### 15.3 LIS / LCS

**Longest Increasing Subsequence — O(n log n) patience sort variant**

```python
from bisect import bisect_left

def lis(nums):
    tails = []
    for x in nums:
        i = bisect_left(tails, x)
        if i == len(tails):
            tails.append(x)
        else:
            tails[i] = x
    return len(tails)
```

**Longest Common Subsequence — O(n·m) DP**

```python
def lcs(a, b):
    n, m = len(a), len(b)
    dp = [[0] * (m + 1) for _ in range(n + 1)]
    for i in range(n):
        for j in range(m):
            if a[i] == b[j]:
                dp[i + 1][j + 1] = dp[i][j] + 1
            else:
                dp[i + 1][j + 1] = max(dp[i][j + 1], dp[i + 1][j])
    return dp[n][m]
```

### 15.4 Interval DP

Problems where `dp[i][j]` depends on a split point `k` inside `[i, j]` — matrix chain multiplication, burst balloons, palindrome partitioning.

```python
def matrix_chain(dims):
    n = len(dims) - 1
    dp = [[0] * n for _ in range(n)]
    for length in range(2, n + 1):
        for i in range(n - length + 1):
            j = i + length - 1
            dp[i][j] = float("inf")
            for k in range(i, j):
                dp[i][j] = min(dp[i][j], dp[i][k] + dp[k + 1][j] + dims[i] * dims[k + 1] * dims[j + 1])
    return dp[0][n - 1]
```

**Complexity** — O(n³).

### 15.5 Tree DP

Root the tree; compute per-node states from children. Classic: house robber III, diameter of a tree.

```python
def rob_tree(root):
    def dfs(node):
        if not node: return (0, 0)                # (rob, skip)
        lr, ls = dfs(node.left)
        rr, rs = dfs(node.right)
        rob_here = node.val + ls + rs
        skip_here = max(lr, ls) + max(rr, rs)
        return (rob_here, skip_here)
    return max(dfs(root))
```

### 15.6 Bitmask DP

State compresses a subset of ≤ ~20 items into an integer mask. Classics: travelling salesman (Held–Karp), assignment problem.

```python
def tsp(dist):
    n = len(dist)
    INF = float("inf")
    dp = [[INF] * n for _ in range(1 << n)]
    dp[1][0] = 0                                  # start at city 0
    for mask in range(1 << n):
        for u in range(n):
            if not (mask >> u) & 1: continue
            if dp[mask][u] == INF: continue
            for v in range(n):
                if (mask >> v) & 1: continue
                nmask = mask | (1 << v)
                cand = dp[mask][u] + dist[u][v]
                if cand < dp[nmask][v]:
                    dp[nmask][v] = cand
    full = (1 << n) - 1
    return min(dp[full][v] + dist[v][0] for v in range(1, n))
```

**Complexity** — O(2ⁿ · n²) time, O(2ⁿ · n) space.

### Common DP pitfalls

- **Wrong state**: the first DP you write almost always has an incorrect or incomplete state. Re-verify `dp[i]` really answers the subproblem.
- **Transition order** for knapsack (reverse for 0/1, forward for unbounded).
- **Base cases** often off: `dp[0][0] = 1` for "empty subset, sum zero" style combinatorics.
- **Memoization keys** that aren't hashable — convert lists to tuples.
- **Space blow-up**: rolling arrays drop a dimension when only the previous layer matters.

**Related** — Divide & Conquer, Greedy, Backtracking (unmemoized DP), Graph Traversal (DP on DAGs).

---

## 16. Bit Manipulation

**When to reach for it**
- Working with subsets of a small universe (n ≤ 20–30) via bitmasks.
- XOR tricks: finding the unique element, swap-without-temp, parity.
- Low-level optimizations, packed state, set operations in O(1).

**Key insight**
Integers *are* bit vectors — `|` is union, `&` is intersection, `^` is symmetric difference, `~` is complement, `>> / <<` are shifts. For subset enumeration, iterate `mask` over `[0, 2ⁿ)`.

**Essential tricks**

```python
x & (x - 1)        # clear the lowest set bit
x & -x             # isolate the lowest set bit
bin(x).count("1")  # popcount (use int.bit_count() in Python 3.10+)
x ^ y              # differing bits
(1 << n) - 1       # mask covering n low bits
```

**Canonical example — Single number (everyone appears twice except one)**

```python
def single_number(nums):
    result = 0
    for x in nums:
        result ^= x                               # duplicates cancel
    return result
```

**Enumerate all subsets of a mask** (used in subset-DP optimizations):

```python
def subsets_of(mask):
    sub = mask
    while sub:
        yield sub
        sub = (sub - 1) & mask
    yield 0                                       # don't forget the empty subset
```

**Complexity** — Subset enumeration over a universe of size n: O(3ⁿ) total across all masks (not 4ⁿ), which is why bitmask DP is feasible.

**Common pitfalls**
- Operator precedence: `x & 1 == 0` parses as `x & (1 == 0)`. Parenthesize.
- Sign issues with `~` on fixed-width integers in C/C++/Java (Python ints are arbitrary precision — different gotcha).
- Forgetting to include the empty subset in subset enumerations.
- Off-by-one in bit positions (0-indexed from the LSB).

**Related** — Dynamic Programming (bitmask DP), Hashing (bitset as set), Backtracking (used-mask).

---

## Pattern selection cheatsheet

| Problem signal | First pattern to try |
|---|---|
| Sorted array, pair/triplet target | Two Pointers |
| Longest/shortest contiguous subarray with a constraint | Sliding Window |
| Linked-list cycle / middle / palindrome | Fast & Slow Pointers |
| Many range-sum queries on static data | Prefix Sums |
| "Minimum X such that…" on a monotonic predicate | Binary Search on Answer |
| "Next greater/smaller" / histogram / sliding-window max | Monotonic Stack/Queue |
| Top-K / streaming median / k-way merge | Heap |
| Enumerate all combinations/permutations with constraints | Backtracking |
| Independent halves recombinable in a clever merge | Divide & Conquer |
| Interval overlap / scheduling / sweep | Greedy + sort |
| Reachability / shortest path in unweighted graph | BFS |
| Cycles, topo order, components | DFS / Union-Find |
| Weighted shortest path, non-negative weights | Dijkstra |
| Connect everything at minimum cost | Kruskal / Prim |
| Overlapping subproblems, optimal substructure | Dynamic Programming |
| Small universe subsets / XOR tricks | Bit Manipulation |

---

## Where to go from here

This file is the overview. Each pattern deserves a focused follow-up note (e.g. `DSA/sliding-window.md`, `DSA/binary-search-patterns.md`, `DSA/dp-knapsack.md`) with:

- A reusable template.
- 2–3 canonical annotated solutions.
- A "when to reach for this" checklist.
- Cross-links back to related patterns here.

Work through problems *pattern-by-pattern* rather than at random — pattern recognition is the single highest-leverage skill in interview prep and competitive programming.
