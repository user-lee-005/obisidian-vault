# 6. Binary Search

> **Core Idea:** Halve the search space each step — O(log n).
>
> **Key Insight:** Sometimes binary search is on the **ANSWER space**, not the array.

---

## 🔍 Recognition Signals

- Not just sorted arrays!
- Search space optimization
- Minimizing / maximizing an answer
- "Minimum possible maximum" or "maximum feasible minimum"

## 🏷️ Keywords in Problem Statement

- "minimum possible"
- "maximum feasible"
- "sorted"
- "search"
- "find the smallest/largest that satisfies..."

---

## 📝 Must-Do Problems

| Problem | Difficulty | Type | LeetCode |
|---------|-----------|------|----------|
| Binary Search | Easy | Classic | [#704](https://leetcode.com/problems/binary-search/) |
| Search in Rotated Sorted Array | Medium | Modified | [#33](https://leetcode.com/problems/search-in-rotated-sorted-array/) |
| Koko Eating Bananas | Medium | Answer Space | [#875](https://leetcode.com/problems/koko-eating-bananas/) |
| Capacity To Ship Packages | Medium | Answer Space | [#1011](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/) |
| Find Minimum in Rotated Sorted Array | Medium | Modified | [#153](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/) |

---

## 📐 Two Flavors

### 1. Classic Binary Search (on Array)
Search for a target in sorted data.

### 2. Binary Search on Answer Space
The answer lies in a range `[lo, hi]`. For each candidate answer `mid`, check if it's feasible.

> [!tip] Whenever you see "minimize the maximum" or "maximize the minimum" — think binary search on answer.

---

## 🧩 Templates (Java)

### Classic Binary Search

```java
int lo = 0, hi = nums.length - 1;
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;
    if (nums[mid] == target) return mid;
    else if (nums[mid] < target) lo = mid + 1;
    else hi = mid - 1;
}
return -1;
```

### Binary Search on Answer Space (Koko Eating Bananas)

```java
int lo = 1, hi = maxPile;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (canFinish(piles, mid, h)) {
        hi = mid; // try smaller speed
    } else {
        lo = mid + 1; // need faster
    }
}
return lo;

boolean canFinish(int[] piles, int speed, int h) {
    int hours = 0;
    for (int pile : piles) {
        hours += (pile + speed - 1) / speed; // ceil division
    }
    return hours <= h;
}
```

---

## ⚠️ Common Mistakes

1. **Integer overflow** in `(lo + hi) / 2` → use `lo + (hi - lo) / 2`
2. **Infinite loop** — make sure `lo` or `hi` always changes
3. **Off-by-one** — `lo <= hi` vs `lo < hi` depends on the variant

---

## 🔗 Related Patterns

- [[Two Pointers]] — both work well on sorted data

---

**← Back to [[DSA Patterns]]**
