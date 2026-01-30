# Lessons Learned

This document captures the **conceptual corrections, operational mistakes, and design insights** gained while building and validating this network. It is intentionally reflective and focuses on *why things behaved the way they did*, not just how they were fixed.

---

## 1. Router-on-a-Stick Is Logically Simple, But Operationally Fragile

**Initial assumption:** Creating VLAN subinterfaces on a router is straightforward once encapsulation is configured.

**What actually happened:** Subinterfaces remained down even after being configured correctly.

**Root cause:** The parent physical interface was administratively down. Subinterfaces depend entirely on the state of their parent interface.

**Correction:** Enabled the parent interface with `no shutdown`.

**Key takeaway:** In router-on-a-stick designs, the physical interface is not “optional plumbing.” If it’s down, *everything above it silently fails*.

---

## 2. Trunking Requires an Explicit Language Agreement

**Symptom:** Attempting to force an interface into trunk mode failed with an encapsulation error.

**Root cause:** The switch supported multiple trunking encapsulations. Trunk mode cannot be enforced until the encapsulation is explicitly set.

**Fix:** Configured trunk encapsulation to `dot1q` before enabling trunk mode.

**Key takeaway:** Trunking is a negotiated behavior. When automation fails, being explicit avoids ambiguity.

---

## 3. ACLs Are Stateless 

**Symptom:**

* VLAN 20 → VLAN 10 traffic was correctly blocked
* VLAN 10 → VLAN 20 traffic timed out unexpectedly

**Root cause:** An inbound extended ACL on the VLAN 20 subinterface denied ICMP return traffic due to rule ordering. Echo replies were matched by a broad deny rule.

**Fix:** Reordered the ACL to explicitly permit ICMP echo-reply traffic before denying initiation attempts.

**Key takeaway:** ACLs do not track sessions. If return traffic isn’t explicitly allowed, communication breaks asymmetrically.

---

## 4. NAT Must Live at the Edge 

**Initial failure:** Internal hosts could not reach the internet despite correct routing.

**Root cause:** NAT behavior was misunderstood and initially constrained by the GNS3 NAT node. Additionally, return traffic from the internet had no valid path back to private IPs.

**Fix:**

* Replaced the NAT node with a Cloud interface
* Centralized NAT on R1 only
* Defined clear inside vs outside interfaces

**Key takeaway:** NAT is not a convenience feature. It is an **edge responsibility**, and placing it anywhere else creates broken return paths.

---

## 5. OSPF Wildcard Masks Required a Mental Model Shift

**Initial confusion:** Why OSPF uses wildcard masks instead of subnet masks.

**Understanding gained:**

* `0` means *match exactly*
* `255` means *ignore*

Using `192.168.10.0 0.0.0.255` instructs the router to enable OSPF on any interface matching the first three octets.

**Key takeaway:** OSPF configuration is interface-discovery driven, not subnet-definition driven. Once this clicks, OSPF becomes predictable.

---

## 6. “Administratively Down” vs “Down” Is a Crucial Diagnostic Signal

**Observation:** Interfaces reported different down states during testing.

**Insight gained:**

* *Administratively down* indicates an intentional shutdown
* *Down/down* suggests physical or Layer 1/2 issues

**Key takeaway:** Interface status messages are not cosmetic. They immediately narrow the troubleshooting domain.

---

## 7. Layer 2 Switches Do Not Need Default Gateways for Forwarding

**Common misconception:** All devices need a default gateway to forward traffic.

**Correction:** Layer 2 switches only require a default gateway for *management traffic originating from the switch itself*, not for forwarding user traffic.

**Key takeaway:** Understanding device roles prevents unnecessary and misleading configuration.

---

## 8. Serial Links Require Clock Ownership

**Symptom:** Serial interfaces failed to synchronize properly.

**Root cause:** Clock rate was not configured on the DCE side of the link.

**Understanding gained:**

* DCE provides timing
* DTE consumes timing

**Key takeaway:** Serial communication requires a single timing authority. Without it, links remain unstable or down.

---

## 9. Duplicate IP Addressing Can Cause Non-Obvious Failures

**Symptom:** A serial interface unexpectedly dropped after resetting another interface.

**Root cause:** Two interfaces were configured with IPs in the same subnet, creating routing ambiguity.

**Fix:** Removed the duplicate addressing and reset the affected interface.

**Key takeaway:** IP conflicts can destabilize unrelated interfaces due to routing table recalculations.

---

## 10. Systematic Troubleshooting Beats Command Guessing

**Pattern observed:** Most failures were not caused by single misconfigurations, but by *interaction between layers*.

**Effective method:**

1. Identify the symptom
2. Isolate the layer involved
3. Verify assumptions
4. Apply a targeted fix
5. Confirm with verification commands

**Key takeaway:** Structured troubleshooting scales. Random command execution does not.

---

## Final Reflection

This project reinforced that real networking work is less about typing commands and more about **reasoning through behavior**. Most issues were not complex; they were subtle, layered, and required disciplined analysis to resolve.

The value of this lab came not from final connectivity, but from understanding *why it failed along the way*.
