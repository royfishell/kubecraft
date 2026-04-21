# Lesson 2 Exercise 5

## Why does a `/30` subnet mask prevent us from reaching `.1`?

`10.1.1.200/30` cannot reach `10.1.1.1` because the addresses are not in the same network.

## How does a subnet mask determine what is "local"?

An IP address consists of 4 sections called _octets_ with each _octet_ representing an 8 bit value between 0-256.

The _subnet mask_ determines the boundary between the _network_ and _host_ sections of the IP address.

In _CIDR Notation_, the _subnet mask_ is represented with `/x` where `x` is the number of bits reserved for the _network_. The remaining bits are used for the _host_ section.

> **Example 1:** `192.168.1.1/24`

In this example, 24 bits are used for the _network_ and the remaining 8 are used for the _host_.

8^2 = **256 potential addresses**

- **Network:** `192.168.1.0`
- **Broadcast:** `192.168.1.256`
- **Usable range:** `192.168.1.2-255`

> **Example 2:** `10.4.1.192/30`

**Network Bits:** 30
**Host Bits**: 2

2^2 = **4 potential addresses**

- **Network:** `10.4.1.192`
- **Broadcast:** `10.4.1.195`
- **Usable range:** `10.4.1.193 - 10.4.1.194`
