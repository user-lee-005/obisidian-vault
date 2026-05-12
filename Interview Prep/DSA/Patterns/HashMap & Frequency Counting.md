# 1. HashMap / Frequency Counting

> **Core Idea:** Trade memory for O(1) lookup.

---

## 🔍 Recognition Signals

- Duplicates
- Frequency / counting occurrences
- Lookup optimization
- Pair finding
- Grouping

## 🏷️ Keywords in Problem Statement

- "find duplicates"
- "most frequent"
- "group"
- "anagram"
- "two sum"
- "count distinct"

---

## 📝 Must-Do Problems

| Problem | Difficulty | LeetCode |
|---------|-----------|----------|
| Two Sum | Easy | [#1](https://leetcode.com/problems/two-sum/) |
| Valid Anagram | Easy | [#242](https://leetcode.com/problems/valid-anagram/) |
| Group Anagrams | Medium | [#49](https://leetcode.com/problems/group-anagrams/) |
| Top K Frequent Elements | Medium | [#347](https://leetcode.com/problems/top-k-frequent-elements/) |
| Longest Consecutive Sequence | Medium | [#128](https://leetcode.com/problems/longest-consecutive-sequence/) |

---

## 💡 Interview Thought Process

> [!quote] "Repeated lookups suggest hashmap optimization."

### When to Use

Ask yourself: **"Do I need to find/check/count something repeatedly?"**

- If checking existence → `HashSet`
- If counting frequency → `HashMap<Key, Count>`
- If grouping → `HashMap<Key, List<Value>>`

### Complexity

| Operation | HashMap | Brute Force |
|-----------|---------|-------------|
| Lookup | O(1) | O(n) |
| Insert | O(1) | — |
| Space | O(n) | O(1) |

---

## 🧩 Template (Java)

```java
// Frequency counting pattern
Map<Integer, Integer> freq = new HashMap<>();
for (int num : nums) {
    freq.put(num, freq.getOrDefault(num, 0) + 1);
}

// Two Sum pattern — complement lookup
Map<Integer, Integer> seen = new HashMap<>();
for (int i = 0; i < nums.length; i++) {
    int complement = target - nums[i];
    if (seen.containsKey(complement)) {
        return new int[]{seen.get(complement), i};
    }
    seen.put(nums[i], i);
}
```

---

## 🔗 Related Patterns

- Often combined with [[Sliding Window]] for substring problems
- [[Heap & Priority Queue]] for "Top K" after frequency counting

---

**← Back to [[DSA Patterns]]**
