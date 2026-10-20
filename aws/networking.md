## DevOps/SRE networking interview cheat sheet


### Networking — Senior DevOps/SRE Interview Cheatsheet

A strong networking understanding over cloud should show that you can trace a request **end to end**:

```text
User
 ↓
DNS
 ↓
Internet
 ↓
CDN
 ↓
WAF
 ↓
Load Balancer
 ↓
Ingress
 ↓
Kubernetes Service
 ↓
Pod
 ↓
Application
 ↓
Database
```

The key interview habit is:

> **At every hop, ask: Did the request arrive? Did the connection establish? Did the next hop respond? Where did latency or packet loss appear?**

---

## 1. End-to-End Request Flow

Assume:

```text
https://api.company.com/orders
```

A user sends a request.

Conceptually:

```text
Browser
   ↓
DNS lookup
   ↓
api.company.com → public IP / ALB hostname
   ↓
Internet
   ↓
CloudFront/CDN
   ↓
AWS WAF
   ↓
Application Load Balancer
   ↓
EKS Ingress / Target
   ↓
Kubernetes Service
   ↓
Pod
   ↓
Application
   ↓
RDS/Aurora
```

A senior-level answer should be able to explain **what can fail at each layer**.

For example:

| Layer | Possible issue |
|---|---|
| DNS | NXDOMAIN, stale cache, wrong record |
| CDN | origin unavailable, stale configuration |
| WAF | request blocked |
| ALB | unhealthy targets |
| Ingress | routing rule mismatch |
| Service | wrong selector / no endpoints |
| Pod | not listening / overloaded |
| App | thread/connection pool exhausted |
| DB | slow queries / connection exhaustion |

---

# 2. TCP — The Foundation

TCP provides:

- connection-oriented communication
- ordered delivery
- retransmission
- flow control
- congestion control

HTTP/1.1 and HTTP/2 commonly run over TCP.

---

# 3. TCP 3-Way Handshake

Before application data flows:

```text
Client                           Server
  |                                |
  | -------- SYN ----------------> |
  |                                |
  | <------ SYN-ACK ---------------|
  |                                |
  | -------- ACK ----------------> |
  |                                |
  |      Connection established    |
```

### SYN

Client says:

> I want to establish a connection.

### SYN-ACK

Server says:

> I received your request and I'm ready.

### ACK

Client confirms.

Then application traffic can flow.

---

# 4. How to Troubleshoot TCP Handshake Failure

Suppose:

```bash
curl https://api.company.com
```

hangs.

Capture:

```bash
tcpdump -nn host <server-ip>
```

If you see:

```text
Client → SYN
Client → SYN
Client → SYN
```

but **no SYN-ACK**, investigate:

- Security Group
- NACL
- firewall
- route table
- server not listening
- load balancer problem
- wrong IP
- network path

If:

```text
SYN
SYN-ACK
ACK
```

works, networking at that level is probably okay.

Now investigate TLS/application behavior.

This distinction is extremely useful in interviews.

---

# 5. TCP Connection Termination

Normal termination uses FIN/ACK.

Conceptually:

```text
Client                           Server

FIN ---------------------------->
             <---------------- ACK

             <---------------- FIN
ACK ----------------------------->
```

Connections can also terminate with:

```text
RST
```

Reset usually means the connection was terminated abruptly.

Examples:

```text
connection reset by peer
```

can happen if:

- application crashes
- load balancer resets connection
- firewall rejects connection
- application explicitly closes unexpectedly

---

# 6. TCP Retransmissions

TCP expects packets to arrive reliably.

If a packet doesn't arrive:

```text
Packet sent
    ↓
No ACK
    ↓
Timeout / duplicate ACK
    ↓
Retransmission
```

High retransmissions may indicate:

- packet loss
- network congestion
- overloaded devices
- bad network path
- MTU issues

Useful:

```bash
ss -s
```

and:

```bash
netstat -s
```

or packet capture:

```bash
tcpdump
```

