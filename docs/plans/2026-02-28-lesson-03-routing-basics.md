# Lesson 03: Routing Basics -- Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build lesson 03 that teaches static routing on a hub-and-spoke topology, extending the Ansible automation from lesson 02 with a routes template.

**Architecture:** Hub-and-spoke topology (srl1 hub, srl2/srl3 spokes, 3 hosts). Ansible playbook extends lesson 02 pattern with a second Jinja2 template for static routes using SR Linux next-hop-groups. Six exercises including four break/fix scenarios covering asymmetric routing, black holes, routing loops, and unreachable next-hops.

**Tech Stack:** containerlab, Nokia SR Linux 24.10.1, Alpine 3.20, Ansible (uri module + JSON-RPC), Jinja2, pytest

**Design doc:** `docs/plans/2026-02-28-lesson-03-routing-basics-design.md`

---

## Task 1: Topology File

**Files:**
- Create: `lessons/clab/03-routing-basics/topology/lab.clab.yml`

**Step 1: Create directory structure**

```bash
mkdir -p lessons/clab/03-routing-basics/{topology,ansible/{templates,host_vars},exercises,solutions,tests}
```

**Step 2: Write topology file**

```yaml
# Lesson 3: Routing Basics -- Hub-and-Spoke Topology
# host1 -- srl1 (hub) -- srl2 -- host2
#              |
#             srl3 -- host3

name: routing-basics

topology:
  nodes:
    srl1:
      kind: srl
      image: ghcr.io/nokia/srlinux:24.10.1
      mgmt-ipv4: 172.20.20.11

    srl2:
      kind: srl
      image: ghcr.io/nokia/srlinux:24.10.1
      mgmt-ipv4: 172.20.20.12

    srl3:
      kind: srl
      image: ghcr.io/nokia/srlinux:24.10.1
      mgmt-ipv4: 172.20.20.13

    host1:
      kind: linux
      image: alpine:3.20
      exec:
        - ip link set eth1 up
        - ip addr add 10.1.1.2/24 dev eth1
        - ip route add default via 10.1.1.1 dev eth1

    host2:
      kind: linux
      image: alpine:3.20
      exec:
        - ip link set eth1 up
        - ip addr add 10.1.4.2/24 dev eth1
        - ip route add default via 10.1.4.1 dev eth1

    host3:
      kind: linux
      image: alpine:3.20
      exec:
        - ip link set eth1 up
        - ip addr add 10.1.5.2/24 dev eth1
        - ip route add default via 10.1.5.1 dev eth1

  links:
    - endpoints: ["host1:eth1", "srl1:e1-1"]
    - endpoints: ["srl1:e1-2", "srl2:e1-1"]
    - endpoints: ["srl1:e1-3", "srl3:e1-1"]
    - endpoints: ["srl2:e1-2", "host2:eth1"]
    - endpoints: ["srl3:e1-2", "host3:eth1"]
```

**Step 3: Commit**

```bash
git add lessons/clab/03-routing-basics/topology/
git commit -m "feat(lesson-03): add hub-and-spoke topology file"
```

---

## Task 2: Ansible Configuration (Interfaces + Routes)

**Files:**
- Create: `lessons/clab/03-routing-basics/ansible/inventory.yml`
- Create: `lessons/clab/03-routing-basics/ansible/host_vars/srl1.yml`
- Create: `lessons/clab/03-routing-basics/ansible/host_vars/srl2.yml`
- Create: `lessons/clab/03-routing-basics/ansible/host_vars/srl3.yml`
- Create: `lessons/clab/03-routing-basics/ansible/templates/srl_interfaces.json.j2`
- Create: `lessons/clab/03-routing-basics/ansible/templates/srl_routes.json.j2`
- Create: `lessons/clab/03-routing-basics/ansible/playbook.yml`

**Step 1: Write inventory**

```yaml
---
all:
  children:
    routers:
      hosts:
        srl1:
          ansible_host: 172.20.20.11
        srl2:
          ansible_host: 172.20.20.12
        srl3:
          ansible_host: 172.20.20.13
```

**Step 2: Write host_vars**

