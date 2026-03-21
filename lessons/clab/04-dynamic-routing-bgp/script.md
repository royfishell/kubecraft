# Lesson 4: Dynamic Routing with BGP - Video Script

## Lesson Information

| Field | Value |
|-------|-------|
| **Lesson Number** | 04 |
| **Title** | Dynamic Routing with BGP |
| **Duration Target** | 12-14 minutes |
| **Prerequisites** | Lessons 0-3, gNMIc installed (`gnmic version`) |
| **Learning Objectives** | Explain why dynamic routing exists, configure eBGP on SR Linux, use gNMIc for model-driven config, observe BGP path selection, diagnose BGP failures |

---

## Pre-Recording Checklist

- [ ] Lab environment tested (containerlab installed, Docker running)
- [ ] gNMIc installed: `gnmic version`
- [ ] Lesson 03 lab destroyed: `containerlab destroy --all`
- [ ] SR Linux image pulled: `docker pull ghcr.io/nokia/srlinux:24.10.1`
- [ ] Alpine image pulled: `docker pull alpine:3.20`
- [ ] Screen resolution set (1920x1080)
- [ ] Terminal font size increased (14-16pt)
- [ ] Notifications disabled
- [ ] Clean terminal: `clear && history -c`
- [ ] No labs running: `containerlab inspect --all`

---

## Script

### Opening Hook (30 seconds)

> **[VOICEOVER - Terminal visible]**
>
> "Last time, we configured 8 static routes across 3 routers using Ansible. It worked -- but remember exercise 6? We disabled a link and connectivity broke permanently. Static routes can't adapt. Today we replace every static route with BGP -- and when a link goes down, the network heals itself."

**Visual:** Terminal showing the broken ping from lesson 03 exercise 6

---

### Section 1: Why Dynamic Routing? (2 minutes)

> **[VOICEOVER]**
>
> "Static routes worked fine for three routers. But let's talk about scale. Three routers needed 8 static routes. Ten routers? Around 90. Fifty routers? Over 2000 manual entries. Every time you add a router, you touch every existing device. It's O(n-squared) manual work.
>
> And it's not just scale -- it's resilience. When a link fails with static routes, traffic blackholes. It stays broken until a human logs in and fixes the config. In lesson 3, we saw this firsthand.
>
> Dynamic routing protocols solve both problems. Routers discover their neighbors, exchange reachability information, and adapt when things change. A link goes down, routes are withdrawn, and traffic reconverges -- automatically, in seconds.
>
> There are several protocols -- OSPF, IS-IS, BGP. We're going straight to BGP because it's the protocol you'll encounter in data center and Kubernetes networking. Calico, MetalLB, spine-leaf fabrics -- all BGP."

**Visual:** Scaling comparison diagram -- static route count at 3, 10, 50 routers

**Key Points:**
- Static routes don't scale: 3 routers = 8 routes, 10 = ~90, 50 = 2000+
- Static routes don't adapt: link fails, traffic blackholes until a human fixes it
- Dynamic protocols: discover neighbors, exchange reachability, adapt to changes
- BGP is the data center and Kubernetes networking protocol

**Transition:** "Let's look at how BGP actually works."

---

### Section 2: BGP Fundamentals (3 minutes)

> **[VOICEOVER]**
>
> "BGP -- Border Gateway Protocol -- is the routing protocol that runs the internet. Every ISP, every cloud provider, every major data center fabric uses it. Here are the core concepts you need.
>
> First, Autonomous Systems. An AS is a network under one administrative domain. In our lab, each router gets its own AS number -- 65001 for srl1, 65002 for srl2, 65003 for srl3. When routers in different ASes peer, that's called eBGP -- external BGP. This is the model used in data center spine-leaf fabrics.
>
> Second, peering is explicit. Unlike OSPF which auto-discovers neighbors on a shared network, BGP requires you to manually configure each neighbor. You tell srl1: 'peer with 10.1.2.2 in AS 65002.' And you tell srl2: 'peer with 10.1.2.1 in AS 65001.' Both sides must agree.
>
> Third, session states. BGP peers go through a state machine: Idle, Connect, OpenSent, OpenConfirm, Established. Established is the goal -- that means routes are flowing. If you see a session stuck in Active or Connect, something is wrong: check the IP addresses, the ASN configuration, and whether the peer is reachable.
>
> And here's the trap that catches everyone on SR Linux: routing policies. SR Linux is secure by default -- it won't accept OR advertise BGP routes unless you explicitly allow it. You need both an import policy and an export policy.
>
> In our lab, we use three policies. First, `import-all` -- this simply accepts all received routes. Second, `export-connected` -- this matches directly connected subnets and advertises them. Third, `export-bgp` -- this matches routes learned via BGP and re-advertises them to other peers.
>
> The interesting part is how these chain together. When you apply multiple export policies to a peer-group, SR Linux evaluates them in order. `export-connected` has a default-action of `next-policy` -- if a route doesn't match the 'local protocol' statement, it gets passed to the next policy instead of being rejected. `export-bgp` then checks if it's a BGP-learned route. This gives you modular, composable policies instead of one giant rule."