---

# 7. Network Latency vs Application Latency

A request might take:

```text
2 seconds
```

but network latency may only be:

```text
20 ms
```

The rest might be:

```text
Database       900 ms
External API   700 ms
Queue wait     300 ms
Application     80 ms
Network         20 ms
```

Therefore:

> Don't blame "the network" simply because requests are slow.

You need to determine **where time is spent**.

---

# 8. TCP Congestion Control

TCP adjusts how much data it sends based on perceived network conditions.

Conceptually:

```text
Send slowly
   ↓
Network healthy
   ↓
Increase sending rate
   ↓
Packet loss detected
   ↓
Reduce sending rate
```

Important concepts:

- congestion window
- slow start
- packet loss
- RTT
- bandwidth-delay product

You normally don't need algorithm-level detail unless interviewing for network-heavy SRE roles, but know why:

> Packet loss can dramatically reduce TCP throughput.

---

# 9. Connection Timeout

Important distinction:

### Connect timeout

How long you wait to establish TCP.

Example:

```text
SYN
 ↓
waiting
 ↓
no SYN-ACK
 ↓
connect timeout
```

### Read timeout

TCP connection exists, but application isn't returning data quickly enough.

```text
Connection established
      ↓
request sent
      ↓
waiting for response
      ↓
read timeout
```

This distinction is extremely useful.

Example interview answer:

> A connect timeout makes me investigate network reachability, routing, firewalls, listener availability, or overloaded connection queues. A read timeout means the connection was established, so I'd focus more on application processing, downstream services, or database latency.

---

# 10. TCP Keepalive

Keepalive checks whether an otherwise idle TCP connection is still alive.

Different from HTTP keep-alive.

TCP keepalive:

```text
Connection idle
     ↓
Kernel probes peer
     ↓
Peer responds
     ↓
Connection remains alive
```

Useful for detecting dead peers.

---

# 11. HTTP Keep-Alive

HTTP keep-alive means reusing a TCP connection for multiple HTTP requests.

Without reuse:

```text
Request 1
TCP handshake
TLS handshake
HTTP request
close

Request 2
TCP handshake
TLS handshake
HTTP request
close
```

Expensive.

With connection reuse:

```text
TCP + TLS established
       ↓
Request 1
Request 2
Request 3
Request 4
```

Much more efficient.

---

# 12. Connection Pooling

Applications commonly maintain pools.

Example:

```text
Application
    |
    +---- DB connection 1
    +---- DB connection 2
    +---- DB connection 3
    +---- DB connection 4
```

Instead of creating a new DB connection per request.

### Classic incident

```text
200 application threads
20 DB connections
```

All 20 connections are busy.

Remaining requests wait.

CPU:

```text
35%
```

but latency becomes huge.

This is why:

> Low CPU doesn't mean the application has capacity.

---

# 13. Ephemeral Ports

When a client opens an outbound TCP connection, the OS chooses a temporary source port.

Example:

```text
Client:

10.0.1.20:53214
```

connects to:

```text
Database:

10.0.5.20:5432
```

Connection tuple:

```text
10.0.1.20:53214 → 10.0.5.20:5432
```

`53214` is an ephemeral port.

---

# 14. Ephemeral Port Exhaustion

This is an excellent senior interview scenario.

Suppose one application opens enormous numbers of outbound connections.

```text
Application
   ↓
Thousands of connections
   ↓
Ephemeral ports consumed
   ↓
New connections fail
```

Symptoms:

```text
connect timeout
cannot assign requested address
connection failures
```

Check:

```bash
ss -tan
```

and Linux ephemeral port range:

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

Connection reuse/pooling can dramatically reduce this problem.

---

# 15. TIME_WAIT

After TCP closes, one side may remain in:

```text
TIME_WAIT
```

temporarily.

Purpose:

- prevent delayed packets from old connections interfering with new connections
- ensure final connection state settles correctly

Check:

```bash
ss -tan state time-wait
```

