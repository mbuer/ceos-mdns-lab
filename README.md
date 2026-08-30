# cEOS mDNS Gateway Lab

A Containerlab-based learning and validation environment for testing Layer 3
routing, multicast DNS (mDNS), and Arista EOS mDNS gateway functionality.

The final goal is to validate mDNS-based service discovery between devices on
separate /30 networks and ultimately test the implementation with physical
SmartPanel hardware.

## Lab Progression

1. cEOS and Containerlab fundamentals
2. Static Layer 3 routing
3. OSPF
4. Multicast and mDNS fundamentals
5. mDNS across Layer 3 boundaries
6. Arista mDNS Gateway
7. Physical SmartPanel validation

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
