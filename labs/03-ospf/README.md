# Lab 03 - OSPF

This lab replaces the static router-to-router routes from Lab 02 with OSPF.

The physical topology and IP addressing remain unchanged. The main difference is how the cEOS routers learn the remote endpoint networks.

In Lab 02, each cEOS router was manually configured with a static route. In this lab, the routers discover each other through OSPF and dynamically exchange routing information.

## Objective

The goals of this lab are to:

- configure OSPF between two cEOS routers
- establish an OSPF neighbor relationship
- dynamically advertise the endpoint /30 networks
- verify OSPF-learned routes in the routing table
- restore end-to-end connectivity without router static routes
- understand OSPF Router IDs, areas, neighbor formation, and DR/BDR behavior
- make the configuration reproducible through Containerlab startup configs

## Topology

```text
client1                 cEOS1                 cEOS2                 client2
10.10.10.2/30           Et1                   Et1                   10.10.20.2/30
     |                  10.10.10.1           10.10.20.1                 |
     |                       |                     |                      |
     +-----------------------+                     +----------------------+
                             |                     |
                            Et2                   Et2
                        10.10.100.1 -------- 10.10.100.2
                             10.10.100.0/30
```

The routed path remains:

```text
client1 -> cEOS1 -> cEOS2 -> client2
```

## Addressing

### Client1 network

```text
10.10.10.0/30

10.10.10.1  cEOS1 Ethernet1
10.10.10.2  client1
```

### Router transit network

```text
10.10.100.0/30

10.10.100.1  cEOS1 Ethernet2
10.10.100.2  cEOS2 Ethernet2
```

### Client2 network

```text
10.10.20.0/30

10.10.20.1  cEOS2 Ethernet1
10.10.20.2  client2
```

## Starting Point

Lab 03 was copied from Lab 02.

The endpoint routes were intentionally kept:

```text
client1:
10.10.20.0/30 via 10.10.10.1

client2:
10.10.10.0/30 via 10.10.20.1
```

The Linux clients do not participate in OSPF. They still need to send traffic for remote networks to their local cEOS gateway.

The static routes on the cEOS routers were removed:

```text
cEOS1:
ip route 10.10.20.0/30 10.10.100.2

cEOS2:
ip route 10.10.10.0/30 10.10.100.1
```

Before OSPF was configured, cEOS1 only knew its directly connected networks:

```text
C 10.10.10.0/30
  directly connected, Ethernet1

C 10.10.100.0/30
  directly connected, Ethernet2
```

It had no route to:

```text
10.10.20.0/30
```

This intentionally created the baseline failure condition for the lab.

## OSPF Configuration

OSPF process 1 is used on both cEOS routers.

### cEOS1

```text
router ospf 1
   router-id 1.1.1.1
   network 10.10.10.0/30 area 0.0.0.0
   network 10.10.100.0/30 area 0.0.0.0
```

### cEOS2

```text
router ospf 1
   router-id 2.2.2.2
   network 10.10.20.0/30 area 0.0.0.0
   network 10.10.100.0/30 area 0.0.0.0
```

## OSPF Process ID

The command:

```text
router ospf 1
```

starts OSPF process `1`.

The process ID is locally significant to the router. It identifies the local OSPF process and does not need to match between OSPF neighbors.

Both routers use process ID `1` in this lab for consistency.

## Router IDs

Each OSPF router has a unique Router ID:

```text
cEOS1 = 1.1.1.1
cEOS2 = 2.2.2.2
```

The Router ID identifies the router inside OSPF. It does not need to correspond to a physical interface address.

## OSPF Area 0

All OSPF-enabled interfaces in this lab are placed in:

```text
Area 0.0.0.0
```

Area 0 is the OSPF backbone area.

Because this topology is small, only a single OSPF area is required.

## Network Statements

On cEOS1:

```text
network 10.10.10.0/30 area 0.0.0.0
network 10.10.100.0/30 area 0.0.0.0
```

On cEOS2:

