---
title:
  - Docker Basics
tags:
  - cloud
  - fundamentals
  - docker
stack: docker
created: 2026-06-26
---
## Docker Image vs Container

| Image                                                                          | Container                                                                                    |
| ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| Read-only blueprint                                                            | Running instance of an image                                                                 |
| Contains:<br>1. OS<br>2. Java<br>3. Spring boot jar<br>4. Libraries and config | Images are required for creating<br>containers<br><br>One image can have multiple containers |

---
## Dockerfile Instructions

- These are the ones that we would be 95% of the time
### FROM
- FROM - Starts from an existing image
- `FROM eclipse-temurin:21-jdk`  --> Start with Java 21 already installed

### WORKDIR
- Changes current directory
- `WORKDIR /app` --> equivalent to `cd app` if not exist `mkdir app; cd app`

### COPY
- Copies files
- `COPY build/libs/app.jar app.jar`
- Host: builds/libs/app.jar whereas in Container: /app/app.jar

### RUN
- Runs commands while building the image
- `RUN apt-get update` or `RUN chmod +x gradlew` (Executed only during build)

### CMD
- Command executed when container starts
- `CMD ["java", "-jar", "app.jar"]` equivalent to `java -jar app.jar`

### ENTRYPOINT
- Defines the main executable
- `ENTRYPOINT ["java", "-jar", "app.jar"]`
- This vs CMD:
	- docker run image another-command --> this another-command replaces the CMD
	- entrypoint --> always executes, spring boot deployments generally use ENTRYPOINT

### EXPOSE
- The port that container listens
- `EXPOSE 8080` --> does not actually publish the port, publishing happens with `docker run -p 8080:8080`

### ENV
- Environment variables
- `ENV SPRING_PROFILES_ACTIVE=prod`
- Spring can directly read this

### LAYERS
- Every Dockerfile instruction creates a layer
```Dockerfile
FROM
↓
Layer 1

COPY
↓
Layer 2

RUN
↓
Layer 3

CMD
↓
Layer 4
```
- Docker caches layers
- If nothing changed, Docker reuses them
- Huge speed improvement
---
## Docker Cache

```Dockerfile
COPY . .
RUN ./gradlew build
```
- This invalidates everything whenever the source code changes
```Dockerfile
COPY build.gradle .

RUN gradle dependencies

COPY src src

RUN gradle build
```
- Now dependencies are cached
- This is y ordering matters
---
## COPY vs ADD
- Always use COPY
- ADD has extra behavior, download URLs, unpack archives --> ususally unnecessary
---
## .dockerignore
- Exactly like .gitignore
- Without it docker uploads to the Docker daemon, example:
```Dockerfile
.git
.idea
node_modules
build
.gradle
target
logs
```

---

## Build Context
- When you run `docker build .` the '.' is the build context
- Docker can copy the files only present in this context
- `COPY ../secrets.txt .` --> fails 
---
## Volumes
- Container storage disappears when container is removed
- Volumes persist
```
Host
↓
Volume
↓
Container
```
- This is useful for:
	- Uploads 
	- logs
	- database files
---
## Ports
- `docker run -p 80:8080` --> Host - 80 connected to Container 8080

---
## Running as ROOT

```Dockerfile
# this means the user is root not advisable to use
USER root 

# Instead use the following as this limits the impact of compramise inside the container
RUN adduser appuser
USER appuser
```
---
## Health Check
- Docker can periodically verify your application is healthy
- Example:
`HEALTHCHECK CMD curl --fail http://localhost:8080/actuator/health || exit 1`
- This works well for spring boot applications with the actuator health endpoint enabled
---
## Typical Production Flow

```Dockerfile
GitHub
↓
Gradle Build
↓
bootJar
↓
Docker Build
↓
Docker Image
↓
Push to Registry
↓
Cloud VM/Kubernetes
↓
docker pull
↓
docker run
↓
Spring Boot Running
```

---
## Sample file

