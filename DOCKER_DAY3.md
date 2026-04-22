# 🐳 Docker — Day 3: Containers, Images, Processes & Dockerfiles

> **Series:** Docker Zero to Hero | **Day:** 3 of N  
> **Infrastructure:** AWS EC2 (t2.medium, Amazon Linux)  
> **Audience:** Freshers 🟢 → Intermediate 🔵 → Advanced 🔴

---

## 📋 Table of Contents

| # | Topic |
|---|-------|
| 1 | [What is Docker? — Mental Models](#1-what-is-docker--mental-models) |
| 2 | [Docker vs Virtual Machines — The Real Difference](#2-docker-vs-virtual-machines--the-real-difference) |
| 3 | [Setting Up — EC2 + Docker](#3-setting-up--ec2--docker) |
| 4 | [Docker Images — Layers, Pulling, Hub](#4-docker-images--layers-pulling-hub) |
| 5 | [Running Containers — `-it` vs `-dt`](#5-running-containers---it-vs--dt) |
| 6 | [🔑 The Golden Rule — One Process Must Run](#6--the-golden-rule--one-process-must-run) |
| 7 | [Running a Flask App Inside Docker](#7-running-a-flask-app-inside-docker) |
| 8 | [Detach Without Killing — `Ctrl+P, Q`](#8-detach-without-killing--ctrlp-q) |
| 9 | [Monitoring Containers — `docker ps`](#9-monitoring-containers--docker-ps) |
| 10 | [Exec Into Running Containers — `docker exec`](#10-exec-into-running-containers--docker-exec) |
| 11 | [Managing Images — `docker images`](#11-managing-images--docker-images) |
| 12 | [Introduction to Dockerfile](#12-introduction-to-dockerfile) |
| 13 | [Port Mapping — `-p` Flag Explained](#13-port-mapping---p-flag-explained) |
| 14 | [Real-World Scenarios & Industry Use Cases](#14-real-world-scenarios--industry-use-cases) |
| 15 | [Quick Reference Cheat Sheet](#15-quick-reference-cheat-sheet) |

---

## 1. What is Docker? — Mental Models

### 🟢 For Freshers — The Lunchbox Analogy

Imagine you cook biryani at home. It tastes perfect. But when you carry it to a friend's house, they don't have the same spices, vessel, or gas stove. The food experience changes.

**Docker solves this for software.**

Docker is a **lunchbox for your application** — it packs your code, the runtime (Python/Node/Java), all libraries, and config files into one sealed unit called a **container**. No matter where you open that box — your laptop, a colleague's machine, or a cloud server — everything works exactly the same.

> 💡 **One-liner:** Docker = "It works on my machine" → packaged and shipped everywhere.

### 🔵 For Intermediate Developers

Docker is a **containerization platform** that uses two Linux kernel features:

| Kernel Feature | What It Does |
|----------------|-------------|
| **Namespaces** | Isolation — each container has its own filesystem, network, process tree, users |
| **cgroups** (Control Groups) | Resource limits — CPU, memory, I/O per container |

This gives containers the illusion of being separate machines, but they all **share the host OS kernel** — making them far lighter than Virtual Machines.

### 🔴 For Advanced / Interview Prep

The Docker architecture has three core components:

```
┌───────────────────────────────────────────────────┐
│                  Docker Client                    │  ← Your terminal (docker run, build, etc.)
└──────────────────────┬────────────────────────────┘
                       │ REST API
┌──────────────────────▼────────────────────────────┐
│                  Docker Daemon (dockerd)           │  ← Background service managing everything
│   ┌──────────────┐  ┌──────────────┐              │
│   │  Containers  │  │   Images     │              │
│   └──────────────┘  └──────────────┘              │
└───────────────────────────────────────────────────┘
                       │
┌──────────────────────▼────────────────────────────┐
│              Container Runtime (containerd)        │  ← Actually creates/runs containers
│              (uses runc under the hood)            │
└───────────────────────────────────────────────────┘
```

> `docker` CLI → talks to `dockerd` → which delegates to `containerd` → which uses `runc` (OCI-compliant runtime) to actually create Linux namespaces and cgroups.

---

## 2. Docker vs Virtual Machines — The Real Difference

This comparison is asked in **almost every DevOps/Cloud interview.**

```
┌──────────────────────────────────┐   ┌──────────────────────────────────┐
│       VIRTUAL MACHINE            │   │          DOCKER CONTAINER         │
├──────────────────────────────────┤   ├──────────────────────────────────┤
│  App A  │  App B  │  App C       │   │  App A  │  App B  │  App C       │
├─────────┼─────────┼──────────────┤   ├─────────┴─────────┴──────────────┤
│  Guest  │  Guest  │  Guest OS    │   │         Docker Engine             │
│  OS     │  OS     │              │   ├──────────────────────────────────┤
├─────────┴─────────┴──────────────┤   │           Host OS Kernel          │
│         Hypervisor               │   ├──────────────────────────────────┤
├──────────────────────────────────┤   │           Physical Hardware       │
│         Physical Hardware        │   └──────────────────────────────────┘
└──────────────────────────────────┘
  Each VM: full OS = GBs of overhead     Each container: shares kernel = MBs
  Boot time: minutes                     Boot time: milliseconds
  Isolation: hardware-level              Isolation: kernel namespace-level
```

| Parameter | Virtual Machine | Docker Container |
|-----------|----------------|-----------------|
| Boot Time | 1–5 minutes | < 1 second |
| Size | GBs (full OS copy) | MBs (app + deps only) |
| OS | Each VM has its own | All share host kernel |
| Isolation | Strong (hypervisor) | Good (namespaces) |
| Performance | Overhead from hypervisor | Near-native |
| Use Case | Full OS-level isolation | App deployment, microservices |

> 🔴 **Advanced Note:** VMs use Type-1 (bare-metal) or Type-2 (hosted) hypervisors. Docker uses kernel namespaces — meaning if your app has a kernel-level exploit, it can potentially escape the container. This is why security-sensitive workloads sometimes combine both (containers inside VMs — used by AWS Fargate, Google Cloud Run).

---

## 3. Setting Up — EC2 + Docker

### Why AWS EC2 t2.medium?

| Instance | vCPUs | RAM | Monthly Cost | Verdict |
|----------|-------|-----|-------------|---------|
| t2.micro | 1 | 1 GB | Free tier | ❌ Too small — OOM errors with multiple containers |
| t2.small | 1 | 2 GB | ~$17 | ⚠️ Tight for multi-container setups |
| **t2.medium** | **2** | **4 GB** | **~$33** | **✅ Comfortable for learning Docker** |
| t2.large | 2 | 8 GB | ~$67 | ✅ Good for production-like setups |

> 🟢 **Real-world tip:** In production, container orchestrators like Kubernetes have minimum node requirements. A typical worker node starts at 2 vCPUs / 4 GB RAM — matching t2.medium.

### Installing Docker on Amazon Linux

```bash
# Step 1: Update package index
sudo yum update -y

# Step 2: Install Docker
sudo yum install docker -y

# Step 3: Start Docker service right now
sudo systemctl start docker

# Step 4: Auto-start Docker on every reboot
sudo systemctl enable docker

# Step 5: Give your user Docker permissions (avoid sudo every time)
sudo usermod -aG docker ec2-user

# ⚠️ IMPORTANT: Log out and log back in for group change to take effect
exit
# (reconnect via SSH)

# Step 6: Verify everything is working
docker --version
docker info
sudo systemctl status docker
```

#### What does each command actually do?

```
systemctl start docker   → Starts the Docker daemon (dockerd) right now
systemctl enable docker  → Creates a systemd symlink so it auto-starts on reboot
usermod -aG docker       → Adds ec2-user to the "docker" Unix group
                           Docker socket (/var/run/docker.sock) is group-owned by "docker"
                           Group members can access the socket without sudo
```

> 🔴 **Security Note:** Adding a user to the `docker` group is effectively giving them **root-equivalent access** to the host. Any user in the docker group can mount the host filesystem inside a container and escape with full root. In production, use **rootless Docker** or control access via sudo policies.

---

## 4. Docker Images — Layers, Pulling, Hub

### What is a Docker Image?

🟢 **Simple:** An image is like an **ISO file** (or a game disc). It's read-only. When you "run" it, you get a live container — like installing the game.

🔵 **Technical:** An image is an ordered collection of **read-only filesystem layers** stacked using Union Filesystem (OverlayFS). Each layer represents a change — adding files, installing packages, etc.

```
┌─────────────────────────────┐  ← Your app layer (COPY app.py)
├─────────────────────────────┤  ← pip install flask
├─────────────────────────────┤  ← Python runtime layer
├─────────────────────────────┤  ← Base OS layer (ubuntu/debian)
└─────────────────────────────┘  ← Scratch (empty)
          Image Layers (read-only)

When container runs:
┌─────────────────────────────┐  ← Container layer (read-WRITE) — your changes go here
├─────────────────────────────┤  ← Your app layer
├─────────────────────────────┤  ← pip install flask
...                              (all layers below are shared, read-only)
```

> 🔴 **Why this matters:** If 10 containers run from the same image, they all **share** the same read-only layers. Only the top writable layer is unique per container. This saves massive amounts of disk space in production.

### Pulling Images from Docker Hub

```bash
# Pull latest Ubuntu
docker pull ubuntu

# Pull specific version (always prefer pinned versions in production!)
docker pull ubuntu:22.04

# Pull Amazon Linux
docker pull amazonlinux

# Pull Python 3 (includes pip already)
docker pull python:3

# Pull lightweight Python (Alpine-based — much smaller!)
docker pull python:3-alpine

# See all locally available images
docker images
```

### Understanding Pull Output

```
$ docker pull ubuntu

Using default tag: latest            ← No version specified → gets 'latest'
latest: Pulling from library/ubuntu
3b65ec22a9e9: Pull complete          ← Layer 1 downloaded
4f4fb700ef54: Pull complete          ← Layer 2 (cached if shared with another image)
Digest: sha256:a3765b4d7...          ← Cryptographic hash — guarantees image integrity
Status: Downloaded newer image for ubuntu:latest
docker.io/library/ubuntu:latest      ← Full image reference
```

### Image Size Comparison — Why It Matters

```bash
$ docker images

REPOSITORY       TAG        SIZE
ubuntu           latest     77.9 MB    ← Minimal OS
amazonlinux      latest     163 MB     ← Amazon's minimal OS
python           3          917 MB     ← Python + Debian base + build tools
python           3-alpine   51 MB      ← Python + Alpine OS (minimal)
```

> 🔴 **Production Best Practice:** Use Alpine-based images (`python:3-alpine`, `node:alpine`) in production. Smaller images mean:
> - Faster CI/CD pipelines (less to push/pull)
> - Smaller attack surface (fewer packages = fewer CVEs)
> - Lower registry storage costs

### Flask is NOT a Docker image

```
❌ WRONG: docker pull flask          # Flask is a Python package, not a Docker image
✅ CORRECT: docker pull python:3    # Then pip install flask INSIDE the container
```

---

## 5. Running Containers — `-it` vs `-dt`

This is one of the **most commonly misunderstood Docker concepts** — get this right.

### The Flags

| Flag | Full Name | Effect |
|------|-----------|--------|
| `-i` | `--interactive` | Keeps STDIN open — you can type input |
| `-t` | `--tty` | Allocates a pseudo-terminal (proper shell prompt) |
| `-d` | `--detach` | Runs container in background, returns shell to you |

### `-it` — Interactive Foreground Mode

```bash
# Run Ubuntu and drop into a bash shell
docker run -it ubuntu /bin/bash

# Prompt changes to:
root@a3f2c9d1b4e5:/#    ← You are INSIDE the container now
                           The hex string is the container ID
```

```bash
# Run Amazon Linux
docker run -it amazonlinux /bin/bash
bash-4.2#    ← Amazon Linux bash prompt
```

🟢 **What's happening:** You opened a "window" into the container. Your terminal is now connected to the container's shell. The container runs as long as you stay in bash.

🔵 **Technical detail:** `-it` together connects your terminal's STDIN/STDOUT/STDERR to the container's bash process. `/bin/bash` is **PID 1** inside the container — the main process keeping it alive.

### `-dt` — Detached Background Mode

```bash
# Run Amazon Linux in background
docker run -dt amazonlinux

# Output: container ID (full)
b7e3f9a2c1d4e8f6a9b0c3d5e7f1a2b4c5d6e7f8a9b0c1d2

# Your terminal prompt returns immediately — container runs silently in background
ec2-user@ip-172-31-x-x:~$
```

🟢 **What's happening:** Like starting a music app in the background on your phone. The app plays, but you're doing other things.

🔵 **Why `-dt` and not just `-d`?**

```
-d alone: runs in background, but no TTY allocated
          → For bash-based containers, no TTY = no terminal session to hold = container may exit

-t alone: allocates a TTY but you can still see output
-dt:      runs in background WITH a TTY allocated
          → Container stays alive because bash has a terminal to live in
          → You can later exec into it
```

### Side-by-Side Comparison

```
                -it                          -dt
          (Interactive)                  (Detached)
         ┌────────────────┐            ┌─────────────────┐
         │ You are INSIDE │            │ Runs in          │
         │ the container  │            │ BACKGROUND       │
         └────────────────┘            └─────────────────┘
         Terminal: OCCUPIED            Terminal: FREE
         Access: Direct typing         Access: via docker exec
         Container life: = your session Container life: independent
         Best for: Debugging, exploring Best for: Services, apps, daemons
```

### Real-World When to Use What

| Situation | Use |
|-----------|-----|
| Exploring a new base image | `-it` |
| Installing/testing packages manually | `-it` |
| Running a long-lived web server | `-dt` |
| Running a database container | `-dt` |
| CI/CD pipelines running scripts | `-it` (or no flags, just `run`) |
| Debugging a running production container | `docker exec -it` |

---

## 6. 🔑 The Golden Rule — One Process Must Run

> **A Docker container lives only as long as AT LEAST ONE process is running inside it.**  
> When all processes stop → container stops. No exceptions.

### 🟢 Simple Analogy

A container is like a **campfire**. The fire burns as long as there's wood (a process). The moment the last log burns out (last process ends), the fire dies. Docker doesn't have a "keep-alive heartbeat" — it only tracks processes.

### 🔵 Technical Explanation

Every container has a **PID 1** — the first process started inside it. Docker monitors PID 1:
- **PID 1 exits** → Docker sends `SIGTERM` to all other processes → container stops
- Unlike a normal Linux system, there is no init system (systemd/upstart) inside a container by default
- The container IS the process. Nothing more.

### Demonstrating the Rule

```bash
# ✅ WORKS: bash is the process keeping container alive
docker run -it ubuntu /bin/bash
root@abc123:/# pwd           # Container running — bash is alive
root@abc123:/# exit          # bash exits → container STOPS

# ✅ WORKS: sleep infinity is the process (useful for debugging)
docker run -dt ubuntu sleep infinity
# Container stays alive because sleep infinity never finishes

# ⚠️ STOPS IMMEDIATELY: No process to keep alive
docker run ubuntu
# Starts → nothing to run → exits in milliseconds

# ⚠️ STOPS AFTER COMMAND: echo runs and finishes
docker run ubuntu echo "hello"
# Output: hello
# Container: Exited (0) — done, gone

# Proof — see all containers including stopped ones
docker ps -a
# CONTAINER ID   IMAGE   COMMAND         STATUS
# a1b2c3d4       ubuntu  "echo hello"    Exited (0) 30 seconds ago
```

### Common Pitfall for Freshers

```bash
# ❌ WRONG: You think this keeps the container alive
docker run ubuntu /bin/bash
# Container immediately exits! Why?
# Because docker run without -it has no STDIN — bash sees no input and exits.

# ✅ CORRECT
docker run -it ubuntu /bin/bash   # Interactive: bash stays alive
docker run -dt ubuntu /bin/bash   # Detached: bash stays alive in background
```

### Real-World Implication

```
In production Kubernetes environments:
→ Pod = wrapper around a container
→ If the main container process crashes → Pod goes into CrashLoopBackOff
→ Kubernetes restart policy kicks in
→ This is why apps must handle signals (SIGTERM) gracefully before exiting

Example: A Flask app should catch SIGTERM and close DB connections cleanly
         before exiting, so Docker/Kubernetes gets a clean shutdown.
```

---

## 7. Running a Flask App Inside Docker

### Context: What We Built

A Python web application using Flask that runs inside a `python:3` container.

### The App — `app.py`

```python
# app.py — Simple Flask Web Application
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from Docker! 🐳"

@app.route('/health')
def health():
    return {"status": "healthy"}, 200

if __name__ == '__main__':
    # 0.0.0.0 = listen on ALL interfaces (required for Docker networking)
    # 127.0.0.1 (default) would only be accessible INSIDE the container
    app.run(host='0.0.0.0', port=5000, debug=True)
```

### Step-by-Step: Running Flask in Docker

```bash
# Step 1: Start a Python 3 container interactively
docker run -it python:3 /bin/bash

# You're now INSIDE the container:
root@7f3a9c2b1d4e:/#

# Step 2: Install Flask (not included in python:3 image — it's a 3rd party package)
pip install flask

# Step 3: Create app.py directly inside the container
cat > app.py << 'EOF'
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello from Docker! 🐳"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Step 4: Run the app
python app.py

# Expected Output:
#  * Serving Flask app 'app'
#  * Running on all addresses (0.0.0.0)
#  * Running on http://127.0.0.1:5000
#  * Running on http://172.17.0.2:5000    ← Container's internal IP
#  Press CTRL+C to quit
```

### Why `host='0.0.0.0'`? — Critical Concept

```
WITHOUT 0.0.0.0 (default: 127.0.0.1):

  [ Your Browser ]
        │
  [ EC2 Host: 172.31.x.x ]
        │
  [ Container: 172.17.0.2 ]
        │
  Flask listens on 127.0.0.1 (container's own localhost)
        ✗ Traffic from outside container can NEVER reach Flask

WITH 0.0.0.0:

  [ Your Browser ]
        │
  [ EC2 Host: 172.31.x.x ] ──(port mapping)──▶ [ Container: 172.17.0.2 ]
                                                        │
                                                Flask listens on 0.0.0.0
                                                (ALL interfaces — accepts all incoming)
                                                        ✓ Traffic reaches Flask!
```

### What Happens to the Container When Flask Runs?

```
Before app.py: PID 1 = /bin/bash
After python app.py: bash spawns python as a child process

Process tree inside container:
  bash (PID 1)
    └── python app.py (PID 12)  ← Flask is running here
                                   Container alive because bash (PID 1) is alive
```

---

## 8. Detach Without Killing — `Ctrl+P, Q`

### The Problem

You're inside a container with Flask running. If you type `exit` or press `Ctrl+D`, bash exits → container stops → Flask dies. You need a way to **step out without stopping anything**.

### The Solution

```
Keyboard shortcut:  Ctrl+P  →  then  Q
```

```bash
# You're inside container, Flask is running
root@7f3a9c2b1d4e:/# python app.py
 * Running on http://0.0.0.0:5000

# Press Ctrl+P, then Q
[detached]

# You're back on EC2 host!
ec2-user@ip-172-31-x-x:~$

# Flask is STILL running inside the container — verify with:
docker ps
# CONTAINER ID   IMAGE      COMMAND          STATUS
# 7f3a9c2b1d4e   python:3   "/bin/bash"      Up 12 minutes  ← Still running!
```

### How It Works — Under the Hood

🟢 **Simple:** Like pressing the Home button on your phone. The app keeps running, you just exit to the home screen.

🔴 **Technical:** `Ctrl+P, Q` sends an **escape sequence** to the Docker client, which disconnects the **pseudo-TTY** from your terminal session. The container's processes receive **no signal** — SIGTERM is never sent. The container, bash, and Flask continue running in their namespace, completely unaffected.

### All Exit Methods Compared

| Action | Keys / Command | What Happens to Container |
|--------|---------------|--------------------------|
| Proper exit | `exit` or `Ctrl+D` | ⛔ Container STOPS (bash exits → PID 1 gone) |
| Detach safely | `Ctrl+P, Q` | ✅ Container KEEPS RUNNING |
| Stop gracefully | `docker stop <id>` | ⛔ Sends SIGTERM → waits 10s → SIGKILL |
| Kill immediately | `docker kill <id>` | ⛔ Sends SIGKILL immediately (no cleanup) |
| Pause | `docker pause <id>` | ⏸️ Freezes container (SIGSTOP to all processes) |

> 🔴 **`docker stop` vs `docker kill`:** In production, always use `docker stop`. It sends `SIGTERM` first, giving your app time to flush logs, close DB connections, and finish in-flight requests. `docker kill` is SIGKILL — hard stop, no cleanup, potential data corruption.

---

## 9. Monitoring Containers — `docker ps`

### `docker ps` — Running Containers Only

```bash
docker ps
```

```
CONTAINER ID   IMAGE        COMMAND            CREATED         STATUS          PORTS     NAMES
a3f2c9d1b4e5   ubuntu       "/bin/bash"        10 mins ago     Up 10 minutes             eager_tesla
b7e3f9a2c1d4   amazonlinux  "/bin/bash"        8 mins ago      Up 8 minutes              jovial_curie
c9f1d2e4a5b6   python:3     "python app.py"    5 mins ago      Up 5 minutes              flappy_hawk
```

### Reading Each Column

| Column | What It Means | Example |
|--------|--------------|---------|
| `CONTAINER ID` | First 12 chars of full 64-char ID | `a3f2c9d1b4e5` |
| `IMAGE` | Source image name:tag | `python:3` |
| `COMMAND` | **The process keeping it alive** | `python app.py` |
| `CREATED` | How long ago it started | `10 minutes ago` |
| `STATUS` | `Up` = alive, `Exited` = stopped | `Up 10 minutes` |
| `PORTS` | Host→Container port mapping | `0.0.0.0:5000->5000/tcp` |
| `NAMES` | Auto-generated (or `--name` flag) | `eager_tesla` |

> 🔵 **The COMMAND column is your best friend for debugging.** It tells you exactly which process is keeping the container alive. If a container exits unexpectedly, this is the first thing to check.

### `docker ps -a` — ALL Containers (Including Stopped)

```bash
docker ps -a
```

```
CONTAINER ID   IMAGE     COMMAND           STATUS
a3f2c9d1b4e5   ubuntu    "/bin/bash"       Up 10 minutes         ← Running
d4e5f6a7b8c9   ubuntu    "echo hello"      Exited (0) 2 min ago  ← Stopped normally
e5f6a7b8c9d0   python:3  "python app.py"   Exited (1) 5 min ago  ← Crashed (exit code 1)
```

> 🔵 **Exit codes matter:**
> - `Exited (0)` → Process finished successfully (expected)
> - `Exited (1)` → Process crashed with error
> - `Exited (137)` → Killed by SIGKILL (OOM or docker kill)
> - `Exited (143)` → Killed by SIGTERM (docker stop)

### Useful `docker ps` Flags

```bash
docker ps               # Running containers
docker ps -a            # All containers (running + stopped)
docker ps -q            # Quiet mode: only container IDs (useful in scripts)
docker ps --no-trunc    # Show full container ID (64 chars)
docker ps -l            # Last created container
docker ps -n 5          # Last 5 containers

# Practical script: Stop all running containers
docker stop $(docker ps -q)

# Remove all stopped containers
docker rm $(docker ps -aq -f status=exited)
```

---

## 10. Exec Into Running Containers — `docker exec`

### What is `docker exec`?

`docker exec` lets you **run a new command inside an already-running container** — without stopping or interrupting it.

```bash
# Syntax
docker exec [OPTIONS] <CONTAINER_ID or NAME> <COMMAND>
```

### Examples from Class

```bash
# First, find the running container's ID
docker ps

# Get an interactive shell inside a running container
docker exec -it b7e3f9a2c1d4 /bin/bash

# You're now inside — a second session alongside Flask/bash
bash-4.2#

# Other useful exec commands:
docker exec b7e3f9a2c1d4 ps aux          # See processes running inside
docker exec b7e3f9a2c1d4 env             # See environment variables
docker exec b7e3f9a2c1d4 cat /etc/os-release  # Check OS version
docker exec -it b7e3f9a2c1d4 python3    # Open Python REPL inside container
docker exec -it b7e3f9a2c1d4 yum install -y curl  # Install package in running container
```

### `docker run` vs `docker exec` — Critical Difference

| | `docker run` | `docker exec` |
|---|---|---|
| **Creates** | A brand new container | No new container |
| **Works on** | An image | An **already-running** container |
| **Result** | New isolated environment | New process inside existing namespace |
| **Use when** | Starting fresh | Debugging live containers |
| **Requires running container?** | No (creates one) | **YES** |

### Why Does `exec` Require a Running Container?

🔴 **Technical:** `docker exec` works by injecting a new process into the **existing Linux namespaces** of the running container (PID namespace, network namespace, mount namespace, etc.). If the container is stopped, those namespaces no longer exist — there's nothing to inject into. This is a direct consequence of the Golden Rule.

### Real-World Use Case — Production Debugging

```bash
# Scenario: Your Flask API returns 500 errors in production
# You don't want to restart the container (might lose state/logs)

# Step 1: Find the container
docker ps
# c9f1d2e4a5b6   python:3   "python app.py"   Up 2 hours   myflaskapp

# Step 2: Get inside without stopping it
docker exec -it c9f1d2e4a5b6 /bin/bash

# Step 3: Debug inside
root@c9f1d2e4a5b6:/app# cat app.log         # Read logs
root@c9f1d2e4a5b6:/app# env | grep DB       # Check DB connection vars
root@c9f1d2e4a5b6:/app# curl localhost:5000  # Test from inside the container
root@c9f1d2e4a5b6:/app# python -c "import flask; print(flask.__version__)"

# Step 4: Detach without stopping
Ctrl+P, Q
```

---

## 11. Managing Images — `docker images`

```bash
docker images
# or
docker image ls
```

```
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
ubuntu        latest    b1d9df8ab815   2 weeks ago    77.9MB
amazonlinux   latest    f6be4802f3f4   3 weeks ago    163MB
python        3         a7d3b7f9c123   1 week ago     917MB
python        3-alpine  d2b4f6e8a1c3   1 week ago     51MB
```

### Useful Image Commands

```bash
# List all images
docker images

# Remove a specific image
docker rmi ubuntu

# Remove image by ID
docker rmi b1d9df8ab815

# Remove ALL unused images (not used by any container)
docker image prune

# Remove ALL images (dangerous — careful in production!)
docker image prune -a

# Inspect image layers and metadata
docker inspect ubuntu

# See image layer history
docker history python:3

# Tag an image (rename/version it)
docker tag python:3 my-python:v1.0
```

### Understanding Image Tags

```bash
# These are all the SAME image (if no custom tag defined):
docker pull ubuntu           # ubuntu:latest
docker pull ubuntu:latest    # same
docker pull ubuntu:22.04     # specific version — different image

# In production: ALWAYS pin to specific versions
docker pull python:3.11.4-slim    # ✅ Reproducible
docker pull python:latest         # ❌ Might break on next release
```

---

## 12. Introduction to Dockerfile

> A **Dockerfile** is a plain text script that automates image creation. Instead of manually pulling images, installing packages, and copying files every time, you define it once — and `docker build` repeats it perfectly every time.

### 🟢 Simple Analogy

If an image is a cake, the Dockerfile is the **recipe card**. Anyone can follow the recipe and bake the exact same cake. No more "I don't know what ingredients I used" situations.

### The Dockerfile from Class

```dockerfile
# ─────────────────────────────────────────────────
# Dockerfile: Runs a Flask app on Amazon Linux
# Build: docker build -t my-flask-app .
# Run:   docker run -dt -p 5000:5000 my-flask-app
# ─────────────────────────────────────────────────

# 1️⃣ FROM — Start from a base image
#    Every Dockerfile MUST start with FROM
#    Think of it as: "Begin with this OS/runtime"
FROM amazonlinux:latest

# 2️⃣ RUN — Execute commands during BUILD time (not runtime)
#    Each RUN creates a new image layer
#    Combine commands with && to reduce layers
RUN yum update -y && yum install -y python3 pip

# 3️⃣ WORKDIR — Set the working directory inside the image
#    Like doing: mkdir /app && cd /app
#    All subsequent commands run relative to this path
WORKDIR /app

# 4️⃣ COPY — Copy files from your HOST machine into the image
#    Syntax: COPY <source-on-host> <dest-in-image>
#    The "." means current WORKDIR (/app)
COPY app.py .

# 5️⃣ RUN again — Install Python dependencies
#    This runs pip install inside the image during build
RUN pip install flask

# 6️⃣ CMD — The default command when a container STARTS
#    This is the process that will be PID 1
#    Using JSON array format ["cmd", "arg"] is recommended (exec form)
CMD ["python3", "app.py"]
```

### Dockerfile Instructions Reference

| Instruction | When It Runs | Purpose | Example |
|-------------|-------------|---------|---------|
| `FROM` | Build time | Base image | `FROM python:3.11-slim` |
| `RUN` | Build time | Install/configure | `RUN apt-get install -y curl` |
| `COPY` | Build time | Copy local files → image | `COPY app.py /app/` |
| `ADD` | Build time | Like COPY + can extract tarballs | `ADD archive.tar.gz /app/` |
| `WORKDIR` | Build time | Set working directory | `WORKDIR /app` |
| `ENV` | Build & Runtime | Set environment variables | `ENV PORT=5000` |
| `EXPOSE` | Documentation | Document which port app uses | `EXPOSE 5000` |
| `CMD` | Runtime | Default command to run | `CMD ["python3", "app.py"]` |
| `ENTRYPOINT` | Runtime | Fixed command (CMD appended to it) | `ENTRYPOINT ["python3"]` |
| `ARG` | Build time only | Build-time variables | `ARG VERSION=1.0` |
| `VOLUME` | Runtime | Declare mount points | `VOLUME ["/data"]` |
| `USER` | Build & Runtime | Set user to run as | `USER appuser` |

### `CMD` vs `ENTRYPOINT` — Common Interview Question

```dockerfile
# CMD: Default command. Can be overridden at docker run.
CMD ["python3", "app.py"]
# docker run myapp             → runs: python3 app.py
# docker run myapp /bin/bash   → runs: /bin/bash  (CMD is replaced)

# ENTRYPOINT: Fixed command. Arguments are appended.
ENTRYPOINT ["python3"]
CMD ["app.py"]               # Default argument
# docker run myapp            → runs: python3 app.py
# docker run myapp other.py   → runs: python3 other.py  (CMD is replaced, ENTRYPOINT stays)
```

### Building and Running Your Image

```bash
# Build the image
# -t = tag (name:version)
# . = build context (look for Dockerfile in current directory)
docker build -t my-flask-app:1.0 .

# Watch the build output — each step = one layer:
# Step 1/6 : FROM amazonlinux:latest
# Step 2/6 : RUN yum update -y && yum install -y python3 pip
# ...

# Run a container from your image
docker run -dt -p 5000:5000 my-flask-app:1.0

# Access the app
curl http://localhost:5000
# Or in browser: http://<EC2-Public-IP>:5000
```

### Layer Caching — Why Order in Dockerfile Matters

```dockerfile
# ❌ BAD: Copying code before installing dependencies
COPY . .
RUN pip install flask       # If ANY file changes, this re-runs
                             # Reinstalls flask every time you change one line of code

# ✅ GOOD: Install dependencies first, then copy code
COPY requirements.txt .
RUN pip install -r requirements.txt   # Cached until requirements.txt changes
COPY . .                              # Only code changes bust this layer
```

> 🔴 **Docker caches layers.** If a layer's instruction hasn't changed, Docker reuses the cached result. Put things that change LEAST OFTEN at the top. Your source code changes most often — put `COPY . .` near the bottom.

---

## 13. Port Mapping — `-p` Flag Explained

```bash
docker run -dt -p 5000:5000 my-flask-app
#               │     │
#               │     └── Container port (Flask listens here)
#               └───────── Host port (EC2/your machine)
```

### How Port Mapping Works

```
Internet Browser
      │
      │ requests http://<EC2-IP>:5000
      ▼
┌─────────────────────────────┐
│  EC2 Host                   │
│  Network Interface: 5000    │◄──── -p 5000:5000 creates this mapping
│         │                   │
│    ┌────▼────────────────┐  │
│    │  Docker Container   │  │
│    │  Flask: 0.0.0.0:5000│  │
│    └─────────────────────┘  │
└─────────────────────────────┘
```

### Port Mapping Examples

```bash
# Same port on host and container
docker run -p 5000:5000 my-flask-app

# Different ports (host:container)
docker run -p 8080:5000 my-flask-app   # Access on :8080, Flask runs on :5000

# Map to all interfaces (default) vs specific IP
docker run -p 5000:5000 my-flask-app          # All interfaces: 0.0.0.0:5000
docker run -p 127.0.0.1:5000:5000 my-flask-app  # Only localhost (not public)

# Multiple port mappings
docker run -p 5000:5000 -p 443:443 my-flask-app

# Random host port (Docker picks)
docker run -p :5000 my-flask-app      # Check with docker ps to see which port
```

> ⚠️ **Security Note:** Always configure AWS Security Groups to allow only the ports you need. Even if Docker maps a port, if the Security Group doesn't allow it, traffic won't reach your EC2.

---

## 14. Real-World Scenarios & Industry Use Cases

### Scenario 1: "Works on my machine" Problem — Solved

```
Developer A (Mac):         Developer B (Windows):      Production (Linux):
Python 3.9                 Python 3.11                 Python 3.8
Flask 2.0                  Flask 2.3                   Flask 1.1
Different SQLite version    MySQL installed             PostgreSQL

Result: App behaves         Result: Different bugs      Result: Crashes in prod
differently on each machine

❌ Without Docker: 3 different environments = 3 different problems

✅ With Docker:
Everyone uses: docker run -it my-flask-app
→ Same Python 3.11, same Flask 2.3, same everything
→ "It works the same everywhere"
```

### Scenario 2: Microservices Architecture

```
E-commerce App (Traditional)          E-commerce App (Docker)
────────────────────────────          ───────────────────────
One giant server with:                Multiple containers:
- User service (Python)               [user-service] python:3 container
- Product service (Node.js)           [product-service] node:18 container
- Payment service (Java)              [payment-service] openjdk:17 container
- Database (MySQL)                    [mysql] mysql:8 container
- Cache (Redis)                       [redis] redis:7 container

Problems:                             Benefits:
- One crash = everything down         - Each service isolated
- Can't scale individual services     - Scale product-service × 5 during sale
- Language upgrade breaks everything  - Update each service independently
```

### Scenario 3: CI/CD Pipeline

```
Developer pushes code to GitHub
         │
         ▼
GitHub Actions / Jenkins starts:
         │
         ├── docker build -t myapp:$GIT_COMMIT .    # Build fresh image
         ├── docker run myapp:$GIT_COMMIT pytest     # Run tests in container
         ├── docker push myregistry/myapp:$GIT_COMMIT  # Push to registry
         │
         ▼
Kubernetes pulls image, deploys new containers
         │
         ▼
Zero-downtime rolling update: old containers replaced one by one
```

### Scenario 4: `docker exec` for Production Debugging

```bash
# Production: Flask app is giving 500 errors but still running

# 1. Find it
docker ps | grep flask-app
# abc123   flask-app   "python app.py"   Up 6 hours

# 2. Get inside without restarting
docker exec -it abc123 /bin/bash

# 3. Investigate
root@abc123:/app# python -c "import psycopg2; print('DB module OK')"
root@abc123:/app# env | grep DATABASE_URL  # Check if DB URL is set
root@abc123:/app# curl -s localhost:5000/health  # Internal health check
root@abc123:/app# tail -f app.log  # Watch live logs

# 4. Fix (e.g., set a missing env var temporarily)
root@abc123:/app# export DATABASE_URL="postgresql://..."
root@abc123:/app# python app.py  # Restart manually from inside

# 5. Exit without killing anything
Ctrl+P, Q
```

### Scenario 5: Data Science / ML Team

```bash
# Share exact Jupyter environment with reproducible results
docker pull jupyter/scipy-notebook

docker run -dt \
  -p 8888:8888 \
  -v $(pwd)/notebooks:/home/jovyan/work \   # Mount local notebooks
  jupyter/scipy-notebook

# Team member gets a URL with token → exact same Python/pandas/numpy/sklearn
# No "conda env export" confusion, no version mismatches
```

---

## 15. Quick Reference Cheat Sheet

### Container Lifecycle

```bash
# Create and start
docker run -it ubuntu /bin/bash          # Interactive foreground
docker run -dt ubuntu                    # Detached background
docker run --name mycontainer -dt ubuntu # Named container
docker run -dt -p 8080:5000 myapp        # With port mapping

# Start/Stop/Restart
docker start <container_id>              # Restart stopped container
docker stop <container_id>               # Graceful stop (SIGTERM)
docker restart <container_id>            # Stop + Start
docker kill <container_id>               # Immediate stop (SIGKILL)
docker pause <container_id>              # Freeze container
docker unpause <container_id>            # Unfreeze container

# Remove
docker rm <container_id>                 # Remove stopped container
docker rm -f <container_id>              # Force remove (even if running)
docker rm $(docker ps -aq)               # Remove ALL stopped containers
```

### Image Commands

```bash
docker images                           # List all local images
docker pull ubuntu:22.04                # Pull specific version
docker push myrepo/myimage:1.0          # Push to registry
docker build -t myapp:1.0 .             # Build from Dockerfile
docker rmi ubuntu                       # Remove image
docker image prune                      # Remove unused images
docker inspect ubuntu                   # Full metadata
docker history python:3                 # Layer history
docker save ubuntu > ubuntu.tar         # Export image to file
docker load < ubuntu.tar                # Import image from file
```

### Monitoring & Debugging

```bash
docker ps                               # Running containers
docker ps -a                            # All containers
docker ps -q                            # Only IDs
docker logs <container_id>              # View stdout/stderr logs
docker logs -f <container_id>           # Follow logs (live)
docker logs --tail 100 <container_id>   # Last 100 lines
docker stats                            # Live CPU/memory stats for all containers
docker stats <container_id>             # Stats for specific container
docker top <container_id>               # Processes inside container
docker inspect <container_id>           # Full container metadata (JSON)
docker diff <container_id>              # Files changed vs base image
```

### Exec & Interaction

```bash
docker exec -it <id> /bin/bash          # Get shell inside running container
docker exec <id> <command>              # Run single command
docker attach <id>                      # Reattach to detached container
# Ctrl+P, Q                             # Detach without stopping
```

### Dockerfile Quick Build

```bash
docker build -t myapp:latest .          # Build from current directory
docker build -t myapp:1.0 -f MyDockerfile .  # Specific Dockerfile name
docker build --no-cache -t myapp .      # Force rebuild all layers
docker build --build-arg VERSION=1.2 . # Pass build args
```

### Cleanup (Housekeeping)

```bash
docker system df                        # Disk usage by Docker
docker system prune                     # Remove all unused resources
docker system prune -a                  # Remove EVERYTHING unused (including images)
docker container prune                  # Remove stopped containers
docker image prune -a                   # Remove unused images
docker volume prune                     # Remove unused volumes
docker network prune                    # Remove unused networks
```

---

## 🏗️ Architecture Summary — Everything Together

```
┌─────────────────────────────────────────────────────────────────┐
│                          Your Code                              │
│                        app.py + deps                            │
└────────────────────────────┬────────────────────────────────────┘
                             │  COPY (Dockerfile)
┌────────────────────────────▼────────────────────────────────────┐
│                       Docker Image                              │
│   [OS Layer] → [Runtime Layer] → [Deps Layer] → [App Layer]     │
│   (Read-only, stackable, shareable, cached)                     │
└────────────────────────────┬────────────────────────────────────┘
                             │  docker run
┌────────────────────────────▼────────────────────────────────────┐
│                      Docker Container                           │
│   Image layers (read-only) + Container layer (read-write)       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  PID 1: python app.py                                   │   │
│   │  (The Golden Rule: this process must live)              │   │
│   └─────────────────────────────────────────────────────────┘   │
│   Network: bridge (172.17.0.x)                                  │
│   Port mapping: host:5000 → container:5000                      │
└─────────────────────────────────────────────────────────────────┘
         │                          │
    docker ps                  docker exec -it
    docker logs                (inject new process)
    docker stats               (into running namespace)
```

---

## 📝 Day 3 — Key Takeaways

| # | Concept | Remember This |
|---|---------|---------------|
| 1 | **Container vs VM** | Container = shared kernel + namespace isolation. VM = full OS copy |
| 2 | **Image vs Container** | Image = read-only blueprint. Container = live, running instance |
| 3 | **-it vs -dt** | `-it` = you're inside. `-dt` = runs in background |
| 4 | **Golden Rule** | Container dies when last process dies. Period. |
| 5 | **0.0.0.0 binding** | Always use `host='0.0.0.0'` in Docker to accept outside traffic |
| 6 | **Ctrl+P, Q** | Detach without killing. The safe exit. |
| 7 | **docker exec** | Get inside a running container without stopping it |
| 8 | **Port mapping** | `-p HOST:CONTAINER` — bridges host network to container network |
| 9 | **Dockerfile** | Declarative recipe. `FROM` → `RUN` → `COPY` → `CMD` |
| 10 | **Layer caching** | Change-rarely things go at top. Source code goes at bottom |

---

> 📅 **Next Class Preview:** Deep dive into Dockerfiles, multi-stage builds, Docker Compose for multi-container apps, and networking between containers.

---

*Notes prepared from Day 3 class session. Enriched with industry context and production patterns.*
