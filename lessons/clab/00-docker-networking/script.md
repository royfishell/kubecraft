# Lesson 0: Docker Networking Fundamentals - Video Script

## Lesson Information

| Field | Value |
|-------|-------|
| **Lesson Number** | 00 |
| **Title** | Docker Networking Fundamentals |
| **Duration Target** | 12-14 minutes |
| **Prerequisites** | Basic Docker knowledge, Linux command line |
| **Learning Objectives** | Understand network drivers, create custom networks, connect containers across networks |

---

## Pre-Recording Checklist

- [ ] Docker installed and running
- [ ] No stale containers: `docker rm -f $(docker ps -aq) 2>/dev/null`
- [ ] No stale networks: `docker network prune -f`
- [ ] Screen resolution set (1920x1080)
- [ ] Terminal font size increased (14-16pt)
- [ ] Notifications disabled
- [ ] Clean terminal: `clear && history -c`

---

## Script

### Opening Hook (30 seconds)

> **[VOICEOVER - Terminal visible]**
>
> "Before we can build network labs, we need to understand how Docker handles networking. If you've ever wondered how containers talk to each other, why some can communicate by name and others can't, or how to isolate services - this lesson will clear that up.
>
> This is the foundation for everything we'll do with containerlab, where we'll run actual network operating systems in containers."

**Visual:** Terminal with `docker network ls` showing default networks

---

### Section 1: The Problem & Network Drivers (2 minutes)

> **[VOICEOVER]**
>
> "Every container needs networking. But different situations need different approaches. Sometimes containers should be isolated, sometimes they need to share the host's network, and sometimes they need their own private network.
>
> Docker solves this with network drivers. Let's see what's available by default."

```bash
docker network ls
```

**Expected Output:**
```
NETWORK ID     NAME      DRIVER    SCOPE
a1b2c3d4e5f6   bridge    bridge    local
g7h8i9j0k1l2   host      host      local
m3n4o5p6q7r8   none      null      local
```

> "Three default networks, each using a different driver:
>
> **Bridge** - the default. Creates an isolated network where containers get their own IPs.
>
> **Host** - no isolation. Container uses the host's network directly.
>
> **None** - complete isolation. No networking at all.
>
> For our network labs, we'll mostly use bridge networks because they give us the isolation and control we need."

**Key Points:**
- Bridge = isolated container network (most common)
- Host = share host network (performance, no isolation)
- None = no networking (special cases)

---

### Section 2: Default Bridge Limitations (3 minutes)

> **[VOICEOVER]**
>
> "Let's see the default bridge in action and understand its limitations."

```bash
# Run two containers on default network
docker run -d --name web1 alpine sleep 3600
docker run -d --name web2 alpine sleep 3600
```

```bash
# Get their IP addresses
docker inspect web1 --format='{{.NetworkSettings.Networks.bridge.IPAddress}}'
docker inspect web2 --format='{{.NetworkSettings.Networks.bridge.IPAddress}}'
```

> "They have IPs. Let's test connectivity."

```bash
# Install ping and test by IP
docker exec web1 apt-get update -qq && docker exec web1 apt-get install -y -qq iputils-ping
docker exec web1 ping -c 2 <web2-ip>
```

> "Ping by IP works. But what about by name?"

```bash
# Try to ping by container name
docker exec web1 ping -c 2 web2
```

**Expected:** `ping: web2: Name or service not found`

> "It fails. The default bridge network does NOT provide DNS resolution between containers. You can only communicate by IP address - and since IPs can change when containers restart, this is fragile.
>
> This is the first thing custom bridge networks fix."

```bash
# Cleanup
docker rm -f web1 web2
```

---

### Section 3: Custom Bridge Networks (3 minutes)

> **[VOICEOVER]**
>
> "Let's create a custom network and see the difference."

```bash
# Create custom network
docker network create app-network
```

```bash
# Run containers on our custom network
docker run -d --name web --network app-network alpine sleep 3600
docker run -d --name client --network app-network alpine sleep 3600
```

> "Now let's test DNS resolution."

```bash
# Ping by name - this should work!
docker exec client ping -c 2 web
```

> "It works. Custom bridge networks have an embedded DNS server that automatically resolves container names. This is huge for service discovery.
>
> Let's look at what Docker created."

```bash
docker network inspect app-network
```

