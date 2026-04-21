# Lesson 2 Exercise 1 Results

## Pings

| Host | Target | Result |
| ------------- | -------------- | -------------- |
| host1 | 10.1.1.1 | **SUCCESS** |
| host2 | 10.1.3.1 | **SUCCESS** |
| srl1 | 10.1.2.2 | **SUCCESS** |
| host1 | 10.1.3.2 | **FAIL** |


## ARP Table Entries

**Host1**

| IP Address | MAC Address | Interface | 
| ------------- | --------------  | --------------  |
| 10.1.1.1 | 1a:bb:02:ff:00:01 | eth1| 
| 172.20.20.1 | 6a:e8:02:ce:15:84 | eth0 | 

**Srl1**

| IP Address | MAC Address | Interface | 
| ------------- | --------------  | --------------  |
| 10.1.1.2 | AA:C1:AB:73:66:9B | e1-1| 
| 10.1.2.2 | 1A:76:03:FF:00:01 | e1-2 | 
| 172.20.20.1 | 6A:E8:02:CE:15:84 | mgmt0 | 


## Why Cross-Subnet Pings Fail

`host1` is on the `10.0.1.0/24` network whereas `host2` is on the `10.1.3.0/24` network. Because they are separate networks, they are unable to communicate at the _Data Link_ layer (L2).
