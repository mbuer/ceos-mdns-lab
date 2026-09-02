# Lab 05 - mDNS Across Layer 3 Boundaries

## Objective

This lab examines how mDNS behaves across routed Layer 3 boundaries.

The previous lab proved that generic IPv4 multicast can be routed between two
subnets using:

- IGMP
- PIM Sparse Mode
- a Rendezvous Point
- multicast routing
- Reverse Path Forwarding

This lab tests whether mDNS behaves the same way.

The specific question is:

> Does mDNS traffic sent to `224.0.0.251:5353` cross the routed boundary between
> client1 and client2 when normal multicast routing is already configured?

---

## Topology

```text
client1
10.10.10.2/30
    |
    |
cEOS1
10.10.10.1/30
    |
    | 10.10.100.0/30
    |
cEOS2
10.10.20.1/30
    |
    |
client2
10.10.20.2/30
```

The routed path is:

```text
client1 -> cEOS1 -> cEOS2 -> client2
```

OSPF provides unicast routing between the endpoint networks.

The multicast configuration from Lab 04 is retained so that generic multicast
routing is already functional.

This allows mDNS behavior to be compared directly against the multicast traffic
tested in Lab 04.

---

## mDNS Basics

mDNS uses:

```text
IPv4 multicast address: 224.0.0.251
UDP port:               5353
```

mDNS is designed for discovery on the local network segment.

Typical mDNS traffic uses an IP TTL of:

```text
255
```

Unlike the multicast group used in Lab 04:

```text
239.1.1.1
```

the mDNS multicast address belongs to the IPv4 link-local multicast range.

The purpose of this lab is to observe the resulting behavior directly rather
than assume that generic multicast routing will also carry mDNS.

---

## Initial Linux Routing Behavior

Before testing mDNS, the routing decision on client1 was checked:

```bash
docker exec clab-mdns-l3-client1 \
  ip route get 224.0.0.251
```

Initially Linux selected the Containerlab management interface:

```text
multicast 224.0.0.251 dev eth0 src 172.20.20.11
```

This would send the test traffic into the Containerlab management network rather
than the lab data plane.

A host route was therefore added:

```bash
docker exec clab-mdns-l3-client1 \
  ip route add 224.0.0.251/32 dev eth1
```

The routing decision then became:

```text
multicast 224.0.0.251 dev eth1 src 10.10.10.2
```

This confirms that mDNS traffic is sent through the intended lab interface.

The route is persisted in `topology.clab.yml`.

---

## Creating a Valid mDNS Query

A Python script using only the standard library was used to construct a valid
DNS query.

The query requests:

```text
_services._dns-sd._udp.local
```

with DNS record type:

```text
PTR
```

The packet is sent to:

```text
224.0.0.251:5353
```

with multicast TTL:

```text
255
```

Example sender:

```bash
docker exec clab-mdns-l3-client1 python3 -c '
import socket, struct

name = "_services._dns-sd._udp.local"
qname = b"".join(
    bytes([len(x)]) + x.encode()
    for x in name.split(".")
) + b"\x00"

packet = struct.pack("!HHHHHH", 0, 0, 1, 0, 0, 0)
packet += qname
packet += struct.pack("!HH", 12, 1)

s = socket.socket(
    socket.AF_INET,
    socket.SOCK_DGRAM,
    socket.IPPROTO_UDP
)

s.setsockopt(
    socket.IPPROTO_IP,
    socket.IP_MULTICAST_IF,
    socket.inet_aton("10.10.10.2")
)

s.setsockopt(
    socket.IPPROTO_IP,
    socket.IP_MULTICAST_TTL,
    255
)

s.sendto(packet, ("224.0.0.251", 5353))

print("sent valid mDNS query")
'
```

---

## Source-Side Packet Capture

The mDNS query was captured on client1:

```bash
docker exec -it clab-mdns-l3-client1 \
  tcpdump -l -ni eth1 -vv udp port 5353
```

Observed packet:

```text
02:11:20.671659 IP
    (tos 0x0, ttl 255, id 10721, offset 0,
     flags [DF], proto UDP (17), length 74)

    10.10.10.2.56013 > 224.0.0.251.5353:

    [udp sum ok]
    0 PTR (QM)?
    _services._dns-sd._udp.local.
```

This proves that:

