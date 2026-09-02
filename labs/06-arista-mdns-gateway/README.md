# Lab 06 - Arista mDNS Gateway

## Result

**Completed successfully**

This lab demonstrates DNS-SD service discovery across two routed Layer 3
networks using the Arista EOS mDNS Gateway.

A service advertised by `client2`:

```text
Test Web._http._tcp.local
```

was successfully discovered from the `client1` subnet after a full
Containerlab destroy/redeploy.

Final discovery path:

```text
client2
10.10.20.2
   |
   | local mDNS / DNS-SD
   v
+----------------+
|     cEOS2      |
|   mDNS GW      |
+----------------+
        ||
        || gateway peering / DSO
        ||
+----------------+
|     cEOS1      |
|   mDNS GW      |
+----------------+
   |
   | new local mDNS response
   v
client1
10.10.10.2
```

The important result is:

> The original `224.0.0.251` multicast packet is not routed through the
> network using PIM. The Arista gateways understand DNS-SD information,
> exchange gateway state, and generate mDNS responses on the remote local link.

---

# 1. Objective

Lab 05 proved that normal mDNS does not cross the routed boundary:

```text
client1
   |
   | 224.0.0.251:5353
   v
cEOS1
   X
   X Layer 3 boundary
   X
cEOS2
   |
client2
```

Lab 06 answers:

> Can Arista mDNS Gateway extend DNS-SD discovery between these two routed
> endpoint networks?

The answer is **yes**.

---

# 2. Topology

```text
client1
10.10.10.2/30
    |
    | Ethernet1
    |
cEOS1
10.10.10.1/30
    |
    | Ethernet2
    | 10.10.100.1/30
    |
    | 10.10.100.0/30
    |
    | 10.10.100.2/30
    | Ethernet2
    |
cEOS2
10.10.20.1/30
    |
    | Ethernet1
    |
client2
10.10.20.2/30
```

OSPF provides unicast reachability between the endpoint networks.

The PIM configuration from Lab 04 remains present, but PIM is **not**
responsible for the mDNS discovery demonstrated here.

---

# 3. mDNS and DNS-SD

## mDNS

mDNS means **Multicast DNS**.

IPv4 mDNS normally uses:

```text
Destination: 224.0.0.251
UDP port:    5353
Domain:      .local
```

It provides DNS-like queries and responses on a local network without requiring
a conventional DNS server.

mDNS is intentionally local-link oriented. This is why normal mDNS stopped at
the router in Lab 05.

---

## DNS-SD

DNS-SD means **DNS-Based Service Discovery**.

It describes services using normal DNS records.

Our test service was:

```text
Test Web._http._tcp.local
```

The important records were:

```text
PTR   service instance
SRV   hostname and port
TXT   service metadata
A     IPv4 address
```

Our service resolved conceptually as:

```text
_http._tcp.local
        |
        | PTR
        v
Test Web._http._tcp.local
        |
        | SRV
        v
client2.local:8080
        |
        | A
        v
10.10.20.2
```

with metadata:

```text
TXT path=/
```

A useful distinction is:

```text
mDNS   = local multicast DNS transport
DNS-SD = service-discovery information carried using DNS records
```

---

# 4. Why PIM Does Not Solve mDNS

Lab 04 routed ordinary multicast traffic using:

```text
IGMP
PIM Sparse Mode
RP
RPF
```

PIM builds a multicast forwarding tree:

```text
sender
   |
   v
router
   |
   | multicast forwarding
   v
router
   |
   v
receiver
```

PIM does not need to understand the application inside the UDP payload.

The mDNS Gateway is different.

It understands concepts such as:

```text
service types
queries
responses
DNS records
local links
remote links
remote gateways
```

So the architecture is closer to:

```text
local mDNS
    |
    v
mDNS Gateway
    ||
    || gateway communication
    ||
mDNS Gateway
    |
    v
new local mDNS response
```