`host_vars/srl1.yml` (hub -- 3 interfaces, 2 static routes to host subnets behind spokes):
```yaml
---
interfaces:
  - name: ethernet-1/1
    ipv4_address: 10.1.1.1/24
    description: Link to host1
  - name: ethernet-1/2
    ipv4_address: 10.1.2.1/24
    description: Link to srl2
  - name: ethernet-1/3
    ipv4_address: 10.1.3.1/24
    description: Link to srl3

static_routes:
  - prefix: 10.1.4.0/24
    next_hop: 10.1.2.2
    description: host2 subnet via srl2
  - prefix: 10.1.5.0/24
    next_hop: 10.1.3.2
    description: host3 subnet via srl3
```

`host_vars/srl2.yml` (spoke -- 2 interfaces, 3 static routes via hub):
```yaml
---
interfaces:
  - name: ethernet-1/1
    ipv4_address: 10.1.2.2/24
    description: Link to srl1
  - name: ethernet-1/2
    ipv4_address: 10.1.4.1/24
    description: Link to host2

static_routes:
  - prefix: 10.1.1.0/24
    next_hop: 10.1.2.1
    description: host1 subnet via hub
  - prefix: 10.1.3.0/24
    next_hop: 10.1.2.1
    description: srl1-srl3 link via hub
  - prefix: 10.1.5.0/24
    next_hop: 10.1.2.1
    description: host3 subnet via hub
```

`host_vars/srl3.yml` (spoke -- 2 interfaces, 3 static routes via hub):
```yaml
---
interfaces:
  - name: ethernet-1/1
    ipv4_address: 10.1.3.2/24
    description: Link to srl1
  - name: ethernet-1/2
    ipv4_address: 10.1.5.1/24
    description: Link to host3

static_routes:
  - prefix: 10.1.1.0/24
    next_hop: 10.1.3.1
    description: host1 subnet via hub
  - prefix: 10.1.2.0/24
    next_hop: 10.1.3.1
    description: srl1-srl2 link via hub
  - prefix: 10.1.4.0/24
    next_hop: 10.1.3.1
    description: host2 subnet via hub
```

**Step 3: Copy interface template from lesson 02**

Copy `lessons/clab/02-ip-fundamentals/ansible/templates/srl_interfaces.json.j2` to `lessons/clab/03-routing-basics/ansible/templates/srl_interfaces.json.j2` (identical content).

**Step 4: Write routes template**

`templates/srl_routes.json.j2`:
```jinja2
{# Generates SR Linux CLI commands for static route configuration.
   Variables come from host_vars/<hostname>.yml:
     static_routes:
       - prefix: 10.1.4.0/24
         next_hop: 10.1.2.2
         description: host2 subnet via srl2

   SR Linux requires a next-hop-group for each static route.
   We create one group per route, named after the prefix with
   dots and slashes replaced (e.g., nhg-10-1-4-0-24).
#}
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "cli",
  "params": {
    "commands": [
      "enter candidate",
{% for route in static_routes %}
{% set nhg_name = "nhg-" + route.prefix | replace(".", "-") | replace("/", "-") %}
      "set / network-instance default next-hop-groups group {{ nhg_name }} admin-state enable",
      "set / network-instance default next-hop-groups group {{ nhg_name }} nexthop 1 ip-address {{ route.next_hop }}",
      "set / network-instance default static-routes route {{ route.prefix }} admin-state enable",
      "set / network-instance default static-routes route {{ route.prefix }} next-hop-group {{ nhg_name }}",
{% endfor %}
      "commit now"
    ]
  }
}
```

**Step 5: Write playbook**

