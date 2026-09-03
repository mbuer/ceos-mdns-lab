# Lab 07 - mDNS Gateway Scale Testing

## Goal

Scale the proven Lab 06 Arista mDNS Gateway design while keeping the same
two-gateway architecture.

The purpose of this lab is to determine how the configuration and discovery
behavior scale as more routed `/30` mDNS domains are added.

Planned progression:

```text
2 + 2   = 4 endpoints
5 + 5   = 10 endpoints
10 + 10 = 20 endpoints
20 + 20 = 40 endpoints
40 + 40 = 80 endpoints
```

Each endpoint represents a potential SmartPanel discovery domain.

## Current Status

```text
2 + 2   = 4 endpoints     COMPLETE
5 + 5   = 10 endpoints    COMPLETE
10 + 10 = 20 endpoints    NEXT
20 + 20 = 40 endpoints    PLANNED
40 + 40 = 80 endpoints    PLANNED
```

The 2x2 and 5x5 checkpoints have been validated and preserved.

## Checkpoint Structure

Each scale stage is self-contained with its own Containerlab topology and cEOS
configuration.

```text
07-mdns-scale-testing/
├── README.md
├── 02x02/
│   ├── topology.clab.yml
│   └── config/
│       ├── ceos1.cfg
│       └── ceos2.cfg
└── 05x05/
    ├── topology.clab.yml
    └── config/
        ├── ceos1.cfg
        └── ceos2.cfg
```

## 2x2 Checkpoint

The first scale checkpoint used two routed `/30` endpoint networks behind each
cEOS gateway.

```text
client1 -------- Et1  cEOS1  Et48 -------- Et48  cEOS2  Et1 -------- client2
10.10.10.2             |      10.10.100.0/30       |               10.10.20.2
                       |
client3 -------- Et2   |                           Et2 -------- client4
10.10.10.6                                         10.10.20.6
```

### Result

```text
OSPF adjacency                        PASS
Layer 3 endpoint routing              PASS
2 local mDNS links per gateway        PASS
2 remote mDNS links per gateway       PASS
DSO gateway connection                PASS
DNS-SD service learning               PASS
client1 remote discovery              PASS
client3 remote discovery              PASS
destroy/redeploy reproducibility      PASS
```

## Multi-Link Service Policy

An important scaling behavior was discovered during the 2x2 test.

Repeating commands such as:

```text
query Ethernet1
query Ethernet2
```

does not append the second interface. The later command replaces the previous
value.

EOS provides explicit list operations using `add`.

The working configuration was built using commands such as:

```text
query Ethernet1
query add Ethernet2
```

This behavior is important when scaling one mDNS service rule across many
routed interfaces.

## 5x5 Checkpoint

The second checkpoint expanded the same architecture to five routed `/30`
endpoint networks behind each gateway.

cEOS1:

```text
Et1  10.10.10.1/30   client1  10.10.10.2
Et2  10.10.10.5/30   client3  10.10.10.6
Et3  10.10.10.9/30   client5  10.10.10.10
Et4  10.10.10.13/30  client7  10.10.10.14
Et5  10.10.10.17/30  client9  10.10.10.18
```

cEOS2:

```text
Et1  10.10.20.1/30   client2   10.10.20.2
Et2  10.10.20.5/30   client4   10.10.20.6
Et3  10.10.20.9/30   client6   10.10.20.10
Et4  10.10.20.13/30  client8   10.10.20.14
Et5  10.10.20.17/30  client10  10.10.20.18
```

The routed transit remained:

```text
cEOS1 Ethernet48  10.10.100.1/30
cEOS2 Ethernet48  10.10.100.2/30
```

### Routing Validation

OSPF reached `FULL` across Ethernet48.

Five mDNS links became active on each gateway.

Routed reachability was verified across multiple endpoint combinations,
including edge-to-edge and middle-to-middle tests.

### DNS-SD Service Learning

Five `_http._tcp.local` services were advertised behind cEOS2:

```text
Test Web 2  -> client2.local  -> 10.10.20.2:8080
Test Web 4  -> client4.local  -> 10.10.20.6:8080
Test Web 6  -> client6.local  -> 10.10.20.10:8080
Test Web 8  -> client8.local  -> 10.10.20.14:8080
Test Web 10 -> client10.local -> 10.10.20.18:8080
```

cEOS2 learned `_http._tcp.` independently on Ethernet1 through Ethernet5 and
associated each advertised service with the correct local interface.

### Remote Discovery Validation

Remote discovery was tested from client1 and client9 behind cEOS1.

client1 queried from:

```text
10.10.10.2:5353
```

and received a gateway-generated multicast response from:

```text
10.10.10.1:5353
```

client9 queried from:

```text
10.10.10.18:5353
```

and received a gateway-generated multicast response from:

```text
10.10.10.17:5353
```

Both responses contained all five remote services.

The response contained:

```text
5 PTR records
5 SRV records
5 A records
5 TXT records
Arista gateway TXT record
```

The observed packet size was 746 bytes.

The gateway response arrived approximately 2.1 seconds after the query in both
captures.

### 5x5 Result

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

The 5x5 functional scale test is complete.

## Next Step

Build the 10x10 checkpoint:

```text
10 endpoints behind cEOS1
10 endpoints behind cEOS2

20 total routed mDNS discovery domains
```

The next test will repeat the same validation pattern while observing:

- OSPF routing
- active mDNS Link IDs
- DNS-SD service learning
- discovery completeness
- response size
- response latency
- cEOS CPU and memory usage
- configuration growth
- clean destroy/redeploy behavior