```text
network 10.10.20.0/30 area 0.0.0.0
network 10.10.100.0/30 area 0.0.0.0
```

These statements cause OSPF to operate on matching interfaces.

The endpoint-facing networks are advertised into OSPF:

```text
cEOS1 advertises 10.10.10.0/30
cEOS2 advertises 10.10.20.0/30
```

The transit network:

```text
10.10.100.0/30
```

is where cEOS1 and cEOS2 discover each other and form their OSPF adjacency.

## Neighbor Formation

After OSPF was initially configured only on cEOS1:

```text
show ip ospf neighbor
```

returned no neighbors.

This was expected because cEOS2 was not yet participating in OSPF.

Once OSPF was configured on cEOS2, the routers discovered each other across Ethernet2.

The transit interfaces were:

```text
cEOS1 Ethernet2 = 10.10.100.1/30
cEOS2 Ethernet2 = 10.10.100.2/30
```

Basic Layer 3 connectivity across the transit network was also confirmed with ICMP.

## OSPF Interface State

On cEOS2:

```text
show ip ospf interface Ethernet2
```

reported:

```text
Interface Address 10.10.100.2/30
Area 0.0.0.0
Network Type Broadcast
Neighbor Count is 1
```

This confirmed that Ethernet2 was participating in OSPF and that an OSPF neighbor had been discovered.

## DR and BDR

Ethernet interfaces use the OSPF broadcast network type by default.

On a broadcast OSPF network, routers elect:

```text
DR  = Designated Router
BDR = Backup Designated Router
```

The DR coordinates OSPF operation on the shared broadcast segment. The BDR is available to take over if the DR disappears.

In this lab, the election resulted in:

```text
DR  = cEOS2, Router ID 2.2.2.2
BDR = cEOS1, Router ID 1.1.1.1
```

Both routers had the default OSPF priority of `1`.

Because the priorities were equal, the higher Router ID won the election:

```text
2.2.2.2 > 1.1.1.1
```

Therefore cEOS2 became the DR.

The DR/BDR role only affects OSPF operation on that Ethernet segment. It does not make cEOS2 the primary router for normal IP forwarding.

For this two-router /30 link, the default broadcast behavior was intentionally left unchanged for learning purposes.

## Dynamically Learned Routes

After the OSPF adjacency formed, cEOS1 learned the client2 network:

```text
O        10.10.20.0/30 [110/20]
           via 10.10.100.2, Ethernet2
```

cEOS2 learned the client1 network:

```text
O        10.10.10.0/30 [110/20]
           via 10.10.100.1, Ethernet2
```

The `O` route code identifies an OSPF-learned route.

This is the main difference compared with Lab 02:

```text
Lab 02:
S = static route

Lab 03:
O = OSPF-learned route
```

## Administrative Distance and Metric

The OSPF routes appeared as:

```text
[110/20]
```

The first value, `110`, is the administrative distance for OSPF.

The second value, `20`, is the OSPF path cost.

In this topology, the learned route has a total cost of 20.

## End-to-End Verification

After OSPF learned the remote networks, client1 was again able to reach client2:

```bash
docker exec clab-ospf-client1 ping -c 4 10.10.20.2
```

Result:

```text
4 packets transmitted
4 packets received
0% packet loss
```

The replies arrived with:

```text
ttl=62
```

The reply crossed:

```text
client2 -> cEOS2 -> cEOS1 -> client1
```

With an initial TTL of 64, each router decremented the TTL once:

```text
64 -> 63 -> 62
```

This confirms that the packet still crossed two Layer 3 hops.

The forwarding path remained unchanged from Lab 02. Only the route-learning mechanism changed.

## Static Routing vs OSPF

Lab 02 required each router to be manually told how to reach the remote network.

Example:

```text
ip route 10.10.20.0/30 10.10.100.2
```

With OSPF, the routers instead:

1. discover each other
2. form an OSPF neighbor relationship
3. exchange information about reachable networks
4. calculate routing paths
5. install learned routes into the routing table

