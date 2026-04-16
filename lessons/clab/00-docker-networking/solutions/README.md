# Lesson 0 Solutions

Reference solutions for the Container Networking exercises.

## Exercise 1: Inspect Docker's Network Plumbing

**What is the IP of docker0? How does it relate to the default gateway?**

The `docker0` bridge is typically `172.17.0.1`. Inside each container, `ip route` shows this as the default gateway:

```
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0 scope link src 172.17.0.2
```

The bridge IS the gateway -- packets from containers destined outside the subnet go to `docker0`, and the host routes them.

**How to match a host-side veth to its container-side eth0:**

Start from **inside the container** -- this is the reliable direction. Each container namespace has its own interface index numbering, so `eth0` is always index 2 in every container. That means the host-side `@if2` is the same for all containers and doesn't help you distinguish them.

But the container's `@ifN` references the **host-side** index, which is globally unique:

```bash
# Inside c1
docker exec c1 ip addr
```

```
2: eth0@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 172.17.0.2/16 scope global eth0
```

The `@if13` tells you the host-side peer is interface index 13.

```bash
# On the host
ip link show master docker0
```

```
13: veth91835c2@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    master docker0 state UP ... link-netnsid 0
14: veth6ea59a4@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    master docker0 state UP ... link-netnsid 1
```

Index 13 is `veth91835c2` -- that's c1's veth pair. Both host-side veths show `@if2` because each namespace numbers `eth0` as index 2 independently. The `link-netnsid` (0 vs 1) identifies which namespace each belongs to, but reading `@ifN` from inside the container is the simplest way to trace the pair.

**Default bridge subnet:** Typically `172.17.0.0/16`.

---

## Exercise 2: Build a Container Network from Scratch

### Complete command sequence

```bash
# 1. Create namespaces
sudo ip netns add red
sudo ip netns add blue

# 2. Verify
sudo ip netns list
```

**Expected output:**
```
blue
red
```

```bash
# 3. Create bridge
sudo ip link add br-study type bridge
sudo ip link set br-study up
sudo ip addr add 10.0.0.254/24 dev br-study

# 4. Wire up red namespace
sudo ip link add veth-r type veth peer name veth-r-br
sudo ip link set veth-r netns red
sudo ip link set veth-r-br master br-study
sudo ip link set veth-r-br up

# 5. Wire up blue namespace
sudo ip link add veth-b type veth peer name veth-b-br
sudo ip link set veth-b netns blue
sudo ip link set veth-b-br master br-study
sudo ip link set veth-b-br up

# 6. Configure IPs (but NOT loopback yet)
sudo ip netns exec red ip addr add 10.0.0.1/24 dev veth-r
sudo ip netns exec red ip link set veth-r up

sudo ip netns exec blue ip addr add 10.0.0.2/24 dev veth-b
sudo ip netns exec blue ip link set veth-b up
```

```bash
# 7. Test connectivity between namespaces
sudo ip netns exec red ping -c 3 10.0.0.2
```

**Expected output:**
```
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.040 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.038 ms

--- 10.0.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
```

```bash
sudo ip netns exec red ping -c 2 10.0.0.254
```

**Expected output:**
```
PING 10.0.0.254 (10.0.0.254) 56(84) bytes of data.
64 bytes from 10.0.0.254: icmp_seq=1 ttl=64 time=0.030 ms
64 bytes from 10.0.0.254: icmp_seq=2 ttl=64 time=0.028 ms
```

```bash
# 8. Try pinging loopback (fails -- lo is DOWN)
sudo ip netns exec red ping -c 2 127.0.0.1
```

**Expected output:**
```
ping: connect: Network is unreachable
```

```bash
# 9. Bring up loopback and verify
sudo ip netns exec red ip link set lo up
sudo ip netns exec blue ip link set lo up
sudo ip netns exec red ping -c 2 127.0.0.1
```

**Expected output:**
```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.020 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.018 ms
```

### Answers to questions

**Why did 127.0.0.1 fail before bringing up `lo`, even though pinging the other namespace worked?**

