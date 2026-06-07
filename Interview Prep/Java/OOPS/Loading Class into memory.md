- The checking happens at the metaspace in java 8+
- If there is no information in the metaspace, JVM uses `ClassLoader` and loads the class into the metaspace
- Metaspace stores:
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
- Heap Stores:
```
Employee Object #1

id=101
name="Lee"
```
---

# Metaspace Cleaning

- Heap memory is cleaned by GC
- Metaspace is cleared by Class Unloading (refer [[Class Loading and Unloading]])