---
title: Basic Object Creation
tags:
  - java
  - backend
  - beginner
  - foundations
created: 2026-06-06
aliases:
  - Object Lifecycle
status: complete
---
# Phase 1: Before OOP

- If we need to create multiple employees, we would require to create
```
String employee1Name = "Leela"
String employee2Name = "Lee"
```
- This becomes impossible to manage
---
# Phase 2: Class is born

- So we create a blueprint of the objects first, which is called a CLASS
```
Class Employee {
	String name,
	int id,
	double salary
}
```
- Until now we don't have anything in the memory just a blue print
---
# Phase 3: Creating the first object

- Now we write, `Employee emp`
- This would create an uninitialized reference, `emp -> null`
---

# Phase 4: JVM sees new

- The JVM reads ***new*** keyword
- Then it checks, is class loaded, if not loads it into the memory (refer [[Loading Class into memory]])
- Metaspace stores the following information
```
Employee
|
+-- field id
+-- field name
+-- field salary
+-- constructor
+-- method info
+-- bytecode info
```
- Now the memory contains only the blueprint still no object is created.
- JVM then allocates space for the object in the heap memory.
- Since these are primitives, JVM does not load garbage value, it initializes the default values
```java
name = null
id = 0
salary = 0.0
```
- Now JVM says memory allocation done, time to initialize the object and the constructor executes, which is called by the JVM automatically
- After the constructor finishes, `new Employee()` returns the address of the object
- And this address is referenced to the _emp_ variable in the memory
```
Stack                Heap

emp -----------> Employee Object
                     id=0
                     name=null
```
---

## Why constructor exists

### Default constructor
- If every object has to have some default value during the initialization then we can use constructor and initialize the values rather than having every time creating the object and setting the values to the object
### Parameterized Constructor
- If every employee has different data, we pass values so we use parameterized constructors
---
# JVM Story

```
Allocate memory
      ↓
Put default values
      ↓
Call constructor chain (this(...) / super(...))
      ↓
Pass 101 and "Lee"
      ↓
Assign values
      ↓
Return reference
```
---

# What is `this`

- `this` is a reference to the **current object** — the one on which the constructor or method was called
- JVM automatically passes the heap address of the current object as a hidden argument to every instance method and constructor

---

## Why it's needed

When a parameter name clashes with an instance variable name, the compiler would shadow the field:

```java
Employee(int id, String name) {
    id = id;     // WRONG: assigns parameter to itself, field unchanged
    name = name; // WRONG: same problem
}
```

`this` fixes the ambiguity — `this.id` means the field on the heap object:

```java
Employee(int id, String name) {
    this.id = id;     // heap field  ←  parameter
    this.name = name;
}
```

---

## Constructor chaining with `this()`

`this()` calls another constructor in the **same class**. Must be the **first statement**.

```java
class Employee {
    int id;
    String name;
    double salary;

    Employee(int id, String name, double salary) {
        this.id = id;
        this.name = name;
        this.salary = salary;
    }

    Employee(int id, String name) {
        this(id, name, 0.0);  // delegates to the 3-arg constructor
    }

    Employee() {
        this(0, "Unknown");   // delegates to the 2-arg constructor
    }
}
```

JVM story for `new Employee()`:
```
Allocate heap space
      ↓
Put default values (id=0, name=null, salary=0.0)
      ↓
Enter Employee() → this(0, "Unknown")
      ↓
Enter Employee(0, "Unknown") → this(0, "Unknown", 0.0)
      ↓
Enter Employee(0, "Unknown", 0.0) → assign fields
      ↓
Return reference
```

---

## `this` as a return value

Returning `this` enables **method chaining** (builder pattern):

```java
class Employee {
    Employee setName(String name) {
        this.name = name;
        return this;  // same object reference returned
    }

    Employee setSalary(double salary) {
        this.salary = salary;
        return this;
    }
}

// Usage
Employee emp = new Employee().setName("Lee").setSalary(50000);
```