The gateway is application-aware rather than simply forwarding the original
multicast packet.

---

# 5. What Is DSO?

DSO means:

```text
DNS Stateful Operations
```

DSO is standardized in RFC 8490.

Traditional DNS is easy to imagine as individual transactions:

```text
query -> response -> finished
```

DSO provides a framework for a **persistent stateful DNS session**:

```text
Gateway A
    ||
    || persistent session
    ||
Gateway B
```

In this lab, EOS uses a DSO relationship between the two mDNS gateways.

The observed gateway connection was:

```text
cEOS1 10.10.100.1
        ||
        || TCP 8853
        ||
cEOS2 10.10.100.2
```

EOS reported:

```text
Gateway DSO connections
Address          Port       Status
10.10.100.2      8853       connected
```

Important:

> DSO is not simply the original `224.0.0.251` packet being tunneled through
> the network.

The final endpoint capture proves that cEOS1 generates a new local mDNS
response.

This lab did not decode the internal DSO payload, so no stronger claim is made
about exactly how EOS represents service information inside the DSO session.

---

# 6. Containerlab Topology

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

# 7. Linux mDNS Route

Containerlab gives each Linux endpoint a management interface in addition to
the lab interface.

Initially, client1 selected its management interface for `224.0.0.251`:

```text
multicast 224.0.0.251 dev eth0 src 172.20.20.x
```

The topology therefore adds:

```text
ip route add 224.0.0.251/32 dev eth1
```

The resulting path becomes:

```text
multicast 224.0.0.251 dev eth1 src 10.10.10.2
```

This ensures the test uses the intended lab data plane.

---

# 8. Arista mDNS Gateway Configuration

Several separate configuration pieces are required.

## Enable mDNS globally

```text
mdns
   no disabled
```

Verification:

```text
show mdns status
```

Expected:

```text
mDNS is running
```

---

## Enable and identify the local mDNS link

cEOS1:

```text
interface Ethernet1
   mdns ipv4 link client1-link
```

cEOS2:

```text
interface Ethernet1
   mdns ipv4 link client2-link
```

Verification:

```text
show mdns links
```

Example:

```text
Interface    Address Family    Link ID         Status
Ethernet1    ipv4              client1-link    active
```

The Link ID identifies the local mDNS domain to remote gateways.

---

## Configure remote gateways

cEOS1:

```text
remote-gateway ipv4 10.10.100.2
```

cEOS2:

```text
remote-gateway ipv4 10.10.100.1
```

---

## Enable DSO

On both routers:

```text
dso server ipv4
```

Verification:

```text
show mdns status
```

Expected:

```text
DSO server is running
Gateway DSO connections ... connected
```

---

## Configure service rules

cEOS1:

```text
service test
   type any
   query Ethernet1
   response interface Ethernet1
   response link client2-link
```

cEOS2:

```text
service test
   type any
   query Ethernet1
   response interface Ethernet1
   response link client1-link
```

Meaning:

```text
type any
```

allows any DNS-SD service type for this lab.

```text
query Ethernet1
```

accepts queries from the local endpoint network.

```text
response interface Ethernet1
```

accepts locally learned service responses.

```text
response link clientX-link
```

allows service information associated with the remote named mDNS link to
participate in the rule.

The Link ID plus `response link` configuration was the critical missing piece
during troubleshooting.

---

# 9. Final Relevant Router Configuration

## cEOS1

```text
interface Ethernet1
   no switchport
   ip address 10.10.10.1/30
   mdns ipv4 link client1-link
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
      response link client2-link
```

## cEOS2

```text
interface Ethernet1
   no switchport
   ip address 10.10.20.1/30
   ip igmp
   mdns ipv4 link client2-link
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
      response link client1-link
```

Full configurations are stored in:

```text
config/ceos1.cfg
config/ceos2.cfg
```

---

# 10. Test Service

The endpoint image did not contain:

