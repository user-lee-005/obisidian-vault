# 2. Sliding Window

> 🔥🔥 **MOST IMPORTANT pattern.**
>
> **Core Idea:** Avoid recalculating entire ranges repeatedly.

---

## 🔍 Recognition Signals

- Substring / subarray
- Contiguous sequence
- Longest / shortest window
- "At most K"
- Fixed-size window

## 🏷️ Keywords in Problem Statement

- "longest substring"
- "continuous"
- "window"
- "subarray"
- "consecutive"

---

## 📐 Two Types

### Fixed Window
- Window size is **known** upfront.
- Example: Max sum of subarray of size K.

### Dynamic Window
- **Expand/shrink** based on a condition.
- Example: Longest substring without repeating characters.

---

## 📝 Must-Do Problems

| Problem | Difficulty | Type | LeetCode |
|---------|-----------|------|----------|
| Longest Substring Without Repeating Characters | Medium | Dynamic | [#3](https://leetcode.com/problems/longest-substring-without-repeating-characters/) |
| Permutation in String | Medium | Fixed | [#567](https://leetcode.com/problems/permutation-in-string/) |
| Minimum Size Subarray Sum | Medium | Dynamic | [#209](https://leetcode.com/problems/minimum-size-subarray-sum/) |
| Minimum Window Substring | Hard | Dynamic | [#76](https://leetcode.com/problems/minimum-window-substring/) |
| Max Consecutive Ones III | Medium | Dynamic | [#1004](https://leetcode.com/problems/max-consecutive-ones-iii/) |

---

## 💡 Interview Signal

> [!important] If brute force involves **nested loops over contiguous ranges** → think **sliding window**.

### Optimization Story (say this in interviews):

> "Brute force recalculates overlapping windows repeatedly, so sliding window reduces redundant computation from O(n²) to O(n)."

---

## 🧩 Templates (Java)

### Fixed Window

```java
// Max sum of subarray of size K
int windowSum = 0;
int maxSum = Integer.MIN_VALUE;

for (int i = 0; i < nums.length; i++) {
    windowSum += nums[i];
    if (i >= k - 1) {
        maxSum = Math.max(maxSum, windowSum);
        windowSum -= nums[i - (k - 1)]; // shrink from left
    }
}
```

### Dynamic Window

```java
// Longest substring without repeating characters
Map<Character, Integer> window = new HashMap<>();
int left = 0, maxLen = 0;

for (int right = 0; right < s.length(); right++) {
    char c = s.charAt(right);
    window.put(c, window.getOrDefault(c, 0) + 1);

    while (window.get(c) > 1) { // shrink condition
        char leftChar = s.charAt(left);
        window.put(leftChar, window.get(leftChar) - 1);
        left++;
    }

    maxLen = Math.max(maxLen, right - left + 1);
}
```

---

## ⚠️ Common Mistakes

1. Forgetting to **shrink** the window (infinite expansion)
2. Off-by-one on window size (`right - left + 1`, not `right - left`)
3. Not handling the **empty string** edge case

---

## 🔗 Related Patterns

- Often combined with [[HashMap & Frequency Counting]] for character counting
- [[Two Pointers]] is a generalization (sliding window = two pointers on contiguous data)

---

**← Back to [[DSA Patterns]]**
