# 🐳 Docker Fundamentals & Node.js Optimization

## 📌 Overview
This repository demonstrates core Docker concepts with hands-on examples:
- Dockerfile basics
- CMD vs ENTRYPOINT
- RUN vs CMD differences
- ADD vs COPY
- Port mapping & EXPOSE
- Node.js image optimization

---

# 🧱 1. Basic Dockerfile

```dockerfile
FROM ubuntu:latest
CMD ["echo", "Hello World"]
```

### 🔍 Explanation
- `FROM` → Base image  
- `CMD` → Default command when container starts  

---

# ⚙️ 2. CMD Behavior

### Run with default CMD
```bash
docker run img1
```
👉 Output:
```
Hello World
```

### Override CMD
```bash
docker run img1 echo welcome
```
👉 Output:
```
welcome
```

### ❗ Key Point
CMD gets **replaced** when you pass a command.

---

# 🚨 3. Common Error

```bash
docker run img1 hi
```

👉 Error:
```
executable file not found
```

### 🔍 Reason
Docker tries to execute `hi` as a command, which doesn’t exist.

---

# 🔥 4. ENTRYPOINT vs CMD

## Example 1
```dockerfile
FROM ubuntu:latest
ENTRYPOINT ["echo"]
CMD ["Hello World"]
```

### Behavior
```bash
docker run image
→ Hello World

docker run image DevOps
→ DevOps
```

---

## Example 2
```dockerfile
FROM ubuntu:latest
ENTRYPOINT ["echo", "Hello"]
```

```bash
docker run image
→ Hello

docker run image DevOps
→ Hello DevOps
```

---

## 🎯 Difference

| Instruction | Behavior |
|------------|---------|
| CMD | Replaced |
| ENTRYPOINT | Appended |

---

# ⚡ 5. RUN vs CMD vs ENTRYPOINT

| Instruction | Execution Time | Purpose |
|------------|--------------|--------|
| RUN | Build time | Install packages |
| CMD | Runtime | Default command |
| ENTRYPOINT | Runtime | Fixed executable |

---

# 📦 6. ADD vs COPY

## Example
```dockerfile
ADD https://github.com/torvalds/linux/archive/refs/tags/v5.14.tar.gz /app/
```

### 🔍 ADD Features
- Downloads from URL  
- Extracts `.tar` files automatically  

### ⚠️ Best Practice
Use `COPY` unless you need ADD-specific features.

---

# 🔄 7. Container Lifecycle

```bash
docker run img1
docker ps
```

👉 No running container

```bash
docker ps -a
```

👉 Container exited

### 🔍 Reason
Container stops after executing the command.

---

# 🖥️ 8. Access Running Container

```bash
docker exec -it <container_id> /bin/bash
```

---

# 🚀 9. Node.js Docker Image (Unoptimized)

```dockerfile
FROM node:18

WORKDIR /app
COPY . .
RUN npm install
EXPOSE 3000
CMD ["npm", "start"]
```

### ❌ Issues
- Large size (~1GB)
- Includes dev dependencies
- Slow build
- Runs as root

---

# ✅ 10. Optimized Node.js Docker Image

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

### ✅ Benefits
- Small image (~120MB)
- Faster builds
- Secure (non-root user)
- Better caching

---

# 📊 Image Comparison

| Image | Size |
|------|------|
| node:18 | ~1GB |
| node:18-alpine | ~120MB |

---

# 🌐 11. EXPOSE vs Port Mapping

### Dockerfile
```dockerfile
EXPOSE 3000
```

👉 Only documentation

---

### Run command
```bash
docker run -p 3002:3000 my-node-app
```

### 🔍 Meaning
- Host Port: **3002**
- Container Port: **3000**

---

### Access Application
```
http://<EC2-IP>:3002
```

---

# 🧠 12. Docker Image Layers

```bash
docker history node
```

### 🔍 Insight
- Each instruction = layer
- Large layers = heavy operations

---

# ⚖️ 13. node:18 vs node:18-alpine

| Feature | node:18 | node:18-alpine |
|--------|--------|----------------|
| Base OS | Debian | Alpine |
| Size | Large | Small |
| Use Case | Dev | Production |

---

# 🧪 14. Troubleshooting

## Image not found
```bash
docker run my-node-app
```

### Fix
```bash
docker build -t my-node-app .
```

---

## Wrong tag
```
node:18-alphine ❌
node:18-alpine ✅
```

---

## CMD syntax error
```
CMD["echo","Hello"] ❌
CMD ["echo", "Hello"] ✅
```

---

# 🎯 Key Takeaways

- CMD → Default command (replaceable)
- ENTRYPOINT → Fixed command (appends args)
- EXPOSE → Documentation only
- `-p` → Actual port mapping
- Alpine → Lightweight and efficient
- Optimize images for production

---

# 🚀 Conclusion

Docker is not just about running containers —  
it’s about building **efficient, secure, and production-ready images**.
