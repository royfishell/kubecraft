# Lesson 1: Containerlab Primer - Video Script

## Lesson Information

| Field | Value |
|-------|-------|
| **Lesson Number** | 01 |
| **Title** | Containerlab Primer |
| **Duration Target** | 12-15 minutes |
| **Prerequisites** | Lesson 0: Docker Networking, Linux environment with containerlab |
| **Learning Objectives** | Deploy containerlab topology, understand topology files, connect to network devices |

---

## Pre-Recording Checklist

- [ ] Lab environment tested (containerlab installed, Docker running)
- [ ] SR Linux image pulled: `docker pull ghcr.io/nokia/srlinux:24.10.1`
- [ ] Screen resolution set (1920x1080)
- [ ] Terminal font size increased (14-16pt)
- [ ] Notifications disabled
- [ ] Clean terminal: `clear && history -c`
- [ ] No labs running: `containerlab destroy --all`

---

## Script

### Opening Hook (30 seconds)

> **[VOICEOVER - Terminal visible]**
>
> "As a DevOps engineer, you've probably hit a networking issue and thought 'I wish I could just test this somewhere safe.' Maybe a route isn't working, packets are being dropped, or you need to understand how traffic flows between services.
>
> That's exactly what containerlab gives you - a way to spin up real network operating systems in containers and test changes before they hit production.
>
> In this lesson, you'll deploy your first network lab and learn the fundamentals of containerlab."

**Visual:** Terminal with `docker ps` showing no containers

---

### Section 1: What is Containerlab? (2 minutes)

> **[VOICEOVER]**
>
> "Containerlab is an open-source tool that lets you run network operating systems in Docker containers. Think of it like Docker Compose, but specifically designed for network topologies.
>
> Instead of expensive physical routers or heavy virtual machines, you can run production-grade network operating systems like Nokia SR Linux, Arista EOS, or open-source options like FRRouting - all in lightweight containers.
>
> For DevOps engineers, this is powerful because you can:
> - Test network changes before deploying them
> - Reproduce network issues from production in a safe environment
> - Build network automation and validate it works
> - Include network testing in your CI/CD pipelines

**Visual:** Show containerlab.dev website briefly

> "The containerlab documentation at containerlab.dev is excellent - it's your primary reference. There's also an active Discord community where you can get help."

**Visual:** Scroll through docs quickly, show Discord link

> "And if you search GitHub for the containerlab topic, you'll find hundreds of example labs from the community."

**Transition:** "Let's get our environment set up."

---

### Section 2: Environment Setup (2 minutes)

> **[VOICEOVER]**
>
> "For this course, you need containerlab and Docker installed on your Linux system. If you haven't set this up yet, follow the Linux Setup guide in the course documentation."

**Demo Commands:**
```bash
# Navigate to the lesson
cd lessons/clab/01-containerlab-primer

# Verify containerlab is installed
containerlab version
```

**Expected Output:**
```
                           _                   _       _
                 _        (_)                 | |     | |
 ____ ___  ____ | |_  ____ _ ____   ____  ____| | ____| | _
/ ___) _ \|  _ \|  _)/ _  | |  _ \ / _  )/ ___) |/ _  | || \
( (__| |_|| | | | |_( ( | | | | | ( (/ /| |   | ( ( | | |_) )
\____)___/|_| |_|\___)_||_|_|_| |_|\____)_|   |_|\_||_|____/

    version: 0.60.1
```

> "Containerlab is ready. Now we need a network operating system image."

```bash
# Pull Nokia SR Linux (free, no registration required)
docker pull ghcr.io/nokia/srlinux:24.10.1
```

> "Notice we're using a specific version tag - not 'latest'. This ensures our labs are reproducible."

```bash
# Verify the image
docker images | grep srlinux
```

**Key Points:**
- Specific version tags for reproducibility
- SR Linux is free - no license needed for learning
- Image is about 2GB

---

### Section 3: Understanding Topology Files (3 minutes)

> **[VOICEOVER]**
>
> "Containerlab uses YAML files to define network topologies. Let's look at our first topology."

```bash
# View the topology file
cat topology/lab.clab.yml
```

**Expected Output:**
```yaml
name: first-lab

topology:
  nodes:
    srl1:
      kind: srl
      image: ghcr.io/nokia/srlinux:24.10.1

    srl2:
      kind: srl
      image: ghcr.io/nokia/srlinux:24.10.1

  links:
    - endpoints: ["srl1:e1-1", "srl2:e1-1"]
```

> "Let me break this down:
>
> The **name** field identifies your lab. It's used in container names, so keep it short and meaningful.
>
> Under **topology.nodes**, we define our network devices. Each node has:
> - A name - here 'srl1' and 'srl2'
> - A **kind** - this tells containerlab what type of device it is. 'srl' means SR Linux
> - An **image** - the Docker image to use
>
> Under **topology.links**, we define connections between nodes. The format is:
> `endpoints: ["node1:interface", "node2:interface"]`
>
> For SR Linux, 'e1-1' means ethernet-1/1. This connects port 1 on srl1 to port 1 on srl2."

**Visual:** Draw simple diagram or show Mermaid diagram

```
srl1 [e1-1] -------- [e1-1] srl2
```

**Key Points:**
- YAML format, easy to version control
- Nodes define devices
- Links define connections
- Interface naming varies by device type

---

### Section 4: Deploying Your First Lab (3 minutes)

