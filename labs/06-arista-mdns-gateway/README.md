# Lab 06 - Arista mDNS Gateway

## Status

**In progress**

This lab introduces the Arista EOS mDNS Gateway and builds directly on the
behavior observed in Lab 05.

Lab 05 proved that normal mDNS traffic does not traverse the routed Layer 3
boundary between the two endpoint networks.

Lab 06 asks the next question:

> Can the Arista mDNS Gateway extend service discovery between those routed
> networks without treating mDNS as ordinary routed multicast?

The Arista mDNS Gateway control plane is now operational in this lab.

Actual end-to-end DNS-SD service discovery is the next validation step.

---

# 1. Why This Lab Exists

The topology contains two devices on different Layer 3 subnets:

```text
client1                                      client2
10.10.10.2/30                              10.10.20.2/30
     |                                           |
     |                                           |
   cEOS1 ----------- 10.10.100.0/30 ---------- cEOS2
10.10.10.1                                   10.10.20.1
```

Normal unicast routing works.

Generic multicast routing also works because previous labs configured:

- IGMP
- PIM Sparse Mode
- a Rendezvous Point
- multicast routing
- OSPF for unicast reachability

However, Lab 05 demonstrated that mDNS still does not cross this boundary.

The reason is important:

> mDNS is deliberately designed around local-link discovery.

The Arista mDNS Gateway exists to extend that discovery domain in a controlled,
application-aware way.

---

# 2. mDNS and DNS-SD Are Not the Same Thing

These two terms are often used together, but they describe different parts of
the system.

## mDNS

mDNS means:

```text
Multicast DNS
```

It provides DNS-like name resolution and query/response behavior on the local
network without requiring a conventional DNS server.

IPv4 mDNS uses:

```text
Multicast destination: 224.0.0.251
UDP port:              5353
```

A compliant mDNS querier normally sends mDNS queries from UDP source port
`5353` and listens for responses on the same port.

mDNS therefore answers questions such as:

```text
Who owns printer.local?
```

or carries the DNS queries used by DNS-SD.

---

## DNS-SD

DNS-SD means:

```text
DNS-Based Service Discovery
```

DNS-SD defines how devices describe and discover **services** using normal DNS
record types.

For example, a device may advertise a web interface as:

```text
Test Web._http._tcp.local
```

DNS-SD allows another device to discover:

1. that the service exists
2. its instance name
3. which host provides it
4. which TCP/UDP port it uses
5. optional metadata

DNS-SD does not invent a completely new protocol.

It uses standard DNS records such as:

```text
PTR
SRV
TXT
A
AAAA
```

When DNS-SD is combined with mDNS, this service discovery can happen
automatically on the local link without a conventional DNS infrastructure.

---

# 3. How DNS-SD Service Discovery Works

A useful mental model is:

```text
"What services exist?"
        |
        v
       PTR
        |
        v
"What instance provides this service?"
        |
        v
       SRV
        |
        +----> hostname
        |
        +----> port

       TXT
        |
        +----> metadata

       A / AAAA
        |
        +----> IP address
```

Consider this service:

```text
Test Web._http._tcp.local
```

The records could conceptually look like this.

## PTR

```text
_http._tcp.local
    ->
Test Web._http._tcp.local
```

This means:

> There is an instance named `Test Web` providing an HTTP service.

---

## SRV

```text
Test Web._http._tcp.local
    ->
test-host.local
port 8080
```

This tells the client:

> The service is hosted by `test-host.local` on TCP port `8080`.

---

## TXT

```text
Test Web._http._tcp.local
    ->
path=/
```

TXT records carry service-specific metadata.

---

## A record

```text
test-host.local
    ->
10.10.20.2
```

Now the client has enough information to connect:

```text
10.10.20.2:8080
```

---

# 4. What `_services._dns-sd._udp.local` Means

Earlier in the lab we generated this query:

```text
_services._dns-sd._udp.local
```

