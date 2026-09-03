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

**2×2 checkpoint completed successfully**

```text
client1 -------- Et1  cEOS1  Et48 -------- Et48  cEOS2  Et1 -------- client2
10.10.10.2             |      10.10.100.0/30       |               10.10.20.2
                       |
client3 -------- Et2   |                           Et2 -------- client4
10.10.10.6                                         10.10.20.6
```

Endpoint networks:

```text
cEOS1 side

10.10.10.0/30
  gateway: 10.10.10.1
  client1: 10.10.10.2

10.10.10.4/30
  gateway: 10.10.10.5
  client3: 10.10.10.6
```

```text
cEOS2 side

10.10.20.0/30
  gateway: 10.10.20.1
  client2: 10.10.20.2

10.10.20.4/30
  gateway: 10.10.20.5
  client4: 10.10.20.6
```

The inter-switch routed transit uses:

```text
cEOS1 Ethernet48: 10.10.100.1/30
cEOS2 Ethernet48: 10.10.100.2/30
```

## Routing

OSPF runs across `Ethernet48`.

The transit interface is explicitly configured as:

```text
ip ospf network point-to-point
```

This avoids unnecessary DR/BDR behavior on the direct router-to-router `/30`.

The OSPF adjacency reached:

```text
FULL
```

and both remote endpoint networks were learned dynamically.

End-to-end routing was verified:

```text
client1 -> client2   PASS
client3 -> client4   PASS
```

## mDNS Links

cEOS1:

```text
Ethernet1 -> client1-link
Ethernet2 -> client3-link
```

cEOS2:

```text
Ethernet1 -> client2-link
Ethernet2 -> client4-link
```

All four links were operationally reported as:

```text
active
```

## Multi-Link Service Policy

An important scaling behavior was discovered during this lab.

Repeating:

```text
query Ethernet1
query Ethernet2
```

does not append a second entry. The later command replaces the first.

EOS provides explicit list operations:

```text
add
remove
```

The working configuration was therefore built using:

```text
query Ethernet1
query add Ethernet2
```

and equivalent commands for response interfaces and remote links.

The resulting running configuration on cEOS1 is:

```text
service test
   type any
   query Ethernet1, Ethernet2
   response interface Ethernet1, Ethernet2
   response link client2-link client4-link
```

cEOS2:

```text
service test
   type any
   query Ethernet1, Ethernet2
   response interface Ethernet1, Ethernet2
   response link client1-link client3-link
```

This is an important result because one service rule can contain multiple local
interfaces and multiple remote mDNS links.

## DNS-SD Test Services

Two `_http._tcp.local` services were advertised behind cEOS2.

client2:

```text
Test Web 2._http._tcp.local
client2.local
10.10.20.2
TCP 8080
TXT path=/
```

client4:

```text
Test Web 4._http._tcp.local
client4.local
10.10.20.6
TCP 8080
TXT path=/
```

cEOS2 learned both services:

```text
Test Web 2._http._tcp.local -> Ethernet1
Test Web 4._http._tcp.local -> Ethernet2
```

## Remote Discovery Validation

### client1

client1 queried:

```text
PTR _http._tcp.local
```

and received a gateway-generated response from:

```text
10.10.10.1:5353
```

containing both remote services:

```text
Test Web 2._http._tcp.local
Test Web 4._http._tcp.local
```

with:

```text
client2.local -> 10.10.20.2:8080
client4.local -> 10.10.20.6:8080
```

### client3

The same query was sent from client3.

client3 received the response from its local gateway:

```text
10.10.10.5:5353
```

and again received both remote service records.

## 2×2 Result

The first scale checkpoint proves:

```text
2 local mDNS links per gateway        PASS
2 remote mDNS links per gateway       PASS
2 DNS-SD services learned             PASS
client1 discovers both services       PASS
client3 discovers both services       PASS
DSO gateway relationship connected    PASS
OSPF routing operational              PASS
```

## Configuration Snapshots

Each scale stage stores its validated cEOS configuration separately.

```text
config/
├── 02x02/
│   ├── ceos1.cfg
│   └── ceos2.cfg
├── 05x05/
├── 10x10/
├── 20x20/
└── 40x40/
```

This prevents later scale stages from overwriting previously validated
configurations.

The topology file currently references:

```text
config/02x02/ceos1.cfg
config/02x02/ceos2.cfg
```

## Next Step

Scale the same architecture to:

```text
5 endpoints behind cEOS1
5 endpoints behind cEOS2

10 total mDNS discovery domains
```

The next checkpoint will verify:

- all ten `/30` endpoint networks
- all ten mDNS Link IDs
- multi-link service policy
- all expected DNS-SD services discovered
- discovery completeness
- discovery latency
- cEOS mDNS / DSO counters
- VM and container resource usage
