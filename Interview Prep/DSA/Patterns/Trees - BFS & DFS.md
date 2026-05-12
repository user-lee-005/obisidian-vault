# 7. Trees (BFS / DFS)

> 🔥🔥 **ABSOLUTE MUST.** Trees appear in almost every interview.

---

## 🔍 Recognition Signals

- Hierarchy
- Traversal (inorder, preorder, postorder, level-order)
- Shortest path in tree
- Recursive structure
- Subtree calculations

---

## 📐 DFS vs BFS

| Approach | Best For | Implementation |
|----------|----------|----------------|
| **DFS** | Path exploration, subtree calculations, recursion | Recursion or explicit stack |
| **BFS** | Level-order traversal, shortest path, nearest node | Queue |

---

## 📝 Must-Do Problems

| Problem | Difficulty | Approach | LeetCode |
|---------|-----------|----------|----------|
| Maximum Depth of Binary Tree | Easy | DFS | [#104](https://leetcode.com/problems/maximum-depth-of-binary-tree/) |
| Binary Tree Level Order Traversal | Medium | BFS | [#102](https://leetcode.com/problems/binary-tree-level-order-traversal/) |
| Lowest Common Ancestor | Medium | DFS | [#236](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/) |
| Path Sum | Easy | DFS | [#112](https://leetcode.com/problems/path-sum/) |
| Validate BST | Medium | DFS | [#98](https://leetcode.com/problems/validate-binary-search-tree/) |
| Invert Binary Tree | Easy | DFS/BFS | [#226](https://leetcode.com/problems/invert-binary-tree/) |
| Diameter of Binary Tree | Easy | DFS | [#543](https://leetcode.com/problems/diameter-of-binary-tree/) |

---

## 🧩 Templates (Java)

### DFS — Max Depth (Recursive)

```java
public int maxDepth(TreeNode root) {
    if (root == null) return 0;
    return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
}
```

### BFS — Level Order Traversal

```java
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;

    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> level = new ArrayList<>();

        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        result.add(level);
    }
    return result;
}
```

### DFS — Lowest Common Ancestor

```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    if (left != null && right != null) return root;
    return left != null ? left : right;
}
```

---

## 💡 Interview Tips

> [!tip] Tree Traversal Orders
> - **Inorder** (Left → Root → Right): gives sorted order for BST
> - **Preorder** (Root → Left → Right): used to serialize/copy tree
> - **Postorder** (Left → Right → Root): used for deletion, bottom-up calculations

---

## 🔗 Related Patterns

- [[Graphs]] — trees are a special case of graphs (connected, acyclic)
- [[Backtracking]] — DFS with backtracking for path/combination problems on trees

---

**← Back to [[DSA Patterns]]**
