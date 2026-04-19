Designing system for million users is complex, lets start from building one user

**Step 1: Building a Single-Server Setup**
- Lets imagine everything like Web App, Database, Cache in a single server.!

**Step 2: Understanding the request flow**
- First the user opens the app in the browser or mobile app.
- That sends request the to DNS for resolving the domain name to the ip address.
- Once the user receives the ip address, the user sends the HTTP request and receives the html or json which is requested.

**Step 3: Identifying traffic source**
- Source is the web browser or mobile app.
- We need to identify areas where a single server might fall short as user demand increases.

---

**Key Takeaways**
1. *Start Small*: Begin with a straight forward single server-setup to understand the essential components of the system architecture.
2. *Request Flow*: Understand how requests flow thru system which is fundamental for building more scalable system
3. *Traffic Sources*: Web apps and mobile apps and how do they interact with the server

---

Single server cannot handle the increasing demand of the users so we could split this server into 2, 
1. Web Tier --> Have the web app and the cache
2. Data Tier --> Has the database