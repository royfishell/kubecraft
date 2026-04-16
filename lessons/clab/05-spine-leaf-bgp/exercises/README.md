# Lesson 5 Exercises: Spine-Leaf BGP Fabric

Complete these exercises to understand how Clos spine-leaf fabrics provide scalable, resilient data center connectivity using eBGP.

## Exercise 1: Deploy and Configure the Fabric

**Objective:** Deploy the Clos fabric, configure eBGP on all devices, and verify end-to-end connectivity.

### Steps

1. Deploy the topology:
   ```bash
   cd lessons/clab/05-spine-leaf-bgp
   containerlab deploy -t topology/lab.clab.yml
   ```

2. Apply BGP config to all 6 routers using gNMIc (run from the `gnmic/` directory so `.gnmic.yml` provides credentials):
   ```bash
   cd gnmic
   gnmic -a clab-spine-leaf-bgp-spine1:57400 set --request-file configs/spine1-bgp.json
   gnmic -a clab-spine-leaf-bgp-spine2:57400 set --request-file configs/spine2-bgp.json
   gnmic -a clab-spine-leaf-bgp-leaf1:57400 set --request-file configs/leaf1-bgp.json
   gnmic -a clab-spine-leaf-bgp-leaf2:57400 set --request-file configs/leaf2-bgp.json
   gnmic -a clab-spine-leaf-bgp-leaf3:57400 set --request-file configs/leaf3-bgp.json
   gnmic -a clab-spine-leaf-bgp-leaf4:57400 set --request-file configs/leaf4-bgp.json
   ```

3. Before verifying, examine what the config files applied. Open `gnmic/configs/leaf1-bgp.json` and identify:
   - **Routing policies:** `import-all`, `export-connected`, and `export-bgp` -- why does a default-deny NOS like SR Linux need all three?
   - **Prefix-set filter:** `host-subnets` on `export-connected` -- what does `10.20.0.0/16 mask-length-range 24..24` match, and what does it exclude?
   - **Multipath:** `maximum-paths: 2` under `afi-safi ipv4-unicast` -- what is the SR Linux default, and why does ECMP require changing it?
   - **Shared spine ASN:** Why do both spine neighbors have the same peer-as (65000)? What does RFC 7938 say about spine ASN assignment?

4. Verify all 8 BGP sessions are established:
   ```bash
   docker exec -it clab-spine-leaf-bgp-leaf1 sr_cli -c "show network-instance default protocols bgp neighbor"
   ```
   Check all 6 devices -- each leaf should have 2 sessions (one per spine), each spine should have 4 sessions (one per leaf).

5. Verify end-to-end connectivity with cross-leaf pings:
   ```bash
   docker exec clab-spine-leaf-bgp-host1 ping -c 3 10.20.2.2  # host1 -> host2
   docker exec clab-spine-leaf-bgp-host1 ping -c 3 10.20.3.2  # host1 -> host3
   docker exec clab-spine-leaf-bgp-host1 ping -c 3 10.20.4.2  # host1 -> host4
   docker exec clab-spine-leaf-bgp-host2 ping -c 3 10.20.4.2  # host2 -> host4
   ```

### Deliverables

- BGP neighbor output from a leaf (showing 2 established sessions)
- BGP neighbor output from a spine (showing 4 established sessions)

---

## Exercise 2: Read the Fabric Routing Table -- Observe ECMP

**Objective:** Understand equal-cost multipath (ECMP) in a spine-leaf fabric and the configuration required to enable it.

### Steps

1. Verify that BGP multipath is configured on leaf1:
   ```bash
   docker exec -it clab-spine-leaf-bgp-leaf1 sr_cli -c "info network-instance default protocols bgp afi-safi ipv4-unicast multipath"
   ```
   You should see `maximum-paths 2`. SR Linux defaults to `maximum-paths 1` (single best path only). Without this setting, BGP picks one spine and ignores the other -- no ECMP.

2. Check leaf1's routing table:
   ```bash
   docker exec -it clab-spine-leaf-bgp-leaf1 sr_cli -c "show network-instance default route-table ipv4-unicast summary"
   ```

3. Look for routes to other leaves' host subnets (10.20.2.0/24, 10.20.3.0/24, 10.20.4.0/24). Each should show 2 next-hops (ECMP) -- one via each spine. The summary at the bottom should show `IPv4 prefixes with active ECMP routes: 3`.

4. Verify that only host subnets appear in BGP -- no /31 fabric link prefixes. The `host-subnets` prefix-set on `export-connected` filters these out. Compare leaf1's BGP routes to its local routes to confirm.

5. Traceroute from host1 to host4 -- run it multiple times:
   ```bash
   docker exec clab-spine-leaf-bgp-host1 traceroute -n -w 2 10.20.4.2
   docker exec clab-spine-leaf-bgp-host1 traceroute -n -w 2 10.20.4.2
   docker exec clab-spine-leaf-bgp-host1 traceroute -n -w 2 10.20.4.2
   ```
   The path is always 4 hops: host -> leaf1 -> spine -> leaf4 -> host4. But which spine may vary between runs.

6. Answer these questions:
   - How many hops between any two hosts? (Always 4: host-leaf-spine-leaf-host)
   - How many equal-cost paths exist between any two leaves? (2, one per spine)
   - What happens to bandwidth if you add a third spine? (50% more aggregate bandwidth, 3 ECMP paths)

### Deliverables

- Routing table showing ECMP entries (2 next-hops per remote leaf subnet)
- Traceroute output from multiple runs
- Answers to the 3 questions above

---

## Exercise 3: Break/Fix -- Spine Failure (Fabric Resilience)

