```java
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running");
    }
};
```

- An **anonymous inner class** in Java is an **inner class without a name**, created for **single-use**
- It is declared and instantiated in a single expression, typically to **override methods** of a class or interface without creating a separate named subclass. This is especially useful for **event handling**, **callbacks**, or **on-the-fly method customization**.