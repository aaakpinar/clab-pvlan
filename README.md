# Containerlab Setup

This lab is to experience how isolated ports, VLAN re-write, BUM storm control etc. can be implemented in SR Linux.

Simply clone this repo and run:

```
clab deploy -t hetzner-01.clab.yml
```

You can ssh to clients with `admin/srllabs@123`.

```
ssh admin@clab-hetzner-01-client1
```

Also to SR Linux:

```
ssh admin@clab-hetzner-01-tor1
```

To list the nodes, simply run:

```
clab inspect -t hetzner-01.clab.yml
```

# The topology

![image](https://github.com/aaakpinar/clab-pvlan/assets/17744051/f46b1a62-6b95-47f2-8e99-262b8cdeef69)

Agg1, ToR1 and 2 are SR Linux. ToR3 is IXR-s and optional for SROS test. 

>The ToR and the clients connected to it are commented out to focus on SRL.

    ```yaml
    --8<-- "https://raw.githubusercontent.com/aaakpinar/clab-pvlan/main/hetzner-01.clab.yml"
    ```

See the configs [here](configs/).

# Isolated Ports

The client interfaces are considered to be isolated so that they can only send/receive traffic from the uplink. 

<img src="https://github.com/aaakpinar/clab-pvlan/assets/17744051/37185631-b75f-48bd-81e6-8dde46e27638" width=60% height=60%>

The clients can only reach to the gateway, 192.168.0.1.

## Isolation with MAC-filter

In/out mac-filters are applied for the isolation in ToR1.

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

## Isolation with IP-filter

Ingree IP-filter applied to allow traffic only to gateway.

```
set / acl
set / acl ipv4-filter allow-only-promiscious-in
set / acl ipv4-filter allow-only-promiscious-in entry 10
set / acl ipv4-filter allow-only-promiscious-in entry 10 action
set / acl ipv4-filter allow-only-promiscious-in entry 10 action accept
set / acl ipv4-filter allow-only-promiscious-in entry 10 match
set / acl ipv4-filter allow-only-promiscious-in entry 10 match destination-ip
set / acl ipv4-filter allow-only-promiscious-in entry 10 match destination-ip address 192.168.0.1
set / acl ipv4-filter allow-only-promiscious-in entry 10 match destination-ip mask 0.0.0.0
set / acl ipv4-filter allow-only-promiscious-in entry 1000
set / acl ipv4-filter allow-only-promiscious-in entry 1000 action
set / acl ipv4-filter allow-only-promiscious-in entry 1000 action drop
```

A static proxy ARP entry is added under the MAC-VRF as a security mechanism so ARP requests for gateway are always correct and replied by the ToR.

```
set / network-instance mac-vrf-100 bridge-table proxy-arp
set / network-instance mac-vrf-100 bridge-table proxy-arp static-entries
set / network-instance mac-vrf-100 bridge-table proxy-arp static-entries neighbor 192.168.0.1
set / network-instance mac-vrf-100 bridge-table proxy-arp static-entries neighbor 192.168.0.1 link-layer-address 1A:D9:00:FF:00:41
```

Also, a static MAC forwarding entry for the gateway to ensure L2 security.
```
set / network-instance mac-vrf-100 bridge-table static-mac
set / network-instance mac-vrf-100 bridge-table static-mac mac 1A:D9:00:FF:00:41
set / network-instance mac-vrf-100 bridge-table static-mac mac 1A:D9:00:FF:00:41 destination ethernet-1/49.0
```

# Storm Control (BUM Rate-Limiting)

To rate-limit BUM traffic on bridged subinterfaces.

```
set / interface ethernet-1/11 ethernet storm-control units kbps
set / interface ethernet-1/11 ethernet storm-control broadcast-rate 32000
set / interface ethernet-1/11 ethernet storm-control multicast-rate 32000
set / interface ethernet-1/11 ethernet storm-control unknown-unicast-rate 0
```

# VLAN Re-write

SR Linux has mac-vrfs instead of dedicated VLAN switching domains. That gives us flexibility to assign different VLANs on the access interfaces, which would only be significant on the link.

In this diagram, ToR1 gets traffic from the client7 with VLAN 400 into the `mac-vrf-127` and forwards it to `agg1` with the VLAN ID 127 on the uplink.

<img src="https://github.com/aaakpinar/clab-pvlan/assets/17744051/f5834b2a-cb8a-40a7-8def-d91b0666b725" width=50% height=50%>

```
set / network-instance mac-vrf-127
set / network-instance mac-vrf-127 type mac-vrf
set / network-instance mac-vrf-127 admin-state enable
set / network-instance mac-vrf-127 interface ethernet-1/13.400
set / network-instance mac-vrf-127 interface ethernet-1/49.127
```
 

