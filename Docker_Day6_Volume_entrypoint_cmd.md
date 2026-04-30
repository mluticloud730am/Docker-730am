# 🐳 Docker Learning Notes

Personal study notes from hands-on lab sessions covering Docker core concepts — from basics to real-time production usage.

---

## 📁 Contents

| File | Description |
|---|---|
| `Docker_Complete_Notes.md` | Main notes — volumes, ENTRYPOINT vs CMD, Dockerfile, EBS/EFS, K8s |

---

## 📚 Topics Covered

- **Docker Fundamentals** — Images, containers, layers, lifecycle
- **Docker Volumes** — Named volumes, bind mounts, tmpfs, volume drivers
- **Cloud Storage** — AWS EBS vs EFS for Docker persistence
- **ENTRYPOINT vs CMD** — Deep dive with hands-on examples and common mistakes
- **Dockerfile** — Complete instruction reference and production best practices
- **Networking** — Port mapping, container-to-container communication
- **Docker Swarm** — Overview and why it's replaced by Kubernetes
- **Kubernetes** — EKS, AKS, GKE and the career path forward
- **Interview Q&A** — Fresher to 13 years experience level
- **Cheat Sheet** — All essential commands in one place

---

## 🧪 Lab Environment

- **Platform:** AWS EC2 (Amazon Linux)
- **Session Date:** April 30, 2025
- **Key hands-on:** Docker volumes (`vol1`, `vol2`, `vol3`, `vol4`), Flask app containerization, ENTRYPOINT/CMD behavior

---

## ⚡ Quick Reference

```bash
# Volume basics
docker volume create vol1
docker run -it --name=con1 -v vol1:/vol1 ubuntu
docker volume ls
docker volume inspect vol1

# Build and run Flask app
docker build -t img3 .
docker run -dt -p 5000:5000 img3
docker exec -it <container_id> /bin/bash

# Override CMD at runtime (ENTRYPOINT stays)
docker run -dt img3 app1.py

# Override ENTRYPOINT at runtime
docker run -it --entrypoint /bin/bash img3

# Cleanup
docker system prune -a --volumes
```

---

## 🗺️ Learning Path

```
Docker Basics → Dockerfile → Volumes → Networking
      ↓
Docker Compose (multi-container apps)
      ↓
Kubernetes Fundamentals (Pods, Deployments, Services)
      ↓
Cloud Managed K8s — EKS / AKS / GKE
      ↓
Helm, CI/CD, GitOps (ArgoCD), Service Mesh (Istio)
```

---

## 📝 Notes to Self

- [ ] Hands-on: ENTRYPOINT wrapper script pattern
- [ ] Hands-on: Multi-stage Docker builds
- [ ] Deep dive: Docker Swarm vs Kubernetes (orchestration comparison)
- [ ] Practice: EBS-backed Docker volume on EC2
- [ ] Start: Kubernetes basics (minikube or EKS)

---

*Maintained by: Veera | Last updated: April 30, 2025*
