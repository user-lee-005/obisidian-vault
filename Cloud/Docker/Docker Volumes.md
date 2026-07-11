---
title: Docker Volumes
topic: infra
tags:
  - docker
  - cloud
  - infrastructure
stack: docker
prerequisites:
  - "[[Docker Basics]]"
created: 2026-06-26
---
A container is **ephemeral**, meaning its writable storage is deleted when the container is removed.

```
Image (Read Only)
        │
        ▼
Container
        │
        ▼
Writable Layer
```

Everything written inside the container (logs, uploads, temporary files, generated PDFs, etc.) goes into the writable layer.

When the container is removed:

```
Container
↓
Writable Layer
↓
❌ Deleted
```

To persist data, Docker provides **Volumes**.

```
Host
↓
Docker Volume
↓
Container
```

The volume exists independently of the container.

---

## Why Volumes?

Without a volume

```
Container
↓
/app/uploads
↓
invoice.pdf
```

```
docker rm container
```

```
Container ❌
invoice.pdf ❌
```

With a volume

```
Container
↓
/app/uploads
↓
Docker Volume
↓
invoice.pdf
```

```
docker rm container
```

```
Container ❌
Volume ✔
invoice.pdf ✔
```

The data survives container recreation.

---

## Container Writable Layer

Every running container has a writable layer.

```
Image (Read Only)
↓
Writable Layer
```

The writable layer stores:

- Logs
- Uploaded files
- Temporary files
- Cache
- Generated reports

The writable layer is deleted when the container is deleted.

---

## Named Volume

Docker manages the storage.

Create

```bash
docker volume create postgres-data
```

Mount

```bash
docker run -v postgres-data:/var/lib/postgresql/data
```

```
Docker Volume
postgres-data
        │
        ▼
PostgreSQL Data Folder
```

Use Cases

- PostgreSQL
- MySQL
- MongoDB
- Redis persistence
- User uploads
- Generated PDFs
- Application data

---

## Anonymous Volume

Example

```bash
docker run -v /var/lib/postgresql/data
```

Docker automatically creates an unnamed volume.

Example

```
Volume

87d9ac23...
```

Mostly used for temporary containers.

---

## Bind Mount

Instead of Docker managing storage, the host directory is mounted.

Linux

```bash
docker run -v /home/lee/uploads:/app/uploads
```

Windows

```bash
docker run -v C:/Users/Lee/uploads:/app/uploads
```

```
Host Folder
↓
Container Folder
```

Changes are immediately visible on both sides.

Use Cases

- Local development
- Live code reload
- Configuration files
- Viewing generated files directly

---

## Named Volume vs Bind Mount

| Feature | Named Volume | Bind Mount |
|----------|--------------|------------|
| Managed by Docker | ✅ | ❌ |
| Managed by Host | ❌ | ✅ |
| Good for Production | ✅ | Sometimes |
| Good for Development | Sometimes | ✅ |
| Stores Database Files | ✅ | Possible but uncommon |
| Stores Source Code | ❌ | ✅ |

---

## Sharing Volumes

One volume can be mounted into multiple containers.

```
Shared Volume
        │
 ┌──────┴──────┐
 ▼             ▼
Container A  Container B
```

Useful for

- Shared uploads
- Shared logs
- Shared assets

Applications must handle concurrent writes if multiple containers modify the same files.

---

## Useful Commands

Create

```bash
docker volume create uploads
```

List

```bash
docker volume ls
```

Inspect

```bash
docker volume inspect uploads
```

Remove

```bash
docker volume rm uploads
```

Remove unused volumes

```bash
docker volume prune
```

---

## Typical Production Example

```
	        Docker Host

       Spring Boot Container
               │
               ▼
        uploads Volume

       PostgreSQL Container
               │
               ▼
     postgres-data Volume

         Redis Container
               │
               ▼
        redis-data Volume
```

Each service has its own persistent storage.

---

## Container Storage vs Volumes

| Feature | Container Storage | Docker Volume |
|----------|-------------------|---------------|
| Persists after container deletion | ❌ | ✅ |
| Good for Databases | ❌ | ✅ |
| Good for Uploads | ❌ | ✅ |
| Temporary Files | ✅ | Sometimes |
| Managed by Docker | N/A | ✅ |

---

## Kubernetes Volumes vs Docker Volumes

Both solve the same problem:

> Mount storage into containers.

### Docker

Docker manages the volume.

```
Docker Volume
        │
        ▼
	Container
```

Example

```bash
docker run -v postgres-data:/var/lib/postgresql/data
```

---

### Kubernetes

Kubernetes manages the volume.

```
Volume Source
(ConfigMap / Secret / PVC / emptyDir)

            │
            ▼

         Pod Volume

      ┌─────┴─────┐
      ▼           ▼
Container A   Container B
```

A Pod can mount the same volume into multiple containers.

Common Kubernetes Volume Types

- `emptyDir` → Temporary storage shared by containers in a Pod.
- `ConfigMap` → Mount configuration files.
- `Secret` → Mount sensitive information.
- `PersistentVolumeClaim (PVC)` → Persistent storage backed by disks/cloud storage.
- `hostPath` → Mount a directory from the node (rarely used in production).

---

## Sidecar Pattern Example

A ConfigMap can be mounted into both the application container and a sidecar.

```
ConfigMap
      │
      ▼
Shared Volume
      │
 ┌────┴────┐
 ▼         ▼
Main App  Sidecar
```

Both containers can read the mounted files.

Example use cases

- GCS credentials
- Nginx configuration
- Fluent Bit configuration
- Application configuration

---

## Mental Model

Think of a container like RAM.

```
Container
↓
Writable Layer
↓
Temporary
```

Think of a volume like an SSD.

```
Container
↓
Volume
↓
Persistent Storage
```

Delete the container and recreate it.

The application starts fresh, but the data stored in the volume remains.