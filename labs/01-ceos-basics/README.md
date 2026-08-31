# Lab 01 - cEOS Basics

## Objective

Build a reproducible Containerlab environment with one Arista cEOS router and
two Linux endpoints on separate /30 networks.

The lab introduces:

- Containerlab and cEOS lifecycle
- management plane vs. lab data plane
- routed EOS interfaces
- IPv4 forwarding
- Linux static routes
- packet capture and Wireshark validation

## Topology

    client1                         cEOS1                         client2
    10.10.10.2/30          Ethernet1     Ethernet2        10.10.20.2/30
         |                  10.10.10.1   10.10.20.1             |
         +----------------------|             |------------------+

Each endpoint is placed on its own /30 network.

### Network 1

    10.10.10.0/30

    10.10.10.0  network
    10.10.10.1  cEOS Ethernet1 / gateway
    10.10.10.2  client1
    10.10.10.3  broadcast

### Network 2

    10.10.20.0/30

    10.10.20.0  network
    10.10.20.1  cEOS Ethernet2 / gateway
    10.10.20.2  client2
    10.10.20.3  broadcast

The lab convention is to use the lowest usable address as the gateway.

## Management vs. Data Plane

Containerlab creates a separate Docker management network on `172.20.20.0/24`.

For the Linux endpoints:

- `eth0` = Containerlab management
- `eth1` = lab traffic

For cEOS:

- `Management0` = Containerlab management
- `Ethernet1` and `Ethernet2` = lab traffic

The `172.20.20.x` management addresses are dynamically assigned and may change
between deployments. They are therefore not hard-coded into the lab
configuration.

The Linux management default route is preserved. Explicit routes send only the
lab traffic through cEOS.

## cEOS Layer 3 Configuration

EOS interfaces are switchports by default. They must first be converted to
routed interfaces:

    interface Ethernet1
       no switchport
       ip address 10.10.10.1/30

    interface Ethernet2
       no switchport
       ip address 10.10.20.1/30

Configuring routed interfaces creates connected routes, but IPv4 forwarding
also requires:

    ip routing

Without `ip routing`, both connected networks appeared in `show ip route`, but
cEOS did not forward packets between them.

## Endpoint Routing

Containerlab uses `eth0` for endpoint management, so its existing default route
is left untouched.

Instead, each endpoint receives an explicit route to the remote /30:

    client1:
    10.10.20.0/30 via 10.10.10.1

    client2:
    10.10.10.0/30 via 10.10.20.1

These commands are defined using Containerlab `exec` entries in
`topology.clab.yml`.

## Reproducible Deployment

The cEOS lab configuration is stored in:

    config/ceos1.cfg

The endpoint configuration is stored directly in:

    topology.clab.yml

A clean deployment can therefore reconstruct the complete routed lab without
manual IP configuration.

A normal clean deployment reads the version-controlled startup configuration:

    sudo containerlab destroy -t topology.clab.yml
    sudo containerlab deploy -t topology.clab.yml

When an existing lab must be rebuilt and its configuration forcibly reapplied,
Containerlab's `--reconfigure` option can be used.

The generated `clab-ceos-basics/` directory contains runtime state and is
excluded from Git.

cEOS should be managed through Containerlab rather than independently starting
its Docker container.

## Verification

After a clean deployment, client1 successfully reached client2:

    10.10.10.2 -> cEOS -> 10.10.20.2

Test command:

    docker exec clab-ceos-basics-client1 ping -c 4 10.10.20.2

The replies arrived with TTL 63, confirming that the packets crossed a Layer 3
hop.

Interface counters on `client1:eth1` also increased by four transmitted and
four received packets during a four-packet ping test.

## Packet Capture

Traffic was captured directly on the client data-plane interface:

    docker exec clab-ceos-basics-client1 sh -c '
    tcpdump -ni eth1 -c 8 icmp &
    sleep 1
    ping -c 4 10.10.20.2
    wait
    '

The capture showed:

    10.10.10.2 > 10.10.20.2  ICMP echo request
    10.10.20.2 > 10.10.10.2  ICMP echo reply

A PCAP was copied to the Windows Utility VM and inspected with Wireshark.

This demonstrated an important Layer 3 concept: the IP destination remains the
remote endpoint, while the Ethernet destination on the first subnet is the MAC
address of the cEOS next-hop gateway.

PCAP files are excluded from Git.

## Useful Commands

EOS:

    show interfaces status
    show ip interface brief
    show ip route
    show running-config
    show startup-config

Linux endpoint:

    ip -br addr
    ip route
    ip route get <destination>
    ip neigh
    ip -s link

## Lessons Learned

1. Containerlab management addressing is separate from lab addressing.
2. EOS switchports require `no switchport` before assigning routed IPs.
3. `ip routing` is required for forwarding between routed interfaces.
4. Linux endpoint management routes should remain separate from lab routes.
5. Containerlab should manage the lifecycle of cEOS containers.
6. Runtime configuration disappears when containers are recreated unless it is
   represented declaratively.
7. A version-controlled `startup-config` makes the cEOS configuration
   reproducible across clean deployments.
8. Packet capture provides independent proof of the forwarding path.

## Result

Lab 01 is complete.

A clean Containerlab deployment creates two /30 endpoint networks, configures
cEOS as the Layer 3 gateway, installs the required endpoint routes, and provides
working end-to-end connectivity without manual network configuration.

The next lab will extend this foundation to multiple cEOS routers and static
routing between them.
