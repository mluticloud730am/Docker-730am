# Docker Complete Notes — Volumes, ENTRYPOINT vs CMD, Dockerfile & More
> Covers: Fresher → 13 Years Real-Time Experience Level  
> Session Date: April 30, 2025 | Based on hands-on EC2 (Amazon Linux) lab

---

## Table of Contents
1. [Docker Fundamentals — Why Containers?](#1-docker-fundamentals--why-containers)
2. [Docker Storage Deep Dive — Volumes, Bind Mounts, tmpfs](#2-docker-storage-deep-dive)
3. [Hands-On Walkthrough — What Happened in Your Lab](#3-hands-on-walkthrough)
4. [EBS vs EFS — Cloud Storage for Docker Volumes](#4-ebs-vs-efs--cloud-storage-for-docker-volumes)
5. [ENTRYPOINT vs CMD — The Most Misunderstood Concept](#5-entrypoint-vs-cmd--the-most-misunderstood-concept)
6. [Dockerfile — Complete Reference](#6-dockerfile--complete-reference)
7. [docker history — Understanding Image Layers](#7-docker-history--understanding-image-layers)
8. [Docker Networking Basics](#8-docker-networking-basics)
9. [Docker Swarm — Overview & Why It's Deprecated](#9-docker-swarm--overview--why-its-deprecated)
10. [Kubernetes — Where the Industry Has Gone](#10-kubernetes--where-the-industry-has-gone)
11. [Interview Questions — Fresher to Senior](#11-interview-questions--fresher-to-senior)
12. [Cheat Sheet — All Commands at a Glance](#12-cheat-sheet--all-commands-at-a-glance)

---

## 1. Docker Fundamentals — Why Containers?

### The Core Problem Docker Solves

| Problem | Traditional VM | Docker Container |
|---|---|---|
| Startup time | Minutes | Milliseconds |
| Size | GBs | MBs |
| OS overhead | Full OS per VM | Shared host kernel |
| Portability | Low (image is huge) | High ("it works on my machine" is solved) |
| Density (containers per host) | ~10s | ~100s |

### Container Lifecycle (Mental Model)

```
Dockerfile  →  docker build  →  Image  →  docker run  →  Container
                                  ↑                           |
                              docker push               (ephemeral!)
                                  |                           |
                             Registry (DockerHub)        Data lost on
                             (ECR, ACR, GCR)            container removal
                                                              ↓
                                                    Use Volumes for persistence!
```

### Key Terms

| Term | What it Means |
|---|---|
| Image | Read-only template (blueprint). Like a class in OOP. |
| Container | Running instance of an image. Like an object in OOP. |
| Layer | Each Dockerfile instruction creates an immutable layer. |
| Registry | Remote storage for images (DockerHub, ECR, ACR). |
| Volume | Persistent storage that outlives a container. |
| Bind Mount | Map a specific host directory into the container. |

---

## 2. Docker Storage Deep Dive

### Why Data is Lost Without Volumes

Docker containers use a **writable layer** on top of read-only image layers (Union File System / overlay2). When a container is deleted, its writable layer is also deleted. Everything written inside the container that is NOT in a mounted volume is **gone forever**.

```
┌─────────────────────────────────┐
│  Container Writable Layer       │  ← LOST when container removed
├─────────────────────────────────┤
│  Image Layer 4 (CMD)            │
│  Image Layer 3 (pip install)    │  ← Read-only, shared across containers
│  Image Layer 2 (COPY)           │
│  Image Layer 1 (FROM python)    │
└─────────────────────────────────┘
```

### Three Types of Docker Storage

#### Type 1: Named Volumes (What you used — `vol1`, `vol3`, `vol4`)

Docker manages the location. Data lives in `/var/lib/docker/volumes/<name>/_data` on the host.

```bash
# Create a volume
docker volume create vol1

# Use in container
docker run -it --name=con1 -v vol1:/vol1 ubuntu

# Inspect volume details
docker volume inspect vol1

# List all volumes
docker volume ls

# Remove a volume (only when no container uses it)
docker volume rm vol1

# Remove ALL unused volumes (careful in production!)
docker volume prune
```

**When to use:** Databases (MySQL, PostgreSQL), application state, logs that need to persist.

#### Type 2: Bind Mounts (Host Path Mapping)

You specify the exact host directory. Useful for development — changes on host reflect immediately in container.

```bash
# Syntax: -v /host/path:/container/path
docker run -it -v /home/ec2-user/myapp:/app ubuntu

# Real-world dev workflow: mount source code
docker run -dt -v $(pwd):/app -p 5000:5000 img3
# Now editing app.py on host immediately reflects inside container
```

**When to use:** Local development, sharing config files, injecting secrets from host.

**Important difference from named volumes:**
```
Named Volume:   Docker controls where host path is → /var/lib/docker/volumes/...
Bind Mount:     YOU control the host path → /home/user/project or /etc/nginx
```

#### Type 3: tmpfs Mounts (In-Memory Only)

Data stored in host memory, never written to disk. Lost when container stops.

```bash
docker run -dt --tmpfs /tmp:rw,size=100m ubuntu
```

**When to use:** Sensitive data like secrets, tokens — you don't want them on disk at all. High-speed temporary scratch space.

### Volume Lifecycle — The Critical Point

```bash
# Scenario: con1 writes file1 to vol1. con1 is deleted.
docker rm con1         # Container gone. vol1 still exists!
docker volume ls       # vol1 is still listed

# New container con2 mounts same vol1 → sees file1!
docker run -it --name=con2 -v vol1:/vol1 ubuntu
ls /vol1               # file1 is there!

# This is the whole point of volumes — data outlives containers
```

### What Happened in Your Lab — Data Flow Diagram

```
Host: /var/lib/docker/volumes/vol1/_data/
           │
     ┌─────┴──────┐
     │            │
   con1          con2            (both mount vol1:/vol1)
   Creates      Sees file1
   file1        Creates file2
                               
After both containers exit → file1 & file2 still in vol1/_data on host
```

### Volume Sharing Between Containers

```bash
# Both containers share the SAME volume — real-time sync
docker run -dt --name=writer -v shared_vol:/data ubuntu bash -c "while true; do date >> /data/log.txt; sleep 1; done"
docker run -it --name=reader -v shared_vol:/data ubuntu tail -f /data/log.txt
```

**Real-world use case:** Log aggregator containers reading logs written by app containers.

### Read-Only Volumes (Security Best Practice)

```bash
# Container can read from volume but NOT write
docker run -it -v vol1:/vol1:ro ubuntu

# Useful for: config files, SSL certs, secrets
```

### Volume Drivers — Beyond Local Storage

| Driver | Storage Backend | Use Case |
|---|---|---|
| local (default) | Host filesystem | Development, single-host |
| rexray/ebs | AWS EBS | Production single-AZ persistence |
| rexray/efs | AWS EFS | Multi-host shared storage |
| azure-file | Azure File Share | Azure containers |
| nfs | NFS server | On-premise shared storage |
| vieux/sshfs | Remote SSH host | Simple remote storage |

---

## 3. Hands-On Walkthrough

### What Actually Happened Step-by-Step in Your Lab

**Step 1: Install Docker and create vol1**
```bash
systemctl start docker
systemctl enable docker          # Auto-start on reboot
docker volume create vol1        # Creates /var/lib/docker/volumes/vol1/_data/
```

**Step 2: con1 mounts vol1 and creates file1**
```bash
docker run -it --name=con1 -v vol1:/vol1 ubuntu
# Inside container:
cd /vol1
touch file1         # Written to /var/lib/docker/volumes/vol1/_data/file1 on HOST
```

**Step 3: Verify on host** (while con1 still running or after exit)
```bash
ls /var/lib/docker/volumes/vol1/_data/
# → file1    ← Confirms volume mount is working
```

**Step 4: con2 mounts same vol1 and sees file1 + creates file2**
```bash
docker run -it --name=con2 -v vol1:/vol1 ubuntu
ls /vol1            # → file1  (inherited from vol1)
touch /vol1/file2   # Creates file2
```

**Step 5: con3 uses a DIFFERENT volume name (vol3)**
```bash
docker run -it --name=con3 -v vol3:/vol1 ubuntu
# vol3 is created automatically (Docker creates volumes on demand)
# /vol1 inside con3 is EMPTY — different backing volume
```

**Key lesson here:** The container path `/vol1` is the same in con2 and con3, but they map to different Docker volumes (`vol1` vs `vol3`). Container path ≠ volume name.

**Step 6: Common mistake seen in your lab**
```bash
docker volumes ls     # WRONG — 'volumes' is not a valid subcommand
docker volume ls      # CORRECT — 'volume' (singular)
```

### Volume Naming vs Container Path — Clarification Table

| Container | Volume Name | Container Path | What it Means |
|---|---|---|---|
| con1 | vol1 | /vol1 | vol1 data lives at /vol1 inside container |
| con2 | vol1 | /vol1 | Same vol1 — shares data with con1 |
| con3 | vol3 | /vol1 | Different volume (vol3) — isolated data |
| con6 | vol4 | /volz | vol4 data at /volz path inside container |

---

## 4. EBS vs EFS — Cloud Storage for Docker Volumes

### AWS EBS (Elastic Block Store)

- **Type:** Block storage (like a physical hard drive attached to one EC2)
- **Attachment:** Single EC2 instance at a time (single-host)
- **Use Case:** Database volumes (MySQL data dir, PostgreSQL), single-container stateful apps
- **Docker Volume Driver:** `rexray/ebs` or `docker-volume-rexray`
- **AZ Constraint:** EBS volume and EC2 must be in the **same Availability Zone**

```bash
# Create Docker volume backed by EBS (requires rexray plugin)
docker volume create -d rexray/ebs --name mydb_vol \
  -o size=20 -o volumeType=gp3

docker run -dt --name=mysql_container \
  -v mydb_vol:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8
```

### AWS EFS (Elastic File System)

- **Type:** Network file system (NFS-based)
- **Attachment:** Multiple EC2 instances simultaneously — **multi-host, multi-AZ**
- **Use Case:** Shared files across multiple containers/hosts, media uploads, ML model artifacts
- **Docker Volume Driver:** `rexray/efs` or native NFS bind mount
- **Cost:** More expensive per GB than EBS, but shared

```bash
# Mount EFS directly as bind mount (simpler approach)
# First mount EFS on host:
mount -t efs fs-12345678:/ /mnt/efs

# Then use bind mount
docker run -dt --name=app1 -v /mnt/efs/data:/app/data myimage
docker run -dt --name=app2 -v /mnt/efs/data:/app/data myimage
# app1 and app2 share the same EFS-backed data
```

### EBS vs EFS Decision Tree

```
Need storage for Docker containers?
         │
         ├─ Single container / single host?
         │         └─ Use EBS (faster, cheaper, lower latency)
         │
         └─ Multiple containers across multiple hosts?
                   └─ Use EFS (shared, concurrent access)
                             │
                             ├─ Need very low latency? → EFS with Max I/O
                             └─ Cost sensitive? → S3 + sync script
```

### S3 — Not a Direct Volume but Used in Practice

S3 cannot be mounted as a native Docker volume (it's object storage, not block/file). But in practice:

```bash
# Use AWS CLI inside container to sync to/from S3
docker run -it -e AWS_ACCESS_KEY_ID=xxx -e AWS_SECRET_ACCESS_KEY=yyy \
  amazon/aws-cli s3 sync s3://mybucket/data /app/data
```

---

## 5. ENTRYPOINT vs CMD — The Most Misunderstood Concept

### Quick Mental Model

```
ENTRYPOINT = The executable that ALWAYS runs (the "what" — not easily overridden)
CMD        = Default arguments passed to ENTRYPOINT (easily overridden at runtime)
```

### What Your Lab Showed

**img1 Dockerfile (CMD only):**
```dockerfile
CMD ["python", "app.py"]
```
→ `docker history img1` shows: `CMD ["python" "app.py"]`

**img3 Dockerfile (ENTRYPOINT + CMD):**
```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```
→ `docker history img3` shows: `ENTRYPOINT ["python"]` and `CMD ["app.py"]`

**Why this matters — runtime behavior:**

```bash
# img1 — CMD only
docker run img1                    # Runs: python app.py   (default CMD)
docker run img1 python app1.py     # Runs: python app1.py  (CMD fully replaced)
docker run img1 /bin/bash          # Runs: /bin/bash        (CMD fully replaced)

# img3 — ENTRYPOINT + CMD
docker run img3                    # Runs: python app.py   (ENTRYPOINT + CMD)
docker run img3 app1.py            # Runs: python app1.py  (CMD replaced, ENTRYPOINT stays!)
docker run img3 /bin/bash          # Runs: python /bin/bash (WRONG — /bin/bash passed as arg to python)
                                   # ↑ This is the SyntaxError you saw in your lab!
```

### The SyntaxError Explained

You ran:
```bash
docker run -it img3 /bin/bash
```

img3 has `ENTRYPOINT ["python"]`. So Docker ran:
```
python /bin/bash
```

Python tried to execute the binary `/bin/bash` as a Python script, and since `/bin/bash` starts with binary bytes (not valid UTF-8 Python code), you got:
```
SyntaxError: Non-UTF-8 code starting with '\x82' in file /bin/bash
```

**Fix:** Either override ENTRYPOINT at runtime:
```bash
docker run -it --entrypoint /bin/bash img3
```
Or use `docker exec` on an already-running container:
```bash
docker exec -it <container_id> /bin/bash
```

### Full Comparison Table

| Scenario | Dockerfile | docker run command | What actually runs |
|---|---|---|---|
| CMD only | `CMD ["python","app.py"]` | `docker run img` | `python app.py` |
| CMD only, override | `CMD ["python","app.py"]` | `docker run img python app1.py` | `python app1.py` |
| ENTRYPOINT only | `ENTRYPOINT ["python"]` | `docker run img` | `python` (no args — likely error) |
| ENTRYPOINT + CMD | `ENTRYPOINT ["python"]` `CMD ["app.py"]` | `docker run img` | `python app.py` |
| ENTRYPOINT + CMD, override CMD | `ENTRYPOINT ["python"]` `CMD ["app.py"]` | `docker run img app1.py` | `python app1.py` |
| ENTRYPOINT + CMD, override ENTRYPOINT | `ENTRYPOINT ["python"]` `CMD ["app.py"]` | `docker run --entrypoint /bin/bash img` | `/bin/bash` |

### Shell Form vs Exec Form — Critical Difference

#### Exec Form (Recommended — what you used)
```dockerfile
ENTRYPOINT ["python"]          # JSON array — runs directly, NO shell wrapper
CMD ["app.py"]
```
- PID 1 in the container is your process (`python`)
- Signals (SIGTERM, SIGKILL) go directly to your process
- Proper graceful shutdown

#### Shell Form (Avoid in production)
```dockerfile
ENTRYPOINT python              # Shell form — runs as: /bin/sh -c "python"
CMD app.py
```
- PID 1 is `/bin/sh`, not `python`
- SIGTERM sent to `/bin/sh` — your app may NOT shut down gracefully
- `docker stop` may wait 10s then SIGKILL — bad for databases

### Real-World ENTRYPOINT Patterns

**Pattern 1: Wrapper Script (Most Common in Production)**
```dockerfile
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["app.py"]
```
```bash
# entrypoint.sh
#!/bin/bash
# Perform initialization (wait for DB, set env vars, etc.)
until nc -z $DB_HOST 5432; do
  echo "Waiting for database..."
  sleep 2
done
# Hand off to CMD
exec "$@"
```

**Pattern 2: CLI Tool (kubectl, aws-cli pattern)**
```dockerfile
ENTRYPOINT ["aws"]
CMD ["help"]
```
```bash
docker run myawscli s3 ls         # Runs: aws s3 ls
docker run myawscli ec2 describe  # Runs: aws ec2 describe
```

**Pattern 3: Database Container Pattern (MySQL, PostgreSQL)**
```dockerfile
ENTRYPOINT ["docker-entrypoint.sh"]   # Init script for first-run setup
CMD ["mysqld"]                         # Default: start MySQL daemon
```

---

## 6. Dockerfile — Complete Reference

### Your Dockerfile Explained Line by Line

```dockerfile
FROM python:3.6
# Base image — starts from official Python 3.6 image (~900MB with Debian)
# Note: python:3.6-slim would be ~150MB — always prefer slim for production

MAINTAINER veera "veera.narni232@gmail.com"
# Deprecated instruction — use LABEL instead:
# LABEL maintainer="veera.narni232@gmail.com"

COPY . /app
# Copies everything in build context (.) into /app in image
# Better practice: .dockerignore to exclude node_modules, .git, __pycache__

WORKDIR /app
# Sets working directory for subsequent RUN, CMD, ENTRYPOINT, COPY instructions
# Also the default directory when you exec into the container

RUN pip install -r requirements.txt
# Runs during BUILD time — installs dependencies into image layer
# Each RUN creates a new image layer

EXPOSE 5000
# Documents that the container listens on port 5000
# Does NOT actually publish the port — that's done with -p in docker run

ENTRYPOINT ["python"]
# The executable — always runs. Hard to override accidentally.

CMD ["app.py"]
# Default argument to ENTRYPOINT. Override with: docker run img3 app1.py
```

### Complete Dockerfile Instructions Reference

```dockerfile
FROM ubuntu:22.04                          # Base image (must be first)
ARG BUILD_DATE                             # Build-time variable (not available at runtime)
ENV APP_ENV=production                     # Runtime environment variable
LABEL version="1.0" maintainer="team"     # Metadata (replaces MAINTAINER)
WORKDIR /app                               # Working directory
COPY src/ /app/src/                        # Copy files (preferred over ADD)
ADD archive.tar.gz /app/                   # Copy + auto-extract tar.gz (use sparingly)
RUN apt-get update && apt-get install -y curl   # Execute during build
EXPOSE 8080                                # Document port (informational)
VOLUME ["/data"]                           # Create anonymous volume at build time
USER appuser                               # Switch to non-root user (security!)
HEALTHCHECK --interval=30s CMD curl -f http://localhost:8080/ || exit 1
ENTRYPOINT ["python"]                      # Main executable
CMD ["main.py"]                            # Default args to entrypoint
```

### Best Practices for Production Dockerfiles

**1. Layer Caching — Order matters!**
```dockerfile
# BAD — code changes invalidate pip install cache
COPY . /app
RUN pip install -r requirements.txt

# GOOD — requirements rarely change, code changes often
COPY requirements.txt /app/
RUN pip install -r /app/requirements.txt
COPY . /app
```

**2. Use slim/alpine base images**
```dockerfile
FROM python:3.11-slim   # 150MB vs 900MB for full python:3.11
```

**3. Multi-stage builds — Keep images small**
```dockerfile
# Stage 1: Builder
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# Stage 2: Production image (only copies what's needed)
FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY . .
CMD ["python", "app.py"]
```

**4. Run as non-root**
```dockerfile
RUN adduser --disabled-password appuser
USER appuser
```

**5. Combine RUN commands**
```dockerfile
# BAD — creates 3 layers
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# GOOD — single layer, smaller image
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

**6. Use .dockerignore**
```
# .dockerignore
.git
__pycache__
*.pyc
.env
node_modules
*.log
.DS_Store
```

---

## 7. docker history — Understanding Image Layers

### What You Saw in Your Lab

```bash
docker history img1   # Shows CMD ["python" "app.py"] — combined CMD
docker history img3   # Shows ENTRYPOINT ["python"] + CMD ["app.py"] — separate
```

Each line in `docker history` is a **layer**. Layers with size `0B` are metadata-only (ENV, EXPOSE, CMD, ENTRYPOINT, WORKDIR). Layers with actual size contain file changes.

```bash
# More readable output
docker history img3 --no-trunc          # Show full commands (no truncation)
docker history img3 --format "{{.Size}}\t{{.CreatedBy}}"  # Custom format

# View actual layer contents
docker inspect img3                     # Full JSON metadata
docker inspect img3 --format '{{.RootFS.Layers}}'  # Layer SHA256s
```

### Why Layers Matter — Caching

When you rebuild img3 after only changing `app1.py`:
- Layers 1–3 (FROM, MAINTAINER, COPY if unchanged) → **CACHE HIT** (instant!)
- Layer 4 (COPY . /app) → Cache miss only if files changed
- Layer 5 (pip install) → **CACHE HIT** if requirements.txt unchanged — this is why second build was only 4.7s vs 28.9s

```
First build:  28.9s  (downloading python:3.6 + installing pip packages)
Second build:  4.7s  (cache used for everything except COPY)
```

---

## 8. Docker Networking Basics

### Default Networks

```bash
docker network ls
# bridge    → Default. Containers get their own IP. Access each other by IP or name.
# host      → Container shares host network stack. No port mapping needed.
# none      → No network. Completely isolated.
```

### Port Publishing — Mapping Host to Container

```bash
# -p host_port:container_port
docker run -dt -p 8080:5000 img3
# → http://host_ip:8080 → container port 5000

# Multiple ports
docker run -dt -p 8080:5000 -p 443:443 img3

# Bind to specific host IP
docker run -dt -p 127.0.0.1:8080:5000 img3   # localhost only
```

### Why Your Flask Apps Showed 5000/tcp But Were Not Accessible

```bash
docker ps
# PORTS: 5000/tcp     ← This means EXPOSE 5000, but NO port mapping!
# PORTS: 0.0.0.0:8080->5000/tcp  ← This means actually published
```

To access Flask from outside the EC2:
```bash
docker run -dt -p 5000:5000 img3
# And open port 5000 in EC2 Security Group inbound rules
```

### Container-to-Container Communication

```bash
# Create custom network
docker network create mynet

# Both containers on same network — can reach each other by NAME
docker run -dt --name=flask_app --network=mynet img3
docker run -dt --name=nginx --network=mynet nginx

# Inside nginx container:
curl http://flask_app:5000    # Uses container name as DNS!
```

---

## 9. Docker Swarm — Overview & Why It's Deprecated

### What Docker Swarm Was

Docker Swarm was Docker's built-in container orchestration tool. It allowed you to manage a cluster of Docker hosts (nodes) and deploy services across them.

```bash
# Initialize a swarm (on manager node)
docker swarm init --advertise-addr <MANAGER_IP>

# Join worker nodes
docker swarm join --token <TOKEN> <MANAGER_IP>:2377

# Deploy a service (replicated across nodes)
docker service create --name webapp --replicas 3 -p 80:5000 img3

# Scale up/down
docker service scale webapp=5
```

### Why Swarm is Deprecated / Replaced

| Feature | Docker Swarm | Kubernetes |
|---|---|---|
| Setup complexity | Simple | Complex |
| Learning curve | Low | High |
| Auto-scaling | Basic | Advanced (HPA, VPA) |
| Self-healing | Basic | Advanced |
| Rolling updates | Yes | Yes + canary, blue-green |
| Service discovery | Built-in | Built-in + service mesh |
| Ecosystem/plugins | Limited | Massive (CNCF) |
| Cloud managed | No major support | EKS, AKS, GKE |
| Active development | Minimal | Very active |
| Industry adoption | Declining | Dominant |

**Swarm is still fine for:** Small teams, simple workloads, teams that can't invest in Kubernetes learning.
**Industry reality:** 90%+ of production container orchestration is Kubernetes.

---

## 10. Kubernetes — Where the Industry Has Gone

### Cloud Managed Kubernetes (What "90% of the industry" means)

| Cloud | Service | Free Control Plane? |
|---|---|---|
| AWS | EKS (Elastic Kubernetes Service) | No ($0.10/hr) |
| Azure | AKS (Azure Kubernetes Service) | Yes (free control plane) |
| GCP | GKE (Google Kubernetes Engine) | No (Autopilot free) |
| Red Hat | OpenShift (on-prem + cloud) | No |
| CNCF | k3s (lightweight, edge) | Yes |

### Docker → Kubernetes Mental Map

| Docker Concept | Kubernetes Equivalent |
|---|---|
| Container | Container (inside a Pod) |
| `docker run` | Pod |
| Docker Compose service | Deployment |
| docker volume | PersistentVolume (PV) + PersistentVolumeClaim (PVC) |
| docker network | Service (ClusterIP, NodePort, LoadBalancer) |
| `docker run -e VAR=val` | ConfigMap / Secret |
| Docker Swarm service | Deployment + Service |
| docker-compose.yml | Kubernetes YAML manifests |

### The Career Path

```
Fresher          →   Docker basics, Dockerfile, volumes, networking
Junior DevOps    →   Docker Compose, basics of K8s (pods, deployments)
Mid-level        →   EKS/AKS/GKE, Helm charts, CI/CD with Docker
Senior           →   Service mesh (Istio), GitOps (ArgoCD), K8s operators
13yr Experience  →   Platform engineering, multi-cluster, FinOps, Kubernetes security
```

---

## 11. Interview Questions — Fresher to Senior

### Fresher Level

**Q: What is the difference between a Docker image and a container?**
A: An image is a read-only blueprint (like a class). A container is a running instance of that image (like an object). Multiple containers can run from the same image.

**Q: What happens to data when a container is deleted?**
A: Data inside the container's writable layer is lost. To persist data, you must use Docker volumes or bind mounts, which exist outside the container lifecycle.

**Q: Where does Docker store volumes by default?**
A: `/var/lib/docker/volumes/<volume_name>/_data` on the host filesystem.

**Q: What does EXPOSE do in a Dockerfile?**
A: It documents that the container listens on a port. It does NOT actually publish/open the port. You need `-p host_port:container_port` in `docker run` to actually publish it.

### Mid Level

**Q: What is the difference between ENTRYPOINT and CMD?**
A: ENTRYPOINT defines the main executable (not easily overridden). CMD provides default arguments to ENTRYPOINT (easily overridden at runtime). When both are used together, CMD arguments are passed to the ENTRYPOINT command. CMD alone defines a default command that can be completely replaced.

**Q: Can two containers share the same Docker volume? What are the risks?**
A: Yes. Both containers can read and write to the same volume simultaneously. The risk is race conditions and data corruption if both write to the same files without locking. In practice, this is fine for one-writer/multiple-readers setups.

**Q: What is the difference between `docker stop` and `docker kill`?**
A: `docker stop` sends SIGTERM (graceful shutdown signal), then SIGKILL after 10 seconds. `docker kill` immediately sends SIGKILL. Using exec form in Dockerfile ensures your process receives SIGTERM properly.

**Q: What is a multi-stage Docker build and why would you use it?**
A: It uses multiple `FROM` statements. Earlier stages handle build/compilation. The final stage copies only the resulting artifacts. This keeps the final image small and free of build tools, compilers, and dev dependencies.

### Senior Level

**Q: A containerized app is slow to restart after `docker stop`. What do you investigate?**
A: First check if ENTRYPOINT/CMD uses exec form vs shell form. Shell form means PID 1 is `/bin/sh`, which doesn't forward SIGTERM to the app — so Docker waits 10 seconds before SIGKILL. Also check if the app has a SIGTERM handler that actually terminates it.

**Q: How would you handle database persistence in Kubernetes vs Docker?**
A: In Docker: named volume with EBS driver (single-node) or EFS (multi-node). In Kubernetes: PersistentVolumeClaim backed by an EBS StorageClass (single-AZ) or EFS StorageClass (multi-AZ). Kubernetes StatefulSets are preferred for databases as they provide stable pod names and ordered deployment.

**Q: How do you pass secrets to containers without baking them into the image?**
A: Never store secrets in Dockerfile or image. Options: environment variables (basic, still visible in `docker inspect`), Docker secrets (Swarm), Kubernetes Secrets, AWS Secrets Manager/Parameter Store with IAM roles, HashiCorp Vault. Best practice is IAM role-based access so no secrets are injected at all.

**Q: A production container is consuming excessive memory. How do you limit and investigate?**
A: Set resource limits: `docker run --memory=512m --memory-swap=512m img`. Investigate with `docker stats`, `docker exec <id> top`, check for memory leaks in the app, review application logs. In Kubernetes, use `resources.limits.memory` in pod spec.

---

## 12. Cheat Sheet — All Commands at a Glance

### Images
```bash
docker build -t myimage:v1 .             # Build image
docker build -t myimage:v1 -f Custom.dockerfile .  # Custom Dockerfile
docker images                            # List images
docker image ls                          # Same as above
docker pull ubuntu:22.04                 # Pull from registry
docker push myrepo/myimage:v1            # Push to registry
docker tag myimage:v1 myrepo/myimage:v1  # Tag image
docker rmi myimage:v1                    # Remove image
docker image prune                       # Remove dangling images
docker image prune -a                    # Remove ALL unused images
docker history myimage                   # Show layer history
docker inspect myimage                   # Full metadata JSON
```

### Containers
```bash
docker run -it ubuntu bash              # Interactive terminal
docker run -dt img3                     # Detached (background)
docker run -dt -p 8080:5000 img3        # With port mapping
docker run -dt -v vol1:/data img3       # With volume
docker run -dt -e KEY=VALUE img3        # With env variable
docker run --rm img3                    # Auto-remove on exit
docker ps                               # Running containers
docker ps -a                            # All containers (including stopped)
docker stop con1                        # Graceful stop (SIGTERM)
docker kill con1                        # Force stop (SIGKILL)
docker rm con1                          # Remove stopped container
docker rm -f con1                       # Force remove running container
docker logs con1                        # View logs
docker logs -f con1                     # Follow logs (tail -f)
docker exec -it con1 /bin/bash          # Shell into running container
docker exec con1 env                    # Run command in container
docker stats                            # Live resource usage
docker inspect con1                     # Full container metadata
docker cp con1:/app/file.txt ./         # Copy file from container
```

### Volumes
```bash
docker volume create vol1               # Create named volume
docker volume ls                        # List volumes
docker volume inspect vol1              # Details including mount point
docker volume rm vol1                   # Remove volume (not in use)
docker volume prune                     # Remove all unused volumes
docker run -v vol1:/data img            # Named volume
docker run -v $(pwd):/app img           # Bind mount (current dir)
docker run -v /host/path:/container img # Bind mount (specific path)
docker run -v vol1:/data:ro img         # Read-only volume
```

### Networks
```bash
docker network ls                       # List networks
docker network create mynet             # Create network
docker network inspect bridge           # Network details
docker run --network=mynet img          # Connect to network
docker network connect mynet con1       # Connect running container
docker network disconnect mynet con1    # Disconnect container
```

### System & Cleanup
```bash
docker system df                        # Disk usage breakdown
docker system prune                     # Remove unused: containers, networks, images
docker system prune -a --volumes        # FULL cleanup (CAREFUL in production!)
docker version                          # Client + server version
docker info                             # System-wide info
```

---

## Key Takeaways from Today's Session

1. **Volumes persist data** beyond container lifecycle — stored in `/var/lib/docker/volumes/<name>/_data`
2. **Multiple containers can share a volume** — vol1 was shared between con1 and con2
3. **Docker creates volumes on demand** — `vol3` and `vol4` were auto-created when referenced in `docker run`
4. **`docker volume ls` not `docker volumes ls`** — the subcommand is singular
5. **ENTRYPOINT is the executable, CMD is default args** — together they combine at runtime
6. **The SyntaxError** happened because `ENTRYPOINT ["python"]` in img3 tried to run `/bin/bash` as a Python script
7. **Exec form (`["python", "app.py"]`) is preferred** over shell form for proper signal handling
8. **Layer caching works from top to bottom** — put frequently changing instructions (like COPY code) at the bottom
9. **EBS = single host, EFS = multi-host** for cloud-backed Docker volumes
10. **Kubernetes has replaced Swarm** — learn Docker fundamentals, then move to K8s

---

*Notes compiled: April 30, 2025 | Level: Fresher → 13 Years Experience*