This is a special DNS-SD meta-query.

It does **not** mean:

```text
find a web server
```

or:

```text
find a specific SmartPanel
```

Instead it asks approximately:

> What kinds of DNS-SD services are currently being advertised?

For example, a response might indicate that these service types exist:

```text
_http._tcp.local
_ipp._tcp.local
_raop._tcp.local
```

This is useful for troubleshooting and service enumeration, but it is not the
same as browsing for an actual service instance.

That distinction becomes important later in this lab.

---

# 5. Why mDNS Normally Stops at Layer 3

mDNS uses the IPv4 multicast address:

```text
224.0.0.251
```

This address is intended for link-local mDNS communication.

A normal mDNS packet therefore behaves conceptually like this:

```text
Host A
   |
   | 224.0.0.251:5353
   v
local subnet
   |
   X
   X router boundary
   X
```

The router is not expected to behave as if this were an ordinary
administratively scoped multicast group.

This is exactly what Lab 05 demonstrated.

---

# 6. Lab 05 Baseline

The Lab 05 test generated a valid DNS packet and sent it from:

```text
10.10.10.2
```

to:

```text
224.0.0.251:5353
```

Packet capture on client1 showed:

```text
IP
    ttl 255

10.10.10.2.56013 > 224.0.0.251.5353

PTR (QM)?
_services._dns-sd._udp.local.
```

The packet was visible on the source-facing side of cEOS1.

It was **not** visible on the cEOS1 transit interface toward cEOS2.

The observed behavior was therefore:

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

This established the baseline for Lab 06.

---

# 7. Why PIM Does Not Solve mDNS

Lab 04 successfully routed this multicast group:

```text
239.1.1.1
```

using PIM Sparse Mode.

That process looked conceptually like this:

```text
receiver
   |
   | IGMP
   v
router
   |
   | PIM multicast tree
   v
router
   |
 sender
```

PIM does not care whether the multicast payload contains:

- audio
- video
- telemetry
- arbitrary UDP data

It builds multicast forwarding state using concepts such as:

```text
(S,G)
(*,G)
RPF
RP
```

mDNS Gateway is fundamentally different.

It understands **DNS and DNS-SD semantics**.

Instead of simply routing every `224.0.0.251` packet, it understands concepts
such as:

```text
service types
queries
responses
records
links
remote gateways
```

That is why the Arista configuration contains commands such as:

```text
service
type
query
response
```

rather than simply:

```text
route 224.0.0.251
```

---

# 8. PIM vs mDNS Gateway

The conceptual difference is:

## Routed multicast

```text
sender
   |
   v
+--------+
| router |
|  PIM   |
+--------+
    |
    | multicast forwarding tree
    |
+--------+
| router |
|  PIM   |
+--------+
   |
   v
receiver
```

The routers forward the multicast stream itself.

---

## mDNS Gateway

```text
client1
   |
   | local mDNS
   | 224.0.0.251:5353
   v
+-----------+
|   cEOS1   |
| mDNS GW   |
+-----------+
      ||
      || gateway communication
      ||
+-----------+
|   cEOS2   |
| mDNS GW   |
+-----------+
   |
   | local mDNS
   v
client2
```

The gateways extend discovery knowledge between otherwise separate local mDNS
domains.

The original mDNS packet is therefore not simply routed hop-by-hop like the
multicast stream from Lab 04.

---

# 9. Arista mDNS Gateway Architecture

Arista describes its mDNS Gateway as extending the normal link-local scope of
mDNS to additional subnets.

Arista also supports peering mDNS gateways.

This allows directly connected subnets behind different gateways to participate
in a larger logical mDNS discovery domain.

Our topology uses exactly this model:

```text
10.10.10.0/30
     |
     |
  cEOS1
 mDNS GW
     |
     | routed transit
     |
10.10.100.0/30
     |
     |
  cEOS2
 mDNS GW
     |
     |
10.10.20.0/30
```

