# 8. Graphs

> **Core Patterns:** BFS, DFS, Topological Sort.

---

## 🔍 Recognition Signals

- Dependencies
- Connections / routes
- Network / grid traversal
- Prerequisites

## 🏷️ Keywords in Problem Statement

- "can finish" (course schedule)
- "connected"
- "path"
- "cycle"
- "network"
- "island"

---

## 📝 Must-Do Problems

| Problem | Difficulty | Pattern | LeetCode |
|---------|-----------|---------|----------|
| Number of Islands | Medium | DFS/BFS on grid | [#200](https://leetcode.com/problems/number-of-islands/) |
| Clone Graph | Medium | BFS/DFS + HashMap | [#133](https://leetcode.com/problems/clone-graph/) |
| Course Schedule | Medium | Topological Sort | [#207](https://leetcode.com/problems/course-schedule/) |
| Course Schedule II | Medium | Topological Sort | [#210](https://leetcode.com/problems/course-schedule-ii/) |
| Pacific Atlantic Water Flow | Medium | Multi-source DFS | [#417](https://leetcode.com/problems/pacific-atlantic-water-flow/) |

---

## 📐 Core Graph Patterns

### 1. BFS (Breadth-First Search)
- **Shortest path** in unweighted graphs
- **Level-order** exploration

### 2. DFS (Depth-First Search)
- **Connected components**
- **Cycle detection**
- **Path finding**

### 3. Topological Sort
- **Ordering with dependencies** (e.g., build systems, course prerequisites)
- Only works on **DAGs** (Directed Acyclic Graphs)

---

## 🧩 Templates (Java)

### BFS — Number of Islands (Grid)

```java
public int numIslands(char[][] grid) {
    int count = 0;
    for (int i = 0; i < grid.length; i++) {
        for (int j = 0; j < grid[0].length; j++) {
            if (grid[i][j] == '1') {
                count++;
                bfs(grid, i, j);
            }
        }
    }
    return count;
}

void bfs(char[][] grid, int r, int c) {
    Queue<int[]> queue = new LinkedList<>();
    queue.offer(new int[]{r, c});
    grid[r][c] = '0';
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};

    while (!queue.isEmpty()) {
        int[] cell = queue.poll();
        for (int[] d : dirs) {
            int nr = cell[0] + d[0], nc = cell[1] + d[1];
            if (nr >= 0 && nr < grid.length && nc >= 0 &&
                nc < grid[0].length && grid[nr][nc] == '1') {
                grid[nr][nc] = '0';
                queue.offer(new int[]{nr, nc});
            }
        }
    }
}
```

### Topological Sort — Course Schedule (Kahn's BFS)

```java
public boolean canFinish(int numCourses, int[][] prerequisites) {
    int[] indegree = new int[numCourses];
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) adj.add(new ArrayList<>());

    for (int[] pre : prerequisites) {
        adj.get(pre[1]).add(pre[0]);
        indegree[pre[0]]++;
    }

    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) {
        if (indegree[i] == 0) queue.offer(i);
    }

    int completed = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        completed++;
        for (int next : adj.get(course)) {
            if (--indegree[next] == 0) queue.offer(next);
        }
    }
    return completed == numCourses;
}
```

---

## 💡 Graph Representation

| Type | When to Use |
|------|------------|
| **Adjacency List** | Sparse graphs (most interview problems) |
| **Adjacency Matrix** | Dense graphs, quick edge lookup |
| **Grid** | 2D matrix problems (islands, paths) |

---

## 🔗 Related Patterns

- [[Trees - BFS & DFS]] — trees are acyclic connected graphs
- [[Backtracking]] — for exhaustive path exploration

---

**← Back to [[DSA Patterns]]**