Large amounts of TIME_WAIT often indicate:

```text
lots of short-lived connections
```

Possible fixes:

- use HTTP keep-alive
- connection pooling
- avoid unnecessarily recreating connections

Do **not** immediately start tuning kernel parameters.

Senior answer:

> I first determine why the application is creating so many short-lived connections before changing TCP settings.

---

# 16. HTTP/1.1

HTTP/1.1 commonly uses TCP.

Historically:

```text
TCP connection
   ↓
Request
   ↓
Response
```

Keep-alive allows connection reuse.

One limitation is concurrency behavior compared with newer HTTP versions.

Browsers often compensate by establishing multiple TCP connections.

---

# 17. HTTP/2

HTTP/2 allows multiple logical streams over one TCP connection.

```text
             TCP Connection
                  |
      ----------------------------
      |             |            |
   Stream 1      Stream 2      Stream 3
```

Advantages:

- multiplexing
- header compression
- fewer TCP connections
- better connection efficiency

But because HTTP/2 still relies on TCP, packet loss can impact multiple streams sharing that connection.

---

# 18. HTTP/3

HTTP/3 uses:

```text
QUIC
```

over:

```text
UDP
```

rather than TCP.

Conceptually:

```text
HTTP/1.1 → TCP
HTTP/2   → TCP
HTTP/3   → QUIC/UDP
```

Benefits include improved handling of connection setup and packet loss across independent streams.

For interviews, that high-level distinction is usually enough.

---

# 19. HTTP Headers

Important headers to understand:

```text
Host
Authorization
Content-Type
Accept
User-Agent
Cookie
Set-Cookie
Cache-Control
X-Forwarded-For
X-Forwarded-Proto
```

Especially important behind proxies/load balancers:

```text
X-Forwarded-For
```

may preserve the original client IP.

Example:

```text
User → ALB → NGINX → App
```

The app may otherwise only see the proxy's IP.

---

# 20. Cookies

Cookies allow servers to maintain client state.

Example:

```text
Browser
  ↓ login
Server

Set-Cookie:
session=abc123
```

Future requests:

```text
Cookie:
session=abc123
```

Important security attributes:

```text
Secure
HttpOnly
SameSite
```

---

# 21. TLS

HTTPS is essentially:

```text
HTTP
 ↓
TLS
 ↓
TCP
```

For HTTP/1.1 and HTTP/2.

TLS provides:

- encryption
- integrity
- authentication

---

# 22. TLS Handshake — Interview Level

Simplified:

```text
Client
   |
Client Hello
   ↓
Server
   |
Server Hello
Certificate
   ↓
Client validates certificate
   ↓
Key agreement
   ↓
Encrypted session established
```

The client validates things such as:

- hostname
- certificate expiry
- trust chain
- certificate authority

---

# 23. TLS Production Failure

Example:

```bash
curl https://api.company.com
```

returns:

```text
certificate has expired
```

Possible causes:

- expired certificate
- wrong certificate
- hostname mismatch
- missing intermediate certificate
- certificate rotation failed

Useful:

```bash
openssl s_client -connect api.company.com:443 \
-servername api.company.com
```

Then inspect:

```text
subject
issuer
expiry
certificate chain
```

---

# 24. DNS Resolution

When resolving:

```text
api.company.com
```

conceptually:

```text
Client
  ↓
Recursive Resolver
  ↓
Root DNS
  ↓
.com TLD
  ↓
company.com Authoritative DNS
  ↓
api.company.com
  ↓
IP / target
```

Important nuance:

The client usually doesn't directly query every level itself.

It asks a **recursive resolver**, which performs or obtains the lookup.

---

# 25. Recursive vs Authoritative DNS

### Recursive resolver

Finds the answer on behalf of the client.

Examples might include:

- ISP resolver
- corporate resolver
- cloud DNS resolver

### Authoritative server

Owns the DNS records for the domain.

Example:

```text
company.com
```

authoritative DNS says:

```text
api.company.com → something
```

---

# 26. DNS Records

### A

IPv4 address.

```text
api.company.com → 203.0.113.10
```

### AAAA

IPv6.

```text
api.company.com → 2001:db8::1
```

### CNAME

Alias.

```text
api.company.com
      ↓
my-alb.amazonaws.com
```

### NS

Specifies authoritative name servers.

### TXT

Arbitrary text.

Often used for:

- domain verification
- SPF
- DKIM
- security validation

---

# 27. DNS TTL

TTL determines how long DNS information may be cached.

Example:

```text
TTL = 300 seconds
```

Resolvers may cache the answer for five minutes.

This matters during migrations.

Suppose:

```text
Old ALB → 1.2.3.4
New ALB → 5.6.7.8
```

You update DNS.

Some clients may still use the old cached answer until TTL expires.

### Migration practice

Before major migration:

```text
TTL 3600
 ↓
lower to 60/300 beforehand
 ↓
perform cutover
 ↓
raise TTL afterward
```

---

# 28. DNS Debugging

Useful:

```bash
dig api.company.com
```

Query particular record:

```bash
dig A api.company.com
```

Trace delegation:

```bash
dig +trace company.com
```

Check resolver:

```bash
nslookup api.company.com
```

Useful interview flow:

```text
Does hostname resolve?
       ↓
Correct record?
       ↓
Correct authoritative server?
       ↓
Old cache?
       ↓
Correct IP/target?
```

---

# 29. AWS VPC

Think of a VPC as an isolated virtual network.

Example:

```text
VPC
10.0.0.0/16
```

Inside:

```text
Public Subnets
Private Subnets
Database Subnets
```

---

# 30. CIDR

Example:

```text
10.0.0.0/16
```

means a block of IP addresses.

Possible subnet split:

```text
VPC
10.0.0.0/16

AZ-A
Public:   10.0.1.0/24
Private:  10.0.11.0/24

AZ-B
Public:   10.0.2.0/24
Private:  10.0.12.0/24
```

Important interview topic:

> Don't make VPC/subnet CIDRs so small that EKS runs out of IP addresses.

Very important with Kubernetes because pods may consume VPC IPs depending on the CNI/networking design.

---

# 31. Public vs Private Subnet

The subnet itself isn't magically "public."

It becomes effectively public when its routing and addressing permit internet connectivity.

Typical public subnet route:

```text
0.0.0.0/0 → Internet Gateway
```

Typical private subnet route:

```text
0.0.0.0/0 → NAT Gateway
```

Private resources do not have direct inbound internet reachability.

---

# 32. Route Table

A route table determines:

> Where should this packet go?

Example:

```text
Destination       Target

10.0.0.0/16       local
0.0.0.0/0         nat-abc123
```

Meaning:

```text
Traffic inside VPC
     ↓
local route

Everything else
     ↓
NAT Gateway
```

---

# 33. Internet Gateway

IGW connects the VPC to the internet.

Architecture:

```text
Internet
   |
Internet Gateway
   |
Public subnet
```

A public-facing ALB typically uses public subnets connected appropriately to an IGW.

---

# 34. NAT Gateway

Allows private resources to initiate outbound internet communication.

Example:

```text
EKS Node
Private Subnet
      |
      ↓
NAT Gateway
Public Subnet
      |
      ↓
Internet Gateway
      |
      ↓
Internet
```

Common use:

```text
Private EKS worker
   ↓
pull container image / contact external API
```

while not exposing the node directly to inbound internet traffic.

---

# 35. NAT Gateway Production Issue

Classic scenario:

```text
Pods cannot reach external APIs.
```

Investigate:

```text
Pod
 ↓
Node/ENI
 ↓
Route table
 ↓
NAT Gateway
 ↓
IGW
 ↓
Internet
```

Possible problems:

- private subnet missing default route
- NAT Gateway unavailable/misconfigured
- NACL blocks traffic
- DNS problem
- ephemeral/SNAT port exhaustion
- downstream endpoint blocking source IP

