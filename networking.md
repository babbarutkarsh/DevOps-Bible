# Networking — DevOps Interview Preparation

## Table of Contents

- [OSI Model & TCP/IP](#osi-model--tcpip)
- [DNS Deep Dive](#dns-deep-dive)
- [HTTP/HTTPS & TLS](#httphttps--tls)
- [Load Balancing](#load-balancing)
- [Proxies — Forward & Reverse](#proxies--forward--reverse)
- [CDN (Content Delivery Network)](#cdn-content-delivery-network)
- [Subnetting & CIDR](#subnetting--cidr)
- [VPN & Tunneling](#vpn--tunneling)
- [Firewalls & Network Security](#firewalls--network-security)
- [TCP vs UDP Deep Dive](#tcp-vs-udp-deep-dive)
- [NAT & Port Forwarding](#nat--port-forwarding)
- [Network Troubleshooting](#network-troubleshooting)
- [Cloud Networking Concepts](#cloud-networking-concepts)
- [Scenario-Based Questions](#scenario-based-questions)

---

## OSI Model & TCP/IP

### Q: Explain the OSI model and map it to real-world protocols.

```
OSI Layer          TCP/IP Model       Protocols / Examples
─────────────────────────────────────────────────────────
7. Application  ┐
6. Presentation ├─ Application       HTTP, HTTPS, DNS, SSH, FTP, SMTP, SNMP
5. Session      ┘

4. Transport    ── Transport         TCP, UDP, QUIC

3. Network      ── Internet          IP, ICMP, ARP, IGMP

2. Data Link    ┐
1. Physical     ┘─ Network Access    Ethernet, Wi-Fi, MAC addresses, Switches
```

**How data flows through layers (encapsulation):**

```
Application Data
       │
       ▼
┌──────────────────┐
│ [TCP Header][Data]│  ← Segment (Layer 4)
└──────────────────┘
       │
       ▼
┌─────────────────────────┐
│ [IP Header][TCP][Data]   │  ← Packet (Layer 3)
└─────────────────────────┘
       │
       ▼
┌──────────────────────────────────┐
│ [Eth Header][IP][TCP][Data][FCS] │  ← Frame (Layer 2)
└──────────────────────────────────┘
       │
       ▼
    Bits on wire/wireless (Layer 1)
```

### Q: What happens when you type `google.com` in a browser?

This is the **#1 most asked networking question** in interviews.

```
1. URL Parsing
   Browser parses: scheme (https), host (google.com), path (/)

2. DNS Resolution
   Browser cache → OS cache → /etc/hosts → Recursive resolver
   → Root DNS → .com TLD → google.com authoritative NS
   → Returns IP (e.g., 142.250.190.14)

3. TCP Connection (3-way handshake)
   Client → SYN → Server
   Client ← SYN+ACK ← Server
   Client → ACK → Server

4. TLS Handshake (for HTTPS)
   ClientHello (supported ciphers, TLS version)
   ServerHello (chosen cipher, certificate)
   Client verifies certificate chain → CA trust
   Key exchange (ECDHE) → Shared secret established
   Symmetric encryption begins

5. HTTP Request
   GET / HTTP/2
   Host: google.com
   User-Agent: Chrome/...
   Accept: text/html

6. Server Processing
   Load balancer → Web server → Application logic
   → Database queries → Template rendering

7. HTTP Response
   HTTP/2 200 OK
   Content-Type: text/html
   Content-Encoding: gzip
   [HTML body]

8. Browser Rendering
   Parse HTML → Build DOM tree
   Parse CSS → Build CSSOM
   Execute JavaScript
   Build Render Tree → Layout → Paint → Composite

9. Connection Management
   HTTP/2: Multiplexed streams over single TCP connection
   Keep-alive for subsequent requests
```

---

## DNS Deep Dive

### Q: Explain DNS record types.

| Record Type | Purpose | Example |
|-------------|---------|---------|
| **A** | Maps hostname → IPv4 | `web.example.com → 93.184.216.34` |
| **AAAA** | Maps hostname → IPv6 | `web.example.com → 2606:2800:220:1:...` |
| **CNAME** | Alias to another hostname | `www.example.com → web.example.com` |
| **MX** | Mail server for domain | `example.com → mail.example.com (priority 10)` |
| **NS** | Nameserver for zone | `example.com → ns1.example.com` |
| **TXT** | Text data (SPF, DKIM, verification) | `"v=spf1 include:_spf.google.com ~all"` |
| **SOA** | Start of Authority (zone metadata) | Serial, refresh, retry, expire |
| **PTR** | Reverse DNS (IP → hostname) | `34.216.184.93 → web.example.com` |
| **SRV** | Service location (port + host) | `_sip._tcp.example.com → sipserver:5060` |
| **CAA** | Certificate Authority Authorization | Restricts which CAs can issue certs |

### Q: What is the difference between authoritative and recursive DNS?

| Type | Role | Example |
|------|------|---------|
| **Recursive Resolver** | Takes client query and finds the answer by querying other DNS servers | 8.8.8.8, 1.1.1.1, ISP DNS |
| **Authoritative Server** | Holds the actual DNS records for a zone and gives definitive answers | ns1.google.com for google.com |

```
Client → Recursive Resolver → Root (.) → TLD (.com) → Authoritative (example.com)
                                                              │
                                                              ▼
                                                         Returns A record
```

### Q: What is DNS TTL and how does it affect deployments?

**TTL (Time To Live)** tells resolvers how long to cache a record (in seconds).

| TTL Value | Use Case |
|-----------|----------|
| 300 (5 min) | Frequently changed records, pre-migration |
| 3600 (1 hour) | Standard for most records |
| 86400 (1 day) | Stable records (NS, MX) |

**Migration strategy:**
1. **Days before migration:** Lower TTL to 60-300 seconds
2. **During migration:** Update DNS record to new IP
3. **Wait for old TTL to expire** — worst case is the old TTL duration
4. **After migration:** Raise TTL back to normal

### Q: Explain DNS-based load balancing.

**Round-Robin DNS:** Multiple A records for the same hostname.
```
example.com  A  10.0.0.1
example.com  A  10.0.0.2
example.com  A  10.0.0.3
```
DNS rotates the order in responses. **Limitations:** no health checks, no session persistence, TTL caching skews distribution.

**Better alternatives:** Route 53 weighted/latency/geolocation routing, or use proper load balancers (ALB/NLB).

---

## HTTP/HTTPS & TLS

### Q: Explain TLS handshake (TLS 1.3).

```
Client                                Server
  │                                      │
  │── ClientHello ─────────────────────►│
  │   (TLS version, cipher suites,       │
  │    random, key_share)                │
  │                                      │
  │◄── ServerHello ─────────────────────│
  │    (chosen cipher, key_share)        │
  │◄── EncryptedExtensions ─────────────│
  │◄── Certificate ─────────────────────│
  │◄── CertificateVerify ──────────────│
  │◄── Finished ────────────────────────│
  │                                      │
  │── Finished ─────────────────────────►│
  │                                      │
  │◄═══════ Encrypted Data ══════════►│
```

**TLS 1.3 improvements over 1.2:**
- **1-RTT** handshake (vs 2-RTT in 1.2)
- **0-RTT** resumption for repeat connections
- Removed insecure ciphers (RC4, 3DES, SHA-1)
- Mandatory **Forward Secrecy** (ECDHE)

### Q: HTTP/1.1 vs HTTP/2 vs HTTP/3?

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Multiplexing | No (one req per conn) | Yes (streams over 1 TCP) | Yes (streams over QUIC/UDP) |
| Head-of-line blocking | Yes | TCP level only | None (QUIC handles per-stream) |
| Header compression | None | HPACK | QPACK |
| Server push | No | Yes | Yes |
| Protocol | TCP | TCP | UDP (QUIC) |
| Connection setup | TCP + TLS (3 RTT) | TCP + TLS (2-3 RTT) | 1 RTT (0-RTT resume) |

### Q: What are HTTP status codes to know?

| Code | Meaning | When |
|------|---------|------|
| **200** | OK | Successful request |
| **201** | Created | Resource created (POST) |
| **204** | No Content | Success, empty body (DELETE) |
| **301** | Moved Permanently | Permanent redirect (SEO) |
| **302** | Found | Temporary redirect |
| **304** | Not Modified | Cached version is still valid |
| **400** | Bad Request | Malformed request |
| **401** | Unauthorized | Authentication required |
| **403** | Forbidden | Authenticated but not authorized |
| **404** | Not Found | Resource doesn't exist |
| **429** | Too Many Requests | Rate limited |
| **500** | Internal Server Error | Server bug |
| **502** | Bad Gateway | Upstream server error (common with reverse proxies) |
| **503** | Service Unavailable | Server overloaded or in maintenance |
| **504** | Gateway Timeout | Upstream timeout |

---

## Load Balancing

### Q: Explain load balancing algorithms.

| Algorithm | How it Works | Best For |
|-----------|-------------|----------|
| **Round Robin** | Sequential distribution | Equal-capacity servers |
| **Weighted Round Robin** | Proportional to weight | Mixed-capacity servers |
| **Least Connections** | Sends to server with fewest active connections | Variable request duration |
| **Weighted Least Connections** | Least conn + weights | Mixed capacity + variable requests |
| **IP Hash** | Hash client IP → consistent server | Session persistence without cookies |
| **Consistent Hashing** | Minimal redistribution on server change | Caching layers, distributed systems |
| **Random** | Random selection | Simple, surprisingly effective at scale |
| **Least Response Time** | Fastest responding server | Latency-sensitive apps |

### Q: L4 vs L7 Load Balancing?

| Feature | L4 (Transport) | L7 (Application) |
|---------|----------------|-------------------|
| Operates on | TCP/UDP packets | HTTP requests |
| Decision based on | IP, port | URL, headers, cookies, body |
| SSL termination | Pass-through or terminate | Terminates SSL |
| Performance | Faster (less inspection) | Slower (full request parsing) |
| Content routing | No | Yes (path-based, host-based) |
| AWS | NLB | ALB |
| Examples | HAProxy TCP, IPVS | Nginx, HAProxy HTTP, Envoy |

**L7 routing example (Nginx):**
```nginx
upstream api_servers {
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
}

upstream web_servers {
    server 10.0.2.10:80;
    server 10.0.2.11:80;
}

server {
    listen 443 ssl;

    location /api/ {
        proxy_pass http://api_servers;
    }

    location / {
        proxy_pass http://web_servers;
    }
}
```

### Q: What is health checking in load balancers?

**Active Health Checks:** LB periodically sends requests to backends.
```
Types:
- TCP check: Can I connect to port?
- HTTP check: Does GET /health return 200?
- Custom script: Application-specific logic

Parameters:
- Interval: How often to check (e.g., 10s)
- Timeout: How long to wait for response (e.g., 5s)
- Healthy threshold: Consecutive successes to mark healthy (e.g., 3)
- Unhealthy threshold: Consecutive failures to mark unhealthy (e.g., 2)
```

**Passive Health Checks:** LB monitors real traffic for errors/timeouts and removes failing backends.

---

## Proxies — Forward & Reverse

### Q: Forward Proxy vs Reverse Proxy?

```
Forward Proxy:                          Reverse Proxy:
Client → [Forward Proxy] → Server      Client → [Reverse Proxy] → Server(s)
Client knows it's using a proxy         Client doesn't know about backend servers
```

| Feature | Forward Proxy | Reverse Proxy |
|---------|--------------|---------------|
| Sits in front of | Clients | Servers |
| Purpose | Privacy, filtering, caching | Load balancing, SSL, caching, security |
| Client config | Required | Not required |
| Examples | Squid, corporate proxies | Nginx, HAProxy, Envoy, Traefik |
| Hides | Client identity | Server identity |

### Q: Explain Nginx as a reverse proxy.

```nginx
# /etc/nginx/conf.d/app.conf

upstream backend {
    least_conn;
    server 10.0.1.10:8080 weight=3;
    server 10.0.1.11:8080 weight=1;
    server 10.0.1.12:8080 backup;    # Only used when others are down

    keepalive 32;                      # Connection pooling to upstream
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    ssl_certificate     /etc/ssl/certs/app.crt;
    ssl_certificate_key /etc/ssl/private/app.key;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Strict-Transport-Security "max-age=31536000" always;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }

    # Static file caching
    location /static/ {
        root /var/www;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## CDN (Content Delivery Network)

### Q: How does a CDN work?

```
User (Mumbai) ──► CDN Edge (Mumbai POP) ──► Origin Server (US-East)
                       │
                  Cache HIT? ──► Return cached content (fast!)
                       │
                  Cache MISS? ──► Fetch from origin, cache it, return
```

**Key concepts:**
- **POP (Point of Presence):** Edge location close to users
- **Cache HIT ratio:** % of requests served from cache (higher = better)
- **TTL:** How long content is cached at edge
- **Cache invalidation:** Purge stale content (`Cache-Control`, `Surrogate-Key`)
- **Origin Shield:** Extra caching layer between edge and origin to reduce origin load

**Cache-Control headers:**
```
Cache-Control: public, max-age=86400          # Cache for 1 day
Cache-Control: private, no-cache              # Don't cache (user-specific)
Cache-Control: no-store                       # Never cache (sensitive data)
Cache-Control: s-maxage=3600                  # CDN-specific TTL
Cache-Control: stale-while-revalidate=60      # Serve stale while fetching fresh
```

---

## Subnetting & CIDR

### Q: Explain CIDR notation and subnetting.

**CIDR (Classless Inter-Domain Routing):** `10.0.0.0/24`

The `/24` means the first 24 bits are the network portion.

```
IP:      10.0.0.0
Binary:  00001010.00000000.00000000.00000000
Mask:    11111111.11111111.11111111.00000000  = /24 = 255.255.255.0
         ├────── Network (24 bits) ──────┤├ Host ┤
```

**Quick reference:**

| CIDR | Subnet Mask | # Hosts | Use Case |
|------|-------------|---------|----------|
| /32 | 255.255.255.255 | 1 | Single host |
| /28 | 255.255.255.240 | 14 | Small subnet |
| /24 | 255.255.255.0 | 254 | Standard subnet |
| /20 | 255.255.240.0 | 4094 | Large subnet |
| /16 | 255.255.0.0 | 65534 | VPC default |
| /8 | 255.0.0.0 | 16M+ | Class A |

**Private IP ranges (RFC 1918):**
```
10.0.0.0/8       (10.0.0.0 – 10.255.255.255)      Class A
172.16.0.0/12    (172.16.0.0 – 172.31.255.255)     Class B
192.168.0.0/16   (192.168.0.0 – 192.168.255.255)   Class C
```

### Q: How to calculate subnets?

**Example: Divide 10.0.0.0/16 into /24 subnets:**

```
10.0.0.0/16 has 16 host bits → 65534 hosts
10.0.0.0/24 has 8 host bits → 254 hosts

Number of /24 subnets = 2^(24-16) = 256 subnets

10.0.0.0/24    (10.0.0.1 – 10.0.0.254)
10.0.1.0/24    (10.0.1.1 – 10.0.1.254)
10.0.2.0/24    (10.0.2.1 – 10.0.2.254)
...
10.0.255.0/24  (10.0.255.1 – 10.0.255.254)
```

**Formula:**
- Hosts per subnet = 2^(32 - prefix) - 2 (subtract network and broadcast)
- Number of subnets = 2^(new_prefix - old_prefix)

---

## VPN & Tunneling

### Q: Types of VPN and tunneling protocols.

| Type | Protocol | Layer | Use Case |
|------|----------|-------|----------|
| **IPSec** | ESP/AH | L3 | Site-to-site VPN (AWS VPN) |
| **OpenVPN** | Custom over UDP/TCP | L3/L4 | Remote access VPN |
| **WireGuard** | Custom over UDP | L3 | Modern, fast, simple VPN |
| **SSH Tunnel** | SSH | L4 | Quick port forwarding |
| **GRE** | GRE | L3 | Encapsulating diverse protocols |
| **VXLAN** | UDP 4789 | L2 over L3 | Overlay networks (Kubernetes, cloud) |

**SSH Tunneling (very commonly asked):**
```bash
# Local port forwarding: Access remote service through local port
ssh -L 8080:remote-db:5432 bastion-host
# Now localhost:8080 → remote-db:5432 via bastion

# Remote port forwarding: Expose local service on remote
ssh -R 9090:localhost:3000 remote-server
# Now remote-server:9090 → your-machine:3000

# Dynamic SOCKS proxy
ssh -D 1080 bastion-host
# Configure apps to use SOCKS5 proxy localhost:1080

# Jump host / ProxyJump
ssh -J bastion-host private-server
# Or in ~/.ssh/config:
# Host private-server
#   ProxyJump bastion-host
```

---

## Firewalls & Network Security

### Q: What is a WAF (Web Application Firewall)?

A **WAF** operates at Layer 7 and inspects HTTP traffic for attacks:

- **SQL Injection:** `' OR 1=1 --`
- **XSS (Cross-Site Scripting):** `<script>alert('xss')</script>`
- **CSRF:** Forged requests from authenticated sessions
- **DDoS (L7):** HTTP floods, slowloris attacks
- **Bot detection:** Rate limiting, CAPTCHA

**Examples:** AWS WAF, Cloudflare WAF, ModSecurity

**Difference from network firewall:**

| Feature | Network Firewall (L3/L4) | WAF (L7) |
|---------|-------------------------|----------|
| Inspects | IP, ports, protocols | HTTP requests/responses |
| Blocks | IP ranges, port scans | SQL injection, XSS |
| Examples | iptables, Security Groups | AWS WAF, Cloudflare |

### Q: Explain the concept of Zero Trust Networking.

**Traditional:** Castle-and-moat. Trust everything inside the network perimeter.

**Zero Trust:** "Never trust, always verify." Every request is authenticated and authorized regardless of network location.

**Principles:**
1. **Verify explicitly** — Authenticate every request (mTLS, tokens)
2. **Least privilege** — Minimum required access
3. **Assume breach** — Segment networks, encrypt everything, monitor continuously

**Implementation:**
- **Identity-based access** (not IP-based)
- **mTLS** between all services
- **Service mesh** (Istio, Linkerd) for policy enforcement
- **BeyondCorp** (Google's zero trust model) — No VPN needed, every request goes through an access proxy

---

## TCP vs UDP Deep Dive

### Q: Explain TCP flow control and congestion control.

**Flow Control (receiver-side):**
- **Sliding Window** — Receiver advertises window size (how much data it can buffer)
- Sender cannot send more than the window allows
- Window size = 0 → sender pauses (zero window probe)

**Congestion Control (network-side):**

```
                    ┌───────────────────────────────────────────┐
Congestion Window   │     /\                                    │
(cwnd)              │    /  \         ___________               │
                    │   /    \       /           \              │
                    │  /      \     /             \             │
                    │ / Slow   \   / Congestion    \            │
                    │/ Start    \ /  Avoidance      \           │
                    ├───────────×──────────────────────────────►│
                    0     ssthresh                    Time
```

| Phase | Behavior |
|-------|----------|
| **Slow Start** | cwnd doubles every RTT (exponential growth) |
| **Congestion Avoidance** | cwnd increases by 1 per RTT (linear growth) |
| **Fast Retransmit** | Retransmit after 3 duplicate ACKs (no timeout wait) |
| **Fast Recovery** | Halve cwnd, enter congestion avoidance |

**Modern algorithms:** CUBIC (Linux default), BBR (Google — bandwidth-based, not loss-based).

### Q: What is TCP TIME_WAIT and why does it matter?

After closing a TCP connection, the socket enters **TIME_WAIT** for 2×MSL (usually 60s).

**Why it exists:**
1. Ensures late-arriving packets from old connection don't corrupt a new connection on the same port
2. Guarantees the final ACK is delivered to the remote side

**Problem:** High-traffic servers can exhaust ports due to TIME_WAIT accumulation.

```bash
# Check TIME_WAIT sockets
ss -tan state time-wait | wc -l

# Tuning (careful, understand implications):
sysctl net.ipv4.tcp_tw_reuse=1          # Reuse TIME_WAIT sockets for new connections
sysctl net.ipv4.tcp_fin_timeout=15       # Reduce FIN timeout
sysctl net.ipv4.ip_local_port_range="1024 65535"  # More ephemeral ports
```

---

## NAT & Port Forwarding

### Q: Explain NAT types.

| Type | Description | Example |
|------|-------------|---------|
| **SNAT** (Source NAT) | Changes source IP | Private IP → Public IP for internet access |
| **DNAT** (Destination NAT) | Changes destination IP | Public IP:80 → Private server:8080 |
| **PAT** (Port Address Translation) | Many-to-one NAT using ports | Most home routers |
| **Masquerade** | SNAT variant for dynamic IPs | `iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE` |

**NAT in AWS:**
- **NAT Gateway** — Managed SNAT for private subnets to access internet
- **Internet Gateway** — 1:1 NAT for public IPs (Elastic IPs)

---

## Network Troubleshooting

### Q: Essential network troubleshooting commands.

```bash
# 1. Connectivity
ping -c 4 <host>                # ICMP echo (is host reachable?)
traceroute <host>               # Path to destination (UDP by default)
mtr <host>                      # Continuous traceroute (best for diagnosing)

# 2. DNS
dig <domain>                    # Detailed DNS lookup
dig +short <domain>             # Just the IP
dig @8.8.8.8 <domain>          # Query specific DNS server
dig +trace <domain>             # Full resolution path
nslookup <domain>               # Simple lookup

# 3. Port / Service
ss -tunapl                      # All listening sockets
nc -zv <host> <port>            # Test TCP connectivity to port
nmap -p 1-1000 <host>           # Port scan
telnet <host> <port>            # Quick port test

# 4. HTTP
curl -vvv https://example.com   # Verbose HTTP request
curl -o /dev/null -s -w "HTTP Code: %{http_code}\nTime Total: %{time_total}\n" https://example.com
wget --spider https://example.com  # Check if URL exists

# 5. Packet Capture
tcpdump -i eth0 port 80         # Capture HTTP traffic
tcpdump -i any host 10.0.0.5   # All traffic to/from host
tcpdump -i eth0 -w capture.pcap # Save to file (open in Wireshark)
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'  # SYN packets only

# 6. Bandwidth
iperf3 -s                       # Start server
iperf3 -c <server>              # Client test bandwidth

# 7. ARP / Neighbors
arp -a                          # ARP table
ip neigh show                   # Neighbor table
```

### Q: How to troubleshoot "connection refused" vs "connection timed out"?

| Error | Meaning | Cause |
|-------|---------|-------|
| **Connection refused** | Got RST packet back | Port not listening, firewall rejecting |
| **Connection timed out** | No response at all | Firewall dropping (not rejecting), host down, routing issue |

```bash
# Connection refused — service is not listening
ss -tlnp | grep <port>          # Is anything listening?
systemctl status <service>       # Is service running?

# Connection timed out — packets being dropped
iptables -L -n                  # Local firewall?
# Check cloud security groups, NACLs
traceroute -T -p <port> <host>  # Where are packets dying?
```

---

## Cloud Networking Concepts

### Q: Explain VPC architecture.

```
┌──────────────────── VPC (10.0.0.0/16) ────────────────────┐
│                                                             │
│  ┌─── AZ-a ──────────────┐  ┌─── AZ-b ──────────────┐    │
│  │                        │  │                        │    │
│  │  ┌── Public Subnet ──┐│  │  ┌── Public Subnet ──┐│    │
│  │  │  10.0.1.0/24      ││  │  │  10.0.3.0/24      ││    │
│  │  │  NAT GW, ALB      ││  │  │  NAT GW, ALB      ││    │
│  │  └────────────────────┘│  │  └────────────────────┘│    │
│  │                        │  │                        │    │
│  │  ┌── Private Subnet ─┐│  │  ┌── Private Subnet ─┐│    │
│  │  │  10.0.2.0/24      ││  │  │  10.0.4.0/24      ││    │
│  │  │  App servers       ││  │  │  App servers       ││    │
│  │  └────────────────────┘│  │  └────────────────────┘│    │
│  │                        │  │                        │    │
│  │  ┌── DB Subnet ──────┐│  │  ┌── DB Subnet ──────┐│    │
│  │  │  10.0.10.0/24     ││  │  │  10.0.11.0/24     ││    │
│  │  │  RDS, ElastiCache  ││  │  │  RDS replica      ││    │
│  │  └────────────────────┘│  │  └────────────────────┘│    │
│  └────────────────────────┘  └────────────────────────┘    │
│                                                             │
│  Internet Gateway ←→ Route Tables ←→ Security Groups/NACLs │
└─────────────────────────────────────────────────────────────┘
```

### Q: Security Groups vs NACLs?

| Feature | Security Groups | NACLs |
|---------|----------------|-------|
| Level | Instance (ENI) | Subnet |
| State | **Stateful** (return traffic auto-allowed) | **Stateless** (must allow both directions) |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated | Rules evaluated in number order |
| Default | Deny all inbound, allow all outbound | Allow all |

---

## Scenario-Based Questions

### Q: Your application is experiencing intermittent 502 errors. Troubleshoot.

```
502 Bad Gateway = The reverse proxy/LB received an invalid response from upstream

Checklist:
1. Check upstream health
   - Are backend servers healthy?
   - Check LB target group health checks

2. Check upstream logs
   - Application error logs
   - OOM kills? dmesg | grep -i oom

3. Timeout misconfig
   - LB timeout < application processing time?
   - Increase proxy_read_timeout in Nginx

4. Connection limits
   - Upstream max connections exhausted?
   - Check: ss -tn state established | wc -l

5. Upstream crashes
   - Process restarting? systemctl status <app>
   - Memory issues? Check with free -h

6. Keepalive mismatch
   - LB keepalive timeout > upstream keepalive timeout?
   - Race condition: LB sends request on a connection the upstream just closed
```

### Q: Latency increased for users in Europe but not in US. What happened?

```
Possible causes:
1. CDN cache eviction / miss for European POPs
   - Check CDN cache HIT ratio by region

2. DNS routing change
   - GeoDNS misconfigured? dig from European resolver
   - BGP route change? Check BGP looking glass tools

3. Submarine cable issue
   - Check internet health dashboards (Cloudflare Radar, ThousandEyes)

4. Regional infrastructure
   - Did an EU region go down? Check cloud provider status
   - Is traffic being routed to US instead of EU?

5. Certificate/TLS issue
   - OCSP stapling failure in EU? (extra TLS roundtrip)
```

---

## Key Resources

- **Computer Networking: A Top-Down Approach** — Kurose & Ross (THE networking textbook)
- **High Performance Browser Networking** — Ilya Grigorik (free online: hpbn.co)
- **Beej's Guide to Network Programming** — Classic sockets programming guide
- **Cloudflare Learning Center** — https://www.cloudflare.com/learning/
- **Julia Evans' Networking Zines** — Bite-sized networking concepts
- **RFC 2616 (HTTP/1.1), RFC 7540 (HTTP/2), RFC 9000 (QUIC)** — The source of truth
- **Brendan Gregg's Network Tools** — https://www.brendangregg.com/linuxperf.html
