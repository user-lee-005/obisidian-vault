- Each of the APIs would use _different protocols_
- The protocol choice basically decides the API design options

***The API Design Process***

1. Identify the core use cases and user stories
2. Defining the scope and boundaries (to categorize which are required now and which can be built later)
3. Determine the performance requirements (identify the bottlenecks of that api)
4. Consider security constraints (implement the basic security)

**The Design Approaches**

1. Top-down: Start with high-level requirements and workflows
2. Bottom-up: Begin with the existing data models and capabilities
3. Contract-first: Define the api contract first and then implement (similar to top-down)

---

***API Lifecycle Management***

1. Design: Discuss and design the api
2. Development: Develop and test the api (in dev)
3. Deployment and monitoring: Deploy and test the api (in staging and prod)
4. Maintenance: Maintain the developed api (this is y simplicity matters easier to maintain)
5. Deprecation and retirement: APIs would be marked as deprecated once the usage is completed (example migrating from v1 to v2)