---

# 36. Security Groups

Security Groups are **stateful** virtual firewalls.

Example DB SG:

```text
Inbound:

TCP 5432
Source:
Application Security Group
```

Rather than:

```text
0.0.0.0/0 → 5432
```

A strong architecture:

```text
ALB SG
  ↓
EKS/App SG
  ↓
Database SG
```

Allow only required traffic.

---

# 37. NACL

Network ACL operates at subnet level.

Important distinction:

```text
Security Group = stateful
NACL           = stateless
```

Because NACLs are stateless, inbound and outbound rules both matter.

Typical interview question:

> SG allows the traffic, but communication still fails. Why?

Possible answer:

> The subnet NACL could still be blocking the packet or return traffic.

---

# 38. Security Group vs NACL

Remember:

| Security Group | NACL |
|---|---|
| Resource/ENI level | Subnet level |
| Stateful | Stateless |
| Allow rules | Allow + deny rules |
| Return traffic automatically tracked | Return traffic must be allowed appropriately |

---

# 39. VPC Endpoint

VPC endpoints allow private access to supported AWS services without traversing the public internet path.

Example:

```text
Private EKS Node
      ↓
VPC Endpoint
      ↓
S3
```

Instead of:

```text
Private subnet
 ↓
NAT Gateway
 ↓
Internet/public service endpoint
```

Benefits can include:

- reduced NAT dependency
- private connectivity
- potentially lower NAT processing costs
- improved security architecture

---

# 40. Gateway vs Interface Endpoint

High-level distinction:

### Gateway endpoint

Traditionally used for services such as:

```text
S3
DynamoDB
```

### Interface endpoint

Creates network interfaces in your VPC and uses AWS PrivateLink.

Conceptually:

```text
Application
 ↓
Private IP
 ↓
Interface Endpoint
 ↓
AWS Service
```

---

# 41. VPC Peering

Connects two VPCs privately.

```text
VPC-A
  |
Peering
  |
VPC-B
```

Important limitation:

> VPC peering is not inherently transitive.

If:

```text
A ↔ B
B ↔ C
```

that does not automatically mean:

```text
A ↔ C
```

This is a frequent interview question.

---

# 42. Transit Gateway

Useful for connecting many VPCs/networks.

Without it:

```text
VPC-A ↔ VPC-B
  ↕        ↕
VPC-C ↔ VPC-D
```

Complex mesh.

With Transit Gateway:

```text
          VPC-A
            |
VPC-B — Transit Gateway — VPC-C
            |
          VPC-D
```

It acts as a network hub.

Useful for:

- multi-VPC architecture
- hybrid connectivity
- centralized networking

---

# 43. PrivateLink

PrivateLink allows private service exposure through interface endpoints.

Example:

```text
Consumer VPC
     ↓
Interface Endpoint
     ↓
PrivateLink
     ↓
Provider Service
```

Useful when you want to expose a service privately without broad network connectivity between entire VPCs.

### Peering vs PrivateLink

Simplified:

```text
Peering:
network-to-network connectivity

PrivateLink:
service-oriented private connectivity
```

---

# 44. AWS DNS / Route 53

Route 53 can provide:

- public hosted zones
- private hosted zones
- DNS records
- routing policies
- health-check-based routing

Example:

```text
api.company.com
       ↓
Route53
       ↓
ALB
```

Private hosted zone:

```text
db.internal.company.com
```

resolvable inside associated VPCs.

---

# 45. Load Balancer Basics

Two important AWS types you'll discuss often:

### ALB

Layer 7.

Understands:

```text
HTTP
HTTPS
hostnames
paths
headers
```

Can route:

```text
/api → API service
/web → frontend service
```

### NLB

Layer 4.

Primarily operates on:

```text
TCP
UDP
TLS
```

Good for high-performance transport-level workloads or cases needing L4 behavior.

---

# 46. ALB Health Checks

Suppose ALB returns:

```text
503 Service Unavailable
```

Possible reason:

```text
Targets unhealthy
```

Flow:

```text
ALB
 ↓ health check
/health
 ↓
Pod/Application
```

If `/health` returns failure:

```text
ALB marks target unhealthy
```

Then no traffic gets forwarded.

Questions:

```text
Is target registered?
Is health-check path correct?
Is port correct?
Can ALB SG reach target?
Does application return 200?
```

---

# 47. Kubernetes Ingress

Ingress provides HTTP/HTTPS routing into cluster workloads.

Example:

```yaml
host: api.company.com

/api
   ↓
api-service
```

Conceptually:

```text
ALB
 ↓
Ingress
 ↓
Kubernetes Service
 ↓
Pod
```

Depending on the ingress implementation, ALB may integrate directly with targets differently, but that mental model is good for interviews.

---

# 48. Kubernetes Service

Pods are ephemeral.

Pod IP might change:

```text
10.0.11.42
       ↓
pod dies
       ↓
10.0.12.87
```

Applications shouldn't generally depend directly on individual pod addresses.

Service gives a stable abstraction:

```text
Service
   ↓
Pod A
Pod B
Pod C
```

Example:

```text
orders-service:8080
```

---

# 49. Service Selector Issue

Classic Kubernetes networking incident.

Service:

```yaml
selector:
  app: orders
```

Pods:

```yaml
labels:
  app: order-api
```

No match.

Result:

```text
Service
   ↓
NO ENDPOINTS
```

Check:

```bash
kubectl get svc
kubectl get endpoints
kubectl get pods --show-labels
```

This is a very good interview example because the pods may be healthy while the service is still unusable.

---

# 50. Kubernetes DNS

Inside Kubernetes:

```text
orders-service
```

may resolve through cluster DNS.

Typical FQDN:

```text
orders-service.production.svc.cluster.local
```

Flow:

```text
Pod
 ↓
CoreDNS
 ↓
Service DNS record
 ↓
Service/endpoint
```

If service communication fails:

```bash
nslookup orders-service
```

or:

```bash
dig orders-service
```

from a debugging pod.

---

# 51. NetworkPolicy

Kubernetes NetworkPolicy controls pod-level traffic when the chosen networking implementation enforces it.

Example:

```text
Frontend
   ↓ allowed
Backend
   ↓ allowed
Database

Other pod
   X denied
Database
```

A production issue might be:

```text
Pod healthy
Service healthy
DNS resolves
TCP connection times out
```

Check NetworkPolicy.

---

# 52. Real Project Architecture

Here is a realistic interview project.

You can describe it as:

> We ran a highly available customer-facing application on AWS EKS across multiple Availability Zones, with Route 53, WAF, ALB, private worker nodes, RDS/Aurora, and private AWS service connectivity.

Architecture:

```text
                         Users
                           |
                           ↓
                       Route 53
                           |
                           ↓
                     CloudFront
                           |
                           ↓
                         WAF
                           |
                           ↓
                Application Load Balancer
                    /                 \
                   /                   \
                AZ-A                  AZ-B
             Public Subnet         Public Subnet
                   |                   |
                   +--------+----------+
                            |
                            ↓
                          EKS
                   /                 \
                  /                   \
             Private Subnet       Private Subnet
                 AZ-A                 AZ-B
                  |                   |
              Pods/Nodes          Pods/Nodes
                  \                   /
                   \                 /
                    \               /
                       RDS/Aurora
                    Private DB Subnets

Private EKS egress:
Pods
 ↓
NAT Gateway / VPC Endpoints
 ↓
External APIs / AWS Services
```

---

# 53. How to Explain This Project in an Interview

A concise answer:

