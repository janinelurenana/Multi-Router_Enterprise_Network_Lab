## Device Roles and Responsibilities

### Routers
- **R1**: NAT edge router and OSPF neighbor
- **R2**: Inter-VLAN routing and ACL enforcement
- **R3**: Downstream router providing access network connectivity

### Switches
- **SW1**: VLAN 10 and VLAN 20 access ports, trunk uplink to R1
- **SW2**: VLAN 30 access ports, trunk uplink to R2
- **SW3**: VLAN 40 access port, trunk uplink to R3

