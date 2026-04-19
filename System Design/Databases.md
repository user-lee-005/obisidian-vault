**Choosing the right type of database**
There are 2 main options:
- RDBMS
- Non-relational or NoSQL

---
**Relational Databases**

- These DB use SQL for finding and manipulating data
- The data here is structured in the form of tables, consists Rows and Columns
- **Advantages**
	- Support Complex [[Join operations]] across multiple tables 
	- Provide robust data consistency and integrity especially for [[Transactions]]

---
**Non-Relational Databases**

- There are different forms of no sql databases:
	- [[Document stores]] - Mongo DB
	- [[Wide-column stores]] - Cassandra
	- [[Key-Value stores]] - Redis
	- [[Graph stores]] - Neo4j

---

**Advantages**
1. Can handle highly dynamic and large datasets.
2. Optimized for low-latency and scalability.

---

[[Relational vs Non-Relational Databases]]

---

**Scaling**

There are 2 primary ways: a. Vertical, b. Horizontal

1. **Vertical Scaling**:
	- Scale up
	- Adding more resources to the CPU
	- Simple and suitable for low or moderate traffic
	- Limitations: Hard cap on the resource limit, Lack of redundancy (lack of replicas)
2. **Horizontal Scaling**:
	- Scale out
	- Adding more servers to share the load
	- Replicate the original servers
	- More suitable for large scale application with higher fault tolerance.
	- Better scalability

**Distribution of requests**
- Use load balancers for distributing the traffics across multiple servers
- Load balancer controls the fault tolerance

---

[[Load Balancers]]

---



