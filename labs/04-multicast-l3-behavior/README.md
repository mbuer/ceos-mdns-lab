# Lab 04 - Multicast Behavior Across Layer 3

This lab introduces routed IPv4 multicast using the Layer 3 topology built in the previous labs.

The starting topology already had working unicast routing through OSPF:

    client1 -> cEOS1 -> cEOS2 -> client2

The objective was to determine whether multicast traffic behaves the same way.

It does not.

The lab first proves that ordinary Layer 3 routing and OSPF are insufficient to forward multicast between routed networks. It then introduces IGMP, multicast routing, PIM Sparse Mode, a Rendezvous Point, and Reverse Path Forwarding until multicast traffic successfully crosses both routers.

## Objective

The goals of this lab are to:

- generate IPv4 multicast traffic from a Linux endpoint
- distinguish unicast routing from multicast forwarding
- observe multicast traffic on the source subnet
- demonstrate that multicast is not automatically forwarded by an IP router
- understand receiver membership through IGMP
- understand router-to-router multicast signaling through PIM
- configure PIM Sparse Mode
- configure a Rendezvous Point
- understand Reverse Path Forwarding
- inspect the multicast routing table
- successfully transport multicast across two routed cEOS routers
- make the final configuration reproducible through Containerlab

## Topology

```text
client1                 cEOS1                 cEOS2                 client2
10.10.10.2/30           Et1                   Et1                   10.10.20.2/30
     |                  10.10.10.1           10.10.20.1                 |
     |                       |                     |                      |
     +-----------------------+                     +----------------------+
                             |                     |
                            Et2                   Et2
                        10.10.100.1 -------- 10.10.100.2
                             10.10.100.0/30
```

The multicast source is:

```text
10.10.10.2
```

The multicast receiver is:

```text
10.10.20.2
```

The test multicast group is:

```text
239.1.1.1
```

UDP port:

```text
5000
```

## Starting Point

Lab 04 was copied from Lab 03.

The routers therefore already:

- had working Layer 3 interfaces
- had IP routing enabled
- formed an OSPF adjacency
- dynamically learned the remote endpoint networks
- provided successful end-to-end unicast connectivity

Before working on multicast, unicast was verified again:

```bash
docker exec clab-multicast-l3-client1 ping -c 4 10.10.20.2
```

This succeeded after OSPF convergence.

This baseline is important because a later multicast failure could then be attributed to multicast behavior rather than broken Layer 3 routing.

## Multicast Is Not Ordinary Unicast

A unicast packet has one destination host.

For example:

```text
10.10.10.2 -> 10.10.20.2
```

The router can use its normal routing table to determine the next hop.

Multicast behaves differently.

The destination:

```text
239.1.1.1
```

does not identify a single host.

Instead, it identifies a multicast group.

Potentially many receivers on many different networks may join that same group.

The router therefore needs additional information:

- where multicast packets from the source should arrive
- which interfaces contain interested receivers
- which router-to-router path should carry the multicast traffic

OSPF does not provide this multicast forwarding state.

## Choosing the Correct Source Interface

The first multicast test revealed an important Linux routing behavior.

Initially:

```bash
docker exec clab-multicast-l3-client1 ip route get 239.1.1.1
```

returned:

```text
multicast 239.1.1.1 dev eth0 src 172.20.20.x
```

Linux was therefore trying to send the multicast traffic through Containerlab's management interface instead of the lab interface.

This was corrected with:

```bash
ip route add 239.1.1.1/32 dev eth1
```

Afterward:

```bash
ip route get 239.1.1.1
```

returned:

```text
multicast 239.1.1.1 dev eth1 src 10.10.10.2
```

This explicitly places the multicast test group onto the lab data plane.

## Generating Multicast Traffic

Python was used because the Linux endpoint image already contained Python 3 and required no additional packages.

Client1 sent UDP multicast traffic to:

```text
239.1.1.1:5000
```

The multicast interface was explicitly set to:

```text
10.10.10.2
```

and the multicast TTL was set to 5 so that TTL 1 would not prevent routed testing.

A packet capture on client1 confirmed:

```text
10.10.10.2.xxxxx > 239.1.1.1.5000: UDP, length 15
```

This proved that client1 was correctly generating multicast and transmitting it on Ethernet1.

## Initial Multicast Failure

The same traffic was captured on client2's Ethernet1.

No packets arrived.

At this point:

```text
Unicast:
client1 -> cEOS1 -> cEOS2 -> client2
WORKS

Multicast:
client1 -> 239.1.1.1 -> routed boundary
FAILS
```

