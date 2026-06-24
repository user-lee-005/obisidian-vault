```java
class Animal {
    void sound() {
        System.out.println("Animal");
    }
}

class Dog extends Animal {
    @Override
    void sound() {
        System.out.println("Dog");
    }
}

public static void main(String[] args) {
    Animal a = new Dog();
    a.sound();
}
```

The above is an example of polymorphism
### What happens

1. Animal a = new Dog () --> This line creates a Dog object in the heap memory that also has animal parts
```
Dog Object

+----------------+
| Animal Part    |
+----------------+
| Dog Part       |
+----------------+
```

2. During compile time, JVM checks if the method is present in the Animal class or not, If yes code compiles else it does not
3. Then during the runtime JVM executes a.sound(), and at this time it asks what is the actual object type, and it uses the method table (a vtable) associated with the Dog class, therefore Dog.sound() runs
---
> [!tip] Important Rule:
> - Reference Type decides: What CAN be called
> - Object Type decides: What ACTUALLY executes
> - Fields, static methods, and overloaded methods are resolved using the reference type.
> - Instance overridden methods are resolved using the actual object type at runtime.

---
- Fields are not polymorphic methods are polymorphic
```java
class Animal {
    String name = "Animal";
}

class Dog extends Animal {
    String name = "Dog";
}

Animal a = new Dog();
a.name; // Output: Animal 
```
- Field access uses reference type
- Method access uses object type
---
> [!info] Implementation in JVM
> For static methods during compile time, we have invokeStatic as instruction in the byte code this tells the JVM to call the method directly, whereas for instance methods we have invokeVirtual as instruction in bytecode this tells JVM to find the object, find the class metadata, find implementation and then execute