The client-facing subnet on each router remains its own Layer 3 network.

The gateways cooperate to extend service discovery between them.

---

# 10. What Is DSO?

One of the least obvious parts of the configuration is:

```text
DSO
```

DSO means:

```text
DNS Stateful Operations
```

It is standardized in RFC 8490.

Traditional DNS communication is usually transaction-oriented:

```text
query
  ->
response
  ->
done
```

DSO introduces the concept of a **persistent stateful DNS session**.

Instead of creating an isolated relationship for every DNS transaction, two
systems can maintain an ongoing connection:

```text
Gateway A
    ||
    || persistent session
    ||
Gateway B
```

Either side can send stateful DNS-related operations over that connection.

The session can also maintain:

- timeout state
- keepalive behavior
- orderly termination
- extensions for additional stateful DNS operations

In this lab, Arista uses DSO as part of the communication between the two mDNS
gateways.

---

# 11. DSO in This Lab

Our gateways use their routed transit addresses:

```text
cEOS1: 10.10.100.1
cEOS2: 10.10.100.2
```

The gateway connection uses:

```text
TCP port 8853
```

Conceptually:

```text
cEOS1
10.10.100.1
    ||
    || TCP / DSO
    ||
    || port 8853
    ||
10.10.100.2
cEOS2
```

This is important:

> DSO traffic is not the original mDNS multicast packet.

The local endpoints still use:

```text
UDP 5353
224.0.0.251
```

The gateway-to-gateway relationship is separate.

---

# 12. Lab Topology

```text
client1
10.10.10.2/30
    |
    |
Ethernet1
cEOS1
10.10.10.1/30
    |
Ethernet2
10.10.100.1/30
    |
    |
10.10.100.0/30
    |
    |
10.10.100.2/30
Ethernet2
cEOS2
10.10.20.1/30
Ethernet1
    |
    |
client2
10.10.20.2/30
```

The routed path is:

```text
client1 -> cEOS1 -> cEOS2 -> client2
```

---

# 13. Containerlab Configuration

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

# 14. Why client1 Needs an Explicit mDNS Route

Containerlab gives each Linux endpoint a management interface in addition to
the lab interface.

Initially:

```bash
ip route get 224.0.0.251
```

returned:

```text
multicast 224.0.0.251 dev eth0
src 172.20.20.11
```

This would send mDNS through the Containerlab management network.

The lab therefore adds:

```text
224.0.0.251/32 dev eth1
```

Now:

```bash
ip route get 224.0.0.251
```

returns:

```text
multicast 224.0.0.251 dev eth1
src 10.10.10.2
```

This ensures the experiment uses the intended data-plane network.

---

# 15. Enabling mDNS on the Router

The global EOS configuration mode is:

```text
mdns
```

Initially:

```text
show mdns status
```

reported:

```text
mDNS is disabled
```

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

# 16. Enabling a Local mDNS Link

Enabling the global mDNS process is not enough.

Each local interface participating in mDNS also needs:

```text
mdns ipv4
```

For example:

```text
interface Ethernet1
   no switchport
   ip address 10.10.10.1/30
   mdns ipv4
```

Afterward:

```text
show mdns links
```

shows:

```text
Interface       Address Family       Status
Ethernet1       ipv4                 active
```

This was a key troubleshooting discovery.

Before configuring `mdns ipv4`:

```text
show mdns links
```

was empty.

Packet capture showed that mDNS packets physically arrived at Ethernet1, but:

```text
Received MDNS packets: 0
```

remained unchanged.

After enabling:

```text
mdns ipv4
```

the packet was processed by the EOS mDNS subsystem.

---

# 17. Service Rules

The Arista gateway uses service rules.

Our temporary rule is:

```text
service test
   type any
   query Ethernet1
   response interface Ethernet1
```

This can be understood as three separate policy decisions.

---

## `type any`

```text
type any
```