Pinging 10.0.0.2 works because that traffic goes out through `veth-r`, across the bridge, and into the blue namespace -- all of those interfaces are UP. But `127.0.0.1` is handled by the loopback interface (`lo`), which starts DOWN in new namespaces. The kernel won't route to an address on a down interface. Loopback is a separate interface from the bridge-connected veths -- bringing up `veth-r` doesn't help loopback traffic.

**What does `ip link show master br-study` show?**

```
3: veth-r-br: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    master br-study state UP
4: veth-b-br: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    master br-study state UP
```

The two host-side veth interfaces attached to the bridge -- exactly like `ip link show master docker0` shows Docker's veth pairs.

**How is this different from Docker?**

It's the same primitives. Docker automates:
- Namespace creation (one per container)
- Bridge creation (`docker0`)
- veth pair creation and attachment
- IP assignment (via IPAM)
- Route configuration
- iptables rules for NAT

We did all of these steps manually.

---

## Exercise 3: Enable Internet Access (NAT)

### Complete command sequence

```bash
# 1. Find outbound interface
ip route show default
```

**Expected output (varies by system):**
```
default via 192.168.1.1 dev enp6s0 proto dhcp metric 100
```

The interface is `enp6s0` (use whatever your system shows -- on modern Linux this is usually **not** `eth0`).

```bash
# 2. Add default routes
sudo ip netns exec red ip route add default via 10.0.0.254
sudo ip netns exec blue ip route add default via 10.0.0.254

# 3. Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
```

**Expected output:**
```
net.ipv4.ip_forward = 1
```

```bash
# 4. Allow forwarding for br-study
sudo iptables -A FORWARD -i br-study -j ACCEPT
sudo iptables -A FORWARD -o br-study -j ACCEPT
```

```bash
# 5. Add masquerade rule (replace enp6s0 with your interface)
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o enp6s0 -j MASQUERADE
```

```bash
# 6. Test internet access
sudo ip netns exec red ping -c 3 8.8.8.8
```

**Expected output:**
```
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=112 time=20.5 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=112 time=17.8 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=112 time=16.3 ms
```

```bash
# 7. Compare with Docker's rules
sudo iptables -t nat -L POSTROUTING -v
```

**Expected output (relevant lines):**
```
Chain POSTROUTING (policy ACCEPT)
 pkts bytes target     prot opt in     out     source               destination
    3   252 MASQUERADE  all  --  any    enp6s0  10.0.0.0/24          anywhere
    0     0 MASQUERADE  all  --  any    !docker0  172.17.0.0/16        anywhere
```

Docker's rule looks almost identical to ours -- masquerade traffic from its subnet going out any interface except docker0.

### Answers to questions

**What does the masquerade rule do?**

It rewrites the source IP address of outbound packets from 10.0.0.x to the host's IP on the outbound interface. When the reply comes back, the kernel automatically translates the destination IP back to the original namespace IP. This is Source NAT (SNAT) -- the same thing your home router does.

**Why do we need IP forwarding?**

Without `net.ipv4.ip_forward=1`, the Linux kernel drops any packet that arrives on one interface but is destined for another. It only processes packets addressed to its own IPs. IP forwarding tells the kernel to act as a router and forward packets between interfaces.

**Why does Docker's FORWARD policy block our traffic?**

Docker sets the FORWARD chain policy to DROP for security -- it only allows forwarding for traffic on its own networks (docker0 and any custom Docker networks). You can see this with:

```bash
sudo iptables -L FORWARD -v
```

```
Chain FORWARD (policy DROP)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DOCKER-USER  all  --  any    any     anywhere             anywhere
    0     0 DOCKER-FORWARD  all  --  any    any     anywhere             anywhere
```

Our br-study traffic doesn't match any of Docker's FORWARD rules, so it hits the DROP policy. Adding explicit ACCEPT rules for br-study tells the kernel to allow forwarding for our bridge. Docker does this automatically for its own bridges -- we have to do it manually for ours.

**Docker's masquerade rule:**

Docker creates a rule like:
```
MASQUERADE  all  --  anywhere  anywhere  source: 172.17.0.0/16  ! out: docker0
```

