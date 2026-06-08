---
title: Lambda Functions
tags:
  - java-8
  - beginner
  - java
  - foundations
  - fundamentals
created: 2026-06-08
stack: java
prerequisites:
  - "[[Functional Interfaces]]"
status: complete
---
## What is Lambda Functions
- A lambda is simply a shorter way of implementing a Functional Interface.
```java
// Anonymous Class
Calculator c = new Calculator() {
    @Override
    public int calculate(int a, int b) {
        return a + b;
    }
};

// Lambda
Calculator c = (a, b) -> a + b;
```

- Not Allowed:
```java
int x = 10;

Runnable r = () -> {
    System.out.println(x);
};

x++;
```

- The variables captured must be effectively final (that means the variable used in the lambda functions should not be changed after the lambda function execution)
---