```Dockerfile
# ---------- Build Stage ----------
FROM gradle:8.9-jdk21 AS builder
WORKDIR /app
COPY . .
RUN gradle bootJar --no-daemon

# ---------- Runtime Stage ----------
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---
## Spring Boot Executable JAR (`bootJar`)

- `gradle bootJar` creates an executable Spring Boot JAR.
- This JAR is **self-contained**, meaning it includes:
	- Your compiled classes
	- Spring Boot Loader
	- All project dependencies
	- Configuration metadata

Typical structure:

```
app.jar
│
├── META-INF/
├── BOOT-INF/
│   ├── classes/
│   │     *.class
│   │
│   └── lib/
│         spring-web.jar
│         spring-security.jar
│         jackson.jar
│         hibernate.jar
│         postgresql.jar
│         ...
│
└── Spring Boot Loader
```

- The dependencies inside `BOOT-INF/lib` are loaded automatically when executing:

```bash
java -jar app.jar
```

**Key Takeaway**

The runtime image does **not** need Gradle or Maven because every required dependency is already packaged inside `app.jar`.

---

## Multi-Stage Build

Builder Stage

- Uses a JDK and build tools (Gradle/Maven).
- Responsibilities:
	- Download dependencies
	- Compile source code
	- Run annotation processors
	- Execute tests (optional)
	- Produce `app.jar`

Runtime Stage

- Uses only a JRE.
- Copies only the generated JAR from the builder stage.
- Contains:
	- Java Runtime
	- `app.jar`

```
Builder Stage
─────────────
Gradle
JDK
Source Code
Dependencies
↓
bootJar
↓
app.jar

COPY --from=builder

Runtime Stage
─────────────
JRE
app.jar
```

Benefits

- Smaller image size
- Faster deployment
- Lower attack surface
- No source code in production image
- No Gradle/Maven in production image

---

## What Happens to Stage 1?

During `docker build`

```
Stage 1
↓
Compile
↓
app.jar
↓
Stage 2
↓
Final Image
```

- Stage 1 is an **intermediate build stage**.
- Docker keeps it locally as part of the build cache.
- It is **not pushed** to the container registry.
- It can be reused for future builds if the cache is still valid.
- It can be removed using:

```bash
docker builder prune
```

or

```bash
docker system prune
```

**Key Takeaway**

The builder stage exists only to produce artifacts (`app.jar`). It is not part of the final production image.

---

## What Gets Pushed to the Registry?

When executing

```bash
docker push my-app:1.0
```

Docker pushes only the **final runtime image**.

Uploaded

```
Linux Layer
↓
Java Runtime Layer
↓
app.jar Layer
↓
Image Metadata
```

Not uploaded

- Source code
- Gradle
- Build cache
- Compilation output directories
- Builder stage

Only the final image is stored in the registry unless the builder stage is explicitly tagged and pushed.

---

## Complete CI/CD Flow

```
Developer
    │
    │ git push
    ▼
GitHub Repository
    │
    ▼
GitHub Actions Runner
    │
    ├── Checkout Source
    │
    ├── Docker Build
    │       │
    │       ├── Stage 1 (Gradle + JDK)
    │       │       │
    │       │       └── bootJar
    │       │               │
    │       │               └── app.jar
    │       │
    │       └── Stage 2 (JRE)
    │               │
    │               └── COPY app.jar
    │
    ├── Final Docker Image
    │
    └── Push to Registry
            │
            ▼
Production Server
            │
            ├── docker pull
            │
            └── docker run
                    │
                    ▼
             java -jar app.jar
                    │
                    ▼
           Spring Boot Running
```

### Important

The production server **never**:

- Compiles Java
- Runs Gradle
- Downloads Maven dependencies

It simply downloads the already-built Docker image and starts the application.

---

## Builder Stage vs Runtime Stage

| Builder Stage | Runtime Stage |
|---------------|---------------|
| Uses JDK | Uses JRE |
| Contains Gradle/Maven | No build tools |
| Compiles code | Runs application |
| Downloads dependencies | Uses dependencies packaged inside `app.jar` |
| Produces executable JAR | Executes `java -jar app.jar` |
| Intermediate image | Final image |

---

## Why Doesn't the Runtime Need Gradle?

Gradle is only a **build tool**.

Its responsibilities end after producing `app.jar`.

```
Gradle
↓
Compile Java
↓
Package dependencies
↓
Create app.jar
↓
Done
```

The executable JAR already contains:

- Application classes
- Third-party libraries
- Spring Boot Loader

Therefore the runtime only needs:

```
Java Runtime
+
app.jar
```

Nothing else.

---

## Docker Image Internals

A Docker image is **not a single file**.

It is composed of multiple **read-only layers** stacked together.

Example:

```Dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

