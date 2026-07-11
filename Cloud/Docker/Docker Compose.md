---
title: Docker Compose
topic: infra
tags:
  - docker
  - infrastructure
  - networking
  - cloud
stack: docker
prerequisites:
  - "[[Docker Basics]]"
  - "[[Docker Volumes]]"
  - "[[Docker Networking]]"
created: 2026-06-26
---
Docker Compose is a tool used to **define and manage multiple containers as a single application**.

Instead of executing multiple `docker run` commands manually, everything is described in a single `docker-compose.yml` file.

Without Compose

```bash
docker network create app-network

docker volume create postgres-data

docker build -t spring-app .

docker run ...

docker run ...

docker run ...
```

With Compose

```bash
docker compose up
```

Everything starts automatically.

---

## Why Docker Compose?

Docker Compose allows you to manage:

- Multiple Containers
- Networks
- Volumes
- Environment Variables
- Startup Order
- Image Builds

Everything is defined declaratively in a YAML file.

```
docker-compose.yml

        │

        ▼

Application

├── Services
├── Networks
├── Volumes
└── Environment Variables
```

---

## Basic Structure

A Compose file generally consists of

```yaml
services:

networks:

volumes:
```

Think of it as

```
Application
    │
    ▼
Services
    │
    ▼
Networks
    │
    ▼
Volumes
```

---

## Services

A **Service** is a blueprint for creating a container.

Example

```yaml
services:

  app:

  postgres:

  redis:
```

Compose creates

```
Application

↓

3 Containers

↓

app

postgres

redis
```

Each service becomes one running container.

---

## Build vs Image

### build

Used when the application source code is available.

```yaml
services:

  app:
    build: .
```

Equivalent to

```bash
docker build .
```

Compose automatically builds the Docker image before starting the container.

---

### image

Used when the image already exists locally or in a registry.

```yaml
services:

  app:
    image: my-app:1.0
```

Equivalent to

```bash
docker pull my-app:1.0
```

---

> [!TIP]
> Use **build** during development and **image** in production deployments.

---

## Ports

Used to publish container ports to the host.

```yaml
ports:
  - "8080:8080"
```

Equivalent

```bash
docker run -p 8080:8080
```

Format

```
HOST_PORT : CONTAINER_PORT
```

---

## Environment Variables

Used to pass configuration into containers.

```yaml
environment:
  SPRING_PROFILES_ACTIVE: prod
  SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/demo
```

Equivalent

```bash
docker run \
-e SPRING_PROFILES_ACTIVE=prod
```

---

## Volumes

Used for persistent storage.

```yaml
volumes:

  - postgres-data:/var/lib/postgresql/data
```

Equivalent

```bash
docker run \
-v postgres-data:/var/lib/postgresql/data
```

Compose automatically creates the volume if it does not exist.

---

## Networks

Compose automatically creates an isolated network.

Example

```yaml
networks:

  backend:
```

Every service attached to this network can communicate using service names.

Example

```
Spring Boot

↓

postgres

↓

Docker DNS

↓

PostgreSQL Container
```

---

## Docker DNS

Suppose

```yaml
services:

  app:

  postgres:
```

Compose automatically registers

```
postgres

↓

Current Container IP
```

Spring Boot connects using

```properties
jdbc:postgresql://postgres:5432/demo
```

instead of

```properties
jdbc:postgresql://localhost:5432/demo
```

---

> [!IMPORTANT]
> Never use `localhost` for communication between containers.
>
> Use the **service name** instead.

---

## depends_on

Specifies container startup order.

Example

```yaml
depends_on:

  - postgres
```

This means

```
Start PostgreSQL Container

↓

Start Spring Boot Container
```

---

> [!WARNING]
> `depends_on` **does not wait** for PostgreSQL to become ready.
>
> It only waits until the PostgreSQL **container starts**.

```
Postgres Container Started

↓

Still Initializing

↓

Database Ready
```

Spring Boot may attempt to connect before PostgreSQL is ready.

Recommended Solutions

- Spring Retry
- Health Checks
- Wait-for scripts
- Kubernetes Readiness Probes

---

## Complete Example

```yaml
version: "3.9"

services:

  app:

    build: .

    ports:
      - "8080:8080"

    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/demo

    depends_on:
      - postgres

  postgres:

    image: postgres:16

    environment:
      POSTGRES_DB: demo
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password

    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:

  postgres-data:
```

Compose automatically performs

```
Create Network

↓

Create Volume

↓

Build Spring Image

↓

Pull PostgreSQL Image

↓

Start PostgreSQL

↓

Start Spring Boot

↓

Register DNS

↓

Application Running
```

---

## Common Commands

Start Application

```bash
docker compose up
```

Run in Background

```bash
docker compose up -d
```

Stop Containers

```bash
docker compose down
```

Rebuild Images

```bash
docker compose up --build
```

View Logs

```bash
docker compose logs
```

Restart Containers

```bash
docker compose restart
```

List Running Services

```bash
docker compose ps
```

Stop Without Removing Containers

```bash
docker compose stop
```

Start Existing Containers

```bash
docker compose start
```

---

## Compose Lifecycle

```
docker compose up

        │

        ▼

Build Images (if required)

        │

        ▼

Create Network

        │

        ▼

Create Volumes

        │

        ▼

Start Containers

        │

        ▼

Register Container DNS

        │

        ▼

Application Ready
```

---

## Compose vs Docker Run

| Docker Run | Docker Compose |
|------------|----------------|
| Single Container | Multiple Containers |
| Manual Network Creation | Automatic |
| Manual Volume Creation | Automatic |
| Manual Environment Variables | YAML Configuration |
| Manual Startup | One Command |
| Good for Small Containers | Good for Applications |

---

## Compose vs Kubernetes

| Docker Compose | Kubernetes |
|----------------|------------|
| Service | Deployment |
| ports | Service |
| volumes | Volumes / PVC |
| environment | env |
| networks | Cluster Networking |
| build | No Equivalent |
| depends_on | No Direct Equivalent |

---

> [!INFO]
> Kubernetes **does not build Docker images**.
>
> Images must already exist in a container registry before deployment.

---

## Typical Spring Boot Architecture

```
                    Docker Compose

                            │

          ┌─────────────────┼─────────────────┐

          ▼                 ▼                 ▼

     Spring Boot      PostgreSQL         Redis

          │                 │                 │

          └─────────────────┼─────────────────┘

                Docker Network

          ┌─────────────────┼─────────────────┐

          ▼                 ▼                 ▼

    uploads Volume    postgres-data    redis-data
```

---

## Best Practices

✅ Use one Compose file per application.

✅ Use service names instead of IP addresses.

✅ Store secrets in `.env` files or secret managers (avoid hardcoding).

✅ Use named volumes for databases.

✅ Use bind mounts only for development.

✅ Keep application configuration inside environment variables.

✅ Add health checks for production-ready services.

---

## Mental Model

Docker Compose doesn't introduce new Docker concepts.

It simply manages existing Docker components together.

```
Docker Compose

        │

        ▼

Images

↓

Containers

↓

Networks

↓

Volumes

↓

Environment Variables
```

Compose is an **application orchestrator for a single Docker host**.

It is essentially a blueprint describing how an entire application should be built, connected, configured and started.

---

## Summary

| Component | Purpose |
|-----------|---------|
| services | Defines containers |
| build | Builds Docker image |
| image | Uses an existing image |
| ports | Publishes ports |
| environment | Passes environment variables |
| volumes | Persistent storage |
| networks | Container communication |
| depends_on | Controls startup order |
| docker compose up | Starts the application |
| docker compose down | Stops and removes containers |