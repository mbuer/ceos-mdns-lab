# Lab 02 - Static Routing

## Objective

Extend the Lab 01 topology to two Arista cEOS routers and learn how static
routes provide connectivity between networks that are not directly connected.

The lab demonstrates:

- directly connected vs. static routes
- router-to-router transit networks
- next-hop routing
- forward and return paths
- Linux endpoint routing
- two-hop packet forwarding
- packet capture and Wireshark validation

## Topology

    client1              cEOS1                 cEOS2              client2
    10.10.10.2/30        Et1                   Et1                10.10.20.2/30
         |               10.10.10.1           10.10.20.1              |
         |                    |                     |                   |
         +--------------------+                     +-------------------+
                              |                     |
                             Et2                   Et2
                         10.10.100.1 -------- 10.10.100.2
                              10.10.100.0/30

Three /30 networks are used.

### Client 1 Network

    10.10.10.0/30

    10.10.10.1  cEOS1 Ethernet1 / gateway
    10.10.10.2  client1

### Router Transit Network

    10.10.100.0/30

    10.10.100.1  cEOS1 Ethernet2
    10.10.100.2  cEOS2 Ethernet2

### Client 2 Network

    10.10.20.0/30

    10.10.20.1  cEOS2 Ethernet1 / gateway
    10.10.20.2  client2

## Initial Routing State

After configuring only interface addresses, each cEOS router automatically
learned its directly connected networks.

cEOS1:

    C  10.10.10.0/30
    C  10.10.100.0/30

cEOS2:

    C  10.10.20.0/30
    C  10.10.100.0/30

The routers could communicate across the transit network because
`10.10.100.0/30` was directly connected to both routers.

Neither router initially knew how to reach the endpoint network behind the
other router.

## Endpoint Routing

The Containerlab management network remains separate from the lab data plane.

Without an explicit route, client1 attempted to reach `10.10.20.2` using its
management default route:

    10.10.20.2 via 172.20.20.1 dev eth0

The correct route is:

    client1:
    10.10.20.0/30 via 10.10.10.1

Likewise, client2 requires:

    client2:
    10.10.10.0/30 via 10.10.20.1

These routes send inter-network traffic through the cEOS routers instead of
the Containerlab management network.

## Static Routes

cEOS1 requires a route to the network behind cEOS2:

    ip route 10.10.20.0/30 10.10.100.2

This means:

    Destination network: 10.10.20.0/30
    Next hop:             10.10.100.2

cEOS2 requires the reverse route:

    ip route 10.10.10.0/30 10.10.100.1

This provides a complete bidirectional routing path.

## Forward and Return Path

Successful communication requires routing information in both directions.

Forward path:

    client1
       |
       | 10.10.20.0/30 via 10.10.10.1
       v
    cEOS1
       |
       | 10.10.20.0/30 via 10.10.100.2
       v
    cEOS2
       |
       | directly connected
       v
    client2

Return path:

    client2
       |
       | 10.10.10.0/30 via 10.10.20.1
       v
    cEOS2
       |
       | 10.10.10.0/30 via 10.10.100.1
       v
    cEOS1
       |
       | directly connected
       v
    client1

During troubleshooting, routes were deliberately added one at a time. This
showed that successful forwarding requires every device in both the forward
and return paths to know its appropriate next hop.

## Reproducible Configuration

The router configurations are stored in:

    config/ceos1.cfg
    config/ceos2.cfg

Endpoint addressing and routes are defined using Containerlab `exec` entries
in:

    topology.clab.yml

The complete lab can therefore be destroyed and rebuilt without manually
reconfiguring routing.

Deployment:

    sudo containerlab destroy -t topology.clab.yml
    sudo containerlab deploy -t topology.clab.yml --reconfigure

After a complete redeployment, end-to-end routing worked immediately.

## Verification

End-to-end connectivity was tested from client1:

    docker exec clab-static-routing-client1 ping -c 4 10.10.20.2

Result:

    4 packets transmitted
    4 packets received
    0% packet loss

The ICMP replies arrived with TTL 62.

A Linux-generated ICMP packet begins with TTL 64. The reply crossed two Layer
3 routers:

    client2 -> cEOS2 -> cEOS1 -> client1

Each router decremented the TTL by one:

    64 -> 63 -> 62

This provides packet-level evidence that the traffic traversed both cEOS
routers.

## Ethernet and IPv4 Forwarding

The IPv4 source and destination remain end-to-end:

    Source IP:       10.10.10.2
    Destination IP:  10.10.20.2

Ethernet addressing changes at every routed hop.

For the forward path:

    client1 -> cEOS1
    Ethernet: client1 MAC -> cEOS1 Ethernet1 MAC
    IPv4:     10.10.10.2 -> 10.10.20.2

    cEOS1 -> cEOS2
    Ethernet: cEOS1 Ethernet2 MAC -> cEOS2 Ethernet2 MAC
    IPv4:     10.10.10.2 -> 10.10.20.2

    cEOS2 -> client2
    Ethernet: cEOS2 Ethernet1 MAC -> client2 MAC
    IPv4:     10.10.10.2 -> 10.10.20.2

The routers remove the incoming Ethernet encapsulation, inspect the IPv4
destination, select a next hop, decrement TTL, and create a new Ethernet frame
for the next Layer 2 segment.

## Packet Capture

ICMP traffic was captured directly on `client1:eth1`:

    docker exec clab-static-routing-client1 sh -c '
    tcpdump -ni eth1 -w /tmp/lab02-static-routing.pcap -c 8 icmp &
    sleep 1
    ping -c 4 10.10.20.2
    wait
    '

The capture contained:

    4 ICMP Echo Requests
    4 ICMP Echo Replies
    0 dropped packets

The PCAP was copied to the Windows Utility VM and inspected with Wireshark.

Wireshark showed:

    Echo Request: 10.10.10.2 -> 10.10.20.2, TTL 64
    Echo Reply:   10.10.20.2 -> 10.10.10.2, TTL 62

PCAP files are excluded from Git.

## Lessons Learned

1. Directly connected networks appear automatically in the routing table.
2. A router does not automatically know networks connected to another router.
3. Static routes specify where traffic for remote networks should be sent.
4. The next-hop address must itself be reachable through a connected network.
5. Routing must work in both the forward and return directions.
6. Endpoint routing is just as important as router routing.
7. The Containerlab management network must remain separate from lab traffic.
8. Ethernet addressing changes at each routed hop while end-to-end IPv4
   addresses remain unchanged.
9. TTL provides useful evidence of the number of Layer 3 hops traversed.
10. Declarative configuration makes the complete lab reproducible.

## Result

Lab 02 is complete.

Two cEOS routers now route traffic between separate endpoint /30 networks using
a dedicated /30 transit network and static routes.

The complete topology survives destruction and redeployment without manual
network configuration.

The next lab will replace the manually defined router static routes with OSPF
so that the routers dynamically learn the remote networks.
