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



--------------------------

My Manual NOtes

root@ip-172-31-23-207 ec2-user]# ls -lrt
total 12
-rw-r--r--. 1 root root 475 Apr 23 02:19 Dockerfile
-rw-r--r--. 1 root root   6 Apr 23 02:19 requirments.txt
-rw-r--r--. 1 root root 259 Apr 23 02:20 app.py
[root@ip-172-31-23-207 ec2-user]# vi app.py 
[root@ip-172-31-23-207 ec2-user]# vi Dockerfile 
[root@ip-172-31-23-207 ec2-user]# docker build -t image1 .
[+] Building 5.3s (10/10) FINISHED                                                                                                                                                                  docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                          0.0s
 => => transferring dockerfile: 573B                                                                                                                                                                          0.0s
 => [internal] load metadata for docker.io/library/ubuntu:22.04                                                                                                                                               0.1s
 => [internal] load .dockerignore                                                                                                                                                                             0.0s
 => => transferring context: 2B                                                                                                                                                                               0.0s
 => [1/5] FROM docker.io/library/ubuntu:22.04@sha256:962f6cadeae0ea6284001009daa4cc9a8c37e75d1f5191cf0eb83fe565b63dd7                                                                                         0.0s
 => [internal] load build context                                                                                                                                                                             0.0s
 => => transferring context: 1.25kB                                                                                                                                                                           0.0s
 => CACHED [2/5] RUN apt-get update &&     apt-get install -y python3-pip &&     apt-get clean                                                                                                                0.0s
 => CACHED [3/5] WORKDIR /app                                                                                                                                                                                 0.0s
 => [4/5] COPY . /app                                                                                                                                                                                         0.0s
 => [5/5] RUN pip3 install -r requirments.txt                                                                                                                                                                 2.2s
 => exporting to image                                                                                                                                                                                        2.9s 
 => => exporting layers                                                                                                                                                                                       2.9s 
 => => writing image sha256:d63fc68f9d0db3f5ad1876fe69e129848c38d837c853b6e10dc8afb1c435091a                                                                                                                  0.0s 
 => => naming to docker.io/library/image1                                                                                                                                                                     0.0s 
[root@ip-172-31-23-207 ec2-user]# docker images                                                                                                                                                                    
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE                                                                                                                                                        
image1       latest    d63fc68f9d0d   11 seconds ago   507MB
[root@ip-172-31-23-207 ec2-user]# cat Dockerfile 
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
RUN pip3 install -r requirments.txt

# Run the app when container starts
CMD ["python3", "app.py"]
CMD ["python3", "--version"]
[root@ip-172-31-23-207 ec2-user]# cat requirments.txt 
flask
[root@ip-172-31-23-207 ec2-user]# cat app.py 
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return "Hello, Flask is running on port 5000!"

@app.route('/health')
def health():
    return {"status": "UP"}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
[root@ip-172-31-23-207 ec2-user]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED              SIZE
image1       latest    d63fc68f9d0d   About a minute ago   507MB
[root@ip-172-31-23-207 ec2-user]# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
[root@ip-172-31-23-207 ec2-user]# docker run -dt image1
be31fc0fb461b515c1f5d594c72f00077503da3c953d702fbe09985623f0afac
[root@ip-172-31-23-207 ec2-user]# docker ps -a
CONTAINER ID   IMAGE     COMMAND               CREATED         STATUS                     PORTS     NAMES
be31fc0fb461   image1    "python3 --version"   4 seconds ago   Exited (0) 4 seconds ago             funny_bassi
[root@ip-172-31-23-207 ec2-user]# 

-----

Docker only executes the last CMD instruction.

So your container is actually running:

python3 --version

That command finishes instantly → container exits → that’s why you see:

Exited (0)
----

------------------------------------


while creating imgae if i give same image name again is it will override

docker build -t image1 .
Docker creates a new image (new IMAGE ID)
The tag image1:latest is moved to point to the new image
The old image still exists, but becomes dangling (no tag)
Before rebuild:
image1:latest  → IMAGE ID abc123
After rebuild:
image1:latest  → IMAGE ID xyz789   ✅ (new image)
<none>:<none>  → IMAGE ID abc123   ❗ (old dangling image)

Important
If a container is already running using the old image:
It will NOT automatically use the new image
You must stop and recreate container
Instead of always using latest, use version tags:
docker build -t image1:v1 .
docker build -t image1:v2 .
This avoids confusion and helps in rollbacks.
---------------------------------


before kill the container do we need to follow any pre steps veera?


Good question — and nice thinking ahead 👍

🟢 Short answer:

👉 Usually NO special pre-steps needed
But in some cases, YES — you should be careful

✅ Safe to directly stop/kill (your case)

For your Flask app:

docker stop <container_id>
docker rm <container_id>

✔️ No issue — it’s stateless (no important data inside)

⚠️ When you SHOULD take precautions
1. 🗄️ If container has important data
Example: DB, logs, uploaded files
Data inside container will be lost

👉 Do this before stopping:

Use volumes (-v)
Or take backup
2. 🔄 If handling live traffic (production)
Stopping abruptly = downtime

