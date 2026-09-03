# cEOS mDNS Gateway Lab

A Containerlab-based learning and validation environment for testing Layer 3
routing, multicast behavior, multicast DNS (mDNS), and Arista EOS mDNS Gateway
functionality.

The project builds progressively from basic cEOS routing to routed multicast,
then demonstrates why mDNS does not cross Layer 3 boundaries normally and how
the Arista mDNS Gateway can extend DNS-SD service discovery between routed
subnets.

The final goal is validation with physical Riedel SmartPanel hardware and an
assessment of whether mDNS Gateway can support SmartPanel discovery in larger
routed deployments.

## Lab Progression

- [x] Lab 01 - cEOS and Containerlab fundamentals
- [x] Lab 02 - Static Layer 3 routing
- [x] Lab 03 - OSPF
- [x] Lab 04 - Multicast Behavior Across Layer 3
- [x] Lab 05 - mDNS across Layer 3 boundaries
- [x] Lab 06 - Arista mDNS Gateway
- [ ] Lab 07 - mDNS Gateway scale testing
- [ ] Lab 08 - Physical SmartPanel validation

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

The result confirmed that normal Layer 3 multicast routing does not, by itself,
extend mDNS discovery between routed subnets.

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

The response originates from cEOS1 rather than directly from client2.

This demonstrates that the gateway is not simply routing the original mDNS
multicast packet through the network. Instead, the local gateway generates a
new mDNS response containing service information learned from the remote side.

The working implementation uses:

- local `mdns ipv4` participation
- named mDNS Link IDs
- service rules
- remote gateway relationships
- DSO gateway peering
- remote `response link` policy

The Link ID is more than a descriptive interface label. EOS advertises the link
identity to remote mDNS gateways so that service rules can reference remote
discovery domains by name.

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

This addressing model is relevant to the SmartPanel use case because larger
systems may place individual SmartPanels on separate routed `/30` networks.

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

Configuration is considered complete only after the intended behavior can be
observed and reproduced.

Generated Containerlab runtime directories and packet captures are intentionally
excluded from Git.

## SmartPanel Application

A practical production use case for this architecture is a SmartPanel
deployment where many panels reside on separate routed `/30` networks.

SmartPanel uses mDNS only for **service discovery**.

Without an mDNS Gateway, each `/30` represents a separate local mDNS discovery
domain. SmartPanels on different routed subnets therefore cannot discover one
another through normal mDNS.

Conceptually:

```text
SmartPanel 1 /30
       X

SmartPanel 2 /30
       X

SmartPanel 3 /30
       X

SmartPanel N /30
```

An mDNS Gateway can potentially preserve the routed `/30` architecture while
extending only the required service-discovery information between those
networks.

```text
SmartPanel 1 /30 ──┐
SmartPanel 2 /30 ──┤
SmartPanel 3 /30 ──┤
                   ├── Arista mDNS Gateway
SmartPanel 4 /30 ──┤
SmartPanel 5 /30 ──┤
SmartPanel N /30 ──┘
```

This is particularly relevant to the SmartPanel bulk firmware workflow.

The two functions are separate:

```text
mDNS / DNS-SD
      |
      v
service discovery only

then

normal unicast IP communication
      |
      v
firmware transfer
```

The mDNS Gateway is therefore responsible only for making SmartPanels
discoverable across routed Layer 3 boundaries.

It does not carry the firmware image itself.

This makes the architecture attractive for larger SmartPanel deployments
because it can preserve Layer 3 segmentation while restoring the discovery
behavior required by the SmartPanel WebUI.

## Production Considerations

The Lab 06 configuration was intentionally broad in order to prove the feature.

For example:

```text
type any
```

is appropriate for a controlled lab but would likely be too permissive for a
large production environment.

A production implementation should identify the exact DNS-SD service type used
by SmartPanel and permit only the required discovery traffic.

Conceptually:

```text
service smartpanel
   type <SmartPanel DNS-SD service type>
```

This would prevent unrelated mDNS services from unnecessarily participating in
the extended discovery domain.

Before recommending this architecture for a deployment containing hundreds of
SmartPanel `/30` networks, the following should be validated:

- actual SmartPanel bulk-firmware discovery behavior
- exact SmartPanel DNS-SD service type
- supported number of mDNS links on the target Arista platform
- supported number of learned DNS-SD service records
- DSO / mDNS Gateway scale limits
- behavior during large simultaneous discovery events
- convergence and recovery behavior during gateway failure
- convergence and recovery behavior during routing failure
- operational visibility and troubleshooting at production scale

The virtual lab proves the mDNS Gateway mechanism.

It does **not yet prove production-scale SmartPanel compatibility**.

## Why This Architecture Is Interesting

If physical validation succeeds, the design could provide a useful combination:

```text
Layer 3 isolation per SmartPanel
            +
controlled cross-subnet discovery
            +
normal routed IP communication
```

This avoids extending a large Layer 2 broadcast domain simply to support mDNS
discovery.

The network can remain routed while the SmartPanel application receives the
discovery behavior it requires.

For systems containing hundreds of routed SmartPanel networks, this could be a
much cleaner architecture than flattening the network solely for service
discovery.

## Next Steps

Lab 07 scales the proven two-gateway mDNS design before introducing physical
SmartPanels.

The scale progression is:

```text
2 + 2   = 4 endpoints
5 + 5   = 10 endpoints
10 + 10 = 20 endpoints
20 + 20 = 40 endpoints
40 + 40 = 80 endpoints
```

Lab 08 will move from simulated Linux endpoints to physical Riedel SmartPanel
hardware.

The target validation is:

```text
Physical SmartPanel A
        |
       /30
        |
        v
Arista routed network
        |
        | mDNS Gateway
        |
        v
       /30
        |
Physical SmartPanel B
```

The objective is to determine whether SmartPanel B becomes visible through the
actual SmartPanel discovery mechanism when the devices reside on separate
routed networks.

The primary acceptance test will use the real SmartPanel WebUI bulk firmware
workflow:

```text
1. Place SmartPanels on separate routed /30 networks.

2. Enable the required Arista mDNS Gateway policy.

3. Open the SmartPanel bulk firmware interface.

4. Verify whether SmartPanels on remote /30 networks are discovered.

5. Capture the discovery exchange with tcpdump and Wireshark.

6. Identify the exact DNS-SD service type and records used by SmartPanel.

7. Verify that discovery continues to work after a full network
   configuration reload or lab redeploy.
```

If this succeeds, the behavior proven virtually in Labs 05 and 06 will have
been connected directly to the real SmartPanel production use case.
