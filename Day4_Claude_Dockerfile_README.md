# 🐳 Docker & Containerization — Complete Beginner's Guide

> **Based on hands-on class notes** | EC2 + Docker + Flask App Deployment

---

## 📚 Table of Contents

1. [The Big Picture — Why Containers?](#1-the-big-picture--why-containers)
2. [EC2 vs Container — Layered Architecture](#2-ec2-vs-container--layered-architecture)
3. [What is a Docker Image vs Container?](#3-what-is-a-docker-image-vs-container)
4. [What is a Dockerfile?](#4-what-is-a-dockerfile)
5. [Dockerfile Instructions Explained](#5-dockerfile-instructions-explained)
6. [Building Your First Image — Step by Step](#6-building-your-first-image--step-by-step)
7. [Running Containers & Port Mapping](#7-running-containers--port-mapping)
8. [Full Working Example — Flask App](#8-full-working-example--flask-app)
9. [Common Mistakes & Fixes](#9-common-mistakes--fixes)
10. [Key Tips & Best Practices](#10-key-tips--best-practices)
11. [Quick Reference Cheat Sheet](#11-quick-reference-cheat-sheet)

---

## 1. The Big Picture — Why Containers?

Imagine you're building the **IRCTC** train booking app in Python. You need to:

- Install an OS
- Install Python
- Copy your app code
- Install dependencies (like Flask)
- Run the application

If you do all this **manually on an EC2 server**, it works — but what if:
- The server crashes?
- You need to scale to 100 servers?
- A team member sets up a different environment?

You'd have to redo everything from scratch every single time. 😩

### ✅ The Solution: Containers + Docker

With Docker, you write all these steps **once** in a file called `Dockerfile`, build it into an **image (artifact)**, and that image can spin up identical containers **anywhere, anytime** — instantly.

> 💡 **Analogy:**
> - **Dockerfile** = Recipe
> - **Docker Image** = Packaged meal kit (artifact)
> - **Container** = The actual meal on your plate (running instance)

🧠 Big Picture (Flow)
Code → Artifact → Dockerfile → Image → Container

Each step transforms the previous one.

🧱 1) Artifact (Generic concept)

An artifact is any build output.

Examples:
Java → .jar, .war
Python → .whl, app files
Node → compiled bundle

👉 It is language-specific and not runnable alone (needs runtime).

📄 2) Dockerfile (Blueprint)

A Dockerfile is a set of instructions to build an image.

Example:
FROM openjdk:17
COPY app.jar app.jar
CMD ["java", "-jar", "app.jar"]

👉 It defines:

Base environment
Dependencies
How to run the app

👉 It is not executed directly — it’s used to build an image.

📦 3) Image (Immutable package)

A Docker Image is:

Built from Dockerfile
Read-only (immutable)
Layered

👉 Contains:

Artifact (your app)
Runtime (Java, Python, etc.)
OS dependencies
docker build -t myapp .
🚀 4) Container (Running instance)

A container is:

A running instance of an image
Has a writable layer on top
docker run myapp

👉 Multiple containers can run from same image.

🐳 5) Docker (Platform)

Docker is the tool/platform that manages everything:

Builds images
Runs containers
Manages volumes, networks

| Concept    | Type     | Role               | Mutable?          | Example          |
| ---------- | -------- | ------------------ | ----------------- | ---------------- |
| Artifact   | File     | App output         | ✅ Yes             | `app.jar`        |
| Dockerfile | Script   | Build instructions | ✅ Yes             | `FROM ubuntu`    |
| Image      | Package  | App + runtime      | ❌ No (immutable)  | `myapp:1.0`      |
| Container  | Runtime  | Running app        | ✅ Yes (temporary) | Running instance |
| Docker     | Platform | Tooling            | —                 | Docker Engine    |


---

## 2. EC2 vs Container — Layered Architecture

### EC2 Server (Traditional)
```
┌─────────────────────────┐
│      Your App (IRCTC)   │
├─────────────────────────┤
│      Python / Runtime   │
├─────────────────────────┤
│      Operating System   │
├─────────────────────────┤
│        Hardware (EC2)   │
└─────────────────────────┘
```

### EC2 with Docker (Modern)
```
┌──────────────────────────────────────────┐
│              EC2 Instance                │
│  ┌────────────────────────────────────┐  │
│  │          Docker Engine             │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │        Container             │  │  │
│  │  │  ┌────────────────────────┐  │  │  │
│  │  │  │   Your App (IRCTC)     │  │  │  │
│  │  │  ├────────────────────────┤  │  │  │
│  │  │  │   Python + pip         │  │  │  │
│  │  │  ├────────────────────────┤  │  │  │
│  │  │  │   Ubuntu Base OS       │  │  │  │
│  │  │  └────────────────────────┘  │  │  │
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
│              Hardware (AWS)              │
└──────────────────────────────────────────┘
```

> 💡 **Key Insight:** A container has its own lightweight OS, but shares the EC2 machine's hardware. This is why containers are **faster and more efficient** than running separate VMs.

---

## 3. What is a Docker Image vs Container?

| Concept | AWS Equivalent | Description |
|---|---|---|
| **Dockerfile** | User Data script | Instructions to build the environment |
| **Docker Image** | Custom AMI | The artifact/snapshot — static, ready to use |
| **Container** | EC2 Instance (from AMI) | The live, running instance of the image |

### Flow Diagram
```
Dockerfile  →  docker build  →  Docker Image  →  docker run  →  Container
(Instructions)   (Build step)     (Artifact)      (Launch)      (Live App)
```

> ⚠️ **Common Confusion:** Running `docker build` creates an **image**, NOT a container. You only get a running container when you do `docker run`.

---

## 4. What is a Dockerfile?

A **Dockerfile** is a plain text file containing step-by-step instructions for Docker to build a custom image.

**Rules:**
- The filename must be exactly `Dockerfile` — capital **D**, no extension
- Every instruction creates a new **layer** in the image
- More layers = larger image size (optimize by combining commands)
- Docker **caches** layers — unchanged layers don't rebuild (makes builds faster)

---

## 5. Dockerfile Instructions Explained

### `FROM` — Set the Base Image
```dockerfile
FROM ubuntu:22.04
```
- Always the **first instruction**
- Pulls a base OS/image from Docker Hub
- Think of it as: "Start with this operating system"
- Common choices: `ubuntu`, `amazonlinux`, `python:3.11-slim`, `alpine`

> 💡 **Tip:** Use specific tags like `ubuntu:22.04` instead of `ubuntu:latest` to avoid surprises when the "latest" version changes.

---

### `ENV` — Set Environment Variables
```dockerfile
ENV DEBIAN_FRONTEND=noninteractive
```
- Sets environment variables inside the container during build AND runtime
- Used here to prevent Ubuntu from asking interactive questions during `apt-get`

---

### `RUN` — Execute Commands During Build
```dockerfile
RUN apt-get update && apt-get install -y python3-pip && apt-get clean
```
- Runs shell commands **while building** the image
- Each `RUN` creates a new layer
- Use `&&` to chain commands into **one layer** (keeps image size smaller)
- `apt-get clean` at the end removes package cache to save space

> 💡 **Optimization Tip:** Instead of:
> ```dockerfile
> RUN apt-get update
> RUN apt-get install -y python3-pip
> RUN apt-get clean
> ```
> Do this (one layer = smaller image):
> ```dockerfile
> RUN apt-get update && apt-get install -y python3-pip && apt-get clean
> ```

---

### `WORKDIR` — Set Working Directory
```dockerfile
WORKDIR /app
```
- Sets the directory inside the container where subsequent commands run
- If the directory doesn't exist, Docker creates it automatically
- Like doing `mkdir /app && cd /app`

---

### `COPY` — Copy Files from Host to Container
```dockerfile
COPY . /app
```
- Copies files from your **local machine** (the build context) into the image
- First argument: source (on your machine)
- Second argument: destination (inside the container)
- `.` means "everything in the current directory"

---

### `CMD` — Default Command to Run the Container
```dockerfile
CMD ["python3", "app.py"]
```
- Specifies what command runs **when the container starts**
- Only one `CMD` is allowed per Dockerfile (last one wins)
- This is the **foreground process** — when it stops, the container stops
- Uses JSON array format (preferred): `["executable", "param1", "param2"]`

> 💡 **Why CMD matters:** A container's base OS default is `/bin/bash` (just a shell). If you don't override it, your app won't start automatically. `CMD` overrides this default with your actual application process.

---

### `EXPOSE` — Document the Port (Best Practice)
```dockerfile
EXPOSE 5000
```
- Documents which port the application listens on
- Does **not** actually publish the port (you still need `-p` in `docker run`)
- Acts as documentation for other developers

---

## 6. Building Your First Image — Step by Step

### Step 1: Create your project files
```
my-project/
├── Dockerfile
├── app.py
└── requirements.txt
```

### Step 2: Write the Dockerfile
```dockerfile
FROM ubuntu:22.04

# Avoid interactive prompts during package install
ENV DEBIAN_FRONTEND=noninteractive

# Install Python and pip
RUN apt-get update && \
    apt-get install -y python3-pip && \
    apt-get clean

# Set working directory inside container
WORKDIR /app

# Copy all project files into /app
COPY . /app

# Install Python dependencies
RUN pip3 install -r requirements.txt

# Run the app when container starts
CMD ["python3", "app.py"]
```

### Step 3: Build the image
```bash
# Standard build (uses file named "Dockerfile" in current dir)
docker build -t myapp:v1 .

# If your Dockerfile has a different name
docker build -t myapp:v1 . -f MyCustomDockerfile
```

**Flags explained:**
- `-t myapp:v1` — Tag/name your image (`name:version`)
- `.` — Build context (current directory — where Docker looks for files to COPY)

### Step 4: Verify the image was created
```bash
docker images
```
```
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
myapp        v1        391cfcea7f24   2 minutes ago    480MB
```

---

## 7. Running Containers & Port Mapping

### Basic Run Commands
```bash
# Run interactively (attached to terminal)
docker run -it myapp:v1

# Run in detached/background mode
docker run -dt myapp:v1

# Run with port mapping (HOST_PORT:CONTAINER_PORT)
docker run -dt -p 5001:5000 myapp:v1
```

**Flags explained:**
- `-i` — Interactive (keep STDIN open)
- `-t` — Allocate a terminal (TTY)
- `-d` — Detached (run in background)
- `-p 5001:5000` — Map port 5001 on the host → port 5000 inside container

### Understanding Port Mapping

```
Your Browser  →  EC2 Host Port 5001  →  Container Port 5000  →  Flask App
http://IP:5001           ↕ mapped                  ↕ listening
```

> 💡 **Why two ports?** The container is isolated — it has its own network. Port 5000 is only accessible *inside* the container. By mapping `-p 5001:5000`, you're creating a "tunnel" so outside traffic on port 5001 reaches the app inside on port 5000.

> ⚠️ **Don't forget:** Even after port mapping, you must **open the port in AWS Security Group** inbound rules, or the browser will say "site can't be reached"!

### Check Running Containers
```bash
docker ps          # Show running containers
docker ps -a       # Show all containers (including stopped)
```

### Check if port is listening on host
```bash
ss -tuln           # Show all listening ports
```

---

## 8. Full Working Example — Flask App

### `app.py`
```python
from flask import Flask
import os

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello from IRCTC App — Version 1.0!"

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 5000))
    app.run(debug=True, host='0.0.0.0', port=port)
```

> 💡 **Why `host='0.0.0.0'`?** By default Flask listens only on `127.0.0.1` (localhost inside the container). Setting `0.0.0.0` makes it listen on ALL interfaces, so traffic from outside the container can reach it.

### `requirements.txt`
```
flask
```

### `Dockerfile`
```dockerfile
FROM ubuntu:22.04
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y python3-pip && apt-get clean
WORKDIR /app
COPY . /app
RUN pip3 install -r requirements.txt
CMD ["python3", "app.py"]
```

### Build & Run
```bash
docker build -t flask-app .
docker run -dt -p 5001:5000 flask-app
```

### Access in Browser
```
http://<YOUR_EC2_PUBLIC_IP>:5001/
```

---

## 9. Common Mistakes & Fixes

### ❌ Mistake 1: Wrong command when running
```bash
# WRONG — the "." is being passed as an argument to the container
docker run -it img3 .
# Error: exec: ".": executable file not found in $PATH
```
```bash
# CORRECT
docker run -it img3
# or with port mapping
docker run -it -p 5000:5000 img3
```

### ❌ Mistake 2: Can't access app from browser
**Checklist:**
1. ✅ Is the container running? (`docker ps`)
2. ✅ Did you map the port? (`-p HOST:CONTAINER`)
3. ✅ Is the port open in **AWS Security Group** (Inbound Rules)?
4. ✅ Is Flask listening on `0.0.0.0` (not just `127.0.0.1`)?

### ❌ Mistake 3: Filename is `dockerfile` (lowercase d)
```bash
# WRONG filename
dockerfile   ← Docker won't find this by default

# CORRECT
Dockerfile   ← Capital D is required
```

### ❌ Mistake 4: Forgetting to rebuild after code changes
```bash
# After editing app.py or Dockerfile, always rebuild
docker build -t myapp:v2 .
```

### ❌ Mistake 5: Typo in requirements filename
```bash
# Class example had a typo:
requirments.txt   ← wrong spelling

# Correct:
requirements.txt  ← standard naming
```

---

## 10. Key Tips & Best Practices

### 🏗️ Layer Caching — Speed Up Builds
Docker caches each layer. If a layer hasn't changed, it reuses it.

**Order your Dockerfile smartly:**
```dockerfile
# ✅ GOOD — install dependencies BEFORE copying code
# Dependencies change rarely → cached most of the time
COPY requirements.txt /app/requirements.txt
RUN pip3 install -r requirements.txt

# Code changes often — copy it last
COPY . /app
```
This way, when you only change `app.py`, Docker reuses the cached pip install layer and only re-runs the COPY.

---

### 🐳 Use Slim Base Images When Possible
```dockerfile
# 480MB — full Ubuntu
FROM ubuntu:22.04

# ~130MB — slim Python image (recommended for Python apps!)
FROM python:3.11-slim
```

---

### 🔁 Container Lifecycle
```
docker run  →  Container CREATED & RUNNING
docker stop →  Container STOPPED (still exists)
docker rm   →  Container DELETED
docker rmi  →  Image DELETED
```

---

### 🏭 Connection to AWS ASG (Auto Scaling Group)
```
Dockerfile
    ↓ docker build
Docker Image
    ↓ push to ECR (Elastic Container Registry)
ECR Image URI
    ↓ used in
EC2 Launch Template / ECS Task Definition
    ↓ used by
Auto Scaling Group (ASG)
    ↓ automatically scales
Multiple identical containers ✅
```

---

## 11. Quick Reference Cheat Sheet

### Dockerfile Instructions
| Instruction | Purpose | Example |
|---|---|---|
| `FROM` | Base image | `FROM ubuntu:22.04` |
| `ENV` | Environment variable | `ENV PORT=5000` |
| `RUN` | Run command at build time | `RUN apt-get install -y curl` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `COPY` | Copy files into image | `COPY . /app` |
| `CMD` | Default run command | `CMD ["python3", "app.py"]` |
| `EXPOSE` | Document the port | `EXPOSE 5000` |

### Docker CLI Commands
| Command | Description |
|---|---|
| `docker build -t name:tag .` | Build image from Dockerfile |
| `docker images` | List all images |
| `docker run -dt -p HOST:CONT image` | Run container (detached + port mapped) |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers |
| `docker stop <id>` | Stop a container |
| `docker rm <id>` | Remove a container |
| `docker rmi <image>` | Remove an image |
| `docker logs <id>` | View container logs |
| `docker exec -it <id> bash` | Enter a running container |

---

## Summary

```
Problem: App setup is manual, not repeatable
    ↓
Solution: Write a Dockerfile (all steps in one place)
    ↓
Run: docker build → Creates Image (Artifact)
    ↓
Run: docker run → Creates Container (Live App)
    ↓
Access: Browser → EC2 IP:PORT (port opened in Security Group)
    ↓
Scale: Push image to ECR → Use in ASG Launch Template → Auto-scale ✅
```

---

*Happy Dockerizing! 🐳*
