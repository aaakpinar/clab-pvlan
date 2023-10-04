This is the first draft. The lab and README will be updated.

# Containerlab Setup

This lab is to experience how isolated ports, VLAN re-write, BUM storm control etc. can be implemented in SR Linux.

Simply clone this repo and run:

```clab deploy -t hetzner-01.clab.yml```

# The topology

![image](https://github.com/aaakpinar/clab-pvlan/assets/17744051/f46b1a62-6b95-47f2-8e99-262b8cdeef69)

Agg1, ToR1 and 2 are SR Linux. ToR3 is IXR-s.

# Isolated Ports

The client interfaces are considered to be isolated so that they can only send/receive traffic from the uplink. 

<img src="https://github.com/aaakpinar/clab-pvlan/assets/17744051/37185631-b75f-48bd-81e6-8dde46e27638" width=60% height=60%>

In/out mac-filters are applied for the isolation.

```
set / acl
set / acl egress-mac-filtering true
set / acl mac-filter allow-promiscious-only-in
set / acl mac-filter allow-promiscious-only-in entry 10
set / acl mac-filter allow-promiscious-only-in entry 10 action
set / acl mac-filter allow-promiscious-only-in entry 10 action accept
set / acl mac-filter allow-promiscious-only-in entry 10 match
set / acl mac-filter allow-promiscious-only-in entry 10 match ethertype arp
set / acl mac-filter allow-promiscious-only-in entry 20
set / acl mac-filter allow-promiscious-only-in entry 20 match
set / acl mac-filter allow-promiscious-only-in entry 20 match destination-mac
set / acl mac-filter allow-promiscious-only-in entry 20 match destination-mac address 1A:D9:00:FF:00:41
set / acl mac-filter allow-promiscious-only-in entry 20 match destination-mac mask FF:FF:FF:FF:FF:FF
set / acl mac-filter allow-promiscious-only-in entry 1000
set / acl mac-filter allow-promiscious-only-in entry 1000 action
set / acl mac-filter allow-promiscious-only-in entry 1000 action drop

set / acl mac-filter allow-promiscious-only-out
set / acl mac-filter allow-promiscious-only-out entry 10
set / acl mac-filter allow-promiscious-only-out entry 10 action
set / acl mac-filter allow-promiscious-only-out entry 10 action accept
set / acl mac-filter allow-promiscious-only-out entry 10 match
set / acl mac-filter allow-promiscious-only-out entry 10 match source-mac
set / acl mac-filter allow-promiscious-only-out entry 10 match source-mac address 1A:D9:00:FF:00:41
set / acl mac-filter allow-promiscious-only-out entry 10 match source-mac mask FF:FF:FF:FF:FF:FF
set / acl mac-filter allow-promiscious-only-out entry 1000
set / acl mac-filter allow-promiscious-only-out entry 1000 action
set / acl mac-filter allow-promiscious-only-out entry 1000 action drop
```

# VLAN Re-write

SR Linux has mac-vrfs instead of dedicated VLAN switching domains. That gives us flexibility to assign different VLANs on the access interfaces, which would only be significant on the link.

In this diagram, ToR1 gets traffic from the client7 with VLAN 400 into the `mac-vrf-127` and switches it to `agg1` and tag it with the id 127 on the uplink.

<img src="https://github.com/aaakpinar/clab-pvlan/assets/17744051/f5834b2a-cb8a-40a7-8def-d91b0666b725" width=50% height=50%>



