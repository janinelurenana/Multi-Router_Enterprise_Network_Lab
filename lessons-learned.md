# Lessons Learned

This section distills the most important takeaways from the project into **five concrete lessons**. These reflect issues encountered during real implementation and validation.

---

## Technical Lessons

* **ACLs are stateless, and rule order directly affects return traffic.**
  Blocking initiation without explicitly permitting replies can cause asymmetric connectivity, where traffic reaches the destination but responses are silently dropped.

* **Router-on-a-stick designs depend entirely on the parent interface state.**
  Subinterfaces inherit the operational status of the physical interface; a shutdown parent interface renders all VLAN gateways nonfunctional despite correct configuration.

* **NAT must be centralized at the network edge to preserve valid return paths.**
  Placing NAT anywhere other than the edge router leads to broken internet connectivity because private IPs are not routable externally.

---

## Design & Process Lessons

* **Dynamic routing (OSPF) reduces operational fragility compared to static routes.**
  OSPF provided automatic route propagation and faster recovery from link changes, making the topology more resilient and easier to scale.

* **Structured troubleshooting is more effective than command-driven guessing.**
  Using a consistent symptom → root cause → fix → verification workflow prevented misdiagnosis and reduced configuration churn across layers.

---

These lessons informed both the final network design and the troubleshooting methodology used throughout the project.