This masquerades traffic from containers (172.17.0.0/16) going out any interface except docker0 itself. Traffic between containers stays on the bridge and doesn't need NAT.

---

## Exercise 4: Break/Fix -- Bridge Down

**Diagnosis:**

```bash
ip link show br-study
```

```
5: br-study: <BROADCAST,MULTICAST> mtu 1500 ...
    link/ether 42:65:a1:xx:xx:xx brd ff:ff:ff:ff:ff:ff
```

Notice the flags: `<BROADCAST,MULTICAST>` without `UP`. When the bridge is up, it shows `<BROADCAST,MULTICAST,UP,LOWER_UP>`. A down bridge cannot forward frames between its ports -- it's like unplugging a switch.

The veth pairs are still attached (`ip link show master br-study` still lists them), but traffic can't flow through the bridge.

**Fix:**

```bash
sudo ip link set br-study up
```

**Verification:**

```bash
sudo ip netns exec red ping -c 2 10.0.0.2
```

**Key lesson:** A Linux bridge must be UP to forward traffic. Even if all veth pairs are correctly attached and all interfaces inside namespaces have IPs, a downed bridge stops all communication. This is the virtual equivalent of a switch losing power.

---

## Exercise 5: Break/Fix -- Missing Masquerade

**Diagnosis:**

```bash
sudo iptables -t nat -L POSTROUTING -v -n
```

```
Chain POSTROUTING (policy ACCEPT)
 pkts bytes target     prot opt in     out     source               destination
    0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0
```

Only Docker's masquerade rule is present. Our rule for 10.0.0.0/24 is gone.

**Why local works but internet doesn't:**

Local traffic (red to blue, red to bridge) stays on the br-study bridge. It never leaves the host, so it never needs NAT. The bridge forwards frames based on MAC addresses -- this is pure Layer 2 switching.

Internet traffic must leave the host via the outbound interface (e.g., enp6s0). Without masquerade, the source IP is 10.0.0.1 -- a private, non-routable address. The ISP's routers don't know how to send replies back to 10.0.0.1, so the traffic is silently dropped. Masquerade rewrites the source IP to the host's public IP, and the kernel tracks the mapping so return traffic gets translated back.

**Fix:**

```bash
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o enp6s0 -j MASQUERADE
```

**Verification:**

```bash
sudo ip netns exec red ping -c 3 8.8.8.8
```

**Key lesson:** Masquerade (SNAT) is only needed when traffic crosses a network boundary -- from private addresses to the internet. Local bridge traffic doesn't need NAT because both endpoints are directly reachable at Layer 2.

---

## Extreme Challenge 1: Selective Namespace Isolation

### Overview

Three namespaces (red, blue, green), each on its own bridge and subnet. A default DROP policy on the FORWARD chain ensures nothing is allowed unless explicitly permitted. Stateful rules (`conntrack`) handle return traffic so that only the initiator direction matters.

### Network Layout

| Namespace | Bridge    | Namespace IP | Bridge IP    | Subnet       |
|-----------|-----------|-------------|--------------|--------------|
| red       | br-red    | 10.1.0.1    | 10.1.0.254   | 10.1.0.0/24  |
| blue      | br-blue   | 10.2.0.1    | 10.2.0.254   | 10.2.0.0/24  |
| green     | br-green  | 10.3.0.1    | 10.3.0.254   | 10.3.0.0/24  |

### Step 1: Create namespaces

```bash
sudo ip netns add red
sudo ip netns add blue
sudo ip netns add green
```

### Step 2: Create bridges

```bash
sudo ip link add br-red type bridge
sudo ip link set br-red up
sudo ip addr add 10.1.0.254/24 dev br-red

sudo ip link add br-blue type bridge
sudo ip link set br-blue up
sudo ip addr add 10.2.0.254/24 dev br-blue

sudo ip link add br-green type bridge
sudo ip link set br-green up
sudo ip addr add 10.3.0.254/24 dev br-green
```

### Step 3: Wire up each namespace with veth pairs

