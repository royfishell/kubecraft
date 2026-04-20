# Verficiation of Alpine `eth0`

```
docker exec -it clab-mixed-topology-host1 sh
/ # ip addr show eth0
2: eth0@if48: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether c2:3a:6b:46:88:45 brd ff:ff:ff:ff:ff:ff
    inet 172.20.20.2/24 brd 172.20.20.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 3fff:172:20:20::2/64 scope global flags 02
       valid_lft forever preferred_lft forever
    inet6 fe80::c03a:6bff:fe46:8845/64 scope link
       valid_lft forever preferred_lft forever
```