Produces layers similar to:

```
Layer 5 -> ENTRYPOINT
Layer 4 -> EXPOSE
Layer 3 -> COPY app.jar
Layer 2 -> WORKDIR
Layer 1 -> eclipse-temurin:21-jre
```

The final image is simply the combination of all these layers.

### Why Layers?

Layers provide:

- Reusability
- Faster builds
- Faster image downloads
- Efficient storage

If multiple images use the same base image, Docker stores that base layer only once.

Example

```
Image A

Layer 3 -> app A
Layer 2 -> JRE
Layer 1 -> Ubuntu

Image B

Layer 3 -> app B
Layer 2 -> JRE
Layer 1 -> Ubuntu
```

Docker stores

```
Ubuntu
JRE
App A
App B
```

instead of duplicating Ubuntu and JRE.

---

## Image Manifest

Every Docker image contains a **manifest**.

The manifest stores metadata about the image, such as:

- Image name
- Tag
- Architecture
- Operating System
- Ordered list of layer hashes

Example

```
my-app:1.0

Manifest
│
├── Layer Hash 1
├── Layer Hash 2
├── Layer Hash 3
└── Metadata
```

Docker uses the manifest to reconstruct the complete image.

---

## How Registries Store Images

Container registries (Docker Hub, GitHub Container Registry, Amazon ECR, Azure ACR, etc.) do **not** store one giant image file.

They store:

- Image manifest
- Individual layers

```
Registry

Manifest
│
├── Layer A
├── Layer B
├── Layer C
└── Layer D
```

Images are reconstructed by combining the referenced layers.

---

## Layer Hashing

Each layer has a unique content hash.

Example

```
Layer A
↓

SHA256: abc123...

Layer B
↓

SHA256: def456...
```

If the contents of a layer change, its hash changes.

Docker compares hashes to determine whether a layer already exists locally or in the registry.

---

## Why Pushing Images Is Fast

Suppose Version 1 contains:

```
Layer 1 -> Ubuntu
Layer 2 -> Java
Layer 3 -> app.jar (v1)
```

You modify one Java class and rebuild.

Version 2 becomes:

```
Layer 1 -> Ubuntu
Layer 2 -> Java
Layer 3 -> app.jar (v2)
```

Only Layer 3 changed.

When running

```bash
docker push my-app:v2
```

Docker uploads only:

```
Layer 3
```

The registry already contains Layer 1 and Layer 2.

This makes pushing new versions much faster.

---

## Why Pulling Images Is Fast

Suppose the production server already has

```
Ubuntu
Java
```

installed from a previous image.

When executing

```bash
docker pull my-app:v2
```

Docker downloads only the missing layers.

Example

```
Already Present

Ubuntu
Java

Downloaded

app.jar Layer
```

This significantly reduces deployment time.

---

## Docker Build Cache vs Registry Layers

These are different concepts.

### Docker Build Cache

- Stored locally by the Docker Engine.
- Speeds up image builds.
- Reuses previously executed Dockerfile instructions.

Example

```
COPY build.gradle
↓

Cache Hit

RUN gradle dependencies
↓

Cache Hit
```

---

### Registry Layers

- Stored inside a container registry.
- Speeds up pushing and pulling images.
- Shared across machines.

Example

```
Developer Machine

docker push

↓

Registry

↓

Production Server

docker pull
```

---

## Complete Image Lifecycle

```
Developer
    │
    │ docker build
    ▼
Docker Engine
    │
    ├── Executes Dockerfile
    ├── Creates Layers
    ├── Uses Local Build Cache
    └── Produces Final Image
            │
            ▼
docker push
            │
            ▼
Container Registry
            │
            ├── Stores Manifest
            └── Stores Individual Layers
                    │
                    ▼
Production Server
            │
            ├── Downloads Missing Layers
            ├── Reconstructs Image
            └── Starts Container
```

---

## Summary

| Concept         | Purpose                                         |
| --------------- | ----------------------------------------------- |
| Dockerfile      | Instructions for building an image              |
| Layer           | Immutable result of each Dockerfile instruction |
| Image           | Collection of layers                            |
| Manifest        | Metadata describing the image and its layers    |
| Build Cache     | Speeds up local image builds                    |
| Registry Layers | Speeds up image push and pull operations        |
| Container       | Running instance of an image                    |