> We deployed the application in an AWS VPC across three Availability Zones. Route 53 handled DNS, CloudFront provided edge delivery, and AWS WAF filtered malicious traffic before requests reached an internet-facing ALB. The ALB routed requests into EKS workloads running in private subnets. Kubernetes Services provided stable service discovery between workloads, and CoreDNS handled internal DNS resolution. The database ran in isolated private DB subnets and only accepted connections from the application security groups. EKS nodes and pods used NAT gateways for controlled internet egress, while high-volume AWS service traffic such as S3/ECR was moved to VPC endpoints where appropriate. Infrastructure was defined through Terraform and application networking through Helm/Kubernetes manifests.

That sounds significantly stronger than:

> We deployed EKS behind an ALB.

---

# 54. Great Production Incident Example — 504 Gateway Timeout

Interview question:

> Users are getting 504 errors. How do you troubleshoot?

Don't immediately restart pods.

Trace:

```text
User
 ↓
DNS
 ↓
CloudFront/WAF
 ↓
ALB
 ↓
Ingress
 ↓
Service
 ↓
Pod
 ↓
DB
```

Suppose:

```text
ALB reports 504
```

Check target metrics.

Pods are healthy.

Application logs show:

```text
database connection acquisition timeout
```

DB connections:

```text
Pool size = 50
Active = 50
Waiting = 180
```

Root cause:

```text
Database connection pool exhausted
```

Requests waited too long.

ALB eventually timed them out.

CPU might still only be:

```text
30%
```

### Strong interview answer

> A 504 tells me an upstream component waited too long for its downstream response. I'd identify which component generated the 504 first—CloudFront, ALB, ingress, or application proxy—and then work downstream. In one incident, ALB targets were healthy and TCP connectivity was fine, but application traces showed requests waiting for DB connections. The connection pool had reached its maximum, which increased application response time beyond the ALB timeout. We mitigated the immediate load, optimized the slow queries and pool usage, then added alerts on pool saturation rather than simply increasing timeouts.

Very strong senior answer.

---

# 55. Another Incident — DNS Works but Service Doesn't

Scenario:

```text
api.company.com resolves correctly.
```

But requests time out.

You check:

```bash
dig api.company.com
```

Success.

Then:

```bash
curl -v https://api.company.com
```

hangs during TCP connection.

Packet capture shows:

```text
SYN → server
SYN → server
SYN → server
```

No SYN-ACK.

Now you know:

```text
DNS works
TLS hasn't started
HTTP hasn't started
```

So focus on:

```text
route
SG
NACL
listener
target
network path
```

This is the kind of layered reasoning interviewers want.

---

# 56. Another Incident — Pods Cannot Access Internet

Symptoms:

```text
EKS pods → external API
```

timeout.

Internal pod-to-pod communication works.

DNS works.

Trace:

```text
Pod
 ↓
Private subnet
 ↓
Route table
 ↓
NAT
 ↓
IGW
 ↓
Internet
```

You discover:

```text
Private route table:
NO 0.0.0.0/0 → NAT
```

Root cause:

> A Terraform change associated the private subnet with the wrong route table.

Resolution:

```text
Fix Terraform
 ↓
PR
 ↓
apply
 ↓
validate connectivity
```

This makes a good DevOps project story because it connects:

```text
Terraform configuration
           ↓
runtime AWS networking
           ↓
production incident
           ↓
observability/debugging
           ↓
PR-based permanent fix
```

---

# 57. Troubleshooting Commands to Master

You don't need 50 networking commands. Know what question each tool answers.

| Tool | Question |
|---|---|
| `dig` | Does DNS resolve correctly? |
| `nslookup` | What does DNS return? |
| `curl -v` | Where does HTTP/TLS fail? |
| `ss` | What sockets/connections exist? |
| `ip addr` | What addresses/interfaces exist? |
| `ip route` | Where will traffic be routed? |
| `ping` | Is basic ICMP reachability available? |
| `traceroute` | What network path is taken? |
| `tcpdump` | What packets actually crossed the interface? |
| `openssl s_client` | Is TLS/certificate negotiation correct? |

Especially master:

```bash
dig
curl -v
ss
ip route
tcpdump
openssl s_client
```