👉 Better approach:

Deploy new container first
Then stop old one (zero downtime strategy)
3. 🧾 Check logs (optional but useful)

Before killing:

docker logs <container_id>

Helpful for debugging

4. 🔍 Check running processes
docker exec -it <container_id> bash

Only if you want to inspect before stopping

❌ Avoid using
docker kill <container_id>
Force stops immediately (like power cut ⚡)
Use only if container is stuck
✅ Best practice (simple rule)
Dev / Testing → just stop & remove 👍
Production → graceful stop + backup + rollout
💡 Pro tip (very useful for you)

Use:

docker stop

✔️ Sends SIGTERM → allows app to shut down properly

-------------------------
docker system prune

🧹 docker system prune — in short
👉 It cleans up unused Docker resources to free space.

✅ What it removes
By default:


Stopped containers


Unused networks


Dangling images (<none>)


Build cache



⚠️ What it does NOT remove


Running containers ✅


Tagged images (like image1:latest) ❌



🔥 Command
docker system prune
It will ask:
Are you sure you want to continue? [y/N]

🚀 Force delete (no prompt)
docker system prune -f

💣 Remove EVERYTHING unused (be careful)
docker system prune -a
👉 This will also remove:


All unused images (even with tags)



📌 Example use case (your case)
After rebuilding image multiple times:


Old images → <none>


Containers → exited


Run:
docker system prune
✔️ Cleans all junk
✔️ Frees disk space

⚠️ Warning
Once deleted → ❌ cannot recover

💡 Best practice


Use docker system prune → safe cleanup


Use -a → only when you’re sure


---------------------------------------

Now launch rds mysql standalone with minimum options

database-1 


Modify
Actions
Summary
DB identifier
database-1
CPU
 3.78%
Status
Available
Class
db.t4g.micro
Role
Instance
Current activity
 0 Connections
Engine
MySQL Community
Region & AZ
us-east-1f
Recommendations

Connectivity & security

Monitoring

Logs & events

Configuration

Zero-ETL integrations

Maintenance & backups

Data migrations

Tags

Recommendations
Connect using
Info
Code snippets
Use when connecting through SDK, APIs, or third-party tools including agents.
CloudShell
Use for a quick access to AWS CLI that launches directly from the AWS Management Console.
Endpoints
Use when connecting through any IDE interface.
Database name
mysql
Master username
admin
Internet access gateway
Disabled
Port
3306
Endpoint type

Instance endpoint
Additional configurations
Connectivity & security
Endpoint & port
Endpoint
database-1.cuzycam08u1d.us-east-1.rds.amazonaws.com
Port
3306
Networking
Availability Zone
us-east-1f
VPC
vpc-03b500a3ad64e4ae2
Subnet group
default-vpc-03b500a3ad64e4ae2
Subnets
subnet-0388b1d288e2a53d1
subnet-09a6bef89237ea8ed
subnet-0f50071b0dbc3b035
subnet-0eddf27736b252417
subnet-0fe612789a41fcf8b
subnet-049a2623d40b63a2b
Network type
IPv4
Security
VPC security groups
default (sg-01c682b85a83ce6a8)
Active
Publicly accessible
Yes
Certificate authority
Info
rds-ca-rsa2048-g1
Certificate authority date
May 26, 2061, 05:04 (UTC+05:30)
DB instance certificate expiration date
April 23, 2027, 08:07 (UTC+05:30)


Need to add the steps for security groups needed for both RDS and EC2 with required things for minimum access to any one 

------------------------------

[root@ip-172-31-23-207 ec2-user]# git clone https://github.com/CloudTechDevOps/2nd10WeeksofCloudOps-main.git
bash: git: command not found
[root@ip-172-31-23-207 ec2-user]# yum install git -y
Last metadata expiration check: 0:36:41 ago on Thu Apr 23 02:13:15 2026.
Dependencies resolved.
===================================================================================================================================================================================================================
 Package                                             Architecture                              Version                                                        Repository                                      Size
===================================================================================================================================================================================================================
Installing:
 git                                                 x86_64                                    2.50.1-1.amzn2023.0.1                                          amazonlinux                                     53 k
Installing dependencies:
 git-core                                            x86_64                                    2.50.1-1.amzn2023.0.1                                          amazonlinux                                    4.9 M
 git-core-doc                                        noarch                                    2.50.1-1.amzn2023.0.1                                          amazonlinux                                    2.8 M
 perl-Error                                          noarch                                    1:0.17030-2.amzn2023.0.1                                       amazonlinux                                     42 k
 perl-File-Find                                      noarch                                    1.37-477.amzn2023.0.7                                          amazonlinux                                     25 k
 perl-Git                                            noarch                                    2.50.1-1.amzn2023.0.1                                          amazonlinux                                     41 k
 perl-TermReadKey                                    x86_64                                    2.38-9.amzn2023.0.2                                            amazonlinux                                     36 k
 perl-lib                                            x86_64                                    0.65-477.amzn2023.0.7                                          amazonlinux                                     15 k

Transaction Summary
===================================================================================================================================================================================================================
Install  8 Packages

