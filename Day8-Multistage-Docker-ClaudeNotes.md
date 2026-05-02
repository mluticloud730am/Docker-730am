# 🐳 Day 8 — Docker Multi-Stage Builds
### Deep Dive: Image Optimization, Build Internals & Real-World Patterns

> 📁 Class Repo: [mluticloud730am/2nd10WeeksofCloudOps](https://github.com/mluticloud730am/2nd10WeeksofCloudOps-main_20260502)

---

## 📌 Table of Contents

1. [The Problem — Why Multi-Stage?](#1-the-problem--why-multi-stage)
2. [Traditional VM vs Docker Analogy](#2-traditional-vm-vs-docker-analogy)
3. [Single-Stage Build (The Bloated Way)](#3-single-stage-build-the-bloated-way)
4. [Multi-Stage Build (The Optimized Way)](#4-multi-stage-build-the-optimized-way)
5. [How Stages Work Internally](#5-how-stages-work-internally)
6. [Real Class Output — image1 vs image2](#6-real-class-output--image1-vs-image2)
7. [Spring Boot Backend — Multi-Stage with Maven](#7-spring-boot-backend--multi-stage-with-maven)
8. [Concept Clarity Q&A](#8-concept-clarity-qa)
9. [Image Size & Optimization](#9-image-size--optimization)
10. [Runtime vs Build-Time](#10-runtime-vs-build-time)
11. [Web Server Choice — Apache vs Nginx](#11-web-server-choice--apache-vs-nginx)
12. [Security & Best Practices](#12-security--best-practices)
13. [Docker Internals — Deep Dive](#13-docker-internals--deep-dive)
14. [CI/CD & Production Scenarios](#14-cicd--production-scenarios)
15. [Troubleshooting & Edge Cases](#15-troubleshooting--edge-cases)
16. [Advanced Topics](#16-advanced-topics)
17. [Quick Reference Cheatsheet](#17-quick-reference-cheatsheet)

---

## 1. The Problem — Why Multi-Stage?

When you containerize an application, your first instinct is to put **everything** in one Dockerfile — install Node.js, install dependencies, build the app, and serve it. This works, but the result is a **massively oversized image**.

### What's inside a bloated image?

| Layer | What it adds | Needed in Production? |
|---|---|---|
| Base OS (Debian/Ubuntu) | ~100–600MB | Partially |
| Node.js runtime | ~150MB | ❌ Only for building |
| npm packages (`node_modules`) | ~300–500MB | ❌ Only for building |
| Build tools, compilers | ~50–200MB | ❌ Only for building |
| Your actual built app | ~1–5MB | ✅ YES |

**The entire `node_modules` folder and Node.js runtime are only needed to _create_ the build artifact — not to _serve_ it.**

Multi-stage builds solve this by saying:  
> *"Use one environment to BUILD the app, and a completely separate, minimal environment to RUN it."*

---

## 2. Traditional VM vs Docker Analogy

This is the key mental model from class:

### Traditional Server Setup (2 Physical/VM Servers)

```
┌─────────────────────────┐        ┌─────────────────────────┐
│        SERVER 1         │        │        SERVER 2          │
│  (Build Server)         │   JAR  │  (Application Server)   │
│                         │ ──────▶│                          │
│  Maven installed        │        │  Tomcat installed        │
│  JDK installed          │        │  JRE installed           │
│  Source code            │        │  app.jar running         │
│  → mvn clean install    │        │  → java -jar app.jar     │
└─────────────────────────┘        └─────────────────────────┘
```

### Docker Multi-Stage (Same Concept, 1 Dockerfile)

```
┌─────────────────────────┐        ┌─────────────────────────┐
│  STAGE 1 (FROM maven)   │        │  STAGE 2 (FROM jre)     │
│  "builder"              │  .jar  │  Final Image             │
│                         │ ──────▶│                          │
│  Maven + JDK            │        │  Only JRE (no Maven)    │
│  Source code            │        │  app.jar copied here     │
│  → mvn clean package    │        │  → java -jar app.jar     │
└─────────────────────────┘        └─────────────────────────┘
         Discarded ♻️                      Deployed ✅
```

**Key Insight:**  
In traditional setup = 2 separate physical servers.  
In Docker multi-stage = 2 logical stages inside ONE Dockerfile.  
The result is the same: **only the artifact moves to production, not the build tools.**

---

## 3. Single-Stage Build (The Bloated Way)

This is what was built in class as `image1`:

```dockerfile
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
RUN npm install -g http-server
EXPOSE 80
CMD ["http-server", "build", "-p", "80"]
```

### What happens here step by step:

```
node:18 base image  (~1.1GB)
    ↓
COPY source code    (+1.22MB)
    ↓
npm install         (+353MB)  ← ALL dev + prod dependencies
    ↓
npm run build       (+2.88MB) ← Creates /app/build/ folder
    ↓
npm install -g http-server (+6.5MB)
    ↓
FINAL IMAGE SIZE = 1.45GB  ❌ Massive!
```

### The Problem:
- `node_modules` (353MB) is still inside the image — but you don't need it to serve static files
- Node.js runtime is still there — but you only need a web server
- All build-time tooling is shipped to production — this is wasteful AND insecure

---

## 4. Multi-Stage Build (The Optimized Way)

This is what was built in class as `image2`:

```dockerfile
# ─────────────────────────────────────────────
# STAGE 1: Build React app
# ─────────────────────────────────────────────
FROM node:18 AS build

WORKDIR /app

# Copy dependency manifests first (cache optimization — explained later)
COPY package.json package-lock.json ./

# Install all dependencies
RUN npm install

# Copy rest of source code
COPY . .

# Build the React app → creates /app/build/ folder
RUN npm run build

# ─────────────────────────────────────────────
# STAGE 2: Serve with Apache HTTP Server
# ─────────────────────────────────────────────
FROM httpd:2.4

# Copy ONLY the build output from Stage 1
# Nothing else from Stage 1 comes here
COPY --from=build /app/build/ /usr/local/apache2/htdocs/

EXPOSE 80

CMD ["httpd-foreground"]
```

### What happens here step by step:

```
STAGE 1: node:18
    ↓ npm install, npm run build
    ↓ /app/build/ created (pure HTML/CSS/JS)
    ↓ This stage is DISCARDED after build ♻️

STAGE 2: httpd:2.4 (~118MB base)
    ↓ COPY --from=build /app/build/ → copies only 1.08MB of static files
    ↓
FINAL IMAGE SIZE = 118MB  ✅ 12x smaller!
```

---

## 5. How Stages Work Internally

### Does each `FROM` create a fresh isolated filesystem?

**YES.** Every `FROM` instruction starts a completely new, isolated filesystem — like a fresh server with nothing installed except what that base image provides.

```
FROM node:18 AS build     ← Filesystem A: Debian + Node.js + npm
FROM httpd:2.4            ← Filesystem B: Debian + Apache only
```

These two filesystems have **zero overlap**. Stage 2 cannot see `node_modules`, cannot see source code, cannot see anything from Stage 1 unless you explicitly `COPY --from=`.

### How does `COPY --from=build` work?

```dockerfile
COPY --from=build /app/build/ /usr/local/apache2/htdocs/
```

Breakdown:
- `--from=build` → references the stage named `build` (from `AS build`)
- `/app/build/` → source path **inside Stage 1's filesystem**
- `/usr/local/apache2/htdocs/` → destination path **inside Stage 2's filesystem**

Docker internally:
1. Completes Stage 1 fully (runs all its RUN commands)
2. Snapshots Stage 1's filesystem in memory temporarily
3. Copies the specified path from that snapshot into Stage 2
4. Stage 1 is then discarded — it never becomes part of the final image

You can also reference by stage number (0-indexed) instead of name:
```dockerfile
COPY --from=0 /app/build/ /usr/local/apache2/htdocs/
```
But **naming stages with `AS`** is always preferred for clarity.

---

## 6. Real Class Output — image1 vs image2

### docker images (from class terminal)
```
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
image2       latest    8fe165ad47ed   47 seconds ago   118MB    ✅
image1       latest    723c3d51f5b0   24 minutes ago   1.45GB   ❌
```

### docker history image1 (showing what bloated it)

| Layer | Size | Needed in Prod? |
|---|---|---|
| Debian base OS (2yr old) | ~588MB + 177MB + 48MB | Partially |
| Node.js 18.20.8 install | ~154MB | ❌ No |
| Source code COPY | 1.22MB | ❌ No |
| `npm install` | **353MB** | ❌ No |
| `npm run build` | 2.88MB | ✅ Only the output |
| `npm install -g http-server` | 6.5MB | ❌ No |

### docker history image2 (clean and lean)

| Layer | Size | Needed in Prod? |
|---|---|---|
| Debian trixie base | 78.6MB | ✅ |
| Apache httpd install | ~38MB | ✅ |
| Your build files COPY | **1.08MB** | ✅ |

**Total: 118MB vs 1.45GB → ~12x reduction in image size.**

---

## 7. Spring Boot Backend — Multi-Stage with Maven

Class also demonstrated the same pattern for a Java Spring Boot application. This reinforces that multi-stage is **language-agnostic**.

### Dockerfile from class (backend):

```dockerfile
# ─────────────────────────────────────────────
# Stage 1 — Build JAR using Maven
# ─────────────────────────────────────────────
FROM maven:3.9.9-amazoncorretto-11 AS builder
WORKDIR /app
COPY . /app

# Build the JAR, skip tests for faster build
RUN mvn clean package -Dspring.profiles.active=build -DskipTests

# ─────────────────────────────────────────────
# Stage 2 — Run JAR using only JRE (no Maven)
# ─────────────────────────────────────────────
FROM amazoncorretto:11-alpine-jdk
WORKDIR /app

# Copy only the built JAR from builder stage
COPY --from=builder /app/target/*.jar app.jar

# Environment variables for DB connection (can be overridden at runtime)
ENV SPRING_DATASOURCE_URL="jdbc:mysql://database-1.ckjyc24s4jgb.us-east-1.rds.amazonaws.com:3306/datastore?createDatabaseIfNotExist=true"
ENV SPRING_DATASOURCE_USERNAME="admin"
ENV SPRING_DATASOURCE_PASSWORD="Cloud123"
ENV SPRING_JPA_HIBERNATE_DDL_AUTO="update"
ENV SERVER_PORT=8084

EXPOSE 8084

CMD ["java", "-jar", "app.jar"]
```

### Build output from class:
```
REPOSITORY   TAG       IMAGE ID       CREATED              SIZE
backend      latest    1537427b6447   About a minute ago   336MB   ✅
image2       latest    8fe165ad47ed   25 minutes ago       118MB   ✅
image1       latest    723c3d51f5b0   49 minutes ago       1.45GB  ❌
```

### Container run:
```bash
docker run -dt -p 8085:8084 backend
```
Port mapping: host `8085` → container `8084` (Spring Boot listens on 8084)

### Exec into alpine container (important difference!):
```bash
# Alpine uses /bin/sh, NOT /bin/bash
docker exec -it ab4e7291448c /bin/sh   # ✅ Works

docker exec -it ab4e7291448c /bin/bash # ❌ Error: no such file
# Alpine Linux does not include bash by default
```

### Inside running container:
```
/app # ps aux
PID   USER     TIME  COMMAND
    1 root      0:12 java -jar app.jar   ← Only the app, no Maven
   37 root      0:00 /bin/sh
```

**This proves multi-stage worked:** only the JRE + JAR is running. Maven is completely gone.

### Parallel real-world analogy:

```
Traditional:                        Docker Multi-Stage:
┌────────────────┐                  ┌──────────────────────┐
│ Build Server   │                  │ Stage 1: maven:3.9.9 │
│ Maven + JDK    │                  │ mvn clean package    │
│ mvn install    │──── .jar ────▶   ├──────────────────────┤
└────────────────┘                  │ Stage 2: amazoncorr. │
┌────────────────┐                  │ java -jar app.jar    │
│ App Server     │                  └──────────────────────┘
│ Tomcat/JRE     │
│ java -jar      │
└────────────────┘
```

---

## 8. Concept Clarity Q&A

### Q: Are stages just logical separation, not physical servers?
**A:** Exactly right. In the traditional world (Maven → Tomcat), you literally have two separate machines or VMs. In Docker multi-stage, both stages are defined in **one Dockerfile** but they are logically isolated — each stage has its own filesystem, its own installed packages, and its own working directory. No stage can access another stage's data unless you explicitly `COPY --from=`. It's the same architectural principle, just virtualized inside one build process.

---

### Q: Does each `FROM` create a completely new isolated filesystem?
**A:** Yes, completely. Think of each `FROM` as booting a brand new server with only what that image contains. If `FROM node:18` has Node.js and `FROM httpd:2.4` has Apache, they have nothing in common at the filesystem level. Even if both are based on Debian under the hood, their layer stacks are entirely separate during the build.

---

### Q: How exactly does `COPY --from=build` work internally?
**A:** Docker's BuildKit engine:
1. Executes all instructions in Stage 1 (the `build` stage) and snapshots its final filesystem
2. When it reaches `COPY --from=build` in Stage 2, it reads from that snapshot
3. Only the files you specify are extracted and written into Stage 2's filesystem
4. Stage 1's snapshot is then released from memory
5. The final image contains **only Stage 2's layers** — Stage 1 left no trace

`--from=build` is referencing the **named stage** (a temporary layer/snapshot), not a separate image or running container.

---

## 9. Image Size & Optimization

### Q: Why is image1 1.45GB but image2 only 118MB? What exactly got removed?

Everything that was in Stage 1 (node:18 base, node_modules, source code, npm cache) was **never copied** to Stage 2. Only `/app/build/` (1.08MB of static HTML/CSS/JS) made it across.

```
image1 breakdown:
  node:18 base OS        ~1.09GB
  npm install            ~353MB
  build artifacts        ~2.88MB
  http-server package    ~6.5MB
  ─────────────────────────────
  Total:                  1.45GB

image2 breakdown:
  httpd:2.4 base OS      ~117MB
  Copied build files     ~1.08MB
  ─────────────────────────────
  Total:                  118MB
```

### Q: Is multi-stage build mainly for reducing image size or also for security?

**Both — and security is equally important:**

**Size benefits:**
- Faster image pulls from Docker Hub/ECR
- Less storage cost in registries
- Faster container startup
- Less network bandwidth in Kubernetes deployments

**Security benefits:**
- No compiler/build tools in prod = fewer attack vectors
- No source code exposed in final image
- No `node_modules` = no dev dependencies with known CVEs in prod
- Smaller image = smaller attack surface (fewer OS packages to exploit)
- Build secrets (API keys used during build) don't leak into final image

### Q: When should I use `node:alpine` vs `node:18`?

| Image | Size | Use When |
|---|---|---|
| `node:18` | ~1.1GB | Development, when debugging tools are needed |
| `node:18-slim` | ~250MB | CI/CD builds, good balance |
| `node:18-alpine` | ~170MB | Production builds, when smallest size matters |

**Alpine caveat:** Alpine uses `musl libc` instead of `glibc`. Some npm packages with native C bindings may behave differently. Always test before using alpine in production.

**Best practice for multi-stage:**
```dockerfile
FROM node:18-alpine AS build   # Use alpine for build stage too — saves build time
```

---

## 10. Runtime vs Build-Time

### Q: Why do we need Node.js only in Stage 1 and not in Stage 2?

Node.js serves two completely different purposes in a React project:

| Purpose | Needed? | Stage |
|---|---|---|
| Running `npm install` | ✅ | Stage 1 only |
| Running `npm run build` | ✅ | Stage 1 only |
| **Serving the built HTML/CSS/JS** | ❌ Node not needed | Apache/Nginx in Stage 2 |

`npm run build` transforms your React components (JSX, ES6+, TypeScript) into plain HTML, CSS, and JavaScript files. **This output has zero dependency on Node.js.** Any web server — Apache, Nginx, even Python's http.server — can serve these static files.

### Q: What exactly is inside `/app/build`? Is it pure HTML/CSS/JS?

Yes. After `npm run build`, the `/app/build` folder contains:

```
/app/build/
├── index.html              ← Entry point (1-2KB)
├── static/
│   ├── js/
│   │   ├── main.abc123.js  ← All your React code, bundled & minified
│   │   └── chunk.xyz.js    ← Code-split chunks
│   ├── css/
│   │   └── main.abc.css    ← All styles, minified
│   └── media/
│       └── logo.abc.png    ← Images, fonts, etc.
└── asset-manifest.json     ← Manifest of all files
```

This is standard browser-compatible HTML/CSS/JS. **No Node.js, no JSX, no TypeScript.** Any web server can serve it.

### Q: Can a React app run without Node in production?

**Yes, absolutely.** Once built, a React app is just static files. What you DO need is:
- A web server (Apache, Nginx) to serve the files over HTTP/HTTPS
- The server must be configured to redirect all routes to `index.html` (for React Router to work)

What you do NOT need:
- Node.js runtime
- npm
- node_modules

### Q: `npm start` vs `npm run build` — what's the difference in real deployments?

| Command | What it does | Used in |
|---|---|---|
| `npm start` | Starts dev server with hot reload, source maps, no minification | **Development only** |
| `npm run build` | Creates optimized, minified, production-ready static bundle | **Production / CI-CD** |

**Never use `npm start` in production Docker images.** It's slower, includes dev tools, and isn't optimized. Always use `npm run build` + a proper web server.

---

## 11. Web Server Choice — Apache vs Nginx

### Q: Why Apache (httpd) here? Why not Nginx?

Both work perfectly for serving React static files. The class used Apache (`httpd:2.4`) because it's a widely-known, well-documented server. There's no functional difference for simple static file serving.

### Apache vs Nginx Comparison:

| Feature | Apache (httpd) | Nginx |
|---|---|---|
| Docker image size | ~118MB | ~133MB (similar) |
| Config file | `httpd.conf` | `nginx.conf` |
| Config style | Module-based | Block-based |
| Static file performance | Good | Slightly better |
| Reverse proxy | Yes | Yes (better for microservices) |
| `.htaccess` support | ✅ Yes | ❌ No |
| Industry adoption | Very high | Very high |
| React Router support | Config needed | Config needed |

### For React with React Router, Nginx config example:
```nginx
# nginx.conf
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # This is critical: redirect all paths to index.html
    # so React Router handles the routing
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### For React with React Router, Apache config:
```apache
# .htaccess or httpd.conf
<Directory /usr/local/apache2/htdocs>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
    RewriteEngine On
    RewriteBase /
    RewriteRule ^index\.html$ - [L]
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule . /index.html [L]
</Directory>
```

### Q: What do companies actually use in production?

In most modern production setups:
- **Nginx** is more popular for containerized React apps (lighter, simpler config)
- **Apache** is more common in legacy/enterprise environments
- **CDN + S3** (AWS CloudFront + S3) is the most common pattern for React in cloud — no web server container needed at all!
- **Nginx** is also commonly used as a reverse proxy in front of API services

---

## 12. Security & Best Practices

### Q: Is excluding build tools from final image a best practice?

**Yes — it's a fundamental DevSecOps principle.** Principle of Least Privilege applied to containers:

> *"A container should only contain what it needs to do its job at runtime — nothing more."*

### Q: How does multi-stage improve security?

**Attack surface reduction:**
```
image1 (single-stage):                image2 (multi-stage):
- Node.js runtime (attack vector)     - Apache only
- npm (can be exploited)              - Your static files
- node_modules (~1000 packages,       - Minimal OS
  many with known CVEs)               
- Build tools, compilers              No way to:
- Source code (intellectual prop.)    - Execute Node commands
                                      - Read source code
                                      - Access dev secrets
```

A CVE (security vulnerability) in a dev dependency inside `node_modules` **cannot be exploited** if `node_modules` is not in your production image.

### Q: Can `.env` files accidentally go into the final image?

**Yes — this is a common and dangerous mistake.** Here's how it happens and how to prevent it:

**Risk scenario:**
```dockerfile
COPY . .          # This copies .env file if it exists in your project folder!
RUN npm run build # .env values get baked into JS bundle
```

**Prevention methods:**

1. **Use `.dockerignore`** (most important):
```
# .dockerignore
.env
.env.local
.env.production
.env.*
node_modules
.git
```

2. **Pass env vars at runtime, not build time:**
```bash
docker run -e DB_PASSWORD=secret myapp
# or
docker run --env-file .env myapp
```

3. **Use Docker secrets (for production/Swarm/Kubernetes):**
```yaml
# docker-compose.yml
secrets:
  db_password:
    file: ./db_password.txt
```

4. **For React specifically:** Environment variables prefixed with `REACT_APP_` get baked into the JS bundle at build time. **Never put secrets in `REACT_APP_*` variables.** They will be visible in the browser.

---

## 13. Docker Internals — Deep Dive

### Q: Are intermediate stages stored as images or discarded?

Intermediate stages are **not stored as named images**. They exist as temporary layer caches during the build. After the build completes:

- The final image (Stage 2) is named and stored in your local Docker image store
- Stage 1's layers are kept in the **build cache** for faster future rebuilds (if nothing changed)
- They do NOT appear in `docker images` output
- Run `docker system prune` to clear build cache and reclaim disk space

```bash
docker images        # Only shows final images — intermediate stages not visible
docker system df     # Shows how much space build cache is using
docker builder prune # Clears build cache
```

### Q: Can I reuse a previous stage as base for another stage?

**Yes.** You can chain stages:

```dockerfile
FROM node:18-alpine AS deps
WORKDIR /app
COPY package.json ./
RUN npm install

FROM deps AS build         # Stage 2 is based on Stage 1!
COPY . .
RUN npm run build

FROM httpd:2.4 AS production
COPY --from=build /app/build/ /usr/local/apache2/htdocs/
```

This is useful for:
- Separating dependency installation from build (better caching)
- Creating test stages that share the same deps layer
- Creating debug-friendly intermediate stages

### Q: What happens if I don't name stages?

You can reference stages by their **0-indexed number**:

```dockerfile
FROM node:18          # Stage 0
WORKDIR /app
RUN npm install && npm run build

FROM httpd:2.4        # Stage 1
COPY --from=0 /app/build/ /usr/local/apache2/htdocs/
```

This works but is **fragile** — if you insert a stage, all numbers shift and you must update all references. Always prefer `AS stagename`.

### Q: Is `--from=build` referencing a container, image, or layer?

It's referencing a **build stage snapshot** — specifically the final filesystem state of that named stage after all its instructions have been executed. It is:
- **NOT** a running container
- **NOT** a named Docker image
- **NOT** a Docker Hub image (though `--from` can also reference external images)

It is the **BuildKit intermediate layer** of that stage, held in memory or build cache during the build process.

**Bonus:** You can also `COPY --from` an external Docker image:
```dockerfile
COPY --from=nginx:alpine /etc/nginx/nginx.conf /etc/nginx/nginx.conf
```
This pulls the file directly from the `nginx:alpine` image without creating a full stage.

---

## 14. CI/CD & Production Scenarios

### Q: How would this work in a CI/CD pipeline (Jenkins/GitHub Actions)?

```
Developer pushes code to GitHub
        ↓
CI/CD Pipeline Triggers (Jenkins/GitHub Actions)
        ↓
  Stage 1: docker build -t myapp .
  (Multi-stage build runs inside CI agent)
        ↓
  Stage 2: docker push myregistry/myapp:v1.0
  (Push ONLY the final lean image to registry)
        ↓
  Stage 3: Deploy to production
  docker pull myregistry/myapp:v1.0
  docker run -d -p 80:80 myregistry/myapp:v1.0
```

**The intermediate stages never touch production. Only the final small image is pushed and pulled.**

### Jenkins Pipeline example:
```groovy
pipeline {
    agent any
    stages {
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
            }
        }
        stage('Push to Registry') {
            steps {
                sh 'docker push myregistry/myapp:${BUILD_NUMBER}'
            }
        }
        stage('Deploy') {
            steps {
                sh 'docker pull myregistry/myapp:${BUILD_NUMBER}'
                sh 'docker run -d -p 80:80 myregistry/myapp:${BUILD_NUMBER}'
            }
        }
    }
}
```

### Q: Where does Docker cache help in this build?

Docker caches each layer. If nothing changed in that layer's inputs, it reuses the cached result:

```dockerfile
COPY package.json package-lock.json ./   ← Layer A
RUN npm install                           ← Layer B (expensive — ~30s)
COPY . .                                  ← Layer C (changes with every code push)
RUN npm run build                         ← Layer D
```

**If only your source code changed (Layer C):**
- Layer A: cache hit ✅ (package.json unchanged)
- Layer B: cache hit ✅ (`npm install` result cached — saves 30+ seconds!)
- Layer C: cache miss ❌ (new source code)
- Layer D: re-runs ❌ (must rebuild)

**This is why `COPY package.json ./` and `RUN npm install` come BEFORE `COPY . .`** — it separates the slow dependency install from the fast code copy, maximizing cache hits.

### Q: If only code changes, will `npm install` run again?

**No — if you structure your Dockerfile correctly** (as shown above). Docker sees that `package.json` hasn't changed, so the layer with `npm install` is still valid from cache. Only the code copy and build steps re-execute.

**Wrong order (cache busted every time):**
```dockerfile
COPY . .             # Code changes every push → cache miss
RUN npm install      # Runs every time even if packages didn't change!
```

**Right order (cache preserved):**
```dockerfile
COPY package.json package-lock.json ./   # Only changes when packages change
RUN npm install                           # Cached unless packages change
COPY . .                                  # Changes with every code push
RUN npm run build
```

### Q: How to handle environment variables in React production builds?

React build-time env vars (`REACT_APP_*`) are baked into the bundle:

```bash
# Pass at docker build time via --build-arg
docker build --build-arg REACT_APP_API_URL=https://api.mysite.com -t myapp .
```

```dockerfile
ARG REACT_APP_API_URL
ENV REACT_APP_API_URL=$REACT_APP_API_URL
RUN npm run build
```

**For runtime config (better approach):** Serve a `/config.js` file from your web server that the React app fetches on load, so you can change config without rebuilding the image.

---

## 15. Troubleshooting & Edge Cases

### Q: What happens if the build fails in Stage 1?

Stage 2 never starts. Docker stops at the failed instruction and reports an error. The partial Stage 1 layers may remain in build cache.

**Debugging tip:**
```bash
docker build --no-cache -t myapp .   # Skip cache, rebuild from scratch
docker build --progress=plain -t myapp .  # See full verbose output
```

### Q: Can I debug inside an intermediate stage?

**Yes!** Use `--target` to build only up to a specific stage:

```bash
# Build only Stage 1 and name it for inspection
docker build --target build -t debug-stage .

# Then run it interactively
docker run -it debug-stage /bin/sh

# Inspect what's in /app/build
ls /app/build/
```

This is extremely useful when your multi-stage build is failing and you want to explore the intermediate filesystem.

### Q: How to run only Stage 1 separately?

```bash
docker build --target build -t my-build-stage .
docker run -it my-build-stage /bin/sh
```

### Q: How to inspect contents of `/app/build` inside the final image?

```bash
# Run the container and exec into it
docker run -dt --name inspect-container image2
docker exec -it inspect-container /bin/sh

# Explore the htdocs directory
ls /usr/local/apache2/htdocs/
cat /usr/local/apache2/htdocs/index.html
```

### Important: Alpine containers use `/bin/sh`, NOT `/bin/bash`

From class:
```bash
# ❌ Fails on Alpine-based images
docker exec -it ab4e7291448c /bin/bash
# Error: OCI runtime exec failed: exec: "/bin/bash": stat /bin/bash: no such file

# ✅ Works on Alpine
docker exec -it ab4e7291448c /bin/sh
```

### Common Dockerfile Syntax Error from Class:

**Error encountered:**
```
ERROR: failed to solve: dockerfile parse error on line 3:
FROM requires either one or three arguments
```

**Cause:** Inline comment on `FROM` line:
```dockerfile
FROM node:18 as build #take slim version of node image  ← BROKEN
```

**Fix:**
```dockerfile
# take slim version of node image  ← Comment on separate line
FROM node:18 AS build               ← Clean FROM line
```

Also note: `AS` keyword is case-insensitive (`AS` or `as` both work), but `AS` in uppercase is the convention.

---

## 16. Advanced Topics

### Q: Can we create multi-stage builds for Spring Boot?

**Yes — the class demonstrated exactly this.** Here's the full pattern:

```dockerfile
# Stage 1: Maven build
FROM maven:3.9.9-amazoncorretto-11 AS builder
WORKDIR /app
COPY pom.xml .                    # Copy pom.xml first for dependency caching
RUN mvn dependency:go-offline     # Pre-download deps (cache optimization)
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Minimal JRE runtime
FROM amazoncorretto:11-alpine-jdk
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8084
CMD ["java", "-jar", "app.jar"]
```

Same idea works for Python (pip install → slim runtime), Go (build binary → scratch/distroless), .NET (build → ASP.NET runtime), etc.

### Q: How does this compare with Kubernetes deployments?

In Kubernetes (K8s), you don't run `docker run` manually. Instead:
1. Your multi-stage Dockerfile builds the final lean image
2. You push it to a registry (Docker Hub, ECR, GCR)
3. Kubernetes pulls the image and runs it as a Pod
4. The smaller the image, the faster K8s scales (faster pod startup)
5. Less storage cost in the registry
6. Multi-stage is **even more important** in K8s because images are pulled frequently across many nodes

### Q: Can we push only the final image to Docker Hub?

**Yes — that's exactly what happens automatically.** When you run:
```bash
docker push username/myapp:latest
```
Only the final stage's image is pushed. Intermediate stages exist only during the local build and are never pushed.

### Q: Is multi-stage supported in all Docker versions?

Multi-stage builds were introduced in **Docker 17.05** (released May 2017). Any Docker version from 2017 onwards supports it. You should rarely encounter an environment without multi-stage support today.

Check your Docker version:
```bash
docker --version
# Docker version 24.x.x → fully supported
```

---

## 17. Quick Reference Cheatsheet

### Multi-Stage Dockerfile Template — React + Apache
```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build

FROM httpd:2.4
COPY --from=build /app/build/ /usr/local/apache2/htdocs/
EXPOSE 80
CMD ["httpd-foreground"]
```

### Multi-Stage Dockerfile Template — Spring Boot + Maven
```dockerfile
FROM maven:3.9.9-amazoncorretto-11 AS builder
WORKDIR /app
COPY . /app
RUN mvn clean package -DskipTests

FROM amazoncorretto:11-alpine-jdk
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

### Essential Commands
```bash
# Build multi-stage image
docker build -t myapp .

# Build only a specific stage (for debugging)
docker build --target build -t debug-image .

# Run with port mapping
docker run -dt -p 8085:8084 backend
docker run -dt -p 81:80 image2

# View image size
docker images

# View image layer history
docker history image2
docker history image1

# Exec into container
docker exec -it <container_id> /bin/sh     # Alpine
docker exec -it <container_id> /bin/bash   # Debian/Ubuntu

# Check processes inside container
docker exec -it <container_id> /bin/sh -c "ps aux"

# Clear build cache
docker builder prune

# Remove unused images/containers
docker system prune
```

### Size Comparison Summary

| Image | Type | Size | Production Ready |
|---|---|---|---|
| `image1` | Single-stage Node | 1.45GB | ❌ Never |
| `image2` | Multi-stage → Apache | 118MB | ✅ Yes |
| `backend` | Multi-stage Maven → JRE | 336MB | ✅ Yes |

### When to Use What

| Scenario | Recommended Base |
|---|---|
| React frontend | `node:18-alpine` → `httpd:2.4` or `nginx:alpine` |
| Spring Boot backend | `maven:3.9.9` → `amazoncorretto:alpine-jdk` |
| Python Flask/Django | `python:3.11` → `python:3.11-slim` |
| Go application | `golang:1.21` → `scratch` (0MB base!) |
| Node.js API | `node:18-alpine` → `node:18-alpine` (slim version) |

---

## 📝 Class Notes Summary

| Concept | Key Takeaway |
|---|---|
| Multi-stage = 2-server model | Stage 1 = Build Server, Stage 2 = App Server — same principle as Maven→Tomcat |
| `FROM` creates fresh filesystem | Each stage is completely isolated, like a new server |
| `COPY --from=` bridges stages | Surgically copies only what's needed from one stage to another |
| image1 = 1.45GB, image2 = 118MB | Multi-stage removed Node.js, npm, node_modules, source code |
| Alpine = no `/bin/bash` | Use `/bin/sh` for Alpine-based images |
| Comments on `FROM` line = error | Always put comments on their own line above `FROM` |
| Stage 1 is discarded | Intermediate stages never end up in the final image |
| Docker cache ordering matters | `COPY package.json` → `RUN npm install` → `COPY . .` for best cache hits |
| Security = primary benefit | No build tools in prod = smaller attack surface |
| `--target stagename` = debug | Build only up to a specific stage to inspect intermediate state |

---

> 💡 **Next Steps:** Explore Docker Compose for running multi-container apps (React frontend + Spring Boot backend + MySQL) as a unified stack.

> 📦 **GitHub Repos from Class:**
> - Frontend: [mluticloud730am/2nd10WeeksofCloudOps](https://github.com/mluticloud730am/2nd10WeeksofCloudOps-main_20260502)
> - Backend: [mluticloud730am/Java-springboot-project_1](https://github.com/mluticloud730am/Java-springboot-project_1)
