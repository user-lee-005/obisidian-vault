# 4. Fast & Slow Pointers (Floyd's Tortoise & Hare)

> **Core Idea:** Two pointers moving at different speeds detect cycles and find midpoints.

---

## 🔍 Recognition Signals

- Linked list cycle detection
- Finding the middle node
- Repeated movement / convergence

---

## 📝 Must-Do Problems

| Problem | Difficulty | LeetCode |
|---------|-----------|----------|
| Linked List Cycle | Easy | [#141](https://leetcode.com/problems/linked-list-cycle/) |
| Linked List Cycle II (find start) | Medium | [#142](https://leetcode.com/problems/linked-list-cycle-ii/) |
| Middle of the Linked List | Easy | [#876](https://leetcode.com/problems/middle-of-the-linked-list/) |
| Happy Number | Easy | [#202](https://leetcode.com/problems/happy-number/) |

---

## 🧩 Templates (Java)

### Cycle Detection

```java
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow == fast) return true; // cycle exists
}
return false;
```

### Find Middle Node

```java
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next;
    fast = fast.next.next;
}
return slow; // middle node
```

---

## 💡 Key Insight

- Fast pointer moves **2x** speed of slow.
- If there's a cycle, they **must meet**.
- To find cycle start: reset one pointer to head, move both at **1x** speed — they meet at cycle start.

---

## 🔗 Related Patterns

- [[Two Pointers]] — generalized version for arrays

---

**← Back to [[DSA Patterns]]**