---

# 58. Senior Troubleshooting Framework

When asked:

> "The application isn't reachable."

Think:

```text
1. DNS?
      ↓
2. Routing?
      ↓
3. TCP connection?
      ↓
4. TLS?
      ↓
5. Load balancer?
      ↓
6. Kubernetes ingress/service?
      ↓
7. Pod listener?
      ↓
8. Application?
      ↓
9. Downstream dependency?
```

Example commands:

```bash
dig api.company.com

curl -vk https://api.company.com

ss -lntp

ip route

tcpdump -nn host <IP>

openssl s_client \
  -connect api.company.com:443 \
  -servername api.company.com
```

Kubernetes:

```bash
kubectl get ingress

kubectl get svc

kubectl get endpoints

kubectl get pods -o wide

kubectl describe ingress

kubectl exec <pod> -- \
  curl http://orders-service:8080/health
```

---

# 59. The OSI Model — Only Learn the Useful Version

You don't need to recite all seven layers mechanically.

For DevOps interviews think:

```text
Layer 7
HTTP / DNS / TLS/application protocols

Layer 4
TCP / UDP / ports

Layer 3
IP / routing

Layer 2
Ethernet / MAC / switching
```

Troubleshooting:

```text
DNS resolves?
      ↓
L3 routing works?
      ↓
L4 TCP works?
      ↓
TLS works?
      ↓
HTTP works?
      ↓
Application works?
```

---

# 60. High-Value Interview Questions

Make sure you can answer these quickly:

- What happens when you type `https://google.com` in a browser?
- Explain TCP 3-way handshake.
- Why do we need `TIME_WAIT`?
- What causes TCP retransmissions?
- Connect timeout vs read timeout?
- TCP keepalive vs HTTP keep-alive?
- What is ephemeral port exhaustion?
- HTTP/1.1 vs HTTP/2 vs HTTP/3?
- Explain TLS handshake.
- How does DNS resolution work?
- Recursive vs authoritative DNS?
- A vs AAAA vs CNAME?
- What does TTL do?
- Security Group vs NACL?
- Public vs private subnet?
- NAT Gateway vs Internet Gateway?
- VPC peering vs Transit Gateway?
- Peering vs PrivateLink?
- VPC endpoint vs NAT Gateway?
- ALB vs NLB?
- Kubernetes Service vs Ingress?
- Service resolves but has no endpoints—why?
- Pod is healthy but service unreachable—what do you check?
- ALB returns 502 vs 503 vs 504—how do you investigate?
- EKS pods cannot access internet—how do you debug?
- DNS resolves correctly but connection times out—what next?

---

# 61. One Answer Worth Memorizing

If the interviewer says:

> **"Users report the application is unreachable. Walk me through troubleshooting."**

Give something like:

> I'd troubleshoot it layer by layer rather than restarting services. First I'd verify DNS resolution and confirm the hostname points to the expected endpoint. Then I'd test TCP reachability to determine whether the connection handshake completes. If TCP works, I'd inspect TLS negotiation and certificate validity, followed by the HTTP response and load-balancer status. From there I'd trace traffic through the ingress, Kubernetes Service and endpoints to the pod, verify the process is listening on the expected port, and finally check downstream dependencies such as the database. In AWS I'd also verify route tables, security groups and NACLs where relevant. Packet captures or `curl -v` help me establish exactly which layer is failing.

That answer communicates the key senior trait:

> **You isolate the failing layer instead of guessing.**

And for the AWS/EKS project, the mental model worth memorizing is:

```text
Route53
  ↓
CloudFront
  ↓
WAF
  ↓
ALB
  ↓
EKS Ingress
  ↓
Service
  ↓
Pod
  ↓
RDS

Private egress:
Pod → NAT/VPC Endpoint → destination
```

For interviews, be prepared to point at **every arrow** and explain: **DNS resolution, route selection, firewall policy, TCP connection, TLS termination, load balancing, service discovery, and application dependency behavior.**