```text
avahi-browse
avahi-publish
dns-sd
zeroconf
```

A small Python standard-library advertiser was therefore used on client2.

The service was:

```text
Service type:     _http._tcp.local
Instance:         Test Web._http._tcp.local
Hostname:         client2.local
Port:             8080
IPv4 address:     10.10.20.2
TXT:              path=/
```

The advertisement contained:

```text
PTR
SRV
TXT
A
```

records and was transmitted every five seconds.

Terminal 1 was used to keep this advertiser running throughout the discovery
test.

---

# 11. Local Service Learning

With the advertiser running on client2:

```text
show mdns service type
```

on cEOS2 showed:

```text
Ethernet1    ipv4    _http._tcp.
```

and:

```text
show mdns service rule test
```

showed:

```text
Test Web._http._tcp.local.    Ethernet1
```

This proved that cEOS2 successfully received and understood the local DNS-SD
advertisement.

---

# 12. Discovery Test

Three terminals were used.

## Terminal 1 - advertiser

client2 continuously advertised:

```text
Test Web._http._tcp.local
```

---

## Terminal 2 - packet capture

client1 waited for an mDNS packet not sourced by itself:

```bash
docker exec -it clab-mdns-gateway-client1 \
  tcpdump -ni eth1 -vv -c 1 \
  'udp port 5353 and not src host 10.10.10.2'
```

---

## Terminal 3 - browse query

client1 sent:

```text
PTR _http._tcp.local
```

to:

```text
224.0.0.251:5353
```

from:

```text
10.10.10.2:5353
```

---

# 13. Successful Response

tcpdump captured:

```text
10.10.10.1.5353 > 224.0.0.251.5353

_http._tcp.local.
PTR Test Web._http._tcp.local.

Test Web._http._tcp.local.
SRV client2.local.:8080

client2.local.
A 10.10.20.2

Test Web._http._tcp.local.
TXT "path=/"
```

The response also contained an Arista gateway-related record:

```text
gw._arista._udp.local.
TXT "type=gw" "location="
```

The source address is the most important detail:

```text
10.10.10.1
```

That is cEOS1.

The original service lives at:

```text
10.10.20.2
```

Therefore, cEOS1 is generating a **new local mDNS response** containing service
information learned through the gateway relationship.

---

# 14. Wireshark Evidence

The successful capture was saved as:

```text
captures/lab06-mdns-gateway-success.pcap
```

PCAP files are ignored by Git and are therefore not committed to the
repository.

The capture was copied to the Windows Utility VM and inspected in Wireshark.

It contained two key packets.

## Packet 1

```text
10.10.10.2 -> 224.0.0.251

Standard query
PTR _http._tcp.local
```

## Packet 2

```text
10.10.10.1 -> 224.0.0.251

Standard query response

PTR Test Web._http._tcp.local
TXT
SRV 0 0 8080 client2.local
A 10.10.20.2
```

This is the primary packet-level proof that the mDNS Gateway works.

Useful Wireshark filters:

```text
mdns
```

or:

```text
udp.port == 5353
```

---

# 15. Reproducibility

The final cEOS configurations were saved and exported into:

```text
config/ceos1.cfg
config/ceos2.cfg
```

The complete lab was then destroyed and redeployed:

```bash
sudo containerlab destroy -t topology.clab.yml

sudo containerlab deploy \
  -t topology.clab.yml \
  --reconfigure
```

After redeploy, both routers automatically returned to the expected state.

cEOS1:

```text
mDNS is running
DSO server is running
10.10.100.2:8853 connected
Ethernet1 / client1-link active
response link client2-link
```

cEOS2:

```text
mDNS is running
DSO server is running
10.10.100.1:8853 connected
Ethernet1 / client2-link active
response link client1-link
```

The service advertiser and browse query were then repeated.

The same successful response appeared again:

```text
10.10.10.1.5353 > 224.0.0.251.5353

PTR Test Web._http._tcp.local
SRV client2.local:8080
A 10.10.20.2
TXT path=/
```

Therefore the final Lab 06 configuration is reproducible.

---

# 16. Troubleshooting Lessons

The major failures revealed distinct requirements.

### mDNS configured but not running

Symptom:

```text
mDNS is disabled
```

Fix:

```text
mdns
   no disabled
```

---

### Remote gateway remained `connecting`

Cause:

```text
DSO server is disabled
```

Fix:

```text
dso server ipv4
```

on both gateways.

---

### tcpdump saw packets but EOS counters remained zero

Cause:

Ethernet1 was not enabled as an mDNS link.

Fix:

```text
interface Ethernet1
   mdns ipv4
```

---

### Local service worked but remote discovery did not

The gateways were connected, but the remote mDNS links had not been identified
and referenced by the service policy.

Fix:

```text
mdns ipv4 link client1-link
mdns ipv4 link client2-link
```

and:

```text
response link <remote-link-id>
```

After this, DSO activity and gateway-generated mDNS responses appeared.

---

# 17. Useful EOS Commands

```text
show mdns status
```

Checks the global process, DSO server and peer state.

```text
show mdns links
```

Checks local mDNS interfaces and Link IDs.

```text
show mdns counters
```

Shows mDNS and DSO processing counters.

```text
show mdns service type
```

Shows learned DNS-SD service types.

```text
show mdns service rule test
```

Shows the active service policy and learned service instances.

---

# 18. Final Mental Model

## PIM multicast

```text
source multicast packet
        |
        v
router
        |
        | packet forwarded through multicast tree
        v
router
        |
        v
receiver
```

## Arista mDNS Gateway

```text
service device
      |
      | local mDNS
      v
mDNS Gateway
      ||
      || gateway state / DSO relationship
      ||
mDNS Gateway
      |
      | new local mDNS response
      v
discovering client
```

That distinction is the main lesson of Labs 04–06.

---

# 19. Standards

The concepts used in this lab are defined primarily by:

### RFC 6762 - Multicast DNS

Defines mDNS, including:

```text
224.0.0.251
UDP 5353
.local
```

### RFC 6763 - DNS-Based Service Discovery

Defines DNS-SD and service discovery using records such as:

```text
PTR
SRV
TXT
```

### RFC 8490 - DNS Stateful Operations

Defines the DSO framework for persistent stateful DNS sessions.

The exact EOS configuration syntax used here was verified directly against:

```text
cEOS 4.36.1F
```

---

# 20. Final Takeaways

1. mDNS is deliberately local-link oriented.
2. PIM does not automatically provide cross-subnet mDNS discovery.
3. DNS-SD describes services using standard DNS records.
4. Arista mDNS Gateway is application-aware rather than simple multicast
   forwarding.
5. Participating interfaces require `mdns ipv4`.
6. Link IDs identify local mDNS domains to remote gateways.
7. Service rules determine which local and remote links participate.
8. DSO gateway peering must be established.
9. A connected gateway peer alone is not sufficient; the service policy and
   Link IDs must also be correct.
10. Wireshark proved that cEOS1 generated a local mDNS response containing the
    remote service information.
11. The entire configuration survived a full destroy/redeploy and the final
    discovery test succeeded again.

---

# Final Result

```text
client2
10.10.20.2
      |
      | advertises
      | Test Web._http._tcp.local
      v
    cEOS2
      ||
      || mDNS Gateway
      ||
    cEOS1
      |
      | generates local response
      v
client1
10.10.10.2
```

client1 successfully received enough DNS-SD information to identify:

```text
Service:  Test Web._http._tcp.local
Host:     client2.local
Address:  10.10.20.2
Port:     8080
Metadata: path=/
```

The Layer 3 mDNS discovery boundary demonstrated in Lab 05 was therefore
successfully bridged using Arista EOS mDNS Gateway.