means the rule is not restricted to a specific DNS-SD service type.

Later, a production configuration could potentially restrict discovery to
specific services.

For example:

```text
_http._tcp
```

or another required application-specific type.

---

## `query Ethernet1`

```text
query Ethernet1
```

defines a local link from which DNS-SD/mDNS queries are accepted by this rule.

Conceptually:

```text
client
   |
   | query
   v
Ethernet1
   |
mDNS Gateway
```

---

## `response interface Ethernet1`

```text
response interface Ethernet1
```

defines the local interface from which service responses/announcements are
accepted by the rule.

Conceptually:

```text
service device
   |
   | advertisement / response
   v
Ethernet1
   |
mDNS Gateway
```

---

# 18. Remote Gateway Configuration

cEOS1 identifies cEOS2 as a remote gateway:

```text
remote-gateway ipv4 10.10.100.2
```

cEOS2 identifies cEOS1:

```text
remote-gateway ipv4 10.10.100.1
```

The relationship is therefore symmetric:

```text
cEOS1
10.10.100.1
    |
    | remote gateway
    |
10.10.100.2
cEOS2
```

---

# 19. DSO Server

Initially the gateway relationship remained:

```text
Status: connecting
```

The reason became visible in:

```text
show mdns status
```

which reported:

```text
DSO server is disabled
```

The DSO listener was enabled with:

```text
dso server ipv4
```

on both routers.

After both sides were enabled, the peer relationship established successfully.

---

# 20. Successful Gateway Peering

cEOS1 reported:

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

This is an important milestone.

It proves:

```text
cEOS1 <==== persistent DSO relationship ====> cEOS2
```

The gateway control plane is therefore operational.

---

# 21. Full cEOS1 mDNS Configuration

```text
interface Ethernet1
   no switchport
   ip address 10.10.10.1/30
   mdns ipv4
   pim ipv4 sparse-mode

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

---

# 22. Full cEOS2 mDNS Configuration

```text
interface Ethernet1
   no switchport
   ip address 10.10.20.1/30
   ip igmp
   mdns ipv4
   pim ipv4 sparse-mode

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

# 23. Operational Commands

The most useful EOS commands discovered so far are:

```text
show mdns status
```

Shows:

- whether mDNS is running
- whether DSO server is running
- configured gateway connections
- connection status
- inbound DSO clients

---

```text
show mdns links
```

Shows which local interfaces are participating in mDNS.

Expected:

```text
Ethernet1    ipv4    active
```

---

```text
show mdns counters
```

Shows processing counters including:

```text
Received MDNS packets
Sent MDNS packets
Received DSO packets
Sent DSO packets
Discarded MDNS packets
Ignored One-shot queries
Packet transmission errors
```

---

```text
show mdns service rule test
```

Shows the operational interpretation of the service rule.

Current output:

```text
Query link: Ethernet1
Response link:
Response interface: Ethernet1

Service types:
   Service Name       Interface/Link       Location    Status
------------------ -------------------- -------------- ------
```

No actual service has yet populated this table.

---

# 24. Current Packet-Test Result

After enabling `mdns ipv4`, sending the synthetic mDNS query caused:

```text
Received MDNS packets: 1
```

on cEOS1.

This proves:

```text
client1
   |
   | mDNS
   v
cEOS1 Ethernet1
   |
   v
EOS mDNS process
```

is working.

However:

```text
Sent DSO packets: 0
Sent MDNS packets: 0
```

remained unchanged.

Client2 therefore received nothing.

---

# 25. Why This Does Not Yet Mean the Gateway Is Broken

The test packet was:

```text
_services._dns-sd._udp.local
```

This is the DNS-SD **service-type enumeration meta-query**.

It asks:

```text
"What service types exist?"
```

It does not advertise an actual service.

It also does not prove the complete DNS-SD workflow involving:

```text
PTR
SRV
TXT
A / AAAA
```

