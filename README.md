# cEOS mDNS Gateway Lab

A Containerlab-based learning and validation environment for testing Layer 3
routing, multicast behavior, multicast DNS (mDNS), and Arista EOS mDNS Gateway
functionality.

The project builds progressively from basic cEOS routing to routed multicast,
then demonstrates why mDNS does not cross Layer 3 boundaries normally and how
the Arista mDNS Gateway can extend DNS-SD service discovery between routed
subnets.

The final goal is validation with physical Riedel SmartPanel hardware.

## Lab Progression

- [x] Lab 01 - cEOS and Containerlab fundamentals
- [x] Lab 02 - Static Layer 3 routing
- [x] Lab 03 - OSPF
- [x] Lab 04 - Multicast Behavior Across Layer 3
- [x] Lab 05 - mDNS across Layer 3 boundaries
- [x] Lab 06 - Arista mDNS Gateway
- [ ] Lab 07 - Physical SmartPanel validation

## What Has Been Proven

### Layer 3 Routing

Two Linux endpoints on separate `/30` networks communicate through two cEOS
routers.

The routed path is:

```text
client1 -> cEOS1 -> cEOS2 -> client2
```

Lab 02 established static routing.

Lab 03 replaced the router-to-router static routes with OSPF and validated
dynamic route learning and convergence.

### Routed Multicast

Lab 04 demonstrated routed IPv4 multicast using:

- IGMPv3
- PIM Sparse Mode
- Rendezvous Point
- Reverse Path Forwarding
- multicast routing-table inspection
- tcpdump and Wireshark

Generic multicast traffic was successfully routed between the two endpoint
networks.

### mDNS Layer 3 Boundary

Lab 05 demonstrated that mDNS behaves differently from ordinary routed
multicast.

A valid query to:

```text
224.0.0.251:5353
```

was visible on the source network but was not forwarded across the first Layer
3 boundary.

This established the baseline:

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

### Arista mDNS Gateway

Lab 06 successfully extended DNS-SD service discovery across the routed
boundary using the Arista EOS mDNS Gateway.

A service advertised by client2:

```text
Test Web._http._tcp.local
```

was discovered from the client1 subnet.

The final packet capture showed:

```text
Query:
10.10.10.2 -> 224.0.0.251
PTR _http._tcp.local

Response:
10.10.10.1 -> 224.0.0.251
PTR Test Web._http._tcp.local
SRV client2.local:8080
TXT path=/
A 10.10.20.2
```

The response originates from cEOS1 rather than directly from client2. This
demonstrates that the gateway is not simply routing the original mDNS multicast
packet through the network.

The working implementation uses:

- local `mdns ipv4` links
- named mDNS Link IDs
- service rules
- remote gateway relationships
- DSO gateway peering
- remote `response link` policy

The complete Lab 06 configuration survived a full Containerlab destroy and
redeploy, and the end-to-end discovery test succeeded again afterward.

## Environment

- Ubuntu 24.04 Utility VM
- Docker
- Containerlab
- Arista cEOS 4.36.1F
- Linux test endpoints
- Windows Utility VM with Wireshark for packet analysis

## Addressing Convention

Each test endpoint receives its own `/30` network.

The lower usable address is reserved for the default gateway.

Example:

```text
10.10.10.0/30

10.10.10.1  cEOS gateway
10.10.10.2  endpoint
10.10.10.3  broadcast
```

The inter-router transit network is also a `/30`.

## Validation Method

Each lab follows the same evidence-based workflow:

```text
BUILD -> VERIFY -> CAPTURE -> DOCUMENT -> COMMIT -> PUSH
```

Validation uses:

- EOS operational commands
- Linux routing inspection
- ping
- tcpdump
- Wireshark
- full Containerlab destroy/redeploy testing

Generated Containerlab runtime directories and packet captures are intentionally
excluded from Git.

## Next Step

Lab 07 will move from simulated Linux endpoints to physical Riedel SmartPanel
hardware.

The target validation is:

```text
Physical SmartPanel
        |
        | mDNS / DNS-SD
        v
Arista routed network
        |
        | mDNS Gateway
        v
remote discovery domain
```

The goal is to confirm that the behavior proven with the virtual endpoints also
works with real SmartPanel service discovery.
