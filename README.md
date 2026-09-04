# cEOS mDNS Gateway Lab

A Containerlab-based learning and validation environment for testing Layer 3 routing, multicast behavior, multicast DNS (mDNS), and Arista EOS mDNS Gateway functionality.

The project builds progressively from basic cEOS routing to routed multicast, then demonstrates why mDNS does not cross Layer 3 boundaries normally and how the Arista mDNS Gateway can extend DNS-SD service discovery between routed subnets.

The final goal is validation with physical Riedel SmartPanel hardware and an assessment of whether mDNS Gateway can support SmartPanel discovery in larger routed deployments.

## Lab Progression

- [x] Lab 01 - cEOS and Containerlab fundamentals
- [x] Lab 02 - Static Layer 3 routing
- [x] Lab 03 - OSPF
- [x] Lab 04 - Multicast Behavior Across Layer 3
- [x] Lab 05 - mDNS across Layer 3 boundaries
- [x] Lab 06 - Arista mDNS Gateway
- [x] Lab 07 - mDNS Gateway scale testing
- [ ] Lab 08 - Physical SmartPanel validation

## What Has Been Proven

### Layer 3 Routing

Two Linux endpoints on separate `/30` networks communicate through two cEOS routers.

```text
client1 -> cEOS1 -> cEOS2 -> client2
```

Lab 02 established static routing. Lab 03 replaced the router-to-router static routes with OSPF and validated dynamic route learning and convergence.

### Routed Multicast

Lab 04 demonstrated routed IPv4 multicast using IGMPv3, PIM Sparse Mode, a Rendezvous Point, Reverse Path Forwarding, multicast routing-table inspection, tcpdump, and Wireshark.

### mDNS Layer 3 Boundary

Lab 05 demonstrated that a valid mDNS query to `224.0.0.251:5353` remains local and is not forwarded across a Layer 3 boundary by normal multicast routing.

### Arista mDNS Gateway

Lab 06 successfully extended DNS-SD service discovery across the routed boundary using the Arista EOS mDNS Gateway.

A service advertised by client2:

```text
Test Web._http._tcp.local
```

was discovered from the client1 subnet. The gateway-generated response contained PTR, SRV, TXT, and A records for the remote service.

The working implementation uses local `mdns ipv4` participation, named mDNS Link IDs, service rules, remote gateway relationships, DSO gateway peering, and remote `response link` policy.

### Scale Testing

Lab 07 scaled the proven two-gateway design through:

```text
2 + 2   = 4 endpoints
5 + 5   = 10 endpoints
10 + 10 = 20 endpoints
20 + 20 = 40 endpoints
40 + 40 = 80 endpoints
```

At the largest checkpoint, each cEOS gateway had forty routed `/30` endpoint networks and forty active mDNS Link IDs.

Forty DNS-SD services were advertised behind cEOS2 and learned successfully on Ethernet1 through Ethernet40. A query from client1 returned all forty remote services with complete PTR, SRV, TXT, and A records.

The response was segmented across five mDNS packets:

```text
9 + 9 + 9 + 9 + 4 PTR answers
```

The first gateway response arrived approximately 2.114 seconds after the query.

Across the tested checkpoints, response delay remained close to 2.1 seconds while EOS segmented larger result sets across more response packets.

At the 40x40 checkpoint:

```text
80 endpoint containers
2 cEOS containers
0 B swap used
~3.2 GiB VM memory available
~47 MiB resident memory for McastDns per cEOS node
```

No functional scaling failure was observed through forty remote services and eighty total routed endpoint domains in the simulated lab.

These cEOS/VM observations are not hardware platform scale guarantees.

## Environment

- Ubuntu 24.04 Utility VM
- Docker
- Containerlab
- Arista cEOS 4.36.1F
- Linux test endpoints
- Windows Utility VM with Wireshark for packet analysis

## Addressing Convention

Each test endpoint receives its own `/30` network. The lower usable address is reserved for the default gateway.

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

Validation uses EOS operational commands, Linux routing inspection, ping, tcpdump, Wireshark, and full Containerlab destroy/redeploy testing.

## SmartPanel Application

A practical production use case is a SmartPanel deployment where many panels reside on separate routed `/30` networks.

SmartPanel uses mDNS only for **service discovery**. The actual firmware transfer remains normal unicast IP communication.

An mDNS Gateway can potentially preserve the routed `/30` architecture while extending only the required service-discovery information between those networks.

## Production Considerations

The lab configuration intentionally uses:

```text
type any
```

A production implementation should identify the exact DNS-SD service type used by SmartPanel and permit only the required discovery traffic.

Before recommending this architecture for hundreds of SmartPanel `/30` networks, the following still needs to be validated:

- actual SmartPanel bulk-firmware discovery behavior
- exact SmartPanel DNS-SD service type
- supported number of mDNS links on the target Arista platform
- supported number of learned DNS-SD service records
- DSO / mDNS Gateway scale limits
- behavior during large simultaneous discovery events
- convergence and recovery behavior during gateway or routing failure
- operational visibility and troubleshooting at production scale

The virtual lab proves the mechanism and shows encouraging behavior through eighty simulated routed endpoint domains.

It does **not yet prove production-scale SmartPanel compatibility**.

## Next Step

Lab 08 moves from simulated Linux endpoints to physical Riedel SmartPanel hardware.

The primary acceptance test will use the real SmartPanel WebUI bulk firmware workflow:

```text
1. Place SmartPanels on separate routed /30 networks.
2. Enable the required Arista mDNS Gateway policy.
3. Open the SmartPanel bulk firmware interface.
4. Verify whether SmartPanels on remote /30 networks are discovered.
5. Capture the discovery exchange with tcpdump and Wireshark.
6. Identify the exact DNS-SD service type and records used by SmartPanel.
7. Verify behavior after a full network configuration reload or lab redeploy.
```

If this succeeds, the behavior proven virtually in Labs 05 through 07 will have been connected directly to the real SmartPanel production use case.