This demonstrates a fundamental routing principle:

> Ordinary IP routing and OSPF do not automatically provide routed multicast forwarding.

Additional multicast control-plane state is required.

## IGMP

IGMP stands for Internet Group Management Protocol.

Its role in this lab is communication between a multicast receiver and its local router.

The relationship is:

```text
receiver <---- IGMP ----> local multicast router
```

Client2 joined:

```text
239.1.1.1
```

using a Python multicast socket.

A packet capture showed:

```text
10.10.20.2 > 224.0.0.22: igmp v3 report
[gaddr 239.1.1.1 ...]
```

This is an IGMPv3 membership report.

Conceptually client2 is saying:

```text
"I want to receive multicast group 239.1.1.1."
```

The IGMP packet had TTL 1 because IGMP membership signaling is local to the attached network.

## IGMP Is Not Multicast Routing

IGMP does not tell cEOS1 how to reach client2.

It only tells cEOS2:

```text
There is an interested receiver for 239.1.1.1 on this local interface.
```

The distinction is:

```text
IGMP:
host <-> local router

PIM:
router <-> router
```

This is one of the most important concepts in this lab.

## Enabling Multicast Routing

Multicast forwarding was enabled globally on the cEOS routers:

```text
ip multicast-routing
```

Without this command, EOS reported:

```text
ipv4 multicast routing is not configured on VRF default
```

This demonstrated that configuring multicast features under an interface alone does not activate global IPv4 multicast forwarding.

## Receiver-Side Multicast Configuration

On cEOS2 Ethernet1:

```text
ip igmp
pim ipv4 sparse-mode
```

After global multicast routing was enabled:

```text
show ip igmp interface Ethernet1
```

confirmed:

```text
IGMP on this interface: enabled
Multicast routing on this interface: enabled
IGMP querier: 10.10.20.1
Current IGMP router version: 3
```

cEOS2 therefore became the IGMP router and querier for the client2 subnet.

## IGMP Membership Learned by cEOS2

With client2 joined to `239.1.1.1`:

```text
show ip igmp groups
```

reported:

```text
239.1.1.1  Ethernet1  ...  Last Reporter 10.10.20.2
```

This proves that cEOS2 learned the local receiver membership.

The multicast router now knew:

```text
Group:
239.1.1.1

Interested interface:
Ethernet1

Receiver:
10.10.20.2
```

## PIM

PIM stands for Protocol Independent Multicast.

PIM provides multicast control-plane communication between routers.

In this lab:

```text
client1
   |
 cEOS1
   |
   | PIM
   |
 cEOS2
   |
 client2
```

PIM Sparse Mode was enabled on:

```text
cEOS1 Ethernet1
cEOS1 Ethernet2
cEOS2 Ethernet1
cEOS2 Ethernet2
```

The endpoint-facing interfaces participate in multicast routing.

The Ethernet2 interfaces form the router-to-router PIM relationship.

## PIM Neighbor Formation

After PIM was enabled on both transit interfaces:

```text
show ip pim neighbor
```

on cEOS1 reported:

```text
10.10.100.2  Ethernet2  ...  sparse
```

and cEOS2 reported:

```text
10.10.100.1  Ethernet2  ...  sparse
```

The routers therefore established a PIM adjacency across:

```text
10.10.100.0/30
```

The multicast control plane now had router-to-router connectivity.

## PIM Sparse Mode

This lab uses PIM Sparse Mode.

Sparse Mode assumes multicast receivers are not present everywhere.

Routers build forwarding trees only where receivers explicitly express interest.

This fits the behavior observed in the lab:

```text
client2 joins 239.1.1.1
        |
        v
cEOS2 learns receiver interest
        |
        v
PIM builds multicast state toward the source/RP
```

When the receiver leaves the group, the multicast forwarding state can expire again.

## Rendezvous Point

PIM Sparse Mode uses a Rendezvous Point, or RP.

The RP provides a common meeting point between multicast sources and receiver-side multicast routers.

This lab uses:

```text
RP = 10.10.100.1
```

which is cEOS1's transit address.

Both routers were configured with the same RP:

```text
router pim sparse-mode
   ipv4
      rp address 10.10.100.1
```

Conceptually:

```text
client1
   |
 cEOS1
 RP 10.10.100.1
   |
   | PIM
   |
 cEOS2
   |
 client2
```

The RP is part of how PIM Sparse Mode initially connects multicast sources with interested receiver networks.

## Multicast Source and Group State

A multicast forwarding entry can be described using:

```text
(S,G)
```

where:

```text
S = Source
G = Group
```

