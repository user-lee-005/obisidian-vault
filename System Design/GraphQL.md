- It is a query language that allows the client to request exactly what they need by providing the payload in the request
- This means that it comes with the single endpoint for all the operations
- There are 3 operations:
	1. Query (read)
	2. Mutation (write)
	3. Subscription (real-time)
- This is used for minimizing the round trips for requests
- If the client requires data that takes 3 REST calls that could be retrieved here using 1 GraphQL call
- This is used for complex UIs where on page we need different data and on another page we need different complex nested data then graphQL is the optimal choice
- No versioning required
- Application level caching
- Query language defines the operations
---
Example:
```
query {
	user(id: "123") {
		name
		posts { title, content }
		followers { name }
	}
}
```
