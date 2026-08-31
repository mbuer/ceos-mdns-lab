# cEOS mDNS Gateway Lab

A Containerlab-based learning and validation environment for testing Layer 3
routing, multicast DNS (mDNS), and Arista EOS mDNS gateway functionality.

The final goal is to validate mDNS-based service discovery between devices on
separate /30 networks and ultimately test the implementation with physical
SmartPanel hardware.

## Lab Progression

- [x] Lab 01 - cEOS and Containerlab fundamentals
- [x] Lab 02 - Static Layer 3 routing
- [x] Lab 03 - OSPF
- [ ] Lab 04 - Multicast Behavior Across Layer 3
- [ ] Lab 05 - mDNS across Layer 3 boundaries
- [ ] Lab 06 - Arista mDNS Gateway
- [ ] Lab 07 - Physical SmartPanel validation

## Current Status

Lab 02 introduced a dedicated router transit network and static routes between
the endpoint networks.

Lab 03 replaced those router-to-router static routes with OSPF. The cEOS
routers now form an OSPF adjacency across the transit network and dynamically
learn the remote endpoint networks.

End-to-end connectivity has been validated with ICMP, routing-table inspection,
tcpdump, and Wireshark.

The current routed path is:

    client1 -> cEOS1 -> cEOS2 -> client2

The next step is to examine multicast behavior across Layer 3 boundaries before
moving into mDNS-specific testing.

## Environment

- Ubuntu 24.04 Utility VM
- Docker
- Containerlab
- Arista cEOS 4.36.1F
- Linux test endpoints
- Windows Utility VM with Wireshark for packet analysis

## Addressing Convention

Each test device receives its own /30 network.

The lower usable address is reserved for the default gateway.

Example:

    10.10.10.0/30

    10.10.10.1  cEOS gateway
    10.10.10.2  endpoint
    10.10.10.3  broadcast

