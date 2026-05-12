# 5. Stack / Monotonic Stack

> 🔥 **VERY important in backend interviews now.**
>
> **Core Idea:** Maintain an ordered structure efficiently to find nearest greater/smaller elements.

---

## 🔍 Recognition Signals

- Nearest greater / smaller element
- Next greater element
- Histogram problems
- Temperature-style "wait X days" problems
- Matching brackets / parentheses

## 🏷️ Keywords in Problem Statement

- "next greater"
- "span"
- "previous smaller"
- "monotonic"
- "valid parentheses"

---

## 📝 Must-Do Problems

| Problem | Difficulty | LeetCode |
|---------|-----------|----------|
| Valid Parentheses | Easy | [#20](https://leetcode.com/problems/valid-parentheses/) |
| Daily Temperatures | Medium | [#739](https://leetcode.com/problems/daily-temperatures/) |
| Next Greater Element I | Easy | [#496](https://leetcode.com/problems/next-greater-element-i/) |
| Largest Rectangle in Histogram | Hard | [#84](https://leetcode.com/problems/largest-rectangle-in-histogram/) |
| Online Stock Span | Medium | [#901](https://leetcode.com/problems/online-stock-span/) |

---

## 📐 Types of Monotonic Stacks

| Type | Maintains | Use Case |
|------|-----------|----------|
| **Monotonic Decreasing** | Top is smallest | Next Greater Element |
| **Monotonic Increasing** | Top is largest | Next Smaller Element |

---

## 🧩 Templates (Java)

### Valid Parentheses

```java
Deque<Character> stack = new ArrayDeque<>();
for (char c : s.toCharArray()) {
    if (c == '(' || c == '{' || c == '[') {
        stack.push(c);
    } else {
        if (stack.isEmpty()) return false;
        char top = stack.pop();
        if ((c == ')' && top != '(') ||
            (c == '}' && top != '{') ||
            (c == ']' && top != '[')) return false;
    }
}
return stack.isEmpty();
```

### Next Greater Element (Monotonic Decreasing Stack)

```java
int[] result = new int[nums.length];
Arrays.fill(result, -1);
Deque<Integer> stack = new ArrayDeque<>(); // stores indices

for (int i = 0; i < nums.length; i++) {
    while (!stack.isEmpty() && nums[i] > nums[stack.peek()]) {
        result[stack.pop()] = nums[i];
    }
    stack.push(i);
}
```

### Daily Temperatures

```java
int[] answer = new int[temperatures.length];
Deque<Integer> stack = new ArrayDeque<>();

for (int i = 0; i < temperatures.length; i++) {
    while (!stack.isEmpty() && temperatures[i] > temperatures[stack.peek()]) {
        int prevDay = stack.pop();
        answer[prevDay] = i - prevDay;
    }
    stack.push(i);
}
```

---

## 💡 Interview Thought Process

> "When I need to find the next greater/smaller for each element, a monotonic stack lets me do it in O(n) instead of O(n²) brute force."

---

## 🔗 Related Patterns

- [[Heap & Priority Queue]] — also maintains ordered elements, but for top-K
- [[Binary Search]] — alternative for some "nearest" queries on sorted data

---

**← Back to [[DSA Patterns]]**