The next test should therefore use a real service rather than only an
enumeration query.

---

# 26. Next Test: Real Service Advertisement

The next experiment will create a real DNS-SD service on client2.

Example:

```text
Test Web._http._tcp.local
```

Conceptually, client2 should advertise:

```text
PTR
_http._tcp.local
    ->
Test Web._http._tcp.local
```

plus:

```text
SRV
Test Web._http._tcp.local
    ->
client2.local:8080
```

and:

```text
TXT
Test Web._http._tcp.local
    ->
path=/
```

and:

```text
A
client2.local
    ->
10.10.20.2
```

Then client1 will browse for:

```text
_http._tcp.local
```

The expected successful flow is:

```text
client2
   |
   | local service advertisement
   v
cEOS2 mDNS Gateway
   ||
   || DSO gateway relationship
   ||
cEOS1 mDNS Gateway
   |
   | local discovery
   v
client1
```

---

# 27. Planned Endpoint Tools

The endpoint image will first be checked for:

```text
avahi-browse
avahi-publish
dns-sd
```

If available, these will provide a realistic DNS-SD implementation.

If not, a small Python-based advertiser and browser can be used.

The important goal is to generate **real DNS-SD service records**, not merely
arbitrary UDP traffic.

---

# 28. Reproducibility

The current router configuration has been saved with:

```text
copy running-config startup-config
```

and exported into:

```text
config/ceos1.cfg
config/ceos2.cfg
```

The Containerlab topology references these files:

```yaml
startup-config: config/ceos1.cfg
startup-config: config/ceos2.cfg
```

A full reproducibility test is still required:

```bash
sudo containerlab destroy -t topology.clab.yml

sudo containerlab deploy \
  -t topology.clab.yml \
  --reconfigure
```

After redeploy, verify:

```text
show mdns status
show mdns links
show mdns counters
show mdns service rule test
```

The gateway connection should return automatically to:

```text
connected
```

without manual reconfiguration.

---

# 29. Troubleshooting Timeline

This lab exposed several separate requirements.

Understanding these failures is useful because each one demonstrates a
different layer of the feature.

## Problem 1 - mDNS globally disabled

Configuration existed, but:

```text
mDNS is disabled
```

Fix:

```text
mdns
   no disabled
```

---

## Problem 2 - DSO peer stuck connecting

Status:

```text
Gateway DSO connections
10.10.100.x 8853 connecting
```

Cause:

```text
DSO server is disabled
```

Fix:

```text
dso server ipv4
```

on both routers.

Result:

```text
connected
```

---

## Problem 3 - mDNS packets not counted

tcpdump proved the packet reached Ethernet1, but:

```text
Received MDNS packets: 0
```

and:

```text
show mdns links
```

was empty.

Cause:

```text
Ethernet1 was not configured as an mDNS link.
```

Fix:

```text
interface Ethernet1
   mdns ipv4
```

Result:

```text
Ethernet1 ipv4 active
```

and:

```text
Received MDNS packets: 1
```

---

## Problem 4 - Query still not appears on client2

The gateway control plane is established, but the synthetic enumeration query
has not yet produced a remote client-side packet.

This remains under investigation.

The next test will use a real DNS-SD service advertisement.

---

# 30. Layer-by-Layer Mental Model

The entire lab can be understood as four separate layers.

## Layer 1 - Endpoint mDNS

```text
client
   |
   | UDP 5353
   | 224.0.0.251
   v
local link
```

---

## Layer 2 - Local mDNS Gateway Interface

```text
interface Ethernet1
   mdns ipv4
```

This allows EOS to process mDNS received on that local link.

---

## Layer 3 - EOS Service Policy

```text
service test
   type any
   query Ethernet1
   response interface Ethernet1
```

This determines what discovery information the gateway accepts.

---

## Layer 4 - Gateway Peering

```text
remote-gateway ipv4 <peer>
dso server ipv4
```

This creates the relationship between mDNS gateways.