```bash
# Red namespace
sudo ip link add veth-r type veth peer name veth-r-br
sudo ip link set veth-r netns red
sudo ip link set veth-r-br master br-red
sudo ip link set veth-r-br up
sudo ip netns exec red ip addr add 10.1.0.1/24 dev veth-r
sudo ip netns exec red ip link set veth-r up
sudo ip netns exec red ip link set lo up

# Blue namespace
sudo ip link add veth-b type veth peer name veth-b-br
sudo ip link set veth-b netns blue
sudo ip link set veth-b-br master br-blue
sudo ip link set veth-b-br up
sudo ip netns exec blue ip addr add 10.2.0.1/24 dev veth-b
sudo ip netns exec blue ip link set veth-b up
sudo ip netns exec blue ip link set lo up

# Green namespace
sudo ip link add veth-g type veth peer name veth-g-br
sudo ip link set veth-g netns green
sudo ip link set veth-g-br master br-green
sudo ip link set veth-g-br up
sudo ip netns exec green ip addr add 10.3.0.1/24 dev veth-g
sudo ip netns exec green ip link set veth-g up
sudo ip netns exec green ip link set lo up
```

### Step 4: Add default routes in each namespace

Each namespace needs to route non-local traffic through its bridge IP:

```bash
sudo ip netns exec red ip route add default via 10.1.0.254
sudo ip netns exec blue ip route add default via 10.2.0.254
sudo ip netns exec green ip route add default via 10.3.0.254
```

### Step 5: Enable IP forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

### Step 6: Configure iptables FORWARD rules

The strategy is:

1. Set the default FORWARD policy to DROP (block everything by default).
2. Allow return traffic for established connections using conntrack.
3. Explicitly allow only red-to-blue and blue-to-green as new connections.
4. Green-to-red is never explicitly allowed, so it hits the DROP policy.

**Important:** Docker may have its own FORWARD rules. We insert our rules at the top of the chain so they are evaluated first.

```bash
# Allow return traffic for any connection that was already permitted
sudo iptables -I FORWARD 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow red (br-red) to reach blue (br-blue)
sudo iptables -I FORWARD 2 -i br-red -o br-blue -j ACCEPT

# Allow blue (br-blue) to reach green (br-green)
sudo iptables -I FORWARD 3 -i br-blue -o br-green -j ACCEPT

# Set default policy to DROP (block everything not explicitly allowed)
sudo iptables -P FORWARD DROP
```

**Why this works:**

- When red pings blue, the packet enters via `br-red` and exits via `br-blue`. Rule 2 matches and accepts it. The reply from blue to red is an ESTABLISHED connection, so rule 1 matches and accepts the return.
- When blue pings green, the packet enters via `br-blue` and exits via `br-green`. Rule 3 matches. The reply is ESTABLISHED, so rule 1 handles it.
- When green tries to ping red, the packet enters via `br-green` and exits via `br-red`. No rule matches this combination. It falls through to the DROP policy and is silently discarded. There is no ESTABLISHED state for this flow because green never successfully initiated a connection to red.

### Step 7: Verify the connectivity matrix

```bash
# Red to blue -- should succeed
sudo ip netns exec red ping -c 3 10.2.0.1
```

**Expected output:**
```
PING 10.2.0.1 (10.2.0.1) 56(84) bytes of data.
64 bytes from 10.2.0.1: icmp_seq=1 ttl=63 time=0.060 ms
64 bytes from 10.2.0.1: icmp_seq=2 ttl=63 time=0.045 ms
64 bytes from 10.2.0.1: icmp_seq=3 ttl=63 time=0.042 ms
```

```bash
# Blue to green -- should succeed
sudo ip netns exec blue ping -c 3 10.3.0.1
```

**Expected output:**
```
PING 10.3.0.1 (10.3.0.1) 56(84) bytes of data.
64 bytes from 10.3.0.1: icmp_seq=1 ttl=63 time=0.055 ms
64 bytes from 10.3.0.1: icmp_seq=2 ttl=63 time=0.043 ms
64 bytes from 10.3.0.1: icmp_seq=3 ttl=63 time=0.041 ms
```

