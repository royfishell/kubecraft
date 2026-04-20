# Exercise 6

## Commands Used to Diagnose Issue

Check status of clab topology

```bash
❯ clab inspect -t topology/lab.clab.yml
11:38:54 INFO Parsing & checking topology file=lab.clab.yml
╭─────────────────────┬───────────────────────────────┬─────────┬───────────────────╮
│         Name        │           Kind/Image          │  State  │   IPv4/6 Address  │
├─────────────────────┼───────────────────────────────┼─────────┼───────────────────┤
│ clab-first-lab-srl1 │ srl                           │ running │ 172.20.20.3       │
│                     │ ghcr.io/nokia/srlinux:24.10.1 │         │ 3fff:172:20:20::3 │
├─────────────────────┼───────────────────────────────┼─────────┼───────────────────┤
│ clab-first-lab-srl2 │ srl                           │ exited  │ N/A               │
│                     │ ghcr.io/nokia/srlinux:24.10.1 │         │ N/A               │
╰─────────────────────┴───────────────────────────────┴─────────┴───────────────────╯
```

`docker ps -a` to check container status and identify stopped container

```bash
❯ docker ps -a
CONTAINER ID   IMAGE                           COMMAND                  CREATED         STATUS                       PORTS     NAMES
182a0d172c7a   ghcr.io/nokia/srlinux:24.10.1   "/tini -- fixuid -q …"   3 minutes ago   Up 3 minutes                           clab-first-lab-srl1
8e8e8c797340   ghcr.io/nokia/srlinux:24.10.1   "/tini -- fixuid -q …"   3 minutes ago   Exited (143) 3 minutes ago             clab-first-lab-srl2
```

## How I Fixed It

start `clab-first-lab-srl2`

```bash
❯ docker start clab-first-lab-srl2
clab-first-lab-srl2
```

## Why `docker ps` alone wouldn't show the stopped container

`docker ps` only shows running containers. To see all containers including stopped containers, we need to run `docker ps -a` (all).