Total download size: 7.9 M
Installed size: 41 M
Downloading Packages:
(1/8): git-2.50.1-1.amzn2023.0.1.x86_64.rpm                                                                                                                                        1.4 MB/s |  53 kB     00:00    
(2/8): perl-Error-0.17030-2.amzn2023.0.1.noarch.rpm                                                                                                                                1.5 MB/s |  42 kB     00:00    
(3/8): git-core-doc-2.50.1-1.amzn2023.0.1.noarch.rpm                                                                                                                                32 MB/s | 2.8 MB     00:00    
(4/8): perl-File-Find-1.37-477.amzn2023.0.7.noarch.rpm                                                                                                                             922 kB/s |  25 kB     00:00    
(5/8): git-core-2.50.1-1.amzn2023.0.1.x86_64.rpm                                                                                                                                    36 MB/s | 4.9 MB     00:00    
(6/8): perl-Git-2.50.1-1.amzn2023.0.1.noarch.rpm                                                                                                                                   845 kB/s |  41 kB     00:00    
(7/8): perl-TermReadKey-2.38-9.amzn2023.0.2.x86_64.rpm                                                                                                                             778 kB/s |  36 kB     00:00    
(8/8): perl-lib-0.65-477.amzn2023.0.7.x86_64.rpm                                                                                                                                   594 kB/s |  15 kB     00:00    
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                               39 MB/s | 7.9 MB     00:00     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                           1/1 
  Installing       : git-core-2.50.1-1.amzn2023.0.1.x86_64                                                                                                                                                     1/8 
  Installing       : git-core-doc-2.50.1-1.amzn2023.0.1.noarch                                                                                                                                                 2/8 
  Installing       : perl-lib-0.65-477.amzn2023.0.7.x86_64                                                                                                                                                     3/8 
  Installing       : perl-TermReadKey-2.38-9.amzn2023.0.2.x86_64                                                                                                                                               4/8 
  Installing       : perl-File-Find-1.37-477.amzn2023.0.7.noarch                                                                                                                                               5/8 
  Installing       : perl-Error-1:0.17030-2.amzn2023.0.1.noarch                                                                                                                                                6/8 
  Installing       : perl-Git-2.50.1-1.amzn2023.0.1.noarch                                                                                                                                                     7/8 
  Installing       : git-2.50.1-1.amzn2023.0.1.x86_64                                                                                                                                                          8/8 
  Running scriptlet: git-2.50.1-1.amzn2023.0.1.x86_64                                                                                                                                                          8/8 
  Verifying        : git-2.50.1-1.amzn2023.0.1.x86_64                                                                                                                                                          1/8 
  Verifying        : git-core-2.50.1-1.amzn2023.0.1.x86_64                                                                                                                                                     2/8 
  Verifying        : git-core-doc-2.50.1-1.amzn2023.0.1.noarch                                                                                                                                                 3/8 
  Verifying        : perl-Error-1:0.17030-2.amzn2023.0.1.noarch                                                                                                                                                4/8 
  Verifying        : perl-File-Find-1.37-477.amzn2023.0.7.noarch                                                                                                                                               5/8 
  Verifying        : perl-Git-2.50.1-1.amzn2023.0.1.noarch                                                                                                                                                     6/8 
  Verifying        : perl-TermReadKey-2.38-9.amzn2023.0.2.x86_64                                                                                                                                               7/8 
  Verifying        : perl-lib-0.65-477.amzn2023.0.7.x86_64                                                                                                                                                     8/8 

Installed:
  git-2.50.1-1.amzn2023.0.1.x86_64                      git-core-2.50.1-1.amzn2023.0.1.x86_64           git-core-doc-2.50.1-1.amzn2023.0.1.noarch             perl-Error-1:0.17030-2.amzn2023.0.1.noarch          
  perl-File-Find-1.37-477.amzn2023.0.7.noarch           perl-Git-2.50.1-1.amzn2023.0.1.noarch           perl-TermReadKey-2.38-9.amzn2023.0.2.x86_64           perl-lib-0.65-477.amzn2023.0.7.x86_64               

