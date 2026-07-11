---
title: Docker Networking
topic: infra
tags:
  - docker
  - networking
  - cloud
stack: docker
prerequisites:
  - "[[Docker Basics]]"
created: 2026-06-26
---
## Why does docker need networking
- If we have 3 applications, Spring boot, PostgresQL, Redis, without docker each will be running in different ports sharing the host's network
- With docker, each gets its own isolated network stack
- Each container gets its own:
	- IP
	- Network Interface
	- Routing table
	- ports

---

## Every container has its own localhost
- Suppose the application says 
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/app
```
this will fail if the postgresql is running in some other container as localhost points within the container, it will try to find the PostgreSQL inside itself

---

## Docker creates networks
- When running `docker network ls` we could see,
```
1. bridge
2. host
3. none
```
- These are docker network drivers

### 1. Bridge Network (default)
- When you run `docker run nginx` Docker automatically connects to `bridge`
- Example if we have a docker bridge like:
```
            Docker Bridge Network

      ┌──────────┬──────────┬──────────┐
      │          │          │          |
      ▼          ▼          ▼          ▼

 Spring      PostgreSQL     Redis     Nginx
```

- Every container on this network can communicate
- Each has its own IP
- Docker performs **Network Address Translation (NAT)** so containers can make outbound connections without needing public IP addresses.
- But browser cannot access the spring boot application until docker published it using
  `docker run -p 8080:8080` command
- Container-to-container communication is possible if both are on the same bridge network
- Docker provides **DNS-based service discovery** on user-defined bridge networks, so avoid using IPs as they change
- ### Docker DNS:
	- Docker includes an internal DNS server for user-defined bridge network
	- Lets create 2 containers named spring-app, postgres
	- In the spring boot app if we mention, `jdbc:postgresql://postgres:5432/app` this _postgres_ gets translated to the IP automatically using that DNS server present
	- This is exactly why Docker Compose uses service names

### 2. Host Network
- When we use `docker run --network host` Docker removes network isolation
- Containers uses the host network directly
- ### Advantages
	- Very fast
	- No NAT overhead
- ### Disadvantages
	- No isolation
	- Port conflicts
	- Not portable

### 3. None Network
- `docker run --network none` --> this means  container has:
	- No Internet
	- No Bridge
	- No Communication
- Only localhost
- Useful for highly isolated workloads or debugging
