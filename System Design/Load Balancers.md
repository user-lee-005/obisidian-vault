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