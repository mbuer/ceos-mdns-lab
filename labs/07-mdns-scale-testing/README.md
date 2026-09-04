# Lab 07 - mDNS Gateway Scale Testing

## Goal

Scale the proven Lab 06 Arista mDNS Gateway design while keeping the same
two-gateway architecture.

The purpose of this lab is to determine how configuration and discovery
behavior scale as more routed `/30` mDNS domains are added.

## Current Status

```text
2 + 2   = 4 endpoints     COMPLETE
5 + 5   = 10 endpoints    COMPLETE
10 + 10 = 20 endpoints    COMPLETE
20 + 20 = 40 endpoints    NEXT
40 + 40 = 80 endpoints    PLANNED
```

Each checkpoint is self-contained with its own topology and cEOS configs.

```text
07-mdns-mdns-scale-testing/
├── README.md
├── 02x02/
├── 05x05/
└── 10x10/
```

## Key Configuration Finding

EOS list-based mDNS service commands must use `add` when extending an existing
list. Repeating the same base command replaces the previous value.

Example:

```text
query Ethernet1
query add Ethernet2
```

This allows one service rule to scale across multiple local interfaces and
remote Link IDs.

## 2x2 Checkpoint

```text
OSPF adjacency                        PASS
Layer 3 endpoint routing              PASS
2 local mDNS links per gateway        PASS
2 remote mDNS links per gateway       PASS
DSO gateway connection                PASS
DNS-SD service learning               PASS
remote discovery                      PASS
destroy/redeploy reproducibility      PASS
```

## 5x5 Checkpoint

Five routed `/30` endpoint networks were configured behind each gateway.

```text
OSPF adjacency                        PASS
Layer 3 endpoint routing              PASS
5 local mDNS links per gateway        PASS
5 remote mDNS links per gateway       PASS
5 DNS-SD services learned             PASS
client1 discovers all 5 services      PASS
client9 discovers all 5 services      PASS
complete PTR/SRV/TXT/A response       PASS
gateway response size 746 bytes       OBSERVED
gateway response delay ~2.1 seconds   OBSERVED
```

## 10x10 Checkpoint

Ten routed `/30` endpoint networks were configured behind each gateway.

cEOS1 uses Ethernet1 through Ethernet10 for clients 1, 3, 5, 7, 9, 11, 13,
15, 17, and 19.

cEOS2 uses Ethernet1 through Ethernet10 for clients 2, 4, 6, 8, 10, 12, 14,
16, 18, and 20.

The routed transit remains:

```text
cEOS1 Ethernet48  10.10.100.1/30
cEOS2 Ethernet48  10.10.100.2/30
```

### Routing and mDNS Links

OSPF reached `FULL` across Ethernet48.

All ten mDNS links became active on both gateways.

The service rule correctly included Ethernet1 through Ethernet10 and all ten
remote Link IDs.

Routed reachability was verified across multiple endpoint combinations.

### DNS-SD Service Learning

Ten `_http._tcp.local` services were advertised behind cEOS2:

```text
Test Web 2
Test Web 4
Test Web 6
Test Web 8
Test Web 10
Test Web 12
Test Web 14
Test Web 16
Test Web 18
Test Web 20
```

cEOS2 learned the service independently on Ethernet1 through Ethernet10 and
associated each instance with the correct interface.

### Remote Discovery Validation

Remote discovery was tested from both ends of the cEOS1 side.

client1 queried from:

```text
10.10.10.2:5353
```

and received gateway-generated responses from:

```text
10.10.10.1:5353
```

client19 queried from:

```text
10.10.10.38:5353
```

and received gateway-generated responses from:

```text
10.10.10.37:5353
```

Both clients discovered all ten remote services.

At this scale, the gateway response was split across two mDNS packets:

```text
Packet 1:
9 PTR answers
1278 bytes

Packet 2:
1 PTR answer
230 bytes
```

The second packet contained `Test Web 8`.

Together, the packets contained the complete PTR, SRV, TXT, and A record set
for all ten advertised services.

The observed gateway response delay remained approximately 2.1 seconds.

### 10x10 Result

```text
OSPF adjacency                         PASS
Layer 3 endpoint routing               PASS
10 local mDNS links per gateway        PASS
10 remote mDNS links per gateway       PASS
10 DNS-SD services learned             PASS
client1 discovers all 10 services      PASS
client19 discovers all 10 services     PASS
complete PTR/SRV/TXT/A response        PASS
response split across two packets      OBSERVED
largest response packet 1278 bytes     OBSERVED
second response packet 230 bytes       OBSERVED
gateway response delay ~2.1 seconds    OBSERVED
```

The 10x10 functional scale test is complete.

## Next Step

Build the 20x20 checkpoint:

```text
20 endpoints behind cEOS1
20 endpoints behind cEOS2

40 total routed mDNS discovery domains
```

The next test will repeat the same validation pattern while observing:

- OSPF routing
- active mDNS Link IDs
- DNS-SD service learning
- discovery completeness
- response packet count
- response size
- response latency
- cEOS CPU and memory usage
- configuration growth
- clean destroy/redeploy behavior