```bash
# Green to red -- should FAIL
sudo ip netns exec green ping -c 3 -W 2 10.1.0.1
```

**Expected output:**
```
PING 10.1.0.1 (10.1.0.1) 56(84) bytes of data.

--- 10.1.0.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2003ms
```

### Step 8: Inspect the rules

```bash
sudo iptables -L FORWARD -v -n --line-numbers
```

**Expected output (relevant lines):**
```
Chain FORWARD (policy DROP)
num   pkts bytes target     prot opt in     out     source               destination
1        6   504 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0   ctstate RELATED,ESTABLISHED
2        3   252 ACCEPT     all  --  br-red br-blue 0.0.0.0/0            0.0.0.0/0
3        3   252 ACCEPT     all  --  br-blue br-green 0.0.0.0/0          0.0.0.0/0
```

The conntrack rule (line 1) shows hits from the return traffic of allowed flows. The DROP policy catches everything else, including green-to-red.

### Cleanup

```bash
sudo ip netns del red
sudo ip netns del blue
sudo ip netns del green
sudo ip link del br-red
sudo ip link del br-blue
sudo ip link del br-green
sudo iptables -D FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -D FORWARD -i br-red -o br-blue -j ACCEPT
sudo iptables -D FORWARD -i br-blue -o br-green -j ACCEPT
sudo iptables -P FORWARD ACCEPT
```

**Note:** The last command resets the FORWARD policy back to ACCEPT. If Docker is running, Docker will set it back to DROP on its own when it next restarts or creates a network. Check with `sudo iptables -L FORWARD` after cleanup.

---

## Extreme Challenge 2: DIY Port Forwarding

### Overview

This exercise replicates what Docker does when you run `docker run -p 9090:8080 ...`. We run a service inside a network namespace on port 8080 and use iptables DNAT rules to make it reachable from the host on port 9090.

### Prerequisites

You need a namespace connected to a bridge with IP forwarding enabled. You can reuse the setup from Exercise 2 and 3 (red namespace at 10.0.0.1 on br-study), or create a fresh one. The commands below assume the Exercise 2/3 setup is in place.

### Step 1: Verify the namespace and connectivity

```bash
# Confirm the red namespace exists and has an IP
sudo ip netns exec red ip addr show veth-r
```

You should see `10.0.0.1/24` on `veth-r`. If not, rebuild the Exercise 2 setup first.

### Step 2: Start a service inside the namespace

Open a separate terminal and start a simple HTTP server:

```bash
sudo ip netns exec red python3 -m http.server 8080
```

This starts a Python HTTP server listening on port 8080 inside the red namespace. Leave this running.

If `python3` is not available, use netcat instead:

```bash
sudo ip netns exec red sh -c 'while true; do echo -e "HTTP/1.1 200 OK\r\n\r\nHello from namespace" | nc -l -p 8080 -q 1; done'
```

### Step 3: Verify the service works from inside the namespace

In another terminal:

```bash
sudo ip netns exec red curl -s http://127.0.0.1:8080
```

You should see a directory listing (Python) or "Hello from namespace" (netcat). This confirms the service is running.

### Step 4: Ensure IP forwarding and FORWARD rules are in place

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -A FORWARD -i br-study -j ACCEPT
sudo iptables -A FORWARD -o br-study -j ACCEPT
```

If you already have these from Exercise 3, these commands will add duplicates (harmless but unnecessary). You can check first with `sudo iptables -L FORWARD -v`.

### Step 5: Add DNAT rules for port forwarding

Two rules are needed because Linux processes incoming traffic and locally-originated traffic through different iptables chains:

```bash
# For traffic arriving from external sources (PREROUTING chain)
sudo iptables -t nat -A PREROUTING -p tcp --dport 9090 -j DNAT --to-destination 10.0.0.1:8080