Complete!
[root@ip-172-31-23-207 ec2-user]# git clone https://github.com/CloudTechDevOps/2nd10WeeksofCloudOps-main.git
Cloning into '2nd10WeeksofCloudOps-main'...
remote: Enumerating objects: 1036, done.
remote: Counting objects: 100% (1036/1036), done.
remote: Compressing objects: 100% (352/352), done.
remote: Total 1036 (delta 685), reused 1030 (delta 680), pack-reused 0 (from 0)
Receiving objects: 100% (1036/1036), 3.79 MiB | 45.60 MiB/s, done.
Resolving deltas: 100% (685/685), done.
[root@ip-172-31-23-207 ec2-user]# ls -lrt
total 28
-rw-r--r--.  1 root root     6 Apr 23 02:19 requirments.txt
-rw-r--r--.  1 root root   259 Apr 23 02:20 app.py
-rw-r--r--.  1 root root   475 Apr 23 02:23 Dockerfile
drwxr-xr-x. 11 root root 16384 Apr 23 02:50 2nd10WeeksofCloudOps-main
[root@ip-172-31-23-207 ec2-user]# cd 2nd10WeeksofCloudOps-main/
[root@ip-172-31-23-207 2nd10WeeksofCloudOps-main]# ls -lrt
total 3764
-rw-r--r--. 1 root root       1 Apr 23 02:50 README.md
drwxr-xr-x. 2 root root      61 Apr 23 02:50 Jenkins-Pipeline-Code
drwxr-xr-x. 2 root root     113 Apr 23 02:50 backend
-rw-r--r--. 1 root root 3821788 Apr 23 02:50 architecture.gif
drwxr-xr-x. 3 root root   16384 Apr 23 02:50 terraform_main_ec2
drwxr-xr-x. 2 root root      32 Apr 23 02:50 rds
drwxr-xr-x. 2 root root      40 Apr 23 02:50 mysql
drwxr-xr-x. 2 root root     141 Apr 23 02:50 kubernetes-files
drwxr-xr-x. 2 root root      62 Apr 23 02:50 eks-terraform
-rw-r--r--. 1 root root     926 Apr 23 02:50 docker-compose.yaml
-rw-r--r--. 1 root root    2726 Apr 23 02:50 docker-compose-process-note.md
drwxr-xr-x. 4 root root     133 Apr 23 02:50 client
[root@ip-172-31-23-207 2nd10WeeksofCloudOps-main]# vi rds
[root@ip-172-31-23-207 2nd10WeeksofCloudOps-main]# cd backend/
[root@ip-172-31-23-207 backend]# ls
Dockerfile  index.js  package-lock.json  package.json  test.sql
[root@ip-172-31-23-207 backend]# ls -lrt
total 68
-rw-r--r--. 1 root root  1476 Apr 23 02:50 test.sql
-rw-r--r--. 1 root root   320 Apr 23 02:50 package.json
-rw-r--r--. 1 root root 50795 Apr 23 02:50 package-lock.json
-rw-r--r--. 1 root root  1732 Apr 23 02:50 index.js
-rw-r--r--. 1 root root   640 Apr 23 02:50 Dockerfile
[root@ip-172-31-23-207 backend]# yum install mariadb105
Last metadata expiration check: 0:38:36 ago on Thu Apr 23 02:13:15 2026.
Dependencies resolved.
===================================================================================================================================================================================================================
 Package                                                     Architecture                            Version                                                    Repository                                    Size
===================================================================================================================================================================================================================
Installing:
 mariadb105                                                  x86_64                                  3:10.5.29-1.amzn2023.0.1                                   amazonlinux                                  1.5 M
Installing dependencies:
 mariadb-connector-c                                         x86_64                                  3.3.10-1.amzn2023.0.1                                      amazonlinux                                  211 k
 mariadb-connector-c-config                                  noarch                                  3.3.10-1.amzn2023.0.1                                      amazonlinux                                  9.9 k
 mariadb105-common                                           x86_64                                  3:10.5.29-1.amzn2023.0.1                                   amazonlinux                                   28 k
 perl-Sys-Hostname                                           x86_64                                  1.23-477.amzn2023.0.7                                      amazonlinux                                   16 k

Transaction Summary
===================================================================================================================================================================================================================
Install  5 Packages

Total download size: 1.8 M
Installed size: 19 M
Is this ok [y/N]: y
Downloading Packages:
(1/5): mariadb-connector-c-3.3.10-1.amzn2023.0.1.x86_64.rpm                                                                                                                        5.0 MB/s | 211 kB     00:00    
(2/5): mariadb-connector-c-config-3.3.10-1.amzn2023.0.1.noarch.rpm                                                                                                                 227 kB/s | 9.9 kB     00:00    
(3/5): mariadb105-10.5.29-1.amzn2023.0.1.x86_64.rpm                                                                                                                                 25 MB/s | 1.5 MB     00:00    
(4/5): mariadb105-common-10.5.29-1.amzn2023.0.1.x86_64.rpm                                                                                                                         1.1 MB/s |  28 kB     00:00    
(5/5): perl-Sys-Hostname-1.23-477.amzn2023.0.7.x86_64.rpm                                                                                                                          569 kB/s |  16 kB     00:00    
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                               11 MB/s | 1.8 MB     00:00     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                           1/1 
  Installing       : mariadb-connector-c-config-3.3.10-1.amzn2023.0.1.noarch                                                                                                                                   1/5 
  Installing       : mariadb-connector-c-3.3.10-1.amzn2023.0.1.x86_64                                                                                                                                          2/5 
  Installing       : mariadb105-common-3:10.5.29-1.amzn2023.0.1.x86_64                                                                                                                                         3/5 
  Installing       : perl-Sys-Hostname-1.23-477.amzn2023.0.7.x86_64                                                                                                                                            4/5 
  Installing       : mariadb105-3:10.5.29-1.amzn2023.0.1.x86_64                                                                                                                                                5/5 
  Running scriptlet: mariadb105-3:10.5.29-1.amzn2023.0.1.x86_64                                                                                                                                                5/5 
  Verifying        : mariadb-connector-c-3.3.10-1.amzn2023.0.1.x86_64                                                                                                                                          1/5 
  Verifying        : mariadb-connector-c-config-3.3.10-1.amzn2023.0.1.noarch                                                                                                                                   2/5 
  Verifying        : mariadb105-3:10.5.29-1.amzn2023.0.1.x86_64                                                                                                                                                3/5 
  Verifying        : mariadb105-common-3:10.5.29-1.amzn2023.0.1.x86_64                                                                                                                                         4/5 
  Verifying        : perl-Sys-Hostname-1.23-477.amzn2023.0.7.x86_64                                                                                                                                            5/5 