```yaml
---
# Lesson 3: Configure interfaces and static routes on SR Linux routers
#
# This playbook extends lesson 02's interface configuration with static
# routes. It uses two Jinja2 templates to generate SR Linux CLI commands
# and pushes them via the JSON-RPC API.
#
# Usage:
#   cd ansible
#   ansible-playbook -i inventory.yml playbook.yml

- name: Configure SR Linux routers -- interfaces and static routes
  hosts: routers
  gather_facts: false
  connection: local

  tasks:
    - name: Apply interface configuration via JSON-RPC
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}/jsonrpc"
        method: POST
        url_username: admin
        url_password: NokiaSrl1!
        force_basic_auth: true
        body_format: json
        body: "{{ lookup('template', 'templates/srl_interfaces.json.j2') }}"
        status_code: 200
      register: interface_result

    - name: Show interface configuration result
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: Interface configuration applied"
      when: interface_result.status == 200

    - name: Apply static route configuration via JSON-RPC
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}/jsonrpc"
        method: POST
        url_username: admin
        url_password: NokiaSrl1!
        force_basic_auth: true
        body_format: json
        body: "{{ lookup('template', 'templates/srl_routes.json.j2') }}"
        status_code: 200
      register: routes_result
      when: static_routes is defined

    - name: Show route configuration result
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: Static routes applied"
      when: routes_result is defined and routes_result.status == 200

    - name: Verify routing table
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}/jsonrpc"
        method: POST
        url_username: admin
        url_password: NokiaSrl1!
        force_basic_auth: true
        body_format: json
        body:
          jsonrpc: "2.0"
          id: 2
          method: get
          params:
            commands:
              - path: /network-instance[name=default]/route-table
                datastore: state
        status_code: 200
      register: verify_result

    - name: Display routing table
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} route-table: {{ verify_result.json.result | default('check manually') }}"
```

**Step 6: Commit**

```bash
git add lessons/clab/03-routing-basics/ansible/
git commit -m "feat(lesson-03): add ansible config with interfaces and static routes"
```

---

## Task 3: README

**Files:**
- Create: `lessons/clab/03-routing-basics/README.md`

**Content outline:**

Follow lesson 02 README structure exactly:
1. Title and one-line description
2. Objectives (checkboxed)
3. Prerequisites
4. Video Outline with numbered sections:
   - Section 1: Routing Table Fundamentals (3 min) -- what's in a routing table, directly connected vs static, longest prefix match
   - Section 2: Static Routes on SR Linux (2 min) -- next-hop-groups, route syntax, when to use static routes
   - Section 3: Extending Ansible with Routes (3 min) -- new template, new host_vars section, playbook extension
   - Section 4: Live Demo (3 min) -- deploy, configure, verify end-to-end
   - Section 5: Multi-Hop Packet Trace (2 min) -- trace host1->host3 hop by hop, return path matters
   - Recap + Teaser (30 sec) -- "static routes don't scale; lesson 04 introduces BGP"
5. Lab Topology (Mermaid diagram)
6. IP Addressing table (5 subnets)
7. Static Routes table (hub: 2 routes, spokes: 3 routes each)
8. Files in This Lesson tree
9. Key Commands Reference table
10. Exercises link
11. Common Issues section
12. Navigation links
13. Additional Resources

**Step 1: Write README.md**

(Full content -- approximately 250 lines following lesson 02 format)

**Step 2: Commit**

```bash
git add lessons/clab/03-routing-basics/README.md
git commit -m "feat(lesson-03): add lesson README"
```

---

## Task 4: Exercises

**Files:**
- Create: `lessons/clab/03-routing-basics/exercises/README.md`

**Exercise structure (6 exercises):**

### Exercise 1: Deploy, Configure, and Verify End-to-End

- Deploy topology
- Run Ansible playbook (interfaces + routes)
- Verify: host1 -> host2 ping, host1 -> host3 ping, host2 -> host3 ping
- This is the lesson 02 cliffhanger payoff
- **Deliverables:** Which pings work, routing table from srl1

### Exercise 2: Read the Routing Table

- Run `show network-instance default route-table ipv4-unicast summary` on all 3 routers
- Trace packet path from host2 (10.1.4.2) to host3 (10.1.5.2):
  1. host2 sends to default gw 10.1.4.1 (srl2)
  2. srl2 looks up 10.1.5.0/24 -> next-hop 10.1.2.1 (srl1)
  3. srl1 looks up 10.1.5.0/24 -> next-hop 10.1.3.2 (srl3)
  4. srl3 has 10.1.5.0/24 directly connected -> delivers to host3
