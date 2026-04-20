# 🐳 Docker - Complete Notes (Conceptual + Practical)

---

## 1. Installation & Setup

### 📖 Concept
Docker is a platform that lets you **package, ship, and run applications in isolated environments called containers**. It needs to be installed and its daemon (background service) must be running before you can use it.

### 🛠️ Practical

```bash
# Install Docker
yum install docker -y

# Start the Docker service (daemon)
systemctl start docker

# Check if Docker is running
systemctl status docker

# Verify installation and version
docker --version
docker version   # Shows both Client and Server details
```

### 🔍 Understanding `docker version` Output

```
Client:                          ← The docker CLI tool you type commands into
  Version: 25.0.14
  API version: 1.44
  OS/Arch: linux/amd64

Server:                          ← The Docker daemon running in background
  Engine:
    Version: 25.0.14
    API version: 1.44
  containerd: ...                ← Low-level container runtime
  runc: ...                      ← Executes container processes
  docker-init: ...               ← Init process inside containers
```

> 💡 **Key Insight:** Docker has two parts — the **CLI (client)** you interact with, and the **daemon (server)** that does the actual work. They talk to each other via the Docker API.

---

## 2. Docker Images

### 📖 Concept
A Docker **image** is a **read-only blueprint/template** for creating containers. Think of it like a class in OOP — the image is the class, and the container is the object (instance).

Images are made of **layers**, and each layer represents a change (install a package, copy a file, etc.). Layers are cached and shared across images to save disk space.

### 🛠️ Practical

```bash
# Pull an image from Docker Hub (default: latest tag)
docker pull ubuntu

# Pull a SPECIFIC version using digest (recommended for production)
docker pull ubuntu@sha256:c4a8d5503...
# ↑ This ensures you get EXACTLY that image, no surprises

# List all locally available images
docker images
```

**Output of `docker images`:**
```
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
ubuntu       latest    0b1ebe5dd426   9 days ago    78.1MB
```

### ⚠️ Important Points

| Point | Explanation |
|-------|-------------|
| Default tag is `latest` | `docker pull ubuntu` = `docker pull ubuntu:latest` |
| `--digest` / full sha pull | Use when you need a guaranteed specific image version |
| **Image is local once pulled** | Even if Docker Hub deletes the image, your local copy still works. No dependency on central repo after pull. |
| Layers are shared | Multiple images sharing the same base layer don't duplicate disk usage |

### 📁 Where Images are Stored on Disk

```bash
/var/lib/docker/          ← Docker's root directory
  containerd/             ← containerd data
  overlay2/               ← Image layer storage (OverlayFS)
  docker/                 ← Docker-specific metadata
```

---

## 3. Docker Containers

### 📖 Concept
A **container** is a **running instance of an image**. It is:
- **Isolated** — has its own filesystem, network, and process space
- **Ephemeral** — changes inside a container are lost when it's removed (unless persisted via volumes)
- **Lightweight** — shares the host OS kernel (unlike VMs which have their own OS)

```
Docker Image  ──(docker run)──►  Container (running instance)
   (Class)                           (Object)
```

### 🛠️ Practical

```bash
# Run a container interactively
docker run -it ubuntu /bin/bash
# -i = interactive (keep STDIN open)
# -t = allocate a pseudo-TTY (terminal)

# Run a container in background (detached)
docker run -dt ubuntu /bin/bash
# -d = detach (run in background)
# -t = allocate TTY so the process stays alive

# Run a command directly in a container
docker run -it ubuntu touch file3
```

---

## 4. Container Lifecycle & Process Dependency

### 📖 Concept
This is **the most important concept in Docker:**

> **A container lives as long as its main process (PID 1) is running. When PID 1 exits, the container stops.**

- If you run `docker run ubuntu` with no command → it runs `/bin/bash` → bash exits immediately (no terminal) → container stops
- If you run `docker run -it ubuntu /bin/bash` → bash stays open because you're interacting with it
- If you run `docker run -dt ubuntu /bin/bash` → bash stays open because `-t` gives it a terminal even in background

### 🛠️ Practical

```bash
# This STAYS running (has interactive terminal)
docker run -it ubuntu /bin/bash

# This also STAYS running (detached but has TTY)
docker run -dt ubuntu /bin/bash

# Inside a running container — only 1 process exists initially
ps -ef
# UID  PID  PPID  CMD
# root   1    0   /bin/bash    ← PID 1 is the main process
# root   9    1   ps -ef
```

> 💡 **Rule:** Minimum **one process must be running** inside a container with an interaction point (terminal or a long-running service). Otherwise the container will exit immediately.

---

## 5. `docker run` Creates a NEW Container Every Time