Installed:
  mariadb-connector-c-3.3.10-1.amzn2023.0.1.x86_64    mariadb-connector-c-config-3.3.10-1.amzn2023.0.1.noarch    mariadb105-3:10.5.29-1.amzn2023.0.1.x86_64    mariadb105-common-3:10.5.29-1.amzn2023.0.1.x86_64   
  perl-Sys-Hostname-1.23-477.amzn2023.0.7.x86_64     

Complete!
[root@ip-172-31-23-207 backend]# mysql -h database-1.cuzycam08u1d.us-east-1.rds.amazonaws.com -u admin -pmulticloud < test.sql
[root@ip-172-31-23-207 backend]# mysql -h database-1.cuzycam08u1d.us-east-1.rds.amazonaws.com -u admin -pmulticloud < test.sql
--------------
INSERT INTO `books` (`id`, `title`, `desc`, `price`, `cover`) VALUES
(1, 'MultiCloud', 'this is mutlicloud with devops cource by veera sir nareshit ', 2343.2, 'https://docs.multy.dev/assets/images/multi-cloud-314609adeec670988dff0882a93fdcb0.png'),
(2, 'DevOps', 'if you understand the devops it is very easy', 2342.3, 'https://media.licdn.com/dms/image/v2/D5612AQHsA9dJVtZRdw/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1730826289071?e=2147483647&v=beta&t=JGr0gJH6RE9b1Dk7nHAYF14Kmv96Gm519UOy_rSa3xE')
--------------

ERROR 1062 (23000) at line 29: Duplicate entry '1' for key 'books.PRIMARY'
[root@ip-172-31-23-207 backend]# ls -lrt
total 68
-rw-r--r--. 1 root root  1476 Apr 23 02:50 test.sql
-rw-r--r--. 1 root root   320 Apr 23 02:50 package.json
-rw-r--r--. 1 root root 50795 Apr 23 02:50 package-lock.json
-rw-r--r--. 1 root root  1732 Apr 23 02:50 index.js
-rw-r--r--. 1 root root   640 Apr 23 02:50 Dockerfile
[root@ip-172-31-23-207 backend]# vi .env
[root@ip-172-31-23-207 backend]# cat .env
DB_HOST=database-1.cuzycam08u1d.us-east-1.rds.amazonaws.com
DB_USERNAME=admin
DB_PASSWORD="multicloud"
PORT=3306
[root@ip-172-31-23-207 backend]# 

---------------------


