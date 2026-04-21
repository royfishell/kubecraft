# Lesson 2 Exercise 3

## Diagnostic Commands

```bash
docker exec -it clab-ip-fundamentals-srl1 sr_cli -c "show interface brief"
```

**output:**

```
+---------------------+------------------------------------+------------------------------------+------------------------------------+------------------------------------+------------------------------------+
|        Port         |            Admin State             |             Oper State             |               Speed                |                Type                |            Description             |
+=====================+====================================+====================================+====================================+====================================+====================================+
| ethernet-1/1        | disable                            | down                               | 25G                                |                                    |                                    |
| ethernet-1/2        | enable                             | up                                 | 25G                                |                                    |                                    |
| lo0                 | enable                             | up                                 |                                    |                                    |                                    |
| mgmt0               | enable                             | up                                 | 1G                                 |                                    |                                    |
+---------------------+------------------------------------+------------------------------------+------------------------------------+------------------------------------+------------------------------------+
```

`ethernet-1/1` is disabled, preventing data link communication.

## The Fix

```bash
 docker exec -it clab-ip-fundamentals-srl1 sr_cli
 ```

```
Using configuration file(s): []
Welcome to the srlinux CLI.
Type 'help' (and press <ENTER>) if you need any help using this.
--{ + running }--[  ]--
A:srl1# enter candidate
--{ + candidate shared default }--[  ]--
A:srl1# set / interface ethernet-1/1 admin-state enable
--{ +* candidate shared default }--[  ]--
A:srl1# commit now
All changes have been committed. Leaving candidate mode.
--{ + running }--[  ]--
```

`ethernet-1/1` is reenabled, allowing data link communication.