- the query is a valid DNS packet
- the source address is `10.10.10.2`
- the destination is `224.0.0.251`
- UDP port `5353` is used
- TTL is `255`
- the packet leaves the intended data-plane interface

---

## First Routed Boundary

The next test examined whether cEOS1 forwards the mDNS packet onto the transit
network toward cEOS2.

A capture was started on the transit-facing interface of cEOS1:

```bash
docker exec -it clab-mdns-l3-ceos1 \
  tcpdump -ni eth2 -vv -c 1 udp port 5353
```

The same valid mDNS query was then sent again from client1.

No packet appeared on `eth2`.

This demonstrates that the mDNS packet was not forwarded from the client-facing
side of cEOS1 onto the routed transit network.

The observed path is therefore:

```text
client1 eth1
     |
     | mDNS visible
     v
cEOS1 client-facing interface
     |
     X
     X not forwarded
     X
     |
cEOS1 transit interface
```

---

## Receiver-Side Observation

A packet capture was also run on client2:

```bash
docker exec -it clab-mdns-l3-client2 \
  tcpdump -l -ni eth1 -vv udp port 5353
```

The same mDNS query was transmitted from client1.

No matching mDNS packet was observed on client2.

This confirms the complete behavior:

```text
client1
   |
   | mDNS
   v
cEOS1
   X
   X Layer 3 boundary
   X
cEOS2
   |
client2
```

---

## Comparison With Lab 04

Lab 04 used:

```text
239.1.1.1:5000
```

and successfully routed multicast traffic using:

- IGMP
- PIM Sparse Mode
- an RP
- multicast routing

The multicast path became:

```text
client1
   |
cEOS1
   |
cEOS2
   |
client2
```

Lab 05 uses:

```text
224.0.0.251:5353
```

and produces a different result:

```text
client1
   |
cEOS1
   X
cEOS2
   |
client2
```

The important lesson is:

> Generic multicast routing does not automatically make mDNS discovery work
> across routed Layer 3 boundaries.

---

## Why PIM Does Not Solve This

PIM was configured and functional from the previous lab.

This does not make mDNS behave like the administratively scoped multicast group
used in Lab 04.

mDNS is intentionally link-local discovery traffic.

The multicast routing infrastructure can therefore be operational while mDNS
still remains confined to its local Layer 3 segment.

This distinction is important:

```text
IGMP + PIM
    =
generic routed multicast

mDNS Gateway
    =
controlled mDNS discovery across Layer 3 boundaries
```

These solve different problems.

---

## Packet Flow Observed

```text
Valid mDNS query generated
        |
        v
client1 eth1
        |
        | 224.0.0.251:5353
        | TTL 255
        v
cEOS1 ingress
        |
        X
        X not forwarded onto routed transit
        X
        |
cEOS1 eth2
        |
        X
        |
client2
```

---

## Key Lessons

### 1. mDNS is multicast, but not ordinary routed multicast

mDNS uses multicast transport, but its intended scope is local-link discovery.

---

### 2. PIM does not automatically provide cross-subnet mDNS discovery

A functioning PIM multicast domain does not mean mDNS will traverse Layer 3
boundaries.

---

### 3. Packet capture is essential

The packet path was proven directly:

```text
source interface:      packet present
transit interface:     packet absent
remote receiver:       packet absent
```

This identifies the Layer 3 boundary precisely.

---

### 4. Linux routing decisions matter

Without the explicit route:

```text
224.0.0.251/32 dev eth1
```

Linux selected the Containerlab management interface.

The multicast destination alone does not guarantee that the intended lab
interface will be chosen.

---

### 5. This establishes the baseline for an mDNS Gateway

The desired behavior in the next lab is:

```text
without mDNS Gateway
        |
        v
discovery fails across Layer 3

enable mDNS Gateway
        |
        v
discovery succeeds across Layer 3
```

That creates a clear before-and-after validation.

---

## Result

Lab 05 successfully demonstrated that mDNS traffic sent to:

```text
224.0.0.251:5353
```

remains local to the source-side Layer 3 segment.

The packet was successfully generated and captured on client1, but it was not
observed on the routed transit interface or on client2.

The multicast routing configuration from Lab 04 remained present, confirming
that normal PIM multicast routing does not by itself provide mDNS discovery
across routed boundaries.

Lab 06 will introduce the Arista mDNS Gateway and test whether it can bridge
this discovery boundary in a controlled manner.
