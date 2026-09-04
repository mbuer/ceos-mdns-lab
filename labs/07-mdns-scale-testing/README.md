# Lab 07 - mDNS Gateway Scale Testing

## Goal

Scale the proven Lab 06 Arista mDNS Gateway design while keeping the same two-gateway architecture.

## Current Status

```text
2 + 2   = 4 endpoints     COMPLETE
5 + 5   = 10 endpoints    COMPLETE
10 + 10 = 20 endpoints    COMPLETE
20 + 20 = 40 endpoints    COMPLETE
40 + 40 = 80 endpoints    COMPLETE
```

Lab 07 is complete.

```text
07-mdns-scale-testing/
├── README.md
├── 02x02/
├── 05x05/
├── 10x10/
├── 20x20/
└── 40x40/
```

## Key Configuration Finding

EOS list-based mDNS service commands must use `add` when extending an existing list. Repeating the same base command replaces the previous value.

```text
query Ethernet1
query add Ethernet2
```

## Checkpoint Summary

```text
2x2    PASS
5x5    PASS
10x10  PASS
20x20  PASS
40x40  PASS
```

## 5x5 Checkpoint

```text
5 DNS-SD services learned             PASS
client1 discovers all 5 services      PASS
client9 discovers all 5 services      PASS
complete PTR/SRV/TXT/A response       PASS
gateway response size 746 bytes       OBSERVED
gateway response delay ~2.1 seconds   OBSERVED
```

## 10x10 Checkpoint

The gateway response was split across two mDNS packets:

```text
Packet 1: 9 PTR answers, 1278 bytes
Packet 2: 1 PTR answer, 230 bytes
```

All ten services were returned with complete PTR/SRV/TXT/A records.

## 20x20 Checkpoint

Twenty routed `/30` endpoint networks were configured behind each gateway.

All twenty mDNS links became active on both gateways. Twenty `_http._tcp.local` services were advertised behind cEOS2 and learned on Ethernet1 through Ethernet20.

Remote discovery was validated from both client1 and client39.

The gateway response was split across three packets:

```text
Packet 1: 9 PTR answers, 1286 bytes
Packet 2: 9 PTR answers, 1286 bytes
Packet 3: 2 PTR answers, 358 bytes
```

All twenty services were returned with complete PTR, SRV, TXT, and A records.

A clean redeploy was also used for a cold-start burst test. All twenty advertisers were started together and the gateway moved from 0 to 20 learned services within the next one-second polling interval.

```text
20 DNS-SD services learned             PASS
client1 discovers all 20 services      PASS
client39 discovers all 20 services     PASS
complete PTR/SRV/TXT/A response        PASS
20-service cold-start burst            PASS
gateway response delay ~2.1 seconds    OBSERVED
```

## 40x40 Checkpoint

Forty routed `/30` endpoint networks were configured behind each gateway for a total of eighty simulated endpoint networks.

cEOS1 uses Ethernet1 through Ethernet40 for clients 1, 3, 5, ... 79.

cEOS2 uses Ethernet1 through Ethernet40 for clients 2, 4, 6, ... 80.

The endpoint addressing extends through `.158`, so the aggregate endpoint routes use `/24`.

The routed transit remains:

```text
cEOS1 Ethernet48  10.10.100.1/30
cEOS2 Ethernet48  10.10.100.2/30
```

### Routing and mDNS Links

OSPF reached `FULL` across Ethernet48 on both gateways.

All forty mDNS links became active on both gateways.

The service rule correctly included Ethernet1 through Ethernet40 and all forty remote Link IDs.

### DNS-SD Service Learning

Forty `_http._tcp.local` services were advertised behind cEOS2.

cEOS2 learned the service on Ethernet1 through Ethernet40 and mapped every `Test Web` instance to the expected local interface.

### Remote Discovery Validation

A query from client1 at `10.10.10.2:5353` received gateway-generated responses from `10.10.10.1:5353`.

All forty remote services were returned.

The response was split across five packets:

```text
Packet 1: 9 PTR answers, 1286 bytes
Packet 2: 9 PTR answers, 1286 bytes
Packet 3: 9 PTR answers, 1290 bytes
Packet 4: 9 PTR answers, 1286 bytes
Packet 5: 4 PTR answers, 626 bytes
```

Together, the five packets contained all forty PTR records and the associated SRV, TXT, and A records.

The first response arrived approximately 2.114 seconds after the query.

The capture reported zero packets dropped by the kernel.

### Resource Snapshot

With all forty services active, the Utility VM showed approximately:

```text
7.7 GiB RAM total
4.5 GiB RAM used
3.2 GiB RAM available
0 B swap used
```

Docker reported approximately:

```text
cEOS1 memory: 995 MiB
cEOS2 memory: 1019 MiB
```

EOS process snapshots showed approximately:

```text
cEOS1 CPU utilization: ~18%
cEOS2 CPU utilization: ~20%
```

The `McastDns` process used roughly 47 MiB resident memory on each cEOS node and showed no visible CPU pressure at the sampled moments.

These are cEOS/VM lab observations and should not be interpreted as hardware platform performance limits.

### 40x40 Result

```text
OSPF adjacency                         PASS
40 local mDNS links per gateway        PASS
40 remote mDNS links per gateway       PASS
40 DNS-SD services learned             PASS
client1 discovers all 40 services      PASS
complete PTR/SRV/TXT/A response        PASS
response split across five packets     OBSERVED
largest response packet 1290 bytes     OBSERVED
gateway response delay ~2.114 seconds  OBSERVED
VM swap usage 0 B                      OBSERVED
mDNS process under no visible pressure OBSERVED
```

The 40x40 functional and resource scale test is complete.

## Scaling Observations

```text
5 services   -> 1 mDNS response packet
10 services  -> 2 mDNS response packets
20 services  -> 3 mDNS response packets
40 services  -> 5 mDNS response packets
```

Across the 5-, 10-, 20-, and 40-service checkpoints, the first gateway response remained close to 2.1 seconds after the query.

No functional scaling failure was observed through forty remote services and eighty total routed endpoint domains in this simulated topology.

## Conclusion

Lab 07 demonstrates that the Arista mDNS Gateway mechanism continues to work cleanly as the two-gateway lab scales from two endpoint links per side to forty endpoint links per side.

The result is encouraging for larger routed SmartPanel deployments because the mDNS Gateway can preserve Layer 3 segmentation while returning complete remote DNS-SD service information.

The lab does not establish the supported production limit of a physical Arista platform and does not yet prove compatibility with the actual SmartPanel DNS-SD implementation.

## Next Step

Proceed to Lab 08 with physical Riedel SmartPanel hardware.

The physical validation should determine:

- the exact SmartPanel DNS-SD service type
- the exact PTR/SRV/TXT/A records used by SmartPanel
- whether SmartPanel WebUI bulk-firmware discovery works across routed `/30`s
- whether the production mDNS rule can be narrowed from `type any`
- behavior through the actual target Arista hardware and software version