[root@ip-172-31-23-207 backend]# docker build -t backend .
[+] Building 47.9s (11/11) FINISHED                                                                                                                                                                 docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                          0.0s
 => => transferring dockerfile: 750B                                                                                                                                                                          0.0s
 => [internal] load metadata for docker.io/library/node:18                                                                                                                                                    0.4s
 => [internal] load .dockerignore                                                                                                                                                                             0.0s
 => => transferring context: 2B                                                                                                                                                                               0.0s
 => [1/6] FROM docker.io/library/node:18@sha256:c6ae79e38498325db67193d391e6ec1d224d96c693a8a4d943498556716d3783                                                                                             23.0s
 => => resolve docker.io/library/node:18@sha256:c6ae79e38498325db67193d391e6ec1d224d96c693a8a4d943498556716d3783                                                                                              0.0s
 => => sha256:eb29363371ee2859fad6a3c5af88d4abc6ff7d399addb13b7de3c1f11bdee6b9 2.49kB / 2.49kB                                                                                                                0.0s
 => => sha256:3e6b9d1a95114e19f12262a4e8a59ad1d1a10ca7b82108adcf0605a200294964 48.49MB / 48.49MB                                                                                                              1.1s
 => => sha256:37927ed901b1b2608b72796c6881bf645480268eca4ac9a37b9219e050bb4d84 24.02MB / 24.02MB                                                                                                              1.0s
 => => sha256:c6ae79e38498325db67193d391e6ec1d224d96c693a8a4d943498556716d3783 6.41kB / 6.41kB                                                                                                                0.0s
 => => sha256:b50082bc3670d0396b2d90e4b0e5bb10265ba5d0ee16bf40f9a505f7045ee563 6.39kB / 6.39kB                                                                                                                0.0s
 => => sha256:79b2f47ad4443652b9b5cc81a95ede249fd976310efdbee159f29638783778c0 64.40MB / 64.40MB                                                                                                              2.1s
 => => sha256:e23f099911d692f62b851cf49a1e93294288a115f5cd2d014180e4d3684d34ab 211.36MB / 211.36MB                                                                                                            6.2s
 => => extracting sha256:3e6b9d1a95114e19f12262a4e8a59ad1d1a10ca7b82108adcf0605a200294964                                                                                                                     3.3s
 => => sha256:cda7f44f2bddcc4bb7514474024b3f3705de00ddb6355a33be5ac7808e5b7125 3.32kB / 3.32kB                                                                                                                2.1s
 => => sha256:c6b30c3f16966552af10ac00521f60355b1fcfd46ac1c20b1038587e28583ce7 45.68MB / 45.68MB                                                                                                              3.0s
 => => sha256:3697be50c98b9d071df4637e1d3491d00e7b9f3a732768c876d82309b3c5a145 1.25MB / 1.25MB                                                                                                                2.2s
 => => sha256:461077a72fb7fe40d34a37d6a1958c4d16772d0dd77f572ec50a1fdc41a3754d 446B / 446B                                                                                                                    2.2s
 => => extracting sha256:37927ed901b1b2608b72796c6881bf645480268eca4ac9a37b9219e050bb4d84                                                                                                                     0.7s
 => => extracting sha256:79b2f47ad4443652b9b5cc81a95ede249fd976310efdbee159f29638783778c0                                                                                                                     3.3s
 => => extracting sha256:e23f099911d692f62b851cf49a1e93294288a115f5cd2d014180e4d3684d34ab                                                                                                                     8.7s
 => => extracting sha256:cda7f44f2bddcc4bb7514474024b3f3705de00ddb6355a33be5ac7808e5b7125                                                                                                                     0.0s
 => => extracting sha256:c6b30c3f16966552af10ac00521f60355b1fcfd46ac1c20b1038587e28583ce7                                                                                                                     2.8s
 => => extracting sha256:3697be50c98b9d071df4637e1d3491d00e7b9f3a732768c876d82309b3c5a145                                                                                                                     0.0s
 => => extracting sha256:461077a72fb7fe40d34a37d6a1958c4d16772d0dd77f572ec50a1fdc41a3754d                                                                                                                     0.0s
 => [internal] load build context                                                                                                                                                                             0.0s
 => => transferring context: 55.69kB                                                                                                                                                                          0.0s
 => [2/6] WORKDIR /app                                                                                                                                                                                        5.3s
 => [3/6] COPY package.json package-lock.json ./                                                                                                                                                              0.1s
 => [4/6] RUN npm install                                                                                                                                                                                     6.3s
 => [5/6] RUN npm install -g pm2                                                                                                                                                                             11.7s 
 => [6/6] COPY . .                                                                                                                                                                                            0.0s 
 => exporting to image                                                                                                                                                                                        1.0s 
 => => exporting layers                                                                                                                                                                                       1.0s 
 => => writing image sha256:0bf78f087d9c03b6ee23ff86cf7e381fee56515e4203a6496a5e20122b2f1c01                                                                                                                  0.0s 
 => => naming to docker.io/library/backend                                                                                                                                                                    0.0s 
[root@ip-172-31-23-207 backend]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
backend      latest    0bf78f087d9c   2 minutes ago    1.25GB
image1       latest    d63fc68f9d0d   40 minutes ago   507MB
[root@ip-172-31-23-207 backend]# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
[root@ip-172-31-23-207 backend]# ls -lrta
total 88
-rw-r--r--.  1 root root  1476 Apr 23 02:50 test.sql
-rw-r--r--.  1 root root   320 Apr 23 02:50 package.json
-rw-r--r--.  1 root root 50795 Apr 23 02:50 package-lock.json
-rw-r--r--.  1 root root  1732 Apr 23 02:50 index.js
drwxr-xr-x. 11 root root 16384 Apr 23 02:50 ..
-rw-r--r--.  1 root root   113 Apr 23 02:56 .env
-rw-r--r--.  1 root root   652 Apr 23 02:59 Dockerfile
drwxr-xr-x.  2 root root   113 Apr 23 02:59 .
[root@ip-172-31-23-207 backend]# 

_-------------------------------


[root@ip-172-31-23-207 2nd10WeeksofCloudOps-main]# docker run -dt -p 84:3000 backend
28f42ce96c6696706dcb52c9004a64a1b530f28ae3dcef6de67059772b7ecdd7
[root@ip-172-31-23-207 2nd10WeeksofCloudOps-main]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                                           NAMES
28f42ce96c66   backend   "docker-entrypoint.s…"   3 seconds ago   Up 2 seconds   80/tcp, 0.0.0.0:84->3000/tcp, :::84->3000/tcp   elated_ramanujan
[root@ip-172-31-23-207 2nd10WeeksofCloudOps-main]# 


------------------

