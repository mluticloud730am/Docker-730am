# 🐳 Docker Day 4 — Full Project Notes & Cloud Deployment Guide
### MultiCloud DevOps Project: React + Node.js + MySQL on AWS

---

## 📌 Table of Contents

1. [Project Architecture Overview](#1-project-architecture-overview)
2. [What We Built Today — Quick Summary](#2-what-we-built-today--quick-summary)
3. [Docker Concepts Covered (Enriched)](#3-docker-concepts-covered-enriched)
4. [Security Groups Setup (EC2 + RDS)](#4-security-groups-setup-ec2--rds)
5. [RDS MySQL Setup](#5-rds-mysql-setup)
6. [Backend Container (Node.js)](#6-backend-container-nodejs)
7. [Frontend Container (React + Apache)](#7-frontend-container-react--apache)
8. [Push Images to Docker Hub](#8-push-images-to-docker-hub)
9. [Deploy on AWS ECS (Full Steps)](#9-deploy-on-aws-ecs-full-steps)
10. [ECS with ECR (Amazon Container Registry)](#10-ecs-with-ecr-amazon-container-registry)
11. [Common Errors & Fixes](#11-common-errors--fixes)
12. [Glossary for Freshers](#12-glossary-for-freshers)

---

## 1. Project Architecture Overview

```
                        ┌─────────────────────────────────────────┐
                        │              AWS Cloud (EC2)             │
                        │                                          │
  Browser               │   ┌──────────────┐  ┌────────────────┐  │
  (User) ──── Port 80 ──┼──►│   Frontend   │  │    Backend     │  │
                        │   │ React + HTTPD│  │  Node.js App   │  │
                        │   │  Container   │  │   Container    │  │
                        │   │  Port: 80    │  │   Port: 3000   │  │
                        │   └──────────────┘  └───────┬────────┘  │
                        │         ▲                   │           │
                        │         │ API calls         │           │
                        │         │ Port: 84          ▼           │
                        │         └──────────  ───────────────    │
                        │                     │  AWS RDS MySQL │   │
                        │                     │ Port: 3306     │   │
                        │                     └────────────────┘   │
                        └─────────────────────────────────────────┘
```

**Flow explanation (for freshers):**
- User opens browser → hits EC2 public IP on port 80 → sees React frontend
- React frontend makes API calls to backend on port 84 (which maps to container port 3000)
- Backend (Node.js) connects to RDS MySQL database on port 3306 to read/write data

---

## 2. What We Built Today — Quick Summary

| Component | Technology | Container Port | Host Port |
|-----------|-----------|----------------|-----------|
| Frontend  | React + Apache HTTPD | 80 | 80 |
| Backend   | Node.js + Express | 3000 | 84 |
| Database  | AWS RDS MySQL | 3306 | N/A (managed) |

---

## 3. Docker Concepts Covered (Enriched)

---

### 3.1 ⚠️ Only the LAST CMD Runs in a Dockerfile

**Problem from class:**
```dockerfile
CMD ["python3", "app.py"]
CMD ["python3", "--version"]   # ← This overrides the one above!
```

**What happened:**
- Container ran `python3 --version` (which prints version and exits instantly)
- That's why container showed `Exited (0)` immediately
- `Exited (0)` means the process finished successfully but there was nothing more to do

**✅ Fix:**
```dockerfile
# Keep ONLY one CMD — the one that runs your app
CMD ["python3", "app.py"]
```

**💡 Concept:** In a Dockerfile, `CMD` defines the default command to run when the container starts. Docker only respects the **last** `CMD` instruction. All previous `CMD` lines are ignored. Think of it like a "start button" — you can only have one.

---

### 3.2 🔄 Rebuilding an Image with the Same Name

**What happens when you run `docker build -t image1 .` twice?**

```
BEFORE rebuild:
image1:latest  → IMAGE ID: abc123  ✅

AFTER rebuild:
image1:latest  → IMAGE ID: xyz789  ✅ (new image, tag moved here)
<none>:<none>  → IMAGE ID: abc123  ⚠️  (old image, now "dangling")
```

**Key points:**
- The **tag** (`image1:latest`) is just a pointer/label — it moves to the new image
- The **old image** still exists but has no tag — it's called a **dangling image**
- Running containers using the old image are **NOT** automatically updated
- You must stop → remove → recreate the container to use the new image

**✅ Best Practice — Use version tags:**
```bash
docker build -t myapp:v1 .
docker build -t myapp:v2 .
docker build -t myapp:v3 .
```
This way you can always roll back to a previous version if something breaks.

---

### 3.3 🛑 Container Lifecycle — Stop vs Kill

```
Container States:
  Created ──► Running ──► Stopped/Exited ──► Removed
                │
                └──► Paused (docker pause)
```

| Command | Signal Sent | Effect |
|---------|-------------|--------|
| `docker stop <id>` | SIGTERM (graceful) | App gets time to save data & shut down cleanly |
| `docker kill <id>` | SIGKILL (force) | Immediate termination — like pulling the power plug |
| `docker rm <id>` | — | Removes stopped container permanently |
| `docker rm -f <id>` | SIGKILL + remove | Force stop and remove in one command |

**✅ Always prefer `docker stop` over `docker kill`** — gives your app time to close DB connections, finish requests, flush logs, etc.

---

### 3.4 🧹 Docker System Cleanup — `docker system prune`

Over time Docker accumulates junk:
- Stopped containers
- Dangling images (`<none>:<none>`)
- Unused networks
- Build cache

```bash
# Safe cleanup (stops nothing running)
docker system prune

# Same but no confirmation prompt
docker system prune -f

# Nuclear option — removes ALL unused images (even tagged ones)
docker system prune -a

# Clean only dangling images
docker image prune

# Clean only stopped containers
docker container prune
```

**⚠️ Warning:** Deleted data cannot be recovered. Always check `docker ps -a` and `docker images` before running prune.

---

### 3.5 🗃️ Volumes — Persisting Data

**Problem without volumes:**
- Data inside a container is lost when the container is removed
- Every new container starts fresh

**Solution — Docker Volumes:**
```bash
# Create a named volume
docker volume create mydata

# Run container with volume mounted
docker run -dt -p 3306:3306 \
  -v mydata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=pass \
  mysql:8

# Now even if you remove & recreate the container, data is preserved
```

**For our project:** We used AWS RDS (managed database) so we don't need volumes for MySQL. RDS handles persistence, backups, and availability automatically.

---

### 3.6 🌐 Port Mapping Explained

```bash
docker run -p 84:3000 backend
#              │    └──── Container internal port (where app listens)
#              └───────── Host/EC2 port (what users connect to)
```

Think of it like a hotel room:
- `3000` = Room number inside the hotel (container)
- `84` = Building entrance that maps to that room (host)
- Visitors knock on `84` → they're forwarded to room `3000`

---

### 3.7 📦 Multi-Stage Docker Build (Used in Frontend)

The frontend Dockerfile uses a clever technique called **multi-stage build**:

```dockerfile
# Stage 1: BUILD stage (Node.js)
FROM node:18 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install          # install 500MB of node_modules
COPY . .
RUN npm run build        # produces a small /app/build folder (HTML/CSS/JS)

# Stage 2: SERVE stage (Apache HTTPD)
FROM httpd:2.4           # tiny ~150MB image
COPY --from=build /app/build /usr/local/apache2/htdocs/
# node_modules are NOT copied — they're discarded!
```

**Why this matters:**
| Approach | Final Image Size |
|----------|----------------|
| Single stage (with node_modules) | ~1.2 GB |
| Multi-stage build | ~118 MB |

The final image is **10x smaller** because we only keep the compiled static files, not the build tools.

---

## 4. Security Groups Setup (EC2 + RDS)

Security Groups are AWS firewalls. You must configure them correctly or connections will be blocked.

---

### 4.1 EC2 Security Group Rules

Go to: **AWS Console → EC2 → Security Groups → Select your EC2 SG → Inbound Rules → Edit**

| Rule # | Type | Protocol | Port | Source | Purpose |
|--------|------|----------|------|--------|---------|
| 1 | SSH | TCP | 22 | Your IP only (`x.x.x.x/32`) | SSH into EC2 |
| 2 | HTTP | TCP | 80 | `0.0.0.0/0` | Frontend access |
| 3 | Custom TCP | TCP | 84 | `0.0.0.0/0` | Backend API access |
| 4 | Custom TCP | TCP | 3000 | `0.0.0.0/0` | (Optional) Direct Node access |

**⚠️ Security tip:** For SSH, never use `0.0.0.0/0`. Always restrict to your own IP address.

---

### 4.2 RDS Security Group Rules

Go to: **AWS Console → RDS → database-1 → Connectivity → VPC Security Groups → Edit Inbound Rules**

| Rule # | Type | Protocol | Port | Source | Purpose |
|--------|------|----------|------|--------|---------|
| 1 | MySQL/Aurora | TCP | 3306 | EC2 Security Group ID | Allow EC2 to reach RDS |

**How to set Source as EC2 SG:**
1. Click the Source dropdown
2. Select "Custom"
3. Start typing `sg-` and select your EC2 security group
4. This means: "Only traffic coming FROM EC2 is allowed on port 3306"

**Why this is better than using IP:** EC2 IPs can change (especially on restart). Using the SG ID means the rule stays valid forever.

---

### 4.3 Minimum Access Summary (Principle of Least Privilege)

```
Internet ──► Port 80 ──► EC2 (Frontend)
Internet ──► Port 84 ──► EC2 (Backend)
EC2 SG   ──► Port 3306 ──► RDS (Database)

❌ Internet should NEVER reach RDS directly
❌ SSH should only be open to your own IP
```

---

## 5. RDS MySQL Setup

**Your RDS Configuration (from class):**
```
DB Identifier:  database-1
Engine:         MySQL Community
Instance Class: db.t4g.micro (cheapest option)
Master Username: admin
Endpoint:       database-1.cuzycam08u1d.us-east-1.rds.amazonaws.com
Port:           3306
Publicly Accessible: Yes (needed for EC2 to connect)
```

**Step-by-step: Connect and load data from EC2:**

```bash
# Step 1: Install MySQL client on EC2
yum install mariadb105 -y

# Step 2: Test connection
mysql -h database-1.cuzycam08u1d.us-east-1.rds.amazonaws.com \
      -u admin -pmulticloud

# Step 3: Load SQL dump (creates tables + inserts data)
mysql -h database-1.cuzycam08u1d.us-east-1.rds.amazonaws.com \
      -u admin -pmulticloud < test.sql

# Error you saw: ERROR 1062 (23000): Duplicate entry '1' for key 'books.PRIMARY'
# Reason: You ran the SQL file twice — second run tried to insert same data again
# Fix: The data was already loaded the first time — this error is safe to ignore
```

---

## 6. Backend Container (Node.js)

### 6.1 The `.env` file

```bash
# /backend/.env
DB_HOST=database-1.cuzycam08u1d.us-east-1.rds.amazonaws.com
DB_USERNAME=admin
DB_PASSWORD="multicloud"
PORT=3306
```

**⚠️ NEVER commit `.env` to Git.** Add it to `.gitignore`:
```bash
echo ".env" >> .gitignore
```

### 6.2 Build and Run Backend

```bash
cd /home/ec2-user/2nd10WeeksofCloudOps-main/backend

# Build image
docker build -t backend .

# Run container (map EC2 port 84 → container port 3000)
docker run -dt -p 84:3000 backend

# Verify it's running
docker ps
curl http://localhost:84
```

### 6.3 Backend Dockerfile Explained

```dockerfile
FROM node:18                          # Base image with Node.js 18 pre-installed
WORKDIR /app                          # All commands run inside /app
COPY package.json package-lock.json . # Copy dependency list first (layer caching!)
RUN npm install                       # Install dependencies
RUN npm install -g pm2                # Install PM2 (process manager for Node.js)
COPY . .                              # Copy rest of app code
CMD ["pm2-runtime", "index.js"]       # Start app with PM2 (keeps it alive)
```

**Why copy package.json BEFORE the rest of the code?**
Docker builds layers. If you copy code first, every code change rebuilds `npm install` (slow!). By copying `package.json` first, `npm install` is only re-run when dependencies change — not on every code edit.

---

## 7. Frontend Container (React + Apache)

### 7.1 config.js — The Critical File

```javascript
// client/src/pages/config.js
// This tells React where the backend API is

// Option 1: localhost (works only when browser and server are same machine)
const API_BASE_URL = "http://localhost:84"

// Option 2: EC2 Public IP (use this for actual deployment)
const API_BASE_URL = "http://98.92.112.243:84"

// Option 3: Environment variable (best practice for production)
const API_BASE_URL = process.env.REACT_APP_API_BASE_URL || "http://localhost:84"

export default API_BASE_URL;
```

**⚠️ Important:** After changing `config.js`, you must **rebuild** the frontend image. React bakes the config into the compiled files at build time.

### 7.2 Build and Run Frontend

```bash
cd /home/ec2-user/2nd10WeeksofCloudOps-main/client

# Step 1: Edit config.js with your actual EC2 public IP
vi src/pages/config.js

# Step 2: Rebuild frontend image
docker build -t frontend .

# Step 3: Run container (map EC2 port 80 → container port 80)
docker run -dt -p 80:80 frontend

# Step 4: Open browser
# http://<EC2-PUBLIC-IP>
```

---

## 8. Push Images to Docker Hub

Docker Hub is the world's largest public container registry — like GitHub but for Docker images.

### 8.1 Create Docker Hub Account

1. Go to https://hub.docker.com
2. Sign up (free account allows unlimited public repos)
3. Create a repository — e.g., `yourusername/backend` and `yourusername/frontend`

### 8.2 Login from EC2

```bash
docker login
# Enter your Docker Hub username and password when prompted
# You'll see: Login Succeeded
```

### 8.3 Tag Your Images

Docker Hub requires images to be named: `username/repository:tag`

```bash
# Tag backend image
docker tag backend yourusername/multicloud-backend:v1
docker tag backend yourusername/multicloud-backend:latest

# Tag frontend image
docker tag frontend yourusername/multicloud-frontend:v1
docker tag frontend yourusername/multicloud-frontend:latest
```

### 8.4 Push to Docker Hub

```bash
# Push backend
docker push yourusername/multicloud-backend:v1
docker push yourusername/multicloud-backend:latest

# Push frontend
docker push yourusername/multicloud-frontend:v1
docker push yourusername/multicloud-frontend:latest
```

### 8.5 Pull from Docker Hub (on any machine)

```bash
# Anyone in the world can now run your app with:
docker pull yourusername/multicloud-backend:latest
docker run -dt -p 84:3000 yourusername/multicloud-backend:latest
```

---

## 9. Deploy on AWS ECS (Full Steps)

ECS = Elastic Container Service. AWS manages running your containers — no need to manually SSH into EC2 and run `docker run`.

### 9.1 Concept: ECS Components

```
ECS
├── Cluster        → Group of servers (or serverless) that run containers
│   └── Service    → Keeps N copies of your task always running
│       └── Task   → One running instance of your containers
│           └── Container → Your actual Docker container
└── Task Definition → Blueprint: which image, how much CPU/RAM, env vars, ports
```

Think of it like:
- **Task Definition** = Recipe card (what to cook)
- **Task** = The actual cooked dish
- **Service** = The chef who keeps making new dishes if one gets eaten
- **Cluster** = The kitchen

### 9.2 Step-by-Step ECS Deployment

#### Step 1: Push Images to ECR (Amazon Elastic Container Registry)

ECR is AWS's private Docker registry — more secure than Docker Hub for production.

```bash
# On EC2: Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS \
  --password-stdin 515800584282.dkr.ecr.us-east-1.amazonaws.com

# Create ECR repositories (do this in AWS Console OR CLI)
aws ecr create-repository --repository-name multicloud-backend --region us-east-1
aws ecr create-repository --repository-name multicloud-frontend --region us-east-1

# Tag images for ECR
docker tag backend:latest \
  515800584282.dkr.ecr.us-east-1.amazonaws.com/multicloud-backend:latest

docker tag frontend:latest \
  515800584282.dkr.ecr.us-east-1.amazonaws.com/multicloud-frontend:latest

# Push to ECR
docker push 515800584282.dkr.ecr.us-east-1.amazonaws.com/multicloud-backend:latest
docker push 515800584282.dkr.ecr.us-east-1.amazonaws.com/multicloud-frontend:latest
```

#### Step 2: Create IAM Role for ECS

1. Go to **IAM → Roles → Create Role**
2. Select: **AWS Service → Elastic Container Service → ECS Task**
3. Attach these policies:
   - `AmazonECSTaskExecutionRolePolicy` (allows ECS to pull images & write logs)
   - `AmazonEC2ContainerRegistryReadOnly` (allows pulling from ECR)
4. Name it: `ecsTaskExecutionRole`

#### Step 3: Create ECS Cluster

1. Go to **AWS Console → ECS → Clusters → Create Cluster**
2. Cluster name: `multicloud-cluster`
3. Infrastructure: **AWS Fargate (Serverless)** ← No EC2 to manage!
4. Click **Create**

#### Step 4: Create Task Definition for Backend

1. Go to **ECS → Task Definitions → Create new task definition**
2. Fill in:
   ```
   Task definition family: multicloud-backend-task
   Launch type: Fargate
   Operating system: Linux/X86_64
   CPU: 0.25 vCPU
   Memory: 0.5 GB
   Task role: ecsTaskExecutionRole
   ```
3. Add Container:
   ```
   Container name: backend
   Image URI: 515800584282.dkr.ecr.us-east-1.amazonaws.com/multicloud-backend:latest
   Port mappings: 3000 (TCP)
   ```
4. Add Environment Variables:
   ```
   DB_HOST     = database-1.cuzycam08u1d.us-east-1.rds.amazonaws.com
   DB_USERNAME = admin
   DB_PASSWORD = multicloud
   PORT        = 3306
   ```
5. Click **Create**

#### Step 5: Create Task Definition for Frontend

Repeat Step 4 with:
```
Task definition family: multicloud-frontend-task
Container name: frontend
Image URI: 515800584282.dkr.ecr.us-east-1.amazonaws.com/multicloud-frontend:latest
Port mappings: 80 (TCP)
```

#### Step 6: Create ECS Service for Backend

1. Go to **ECS → Clusters → multicloud-cluster → Services → Create**
2. Fill in:
   ```
   Launch type: Fargate
   Task definition: multicloud-backend-task (Latest)
   Service name: backend-service
   Desired tasks: 1
   ```
3. Networking:
   ```
   VPC: your VPC
   Subnets: select public subnets
   Security group: Create new
     → Inbound: Port 3000 from Anywhere
   Public IP: ENABLED
   ```
4. Click **Create Service**

#### Step 7: Create ECS Service for Frontend

Repeat Step 6 with:
```
Task definition: multicloud-frontend-task
Service name: frontend-service
Security group Inbound: Port 80 from Anywhere
```

#### Step 8: Get the Public IP of Running Tasks

1. Go to **ECS → Clusters → multicloud-cluster → Tasks**
2. Click on a running task
3. Under **Configuration → Public IP** — copy the IP
4. Open in browser: `http://<TASK-PUBLIC-IP>`

#### Step 9: Add Load Balancer (Optional but Recommended)

A Load Balancer gives you a stable URL that doesn't change even when tasks restart.

1. **Create Application Load Balancer (ALB):**
   - Go to **EC2 → Load Balancers → Create**
   - Type: Application Load Balancer
   - Internet-facing, Port 80
   - Target type: IP (for Fargate)

2. **Attach to ECS Service:**
   - When creating the ECS Service, under **Load Balancing** select your ALB
   - This automatically registers/deregisters task IPs

---

## 10. ECS with ECR (Amazon Container Registry)

### Why ECR over Docker Hub for AWS?

| Feature | Docker Hub (Free) | AWS ECR |
|---------|-------------------|---------|
| Privacy | Public by default | Private by default |
| Auth | Docker login | AWS IAM (more secure) |
| Speed | Pull from internet | Pull within AWS (faster + cheaper) |
| Integration | Manual | Native with ECS, CodePipeline, etc. |
| Cost | Free (limited) | $0.10/GB storage |

### IAM Role Setup (Full Steps)

**Create role for ECR access:**

```bash
# AWS CLI: Create ECR repository
aws ecr create-repository \
  --repository-name multicloud-backend \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true

# Get login token (valid 12 hours)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS \
  --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

**In AWS Console:**
1. **IAM → Roles → Create Role**
2. Trusted entity: `Elastic Container Service Task`
3. Add permission: `AmazonEC2ContainerRegistryFullAccess`
4. Name: `ECR-ECS-Role`
5. **Attach this role to your ECS Task Definition**

---

## 11. Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Exited (0)` immediately | Last CMD was `python3 --version` | Remove duplicate CMD; keep only the app CMD |
| `ERROR 1062: Duplicate entry` | SQL file run twice | Safe to ignore — data was already loaded |
| `bash: git: command not found` | Git not installed | `yum install git -y` |
| Container not accessible via browser | Port not opened in EC2 security group | Add inbound rule for that port |
| Can't connect EC2 to RDS | RDS security group blocks EC2 | Add inbound rule: port 3306, source = EC2 security group |
| `docker build` canceled midway | Network timeout pulling base image | Re-run `docker build` — cached layers speed it up |
| Frontend shows API error | config.js has wrong IP | Update to EC2 public IP, rebuild frontend image |
| `<none>:<none>` images accumulating | Rebuilding with same tag | Run `docker image prune` or `docker system prune` |

---

## 12. Glossary for Freshers

| Term | Simple Explanation |
|------|-------------------|
| **Docker Image** | A frozen snapshot/template of your app. Like a ZIP file. |
| **Docker Container** | A running instance of an image. Like unzipping and running the app. |
| **Dockerfile** | A recipe to build a Docker image — step by step instructions. |
| **Docker Hub** | Public store for Docker images. Like App Store for containers. |
| **ECR** | AWS's private Docker image store. |
| **ECS** | AWS service that runs and manages your containers automatically. |
| **Fargate** | ECS mode where AWS manages the servers — you just provide containers. |
| **Task Definition** | Configuration blueprint for running a container in ECS. |
| **Service** | Ensures desired number of tasks are always running. Restarts if one dies. |
| **Security Group** | AWS virtual firewall controlling what traffic can enter/leave. |
| **RDS** | AWS managed database service — AWS handles backups, patching, HA. |
| **Port Mapping** | `-p 84:3000` means EC2 port 84 maps to container's internal port 3000. |
| **Dangling Image** | Old image with no tag (`<none>:<none>`). Safe to delete. |
| **SIGTERM** | Polite shutdown signal (`docker stop`) — gives app time to clean up. |
| **SIGKILL** | Forced shutdown signal (`docker kill`) — immediate termination. |
| **Multi-stage Build** | Build in one image, copy only output to final small image. Saves size. |
| **Volume** | Persistent storage mounted into a container. Data survives container restarts. |
| **.env file** | File with secret config values. NEVER commit to Git. |
| **pm2** | Node.js process manager — restarts app if it crashes. |

---

## Quick Command Reference

```bash
# Docker basics
docker build -t name:tag .          # Build image
docker run -dt -p HOST:CONTAINER img # Run detached with port mapping
docker ps                           # List running containers
docker ps -a                        # List all containers (including stopped)
docker images                       # List images
docker logs <id>                    # View container logs
docker exec -it <id> bash           # SSH into running container
docker stop <id>                    # Graceful stop
docker rm <id>                      # Remove container
docker rmi <id>                     # Remove image
docker system prune -f              # Clean all unused resources

# Docker Hub
docker login                        # Login
docker tag img user/repo:tag        # Tag image
docker push user/repo:tag           # Push to Docker Hub
docker pull user/repo:tag           # Pull from Docker Hub

# ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ACCOUNT.dkr.ecr.REGION.amazonaws.com
docker tag img ACCOUNT.dkr.ecr.REGION.amazonaws.com/repo:tag
docker push ACCOUNT.dkr.ecr.REGION.amazonaws.com/repo:tag
```

---

*Notes compiled from Docker Day 4 — MultiCloud DevOps Class by Veera Sir, NareshIT*
*Date: April 23, 2026*
