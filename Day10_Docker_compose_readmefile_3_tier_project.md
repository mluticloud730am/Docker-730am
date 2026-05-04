# 📚 AuthorBook — 3-Tier Dockerized Application
### 2nd 10 Weeks of CloudOps | Lokesh Edition

> A fully containerized, production-style 3-tier web application built with React, Node.js/Express, and MySQL — deployed on AWS EC2 using Docker Compose.  
> Built and debugged hands-on as part of the **CloudOps learning program by Naresh IT / Lokesh Sir**.

---

## 📋 Table of Contents

1. [What Is This Project?](#-what-is-this-project)
2. [Architecture Overview](#-architecture-overview)
3. [Project Folder Structure](#-project-folder-structure)
4. [Tech Stack](#-tech-stack)
5. [How Each Service Works](#-how-each-service-works)
   - [Frontend (React + Apache)](#1-frontend-react--apache-httpd)
   - [Backend (Node.js + Express + PM2)](#2-backend-nodejs--express--pm2)
   - [MySQL (Custom Docker Image)](#3-mysql-custom-docker-image)
6. [Docker Compose Deep Dive](#-docker-compose-deep-dive)
7. [Step-by-Step Setup Guide](#-step-by-step-setup-guide)
8. [Real Bugs We Hit & How We Fixed Them](#-real-bugs-we-hit--how-we-fixed-them)
9. [Key Concepts Explained (Beginner + Advanced)](#-key-concepts-explained-beginner--advanced)
10. [Deployment Modes](#-deployment-modes)
11. [Kubernetes & EKS Deployment](#-kubernetes--eks-deployment)
12. [Jenkins CI/CD Pipeline](#-jenkins-cicd-pipeline)
13. [Common Commands Cheat Sheet](#-common-commands-cheat-sheet)
14. [Troubleshooting Guide](#-troubleshooting-guide)

---

## 🌟 What Is This Project?

**AuthorBook** is a simple book management web application. Users can view books with their title, description, price, and cover image.

It is intentionally designed as a **3-tier architecture** to teach real-world DevOps/CloudOps concepts:

| Tier | Technology | Purpose |
|------|-----------|---------|
| Presentation (Layer 1) | React + Apache HTTPD | What the user sees in the browser |
| Application (Layer 2) | Node.js + Express + PM2 | Business logic, REST API |
| Data (Layer 3) | MySQL 8.0 | Stores book records persistently |

**For Freshers:** Think of it like a restaurant. The menu you see is the frontend. The chef in the kitchen is the backend. The recipe book is the database. Each layer talks to the next one — they don't skip layers.

**For Experienced Engineers:** This project demonstrates Docker multi-stage builds, inter-container DNS resolution on a custom bridge network, healthcheck-based startup ordering, PM2 process management inside containers, and Apache httpd serving a React SPA build.

---

## 🏗️ Architecture Overview

```
                        ┌─────────────────────────────────────────────┐
                        │                 AWS EC2 Instance             │
                        │              (Amazon Linux, t2.medium)       │
                        │                                              │
  Browser               │  ┌──────────────┐      ┌─────────────────┐  │
  User  ──── Port 80 ──►│  │  FRONTEND    │      │   BACKEND       │  │
                        │  │  React App   │─────►│   Node.js API   │  │
                        │  │  Apache HTTPD│      │   Express + PM2 │  │
                        │  │  Port 80     │      │   Port 80       │  │
                        │  └──────────────┘      └────────┬────────┘  │
                        │                                 │            │
                        │                        ┌────────▼────────┐  │
                        │                        │   MYSQL DB      │  │
                        │                        │   MySQL 8.0     │  │
                        │                        │   Port 3306     │  │
                        │                        │   test.sql init │  │
                        │                        └─────────────────┘  │
                        │                                              │
                        │   All on Docker bridge network: appnet       │
                        └─────────────────────────────────────────────┘
```

**For Freshers:** All 3 containers live inside the same EC2 machine and talk to each other over a private Docker network called `appnet`. From outside, you can only reach the frontend (port 80) and backend (port 84). MySQL is internal only.

**For Experienced Engineers:** Docker creates a software-defined bridge network. Each container gets a DNS name equal to its service name (`mysql`, `backend`, `frontend`). Container-to-container traffic never leaves the host. Port mappings `host:container` expose selected ports externally.

---

## 📁 Project Folder Structure

```
2nd10WeeksofCloudOps_authorbook_Lokesh/
│
├── docker-compose.yaml          # Orchestrates all 3 services
├── architecture.gif             # Architecture diagram
│
├── client/                      # FRONTEND — React Application
│   ├── Dockerfile               # Multi-stage: build React → serve with Apache
│   ├── src/
│   │   └── pages/
│   │       └── config.js        # ⚠️ API base URL — must match backend port
│   ├── public/
│   └── package.json
│
├── backend/                     # BACKEND — Node.js REST API
│   ├── Dockerfile               # Single-stage: install deps, run with PM2
│   ├── index.js                 # Express app — listens on port 80
│   └── package.json
│
├── mysql/                       # DATABASE — Custom MySQL image
│   ├── Dockerfile               # Extends mysql:8.0, copies init SQL
│   └── test.sql                 # Auto-runs on first container start
│
├── kubernetes-files/            # K8s manifests for EKS deployment
├── eks-terraform/               # Terraform code to provision EKS cluster
├── terraform_main_ec2/          # Terraform code to provision EC2
├── rds/                         # Terraform/config for RDS MySQL
└── Jenkins-Pipeline-Code/       # Jenkins CI/CD pipeline definitions
```

---

## 🛠️ Tech Stack

| Component | Technology | Why |
|-----------|-----------|-----|
| Frontend Build | React 18 | Component-based SPA |
| Frontend Server | Apache HTTPD 2.4 | Efficient static file serving |
| Backend Runtime | Node.js 18 | Non-blocking I/O for APIs |
| Backend Framework | Express.js | Lightweight REST API |
| Process Manager | PM2 | Keeps Node.js alive, auto-restarts |
| Database | MySQL 8.0 | Relational data, industry standard |
| Containerization | Docker + Docker Compose | Portable, reproducible environments |
| Cloud Platform | AWS EC2 (Amazon Linux) | Real-world cloud deployment |
| IaC | Terraform | Infrastructure as Code for EKS/EC2 |
| CI/CD | Jenkins | Automated build and deploy pipeline |

---

## 🔍 How Each Service Works

### 1. Frontend (React + Apache HTTPD)

**Location:** `./client/`

```dockerfile
# Stage 1: Build the React app
FROM node:18 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
RUN npm run build               # Produces /app/build/ (static HTML/JS/CSS)

# Stage 2: Serve with Apache
FROM httpd:2.4
COPY --from=build /app/build /usr/local/apache2/htdocs/
EXPOSE 80
CMD ["httpd-foreground"]
```

**For Freshers — What is a multi-stage build?**
- Stage 1 uses a Node.js image to compile your React code into plain HTML, CSS, and JavaScript files (the `build/` folder). This is like a carpenter's workshop — you need tools to build, but not the tools to deliver.
- Stage 2 uses a tiny Apache image to serve those files. The final image does NOT contain Node.js, npm, or your source code — just the compiled output.
- Result: final image is much smaller (~150MB vs ~1GB) and more secure.

**For Experienced Engineers:** This demonstrates Docker multi-stage builds for SPA optimization. The `npm run build` step compiles JSX, bundles via webpack/Create React App, and tree-shakes unused code. Apache `htdocs` is the document root. No server-side rendering — Apache serves `index.html` for all routes (React Router handles client-side navigation).

**Critical config file — `client/src/pages/config.js`:**
```javascript
// This URL is BAKED INTO THE BUILD at compile time
// If you change it, you MUST rebuild the frontend image

const API_BASE_URL = "http://<YOUR-EC2-PUBLIC-IP>:84"
export default API_BASE_URL;
```

⚠️ **This is why frontend needs `--build` after any config change.**

---

### 2. Backend (Node.js + Express + PM2)

**Location:** `./backend/`

```dockerfile
FROM node:18 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
RUN npm install -g pm2          # Process manager installed globally
COPY . .
EXPOSE 80
CMD ["pm2-runtime", "start", "index.js", "--name", "backendapi"]
```

**Key facts about `index.js`:**
- Connects to MySQL using the `mysql2` library
- Reads DB credentials from environment variables (`process.env.DB_HOST`, etc.)
- Listens on **port 80** (hardcoded — not port 3000 or 8080!)
- `/books` endpoint returns all book records as JSON

**For Freshers — What is PM2?**
PM2 is a process manager for Node.js. Without PM2, if your Node.js app crashes, the container dies. PM2 automatically restarts it. `pm2-runtime` is the Docker-friendly version that keeps the container alive by running in the foreground.

**For Experienced Engineers:** `pm2-runtime` replaces the daemon mode (which would cause the container to exit immediately). The backend uses `mysql2` with a connection that is established at startup — if MySQL isn't ready, PM2 retries 16 times before giving up. This is why we need the healthcheck in `docker-compose.yaml`.

---

### 3. MySQL (Custom Docker Image)

**Location:** `./mysql/`

```dockerfile
FROM mysql:8.0
EXPOSE 3306
COPY test.sql /docker-entrypoint-initdb.d/   # Auto-executed on first start
CMD ["mysqld"]
```

**For Freshers — What is `/docker-entrypoint-initdb.d/`?**
The official MySQL Docker image has a special feature: any `.sql` file placed in `/docker-entrypoint-initdb.d/` is automatically executed when the container starts for the first time. This creates your database tables and inserts sample data without you having to run anything manually.

**For Experienced Engineers:** The init scripts run only when the data directory is empty (first boot). If you mount a persistent volume and the data already exists, `test.sql` will NOT re-run. Environment variables `MYSQL_DATABASE`, `MYSQL_USER`, and `MYSQL_PASSWORD` are processed by the entrypoint script to create the database and user before `mysqld` starts accepting connections.

**test.sql creates:**
- The `test` database (matching `MYSQL_DATABASE=test`)
- A `books` table with id, title, desc, price, cover columns
- Sample book records (MultiCloud book, DevOps book)

---

## 🐳 Docker Compose Deep Dive

```yaml
services:

  frontend:
    build:
      context: ./client          # Build from the client/ directory
      dockerfile: Dockerfile
    container_name: frontend-app
    ports:
      - "80:80"                  # host_port:container_port
    networks:
      - appnet
    depends_on:
      - backend                  # Start after backend (but doesn't wait for ready)

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: backend-app
    ports:
      - "84:80"                  # Host port 84 → Container port 80 (app.listen(80))
    environment:
      - DB_HOST=mysql            # 'mysql' = the service name = DNS name on appnet
      - DB_USERNAME=admin
      - DB_PASSWORD=Devops123    # Must match MYSQL_PASSWORD below
      - DB_PORT=3306
    networks:
      - appnet
    depends_on:
      mysql:
        condition: service_healthy   # Wait until MySQL passes healthcheck

  mysql:
    build:
      context: ./mysql
      dockerfile: Dockerfile
    container_name: mysql-db
    environment:
      - MYSQL_ROOT_PASSWORD=Cloud122
      - MYSQL_DATABASE=test
      - MYSQL_USER=admin
      - MYSQL_PASSWORD=Devops123     # Must match backend DB_PASSWORD
    ports:
      - "3306:3306"
    networks:
      - appnet
    healthcheck:
      # Run mysqladmin ping every 5 seconds
      # If it succeeds → container is "healthy" → backend can start
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "admin", "-pDevops123"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 30s        # Wait 30s before first check (for test.sql to run)

networks:
  appnet:
    driver: bridge             # Software-defined Layer 2 network for all 3 containers
```

**For Freshers — Key concepts:**

- **`ports: "84:80"`** means: "if someone connects to my EC2 on port 84, forward it to port 80 inside the container." Think of it as call-forwarding.
- **`DB_HOST=mysql`** — Docker automatically gives each container a DNS name equal to its service name. So `mysql` resolves to the IP of the mysql container on the `appnet` network.
- **`depends_on` with `condition: service_healthy`** — This is the correct way to say "don't start backend until MySQL is actually ready to accept connections." Plain `depends_on` only waits for the container to start, not for the app inside to be ready.

**For Experienced Engineers:**

- The bridge network creates a private subnet (typically 172.20.0.0/16). Each container gets an IP and a DNS A record matching its service name, managed by Docker's embedded DNS resolver.
- `service_healthy` condition uses the healthcheck exit code (0 = healthy). The `start_period` is crucial here — MySQL's `mysqld` starts in seconds, but running `test.sql` init scripts can take 20–30 seconds. Without `start_period`, healthchecks would fail immediately and the container would be marked unhealthy before init is done.

---

## 🚀 Step-by-Step Setup Guide

### Prerequisites

```bash
# Check Docker is installed
docker --version          # Need 25.0+

# Check buildx version (need 0.17.0+)
docker buildx version

# If buildx is old, upgrade it:
BUILDX_VERSION=$(curl -s https://api.github.com/repos/docker/buildx/releases/latest \
  | grep '"tag_name"' | cut -d'"' -f4)
mkdir -p ~/.docker/cli-plugins
curl -SL "https://github.com/docker/buildx/releases/download/${BUILDX_VERSION}/buildx-${BUILDX_VERSION}.linux-amd64" \
  -o ~/.docker/cli-plugins/docker-buildx
chmod +x ~/.docker/cli-plugins/docker-buildx

# Install Docker Compose plugin v2
COMPOSE_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest \
  | grep '"tag_name"' | cut -d'"' -f4)
curl -SL "https://github.com/docker/compose/releases/download/${COMPOSE_VERSION}/docker-compose-linux-x86_64" \
  -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose

# Verify
docker compose version    # Should show v2.x.x
docker buildx version     # Should show 0.17.0+
```

### Step 1: Clone the Repository

```bash
git clone https://github.com/mluticloud730am/2nd10WeeksofCloudOps_authorbook_Lokesh.git
cd 2nd10WeeksofCloudOps_authorbook_Lokesh
```

### Step 2: Set Your EC2 Public IP in Frontend Config

```bash
# Find your EC2 public IP
curl -s http://checkip.amazonaws.com

# Edit the frontend API URL
vi client/src/pages/config.js
```

Change this line:
```javascript
const API_BASE_URL = "http://YOUR-EC2-PUBLIC-IP:84"
```

### Step 3: Start All Services

```bash
docker compose up -d --build
```

This will:
1. Build the MySQL image (copies test.sql)
2. Build the backend image (npm install + PM2)
3. Build the frontend image (npm run build + copy to Apache)
4. Start MySQL first and wait for it to be healthy (~30 seconds)
5. Start backend after MySQL is ready
6. Start frontend last

### Step 4: Verify Everything is Running

```bash
# All 3 containers should show "Up" or "healthy"
docker ps

# Test backend directly
curl http://localhost:84/books

# Expected output:
# [{"id":1,"title":"MultiCloud",...},{"id":2,"title":"DevOps",...}]
```

### Step 5: Open in Browser

```
http://YOUR-EC2-PUBLIC-IP/
```

You should see the AuthorBook web application with book cards.

---

## 🐛 Real Bugs We Hit & How We Fixed Them

These are real errors encountered during this lab — understanding them makes you a better engineer.

---

### Bug 1: `compose build requires buildx 0.17.0 or later`

**What happened:** `docker compose up -d` failed immediately without building anything.

**Root cause:** Amazon Linux EC2 ships with an old Docker from yum repos. Buildx 0.12.1 was installed, but Docker Compose v2 requires ≥ 0.17.0 for parallel builds.

**Fix:** Manually download latest buildx binary from GitHub releases to `~/.docker/cli-plugins/`.

**Lesson:** Always check tool versions when using EC2 with pre-installed packages. Yum/DNF repos often lag behind upstream releases by months.

---

### Bug 2: `mysql -h endpoint -u admin -p cloud123` → `ERROR 1049: Unknown database 'cloud123'`

**What happened:** Tried to connect to MySQL RDS from the command line but got an error about unknown database.

**Root cause:** The `mysql` CLI syntax is `-p` (no space) for inline password, or `-p` alone to prompt. Writing `-p cloud123` passes `cloud123` as the **database name**, not the password!

**Fix:**
```bash
# Wrong:
mysql -h endpoint -u admin -p cloud123

# Correct (prompt for password):
mysql -h endpoint -u admin -p

# Correct (inline, no space):
mysql -h endpoint -u admin -pcloud123
```

**Lesson:** CLI argument parsing is strict. `-p password` ≠ `-ppassword` in the MySQL client.

---

### Bug 3: `ER_ACCESS_DENIED_ERROR` — Backend Can't Connect to MySQL

**What happened:** Backend crashed repeatedly with `Access denied for user 'admin'`.

**Root cause:** Password mismatch.
- Backend env had: `DB_PASSWORD=cloud123`
- MySQL container expected: `MYSQL_PASSWORD=Devops123`

**Fix:** Make both match:
```yaml
backend:
  environment:
    - DB_PASSWORD=Devops123   # match this

mysql:
  environment:
    - MYSQL_PASSWORD=Devops123  # to this
```

**Lesson:** In multi-service apps, credentials must be consistent across all services. Always cross-check env vars between services that talk to each other.

---

### Bug 4: `ECONNREFUSED 172.20.0.2:3306` — Backend Can't Reach MySQL

**What happened:** Even with correct credentials, backend kept crashing with connection refused.

**Root cause:** Race condition. Backend started before MySQL was ready to accept connections. `depends_on: mysql` only means "start the container" — it doesn't wait for MySQL daemon to finish initialization and run `test.sql`.

**Fix:** Add a healthcheck to MySQL and use `condition: service_healthy`:
```yaml
depends_on:
  mysql:
    condition: service_healthy

mysql:
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "admin", "-pDevops123"]
    interval: 5s
    retries: 10
    start_period: 30s
```

**Lesson:** Container startup ≠ application readiness. Always use healthchecks for databases. This is a very common production issue.

---

### Bug 5: Backend Port Mismatch — `84:3000` vs `app.listen(80)`

**What happened:** Backend container started but `curl http://localhost:84/books` got `Connection reset by peer`.

**Root cause:** The Dockerfile had `EXPOSE 3000` (misleading comment), but `index.js` actually had `app.listen(80)`. Compose mapped `84:3000`, so traffic went to port 3000 in the container — which nothing was listening on.

**Fix:**
```yaml
ports:
  - "84:80"    # right side must match app.listen() in index.js
```

**Lesson:** `EXPOSE` in Dockerfile is just documentation — it doesn't actually open any port. The `ports:` mapping in docker-compose is what matters. Always grep the source code to find the actual listening port:
```bash
grep -i "listen\|port" backend/index.js
```

---

### Bug 6: Frontend Calling Wrong Port `:86` Instead of `:84`

**What happened:** Browser showed `Failed to fetch` when loading books. Network tab showed `http://100.31.72.144:86/books`.

**Root cause:** `client/src/pages/config.js` had a stale port from a previous version of the setup.

**Fix:**
```javascript
// Change this:
const API_BASE_URL = "http://100.31.72.144:86"
// To this:
const API_BASE_URL = "http://100.31.72.144:84"
```

**Important:** Since React bakes this value into the compiled bundle at build time, you must **rebuild** the frontend image:
```bash
docker compose up -d --build frontend
```

**Lesson:** React (Create React App) is a static SPA. Values in `.js` config files are embedded at build time, not runtime. Any change to `config.js` requires a rebuild. This is why environment variable injection at runtime (via nginx sub_filter or entrypoint.sh) is the production-preferred approach.

---

## 📖 Key Concepts Explained (Beginner + Advanced)

### What is Docker Compose?

**For Freshers:** Docker Compose is like a recipe that says "I need 3 containers, here's how to build each one, how they connect, and what order to start them." Instead of running 3 separate `docker run` commands with 20 flags each, you write one `docker-compose.yaml` file and run `docker compose up`.

**For Experienced Engineers:** Docker Compose is a declarative orchestration tool for multi-container applications on a single host. It manages the full lifecycle: build, create, start, stop, scale, and teardown. It creates a default bridge network, resolves inter-service dependencies, and supports health-check-based startup ordering. It is NOT suitable for multi-host deployments (use Kubernetes/ECS for that).

---

### What is a Bridge Network?

**For Freshers:** Imagine Docker creates a private WiFi router inside your EC2. All 3 containers connect to it. They can talk to each other using their names (`mysql`, `backend`). Traffic between them never goes to the internet — it's all internal.

**For Experienced Engineers:** Docker bridge networks create a Linux bridge (virtual switch) on the host. Each container gets a veth pair — one end in the container's network namespace, the other connected to the bridge. Docker's embedded DNS resolver maps service names to container IPs. Containers on the same bridge network can communicate via any port without explicit `ports:` mapping.

---

### What is PM2?

**For Freshers:** PM2 is a bodyguard for your Node.js app. If the app crashes, PM2 restarts it. In Docker, we use `pm2-runtime` which is the version designed to run inside containers.

**For Experienced Engineers:** PM2 is a process manager that provides clustering, zero-downtime reloads, log management, and monitoring. `pm2-runtime` is important in containers because it keeps the process in the foreground (PID 1-compatible) and forwards signals correctly. Without it, the container exits when the process daemon detaches. Note: in production Kubernetes, liveness probes replace PM2's restart logic, so PM2 adds overhead without benefit.

---

### What is a Healthcheck?

**For Freshers:** A healthcheck is Docker asking "are you actually ready?" on a schedule. MySQL might be "running" (container started) but not "ready" (database initialized). The healthcheck tests this by running a real connection. Only when the test passes does Docker mark the container as "healthy" and allow dependent containers to start.

**For Experienced Engineers:** Healthchecks run a command inside the container and check the exit code. Exit 0 = healthy, exit 1 = unhealthy. Docker Compose uses the health status to implement `condition: service_healthy` in `depends_on`. The `start_period` is a grace period where failures are ignored — critical for slow-starting services. Kubernetes has equivalent `readinessProbe` and `livenessProbe` concepts.

---

## 🚀 Deployment Modes

This repo supports 3 deployment modes:

### Mode 1: Docker Compose (Single EC2) — This Guide

Used for: Learning, dev/test, demos.

```
EC2 (t2.medium)
└── docker compose up -d
    ├── frontend-app (port 80)
    ├── backend-app  (port 84)
    └── mysql-db     (port 3306)
```

### Mode 2: EC2 + RDS (Production-style, no Kubernetes)

Used for: Cost-effective production setups.

- Replace MySQL container with AWS RDS MySQL
- Update `DB_HOST` to RDS endpoint
- Remove `mysql:` service from docker-compose.yaml
- RDS gives you automated backups, Multi-AZ, and managed patching

```yaml
backend:
  environment:
    - DB_HOST=your-rds-endpoint.rds.amazonaws.com
    - DB_PASSWORD=your-rds-password
    # Remove depends_on: mysql
```

### Mode 3: EKS + Kubernetes (Production, scalable)

Used for: High-availability production workloads.

- Use Terraform in `eks-terraform/` to provision EKS cluster
- Apply Kubernetes manifests from `kubernetes-files/`
- Use AWS RDS for database (not a K8s pod)
- Use Jenkins pipeline from `Jenkins-Pipeline-Code/`

---

## ☸️ Kubernetes & EKS Deployment

The `kubernetes-files/` directory contains K8s manifests. General flow:

```bash
# 1. Provision EKS cluster via Terraform
cd eks-terraform/
terraform init
terraform apply

# 2. Configure kubectl
aws eks update-kubeconfig --name <cluster-name> --region us-east-1

# 3. Apply manifests
kubectl apply -f kubernetes-files/

# 4. Check deployments
kubectl get pods
kubectl get services
```

**For Freshers:** Kubernetes is like Docker Compose but for multiple servers. Instead of running on one EC2, your containers can run across many servers with automatic load balancing, self-healing, and scaling.

---

## 🔧 Jenkins CI/CD Pipeline

The `Jenkins-Pipeline-Code/` directory contains Jenkinsfile pipeline definitions for automated build and deploy.

**Pipeline flow:**
```
Code Push → GitHub Webhook → Jenkins Trigger
→ Build Docker Images
→ Push to ECR (Elastic Container Registry)
→ Deploy to EKS (kubectl apply)
→ Smoke Test
→ Notify
```

**For Freshers:** CI/CD means "every time you push code to GitHub, a robot automatically builds, tests, and deploys your application." Jenkins is that robot.

---

## 📌 Common Commands Cheat Sheet

```bash
# ── Build & Start ──────────────────────────────────────────
docker compose up -d                    # Start all services (detached)
docker compose up -d --build            # Rebuild images then start
docker compose up -d --build frontend   # Rebuild only frontend

# ── Stop & Clean ───────────────────────────────────────────
docker compose down                     # Stop and remove containers + network
docker compose down -v                  # Also remove volumes (wipes DB data!)

# ── Status & Logs ──────────────────────────────────────────
docker compose ps                       # Show service status + health
docker ps -a                            # Show all containers (including stopped)
docker logs backend-app                 # Logs for backend
docker logs -f backend-app              # Follow/live logs
docker logs mysql-db | tail -50         # Last 50 lines of MySQL logs

# ── Debug Inside Container ─────────────────────────────────
docker exec -it backend-app bash        # Shell into backend container
docker exec -it mysql-db bash           # Shell into MySQL container
docker exec -it mysql-db mysql -u admin -pDevops123 test   # MySQL CLI

# ── Network Debug ──────────────────────────────────────────
docker network ls                       # List all networks
docker network inspect appnet           # See IPs of containers on appnet

# ── Test API ───────────────────────────────────────────────
curl http://localhost:84/books          # Test backend from EC2
curl http://localhost:84/               # Test backend root

# ── MySQL from EC2 host ────────────────────────────────────
mysql -h 127.0.0.1 -P 3306 -u admin -pDevops123 test
# Note: use 127.0.0.1 not localhost (socket vs TCP)
```

---

## 🔎 Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `compose build requires buildx 0.17.0` | Buildx too old | Upgrade buildx binary from GitHub releases |
| Backend container `Exited (2)` | App crash — check logs | `docker logs backend-app` |
| `ECONNREFUSED :3306` | MySQL not ready / backend started too early | Add healthcheck + `condition: service_healthy` |
| `ER_ACCESS_DENIED_ERROR` | Password mismatch between backend env and MySQL env | Make `DB_PASSWORD` = `MYSQL_PASSWORD` |
| `curl: Connection reset by peer` | Wrong container port in compose | Check `app.listen(PORT)` in index.js, match in compose |
| Browser shows no books | Frontend config.js has wrong IP or port | Update config.js → rebuild frontend |
| `ERROR 1049: Unknown database` | Wrong mysql CLI syntax | Use `-p` prompt or `-pPASSWORD` (no space) |
| MySQL container healthy but tables missing | test.sql didn't run | Run `docker compose down -v` then `up --build` (clear volume) |
| Port already in use | Another process on same port | `sudo ss -tlnp \| grep <port>` to find it |

---

## 🔐 Security Notes (Important for Production)

1. **Never hardcode passwords** in docker-compose.yaml. Use Docker secrets or `.env` files (excluded from git via `.gitignore`).
2. **Never commit `.env` files** with real credentials to GitHub.
3. **MySQL port 3306** should NOT be exposed publicly on production EC2. Remove `ports: - "3306:3306"` from docker-compose — backend can still reach it via the internal `appnet` network.
4. **Frontend config.js** with hardcoded IP is a development shortcut. Use environment variable injection or a reverse proxy (`/api` prefix with Nginx) for production.
5. Use **AWS Secrets Manager** or **Parameter Store** for storing DB credentials in production.

---

## 👨‍💻 Author

**Rakesh** — 13 years in Application Support | Upskilling in DevOps/CloudOps  
Training: Naresh IT Technology | Lokesh Sir's CloudOps Program  
GitHub: [mluticloud730am](https://github.com/mluticloud730am)

---

## 🙏 Credits

- Original project by [CloudDevops-Tech / Lokesh Sir](https://github.com/CloudDevops-Tech/2nd10WeeksofCloudOps_authorbook)
- Part of the **2nd 10 Weeks of CloudOps** hands-on training program
- All bugs in this README were real bugs hit during the actual lab — documented to help future learners

---

*"Every bug you fix teaches you more than ten tutorials." — Real DevOps wisdom* 🚀
