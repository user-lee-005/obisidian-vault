# 🧠 Pattern Identification Framework

> The **MOST IMPORTANT** interview skill: identifying the right pattern before coding.

---

## Decision Flowchart

When the interviewer gives a problem, ask these questions **in order**:

```
Step 1 → "Is this about a contiguous subarray/substring?"
           YES → Sliding Window

Step 2 → "Do I need repeated fast lookup / counting?"
           YES → HashMap / Set

Step 3 → "Is the input sorted (or can I sort it)?"
           YES → Two Pointers / Binary Search

Step 4 → "Do I need the nearest greater/smaller element?"
           YES → Stack (Monotonic)

Step 5 → "Is there a hierarchy or tree traversal?"
           YES → Tree DFS / BFS

Step 6 → "Are there dependencies or connections?"
           YES → Graph (BFS / DFS / Topological Sort)

Step 7 → "Do I need the top/smallest/largest K repeatedly?"
           YES → Heap / Priority Queue

Step 8 → "Do I need to generate all valid possibilities?"
           YES → Backtracking
```

---

## Quick Reference Table

| Signal | Pattern | Time Complexity |
|--------|---------|----------------|
| Contiguous range | [[Sliding Window]] | O(n) |
| Lookup / count | [[HashMap & Frequency Counting]] | O(n) |
| Sorted input + pairs | [[Two Pointers]] | O(n) |
| Min/max answer space | [[Binary Search]] | O(n log n) |
| Next greater/smaller | [[Stack & Monotonic Stack]] | O(n) |
| Hierarchy / tree | [[Trees - BFS & DFS]] | O(n) |
| Dependencies / grid | [[Graphs]] | O(V + E) |
| Top K / streaming | [[Heap & Priority Queue]] | O(n log k) |
| All possibilities | [[Backtracking]] | O(2ⁿ) or O(n!) |
| Linked list cycle | [[Fast & Slow Pointers]] | O(n) |

---

## 🎤 How to Communicate in Interviews

### Step 1: Understand
- Repeat the problem in your own words
- Clarify edge cases and constraints
- Ask about input size (determines acceptable time complexity)

### Step 2: Brute Force
> *"The brute force would be... which is O(n²). The bottleneck is..."*

### Step 3: Identify Pattern
> *"Since we're looking at contiguous subarrays, this is a sliding window problem."*

### Step 4: Optimize
> *"By maintaining a window, we avoid recomputing overlapping ranges, bringing it to O(n)."*

### Step 5: Code
- Write clean, readable code
- Use meaningful variable names
- Handle edge cases

### Step 6: Test
- Trace through with a small example
- Check edge cases (empty input, single element, all same)

---

## 🎯 Complexity Targets by Input Size

| Input Size (n) | Target Complexity | Patterns |
|----------------|-------------------|----------|
| n ≤ 20 | O(2ⁿ), O(n!) | Backtracking |
| n ≤ 1000 | O(n²) | Brute force OK |
| n ≤ 10⁵ | O(n log n) | Sort + Binary Search |
| n ≤ 10⁶ | O(n) | HashMap, Sliding Window, Two Pointers |
| n ≤ 10⁹ | O(log n) | Binary Search on answer |

---

**← Back to [[DSA Patterns]]**