### 📖 Concept
This is a **very common source of confusion for beginners.**

Every `docker run` command **creates a brand new container** from the image. It does NOT reuse a previous container. So any files/changes made in a previous container are NOT visible in a new one.

```
docker run ubuntu touch file3   → Container A (file3 exists here)  [STOPPED]
docker run ubuntu ls            → Container B (fresh, file3 NOT here) [STOPPED]
```

### 🛠️ Practical

```bash
# Run 1: Creates Container A, touches file3, then exits
docker run -it ubuntu touch file3

# Run 2: Creates Container B — completely fresh — file3 NOT here!
docker run -it ubuntu
root@newcontainer:/# ls       ← file3 is GONE because this is a different container
```

### ✅ How to Fix / Work with the Same Container

**Option 1 — Re-enter the stopped container that has your file:**
```bash
# See all containers (including stopped)
docker ps -a

# Start the stopped container and attach to it
docker start -ai <container_id>
# Now file3 is still there!
```

**Option 2 — Use a Volume to persist data across containers:**
```bash
docker run -it -v mydata:/data ubuntu touch /data/file3
docker run -it -v mydata:/data ubuntu ls /data
# file3 is here! Because both containers share the same volume
```

**Option 3 — Use `docker exec` to enter a RUNNING container:**
```bash
docker exec -it <container_id> /bin/bash
# This enters an ALREADY RUNNING container — not a new one
```

---

## 6. Exiting a Container Without Killing It

### 📖 Concept
When you're inside a container with `docker run -it`, typing `exit` **kills the bash shell (PID 1) → container stops**.

To **detach** (leave the container running in background without stopping it), use the keyboard shortcut:

### 🛠️ Practical

```
Ctrl + P, then Q     ← Detach from container without stopping it
```

```bash
# Start a container
docker run -it ubuntu /bin/bash

# Inside container — do some work
root@container:/# touch importantfile

# Detach WITHOUT killing container
Press: Ctrl+P then Ctrl+Q

# Back on host — container is STILL running
docker ps
# CONTAINER ID   IMAGE    COMMAND       STATUS
# abc123         ubuntu   "/bin/bash"   Up 5 minutes   ← Still running!

# Re-enter the same container
docker exec -it abc123 /bin/bash
```

---

## 7. Managing Containers

### 📖 Concept
Docker gives you commands to view, start, stop, and remove containers at any stage of their lifecycle.

### 🛠️ Practical

```bash
# List RUNNING containers only
docker ps

# List ALL containers (running + stopped)
docker ps -a

# Filter: show only Exited containers
docker ps -a | grep Exited

# Filter: show only Running containers
docker ps -a | grep -v Exited

# Start a stopped container (and attach)
docker start -ai <container_id>

# Stop a running container
docker stop <container_id>

# Remove a stopped container
docker rm <container_id>

# Remove ALL stopped containers at once
docker container prune
```

**Understanding Status in `docker ps -a`:**

```
STATUS
Up 27 minutes          ← Running right now
Exited (0) 13 min ago  ← Stopped cleanly (exit code 0 = success)
Exited (127) 15 min ago ← Stopped with error (127 = command not found)
```

> 💡 **Exit Code 127** means "command not found" — this happened when you tried to run `python` which doesn't exist in the ubuntu image.

---

## 8. `docker exec` — Run Commands in Running Containers

### 📖 Concept
`docker exec` lets you **run an additional command inside an already running container**. It does NOT create a new container. The container must be in running state.

### 🛠️ Practical

```bash
# Open an interactive bash shell in a running container
docker exec -it <container_id> /bin/bash

# Run a single command without interactive shell
docker exec <container_id> ls /tmp

# Inside exec'd shell — you'll see multiple processes
ps aux
# PID 1  = original /bin/bash (from docker run)
# PID 9  = this new /bin/bash (from docker exec)  ← your current session
# PID 17 = ps aux
```

### ❌ Common Mistake

```bash
# This FAILS — container is stopped
docker exec -it 30056636dca8 /bin/bash
# Error: container is not running

# This WORKS — container is running
docker exec -it 6c566c2e3272 /bin/bash
```

### 🔑 `exec` vs `run` vs `attach`

| Command | What it does |
|---------|-------------|
| `docker run` | Creates a **new** container from an image |
| `docker exec` | Runs a command in an **existing running** container |
| `docker attach` | Attaches your terminal to the **main process** of a running container |

---

## 9. Docker Inside Docker? — Not Without Setup

### 📖 Concept
Docker containers are **isolated environments**. They only have what was installed in the image. The `ubuntu` image does NOT have Docker installed inside it.

### 🛠️ Practical

```bash
# Inside a container:
root@6c566c2e3272:/# docker ps
# bash: docker: command not found    ← Docker CLI is not in this container!
```

