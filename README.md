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
- [x] Lab 04 - Multicast Behavior Across Layer 3
- [ ] Lab 05 - mDNS across Layer 3 boundaries
- [ ] Lab 06 - Arista mDNS Gateway
- [ ] Lab 07 - Physical SmartPanel validation

## Current Status

The lab now provides reproducible unicast and multicast connectivity between
Linux endpoints connected through two cEOS routers.

Lab 02 introduced static Layer 3 routing between the endpoint networks.

Lab 03 replaced the router-to-router static routes with OSPF. The cEOS routers
form an OSPF adjacency across the transit network and dynamically learn the
remote endpoint networks.

Lab 04 introduced routed IPv4 multicast. Multicast behavior was validated using
IGMPv3 receiver membership, PIM Sparse Mode, a Rendezvous Point, Reverse Path
Forwarding, multicast routing-table inspection, and packet capture.

End-to-end multicast traffic from client1 to client2 was successfully verified
after a full Containerlab destroy and redeploy.

The current routed path is:

    client1 -> cEOS1 -> cEOS2 -> client2

The next step is to examine mDNS specifically and prove how mDNS discovery
behaves across Layer 3 boundaries before introducing the Arista mDNS Gateway.

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