**Visual:** BGP state machine diagram, policy chain flow on screen

> "One more concept before we start configuring. When a router learns the same prefix from multiple peers, how does it choose? BGP has a best path algorithm. The key steps in order: highest Local Preference wins, then shortest AS Path, then lowest origin type, then lowest MED, then eBGP over iBGP, and finally lowest router ID as a tiebreaker.
>
> In our lab, the one that matters most is AS Path Length. When srl2 learns a route to host3's subnet from both srl1 and srl3, the direct path through srl3 has one AS hop while the path through srl1 has two. Shorter AS path wins. You'll see this in action in the exercises."

**Key Points:**
- AS = administrative domain, each router gets its own AS (eBGP)
- Peering is explicit -- both sides must be configured
- Session states: Idle -> Connect -> OpenSent -> OpenConfirm -> Established
- SR Linux default-deny: must create import AND export policies
- Policy chaining: `next-policy` default-action passes unmatched routes to the next policy
- Best path algorithm: Local Pref > AS Path Length > Origin > MED > eBGP/iBGP > Router ID

**Transition:** "Now let's talk about how we'll push this config to the routers."

---

### Section 3: Introducing gNMIc (2 minutes)

> **[VOICEOVER]**
>
> "In lessons 2 and 3, Ansible sent CLI text commands via JSON-RPC. That works, but there's a more modern approach. gNMIc speaks directly to SR Linux's native configuration interface via gRPC.
>
> Here's the difference. With Ansible and JSON-RPC, you send HTTP POST requests containing CLI command strings wrapped in JSON. You need Jinja2 templates to generate those strings. With gNMIc, you send structured data -- JSON payloads targeting YANG paths -- over gRPC. No templates needed. The JSON is the config."

| | Ansible (JSON-RPC) | gNMIc (gNMI/gRPC) |
|---|---|---|
| Transport | HTTP POST | gRPC (HTTP/2) |
| What you send | CLI text strings | Structured YANG data |
| Template needed? | Yes (Jinja2) | No (JSON is the config) |
| Streaming | No | Yes (subscribe) |

> "YANG paths are like filesystem paths. To reach the BGP config, the path is `/network-instance[name=default]/protocols/bgp`. To reach a specific neighbor: `/network-instance[name=default]/protocols/bgp/neighbor`. Each path targets a specific element in the device's configuration tree.
>
> **[VOICEOVER]**
>
> "Here's our topology. Same hub-and-spoke from lesson 3 -- srl1 is the hub, srl2 and srl3 are spokes, each with a host behind them. But this time, the startup configs only have interfaces. No static routes. Let's deploy."

```bash
cd lessons/clab/04-dynamic-routing-bgp
containerlab deploy -t topology/lab.clab.yml
```

> Let me show you a quick read operation."

```bash
# Read the system name from srl1
gnmic -a clab-dynamic-routing-bgp-srl1:57400 -u admin -p NokiaSrl1! \
  --skip-verify -e json_ietf \
  get --path /system/name
```

> "That's gNMIc reading structured data. No CLI parsing, no regex, no screen scraping. Just clean JSON in and out."

**Visual:** Terminal, split screen comparing Ansible JSON-RPC vs gNMIc approach

**Transition:** "Let's put this into practice. We'll deploy the lab and replace static routes with BGP."

---

### Section 4: Live Demo -- Static to Dynamic (3 minutes)

**Expected output:** Table showing 6 running containers

> "Six containers are up. Interfaces are configured from the startup configs. Let's verify that cross-subnet connectivity doesn't work yet."

```bash
docker exec clab-dynamic-routing-bgp-host1 ping -c 2 10.1.4.2
```

> "Ping fails. No routes. We're back to lesson 2's problem. Now let's fix it with BGP."

```bash
# Apply BGP config to all three routers
gnmic -a clab-dynamic-routing-bgp-srl1:57400 -u admin -p NokiaSrl1! \
  --skip-verify -e json_ietf set --request-file gnmic/configs/srl1-bgp.json

gnmic -a clab-dynamic-routing-bgp-srl2:57400 -u admin -p NokiaSrl1! \
  --skip-verify -e json_ietf set --request-file gnmic/configs/srl2-bgp.json

gnmic -a clab-dynamic-routing-bgp-srl3:57400 -u admin -p NokiaSrl1! \
  --skip-verify -e json_ietf set --request-file gnmic/configs/srl3-bgp.json
```

> "Three commands, three routers configured. Each payload creates three routing policies -- import-all, export-connected, and export-bgp -- then enables BGP with the peer-group and neighbors. Let's check the sessions."

```bash
docker exec -it clab-dynamic-routing-bgp-srl1 sr_cli -c \
  "show network-instance default protocols bgp neighbor"
```

> "Both neighbors show Established. Routes are flowing. Let's verify the routing table."

```bash
docker exec -it clab-dynamic-routing-bgp-srl1 sr_cli -c \
  "show network-instance default route-table ipv4-unicast summary"
```

> "Look at the protocol column. Instead of 'static', you see 'bgp'. These routes were learned automatically. Now the real test."