> 💡 This makes sense — the container has its own isolated filesystem. The host's `docker` binary is not automatically available inside the container.

To use Docker inside Docker (DinD), you'd need to either:
- Mount the Docker socket: `-v /var/run/docker.sock:/var/run/docker.sock`
- Use a Docker-in-Docker image like `docker:dind`

---

## 10. `docker inspect` — Deep Dive into Image/Container Details

### 📖 Concept
`docker inspect` returns **detailed JSON metadata** about an image or container — its configuration, filesystem layers, network settings, mounts, environment variables, etc.

### 🛠️ Practical

```bash
# Inspect an image
docker inspect ubuntu

# Inspect a container
docker inspect <container_id>
```

**Key fields in the output:**

```json
{
  "Id": "sha256:0b1ebe...",          ← Unique image ID
  "RepoTags": ["ubuntu:latest"],     ← Tag names
  "RepoDigests": ["ubuntu@sha256:c4a8d5..."],  ← Immutable digest
  "Architecture": "amd64",           ← CPU architecture
  "Os": "linux",
  "Size": 78140250,                  ← Image size in bytes (~78MB)
  "GraphDriver": {
    "Name": "overlay2",              ← Storage driver used
    "Data": {
      "MergedDir": "/var/lib/docker/overlay2/.../merged",  ← What container sees
      "UpperDir": "/var/lib/docker/overlay2/.../diff",     ← Container's writable layer
    }
  },
  "RootFS": {
    "Layers": ["sha256:538812..."]   ← Image layers
  }
}
```

---

## 11. Images Must Have Required Software

### 📖 Concept
Docker images only contain what was **explicitly installed** during their build. The base `ubuntu` image is minimal — it does NOT include Python, Java, Node.js, etc. by default.

### 🛠️ Practical

```bash
# Trying to run python in ubuntu image FAILS
docker run -it ubuntu python --version
# Error: exec: "python": executable file not found in $PATH

# You must install it first, OR use an image that already has it
docker run -it python:3.11 python --version   ← Use official python image
# Python 3.11.x ← Works!

# Or install manually inside ubuntu container
docker run -it ubuntu /bin/bash
apt-get update && apt-get install -y python3
python3 --version   ← Now works!
```

---

## 12. Summary — Core Docker Concepts

```
┌─────────────────────────────────────────────────────────────┐
│                     DOCKER MENTAL MODEL                     │
├──────────────────┬──────────────────────────────────────────┤
│  Docker Image    │  Blueprint / Template (like a Class)     │
│  Docker Container│  Running instance (like an Object)       │
│  docker pull     │  Download image from registry            │
│  docker run      │  Create + start a NEW container          │
│  docker exec     │  Enter an EXISTING running container      │
│  docker ps       │  List running containers                  │
│  docker ps -a    │  List ALL containers (incl. stopped)     │
│  docker start    │  Start a stopped container               │
│  docker stop     │  Stop a running container                │
│  docker inspect  │  Get detailed metadata (JSON)            │
│  docker images   │  List locally available images           │
│  Ctrl+P then Q   │  Detach from container (keep it running) │
└──────────────────┴──────────────────────────────────────────┘
```

### Key Rules to Always Remember

1. `docker run` = **always a NEW container** — never reuses a previous one
2. Container lives **only as long as PID 1 (main process) is alive**
3. Data inside container is **ephemeral** — lost when container is removed (use volumes to persist)
4. Each container has its **own isolated filesystem** — no sharing between containers by default
5. Images are **immutable** — containers add a writable layer on top
6. `docker exec` only works on **running** containers
7. Use **Ctrl+P+Q** to exit without stopping the container
8. Use `docker pull ubuntu@sha256:...` for **exact reproducible** image versions
9. Once pulled locally, **no dependency on Docker Hub** for that image
10. A container's software is **limited to what's in the image** — base ubuntu has no python!

---

## 13. Volumes — Persisting Data

### 📖 Concept
By default, when a container is removed, all its data is gone. **Volumes** solve this by mounting a storage location that persists independently of the container lifecycle.

### 🛠️ Practical

```bash
# Create a named volume and use it
docker run -it -v mydata:/data ubuntu bash
root@container:/# echo "hello" > /data/test.txt
root@container:/# exit

# Data persists! New container, same volume:
docker run -it -v mydata:/data ubuntu bash
root@newcontainer:/# cat /data/test.txt
# hello   ← Still there!

# Mount host directory into container
docker run -it -v /host/path:/container/path ubuntu bash
```

---

*Notes based on hands-on practice with Docker 25.0.14 on Amazon Linux (kernel 6.1.166) — April 2026*
