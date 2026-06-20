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
- map --> transforms elements
---
## FlatMap

- map() - transforms one element into another
- flatMap() - transforms one element into multiple elements and flattens the stream structure (into single stream)
```java
List<Integer> flatList = List.of(  
	List.of(1,2),  
	List.of(3,4)  
).stream()
.flatMap(List::stream)
.toList(); // results -> [1, 2, 3, 4], map() returns -> [[1,2], [3,4]]
```
---
### Stateless vs Stateful Pipelines

- There are 2 main scenarios in the execution of java streams:
	- When all stages are stateless
	- When not all stages are stateless
- ### Stateless operations:
	- A stateless operation doesn’t need to know about any other element to be able to emit a result. Examples of stateless operations are: [filter](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#filter-java.util.function.Predicate-), [map](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#map-java.util.function.Function-) or [flatMap](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#flatMap-java.util.function.Function-).
- ### Stateful operations:
	- On the contrary, stateful operations need to know about all the elements before emitting a result. Examples of stateful operations are: [sorted](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#sorted--), [limit](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#limit-long-) or [distinct](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html#distinct--).
- If all operations are stateless then the stream can be processed in one go. In Stateful operations, the pipeline is divided into sections using stateful operations as delimiters.

### Stateless pipelines:
```java
final List<Employee> updatedSalaryEmployees = employees
	.stream()
	.filter(employee ->{
		System.out.println("Filtering employee " + employee.getName());
		return employee.getSalary() < 80_000;
	})
	.map(employee ->{
		System.out.println("Mapping employee " + employee.getName());
		long newSal = employee.getSalary() *1.05;
		return new Employee(employee.getName(), newSal);
	})
	.collect(toList());
```
- Here both the operations filter and map are stateless operations
- The execution order would be normal (Filter 1, Map 1, Filter 2, Map 2)

### Stateful pipelines:
```java
final List<Employee> updatedSalaryEmployees = employees.stream()
	.filter(employee ->{
		System.out.println("Filtering employee " + employee.getName());
		return employee.getSalary() < 80_000;
	})
	.sorted() // distinct(), limit(), skip()
	.map(employee ->{
		System.out.println("Mapping employee " + employee.getName());
		long newSal = employee.getSalary() * 1.05;
		return new Employee(employee.getName(), newSal);
	})
	.collect(toList());
```
- Stateful operation buffers all the elements until it reaches the last element and then it emits a result.
- What would happen in this execution is until it sees a stateful operation the flow remains the same, once it sees the stateful operation is stores all the data (until then the next steps wait), once all the data is stored it proceeds to next steps
- Execution order --> Filter 1, Filter 2 -> {sorting takes place here and stores the data} -> Map 1, Map 2
---
## Distinct

- Purpose - removes duplicates
- Stateful operation
```java
List.of(1,2,2,3,3,3,4)
    .stream()
    .distinct()
    .toList(); // returns [1,2,3,4]
```
- Internally, streams use a Set-like mechanism to remember already seen elements
---
## Sorted

- Purpose - Sort stream elements
- Natural Sorting:
```java
List.of(4,2,1,3)
    .stream()
    .sorted()
    .toList();
```
- Reverse Sorting:
```java
List.of(4,2,1,3)
    .stream()
    .sorted(Comparator.reverseOrder())
    .toList();
```
- Custom sorting:
```java
employees.stream()
         .sorted(
             Comparator.comparing(Employee::getSalary)
         )
         .toList();
```
- Custom sorting (descending):
```java
employees.stream()
         .sorted(
             Comparator.comparing(Employee::getSalary)
                       .reversed()
         )
         .toList();
```
---
## Limit

- Purpose - take first n elements
```java
List.of(1,2,3,4,5)
    .stream()
    .limit(3)
    .toList(); // returns [1, 2, 3]
```
---
## Skip

- Purpose - Skip first n elements
```java
List.of(1,2,3,4,5)
    .stream()
    .skip(2)
    .toList(); // returns [3, 4, 5]
```
---
## Peek

- Purpose - Inspect elements
```java
List.of(1,2,3)
    .stream()
    .peek(System.out::println)
    .count();
```
- Don't mutate objects in peek.
---