```bash
docker exec clab-dynamic-routing-bgp-host1 ping -c 3 10.1.4.2
docker exec clab-dynamic-routing-bgp-host1 ping -c 3 10.1.5.2
docker exec clab-dynamic-routing-bgp-host2 ping -c 3 10.1.5.2
```

> "All three work. host1 to host2, host1 to host3, and host2 to host3 through the hub. Same result as lesson 3 -- but with zero static routes."

**Visual:** Full terminal

---

### Section 5: Live Demo -- New Link and Path Selection (2 minutes)

> **[VOICEOVER]**
>
> "Right now, traffic from host2 to host3 goes through the hub: srl2 to srl1 to srl3. Let's trace it."

```bash
docker exec clab-dynamic-routing-bgp-host2 traceroute 10.1.5.2
```

> "Three hops: 10.1.4.1 is srl2, then 10.1.3.2... actually let me read that more carefully. You see three router hops through the hub. Now, there's a pre-wired cable between srl2 and srl3 that we haven't used. Let's turn it up."

```bash
# Configure interfaces on the new link
gnmic -a clab-dynamic-routing-bgp-srl2:57400 -u admin -p NokiaSrl1! \
  --skip-verify -e json_ietf set --request-file gnmic/configs/srl2-new-link.json

gnmic -a clab-dynamic-routing-bgp-srl3:57400 -u admin -p NokiaSrl1! \
  --skip-verify -e json_ietf set --request-file gnmic/configs/srl3-new-link.json

# Add BGP peering between srl2 and srl3
gnmic -a clab-dynamic-routing-bgp-srl2:57400 -u admin -p NokiaSrl1! \
  --skip-verify -e json_ietf set --request-file gnmic/configs/srl2-bgp-srl3.json

gnmic -a clab-dynamic-routing-bgp-srl3:57400 -u admin -p NokiaSrl1! \
  --skip-verify -e json_ietf set --request-file gnmic/configs/srl3-bgp-srl2.json
```

> "Four commands: two to bring up the interfaces with IP addresses, two to establish the BGP peering. Let's check the session."

```bash
docker exec -it clab-dynamic-routing-bgp-srl2 sr_cli -c \
  "show network-instance default protocols bgp neighbor"
```

> "srl2 now has two BGP neighbors -- srl1 and srl3. The new session is Established. Now let's traceroute again."

```bash
docker exec clab-dynamic-routing-bgp-host2 traceroute 10.1.5.2
```

> "Two hops instead of three. This is the BGP best path algorithm in action. srl2 now has two routes to host3's subnet. The old path through srl1 has an AS path of [65001, 65003] -- two hops. The new direct path has an AS path of [65003] -- one hop. Local Preference and Origin are equal, so step 2 of the algorithm breaks the tie: shortest AS path wins. No manual intervention required."

**Visual:** Terminal with traceroute output, before and after comparison

---

### Recap (30 seconds)

> **[VOICEOVER]**
>
> "Let's recap:
>
> - BGP replaces manual route management with automatic route exchange between routers.
> - SR Linux is default-deny: you need both import and export policies, and you can chain export policies using `next-policy`.
> - gNMIc pushes structured config via gRPC -- no templates, no CLI string parsing.
> - When the network topology changes, BGP's best path algorithm adapts automatically. In our lab, shorter AS path won -- but the algorithm checks Local Preference, Origin, MED, and more before it gets there."

---

### Closing (30 seconds)

> **[VOICEOVER]**
>
> "Head to the exercises folder. You'll configure BGP yourself, light up the redundant link, and break things in ways that heal themselves. You'll also hit the most common SR Linux BGP trap -- the missing export policy -- so you know how to diagnose it when it happens in production.
>
> Next lesson: we scale up to a 2-spine, 4-leaf data center fabric -- the architecture that runs under every major cloud and Kubernetes deployment.
>
> Happy labbing!"

**Visual:** Show exercises folder

---

## Post-Recording Checklist

- [ ] Lab destroyed: `containerlab destroy --all`
- [ ] Timing verified: ~12-14 minutes
- [ ] All commands worked correctly
- [ ] Transcript updated with actual output

---

## B-Roll / Supplementary Footage Needed

1. Hub-and-spoke to triangle evolution diagram (adding the srl2-srl3 link)
2. BGP state machine visualization (Idle through Established)
3. AS path comparison animation (3-hop via hub vs 2-hop direct)
4. Side-by-side Ansible JSON-RPC vs gNMIc gNMI comparison
5. Link failure -> convergence animation (route withdrawal and reconvergence)

---

## Notes for Editing

- **0:00-0:30** - Hook, overlay broken ping from lesson 03
- **0:30-2:30** - Why dynamic routing, use scaling diagram overlay
- **2:30-5:30** - BGP fundamentals, state machine diagram and export policy config on screen
- **5:30-7:30** - gNMIc intro, split screen comparing approaches
- **7:30-10:30** - Live demo: deploy and configure BGP, full terminal
- **10:30-12:30** - Live demo: new link and path selection, traceroute before/after
- **12:30-13:00** - Recap and closing, call-to-action overlay for exercises