This removes the need to manually configure a static route for every remote network.

## Endpoint Routing Still Matters

OSPF runs only between the cEOS routers in this lab.

The Linux endpoints are not OSPF routers.

They therefore continue to use explicit routes toward their local gateway.

Client1:

```text
10.10.20.0/30 via 10.10.10.1
```

Client2:

```text
10.10.10.0/30 via 10.10.20.1
```

The distinction is:

```text
Endpoint routes:
remain static

Router-to-router routes:
learned dynamically through OSPF
```

## Reproducible Configuration

The working OSPF configuration was added to:

```text
config/ceos1.cfg
config/ceos2.cfg
```

The topology can therefore be destroyed and rebuilt without manually re-entering the OSPF configuration.

Deployment:

```bash
sudo containerlab deploy -t topology.clab.yml --reconfigure
```

A successful fresh deployment should automatically restore:

- cEOS interface addressing
- IP routing
- endpoint routes
- OSPF processes
- Router IDs
- OSPF network advertisements
- OSPF neighbor formation
- dynamically learned routes

## Verification Commands

Inspect the lab:

```bash
sudo containerlab inspect -t topology.clab.yml
```

Enter cEOS:

```bash
docker exec -it clab-ospf-ceos1 Cli
```

Check OSPF:

```text
show ip ospf
show ip ospf neighbor
show ip ospf interface Ethernet2
```

Check OSPF routes:

```text
show ip route ospf
```

Check all routes:

```text
show ip route
```

Verify endpoint routing:

```bash
docker exec clab-ospf-client1 ip route
docker exec clab-ospf-client2 ip route
```

End-to-end test:

```bash
docker exec clab-ospf-client1 ping -c 4 10.10.20.2
```

## Lessons Learned

- Directly connected routes are automatically installed from interface addressing.
- Routers do not automatically know networks behind another router.
- Static routing manually defines remote destinations and next hops.
- OSPF allows routers to dynamically exchange network reachability.
- Router IDs uniquely identify routers within OSPF.
- OSPF neighbors form across interfaces participating in the same area.
- Area 0 is the OSPF backbone area.
- Ethernet uses the OSPF broadcast network type by default.
- Broadcast OSPF networks elect a DR and BDR.
- Equal OSPF priorities are resolved using the highest Router ID.
- OSPF-learned routes appear with the route code `O`.
- Linux endpoints still require their own routing information even when the routers use OSPF.
- Routing must exist in both directions for end-to-end communication.
- TTL provides useful evidence of the number of Layer 3 hops in the forwarding path.
- Declarative startup configs make the Containerlab topology reproducible.

## OSPF Convergence After Redeploy

After a fresh Containerlab deployment, the OSPF relationship did not become fully usable immediately.

The neighbor was first observed in the `2-WAY` state:

    2.2.2.2  1  default  1  2-WAY  10.10.100.2  Ethernet2

During this period, cEOS1 had not yet installed the remote OSPF route.

An end-to-end ping from client1 therefore returned:

    From 10.10.10.1 Destination Net Unreachable

The response originated from cEOS1, confirming that client1 was correctly forwarding traffic to its local gateway, but cEOS1 did not yet have a route to `10.10.20.0/30`.

After OSPF convergence completed, the neighbor state became:

    FULL/DR

and cEOS1 installed:

    O 10.10.20.0/30 [110/20]
      via 10.10.100.2, Ethernet2

Without any additional configuration changes, the same end-to-end ping then succeeded.

This demonstrates that dynamic routing protocols require a short convergence period after interfaces or routers come online.

## Result

Lab 03 successfully replaced the cEOS static routes from Lab 02 with OSPF.

The routers formed an OSPF adjacency across the `10.10.100.0/30` transit network and dynamically learned the remote endpoint networks.

End-to-end communication between client1 and client2 was restored without router static routes.

The forwarding path remained:

```text
client1 -> cEOS1 -> cEOS2 -> client2
```

The next lab will build on this routed topology and introduce multicast and mDNS behavior.