[root@ip-172-31-23-207 pages]# ls -lrt
total 16
-rw-r--r--. 1 root root  425 Apr 23 02:50 config.js
-rw-r--r--. 1 root root 1585 Apr 23 02:50 Update.jsx
-rw-r--r--. 1 root root 1706 Apr 23 02:50 Books.jsx
-rw-r--r--. 1 root root 1484 Apr 23 02:50 Add.jsx
[root@ip-172-31-23-207 pages]# vi config.js 
[root@ip-172-31-23-207 pages]# cd ../
[root@ip-172-31-23-207 src]# cd con
bash: cd: con: No such file or directory
[root@ip-172-31-23-207 src]# cd pages/
[root@ip-172-31-23-207 pages]# cat config.js
//const API_BASE_URL = "/api";  // For reverse proxy it is mandatory so dont change
const API_BASE_URL = "http://localhost:84"

// const API_BASE_URL = "http://98.92.112.243:84"
//const API_BASE_URL = "http://backend.515800584282.realhandsonlabs.net";
// const API_BASE_URL = process.env.REACT_APP_API_BASE_URL || "http://backend";
//const API_BASE_URL = "REACT_APP_API_BASE_URL_PLACEHOLDER";
export default API_BASE_URL;

[root@ip-172-31-23-207 pages]# pwd
/home/ec2-user/2nd10WeeksofCloudOps-main/client/src/pages
[root@ip-172-31-23-207 pages]# 

--

root@ip-172-31-23-207 2nd10WeeksofCloudOps-main]# cd client/
[root@ip-172-31-23-207 client]# ls -lrt
total 1180
-rw-r--r--. 1 root root     541 Apr 23 02:50 proxy.conf
-rw-r--r--. 1 root root     886 Apr 23 02:50 package.json
-rw-r--r--. 1 root root 1188463 Apr 23 02:50 package-lock.json
-rw-r--r--. 1 root root     263 Apr 23 02:50 entrypoint.sh
-rw-r--r--. 1 root root    1228 Apr 23 02:50 Dockerfile
drwxr-xr-x. 3 root root     181 Apr 23 02:50 src
drwxr-xr-x. 2 root root     120 Apr 23 02:50 public
[root@ip-172-31-23-207 client]# docker build -t frontend .
[+] Building 52.0s (14/14) FINISHED                                                                                                                                                                 docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                          0.0s
 => => transferring dockerfile: 1.33kB                                                                                                                                                                        0.0s
 => [internal] load metadata for docker.io/library/httpd:2.4                                                                                                                                                  0.3s
 => [internal] load metadata for docker.io/library/node:18                                                                                                                                                    0.2s
 => [internal] load .dockerignore                                                                                                                                                                             0.0s
 => => transferring context: 2B                                                                                                                                                                               0.0s
 => [build 1/6] FROM docker.io/library/node:18@sha256:c6ae79e38498325db67193d391e6ec1d224d96c693a8a4d943498556716d3783                                                                                        0.0s
 => [stage-1 1/2] FROM docker.io/library/httpd:2.4@sha256:89a0b59e6d7285f00fc8df952a1579c31eb035cbdccb96690051e8bb6432cbea                                                                                    3.5s
 => => resolve docker.io/library/httpd:2.4@sha256:89a0b59e6d7285f00fc8df952a1579c31eb035cbdccb96690051e8bb6432cbea                                                                                            0.0s
 => => sha256:89a0b59e6d7285f00fc8df952a1579c31eb035cbdccb96690051e8bb6432cbea 10.14kB / 10.14kB                                                                                                              0.0s
 => => sha256:eeeb5c6dd40afcd20fe975babd9e0f75311fb4ae6a2fd473ea717a04b96d66c4 2.09kB / 2.09kB                                                                                                                0.0s
 => => sha256:d20a6ad9358ed372a1f4f7c4756b40499b2002023bb0924ef7c7c6fc963594c9 7.99kB / 7.99kB                                                                                                                0.0s
 => => sha256:3531af2bc2a9c8883754652783cf96207d53189db279c9637b7157d034de7ecd 29.78MB / 29.78MB                                                                                                              0.4s
 => => sha256:5b16784cbecd5d5f8af24ffc9a81a399c31eab252795bfb0f2e5469199f8570b 2.00MB / 2.00MB                                                                                                                0.2s
 => => sha256:d32ec5ee95cb99b3b76a359cc8e0c7ca0bbcbfa83f0e86c10237edbbeb248a67 145B / 145B                                                                                                                    0.1s
 => => sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1 32B / 32B                                                                                                                      0.1s
 => => sha256:b267b545226ba3ec284f36692de382e4dd4a80e33362e6948d9c2c45d2e120eb 13.45MB / 13.45MB                                                                                                              0.3s
 => => sha256:1935f25c7dc42c04314d348d390ebe6a7fd999c4870e66da1c453ebd3fce295b 289B / 289B                                                                                                                    0.3s
 => => extracting sha256:3531af2bc2a9c8883754652783cf96207d53189db279c9637b7157d034de7ecd                                                                                                                     1.9s
 => => extracting sha256:d32ec5ee95cb99b3b76a359cc8e0c7ca0bbcbfa83f0e86c10237edbbeb248a67                                                                                                                     0.0s
 => => extracting sha256:4f4fb700ef54461cfa02571ae0db9a0dc1e0cdb5577484a6d75e68dc38e8acc1                                                                                                                     0.0s
 => => extracting sha256:5b16784cbecd5d5f8af24ffc9a81a399c31eab252795bfb0f2e5469199f8570b                                                                                                                     0.2s
 => => extracting sha256:b267b545226ba3ec284f36692de382e4dd4a80e33362e6948d9c2c45d2e120eb                                                                                                                     0.6s
 => => extracting sha256:1935f25c7dc42c04314d348d390ebe6a7fd999c4870e66da1c453ebd3fce295b                                                                                                                     0.0s
 => [internal] load build context                                                                                                                                                                             0.1s
 => => transferring context: 1.23MB                                                                                                                                                                           0.0s
 => CACHED [build 2/6] WORKDIR /app                                                                                                                                                                           0.0s
 => [build 3/6] COPY package.json package-lock.json ./                                                                                                                                                        0.1s
 => [build 4/6] RUN npm install                                                                                                                                                                              34.1s
 => [build 5/6] COPY . .                                                                                                                                                                                      0.0s 
 => [build 6/6] RUN npm run build                                                                                                                                                                            16.4s 
 => [stage-1 2/2] COPY --from=build /app/build /usr/local/apache2/htdocs/                                                                                                                                     0.0s 
 => exporting to image                                                                                                                                                                                        0.0s 
 => => exporting layers                                                                                                                                                                                       0.0s 
 => => writing image sha256:195544bec9c0a8cb98f84d9288bc6289908091e3dc93c0803a47a5956ab0c665                                                                                                                  0.0s 
 => => naming to docker.io/library/frontend                                                                                                                                                                   0.0s 