> **[VOICEOVER]**
>
> "Let's deploy this topology."

```bash
containerlab deploy -t topology/lab.clab.yml
```

> "Containerlab is now:
> 1. Creating a Docker network for management connectivity
> 2. Starting containers for each node
> 3. Creating virtual ethernet links between the nodes
> 4. Waiting for the devices to boot"

**Expected Output:**
```
INFO[0000] Containerlab v0.60.1 started
INFO[0000] Parsing & checking topology file: lab.clab.yml
INFO[0000] Creating lab directory: /home/user/clab-first-lab
INFO[0000] Creating container: "srl1"
INFO[0000] Creating container: "srl2"
INFO[0001] Creating virtual wire: srl1:e1-1 <--> srl2:e1-1
INFO[0001] Running postdeploy actions for Nokia SR Linux 'srl1' node
INFO[0001] Running postdeploy actions for Nokia SR Linux 'srl2' node
+---+---------------------+--------------+-----------------------+------+---------+
| # |        Name         | Container ID |         Image         | Kind |  State  |
+---+---------------------+--------------+-----------------------+------+---------+
| 1 | clab-first-lab-srl1 | abc123def456 | ghcr.io/nokia/srlinux | srl  | running |
| 2 | clab-first-lab-srl2 | 789ghi012jkl | ghcr.io/nokia/srlinux | srl  | running |
+---+---------------------+--------------+-----------------------+------+---------+
```

> "Both nodes are running. Notice the naming convention: clab-[lab-name]-[node-name].
>
> Let's verify with a couple of commands."

```bash
# Containerlab's inspect command
containerlab inspect -t topology/lab.clab.yml

# Or check Docker directly
docker ps --filter "name=clab"
```

---

### Section 5: Connecting to Devices (3 minutes)

> **[VOICEOVER]**
>
> "Now let's connect to our network devices. SR Linux has its own CLI that we access through docker exec."

```bash
# Connect to srl1
docker exec -it clab-first-lab-srl1 sr_cli
```

> "We're now in the SR Linux CLI. This is a production-grade network operating system - the same software that runs in real Nokia routers."

**Inside SR Linux:**
```
A:srl1# show version
```

> "This shows us the software version and system information."

```
A:srl1# show interface brief
```

> "Here we can see all our interfaces. Notice ethernet-1/1 - that's the interface connected to srl2. It shows as 'up' because the link is established."

```
A:srl1# show system information
```

> "And system information shows hostname, uptime, and resource usage."

```
A:srl1# exit
```

> "Type 'exit' to leave the CLI and return to your shell."

**Key Points:**
- `docker exec -it <container> sr_cli` to enter SR Linux
- Interface naming: e1-1 = ethernet-1/1
- Full production CLI available

---

### Section 6: Topology Visualization (1 minute)

> **[VOICEOVER]**
>
> "Containerlab can generate a visual diagram of your topology."

```bash
# Generate graph
containerlab graph -t topology/lab.clab.yml
```

> "If you have a graphical environment, this opens in your browser. In a headless environment, you can save it to a file."

```bash
# Save to PNG (if graphviz available)
containerlab graph -t topology/lab.clab.yml -o topology.png
```

> "This is helpful for documentation and understanding complex topologies."

---

### Section 7: Cleanup (1 minute)

> **[VOICEOVER]**
>
> "When you're done, destroy the lab to free up resources."

```bash
# Destroy the lab
containerlab destroy -t topology/lab.clab.yml
```

> "This stops and removes the containers. Add the --cleanup flag to also remove configuration files that containerlab creates."

```bash
# Destroy with cleanup
containerlab destroy -t topology/lab.clab.yml --cleanup
```

```bash
# Verify everything is gone
docker ps | grep clab
```

> "No containers - we're clean."

---

### Recap (30 seconds)

> **[VOICEOVER]**
>
> "Let's recap what we covered:
>
> - Containerlab runs network operating systems in Docker containers
> - Topology files are YAML - nodes define devices, links define connections
> - Deploy with `containerlab deploy`, destroy with `containerlab destroy`
> - Connect to SR Linux with `docker exec -it <container> sr_cli`
> - The documentation at containerlab.dev and the Discord community are your go-to resources"

---

### Closing (30 seconds)

> **[VOICEOVER]**
>
> "Now it's your turn. In the exercises folder, you'll:
> - Add a third node to the topology
> - Create a mixed topology with Linux hosts
> - Explore the documentation and community resources
>
> In the next lesson, we'll configure IP addresses on these devices and establish actual connectivity between them.
>
> Happy labbing!"

**Visual:** Show exercises folder structure

---

## Post-Recording Checklist

- [ ] Lab destroyed: `containerlab destroy --all`
- [ ] Timing verified: ~12-15 minutes
- [ ] All commands worked correctly
- [ ] Transcript updated with actual output

---

## B-Roll / Supplementary Footage Needed

1. Containerlab.dev documentation site scroll-through
2. Discord community screenshot
3. GitHub topics/containerlab search results
4. Mermaid diagram animation of topology
5. SR Linux logo/branding (if permitted)

---

## Notes for Editing

- **0:00-0:30** - Hook, can add network diagram animation
- **2:00** - Consider picture-in-picture for docs walkthrough
- **6:00** - Split screen when showing deploy output
- **8:00** - Highlight interface name mapping (e1-1 = ethernet-1/1)
- **End** - Add call-to-action overlay for exercises