For this lab:

```text
S = 10.10.10.2
G = 239.1.1.1
```

Therefore:

```text
(10.10.10.2, 239.1.1.1)
```

means:

> multicast traffic originating from client1 and destined for multicast group 239.1.1.1

The multicast table also showed wildcard group state.

EOS displayed this using source:

```text
0.0.0.0
```

for the group.

Conceptually this represents:

```text
(*,G)
```

or multicast state for the group independent of one specific source.

## Reverse Path Forwarding

Multicast routers must protect against forwarding packets arriving from the wrong direction.

They use Reverse Path Forwarding, or RPF.

Instead of asking:

```text
Where should I send traffic toward the destination?
```

RPF asks:

```text
If I wanted to reach the multicast source using normal unicast routing,
which interface would I use?
```

A multicast packet from that source is expected to arrive on that interface.

This prevents multicast routing loops and establishes the correct upstream direction.

## OSPF and RPF

PIM is called Protocol Independent because it does not require one particular unicast routing protocol.

In this lab, OSPF already provides the unicast routing information.

PIM uses that information for RPF.

On cEOS2 the multicast table showed:

```text
RPF route: [U] 10.10.10.0/30 [110/20] via 10.10.100.1
```

The `U` means the RPF information came from the unicast routing table.

The `[110/20]` route is the same OSPF-learned route introduced in Lab 03.

This demonstrates an important relationship:

```text
OSPF:
provides unicast reachability

PIM:
uses that reachability for multicast RPF decisions
```

OSPF does not route multicast itself.

It provides information that PIM can use.

## Successful Multicast Forwarding

Once all required multicast components were active, client1 sent:

```text
hello multicast
```

to:

```text
239.1.1.1:5000
```

Client2 received:

```text
('10.10.10.2', 36618) hello multicast
```

This proves successful routed multicast transport across both cEOS routers.

The forwarding path was:

```text
client1
10.10.10.2
    |
    v
cEOS1 Ethernet1
    |
    v
cEOS1 Ethernet2
    |
    v
cEOS2 Ethernet2
    |
    v
cEOS2 Ethernet1
    |
    v
client2
10.10.20.2
```

## Multicast Routing Table - cEOS1

With the receiver joined and multicast active, cEOS1 reported:

```text
239.1.1.1
  0.0.0.0, RP 10.10.100.1, flags: WL
    Incoming interface: Register
    Outgoing interface list:
      Ethernet2

  10.10.10.2, flags: SLP
    Incoming interface: Ethernet1
    RPF route: [U] 10.10.10.0/30 [0/0]
    Outgoing interface list:
      Ethernet2
```

The source-specific entry clearly shows:

```text
Incoming interface:
Ethernet1

Outgoing interface:
Ethernet2
```

cEOS1 therefore receives multicast from client1 and forwards it toward cEOS2.

The `L` flag identifies the multicast source as locally attached.

## Multicast Routing Table - cEOS2

cEOS2 reported:

```text
239.1.1.1
  0.0.0.0, RP 10.10.100.1, flags: WL
    Incoming interface: Ethernet2
    RPF route: [U] 10.10.100.0/30 via 10.10.100.1
    Outgoing interface list:
      Ethernet1

  10.10.10.2, flags: SP
    Incoming interface: Ethernet2
    RPF route: [U] 10.10.10.0/30 [110/20] via 10.10.100.1
    Outgoing interface list:
      Ethernet1
```

The source-specific entry shows:

```text
Incoming interface:
Ethernet2

Outgoing interface:
Ethernet1
```

cEOS2 therefore receives the multicast stream from cEOS1 and forwards it onto the receiver subnet.

Together the two multicast routing tables provide direct evidence of the complete forwarding path.

## Why the Receiver Must Stay Joined

During testing, the client2 Python process was stopped.

After its multicast socket closed, the IGMP membership eventually disappeared:

```text
show ip igmp groups
```

became empty.

The multicast routing table also lost its active forwarding path.

When the receiver joined the group again, IGMP membership returned and the multicast forwarding state was rebuilt.

This demonstrates that PIM Sparse Mode forwarding is demand-driven.

The network does not permanently forward multicast toward a subnet when no receiver is interested.

## Protocol Roles

The three routing/control mechanisms used in this lab have distinct responsibilities.

### OSPF

```text
router <-> router
```

Provides unicast network reachability.

Example:

```text
10.10.10.0/30 via 10.10.100.1
```

### IGMP

```text
receiver <-> local router
```

Communicates multicast group membership.

Example:

```text
client2 wants 239.1.1.1
```

