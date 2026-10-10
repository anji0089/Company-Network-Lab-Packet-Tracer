# Troubleshooting log

Faults found and fixed in the Lab 11 company network (Cisco Packet Tracer).
Each entry follows the same format: **symptom, hypothesis, evidence, fix**.

> Status: Issues 1 to 3 and Faults 1 and 2 are complete. Fault 3 is in progress (see the end).

---

## Issue 1: Internet server unreachable (found while building the baseline)

**Symptom**
PC1 could reach its gateway and the LogServer, but not the Internet server (`198.51.100.10`). The ISP router could ping the server, but R1 could not.

**Hypothesis**
R1 is sending traffic for the Internet network toward the wrong interface.

**Evidence**
`show ip route` on R1 listed the Internet network as directly connected to the switch-facing port:

```
C 198.51.100.0/24 is directly connected, GigabitEthernet0/0
L 198.51.100.1/32 is directly connected, GigabitEthernet0/0
```

`Gi0/0` should have no address of its own, because only its subinterfaces (`.10`, `.20`, `.30`) carry addresses. The ISP's address had been entered on the wrong router interface.

**Fix**
```
interface gig0/0
 no ip address
 no shutdown
```
After this, `ping 198.51.100.10` from R1 returned `!!!!!`.

---

## Issue 2: PC3 had a stale address from an earlier configuration

**Symptom**
PC3 (HR) could not reach the internet, while PC1 (Sales) could.

**Evidence**
`ipconfig` on PC3 showed a Sales address with an HR gateway:

```
IPv4 Address: 192.168.10.53
Default Gateway: 192.168.20.1
```

A host can only reach a gateway on its own network, so every off-network packet failed.

**Fix**
Switched PC3 from Static to DHCP again. It then received `192.168.20.51` with gateway `192.168.20.1`, and the ping to `198.51.100.10` worked.

---

## Fault 1: Sales cannot reach the internet, HR can

**Symptom (user report)**
"Nobody in Sales can get to the internet. HR is fine. Sales can still reach the log server."

**Hypothesis**
Sales traffic is not being translated by NAT, because the access list that selects NAT traffic no longer includes the Sales network.

**Evidence**
Narrowing down: PC1 could ping its gateway and the LogServer, and R1 itself could ping the Internet server. So the fault was between the inside network and the internet.

The NAT table contained only HR entries (no `192.168.10.x`):

```
icmp 203.0.113.2:25   192.168.20.51:25   198.51.100.10:25   198.51.100.10:25
icmp 203.0.113.2:26   192.168.20.51:26   198.51.100.10:26   198.51.100.10:26
...
```

Comparing the NAT access list with the baseline:

Before:
```
Standard IP access list 1
    10 permit 192.168.10.0 0.0.0.255 (24 match(es))
    20 permit 192.168.20.0 0.0.0.255 (16 match(es))
```

After the fault:
```
Standard IP access list 1
    20 permit 192.168.20.0 0.0.0.255 (40 match(es))
```

**Fix**
```
ip access-list standard 1
 10 permit 192.168.10.0 0.0.0.255
clear ip nat translation *
```
PC1 reached the internet again, and `show ip nat translations` showed `192.168.10.51` entries.

**Lesson**
Traffic that is not matched by the NAT access list leaves with a private source address, and the internet cannot reply to it. Sales still reached internal destinations because those do not need NAT.

---

## Fault 2: HR loses all connectivity, Sales and servers are fine

**Symptom (user report)**
"HR can't do anything. Sales and the servers are fine." PC3 could not even reach its own gateway.

**Hypothesis**
VLAN 20 is not passing over the trunk between SW1 and R1.

**Evidence**
PC3's settings were correct, and its port was in the right VLAN:

```
IPv4 Address: 192.168.20.51
Default Gateway: 192.168.20.1

20   HR     active    Fa0/3
```

The trunk allowed only VLANs 10 and 30:

```
SW1#show interfaces trunk
Port        Vlans allowed on trunk
Gig0/1      10,30
```

**Fix**
```
interface gig0/1
 switchport trunk allowed vlan all
```
PC3 could then reach `192.168.20.1` and the internet.

**Lesson**
A trunk only carries the VLANs on its allowed list. Frames from an excluded VLAN are dropped at the switch, so the fault appears as a complete loss of connectivity for that VLAN alone.

---

## Fault 3: Port security shutdown (in progress)

**Symptom (user report)**
"My PC (PC2) lost the network after someone moved a laptop onto my desk."

**Before the fault**
```
SW1#show port-security interface fa0/2
Port Status : Secure-up
Maximum MAC Addresses : 1
Sticky MAC Addresses : 1
Last Source Address:Vlan : 0001.C968.9721:10
Security Violation Count : 0
```

**Evidence so far**
After a laptop was connected to Fa0/2 and PC2 was reconnected, the link stayed down:

- `show interfaces status` showed Fa0/2 as `err-disabled`
- `Port Status : Secure-shutdown`
- `Security Violation Count : 1`

**Still to collect**
- The `PSECURE_VIOLATION` line from the LogServer
- The `Last Source Address` after the violation, compared with the laptop's MAC
- The fix (remove the unknown device, then `shutdown` and `no shutdown` on the port) and the "after" output
