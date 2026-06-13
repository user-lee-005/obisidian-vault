---
title: Streams
tags:
  - java
  - java-8
  - beginner
  - foundations
  - fundamentals
  - streams
stack: java
difficulty: beginner-to-intermediate
created: 2026-06-11
status: in-progress
prerequisites:
  - "[[Functional Interfaces]]"
  - "[[Method References]]"
  - "[[Lambda Functions]]"
---
### What is a stream?
- A stream is a sequence of elements supporting aggregate operations.
```
Collection -> Data Storage
Stream -> Data Processing Pipeline
```

---
## Stream Pipeline

> [!tip] Every stream consists of:
> Stream -> Intermediate Operations -> Terminal Operations
> ```java
> names.stream()
> .filter(s -> s.length() > 3)
> .map(String::toUpperCase)
> .toList();
> ```

.stream() --> creates stream
.filter(), .map() --> intermediate operations
.toList() --> terminal operation

---
## Stream Lifecycle

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);
Stream<Integer> stream = nums.stream(); // Creates Stream
stream.count(); // Ends stream
stream.count(); // Runtime Excpetion, stream is already closed
```
---
## Lazy Evaluation

- This is the most important stream concept.
- Till a terminal operation is seen stream just builds the pipeline and does not execute it
### Y this design choice:
1. Avoid unnecessary work:
	- For each operation a new collection would be created
	- Lazy Evaluation allows short circuiting (The stream can stop processing as soon as the final answer is known, instead of processing all elements.)
2. Pipeline Optimization:
```java
list.stream()
.filter(x -> x > 10)
.map(x -> x * 2)
.filter(x -> x < 100)
.collect(Collectors.toList()); 
```
- For this the execution would be, `element1 -> filter -> map -> filter -> collect` and this repeats for all the elements
- This reduces temporary storage and improves caching locality
3. Infinite Streams become possible
```java
Stream.iterate(1, n -> n + 1) // If Eager this would be running inifintely
      .filter(n -> n % 2 == 0)
      .limit(5)
      .forEach(System.out::println);
```
Lazy evaluation allows: Generate one, Process one, Stop when limit is reached
4. For large datasets if eager, More Memory, More copying, More GC Pressure, Less performance.

> [!tip] Short Interview answer:
> Streams are lazily evaluated to enable efficient pipeline processing, short-circuit operations, and support infinite data sources. Intermediate operations only describe the computation, while execution happens when a terminal operation is invoked, allowing Java to avoid unnecessary work and temporary collections.

---
## Intermediate Operations

- Returns stream

```java
filter()
map()
flatMap()
sorted()
distinct()
peek()
limit()
skip()
```
---
## Terminal Operations

- Returns result

```java
collect()
toList()
count()
reduce()
findFirst()
findAny()
forEach()
min()
max()
anyMatch()
allMatch()
noneMatch()
```
---
## Filter vs Map

- filter --> removes elements
- mpa --> transforms elements
---