**Objective:** Observe that a spine-leaf fabric survives spine failures with reduced capacity but no connectivity loss.

### Setup (break it)

```bash
docker exec -it clab-spine-leaf-bgp-spine1 sr_cli
```

Inside SR Linux -- disable all interfaces:
```
enter candidate
set / interface ethernet-1/1 admin-state disable
set / interface ethernet-1/2 admin-state disable
set / interface ethernet-1/3 admin-state disable
set / interface ethernet-1/4 admin-state disable
commit now
exit
```

### Symptom

```bash
# Brief connectivity disruption, then full recovery through spine2
docker exec clab-spine-leaf-bgp-host1 ping -c 10 -i 1 10.20.4.2
```

### Your Task

1. Run continuous pings between hosts on different leaves:
   ```bash
   docker exec clab-spine-leaf-bgp-host1 ping -c 30 -i 1 10.20.4.2
   ```

2. Check leaf1's routing table -- ECMP entries should reduce from 2 paths to 1:
   ```bash
   docker exec -it clab-spine-leaf-bgp-leaf1 sr_cli -c "show network-instance default route-table ipv4-unicast summary"
   ```

3. Check BGP neighbors on leaf1 -- the spine1 session should be down:
   ```bash
   docker exec -it clab-spine-leaf-bgp-leaf1 sr_cli -c "show network-instance default protocols bgp neighbor"
   ```

4. Traceroute -- all traffic now goes through spine2:
   ```bash
   docker exec clab-spine-leaf-bgp-host1 traceroute -n -w 2 10.20.4.2
   ```

5. Fix -- re-enable spine1:
   ```bash
   docker exec -it clab-spine-leaf-bgp-spine1 sr_cli
   ```
   Inside SR Linux:
   ```
   enter candidate
   set / interface ethernet-1/1 admin-state enable
   set / interface ethernet-1/2 admin-state enable
   set / interface ethernet-1/3 admin-state enable
   set / interface ethernet-1/4 admin-state enable
   commit now
   exit
   ```

6. Verify ECMP paths return to 2:
   ```bash
   docker exec -it clab-spine-leaf-bgp-leaf1 sr_cli -c "show network-instance default route-table ipv4-unicast summary"
   ```

### Deliverables

- Ping output showing brief loss then recovery
- Routing table before/after showing ECMP path count change (2 -> 1 -> 2)
- Explanation of fabric resilience: why losing a spine degrades capacity but not connectivity

---

## Extreme Challenge 1: Route Hijack via Longest-Prefix-Match

**Objective:** Demonstrate how a more-specific prefix can hijack traffic in a BGP fabric -- the same mechanism behind real-world BGP hijacking incidents.

**Scenario:** Leaf4 serves host4 on the 10.20.4.0/24 subnet. Your goal: make leaf1 advertise a more-specific prefix that covers host4's address, causing all other leaves to route host4-bound traffic to leaf1 instead of leaf4. The traffic should be blackholed on leaf1 -- packets go in, nothing comes out.

You will need to:
- Create a static route on leaf1 for the hijack prefix with a blackhole next-hop
- Build a routing policy that exports this prefix to the spines via BGP
- Replace leaf1's current export policy with your hijack policy (while still advertising leaf1's own legitimate connected routes)

**Success criteria:**
- Pings from host2 and host3 to host4 (10.20.4.2) fail with 100% packet loss
- Routing tables on leaf2 and leaf3 show the rogue prefix alongside the legitimate /24
- Traceroute from host2 to host4 shows traffic being directed to leaf1 and dying
- After removing your hijack configuration and restoring the original export policy, connectivity to host4 is fully restored

**Hint:** Think about the relationship between /24 and /25 prefix lengths, and review how SR Linux next-hop-groups support blackhole routes. The existing export policy chain (`export-connected`, `export-bgp`) shows the pattern for building policies that match specific routes.

---

## Extreme Challenge 2: Graceful Spine Maintenance

**Objective:** Take spine1 completely out of service for maintenance with zero packet loss during the transition.

**Scenario:** In exercise 3, disabling spine1's interfaces caused brief packet loss during BGP convergence. In production, planned maintenance should never drop traffic. Your goal: use BGP mechanisms to gracefully drain all traffic from spine1 before taking it offline. Every leaf must stop sending traffic through spine1 BEFORE any interfaces go down. Only after verifying that zero traffic flows through spine1 should you disable its interfaces.

**Success criteria:**
- Run continuous pings between hosts on different leaves throughout the entire maintenance window. The ping output must show ZERO packet loss -- not "brief loss then recovery" like exercise 3, but truly zero drops
- After maintenance, re-enable spine1 and verify ECMP paths return to 2 per destination

**Note:** There are multiple valid approaches. Research BGP graceful shutdown (RFC 8326) and consider what knobs SR Linux provides for influencing path selection.

---

## Cleanup

After completing all exercises:

```bash
# Destroy the lab
containerlab destroy -t topology/lab.clab.yml --cleanup

# Verify
docker ps | grep clab
```

## Validation

Run the automated tests:

```bash
pytest tests/ -v
```

## Completion Checklist

- [ ] Exercise 1: Deployed fabric, configured eBGP on all 6 routers, verified cross-leaf connectivity
- [ ] Exercise 2: Read routing tables, observed ECMP paths, ran traceroutes
- [ ] Exercise 3: Simulated spine failure, observed resilience and ECMP path reduction
- [ ] Extreme Challenge 1: Diagnosed and fixed route leak (longest-prefix-match hijack)
- [ ] Extreme Challenge 2: Drained spine1 gracefully with zero packet loss during maintenance

## Next Steps

Commit your exercises to your fork and proceed to Lesson 6 (coming soon).
