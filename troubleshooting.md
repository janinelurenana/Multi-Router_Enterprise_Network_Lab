# Troubleshooting Log

This document records **implementation failures** encountered during the projct and the structured process used to diagnose and resolve them. 

---

## Issue 1: Asymmetric Inter-VLAN Connectivity (ACL-Related)

#### Symptom

* VLAN 20 → VLAN 10 traffic returned **"Administratively prohibited"** (expected behavior)
* VLAN 10 → VLAN 20 traffic resulted in **request timeouts** (unexpected)
* Initial ICMP echo requests from VLAN 10 reached VLAN 20 hosts
* ICMP echo replies never returned to VLAN 10

This indicated partial connectivity with broken return traffic.

---

#### Root Cause

An extended ACL applied **inbound on R1’s VLAN 20 subinterface** was stateless and incorrectly ordered.

* A broad `deny` rule blocking VLAN 20 → VLAN 10 traffic was placed **before** ICMP-specific permit rules
* ICMP echo-reply packets returning from VLAN 20 to VLAN 10 matched the deny statement
* As a result, return traffic was silently dropped

This created **asymmetric connectivity**, a common ACL design pitfall.

---

#### Fix

The ACL was reordered to explicitly allow return traffic before enforcing the deny policy:

```text
ip access-list extended VLAN20_BLOCK
 permit icmp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 echo-reply
 deny   icmp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 echo
 permit ip any any
```

The corrected ACL was applied inbound on the VLAN 20 subinterface:

```text
interface fa0/1.20
 ip access-group VLAN20_BLOCK in
```

---

#### Verification

* VLAN 20 → VLAN 10: ICMP blocked with *administratively prohibited* response
* VLAN 10 → VLAN 20: ICMP successful
* No impact on OSPF adjacencies or NAT functionality

Verification commands:

```text
show access-lists
show ip interface fa0/1.20
ping 192.168.20.x
```

---

## Issue 2: Subinterfaces Remain Down After Configuration

#### Symptom

* VLAN subinterfaces (`fa0/1.10`, `fa0/1.20`) remained **administratively down**
* Correct encapsulation and IP configuration already present

---

#### Root Cause

The parent physical interface (`fa0/1`) was administratively down. Subinterfaces inherit the operational state of the parent interface.

---

#### Fix

Enabled the physical interface:

```text
interface fa0/1
 no shutdown
```

---

#### Verification

```text
show ip interface brief
```

All subinterfaces transitioned to **up/up** state.

---

## Issue 3: Trunk Configuration Rejected on Switch

#### Symptom

* Attempting to set trunk mode failed with encapsulation error

---

#### Root Cause

Trunk encapsulation was left in auto mode. The switch required an explicit encapsulation selection before trunk mode could be enforced.

---

#### Fix

```text
interface gi0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

---

#### Verification

```text
show interfaces trunk
```

Trunk link successfully established.

---

## Issue 4: Internet Access Failure Due to NAT Placement

#### Symptom

* Internal hosts could not ping external IPs (e.g., 8.8.8.8)
* Routing between internal networks was functional

---

#### Root Cause

* NAT was initially constrained by the GNS3 NAT node
* Return traffic from the internet had no valid translation path

---

#### Fix

* Replaced NAT node with Cloud interface
* Centralized NAT on edge router (R1)
* Clearly defined inside and outside NAT interfaces

---

#### Verification

```text
show ip nat statistics
show ip nat translations
ping 8.8.8.8
```

All internal VLANs successfully accessed the internet.

---

## Closing Note

These issues reinforced that **network failures are rarely isolated to a single command**. Most problems emerged from interactions between VLANs, routing, ACLs, and NAT — emphasizing the importance of layered troubleshooting and verification.
