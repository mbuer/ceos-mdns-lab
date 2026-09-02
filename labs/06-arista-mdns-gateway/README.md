# Lab 06 - Arista mDNS Gateway

## Status

**In progress**

This lab builds on Lab 05, which proved that standard mDNS traffic sent to:

```text
224.0.0.251:5353
```

does not cross the routed Layer 3 boundary between the two endpoint networks.

Lab 06 introduces the Arista EOS mDNS Gateway feature and validates the control-plane pieces required to extend mDNS discovery across routed subnets.

The final end-to-end service discovery test is still pending.

---

## Objective

The goal of this lab is to determine whether Arista EOS mDNS Gateway can extend mDNS discovery between:

```text
client1 -> cEOS1 -> cEOS2 -> client2
```

where client1 and client2 reside on separate `/30` routed networks.

The intended final validation is:

```text
without mDNS Gateway
        |
        v
mDNS discovery fails across Layer 3

with mDNS Gateway
        |
        v
mDNS service discovery succeeds across Layer 3
```

---

## Topology

```text
client1
10.10.10.2/30
    |
    |
cEOS1
10.10.10.1/30
    |
    | 10.10.100.0/30
    |
cEOS2
10.10.20.1/30
    |
    |
client2
10.10.20.2/30
```

Transit addresses:

```text
cEOS1 Ethernet2: 10.10.100.1/30
cEOS2 Ethernet2: 10.10.100.2/30
```

The topology inherits the routed multicast configuration from Lab 05 / Lab 04, including:

- OSPF
- multicast routing
- PIM Sparse Mode
- RP configuration
- IGMP on the receiver side

---

## Containerlab Topology

The Lab 06 topology is:

```yaml
name: mdns-gateway

topology:
  nodes:
    ceos1:
      kind: arista_ceos
      image: ceos:4.36.1F
      startup-config: config/ceos1.cfg

    ceos2:
      kind: arista_ceos
      image: ceos:4.36.1F
      startup-config: config/ceos2.cfg

    client1:
      kind: linux
      image: lab-endpoint:latest
      exec:
        - ip addr add 10.10.10.2/30 dev eth1
        - ip link set eth1 up
        - ip route add 10.10.20.0/30 via 10.10.10.1
        - ip route add 224.0.0.251/32 dev eth1

    client2:
      kind: linux
      image: lab-endpoint:latest
      exec:
        - ip addr add 10.10.20.2/30 dev eth1
        - ip link set eth1 up
        - ip route add 10.10.10.0/30 via 10.10.20.1

  links:
    - endpoints: ["client1:eth1", "ceos1:eth1"]
    - endpoints: ["ceos1:eth2", "ceos2:eth2"]
    - endpoints: ["client2:eth1", "ceos2:eth1"]
```

---

## Arista mDNS Gateway CLI

The cEOS 4.36.1F image exposes a dedicated global mDNS configuration mode:

```text
mdns
```

Available options include:

```text
disabled
dso
flooding
remote-gateway
service
```

Service rules expose:

```text
match
query
response
type
```

This confirms that the cEOS image contains the Arista mDNS Gateway feature.

---

## Local mDNS Link Configuration

The client-facing Ethernet1 interface on both cEOS routers was enabled for IPv4 mDNS processing.

### cEOS1

```text
interface Ethernet1
   no switchport
   ip address 10.10.10.1/30
   mdns ipv4
```

### cEOS2

```text
interface Ethernet1
   no switchport
   ip address 10.10.20.1/30
   ip igmp
   mdns ipv4
```

Operational verification:

```text
show mdns links
```

On both routers:

```text
Interface    Address Family    Status
Ethernet1    ipv4              active
```

This was an important discovery.

Before `mdns ipv4` was enabled on the interface:

```text
show mdns links
```

returned an empty table and mDNS counters remained at zero even though packet capture proved the mDNS packet physically reached cEOS1.

---

## Global mDNS Gateway Configuration

### cEOS1

```text
mdns
   no disabled
   remote-gateway ipv4 10.10.100.2
   dso server ipv4
   !
   service test
      type any
      query Ethernet1
      response interface Ethernet1
```

### cEOS2

```text
mdns
   no disabled
   remote-gateway ipv4 10.10.100.1
   dso server ipv4
   !
   service test
      type any
      query Ethernet1
      response interface Ethernet1
```

---

## mDNS Gateway Enablement

Initially:

```text
show mdns status
```

reported:

```text
mDNS is disabled
```

The configured service rules and remote gateway therefore existed in the configuration, but the mDNS process itself was not running.

The feature was enabled with:

```text
mdns
   no disabled
```

Afterward:

```text
show mdns status
```

reported:

```text
mDNS is running
```

---

## DSO Gateway Peering

The two mDNS gateways use DSO over TCP for communication.

Initially, the remote gateway status showed:

```text
Status: connecting
```

while:

```text
DSO server is disabled
```

The DSO server was enabled on both routers with:

```text
dso server ipv4
```

The gateway connection uses TCP port:

```text
8853
```

After enabling the DSO server on both routers, cEOS1 reported:

```text
mDNS is running
Flooding suppression is disabled
DSO server is running

Gateway DSO connections
Address          Port       Status
10.10.100.2      8853       connected

DSO client connections
Address          Port
10.10.100.2      49468
```

This proves that the two mDNS gateways successfully established the remote gateway relationship.

