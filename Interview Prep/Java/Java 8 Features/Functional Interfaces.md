---
title: Functional Interfaces
tags:
  - java-8
  - beginner
  - functional-interface
  - java
created: 2026-06-08
status: complete
prerequisites:
  - "[[Basic Object Creation]]"
  - "[[Polymorphism]]"
  - "[[Constructor Chaining]]"
difficulty: beginner-to-intermediate
---
## Why Functional Interface

- Before Java 8, if u want to pass a behavior (a method) to another method, you had to create an [[Anonymous class]]
- To solve this functional interface was introduced so that methods could be treated like data
---
## What is Functional Interface

- A Functional Interface is an interface that contains exactly one abstract method
```java
@FunctionalInterface
public interface Calculator {
    int calculate(int a, int b);
}
```

- A Functional Interface allows default methods because default methods are not abstract.
```java
interface Test {
    void print();

    default void show() {
    }
}
```

- But it is not allowed to have more than one abstract methods
- **Invalid**:
```java
interface Test {
    void print();
    void show();
}
```
---
### Why this rule

- Because Java must know which method a lambda is implementing.
- `Calculator c = (a, b) -> a + b;`
- Java maps this to `int calculate(int a, int b)`
---
## Built-In Functional Interfaces

## Predicate
- Returns boolean
```java
Predicate<Integer> isEven = n -> n % 2 == 0;
// This returns the result as boolean so we can use this for T -> boolean, where T is generic Predicate<T>
```

## Function
- Accepts one value and returns another
```java
Function<String,Integer> length = s -> s.length();
System.out.println(length.apply("Lee"));
// This converts one value to another, T -> R, where T and R are generics, Function<T, R>
```

## Consume
- Consumes input, returns nothing
```java
Consumer<String> printer = s -> System.out.println(s);
printer.accept("Hello");
// T -> void, Consume<T>
```

## Supplier
- Supplies data takes nothing
```java
Supplier<String> supplier = () -> "Hello";
System.out.println(supplier.get());
```

## Unary Operator
- Input and output same type.
```java
UnaryOperator<Integer> square = n -> n * n;
// T -> T, UnaryOperator<T>
```

## Binary Operator
- Two inputs, same type output
```java
BinaryOperator<Integer> add = (a,b) -> a + b;
// (T, T) -> T, BinaryOperator<T>
```
---
