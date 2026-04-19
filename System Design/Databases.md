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

**Load Balancers**

**7 Strategies and algorithms that are commonly used in load balancing**
1. Round Robin:
	- Simplest form of load balancing
	- Each server in the pool gets the request in sequential manner
	- Works well for servers with similar specifications
2. Least connections:
	- Directs traffic to the server with fewest connections active
	- Useful for applications that have sessions of variable length
3. Least Response Time:
	- More focused on the responsiveness of the server
	- In this the load balancer chooses the server with lowest response time (Highly responsive) and least active connections
	- Effective when the goal is to provide the fastest response time to the requests and also have different servers with different capabilities
4. IP Hash:
	- Default option
	- Determines which server receives the request based on the hash of the client ip address
	- This is useful when u want to connect the clients consistently to the same server
5. Weighted Algorithms:
	- Variants of the above methods that can also be weighted
	- Like weighted round robin and weighted least connections
	- The weights are typically based on their capacity and performance metrics
6. Geographical Algorithms:
	- Location based algorithms
	- Direct request to the server geographically close to the user
	- Useful for global services where latency reduction is important
7. Consistent Hashing:
	- Most popular type
	- Use hash function to distribute data across various nodes
	- We have a hash function inside the load balancer
	- And we usually imagine a hash space along with the function, that forms a [[Hash Ring]] like a circle
	- More complicated way of load balancing
	- Ensures the same client connects consistently to the same server like ip hashing

---
**Health Checks**
- Whenever a server goes down, load balancer ensures the traffic is not sent to that server.
- For identifying this most load balancers come with health check feature.
- This means that the load balancers are continuously monitoring the servers by sending health check requests to all servers, and they have the info on which servers are online and offline.
- Whenever it detects a failure in the health checks it knows that the server is down and does not send any request there until the server is up.

---
**Load Balancers Example and how to implement them**
- Software Load Balancers:
	- Example - Nginx, HAProxy
- Hardware Load Balancers:
	- Example - F5 Load balancer(widely used), Citrix
- Easier solutions are cloud based load balancers (AWS, GCP, Azure)