# 3. Two Pointers

> **Core Idea:** Use left/right pointer movement intelligently to avoid O(n²).

---

## 🔍 Recognition Signals

- Sorted arrays
- Pair sums
- Palindrome checks
- In-place modification

## 🏷️ Keywords in Problem Statement

- "sorted"
- "pair"
- "remove duplicates"
- "container"
- "palindrome"

---

## 📝 Must-Do Problems

| Problem | Difficulty | LeetCode |
|---------|-----------|----------|
| Two Sum II (Sorted) | Medium | [#167](https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/) |
| Container With Most Water | Medium | [#11](https://leetcode.com/problems/container-with-most-water/) |
| Valid Palindrome | Easy | [#125](https://leetcode.com/problems/valid-palindrome/) |
| Move Zeroes | Easy | [#283](https://leetcode.com/problems/move-zeroes/) |
| 3Sum | Medium | [#15](https://leetcode.com/problems/3sum/) |

---

## 💡 When to Use

Ask yourself: **"Is the input sorted or can I sort it? Am I looking for pairs?"**

### Two Variants

1. **Opposite ends** — `left = 0`, `right = n-1`, move towards center (pair sums, palindromes)
2. **Same direction** — both start at 0, fast/slow movement (remove duplicates, partitioning)

---

## 🧩 Templates (Java)

### Opposite Ends (Pair Sum on Sorted Array)

```java
int left = 0, right = nums.length - 1;
while (left < right) {
    int sum = nums[left] + nums[right];
    if (sum == target) return new int[]{left, right};
    else if (sum < target) left++;
    else right--;
}
```

### Same Direction (Remove Duplicates In-Place)

```java
int slow = 0;
for (int fast = 1; fast < nums.length; fast++) {
    if (nums[fast] != nums[slow]) {
        slow++;
        nums[slow] = nums[fast];
    }
}
return slow + 1; // new length
```

---

## 🔗 Related Patterns

- [[Sliding Window]] is a specialized form of two pointers on contiguous data
- [[Fast & Slow Pointers]] for linked list cycle detection

---

**← Back to [[DSA Patterns]]**
