---
title: Method References
created: 2026-06-08
tags:
  - java-8
  - beginner
  - fundamentals
  - method-reference
  - lambda
phase: fundamentals
stack: java
prerequisites:
  - "[[Lambda Functions]]"
---
## What is method reference

- A method reference is just syntactic sugar for a lambda

### Types of Method References:

1. Static method references:
```java
// Actual method
class MathUtil {
    public static int square(int n) {
        return n * n;
    }
}

// Lambda
Function<Integer,Integer> f = n -> MathUtil.square(n);

// Method Reference
Function<Integer,Integer> f = MathUtil::square;
```

2. Instance Method Reference of Particular Object:
```java
// Instance 
String str = "Java";

// Lambda
Supplier<Integer> s = () -> str.length();

// Method Reference
Supplier<Integer> s = () -> str.length();
```

3. Instance Method Reference of Arbitrary Object:
```java
// Instance
List<String> names = List.of("Lee","Java","Spring");

// Lambda
names.forEach(
    s -> System.out.println(s)
);

// Method Reference
names.forEach(System.out::println);
```

4. Constructor Reference:
```java
// Lambda
Supplier<Employee> s = () -> new Employee();

// Method Reference
Supplier<Employee> s = Employee::new;
```

---

# Method References - Compiler Mapping Notes

## Core Idea

A method reference is just a shorter form of a lambda expression.

The compiler does not see:

```java
String::length
```

It effectively thinks:

```java
s -> s.length()
```

The compiler's job is to convert the method reference into a lambda that matches the target Functional Interface.

---

# Step 1: Look at the Functional Interface

Example:

```java
Function<String, Integer> f = String::length;
```

The compiler first looks at:

```java
Function<String, Integer>
```

Which means:

```java
String -> Integer
```

or

```java
(String s) -> Integer
```

Now the compiler needs a method that matches this shape.

---

# Step 2: Find a Matching Method

Method reference:

```java
String::length
```

Actual method:

```java
int length()
```

Compiler notices:

- Input String can become the object
    
- Method returns int
    

Therefore:

```java
String::length
```

becomes:

```java
s -> s.length()
```

---

# Compiler Rule #1

When you see:

```java
ClassName::instanceMethod
```

The first parameter becomes the object.

Remaining parameters become method arguments.

Template:

```java
(a, b, c) -> a.instanceMethod(b, c)
```

---

## Example 1

```java
Function<String, Integer> f = String::length;
```

Compiler sees:

```java
(String) -> Integer
```

Equivalent lambda:

```java
s -> s.length()
```

---

## Example 2

```java
BiFunction<String, String, String> f =
        String::concat;
```

Compiler sees:

```java
(String, String) -> String
```

Equivalent lambda:

```java
(a, b) -> a.concat(b)
```

---

## Example 3

```java
BiPredicate<String, String> p =
        String::startsWith;
```

Compiler sees:

```java
(String, String) -> boolean
```

Equivalent lambda:

```java
(a, b) -> a.startsWith(b)
```

---

# Compiler Rule #2

When you see:

```java
object::instanceMethod
```

The object is already fixed.

Template:

```java
(arg1, arg2) ->
        object.instanceMethod(arg1, arg2)
```

---

## Example

```java
String str = "Java";

Supplier<Integer> s =
        str::length;
```

Equivalent lambda:

```java
() -> str.length()
```

---

## Example

```java
Consumer<String> c =
        System.out::println;
```

Equivalent lambda:

```java
s -> System.out.println(s)
```

The object already exists:

```java
System.out
```

So compiler only maps the method arguments.

---

# Compiler Rule #3

When you see:

```java
ClassName::staticMethod
```

The compiler simply forwards parameters.

Template:

```java
(a, b) ->
        ClassName.staticMethod(a, b)
```

---

## Example

```java
Function<Integer, Integer> f =
        MathUtil::square;
```

Equivalent lambda:

```java
n -> MathUtil.square(n)
```

---

## Example

```java
BiFunction<Integer, Integer, Integer> f =
        Math::max;
```

Equivalent lambda:

```java
(a, b) -> Math.max(a, b)
```

---

# Compiler Rule #4

When you see:

```java
ClassName::new
```

The compiler maps parameters to a constructor.

Template:

```java
(a, b) ->
        new ClassName(a, b)
```

---

## Example

```java
Supplier<Employee> s =
        Employee::new;
```

Equivalent lambda:

```java
() -> new Employee()
```

---

## Example

```java
Function<String, Employee> f =
        Employee::new;
```

Equivalent lambda:

```java
name -> new Employee(name)
```

---

# How To Decode Any Method Reference

## Question 1

What is the target Functional Interface?

Example:

```java
BiFunction<String, String, String>
```

Meaning:

```java
(String, String) -> String
```

---

## Question 2

What method is being referenced?

Example:

```java
String::concat
```

Actual method:

```java
String concat(String str)
```

---

## Question 3

Apply the Mapping Rule

First parameter becomes object:

```java
(a, b)
```

becomes:

```java
a.concat(b)
```

Final lambda:

```java
(a, b) -> a.concat(b)
```

---

# Quick Cheat Sheet

## Static Method

```java
ClassName::staticMethod
```

↓

```java
(a,b) -> ClassName.staticMethod(a,b)
```

---

## Particular Object Method

```java
object::method
```

↓

```java
(a,b) -> object.method(a,b)
```

---

## Arbitrary Object Method

```java
ClassName::instanceMethod
```

↓

```java
(a,b,c) -> a.instanceMethod(b,c)
```

---

## Constructor Reference

```java
ClassName::new
```

↓

```java
(a,b) -> new ClassName(a,b)
```

---

# Interview Golden Rule

Always start from the Functional Interface.

The compiler does NOT start from:

```java
String::concat
```

The compiler starts from:

```java
BiFunction<String,String,String>
```

and then asks:

"Can I map these parameters to concat()?"

If yes, compilation succeeds.