Conceptually:

```text
Endpoint
   |
   | mDNS
   v
Local mDNS Link
   |
   v
Service Policy
   |
   v
mDNS Gateway
   ||
   || DSO
   ||
Remote mDNS Gateway
   |
   v
Remote Local Link
   |
   v
Endpoint
```

---

# 31. Where IGMP and PIM Fit

One important lesson from the previous labs is that these are separate
mechanisms.

```text
IGMP
```

tracks local multicast receiver membership.

```text
PIM
```

builds routed multicast distribution trees.

```text
mDNS Gateway
```

understands and extends mDNS/DNS-SD discovery.

They should not be mentally combined into one multicast protocol.

A simplified comparison:

| Mechanism | Purpose |
|---|---|
| IGMP | Host tells local router which multicast groups it wants |
| IGMP snooping | L2 switch limits multicast forwarding to interested ports |
| PIM | Routers build multicast distribution trees |
| RP | Rendezvous point used by PIM Sparse Mode |
| mDNS | Local DNS-like multicast discovery |
| DNS-SD | Describes and discovers services using DNS records |
| mDNS Gateway | Extends mDNS/DNS-SD between local discovery domains |
| DSO | Persistent stateful DNS communication mechanism used between systems |

---

# 32. Current Lab State

At this checkpoint:

```text
OSPF routing                         WORKING
Generic routed multicast             WORKING
PIM                                  WORKING
RP                                   WORKING

mDNS process                         RUNNING
Ethernet1 mDNS links                 ACTIVE
Service rule                         CONFIGURED

DSO server                           RUNNING
Remote mDNS gateways                 CONNECTED

mDNS packet reception by cEOS1       VERIFIED
Real DNS-SD service advertisement    NOT YET TESTED
Cross-subnet service discovery       NOT YET VERIFIED
Full destroy/redeploy validation     NOT YET COMPLETED
```

---

# 33. Current Conclusion

Lab 06 has successfully established the major control-plane components required
for Arista mDNS Gateway operation.

The lab has demonstrated that:

1. cEOS 4.36.1F exposes the Arista mDNS Gateway feature.
2. mDNS must be enabled globally.
3. Participating local interfaces must explicitly enable `mdns ipv4`.
4. Gateway policy is built around service types, queries and responses.
5. Remote mDNS gateways can be configured across a routed network.
6. DSO must be enabled for the gateway relationship to establish.
7. The gateways successfully reach `connected` state.
8. EOS now processes mDNS received on the local endpoint interface.
9. A simple DNS-SD service-enumeration query has not yet demonstrated
   cross-subnet discovery.
10. The next meaningful test is actual DNS-SD service advertisement and
    discovery.

The final acceptance condition remains:

```text
client2 advertises service
        |
        v
cEOS2
        |
        | mDNS Gateway / DSO
        |
        v
cEOS1
        |
        v
client1 discovers service
```

Until that succeeds after a clean Containerlab redeploy, **Lab 06 remains in
progress**.

---

# 34. References

## Arista

Arista EOS - Multicast DNS Gateway

Arista describes the mDNS Gateway as extending the link-local scope of mDNS to
additional subnets and supporting peering between mDNS gateways.

Feature documentation is associated with EOS 4.32.1F.

The exact CLI in this lab was verified directly against:

```text
cEOS 4.36.1F
```

rather than assumed from documentation.

---

## RFC 6762 - Multicast DNS

Defines Multicast DNS, including:

```text
UDP 5353
224.0.0.251
.local
mDNS query and response behavior
```

---

## RFC 6763 - DNS-Based Service Discovery

Defines DNS-SD, including:

```text
service types
service instances
PTR
SRV
TXT
service-type enumeration
```

---

## RFC 8490 - DNS Stateful Operations

Defines:

```text
DSO = DNS Stateful Operations
```

and the persistent stateful DNS-session model used for ongoing DNS-related
communication.