- **Deliverables:** Routing table screenshots, hop-by-hop trace explanation

### Exercise 3: Break/Fix -- Missing Route (Asymmetric Routing)

**Setup:** Delete srl2's route to 10.1.5.0/24
```
docker exec -it clab-routing-basics-srl2 sr_cli
enter candidate
delete / network-instance default static-routes route 10.1.5.0/24
delete / network-instance default next-hop-groups group nhg-10-1-5-0-24
commit now
```

**Symptom:** host2 cannot ping host3, but host3 CAN ping host2's subnet (10.1.4.2)

**Task:** Diagnose using routing tables on srl2. Explain why one direction works. Fix by re-adding the route.

**Deliverables:** Explanation of asymmetric routing, fix commands

### Exercise 4: Break/Fix -- Wrong Next-Hop (Black Hole)

**Setup:** Change srl1's next-hop for 10.1.5.0/24 to non-existent IP
```
docker exec -it clab-routing-basics-srl1 sr_cli
enter candidate
set / network-instance default next-hop-groups group nhg-10-1-5-0-24 nexthop 1 ip-address 10.1.3.99
commit now
```

**Symptom:** host1 cannot ping host3. Traceroute from host1 shows packets reaching srl1 but going nowhere.

**Task:** Check srl1's routing table -- the route exists but points to a bad next-hop. Use `show arpnd arp-entries` to see srl1 trying to ARP for 10.1.3.99. Fix by correcting the next-hop.

**Deliverables:** Explanation of black hole routing, ARP output showing failed resolution

### Exercise 5: Break/Fix -- Routing Loop

**Setup:** On srl2 AND srl1, create a loop for 10.1.5.0/24:
```
# On srl1: point 10.1.5.0/24 at srl2 instead of srl3
docker exec -it clab-routing-basics-srl1 sr_cli
enter candidate
set / network-instance default next-hop-groups group nhg-10-1-5-0-24 nexthop 1 ip-address 10.1.2.2
commit now
```

Now srl1 sends 10.1.5.0/24 traffic to srl2, and srl2 sends it back to srl1.

**Symptom:** host1 ping to host3 fails. `traceroute` from host1 shows alternating hops between srl1 and srl2 until TTL expires.

**Task:** Run traceroute, identify the loop pattern. Explain TTL. Fix by restoring srl1's next-hop to 10.1.3.2.

**Deliverables:** Traceroute output showing the loop, explanation of TTL

### Exercise 6: Break/Fix -- Unreachable Next-Hop (Link Down)

**Setup:** Admin-disable the srl1-srl3 link:
```
docker exec -it clab-routing-basics-srl1 sr_cli
enter candidate
set / interface ethernet-1/3 admin-state disable
commit now
```

**Symptom:** host1 cannot ping host3. srl1's routing table still shows the route to 10.1.5.0/24 via 10.1.3.2, but the interface is down.

**Task:** Check `show interface brief` on srl1 -- ethernet-1/3 is admin-disabled. Check routing table -- route may still be present but next-hop is unreachable. Fix by re-enabling the interface.

**Deliverables:** Explanation of why a route can exist even when the path is broken

**Step 1: Write exercises/README.md**

(Full content, approximately 300 lines)

**Step 2: Commit**

```bash
git add lessons/clab/03-routing-basics/exercises/
git commit -m "feat(lesson-03): add exercises with 4 break/fix scenarios"
```

---

## Task 5: Solutions

**Files:**
- Create: `lessons/clab/03-routing-basics/solutions/README.md`

**Content:** Full solutions for all 6 exercises following lesson 02 solution format:
- Exercise 1: Expected ping output, routing table output
- Exercise 2: Full hop-by-hop trace with routing table entries at each hop
- Exercise 3: Routing table comparison showing missing route, explanation of why forward path breaks but reverse works, fix commands
- Exercise 4: ARP table showing failed resolution for 10.1.3.99, explanation of black holes, fix
- Exercise 5: Traceroute output showing alternating hops, TTL explanation, fix
- Exercise 6: Interface status output, routing table with unresolvable next-hop, fix
- Key Takeaways section (numbered, bold labels with conceptual explanations)

