# 10. Backtracking

> **Core Idea:** Explore all possibilities by building solutions incrementally and abandoning paths that can't lead to valid solutions.
>
> ⚠️ **Don't go too deep here for now.** Know the template and the classic problems.

---

## 🔍 Recognition Signals

- Combinations / permutations
- Generate all valid possibilities
- "Find all..." or "List all..."
- Constraint satisfaction

---

## 📝 Must-Do Problems

| Problem | Difficulty | LeetCode |
|---------|-----------|----------|
| Subsets | Medium | [#78](https://leetcode.com/problems/subsets/) |
| Permutations | Medium | [#46](https://leetcode.com/problems/permutations/) |
| Combination Sum | Medium | [#39](https://leetcode.com/problems/combination-sum/) |
| Letter Combinations of a Phone Number | Medium | [#17](https://leetcode.com/problems/letter-combinations-of-a-phone-number/) |

---

## 🧩 Template (Java)

### General Backtracking Framework

```java
void backtrack(List<List<Integer>> result, List<Integer> current,
               int[] nums, int start) {
    result.add(new ArrayList<>(current)); // base case / collect result

    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);            // choose
        backtrack(result, current, nums, i + 1); // explore
        current.remove(current.size() - 1);       // un-choose (backtrack)
    }
}
```

### Subsets

```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(result, new ArrayList<>(), nums, 0);
    return result;
}
```

---

## 💡 Key Concept: Choose → Explore → Un-choose

```
1. Choose: Add element to current path
2. Explore: Recurse with remaining elements
3. Un-choose: Remove element (backtrack) to try next option
```

---

## 🔗 Related Patterns

- [[Trees - BFS & DFS]] — backtracking is DFS with undo
- [[Graphs]] — backtracking for path exploration in graphs

---

**← Back to [[DSA Patterns]]**