[root@ip-172-31-23-207 client]# pwd                                                                                                                                                                                
/home/ec2-user/2nd10WeeksofCloudOps-main/client
[root@ip-172-31-23-207 client]# 


--------------

[root@ip-172-31-23-207 client]# pwd                                                                                                                                                                                
/home/ec2-user/2nd10WeeksofCloudOps-main/client
[root@ip-172-31-23-207 client]# docker run -dt -p 80:80 frontend
6b7b3fc2b749da03d8feb32eba56f5ac9572fee4f455329e1f01741781fbe329
[root@ip-172-31-23-207 client]# docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS         PORTS                                           NAMES
6b7b3fc2b749   frontend   "httpd-foreground"       3 seconds ago   Up 2 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp               gracious_kapitsa
28f42ce96c66   backend    "docker-entrypoint.s…"   8 minutes ago   Up 8 minutes   80/tcp, 0.0.0.0:84->3000/tcp, :::84->3000/tcp   elated_ramanujan
[root@ip-172-31-23-207 client]# 




----
in config.js i edited ec2 public ip
root@ip-172-31-23-207 client]# docker stop 6b7b3fc2b749
6b7b3fc2b749
[root@ip-172-31-23-207 client]# docker stop 28f42ce96c66
28f42ce96c66
[root@ip-172-31-23-207 client]# docker rm 6b7b3fc2b749
6b7b3fc2b749
[root@ip-172-31-23-207 client]# docker rm 28f42ce96c66
28f42ce96c66
[root@ip-172-31-23-207 client]# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
[root@ip-172-31-23-207 client]# docker images
REPOSITORY   TAG       IMAGE ID       CREATED             SIZE
frontend     latest    2a7621c1f881   4 minutes ago       118MB
<none>       <none>    195544bec9c0   21 minutes ago      118MB
backend      latest    0bf78f087d9c   31 minutes ago      1.25GB
image1       latest    d63fc68f9d0d   About an hour ago   507MB
[root@ip-172-31-23-207 client]# docker run -dt -p 84:3000 backend
d4890c25aa8d95b8b715e4ce4debafe3097c2fc4db854fec9b5045896e3a0b2d
[root@ip-172-31-23-207 client]# docker build -t frontend .
[+] Building 79.1s (3/3) FINISHED                                                                                                                                                                   docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                                          0.0s
 => => transferring dockerfile: 1.33kB                                                                                                                                                                        0.0s
 => CANCELED [internal] load metadata for docker.io/library/httpd:2.4                                                                                                                                        79.1s
 => [internal] load metadata for docker.io/library/node:18                                                                                                                                                    0.1s
ERROR: failed to solve: Canceled: context canceled
[root@ip-172-31-23-207 client]# docker run -dt -p 80:80 frontend
32c5c64297ef6676becd54dcf2a6417b540f728398299a2819abdaa402edb8b0
[root@ip-172-31-23-207 client]# docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED              STATUS              PORTS                                           NAMES
32c5c64297ef   frontend   "httpd-foreground"       4 seconds ago        Up 3 seconds        0.0.0.0:80->80/tcp, :::80->80/tcp               jolly_williams
d4890c25aa8d   backend    "docker-entrypoint.s…"   About a minute ago   Up About a minute   80/tcp, 0.0.0.0:84->3000/tcp, :::84->3000/tcp   objective_wilson
[root@ip-172-31-23-207 client]# 


---------------

Now ECS we add conatiner registry 


before that role to be created conatiner registry full access or elastics container regsitry 

how i can actually integrate above code with ECS FULL STEPS NEEDED and also how we can push to docker Hub by using CHATGPT help 




