# 🐳 Docker & Containers -- In-Depth Guide

## 1. Core Idea

Docker ensures applications run consistently across environments by
packaging code, runtime, and dependencies together.

## 2. Under the Hood

-   Namespaces: Isolation
-   Cgroups: Resource limits
-   Union FS: Layered images

## 3. Images

-   Immutable, read-only templates
-   Built using Dockerfile

## 4. Containers

-   Running instance of image
-   Lifecycle: Created → Running → Stopped

## 5. Networking

-   Bridge, Host, None
-   Port mapping: host → container

## 6. Storage

-   Volumes for persistence
-   Bind mounts for host linkage

## 7. Security

-   Shared kernel
-   Use non-root user
-   Scan vulnerabilities

## 8. Best Practices

-   Use small images
-   Optimize layers
-   Use .dockerignore

## 9. CI/CD Flow

Code → Build → Image → Deploy

## 10. Summary

Docker enables consistent, fast, and scalable deployments.