**Step 1: Write solutions/README.md**

(Full content, approximately 350 lines)

**Step 2: Commit**

```bash
git add lessons/clab/03-routing-basics/solutions/
git commit -m "feat(lesson-03): add exercise solutions"
```

---

## Task 6: Tests

**Files:**
- Create: `lessons/clab/03-routing-basics/tests/test_routing_basics.py`

**Test classes:**

```python
"""
Lesson 3: Routing Basics -- Validation Tests

Run with: pytest tests/test_routing_basics.py -v

These tests verify:
1. Required tools are installed (containerlab, docker, ansible)
2. Topology file is valid (6 nodes, 5 links, no latest tags)
3. Ansible configuration is valid (inventory, playbook, templates, host_vars)
4. Lab deploys successfully (6 containers)
5. End-to-end connectivity works (host1<->host2, host1<->host3, host2<->host3)
"""

import subprocess
import os
import pytest
import time
from pathlib import Path


LESSON_DIR = Path(__file__).parent.parent
TOPOLOGY_FILE = LESSON_DIR / "topology" / "lab.clab.yml"
ANSIBLE_DIR = LESSON_DIR / "ansible"
LAB_NAME = "routing-basics"


def run_command(cmd: str, timeout: int = 120) -> subprocess.CompletedProcess:
    """Run a shell command and return the result."""
    return subprocess.run(
        cmd, shell=True, capture_output=True, text=True, timeout=timeout
    )


class TestEnvironment:
    """Verify the lab environment is correctly set up."""

    def test_containerlab_installed(self):
        """Containerlab CLI should be available."""
        result = run_command("containerlab version")
        assert result.returncode == 0

    def test_docker_available(self):
        """Docker should be running and accessible."""
        result = run_command("docker version")
        assert result.returncode == 0

    def test_ansible_installed(self):
        """Ansible should be available."""
        result = run_command("ansible --version")
        assert result.returncode == 0

    def test_topology_file_exists(self):
        """Topology file should exist."""
        assert TOPOLOGY_FILE.exists(), f"Not found: {TOPOLOGY_FILE}"


class TestTopologyStructure:
    """Verify topology file has correct structure."""

    @pytest.fixture
    def topology(self):
        """Load topology file."""
        import yaml
        with open(TOPOLOGY_FILE) as f:
            return yaml.safe_load(f)

    def test_has_correct_name(self, topology):
        """Topology should be named routing-basics."""
        assert topology["name"] == "routing-basics"

    def test_has_six_nodes(self, topology):
        """Topology should have 6 nodes (3 routers + 3 hosts)."""
        nodes = topology["topology"]["nodes"]
        assert len(nodes) == 6, f"Expected 6 nodes, got {len(nodes)}"

    def test_has_five_links(self, topology):
        """Topology should have 5 links."""
        links = topology["topology"]["links"]
        assert len(links) == 5, f"Expected 5 links, got {len(links)}"

    def test_no_latest_tags(self, topology):
        """No node should use 'latest' image tag."""
        nodes = topology["topology"]["nodes"]
        for name, config in nodes.items():
            image = config.get("image", "")
            assert "latest" not in image, f"Node {name} uses 'latest' tag"

    def test_routers_have_mgmt_ips(self, topology):
        """All routers should have pinned management IPs."""
        nodes = topology["topology"]["nodes"]
        for name in ["srl1", "srl2", "srl3"]:
            assert "mgmt-ipv4" in nodes[name], f"{name} missing mgmt-ipv4"

    def test_hosts_have_exec(self, topology):
        """All hosts should have exec commands for IP config."""
        nodes = topology["topology"]["nodes"]
        for name in ["host1", "host2", "host3"]:
            assert "exec" in nodes[name], f"{name} missing exec"
            exec_cmds = nodes[name]["exec"]
            assert any("ip addr add" in cmd for cmd in exec_cmds), \
                f"{name} missing IP address in exec"
            assert any("ip route add default" in cmd for cmd in exec_cmds), \
                f"{name} missing default route in exec"


class TestAnsibleStructure:
    """Verify Ansible configuration files are valid."""

    def test_inventory_exists(self):
        """Ansible inventory should exist."""
        assert (ANSIBLE_DIR / "inventory.yml").exists()

    def test_playbook_exists(self):
        """Ansible playbook should exist."""
        assert (ANSIBLE_DIR / "playbook.yml").exists()

    def test_interface_template_exists(self):
        """Interface Jinja2 template should exist."""
        assert (ANSIBLE_DIR / "templates" / "srl_interfaces.json.j2").exists()

    def test_routes_template_exists(self):
        """Routes Jinja2 template should exist."""
        assert (ANSIBLE_DIR / "templates" / "srl_routes.json.j2").exists()

    def test_host_vars_exist(self):
        """Host vars should exist for all 3 routers."""
        for name in ["srl1", "srl2", "srl3"]:
            path = ANSIBLE_DIR / "host_vars" / f"{name}.yml"
            assert path.exists(), f"Missing host_vars for {name}"

    def test_host_vars_have_routes(self):
        """All router host_vars should have static_routes defined."""
        import yaml
        for name in ["srl1", "srl2", "srl3"]:
            path = ANSIBLE_DIR / "host_vars" / f"{name}.yml"
            with open(path) as f:
                data = yaml.safe_load(f)
            assert "static_routes" in data, f"{name} missing static_routes"
            assert len(data["static_routes"]) >= 2, \
                f"{name} should have at least 2 static routes"

    def test_inventory_has_three_routers(self):
        """Inventory should list 3 routers."""
        import yaml
        with open(ANSIBLE_DIR / "inventory.yml") as f:
            data = yaml.safe_load(f)
        routers = data["all"]["children"]["routers"]["hosts"]
        assert len(routers) == 3, f"Expected 3 routers, got {len(routers)}"


class TestLabDeployment:
    """Test that the lab deploys successfully."""

    @pytest.fixture(autouse=True)
    def cleanup(self):
        """Ensure lab is destroyed after test."""
        yield
        run_command(
            f"containerlab destroy -t {TOPOLOGY_FILE} --cleanup 2>/dev/null || true"
        )
        time.sleep(2)

    def test_lab_deploys(self):
        """Lab should deploy without errors."""
        run_command(
            f"containerlab destroy -t {TOPOLOGY_FILE} --cleanup 2>/dev/null || true"
        )
        time.sleep(2)
        result = run_command(
            f"containerlab deploy -t {TOPOLOGY_FILE}", timeout=180
        )
        assert result.returncode == 0, f"Deploy failed: {result.stderr}"

    def test_six_containers_running(self):
        """All 6 containers should be running after deploy."""
        run_command(
            f"containerlab destroy -t {TOPOLOGY_FILE} --cleanup 2>/dev/null || true"
        )
        time.sleep(2)
        run_command(f"containerlab deploy -t {TOPOLOGY_FILE}", timeout=180)
        time.sleep(10)

        result = run_command(
            f"docker ps --filter 'name=clab-{LAB_NAME}' --format '{{{{.Names}}}}'"
        )
        containers = [c for c in result.stdout.strip().split("\n") if c]
        assert len(containers) == 6, f"Expected 6 containers, got: {containers}"


class TestConnectivity:
    """Test end-to-end connectivity after Ansible configuration."""

    @pytest.fixture(autouse=True)
    def deploy_and_configure(self):
        """Deploy lab and apply Ansible configuration."""
        run_command(
            f"containerlab destroy -t {TOPOLOGY_FILE} --cleanup 2>/dev/null || true"
        )
        time.sleep(2)
        run_command(f"containerlab deploy -t {TOPOLOGY_FILE}", timeout=180)
        time.sleep(10)
        run_command(
            f"cd {ANSIBLE_DIR} && ansible-playbook -i inventory.yml playbook.yml",
            timeout=120,
        )
        time.sleep(5)
        yield
        run_command(
            f"containerlab destroy -t {TOPOLOGY_FILE} --cleanup 2>/dev/null || true"
        )

    def test_host1_to_host2(self):
        """host1 should be able to ping host2 (cross-hub)."""
        result = run_command(
            f"docker exec clab-{LAB_NAME}-host1 ping -c 3 -W 5 10.1.4.2"
        )
        assert result.returncode == 0, \
            f"host1 -> host2 failed: {result.stdout}"

    def test_host1_to_host3(self):
        """host1 should be able to ping host3 (cross-hub)."""
        result = run_command(
            f"docker exec clab-{LAB_NAME}-host1 ping -c 3 -W 5 10.1.5.2"
        )
        assert result.returncode == 0, \
            f"host1 -> host3 failed: {result.stdout}"

    def test_host2_to_host3(self):
        """host2 should be able to ping host3 (spoke-to-spoke via hub)."""
        result = run_command(
            f"docker exec clab-{LAB_NAME}-host2 ping -c 3 -W 5 10.1.5.2"
        )
        assert result.returncode == 0, \
            f"host2 -> host3 failed: {result.stdout}"
```

