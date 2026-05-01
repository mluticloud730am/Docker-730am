# 🐳 Docker Deep Dive — Class Notes
### Topics: CMD vs ENTRYPOINT | RUN | ADD vs COPY | EXPOSE | Image Optimization

---

## 📋 Table of Contents
1. [CMD — Container Default Command](#1-cmd--container-default-command)
2. [ENTRYPOINT — Fixed Executable](#2-entrypoint--fixed-executable)
3. [CMD vs ENTRYPOINT — The Core Difference](#3-cmd-vs-entrypoint--the-core-difference)
4. [RUN vs CMD vs ENTRYPOINT — Interview Question](#4-run-vs-cmd-vs-entrypoint--interview-question)
5. [ADD vs COPY](#5-add-vs-copy)
6. [EXPOSE — What Does It Actually Do?](#6-expose--what-does-it-actually-do)
7. [Port Mapping Deep Dive (-p HOST:CONTAINER)](#7-port-mapping-deep-dive)
8. [Image Optimization — node:18 vs node:18-alpine](#8-image-optimization--node18-vs-node18-alpine)
9. [Docker Layer Caching & Best Practices](#9-docker-layer-caching--best-practices)
10. [Quick Command Reference](#10-quick-command-reference)

---

## 1. CMD — Container Default Command

### What is CMD?
`CMD` defines the **default command** that runs when a container starts. It is **not fixed** — it can be overridden at runtime by passing a command to `docker run`.

### Syntax
```dockerfile
CMD ["executable", "arg1", "arg2"]    # Exec form (recommended)
CMD command arg1 arg2                  # Shell form
```

### Live Lab from Class

**Dockerfile:**
```dockerfile
FROM ubuntu:latest
CMD ["echo", "Hello WoRld"]
```

**Build:**
```bash
docker build -t img1 .
```

**Experiment 1 — Override CMD at runtime:**
```bash
docker run -dt img1 echo welcom
# Container ran "echo welcom" — CMD was completely REPLACED
```

**Experiment 2 — Use default CMD:**
```bash
docker run -dt img1
# Container ran "echo Hello WoRld" — CMD was used as-is
```

**Experiment 3 — Pass invalid command:**
```bash
docker run -dt img1 hi
# ERROR: "hi": executable file not found in $PATH
# Because "hi" is not a binary — CMD replacement only accepts executables
```

### 🔑 Key Rule for CMD
> If you pass ANY command in `docker run`, it **completely replaces** the CMD defined in the Dockerfile.
> CMD only executes if no command is given at runtime.

---

## 2. ENTRYPOINT — Fixed Executable

### What is ENTRYPOINT?
`ENTRYPOINT` defines the **main process/executable** of the container. Unlike CMD, it **cannot be replaced** at runtime — only appended to.

### Syntax
```dockerfile
ENTRYPOINT ["executable"]
ENTRYPOINT ["executable", "default-arg"]
```

### Live Lab from Class

**Dockerfile (enpcmb image):**
```dockerfile
FROM ubuntu:latest
ENTRYPOINT ["echo"]
CMD ["Hello WoRld"]
```

```bash
docker run enpcmb hi
# Output: hi
# "echo" stayed fixed, "hi" replaced CMD's default arg
```

**Dockerfile (enp image):**
```dockerfile
FROM ubuntu:latest
ENTRYPOINT ["echo", "Hello"]
# CMD is commented out
```

```bash
docker run enp
# Output: Hello

docker run enp devops
# Output: Hello devops
# "devops" was APPENDED after "Hello"
```

### 🔑 Key Rule for ENTRYPOINT
> Arguments passed in `docker run` are **APPENDED** to ENTRYPOINT — they do NOT replace it.
> ENTRYPOINT = the command that always runs, no matter what.

---

## 3. CMD vs ENTRYPOINT — The Core Difference

| Feature | CMD | ENTRYPOINT |
|---|---|---|
| Purpose | Default command/args | Fixed executable |
| Can be overridden at runtime? | ✅ Yes, fully replaced | ❌ No, only appended |
| Used together? | ✅ Yes, CMD provides default args to ENTRYPOINT | ✅ Yes |
| Behavior when combined | CMD becomes default args | ENTRYPOINT is always the executable |
| Runtime override syntax | `docker run image new-cmd` | `docker run --entrypoint new-cmd image` |

### Combined Usage Pattern (Most Production-Friendly)
```dockerfile
ENTRYPOINT ["echo"]
CMD ["Hello WoRld"]
```

```bash
docker run image          # → echo Hello WoRld  (uses CMD as default args)
docker run image "Rakesh" # → echo Rakesh        (CMD replaced, ENTRYPOINT stays)
```

### 🧠 Mental Model
```
ENTRYPOINT = The "verb" (what action to do) — always runs
CMD        = The "noun" (default data/args) — can be swapped
```

---

## 4. RUN vs CMD vs ENTRYPOINT — Interview Question

This is one of the **most asked Docker interview questions**. Here's a crystal-clear comparison:

| Instruction | When It Runs | Purpose | Layers Created |
|---|---|---|---|
| `RUN` | **During image build** | Install packages, run scripts, setup environment | ✅ Creates a new image layer |
| `CMD` | **When container starts** | Default command (overridable at runtime) | ❌ No new layer |
| `ENTRYPOINT` | **When container starts** | Fixed executable (cannot be replaced, only appended) | ❌ No new layer |

### Analogy for Freshers
Think of building a house:
- **RUN** = Construction work done while building the house (install tiles, paint walls) — happens **once during build**
- **CMD** = The default instruction left for the resident: "Make coffee in the morning" — but they can **choose to do something else**
- **ENTRYPOINT** = The locked front door — the resident **must** go through it, no choice

### Code Example Showing All Three
```dockerfile
FROM ubuntu:latest

# RUN → executes during BUILD, creates a layer
RUN apt-get update && apt-get install -y curl

# ENTRYPOINT → always runs at container start
ENTRYPOINT ["curl"]

# CMD → default arg passed to curl (can be overridden)
CMD ["https://google.com"]
```

```bash
docker run myimage                       # → curl https://google.com
docker run myimage https://github.com    # → curl https://github.com
```

---

## 5. ADD vs COPY

Both instructions copy files into the container image — but they have important differences.

### COPY — Simple, Explicit, Recommended
```dockerfile
COPY <src> <dest>
COPY . .
COPY package*.json ./
```
- Copies files/directories from **local build context** only
- Simple and predictable — does exactly what you say
- ✅ Preferred for most use cases

### ADD — Superpowered COPY (with extras)
```dockerfile
ADD <src> <dest>
ADD https://example.com/file.tar.gz /app/
```
Additional capabilities ADD has over COPY:
- ✅ Can download files from **URLs** (HTTP/HTTPS)
- ✅ Can **auto-extract** `.tar`, `.tar.gz`, `.tgz` archives
- ❌ Less predictable — behavior changes based on source type

### Live Lab from Class

**Dockerfile (ADD with URL):**
```dockerfile
FROM ubuntu:20.04

WORKDIR /app

ADD https://github.com/torvalds/linux/archive/refs/tags/v5.14.tar.gz /app/

CMD ["sh", "-c", "ls /app && sleep 3600"]
```

```bash
docker build -t add .
docker run add
# Output: v5.14.tar.gz   (downloaded from GitHub into /app/)
```

Verified inside container:
```bash
docker exec -it <container_id> /bin/bash
cd /app
ls        # → v5.14.tar.gz
```

### ADD vs COPY Comparison Table

| Feature | COPY | ADD |
|---|---|---|
| Local files | ✅ Yes | ✅ Yes |
| Remote URLs | ❌ No | ✅ Yes |
| Auto-extract tarballs | ❌ No | ✅ Yes (local only) |
| Recommended for local copy | ✅ Preferred | ❌ Use COPY instead |
| Recommended for URL fetch | N/A | ✅ But consider RUN curl instead |

### 🔑 Best Practice
> Use `COPY` by default.
> Use `ADD` only when you specifically need URL download or tarball auto-extraction.

---

## 6. EXPOSE — What Does It Actually Do?

### Common Misconception
Many beginners think `EXPOSE` opens/publishes a port. **It does NOT.**

### What EXPOSE Actually Does
```dockerfile
EXPOSE 3000
```

`EXPOSE` is **documentation/metadata only**. It tells anyone reading the Dockerfile:
> "This application inside the container listens on port 3000."

It does **NOT**:
- Open any port to the outside world
- Make the app accessible from your browser
- Do anything at network level

### Then How Do You Actually Expose a Port?
Using the `-p` flag in `docker run`:
```bash
docker run -p HOST_PORT:CONTAINER_PORT image
```

### From the Class Lab
```bash
# App inside container listens on port 3000
docker run -dt -p 3002:3000 my-node-app
# Now access: http://your-ip:3002
```

Even the `node` image (no EXPOSE 81) worked when mapped:
```bash
docker run -dt -p 82:81 node       # port 82 on host → 81 in container
docker run -dt -p 83:81 my-node-app # port 83 on host → 81 in container
# Both http://3.94.195.76:82 and http://3.94.195.76:83 returned: Welcome to DevOps Training
```

### 🔑 EXPOSE Summary
| | EXPOSE | -p (docker run) |
|---|---|---|
| Who writes it | Developer (in Dockerfile) | DevOps/Operator (at runtime) |
| What it does | Documents intent | Actually publishes the port |
| Required? | No | Yes, to access from outside |
| Network effect | None | Maps host port → container port |

---

## 7. Port Mapping Deep Dive

### Syntax
```bash
docker run -p HOST_PORT:CONTAINER_PORT image
```

### How It Works
```
Browser/User
    │
    ▼
[EC2 Host Machine] :3002          ← You open this
    │   (Host port 3002)
    │   Docker NAT/iptables
    ▼
[Container] :3000                  ← App listens here
```

### Class Examples Explained
```bash
docker run -dt -p 3002:3000 my-node-app
# User hits port 3002 on EC2 → Docker forwards to port 3000 inside container

docker run -dt -p 82:81 node
# User hits port 82 on EC2 → Docker forwards to port 81 inside container

docker run -dt -p 83:81 my-node-app
# User hits port 83 on EC2 → Docker forwards to port 81 inside container
```

### Important: Host Port Must Be Unique
```bash
docker run -p 8080:3000 app1   ✅ OK
docker run -p 8080:3000 app2   ❌ ERROR — port 8080 already in use on host
docker run -p 8081:3000 app2   ✅ OK — different host port
```

---

## 8. Image Optimization — node:18 vs node:18-alpine

This was directly observed in class from `docker images` output:

```
REPOSITORY    SIZE
node          1.09GB    ← FROM node:18 (Debian-based)
my-node-app   127MB     ← FROM node:18-alpine
```

**Same Node.js version. 8x size difference.**

### ❌ Unoptimized Dockerfile
```dockerfile
FROM node:18

WORKDIR /app

COPY . .

RUN npm install

EXPOSE 3000

CMD ["npm", "start"]
```

**Problems:**
- `node:18` is based on Debian — includes compilers, build tools, man pages, locales
- `COPY . .` may include `node_modules`, `.git`, test files
- `npm install` installs ALL dependencies including devDependencies
- Running as root (security risk)

**Result:** 1GB+ image, slow CI/CD, security vulnerabilities

### ✅ Optimized Dockerfile
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install --omit=dev

COPY . .

USER node

EXPOSE 3000

CMD ["npm", "start"]
```

**Why each line matters:**

| Line | Why It Matters |
|---|---|
| `FROM node:18-alpine` | Alpine Linux = ~5MB base vs ~170MB Debian. Only essential packages included |
| `COPY package*.json ./` | Copy only dependency manifests first (enables layer caching) |
| `RUN npm install --omit=dev` | Skip devDependencies (jest, eslint, etc.) — not needed in production |
| `COPY . .` | App code copied after deps (cache-friendly order) |
| `USER node` | Non-root user — limits damage if container is compromised |

**Result:** 127MB image — 8x smaller, faster deploys, production-secure

### What are Dev vs Prod Dependencies in Node.js?

```json
{
  "dependencies": {
    "express": "^4.18.2"      ← PRODUCTION: app needs this to run
  },
  "devDependencies": {
    "jest": "^29.0.0",         ← DEV ONLY: testing framework
    "eslint": "^8.0.0",        ← DEV ONLY: code linting
    "nodemon": "^3.0.0"        ← DEV ONLY: auto-restart during development
  }
}
```

`npm install --omit=dev` skips devDependencies → smaller image, less attack surface

### Layer-by-Layer Comparison (from docker history)

**node:18 (1.09GB):**
```
588MB  → apt-get install (compilers, build tools, Debian packages)
177MB  → more Debian packages
154MB  → Node.js binary installation
117MB  → Debian base OS
```

**node:18-alpine (127MB):**
```
114MB  → Node.js binary on Alpine
7.83MB → Alpine Linux base OS (entire OS!)
5.37MB → apk packages
```

Alpine's entire OS is smaller than Debian's `apt-get update` output!

---

## 9. Docker Layer Caching & Best Practices

### How Caching Works
Docker builds images layer by layer. If a layer hasn't changed, Docker reuses the **cached** version — making rebuilds much faster.

### Why Order of Instructions Matters

```dockerfile
# ❌ BAD ORDER — breaks cache every time code changes
FROM node:18-alpine
WORKDIR /app
COPY . .                  ← Any code change invalidates cache here
RUN npm install           ← Always re-runs even if package.json unchanged
```

```dockerfile
# ✅ GOOD ORDER — cache-friendly
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./     ← Only changes when dependencies change
RUN npm install           ← Cache hit if package.json unchanged ✅
COPY . .                  ← Source code copied last
```

With the good order, changing `app.js` only re-runs the last `COPY` — `npm install` is cached.

### General Dockerfile Best Practice Order
```
1. FROM          → base image
2. ENV/ARG       → environment variables
3. RUN           → install system dependencies
4. WORKDIR       → set working directory
5. COPY deps     → copy package files only
6. RUN install   → install app dependencies (cacheable)
7. COPY src      → copy application code
8. USER          → non-root user
9. EXPOSE        → document port
10. CMD/ENTRYPOINT → startup command
```

---

## 10. Quick Command Reference

### Build & Run
```bash
docker build -t image-name .              # Build image from Dockerfile in current dir
docker build -t image-name /path/to/dir  # Build from specific directory
docker run image-name                    # Run (foreground)
docker run -d image-name                 # Run detached (background)
docker run -dt image-name                # Detached + TTY (keeps running)
docker run -dt -p 8080:3000 image-name  # Detached + port mapping
docker run image-name custom-command     # Override CMD
```

### Inspect & Debug
```bash
docker ps                         # Running containers
docker ps -a                      # All containers (including stopped)
docker images                     # List images
docker history image-name         # Show all layers and sizes
docker inspect image-name         # Full metadata (JSON)
docker exec -it <id> /bin/bash    # Shell into running container
docker logs <id>                  # View container output
```

### Cleanup
```bash
docker stop <id>                  # Stop container
docker rm <id>                    # Remove stopped container
docker rmi image-name             # Remove image
docker system prune               # Remove all unused resources
```

---

## 🎯 Summary Cheatsheet

```
RUN        → Build time  | Installs packages | Creates layers
CMD        → Start time  | Default command   | Fully REPLACED by docker run args
ENTRYPOINT → Start time  | Fixed executable  | Runtime args APPENDED to it

COPY       → Local files only | Simple | Recommended
ADD        → Local + URLs + tar extraction | Use only when needed

EXPOSE     → Documentation only | Does NOT publish port
-p         → Actually maps host:container port | Required for outside access

node:18        → Debian-based | ~1GB  | Dev/debugging
node:18-alpine → Alpine-based | ~127MB | Production
```

---

## 🧪 Exercises to Practice

1. **CMD Override Lab**: Build an image with `CMD ["echo", "default"]`. Run it without args, then with `"custom"`. Observe the difference.

2. **ENTRYPOINT Append Lab**: Build with `ENTRYPOINT ["echo", "Hello"]`. Run without args, then with `"World"`. Observe output.

3. **Port Mapping Lab**: Run the same image twice with `-p 8080:3000` and `-p 8081:3000`. Access both from browser.

4. **Image Slim Lab**: Build the same Node.js app with `node:18` and `node:18-alpine`. Compare sizes using `docker images`.

5. **Layer Cache Lab**: Build optimized Dockerfile. Change `app.js` only and rebuild — notice how `npm install` step is cached.

6. **Exec Lab**: Run a container with `sleep 3600`, then `docker exec -it <id> /bin/bash` and explore the filesystem.

---

*Notes from DevOps Training | Docker Module*