Conceptually:

```text
cEOS1
10.10.100.1
    |
    | DSO / TCP 8853
    |
cEOS2
10.10.100.2
```

---

## Service Rule

A temporary service rule named:

```text
test
```

was configured on both gateways.

Configuration:

```text
service test
   type any
   query Ethernet1
   response interface Ethernet1
```

Operational inspection on cEOS1:

```text
show mdns service rule test
```

returned:

```text
Query link: Ethernet1
Response link:
Response interface: Ethernet1

Service types:
   Service Name       Interface/Link       Location    Status
------------------ -------------------- -------------- ------
```

No concrete service types had yet been learned at this point.

---

## mDNS Test Packet

The same valid mDNS query created in Lab 05 was reused.

The query requests:

```text
_services._dns-sd._udp.local
```

using:

```text
PTR
```

and is sent to:

```text
224.0.0.251:5353
```

with TTL:

```text
255
```

Example sender:

```bash
docker exec clab-mdns-l3-client1 python3 -c '
import socket, struct

name = "_services._dns-sd._udp.local"
qname = b"".join(
    bytes([len(x)]) + x.encode()
    for x in name.split(".")
) + b"\x00"

packet = struct.pack("!HHHHHH", 0, 0, 1, 0, 0, 0)
packet += qname
packet += struct.pack("!HH", 12, 1)

s = socket.socket(
    socket.AF_INET,
    socket.SOCK_DGRAM,
    socket.IPPROTO_UDP
)

s.setsockopt(
    socket.IPPROTO_IP,
    socket.IP_MULTICAST_IF,
    socket.inet_aton("10.10.10.2")
)

s.setsockopt(
    socket.IPPROTO_IP,
    socket.IP_MULTICAST_TTL,
    255
)

s.sendto(packet, ("224.0.0.251", 5353))

print("sent valid mDNS query")
'
```

---

## mDNS Packet Processing

Before `mdns ipv4` was configured on Ethernet1:

```text
show mdns counters
```

showed:

```text
Received MDNS packets: 0
```

even though tcpdump confirmed the packet reached the interface.

After enabling:

```text
interface Ethernet1
   mdns ipv4
```

the counter changed to:

```text
Received MDNS packets: 1
```

This proves the mDNS process is now receiving the packet.

At the same time:

```text
Discarded MDNS packets: 0
Ignored One-shot queries: 0
Sent DSO packets: 0
Sent MDNS packets: 0
```

The gateway therefore received the query but did not yet relay it to the peer or reproduce it on the remote client-facing interface.

---

## Current Result

The current control-plane state is:

```text
mDNS enabled                YES
mDNS process running        YES
Ethernet1 mDNS links active YES
DSO servers running         YES
Remote gateways connected   YES
Service rule configured     YES
mDNS packet received        YES
mDNS packet relayed         NOT YET
```

The current packet flow is therefore:

```text
client1
   |
   | valid mDNS query
   v
cEOS1 Ethernet1
   |
   | Received MDNS packets +1
   v
cEOS1 mDNS Gateway
   |
   | DSO peer exists
   |
   X query not yet relayed
   |
cEOS2
   |
client2
```

---

## Important Finding

The special query:

```text
_services._dns-sd._udp.local
```

is a DNS-SD service-type enumeration query.

It is not itself a concrete advertised service such as:

```text
_http._tcp.local
_ipp._tcp.local
_raop._tcp.local
```

The next test will therefore use an actual advertised DNS-SD service instead of only a service-enumeration query.

---

## Next Test

The next goal is to create a real service on one endpoint.

Example:

```text
Test Service._http._tcp.local
```

The intended test becomes:

```text
client2
advertises:
Test Service._http._tcp.local
        |
        v
cEOS2 mDNS Gateway
        |
        | DSO
        |
        v
cEOS1 mDNS Gateway
        |
        v
client1 discovers service
```

The endpoint image will first be checked for:

```text
avahi-browse
avahi-publish
dns-sd
```

If those tools are unavailable, a small Python-based mDNS advertiser and browser will be used.

---

## Reproducibility Check Still Required

The current cEOS startup configurations have been exported into:

```text
config/ceos1.cfg
config/ceos2.cfg
```

These files include:

- `mdns ipv4`
- `no disabled`
- remote gateway configuration
- DSO server
- service rule

The topology references these files through:

```yaml
startup-config: config/ceos1.cfg
startup-config: config/ceos2.cfg
```

A complete destroy and redeploy test is still required:

```bash
sudo containerlab destroy -t topology.clab.yml
sudo containerlab deploy -t topology.clab.yml --reconfigure
```

After redeploy, verify:

```text
show mdns status
show mdns links
show running-config section mdns
```

The DSO gateway relationship should return to:

```text
connected
```

without manual configuration.

---

## Current Checkpoint

Lab 06 has successfully established the Arista mDNS Gateway control plane.

The following have been proven:

- mDNS Gateway exists in cEOS 4.36.1F
- the global mDNS process can be enabled
- routed interfaces can be activated as mDNS links
- service rules can be defined
- remote mDNS gateways can be configured
- DSO servers can be enabled
- DSO peer connections establish successfully
- mDNS packets are received by the EOS mDNS process

The remaining task is to prove actual DNS-SD service discovery across the Layer 3 boundary.

**Lab 06 is not yet complete.**