**Step 1: Write test file**

**Step 2: Commit**

```bash
git add lessons/clab/03-routing-basics/tests/
git commit -m "feat(lesson-03): add validation tests"
```

---

## Task 7: Video Script

**Files:**
- Create: `lessons/clab/03-routing-basics/script.md`

**Structure** (following VIDEO_SCRIPT_TEMPLATE.md):

1. **Lesson Information table** -- Lesson 03, Routing Basics, 12-15 minutes
2. **Pre-Recording Checklist** -- lab environment, SR Linux image, no running labs
3. **Script sections:**
   - Opening Hook (30 sec): "Last time, cross-subnet ping failed. Today we fix that."
   - Section 1: Routing Table Fundamentals (3 min) -- whiteboard, directly connected vs static, longest prefix match
   - Section 2: Static Routes on SR Linux (2 min) -- next-hop-group concept, CLI syntax
   - Section 3: Extending Ansible (2 min) -- show routes template, host_vars extension
   - Section 4: Live Demo (3 min) -- deploy, configure, verify end-to-end pings
   - Section 5: Packet Trace (2 min) -- trace host2->host3 hop by hop through the hub, return path
   - Recap (30 sec)
   - Closing (30 sec): "Head to exercises. Next lesson: BGP replaces static routes at scale."