### PIM

```text
multicast router <-> multicast router
```

Builds multicast forwarding state between source-side and receiver-side networks.

A useful mental model is:

```text
OSPF:
"How do I reach that IP network?"

IGMP:
"Does anybody on this local network want this multicast group?"

PIM:
"How should routers build a multicast path between the source and interested receivers?"
```

## Final Control and Forwarding Flow

The complete process can be visualized as:

```text
client2 joins 239.1.1.1
        |
        | IGMP
        v
      cEOS2
        |
        | PIM
        v
      cEOS1
        |
        | source 10.10.10.2
        v
      client1
```

Once the multicast tree exists, the actual data flows in the opposite direction:

```text
client1
        |
        | UDP multicast
        v
      cEOS1
        |
        | multicast forwarding
        v
      cEOS2
        |
        | multicast forwarding
        v
      client2
```

This distinction between control-plane signaling and data-plane forwarding is fundamental to understanding multicast.

## Reproducible Configuration

The final cEOS configuration includes:

```text
ip multicast-routing
```

PIM Sparse Mode on the relevant interfaces:

```text
pim ipv4 sparse-mode
```

and the RP:

```text
router pim sparse-mode
   ipv4
      rp address 10.10.100.1
```

cEOS2 additionally runs IGMP on the receiver-facing interface:

```text
interface Ethernet1
   ip igmp
```

Client1 also contains an explicit route for the test multicast group:

```text
239.1.1.1/32 dev eth1
```

This prevents the Containerlab management network from becoming the multicast egress path.

## Verification Commands

### Verify unicast routing

```bash
docker exec clab-multicast-l3-client1 ping -c 4 10.10.20.2
```

### Check OSPF

```text
show ip ospf neighbor
show ip route ospf
```

### Check IGMP

```text
show ip igmp interface Ethernet1
show ip igmp groups
```

### Check PIM interfaces

```text
show ip pim interface
```

### Check PIM neighbors

```text
show ip pim neighbor
```

### Check the multicast routing table

```text
show ip mroute
```

### Verify client multicast route

```bash
docker exec clab-multicast-l3-client1 ip route get 239.1.1.1
```

### Capture multicast traffic

```bash
docker exec clab-multicast-l3-client1 \
  tcpdump -ni eth1 'host 239.1.1.1 and udp port 5000'
```

### Capture IGMP

```bash
docker exec clab-multicast-l3-client2 \
  tcpdump -ni eth1 igmp
```

## Lessons Learned

- Working unicast routing does not imply working multicast routing.
- OSPF does not automatically forward multicast traffic.
- Multicast destinations represent groups rather than individual hosts.
- Linux may choose an unexpected egress interface for multicast if routing is not explicit.
- IGMP communicates receiver membership to the local multicast router.
- IGMPv3 membership reports were observed directly with tcpdump.
- IGMP alone does not provide router-to-router multicast forwarding.
- IPv4 multicast routing must be globally enabled on EOS.
- PIM provides multicast control-plane communication between routers.
- PIM Sparse Mode only builds forwarding state where receiver interest exists.
- PIM neighbors formed across the routed transit network.
- Sparse Mode uses a Rendezvous Point.
- Both routers must have consistent RP information.
- The multicast source and group can be represented as `(S,G)`.
- Group-wide wildcard state can be represented as `(*,G)`.
- RPF validates the correct upstream direction toward a multicast source.
- PIM can use OSPF-learned unicast routes for RPF.
- IGMP membership disappears when the receiver leaves the group.
- Multicast forwarding state changes dynamically as receiver interest changes.
- `show ip mroute` exposes both incoming and outgoing multicast interfaces.
- Packet capture is essential for distinguishing endpoint, routing, and multicast-control problems.
- Declarative Containerlab configuration should reproduce the multicast topology after rebuild.

## Result

Lab 04 successfully demonstrated multicast behavior across Layer 3.

Initially, end-to-end unicast routing worked while multicast traffic remained confined to the source side of the routed topology.

The multicast source was verified with packet capture, and the receiver's IGMPv3 membership reports were observed independently.

After enabling IPv4 multicast routing, IGMP, PIM Sparse Mode, PIM neighbor relationships, and a Rendezvous Point, cEOS built multicast forwarding state between the source and receiver networks.

Client2 successfully received multicast traffic originating from client1 across both cEOS routers.

The final multicast path is:

```text
client1 -> cEOS1 -> cEOS2 -> client2
```

The next lab will apply these multicast fundamentals specifically to mDNS and examine why mDNS service discovery normally stops at a Layer 3 boundary.
