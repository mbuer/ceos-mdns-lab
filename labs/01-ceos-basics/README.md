# Lab 01 - cEOS Basics

## Objective

Learn the basic Containerlab and cEOS workflow before introducing dynamic
routing or mDNS.

## Initial Topology

The first topology contained one cEOS node and one Linux endpoint:

    client1:eth1 -------- ceos1:eth1

Containerlab mapped `ceos1:eth1` to `Ethernet1` inside EOS.

Containerlab also creates a separate management network. The management
interfaces must not be confused with the interfaces carrying lab traffic.

Observed management addressing:

    client1 eth0       172.20.20.2/24
    ceos1 Management0  172.20.20.3/24

## /30 Addressing

The initial experiment demonstrated configuration of a routed cEOS interface.

The addressing convention was then changed to reserve the lower usable address
for the gateway:

    10.10.10.0/30

    10.10.10.1  cEOS Ethernet1 / default gateway
    10.10.10.2  client1
    10.10.10.3  broadcast

The final Lab 01 topology will contain two endpoint networks:

    client1                         client2
    10.10.10.2/30                  10.10.20.2/30
    GW 10.10.10.1                  GW 10.10.20.1
         |                              |
         | Et1                      Et2 |
         +----------- cEOS1 ------------+
             10.10.10.1/30
             10.10.20.1/30

This allows cEOS to perform actual Layer 3 forwarding between two /30 networks.

## Useful EOS Commands

    show version
    show interfaces status
    show ip interface brief

To convert an EOS switchport into a routed interface:

    configure terminal
    interface Ethernet1
       no switchport
       ip address 10.10.10.1/30
       no shutdown

## Containerlab Lifecycle

Manual configuration inside Linux endpoint containers is not persistent across
container recreation.

After a Utility VM reboot, the existing cEOS Docker container did not initialize
correctly when started independently. The clean recovery procedure was:

    sudo containerlab destroy -t topology.clab.yml
    sudo containerlab deploy -t topology.clab.yml

Containerlab should therefore manage the lifecycle of the lab rather than
starting the cEOS container directly with Docker.

## Next Step

Add a second Linux endpoint and configure cEOS with two routed /30 interfaces.
Then validate Layer 3 forwarding and inspect the traffic with packet capture.