4. **Post-Recording Checklist**
5. **B-Roll / Supplementary Footage**
6. **Notes for Editing**

**Step 1: Write script.md**

(Full content, approximately 300 lines)

**Step 2: Commit**

```bash
git add lessons/clab/03-routing-basics/script.md
git commit -m "feat(lesson-03): add video script"
```

---

## Task 8: Update Lesson Index and Course Navigation

**Files:**
- Modify: `lessons/clab/README.md` -- update lesson 3 row with link
- Modify: `README.md` -- update lesson 3 line to remove "(coming soon)"

**Step 1: Update clab README lesson table**

Change lesson 3 row from:
```
| 3 | Routing Basics | Static routes, routing tables | Ansible playbooks |
```
to:
```
| 3 | [Routing Basics](03-routing-basics/) | Static routes, routing tables, hub-and-spoke | Ansible playbooks |
```

**Step 2: Update root README**

Change:
```
Lesson 3   Routing Basics            Static routes, routing tables            (coming soon)
```
to:
```
Lesson 3   Routing Basics            Static routes, hub-and-spoke, Ansible playbooks
```

**Step 3: Commit**

```bash
git add lessons/clab/README.md README.md
git commit -m "docs: add lesson 03 to course index and navigation"
```

---

## Execution Notes

- Container name prefix for this lesson: `clab-routing-basics-`
- All SR Linux CLI in exercises should use the `sr_cli -c "..."` pattern for single commands, `sr_cli` (interactive) for multi-step config changes
- The `traceroute` tool needs to be installed on Alpine hosts. Add `apk add --no-cache traceroute` to the host exec commands OR instruct students to run it manually in the exercise. Prefer adding it to exec for smoother experience.
- The routing loop exercise (Exercise 5) requires traceroute. Update the topology exec for all hosts to include: `- apk add --no-cache iputils traceroute`
