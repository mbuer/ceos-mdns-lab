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
5 + 5   = 10 endpoints    PREPARED
10 + 10 = 20 endpoints    PLANNED
20 + 20 = 40 endpoints    PLANNED
40 + 40 = 80 endpoints    PLANNED
```

The 2x2 checkpoint has been validated and preserved.

The 5x5 checkpoint has been prepared and is ready for validation.

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

A checkpoint can therefore be deployed independently:

```bash
cd 02x02
sudo containerlab deploy -t topology.clab.yml
```

or:

```bash
cd 05x05
sudo containerlab deploy -t topology.clab.yml
```

## 2x2 Checkpoint

The first scale checkpoint uses two routed `/30` endpoint networks behind each
cEOS gateway.

```text
client1 -------- Et1  cEOS1  Et48 -------- Et48  cEOS2  Et1 -------- client2
10.10.10.2             |      10.10.100.0/30       |               10.10.20.2
                       |
client3 -------- Et2   |                           Et2 -------- client4
10.10.10.6                                         10.10.20.6
```

### Routing

OSPF runs across `Ethernet48` using an explicit point-to-point network type:

```text
ip ospf network point-to-point
```

The OSPF adjacency reached `FULL`, remote endpoint networks were learned, and
end-to-end routed connectivity was verified.

### mDNS Links

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

All four mDNS links became active.

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

The resulting cEOS1 service rule was:

```text
service test
   type any
   query Ethernet1, Ethernet2
   response interface Ethernet1, Ethernet2
   response link client2-link client4-link
```

The resulting cEOS2 service rule was:

```text
service test
   type any
   query Ethernet1, Ethernet2
   response interface Ethernet1, Ethernet2
   response link client1-link client3-link
```

This proves that one service rule can contain multiple local query interfaces,
multiple response interfaces, and multiple remote Link IDs.

## DNS-SD Validation

Two `_http._tcp.local` services were advertised behind cEOS2:

```text
Test Web 2._http._tcp.local -> client2.local -> 10.10.20.2:8080
Test Web 4._http._tcp.local -> client4.local -> 10.10.20.6:8080
```

cEOS2 learned both services.

Queries from both client1 and client3 successfully discovered both remote
services through the mDNS Gateway.

The gateway-generated responses contained the expected PTR, SRV, TXT, and A
records.

## 2x2 Result

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

## 5x5 Checkpoint

The next checkpoint expands the same architecture to five routed `/30` endpoint
networks behind each gateway.

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

The routed transit remains:

```text
cEOS1 Ethernet48  10.10.100.1/30
cEOS2 Ethernet48  10.10.100.2/30
```

The 5x5 topology and cEOS configurations are prepared but have not yet been
fully validated.

## Next Step

Deploy and validate the 5x5 checkpoint.

Validation will include:

- OSPF adjacency and routed reachability
- five active mDNS links per gateway
- DSO gateway connectivity
- DNS-SD service learning
- discovery from multiple endpoint networks
- expected versus discovered service count
- discovery latency
- cEOS CPU and memory usage
- clean destroy/redeploy behavior

Once 5x5 is proven, the same checkpoint structure can be extended to 10x10 and
beyond.