> "You can see:
> - The subnet Docker assigned
> - The gateway IP
> - Both containers listed with their IPs
>
> This is similar to how Kubernetes pod networking works - containers can find each other by name."

```bash
# Cleanup
docker rm -f web client
docker network rm app-network
```

---

### Section 4: Network Isolation (2 minutes)

> **[VOICEOVER]**
>
> "What if we want containers that CAN'T communicate? Network isolation."

```bash
# Create two separate networks
docker network create frontend
docker network create backend
```

```bash
# Put containers on different networks
docker run -d --name public-web --network frontend alpine sleep 3600
docker run -d --name database --network backend alpine sleep 3600
```

> "Now try to reach the database from the web container."

```bash
# This will fail - different networks
docker exec public-web ping -c 2 database
```

> "They can't communicate. They're isolated. This is like VLANs in traditional networking - separate broadcast domains.
>
> But what if we have a service that needs access to both? An API server that talks to users AND the database?"

```bash
# Create an API container and connect it to BOTH networks
docker run -d --name api --network frontend alpine sleep 3600
docker network connect backend api
```

```bash
# Now API can reach both
docker exec api ping -c 2 public-web
docker exec api ping -c 2 database
```

> "The API container has interfaces on both networks. It bridges the gap while maintaining isolation between frontend and backend."

```bash
# Cleanup
docker rm -f public-web database api
docker network rm frontend backend
```

---

### Section 5: Docker Compose Networking (2 minutes)

> **[VOICEOVER]**
>
> "In practice, you'll often define networks in Docker Compose. Let's look at a real example."

**Show file content:**
```yaml
services:
  web:
    image: node:alpine
    networks:
      - public

  api:
    image: node:alpine
    command: sleep 3600
    networks:
      - public
      - internal

  db:
    image: postgres:alpine
    environment:
      POSTGRES_PASSWORD: example
    networks:
      - internal

networks:
  public:
  internal:
```

> "This defines three services:
> - web on the public network
> - api on both public and internal
> - db only on internal
>
> The database is completely hidden from the web tier. Only the API can reach it."

```bash
# Deploy
docker compose up -d

# Test - api can reach both
docker compose exec api ping -c 1 web
docker compose exec api ping -c 1 db

# web cannot reach db
docker compose exec web ping -c 1 db  # This fails
```

```bash
# Cleanup
docker compose down
```

---

### Section 6: Why This Matters for Containerlab (1 minute)

> **[VOICEOVER]**
>
> "Why did we cover all this? Because containerlab builds on these concepts.
>
> When you deploy a containerlab topology:
> - Each network device runs in its own container
> - Containerlab creates virtual links between them - similar to our network connections
> - Management traffic uses a Docker network
> - Data plane traffic uses virtual ethernet pairs
>
> Understanding Docker networking helps you troubleshoot containerlab issues and understand what's happening under the hood."

---

### Recap (30 seconds)

> **[VOICEOVER]**
>
> "Quick recap:
>
> - The default bridge network has no DNS - containers talk by IP only
> - Custom bridge networks provide DNS resolution by container name
> - Separate networks provide isolation - like VLANs
> - Containers can join multiple networks to bridge domains
> - Docker Compose makes network definitions declarative
>
> These concepts carry directly into containerlab and Kubernetes networking."

---

### Closing (30 seconds)

> **[VOICEOVER]**
>
> "Head to the exercises folder and practice:
> - Creating custom networks
> - Testing isolation
> - Connecting containers across networks
> - Using Docker Compose for multi-network setups
>
> In the next lesson, we'll use this knowledge to set up containerlab and deploy actual network operating systems.
>
> See you there."

---

## Post-Recording Checklist

- [ ] All containers removed: `docker rm -f $(docker ps -aq)`
- [ ] All custom networks removed: `docker network prune -f`
- [ ] Timing verified: ~12-14 minutes
- [ ] All commands worked correctly

---

## B-Roll / Supplementary Footage Needed

1. Network diagram showing bridge, host, none drivers
2. Animation showing DNS resolution in custom network
3. Diagram showing multi-network container
4. Docker Compose network architecture diagram

---

## Notes for Editing

- **3:00** - Emphasize the DNS failure moment
- **5:00** - Side-by-side showing ping success vs failure
- **7:00** - Diagram overlay showing multi-network archqitecture
- **End** - Preview of containerlab with network devices
