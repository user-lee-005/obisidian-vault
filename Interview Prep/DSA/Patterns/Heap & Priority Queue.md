# 9. Heap / Priority Queue

> **Core Idea:** Efficiently access the smallest/largest element repeatedly.

---

## 🔍 Recognition Signals

- Top K elements
- Kth smallest / largest
- Streaming data (median, running stats)
- Merging sorted collections

---

## 📝 Must-Do Problems

| Problem | Difficulty | LeetCode |
|---------|-----------|----------|
| Kth Largest Element in an Array | Medium | [#215](https://leetcode.com/problems/kth-largest-element-in-an-array/) |
| Top K Frequent Elements | Medium | [#347](https://leetcode.com/problems/top-k-frequent-elements/) |
| Merge K Sorted Lists | Hard | [#23](https://leetcode.com/problems/merge-k-sorted-lists/) |
| Find Median from Data Stream | Hard | [#295](https://leetcode.com/problems/find-median-from-data-stream/) |

---

## 📐 Min-Heap vs Max-Heap

| Type | Java | Use Case |
|------|------|----------|
| **Min-Heap** | `new PriorityQueue<>()` | Kth largest (keep K largest, min on top) |
| **Max-Heap** | `new PriorityQueue<>(Collections.reverseOrder())` | Kth smallest |

> [!tip] **Top K Trick**
> For "K largest" → use a **min-heap** of size K. The top is the Kth largest.
> For "K smallest" → use a **max-heap** of size K. The top is the Kth smallest.

---

## 🧩 Templates (Java)

### Kth Largest Element

```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
for (int num : nums) {
    minHeap.offer(num);
    if (minHeap.size() > k) {
        minHeap.poll(); // remove smallest
    }
}
return minHeap.peek(); // Kth largest
```

### Merge K Sorted Lists

```java
PriorityQueue<ListNode> pq = new PriorityQueue<>(
    (a, b) -> a.val - b.val
);
for (ListNode list : lists) {
    if (list != null) pq.offer(list);
}

ListNode dummy = new ListNode(0);
ListNode curr = dummy;
while (!pq.isEmpty()) {
    ListNode node = pq.poll();
    curr.next = node;
    curr = curr.next;
    if (node.next != null) pq.offer(node.next);
}
return dummy.next;
```

---

## 💡 Complexity

| Operation | Time |
|-----------|------|
| Insert | O(log n) |
| Remove min/max | O(log n) |
| Peek min/max | O(1) |

---

## 🔗 Related Patterns

- [[HashMap & Frequency Counting]] — count first, then heap for Top K
- [[Binary Search]] — alternative for some "Kth" queries on sorted data

---

**← Back to [[DSA Patterns]]**
