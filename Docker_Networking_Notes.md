# Docker Networking — Complete Notes
> Covers: Fresher → 13 Years Real-Time Experience Level  
> Session Date: April 29, 2026 | Hands-on: EC2 (Amazon Linux), t2.medium

---

## Table of Contents
1. [Why Docker Networking Exists](#1-why-docker-networking-exists)
2. [The Three Network Types](#2-the-three-network-types)
3. [Bridge Network — Deep Dive](#3-bridge-network--deep-dive)
4. [Host Network — Deep Dive](#4-host-network--deep-dive)
5. [None Network — Deep Dive](#5-none-network--deep-dive)
6. [Custom Bridge Networks](#6-custom-bridge-networks)
7. [Port Publishing — `-p` vs `-P`](#7-port-publishing----p-vs--p)
8. [Hands-On Walkthrough — What Happened in Your Lab](#8-hands-on-walkthrough--what-happened-in-your-lab)
9. [Real-World Traffic Flow](#9-real-world-traffic-flow)
10. [Docker Networking in Production Reality](#10-docker-networking-in-production-reality)
11. [DNS Resolution Inside Docker Networks](#11-dns-resolution-inside-docker-networks)
12. [Docker Networking vs Kubernetes Networking](#12-docker-networking-vs-kubernetes-networking)
13. [Network Troubleshooting Commands](#13-network-troubleshooting-commands)
14. [Interview Questions — Fresher to Senior](#14-interview-questions--fresher-to-senior)
15. [Cheat Sheet](#15-cheat-sheet)

---

## 1. Why Docker Networking Exists

By default, containers are isolated — they have their own filesystem, their own processes, and their own network. Docker Networking bridges that isolation to allow:

- Containers to talk to each other (inter-container communication)
- External traffic to reach containers (inbound from internet/user)
- Containers to reach the internet (outbound — for pulling packages, APIs, etc.)

Without networking, a container is completely isolated — like a VM with no network card.

### Docker's Responsibility vs Kubernetes' Responsibility

This is an important mindset shift:

```
Docker's Job:
  → Write Dockerfile
  → Build image
  → Understand networking for local dev/testing

Kubernetes' Job (Production):
  → Run containers (pods)
  → Handle networking between pods (CNI plugins — Calico, Flannel, Cilium)
  → Expose services (ClusterIP, NodePort, LoadBalancer, Ingress)
  → Scale, heal, update containers
```

In real production environments, you rarely use `docker network` commands directly. Kubernetes has its own networking model that completely replaces Docker's. But understanding Docker networking makes Kubernetes networking much easier to grasp.

---

## 2. The Three Network Types

```
docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
67897621fc91   bridge    bridge    local   ← Default
da042981f850   host      host      local
cc7ed4a97227   none      null      local
```

| Network Type | IP Assigned | Port Mapping Needed | Container Isolation | Use Case |
|---|---|---|---|---|
| bridge | Yes (172.17.x.x) | Yes (-p flag) | Isolated from host | Default — apps, databases |
| host | No (uses EC2 IP) | No (not applicable) | Shares host network | Performance-critical apps |
| none | No | No | Fully isolated | Security-critical, batch jobs |

---

## 3. Bridge Network — Deep Dive

### How It Works

Bridge network creates a **virtual switch (docker0)** on the host. Every container gets its own IP in the `172.17.0.0/16` subnet. The gateway (`172.17.0.1`) acts like a router — all containers can reach each other through it, similar to how EC2 instances in the same VPC subnet can communicate through the VPC router.

```
Analogy:
  VPC          = Docker Bridge Network
  EC2 private IP = Container IP (172.17.0.x)
  VPC router   = Bridge Gateway (172.17.0.1)
  Internet Gateway = Port mapping (-p) + EC2 SG
```

### What docker inspect Reveals (from your lab)

```json
"Networks": {
    "bridge": {
        "Gateway": "172.17.0.1",     ← All containers share this gateway
        "IPAddress": "172.17.0.2",   ← nginx container 1
        "IPPrefixLen": 16,           ← /16 = 172.17.0.0 to 172.17.255.255 (~65k IPs)
        "MacAddress": "02:42:ac:11:00:02"
    }
}
```

Your containers in the lab:
```
172.17.0.1   → Gateway (docker0 bridge interface on host)
172.17.0.2   → nginx (priceless_almeida) — no port mapping → not externally accessible
172.17.0.3   → nginx (upbeat_wu) — mapped -p 81:80 → accessible via EC2:81
172.17.0.4   → httpd/apache (apache-server) — mapped -p 80:80 → accessible via EC2:80
172.17.0.5   → ubuntu container (for ping test)
```

### Inter-Container Communication Proof (Your Lab)

You proved containers can reach each other inside the bridge network:

```bash
# Inside ubuntu container (172.17.0.5), ping httpd (172.17.0.4)
ping 172.17.0.4
# 64 bytes from 172.17.0.4: icmp_seq=1 ttl=127 time=0.095 ms  ← SUCCESS!
# 0% packet loss — containers communicate freely on the same bridge
```

### Why ttl=127 Instead of 128?

A ping from one container to another via the bridge gateway decrements TTL by 1. TTL starts at 128 (Windows/Linux default) → 128 - 1 hop = 127. This confirms traffic is routing through the gateway, not going directly.

### Bridge Network Isolation Rules

```
Same bridge network  → containers CAN ping each other by IP
Different bridge     → containers CANNOT communicate (isolated)
```

Your lab proved this too:

```bash
# Container in custom 'dev' network (172.18.0.x)
ping 172.17.0.4   # ping from dev network to default bridge
# NO response — different bridge networks are isolated!
```

### Default Bridge Limitation — No DNS by Name

On the **default** bridge network, containers can only reach each other by IP, not by name:

```bash
# Default bridge — NAME resolution fails
docker run -it ubuntu bash
ping nginx_container    # FAILS — default bridge has no automatic DNS
ping 172.17.0.2         # WORKS — IP works

# Custom bridge — NAME resolution works!
docker network create mynet
docker run --network mynet --name app1 nginx
docker run --network mynet ubuntu ping app1  # WORKS — DNS by name!
```

This is one of the biggest reasons to always use custom bridge networks in production setups.

---

## 4. Host Network — Deep Dive

### How It Works

The container **shares the host's network stack completely**. No virtual bridge, no separate IP, no port mapping. The container's process binds directly to the host's network interfaces.

```bash
docker run -dt --network host nginx
```

### What docker inspect Shows (from your lab)

```json
"Networks": {
    "host": {
        "Gateway": "",       ← No gateway — using host directly
        "IPAddress": "",     ← No separate IP — container IS the host
        "MacAddress": ""     ← No separate MAC
    }
}
```

Compare with bridge inspect:
```
Bridge: Gateway=172.17.0.1, IPAddress=172.17.0.2   ← Separate virtual network
Host:   Gateway="",         IPAddress=""             ← Uses EC2's own 172.31.x.x IP
```

### The Port Conflict Problem (Exactly What Happened in Your Lab)

```bash
# Step 1: nginx via host network works when port 80 is free
docker run -dt --network host nginx
# → nginx binds to 0.0.0.0:80 on the EC2 itself → accessible at http://EC2_IP/

# Step 2: Install nginx on the EC2 host itself
yum install nginx -y
systemctl start nginx
# → System nginx now owns port 80

# Step 3: Try another nginx container with host network
docker run -dt --network host --name rakesh nginx
docker logs rakesh
# → bind() to 0.0.0.0:80 failed (98: Address already in use)
# → nginx container CRASHES because port 80 is taken by system nginx!

# Verified with:
ss -tuln | grep 80
# tcp  LISTEN  0.0.0.0:80  ← system nginx owns this port
```

### Warning: `-p` is Ignored with Host Network

```bash
docker run -dt -p 81:80 --network host nginx
# WARNING: Published ports are discarded when using host network mode
```

Port mapping is meaningless with host networking — the container already IS the host. The `-p 81:80` flag is completely ignored. The container still tries to bind to port 80 on the host.

### When to Use Host Network

| Scenario | Justified? |
|---|---|
| High-throughput network performance (low latency apps) | Yes |
| EC2 dedicated entirely to containers (no other services) | Yes |
| Multiple replicas of same app on same host | NO — port conflicts |
| Mixed: some apps on host, some in containers | NO — conflicts |
| Production containerized workloads | Rarely — use bridge with port mapping |

### Host Network in Kubernetes

Kubernetes also supports `hostNetwork: true` in pod specs but it's discouraged and rarely used. Kubernetes provides its own networking model (CNI) that is much more powerful and avoids these conflicts.

---

## 5. None Network — Deep Dive

### How It Works

The container has **no network interface** (except loopback `lo`). It cannot receive or send any network traffic. Completely isolated.

```bash
docker run -dt -p 81:80 --network none nginx
# The -p flag is also meaningless here — there's no network to map!
```

### What docker inspect Shows (from your lab)

```json
"Networks": {
    "none": {
        "Gateway": "",       ← No gateway
        "IPAddress": "",     ← No IP
        "MacAddress": ""     ← No MAC
    }
}
```

### When to Use None Network

| Use Case | Why None? |
|---|---|
| Data processing jobs (read from disk, write to disk) | No network needed, security boundary |
| Cryptocurrency / sensitive computation | Prevent data exfiltration |
| ML model training (data already mounted via volume) | Isolation |
| Security scanning containers | Run untrusted code safely |
| Offline batch jobs | Deterministic, no network interference |

### None Network Security Benefit

```
Attacker exploits a container vulnerability...
→ With bridge: can scan internal network, reach gateway, potentially escape to other containers
→ With none:   can't reach anything — complete blast radius containment
```

---

## 6. Custom Bridge Networks

### Why Create Custom Networks?

The default bridge has two major problems:
1. No DNS resolution by container name
2. All containers on default bridge can talk to each other — no isolation between apps

Custom networks solve both:

```bash
# Create custom network
docker network create dev
docker network create prod
docker network create --subnet=192.168.10.0/24 myapp-net   # Custom subnet

# Run containers in custom networks
docker run -dt --network dev --name frontend nginx
docker run -dt --network dev --name backend myapp

# DNS works by name in custom networks!
docker exec frontend ping backend   # WORKS! DNS resolves 'backend' to its IP

# Isolation: prod containers cannot talk to dev containers
docker run -dt --network prod --name db mysql
docker exec frontend ping db   # FAILS — different network, isolated
```

### Your Lab — Custom `dev` Network

```bash
docker network create dev
# f3631353aeff — new network created

docker network ls
# bridge  bridge   local   ← Default (172.17.x.x)
# dev     bridge   local   ← Custom (172.18.x.x — next subnet)
# host    host     local
# none    null     local

# Container in dev network got a different subnet
"Gateway": "172.18.0.1",
"IPAddress": "172.18.0.2",
# → Isolated from the default bridge (172.17.x.x)
```

### Custom Network IP Ranges

Docker auto-assigns subnets to new networks in sequence:
```
docker0 (default bridge)  →  172.17.0.0/16
first custom network      →  172.18.0.0/16
second custom network     →  172.19.0.0/16
...
```

You can specify your own:
```bash
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/16 \
  --gateway 10.10.0.1 \
  --ip-range 10.10.1.0/24 \
  myapp-net
```

### Connecting a Running Container to a Network

```bash
# Container can be in MULTIPLE networks simultaneously
docker network connect myapp-net existing_container
docker network disconnect myapp-net existing_container
```

---

## 7. Port Publishing — `-p` vs `-P`

### Lowercase `-p` — Explicit Mapping

```bash
# -p host_port:container_port
docker run -dt -p 80:80 nginx         # Host port 80 → container port 80
docker run -dt -p 81:80 nginx         # Host port 81 → container port 80
docker run -dt -p 8080:80 nginx       # Host port 8080 → container port 80
docker run -dt -p 127.0.0.1:80:80 nginx  # Only localhost — not public!
docker run -dt -p 80:80 -p 443:443 nginx # Multiple ports
```

### Uppercase `-P` — Auto Port Assignment (What Your Lab Showed)

```bash
docker run -dt -P nginx
# PORTS: 0.0.0.0:32768->80/tcp   ← Docker picked port 32768 automatically!

ss -tuln | grep 32768
# tcp  LISTEN  0.0.0.0:32768  ← Port is open on host
```

Docker picks a random ephemeral port (range: 32768–60999 by default) and maps it to each EXPOSE'd container port. Useful for:
- Running many replicas without manually tracking ports
- Testing when you don't care which host port is used
- CI/CD pipelines that discover the port programmatically

```bash
# Discover the assigned port programmatically
docker port <container_id>
# 80/tcp -> 0.0.0.0:32768

# Or in scripts:
docker inspect -f '{{(index (index .NetworkSettings.Ports "80/tcp") 0).HostPort}}' container_id
```

### EXPOSE vs Port Mapping — The Confusion

```bash
# EXPOSE in Dockerfile:
EXPOSE 5000   # ← Documentation only! Does NOT open any ports.

# docker ps output:
PORTS: 5000/tcp                     ← EXPOSE only — NOT accessible externally
PORTS: 0.0.0.0:8080->5000/tcp       ← -p flag used — accessible externally
```

EXPOSE is a hint/documentation. Without `-p` in `docker run`, no traffic can reach the container from outside.

---

## 8. Hands-On Walkthrough — What Happened in Your Lab

### Sequence of Events

```
1. docker run -dt nginx
   → Container: 172.17.0.2 (no port mapping)
   → docker ps PORTS: 80/tcp  ← Not accessible from internet
   → EXPOSE in nginx Dockerfile just documents port 80

2. docker run -dt -p 81:80 nginx
   → Container: 172.17.0.3 (bridge network)
   → docker ps PORTS: 0.0.0.0:81->80/tcp ← Accessible at EC2_IP:81

3. docker run -dt -p 80:80 --name apache-server httpd
   → Container: 172.17.0.4 (bridge network)
   → docker ps PORTS: 0.0.0.0:80->80/tcp ← Accessible at EC2_IP:80

4. docker run -it ubuntu /bin/bash
   → Container: 172.17.0.5 (bridge)
   → Installed ping: apt update && apt install iputils-ping -y
   → ping 172.17.0.4 → SUCCESS (0% loss, ttl=127) ← Bridge comms work!

5. docker network create dev
   → New bridge: 172.18.0.0/16, gateway: 172.18.0.1

6. docker run -it --network dev ubuntu
   → Container: 172.18.0.2 (dev network)
   → ping 172.17.0.4 → NO RESPONSE ← Cross-network isolation!

7. docker run -dt --network host nginx
   → Container: no IP, uses EC2's 172.31.24.140
   → PORTS: (empty) — no port mapping needed
   → Accessible at http://EC2_IP/ directly!

8. yum install nginx && systemctl start nginx
   → System nginx now owns port 80 on the host

9. docker run -dt --network host --name rakesh nginx
   → docker logs rakesh: bind() to 0.0.0.0:80 failed (98: Address already in use)
   → Container fails — host port conflict!

10. yum remove nginx  ← removed system nginx to free port 80

11. docker run -dt -p 81:80 --network host nginx
    → WARNING: Published ports are discarded when using host network mode
    → Container still tries port 80 (ignores -p flag in host mode)
    → SUCCESS now that system nginx is removed

12. docker run -dt -P nginx
    → PORTS: 0.0.0.0:32768->80/tcp ← Auto-assigned port 32768!
    → ss -tuln confirms port 32768 is listening

13. docker run -dt -p 81:80 --network none nginx
    → -p flag ignored — no network interface exists
    → Container runs but is completely network-isolated
    → inspect shows: Gateway="", IPAddress="" — no network at all
```

### Mistakes Caught and Corrected

```bash
docker run -ft --network host nginx   # WRONG — 'f' is not a valid flag
docker run -dt --network host nginx   # CORRECT

docker run -dt host --name rakesh nginx  # WRONG — 'host' interpreted as image name!
# "unable to find image 'host:latest'"
docker run -dt --network host --name rakesh nginx  # CORRECT — --network flag needed

dcoker logs 8f356d45a89c  # TYPO
docker logs  8f356d45a89c  # CORRECT
```

---

## 9. Real-World Traffic Flow

### Complete User → Application Path

```
User Browser
    ↓  DNS resolves to EC2 Public IP
Internet
    ↓  Port 80 or 81
EC2 Security Group (inbound rules — allow 80, 81)
    ↓
EC2 Host (172.31.24.140)
    ↓  iptables NAT rules (Docker manages this automatically)
Docker0 Bridge Gateway (172.17.0.1)
    ↓  Routes to container IP
Container Port 80 (e.g., httpd at 172.17.0.4:80)
    ↓
Application responds → reverse path back to user
```

### What Docker Does Behind the Scenes for Port Mapping

When you run `-p 80:80`, Docker automatically creates iptables NAT rules:

```bash
# Docker's auto-generated iptables rules (view them):
iptables -t nat -L DOCKER --line-numbers

# It creates something like:
# DNAT tcp -- anywhere anywhere tcp dpt:80 → 172.17.0.4:80
# This is transparent — Docker manages it, you don't touch it
```

---

## 10. Docker Networking in Production Reality

### What You ACTUALLY Use in Real Projects

In real DevOps/SRE work with Docker:

```
Local Development:
  → docker-compose.yml handles networking automatically
  → All services on a custom bridge by default
  → Services talk to each other by name (db, redis, api)

CI/CD Pipelines:
  → Docker bridge for ephemeral test containers
  → Containers share a network for integration tests

Production (Kubernetes):
  → You DON'T use docker network commands
  → Kubernetes CNI (Calico, Flannel, Cilium) handles pod networking
  → Kubernetes Services handle service discovery + load balancing
```

### Docker Compose Networking (Preview — Next Session Topic)

```yaml
version: '3'
services:
  web:
    image: nginx
    ports: ["80:80"]          # Port mapping
    networks: [frontend]      # Connect to frontend network

  api:
    image: myapp
    networks: [frontend, backend]  # Connected to both networks

  db:
    image: postgres
    networks: [backend]       # Only on backend — not accessible from web

networks:
  frontend:                   # Custom bridge auto-created
  backend:                    # Isolated from frontend
```

```
web ↔ api    (both on frontend network — can communicate)
api ↔ db     (both on backend network — can communicate)
web ✗ db     (different networks — CANNOT communicate directly)
```

This is the correct security model: the web tier cannot directly access the database.

### Port Conflict Prevention Strategy

```bash
# Check what's using a port before running containers
ss -tuln | grep 80         # Modern replacement for netstat
netstat -tulnp | grep 80   # Old way (netstat may not be installed)
lsof -i :80                # List processes on port 80

# Best practice: Never install apps directly on the container host
# The EC2 should be dedicated to running containers only
```

---

## 11. DNS Resolution Inside Docker Networks

### Default Bridge — IP Only

```bash
# Containers on default bridge: no automatic name resolution
docker run --name app1 nginx
docker run ubuntu ping app1   # FAILS — default bridge has no DNS

# Must use IP:
docker run ubuntu ping 172.17.0.2   # WORKS
```

### Custom Bridge — DNS Included

```bash
# Custom bridge has built-in DNS server (127.0.0.11)
docker network create mynet
docker run --network mynet --name web nginx
docker run --network mynet --name db postgres

# Inside web container:
ping db           # WORKS — resolves via Docker's embedded DNS
curl http://db    # WORKS — container name as hostname
```

### How Docker's Embedded DNS Works

```
Container DNS query: "Where is 'db'?"
    ↓
127.0.0.11:53 (Docker's embedded DNS resolver in each container)
    ↓
Docker daemon looks up container name in network config
    ↓
Returns: "db" → 172.18.0.3
    ↓
Container connects to 172.18.0.3
```

View in resolv.conf inside a custom-network container:
```bash
cat /etc/resolv.conf
# nameserver 127.0.0.11    ← Docker's embedded DNS
# options ndots:0
```

### Network Aliases (Load Balancing)

```bash
# Multiple containers can share one DNS name
docker run --network mynet --network-alias app nginx   # replica 1
docker run --network mynet --network-alias app nginx   # replica 2

# Other containers querying "app" get load-balanced across both
# Docker uses round-robin DNS — basic load balancing!
```

---

## 12. Docker Networking vs Kubernetes Networking

| Concept | Docker | Kubernetes |
|---|---|---|
| Default network | bridge (docker0) | Flat pod network (CNI plugin) |
| Container IP | 172.17.x.x (host-scoped) | Routable across ALL nodes in cluster |
| Service discovery | Container name (custom bridge) | Kubernetes Service + CoreDNS |
| Load balancing | Round-robin DNS | Service (ClusterIP, iptables/IPVS) |
| External access | Port mapping (-p) | NodePort / LoadBalancer / Ingress |
| Network policies | None (all containers on same bridge can talk) | NetworkPolicy (fine-grained pod-to-pod rules) |
| Multi-host | Not supported natively | Native (pods on any node) |
| Configuration | CLI flags | YAML manifests |

### Kubernetes Network Flow (Equivalent of Your Lab's Flow)

```
Internet
    ↓
AWS Load Balancer (ALB/NLB)
    ↓
Kubernetes Ingress Controller
    ↓
Kubernetes Service (ClusterIP)
    ↓
Pod (container) IP — managed by CNI (Calico/Flannel)
```

---

## 13. Network Troubleshooting Commands

### Inspect Container Network

```bash
docker inspect <container> | grep -A 20 '"Networks"'
docker inspect <container> -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
docker inspect <container> -f '{{.NetworkSettings.Ports}}'
```

### Check What Ports Are Open on Host

```bash
ss -tuln                        # Modern — show listening ports
netstat -tulnp                  # Classic (may need net-tools package)
ss -tuln | grep 80              # Check specific port
```

### Container-to-Container Testing

```bash
# Install networking tools in ubuntu container
apt update && apt install -y iputils-ping curl net-tools

# Ping another container
ping 172.17.0.4

# Test HTTP connection to another container
curl http://172.17.0.4:80

# DNS lookup (custom network)
nslookup mycontainername
```

### Check Docker Networks

```bash
docker network ls                               # List networks
docker network inspect bridge                   # Detailed bridge info
docker network inspect bridge | grep -A 5 "Containers"  # Who's on this network?
```

### Container Port Mapping

```bash
docker port <container>                         # Show all port mappings
docker port <container> 80                      # Show specific port mapping
```

### Logs for Network Errors

```bash
docker logs <container>                         # See what happened
docker logs <container> | grep -i "error\|fail\|bind"  # Filter network errors
```

---

## 14. Interview Questions — Fresher to Senior

### Fresher Level

**Q: What are the three Docker network types?**
A: Bridge (default — containers get their own IP, route through gateway), Host (container shares EC2/host network stack directly, no separate IP), and None (completely isolated, no network interface at all).

**Q: What is the default Docker network and what subnet does it use?**
A: The `bridge` network is the default. Docker assigns IPs from the `172.17.0.0/16` subnet. The gateway is `172.17.0.1` (the `docker0` virtual interface on the host).

**Q: What does EXPOSE do in a Dockerfile?**
A: EXPOSE is documentation-only. It declares that the container intends to listen on that port, but does NOT open or publish the port. You need `-p host:container` in `docker run` to actually make it accessible.

**Q: What is the difference between `-p` and `-P` in docker run?**
A: `-p` (lowercase) lets you specify an exact host-to-container port mapping. `-P` (uppercase) automatically maps each EXPOSE'd container port to a random ephemeral host port (32768+).

### Mid Level

**Q: Why would a container with `--network host` fail to start?**
A: Because host networking makes the container share the host's network stack directly. If the port the container needs (e.g., 80) is already in use by another process on the host (systemd service, another container, etc.), the `bind()` call fails with "Address already in use". This is exactly what happened in the lab when system nginx was running on port 80.

**Q: What is the difference between the default bridge network and a custom bridge network?**
A: The default bridge network does not support DNS name resolution — containers must communicate by IP. Custom bridge networks have Docker's embedded DNS server (127.0.0.11) built in, so containers can reach each other by container name. Custom networks also provide better isolation — only containers explicitly added to that network can communicate.

**Q: Can a container be on multiple networks simultaneously?**
A: Yes. You can connect a container to multiple networks using `docker network connect`. This is common for "middleware" containers (like an API service that needs to talk to both a frontend network and a backend database network).

**Q: When `-p 81:80` is used with `--network host`, what happens?**
A: Docker ignores the port mapping entirely and prints a warning: "Published ports are discarded when using host network mode." The container still tries to bind to its native port (80) on the host's network directly. The `-p` flag has no effect with host networking.

### Senior Level

**Q: A microservices app has 5 containers — web, api, auth, db, cache. Design the network layout.**
A: Use three custom networks for isolation: `frontend-net` (web + api), `backend-net` (api + auth + cache), `data-net` (auth + db). Web can only reach api. Api can reach auth and cache. Auth is the only service that can reach db. The database is completely isolated from web and cache. This follows the principle of least privilege and limits blast radius of a compromise.

**Q: How does Docker implement port mapping under the hood?**
A: Docker manipulates the host's `iptables` NAT table. When you run `-p 80:80`, Docker adds a DNAT rule that rewrites incoming packets destined for host port 80 to the container's IP and port 80. It also sets up MASQUERADE rules for outbound traffic from containers. You can view these with `iptables -t nat -L DOCKER`.

**Q: What are the security implications of `--network host`?**
A: With host networking, the container can listen on any host port, access any service bound to localhost on the host, potentially reach the EC2 metadata service (169.254.169.254) which could expose IAM credentials, and if the container is compromised, the attacker has direct access to the host's full network stack. It should be avoided in production unless there's a specific, justified performance requirement, and the host should be hardened accordingly.

**Q: How does Kubernetes handle the problem that Docker solves with bridge networking?**
A: Kubernetes uses CNI (Container Network Interface) plugins like Calico, Flannel, or Cilium. Every pod gets a unique, cluster-routable IP address — pods on any node can reach pods on any other node without NAT. Service discovery is handled by Kubernetes Services backed by CoreDNS, which provides name-based discovery (similar to custom bridge DNS but cluster-wide). Network policies provide fine-grained allow/deny rules between pods — far more powerful than Docker's network isolation.

---

## 15. Cheat Sheet

### Network Commands

```bash
# List networks
docker network ls

# Create networks
docker network create mynet
docker network create --driver bridge --subnet 10.10.0.0/16 mynet
docker network create --driver host mynet  # Not commonly done

# Inspect
docker network inspect bridge
docker network inspect mynet

# Connect/disconnect
docker network connect mynet container1
docker network disconnect mynet container1

# Remove
docker network rm mynet
docker network prune           # Remove all unused networks

# Run container with network
docker run -dt --network bridge nginx     # Default bridge
docker run -dt --network host nginx       # Host network (no port mapping)
docker run -dt --network none nginx       # Completely isolated
docker run -dt --network mynet nginx      # Custom network
```

### Port Publishing

```bash
docker run -dt -p 80:80 nginx           # Explicit: host 80 → container 80
docker run -dt -p 8080:80 nginx         # Different host port
docker run -dt -p 127.0.0.1:80:80 nginx # Localhost only
docker run -dt -P nginx                 # Auto-assign all EXPOSE'd ports
docker port <container>                 # Check mapped ports
```

### Debugging

```bash
docker inspect <id> | grep -A 20 Networks   # Network config
docker exec <id> cat /etc/resolv.conf        # DNS config inside container
docker exec <id> ip addr                     # Container IP addresses
docker exec <id> ip route                    # Container routing table
ss -tuln                                     # Host port usage
docker network inspect bridge               # All containers on bridge
```

### Network Type Summary

| Flag | Network | IP | Port Mapping | Use When |
|---|---|---|---|---|
| (none) | bridge | 172.17.x.x | Required (-p) | Default — most cases |
| `--network host` | host | EC2's IP | N/A (ignored) | Need host-level access |
| `--network none` | none | None | N/A | Isolation required |
| `--network mynet` | custom bridge | 172.18.x.x | Required (-p) | Multi-container apps |

---

## Key Takeaways from Today's Session

1. **Default is bridge** — every container gets `172.17.0.x`, gateway is `172.17.0.1`
2. **All containers on the same bridge can ping each other** — confirmed by your ubuntu→httpd ping test
3. **Different bridge networks are isolated** — containers in `dev` (172.18.x.x) cannot reach `bridge` (172.17.x.x)
4. **Host network = EC2's network** — no separate IP, no port mapping, highest risk of port conflicts
5. **The "Address already in use" error** means the host port is taken by another process — check with `ss -tuln`
6. **`-p` with `--network host` is silently ignored** — Docker warns but still runs
7. **`-P` (uppercase)** auto-assigns a random ephemeral port — useful for testing
8. **`--network none` has no IP, no gateway** — `docker inspect` shows empty strings for both
9. **Custom networks have DNS** — containers can reach each other by name; default bridge cannot
10. **In production, Kubernetes owns networking** — Docker networking knowledge helps understand K8s but you rarely use it directly in production

---

*Notes compiled: April 29, 2026 | Level: Fresher → 13 Years Experience*