# For traffic originating on the host itself, e.g., curl localhost (OUTPUT chain)
sudo iptables -t nat -A OUTPUT -p tcp --dport 9090 -d 127.0.0.1 -j DNAT --to-destination 10.0.0.1:8080
```

**Why two rules?**

- **PREROUTING** handles packets arriving from the network (another machine curling your host's IP). These packets enter the network stack from an interface and hit PREROUTING before any routing decision.
- **OUTPUT** handles packets generated locally on the host. When you run `curl localhost:9090`, the packet is locally generated -- it never traverses PREROUTING. The OUTPUT chain in the nat table is where DNAT must happen for local traffic.

Docker adds both of these rules when you use `-p`.

### Step 6: Add masquerade for return traffic

The namespace needs to know where to send reply packets. With DNAT, the destination gets rewritten, but the source is the host's loopback (127.0.0.1) or external IP. The namespace might not have a route back to those addresses through the bridge. Masquerading on the bridge ensures return traffic flows correctly:

```bash
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o br-study -j MASQUERADE
```

If you already have the masquerade rule from Exercise 3 for your outbound interface, you may also need this bridge-specific one for traffic that enters the namespace via the bridge and needs to return the same way.

### Step 7: Test from the host

```bash
curl -s http://localhost:9090
```

**Expected output (Python HTTP server):**
```html
<!DOCTYPE HTML>
<html lang="en">
<head>
...
<title>Directory listing for /</title>
...
```

**Expected output (netcat):**
```
Hello from namespace
```

The request to `localhost:9090` was rewritten by iptables to `10.0.0.1:8080`, forwarded through the bridge into the namespace, and the response came back through the same path. This is exactly what Docker does with `-p 9090:8080`.

### Step 8: Inspect the NAT rules

```bash
sudo iptables -t nat -L -v -n --line-numbers
```

**Relevant output:**
```
Chain PREROUTING (policy ACCEPT)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0    tcp dpt:9090 to:10.0.0.1:8080

Chain OUTPUT (policy ACCEPT)
num   pkts bytes target     prot opt in     out     source               destination
1        1    60 DNAT       tcp  --  *      *       0.0.0.0/0            127.0.0.1    tcp dpt:9090 to:10.0.0.1:8080

Chain POSTROUTING (policy ACCEPT)
num   pkts bytes target     prot opt in     out     source               destination
1        1    60 MASQUERADE all  --  *      br-study 10.0.0.0/24         0.0.0.0/0
```

Compare this with Docker's NAT rules (`sudo iptables -t nat -L -v -n`) -- you will see Docker creates nearly identical DNAT entries in PREROUTING and OUTPUT for every `-p` mapping.

### How Docker does it

When you run `docker run -p 9090:8080 myimage`, Docker:

1. Creates a network namespace (the container).
2. Creates a veth pair connecting the namespace to `docker0`.
3. Assigns an IP to the container (e.g., 172.17.0.2).
4. Adds a DNAT rule in PREROUTING: `-p tcp --dport 9090 -j DNAT --to-destination 172.17.0.2:8080`.
5. Adds a DNAT rule in OUTPUT for localhost access.
6. Adds MASQUERADE and FORWARD rules as needed.

You just did all of this manually. There is no magic -- it is bridges, veth pairs, and iptables NAT rules all the way down.

### Cleanup

Stop the Python HTTP server (Ctrl+C in the terminal where it is running), then remove the iptables rules:

```bash
sudo iptables -t nat -D PREROUTING -p tcp --dport 9090 -j DNAT --to-destination 10.0.0.1:8080
sudo iptables -t nat -D OUTPUT -p tcp --dport 9090 -d 127.0.0.1 -j DNAT --to-destination 10.0.0.1:8080
sudo iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -o br-study -j MASQUERADE
```

If you are done with all exercises, also clean up the namespaces and bridge from Exercise 2.

---

## Key Takeaways

1. **Network namespaces** are what isolate container networking -- each container gets its own
2. **veth pairs** are virtual cables connecting namespaces to bridges
3. **Linux bridges** are virtual switches that connect multiple veth pairs
4. **NAT/masquerade** rewrites source IPs so namespace traffic can reach the internet
5. Docker automates all of this -- understanding the primitives helps you debug and build beyond Docker
