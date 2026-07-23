---
title: Networking
nav_order: 11
description: "Networking — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Networking — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

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
- [Cloud Networking (AWS/GCP/Azure)](#cloud-networking-awsgcpazure)
- [Container & Kubernetes Networking](#container--kubernetes-networking)
- [Network Troubleshooting — Systematic Debug](#network-troubleshooting--systematic-debug)
- [Trending Topics (2025-2026)](#trending-topics-2025-2026)
- [Scenario-Based Questions](#scenario-based-questions)

---

## OSI Model & TCP/IP

### ⭐ Q: Explain the OSI model and map it to real-world protocols.

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

### 🔥 Q: What happens when you type `google.com` in a browser?

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

### 🔥 Q: Explain DNS record types.

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

### ⭐ Q: What is the difference between authoritative and recursive DNS?

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

### 🔥 Q: What is DNS TTL and how does it affect deployments?

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

### 💡 Q: Explain DNS-based load balancing.

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

### 🔥 Q: Explain TLS handshake (TLS 1.3).

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

### ⭐ Q: Explain certificate chains and verification.

```
Certificate Chain Validation:

┌─────────────────────────────────────┐
│    Root CA Certificate (trusted)    │  ← Installed in OS/browser trust store
│    CN: DigiCert Global Root CA      │
└─────────────────────────────────────┘
              │
              │ Signs ▼
┌─────────────────────────────────────┐
│   Intermediate CA Certificate       │
│   CN: DigiCert TLS RSA SHA256 2020  │
└─────────────────────────────────────┘
              │
              │ Signs ▼
┌─────────────────────────────────────┐
│      End-Entity Certificate         │  ← Server sends this chain
│      CN: example.com                │
│      SAN: example.com, *.example.com│
└─────────────────────────────────────┘
```

**Verification steps:**
1. Check signature of example.com cert using Intermediate CA public key
2. Check signature of Intermediate cert using Root CA public key
3. Verify Root CA is in trust store
4. Check certificate validity dates (notBefore / notAfter)
5. Verify hostname matches CN or SAN (Subject Alternative Name)
6. Check revocation status (OCSP / CRL)

**Common errors:**
```bash
# Self-signed certificate
curl: (60) SSL certificate problem: self signed certificate

# Expired certificate
curl: (60) SSL certificate problem: certificate has expired

# Hostname mismatch
curl: (60) SSL: no alternative certificate subject name matches target host name

# Untrusted intermediate
curl: (60) SSL certificate problem: unable to get local issuer certificate
# Fix: Server must send full chain (leaf + intermediate), not just leaf
```

### ⭐ Q: What is mTLS (Mutual TLS)?

**Standard TLS:** Only server proves identity to client.

**mTLS:** Both client and server authenticate each other using certificates.

```
Client                               Server
  │                                     │
  │── ClientHello ────────────────────►│
  │◄── ServerHello + Certificate ─────│
  │◄── CertificateRequest ────────────│  ← Server asks for client cert
  │                                     │
  │── Certificate ────────────────────►│  ← Client sends its cert
  │── CertificateVerify ──────────────►│  ← Client proves ownership
  │── Finished ───────────────────────►│
  │◄── Finished ──────────────────────│
  │◄══════ Encrypted + Authenticated ═►│
```

**Use cases:**
- **Zero-trust service mesh** (Istio, Linkerd) — every pod has a cert
- **API authentication** — client cert = identity, no API keys
- **IoT devices** — device certs for authentication
- **Internal microservices** — replace bearer tokens with certs

**Certificate management:**
- **Manual:** openssl, cert-manager (Kubernetes)
- **Automated:** Vault PKI, SPIFFE/SPIRE, AWS Private CA
- **Rotation:** Short-lived certs (hours/days) auto-rotated

### 🔥 Q: HTTP/1.1 vs HTTP/2 vs HTTP/3?

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Multiplexing | No (one req per conn) | Yes (streams over 1 TCP) | Yes (streams over QUIC/UDP) |
| Head-of-line blocking | Yes | TCP level only | None (QUIC handles per-stream) |
| Header compression | None | HPACK | QPACK |
| Server push | No | Yes | Yes |
| Protocol | TCP | TCP | UDP (QUIC) |
| Connection setup | TCP + TLS (3 RTT) | TCP + TLS (2-3 RTT) | 1 RTT (0-RTT resume) |

### 💡 Q: Deep dive into HTTP/3 and QUIC.

**Why HTTP/3?**
- HTTP/2 still suffers from **TCP head-of-line blocking** — one lost packet blocks all streams
- TCP handshake + TLS handshake = latency
- TCP state tied to 4-tuple (src IP, src port, dst IP, dst port) — breaks on network change (mobile)

**QUIC (Quick UDP Internet Connections):**

```
HTTP/3 Stack:
┌────────────────────┐
│      HTTP/3        │  ← Application protocol
├────────────────────┤
│       QUIC         │  ← Transport + TLS 1.3 built-in
├────────────────────┤
│        UDP         │  ← Unreliable datagram (but QUIC adds reliability)
├────────────────────┤
│        IP          │
└────────────────────┘
```

**Key features:**
- **Built-in TLS 1.3** — encryption is mandatory, reduces handshake to 1-RTT (0-RTT for resumption)
- **Connection ID** — survives IP/port changes (Wi-Fi → cellular seamless)
- **Independent streams** — lost packet only blocks affected stream, not all
- **Congestion control in userspace** — easier to update than TCP (kernel)
- **Forward Error Correction** (optional) — recover lost packets without retransmit

**Adoption (2025):**
- **Google:** 100% of traffic
- **Cloudflare, Fastly:** Enabled by default
- **AWS CloudFront:** HTTP/3 available
- **Browsers:** Chrome, Firefox, Safari, Edge all support
- **Servers:** Nginx (1.25+), Caddy, LiteSpeed

**How to check:**
```bash
# Check if site supports HTTP/3
curl -I --http3 https://cloudflare.com

# Check protocol in browser DevTools → Network → Protocol column
# Look for "h3" (HTTP/3) vs "h2" (HTTP/2)
```

### 🔥 Q: What are HTTP status codes to know?

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

### 🔥 Q: Explain load balancing algorithms.

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

### 🔥 Q: L4 vs L7 Load Balancing?

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

### ⭐ Q: What is health checking in load balancers?

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

### 🔥 Q: Forward Proxy vs Reverse Proxy?

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

### ⭐ Q: Explain Nginx as a reverse proxy.

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

### ⭐ Q: How does a CDN work?

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

### 🔥 Q: Explain CIDR notation and subnetting.

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

### ⭐ Q: How to calculate subnets?

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

### ⭐ Q: Types of VPN and tunneling protocols.

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

### ⭐ Q: What is a WAF (Web Application Firewall)?

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

### ⭐ Q: Explain the concept of Zero Trust Networking.

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

### 💡 Q: Explain TCP flow control and congestion control.

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

### 🔥 Q: Explain TCP handshake and teardown in detail.

**3-way handshake (connection establishment):**

```
Client                           Server
  │                                 │
  │── SYN (seq=100) ───────────────►│  Client initiates, sends random seq #
  │                                 │
  │◄── SYN+ACK (seq=300, ack=101) ─│  Server responds, its own seq + ack
  │                                 │
  │── ACK (ack=301) ───────────────►│  Client acknowledges server's seq
  │                                 │
  │◄══════ Connection ready ═══════►│
```

**4-way teardown (connection close):**

```
Client                           Server
  │                                 │
  │── FIN (seq=500) ───────────────►│  Client done sending
  │                                 │
  │◄── ACK (ack=501) ──────────────│  Server acknowledges
  │                                 │
  │◄── FIN (seq=800) ──────────────│  Server done sending
  │                                 │
  │── ACK (ack=801) ───────────────►│  Client acknowledges
  │                                 │
  │  [TIME_WAIT state: 2×MSL]       │
```

**TCP flags you must know:**

| Flag | Purpose |
|------|---------|
| **SYN** | Synchronize sequence numbers (connection start) |
| **ACK** | Acknowledge received data |
| **FIN** | Finish sending data (graceful close) |
| **RST** | Reset connection (abrupt close) |
| **PSH** | Push data immediately to application |

### 💡 Q: What is TCP TIME_WAIT and why does it matter?

After closing a TCP connection, the socket enters **TIME_WAIT** for 2×MSL (Maximum Segment Lifetime, usually 60s).

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

### 🔥 Q: TCP vs UDP — when to use each?

| Feature | TCP | UDP |
|---------|-----|-----|
| Reliability | Guaranteed delivery, ordered | Best-effort, no retransmit |
| Connection | Connection-oriented (handshake) | Connectionless |
| Overhead | Higher (headers, ACKs, retransmit) | Lower |
| Speed | Slower due to guarantees | Faster |
| Use Cases | HTTP, SSH, file transfer, databases | DNS, video streaming, VoIP, gaming, monitoring (StatsD) |

**When UDP makes sense:**
- **Real-time media** — dropped frame better than delayed frame
- **DNS** — single query/response, retransmit at app layer if needed
- **Monitoring metrics** — occasional loss acceptable, low latency critical
- **Gaming** — position updates, old data is useless

**Modern hybrid:**
- **QUIC** — reliability + congestion control over UDP (HTTP/3)

---

## NAT & Port Forwarding

### ⭐ Q: Explain NAT types.

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

## Container & Kubernetes Networking

### ⭐ Q: How does container networking work?

**Docker default bridge mode:**

```
Host
├── docker0 bridge (172.17.0.1/16)
│   ├── veth1a2b3c ←→ Container A eth0 (172.17.0.2)
│   └── veth4d5e6f ←→ Container B eth0 (172.17.0.3)
│
├── eth0 (public IP)
└── iptables NAT rules (SNAT for outbound)
```

**Modes:**

| Mode | Description | Use Case |
|------|-------------|----------|
| **Bridge** | Virtual bridge, internal IPs | Default, isolated containers |
| **Host** | Share host network stack | High performance, no isolation |
| **None** | No networking | Custom network setup |
| **Overlay** (Swarm/K8s) | Multi-host networking | Distributed containers |

### 🔥 Q: Explain Kubernetes networking model.

**4 requirements:**
1. **Pod-to-Pod** — All pods can communicate without NAT
2. **Pod-to-Service** — ClusterIP load balances to pods
3. **External-to-Service** — NodePort, LoadBalancer, Ingress
4. **Unique IP per pod** — No port conflicts

```
Kubernetes Cluster
┌───────────────────────────────────────────────────────┐
│                                                       │
│  Node 1 (10.0.1.10)              Node 2 (10.0.1.11)  │
│  ┌─────────────────┐            ┌─────────────────┐  │
│  │ Pod A           │            │ Pod B           │  │
│  │ 10.244.1.5:8080 │◄───────────►│ 10.244.2.3:8080│  │
│  └─────────────────┘            └─────────────────┘  │
│         │                                │            │
│         └──────────► Service ClusterIP ◄─┘           │
│                    (10.96.0.10:80)                    │
│                           │                           │
└───────────────────────────┼───────────────────────────┘
                            │
                            ▼
                    LoadBalancer / Ingress
                  (public-ip.cloud.com)
```

### ⭐ Q: What is a CNI (Container Network Interface)?

**CNI** is a plugin standard for container networking. Kubernetes uses CNI to set up pod networking.

**Popular CNI plugins:**

| CNI | Overlay | Network Policy | Performance | Use Case |
|-----|---------|----------------|-------------|----------|
| **Calico** | VXLAN/IPIP optional | Yes (native) | High | Enterprise, network policies |
| **Cilium** | VXLAN/Geneve | Yes (eBPF) | Very High | Modern, eBPF-based |
| **Flannel** | VXLAN | No | Medium | Simple, easy setup |
| **Weave** | VXLAN | Yes | Medium | Mesh networking |
| **AWS VPC CNI** | Native AWS ENI | Yes (SG) | High | EKS, native AWS integration |

**How CNI works:**
1. Kubelet calls CNI plugin when pod is created
2. CNI plugin allocates IP from pool
3. Creates veth pair: one in pod netns, one on host
4. Configures routes and iptables rules
5. Returns IP to kubelet

### 🔥 Q: Explain Kubernetes Services (ClusterIP, NodePort, LoadBalancer).

| Type | Scope | IP | Use Case |
|------|-------|----|---------| 
| **ClusterIP** | Internal only | Virtual IP in cluster | Default, internal services |
| **NodePort** | External via node IP | ClusterIP + port on every node | Dev/test, no LB |
| **LoadBalancer** | External via cloud LB | ClusterIP + cloud LB | Production external access |
| **ExternalName** | DNS CNAME | No IP | Alias to external service |

**Example:**

```yaml
# ClusterIP (default)
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
# Accessible at backend.default.svc.cluster.local:80
# kube-proxy creates iptables rules to load balance to pods

# NodePort (ClusterIP + port on every node)
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: NodePort
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # 30000-32767 range
# Accessible at <any-node-ip>:30080

# LoadBalancer (NodePort + cloud LB)
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
# Cloud controller provisions ELB/ALB/NLB
# Accessible at <cloud-lb-external-ip>:80
```

### ⭐ Q: What is a Service Mesh?

A **service mesh** is an infrastructure layer for service-to-service communication with observability, security, and traffic control.

```
Without Service Mesh:          With Service Mesh:

App A ──► App B                App A ──► [Sidecar] ──► [Sidecar] ──► App B
                                          │                │
                                          └─► Control Plane ◄┘
                                          (mTLS, metrics, routing)
```

**Popular meshes:**

| Mesh | Architecture | Strengths |
|------|-------------|-----------|
| **Istio** | Sidecar (Envoy) | Feature-rich, mature |
| **Linkerd** | Sidecar (Linkerd2-proxy) | Lightweight, simple |
| **Cilium Service Mesh** | Sidecarless (eBPF) | High performance, no sidecar overhead |
| **Consul** | Sidecar (Envoy) | Multi-platform (not just K8s) |

**Features:**
- **mTLS** — Automatic cert provisioning and rotation
- **Traffic management** — Retries, timeouts, circuit breaking, canary
- **Observability** — Distributed tracing, metrics (latency, error rate)
- **Policy** — Authorization, rate limiting

**Sidecar vs Sidecarless:**

| Feature | Sidecar (Istio/Linkerd) | Sidecarless (Cilium) |
|---------|------------------------|----------------------|
| Proxy overhead | Higher (2× memory, CPU per pod) | Lower (eBPF kernel hooks) |
| Maturity | Proven, stable | Newer, evolving |
| Flexibility | Very high (Envoy config) | Growing |
| Complexity | Higher (more containers) | Lower (less infra) |

### 💡 Q: What is eBPF and why does it matter for networking?

**eBPF (extended Berkeley Packet Filter):** Run sandboxed programs in the Linux kernel without changing kernel code.

**Networking use cases:**
- **Packet filtering** — Fast firewalls, DDoS mitigation
- **Observability** — tcpdump, network tracing without overhead
- **Load balancing** — Kernel-level LB (Cilium, Katran)
- **Service mesh** — Sidecarless mesh (no proxy container)
- **Security** — Runtime security (Falco, Tetragon)

**Why it's trending (2025):**
- **Performance** — Kernel-level processing, no context switches
- **Flexibility** — Update network logic without kernel recompile
- **Cloud-native** — Cilium (eBPF CNI) is default in GKE, EKS options

**Tools:**
- **Cilium** — K8s networking + service mesh
- **Katran** — Facebook's L4 load balancer
- **bpftrace** — Dynamic tracing
- **Falco** — Runtime security

---

## Network Troubleshooting — Systematic Debug

### 🔥 Q: Essential network troubleshooting commands.

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

### 🔥 Q: How to troubleshoot "connection refused" vs "connection timed out" vs "connection reset"?

| Error | TCP Behavior | Meaning | Cause |
|-------|--------------|---------|-------|
| **Connection refused** | RST received | Port not listening | Service not running, wrong port |
| **Connection timed out** | No response | Packets dropped/lost | Firewall drop, routing issue, host down |
| **Connection reset** | RST mid-connection | Connection killed | App crash, firewall RST, proxy timeout |

```bash
# Connection refused — service is not listening
ss -tlnp | grep <port>          # Is anything listening?
systemctl status <service>       # Is service running?
lsof -i :<port>                  # What's using the port?

# Connection timed out — packets being dropped
iptables -L -n -v                # Local firewall? (check packet counters)
# Check cloud security groups, NACLs
traceroute -T -p <port> <host>  # Where are packets dying?
mtr -T -P <port> <host>          # Continuous traceroute with loss stats

# Connection reset — mid-connection kill
# Check server logs, look for:
# - App crashes (OOM, segfault)
# - Proxy/LB timeouts (Nginx proxy_read_timeout)
# - Firewall connection tracking table overflow
```

### 🔥 Q: Systematic debug: Service A can't reach Service B.

**Step-by-step checklist:**

```bash
# 1. Basic connectivity (Layer 3)
ping service-b.example.com
# Success? Network path OK. Failure? DNS or routing issue.

# 2. DNS resolution (Layer 7)
dig service-b.example.com
nslookup service-b.example.com
# Returns correct IP? DNS OK. NXDOMAIN? DNS record missing.

# 3. Port reachability (Layer 4)
nc -zv service-b.example.com 443
telnet service-b.example.com 443
# Connection succeeded? Port open. Timeout/refused? See table above.

# 4. Routing path
traceroute service-b.example.com
mtr service-b.example.com
# Where does it stop? Network boundary, firewall, or host?

# 5. Firewall rules
# On Service A (outbound):
iptables -L OUTPUT -n -v
# Cloud: Check Security Group egress rules

# On Service B (inbound):
iptables -L INPUT -n -v
ss -tlnp | grep 443              # Is service listening?
# Cloud: Check Security Group ingress rules, NACLs

# 6. Service health
systemctl status <service-b>
journalctl -u <service-b> -n 50
# Service running? Check logs for errors.

# 7. Application-level (Layer 7)
curl -vvv https://service-b.example.com
# TLS handshake OK? Certificate valid? HTTP response?

# 8. Packet capture (last resort)
# On Service A:
tcpdump -i any -n host service-b-ip and port 443 -w /tmp/a.pcap

# On Service B:
tcpdump -i any -n port 443 and host service-a-ip -w /tmp/b.pcap

# Analyze in Wireshark: Do SYN packets arrive? Are SYN+ACK sent back?
```

### ⭐ Q: Troubleshoot intermittent connection issues.

**Common causes:**

| Symptom | Likely Cause | How to Confirm |
|---------|--------------|----------------|
| **Random timeouts** | Packet loss, congestion | `mtr` shows loss %, `iperf3` bandwidth test |
| **Works sometimes, not others** | Load balancer health check flapping | Check LB target health, app /health endpoint |
| **Slow during peak hours** | Bandwidth/connection exhaustion | `sar -n DEV`, `ss -s` (socket stats) |
| **DNS intermittent** | DNS timeout, TTL too low | `dig +trace`, check resolver logs |
| **TLS errors intermittent** | OCSP stapling failure, cert rotation | `openssl s_client -connect` repeatedly |

```bash
# Monitor packet loss over time
mtr --report --report-cycles 100 service-b.example.com

# Check socket stats
ss -s  # Summary: TCP sockets in various states

# Monitor connection states
watch -n 1 'ss -tan | awk "{print \$1}" | sort | uniq -c'

# DNS response time
while true; do
  time dig service-b.example.com +short
  sleep 5
done

# HTTP latency distribution
ab -n 1000 -c 10 https://service-b.example.com/
# Or use: wrk, vegeta, hey
```

### ⭐ Q: Troubleshoot DNS resolution failures.

```bash
# 1. Check /etc/resolv.conf
cat /etc/resolv.conf
# nameserver 8.8.8.8  ← Which DNS server is configured?

# 2. Query specific DNS server
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com
# Does one work and not the other? DNS server issue.

# 3. Trace full resolution path
dig +trace example.com
# Shows: root → TLD → authoritative
# Where does it fail?

# 4. Check for DNS poisoning / cache issues
dig example.com +short       # What does it return?
dig @8.8.8.8 example.com +short
# Different answers? Local cache or resolver issue.

# 5. Test DNS over TCP (not just UDP)
dig +tcp example.com
# UDP blocked by firewall?

# 6. Check systemd-resolved (common on Ubuntu)
resolvectl status
resolvectl query example.com
journalctl -u systemd-resolved -n 50

# 7. DNS packet capture
tcpdump -i any port 53 -vv
# Are queries being sent? Are responses arriving?
```

### ⭐ Q: Troubleshoot TLS handshake failures.

```bash
# 1. Detailed TLS handshake
openssl s_client -connect example.com:443 -servername example.com
# Look for:
# - Verify return code: 0 (ok) or error
# - Certificate chain
# - Cipher negotiated

# 2. Test specific TLS version
openssl s_client -connect example.com:443 -tls1_2
openssl s_client -connect example.com:443 -tls1_3
# Does TLS 1.2 work but not 1.3? Version mismatch.

# 3. Check certificate validity
openssl s_client -connect example.com:443 </dev/null 2>/dev/null | openssl x509 -noout -dates
# notBefore / notAfter

# 4. Verify certificate chain
openssl s_client -connect example.com:443 -showcerts
# Are intermediate certs included? Missing intermediate = common error.

# 5. Check SNI (Server Name Indication)
openssl s_client -connect 1.2.3.4:443 -servername example.com
# Without -servername, some servers return wrong cert.

# 6. Common errors and fixes
Error: "certificate verify failed"
  → Missing root/intermediate CA in trust store
  → Fix: Update ca-certificates package

Error: "sslv3 alert handshake failure"
  → Cipher mismatch (client and server have no common cipher)
  → Fix: Update OpenSSL, or configure server ciphers

Error: "certificate has expired"
  → Cert expired, check with: openssl x509 -noout -dates

# 7. Test with curl
curl -vvv https://example.com
# Shows full TLS handshake, certificate verification
```

### 💡 Q: Troubleshoot MTU / fragmentation issues.

**MTU (Maximum Transmission Unit):** Largest packet size (typically 1500 bytes for Ethernet).

**Problem:** If MTU is misconfigured, packets are fragmented or dropped, causing slow/broken connections.

```bash
# 1. Check MTU
ip link show eth0
# mtu 1500

# 2. Test path MTU discovery
ping -M do -s 1472 example.com  # 1472 + 28 (IP+ICMP headers) = 1500
# Success? MTU is at least 1500.
ping -M do -s 8972 example.com  # Jumbo frame test (9000 MTU)
# Failure? Path MTU < 9000.

# 3. Traceroute with MTU
traceroute --mtu example.com
# Shows MTU at each hop

# 4. Common MTU values
# 1500: Standard Ethernet
# 1492: PPPoE (DSL)
# 1450: Common in VPN tunnels
# 9000: Jumbo frames (data center)

# 5. Symptoms of MTU issues
# - Ping works, but SSH/HTTP hangs after login
# - Small HTTP requests work, large ones fail
# - VPN works locally, fails over internet

# 6. Fix by lowering MTU
sudo ip link set eth0 mtu 1400
# Or permanently in /etc/network/interfaces or cloud-init
```

### 💡 Q: Troubleshoot port exhaustion.

**Problem:** System runs out of ephemeral ports (client-side ports for outbound connections).

```bash
# 1. Check ephemeral port range
sysctl net.ipv4.ip_local_port_range
# Default: 32768-60999 (28,232 ports)

# 2. Count connections per state
ss -tan | awk '{print $1}' | sort | uniq -c
# TIME-WAIT exhaustion is common

# 3. Check for port exhaustion
ss -tan | wc -l
# If close to port range size, exhaustion likely

# 4. Fixes
# a) Increase port range
sysctl -w net.ipv4.ip_local_port_range="1024 65535"

# b) Enable TIME_WAIT reuse
sysctl -w net.ipv4.tcp_tw_reuse=1

# c) Reduce FIN timeout
sysctl -w net.ipv4.tcp_fin_timeout=15

# d) Connection pooling in application
# Reuse HTTP connections (HTTP/1.1 Keep-Alive, HTTP/2 multiplexing)

# 5. Monitoring
watch -n 1 'ss -tan | wc -l'
```

---

## Cloud Networking (AWS/GCP/Azure)

### 🔥 Q: Explain VPC architecture.

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

**Key components:**
- **VPC** — Isolated network (AWS) / VNet (Azure) / VPC (GCP)
- **Subnets** — Subdivide VPC by AZ, each has route table
- **Public subnet** — Route table has IGW route (0.0.0.0/0 → IGW)
- **Private subnet** — No direct internet access, uses NAT Gateway
- **Internet Gateway** — VPC-level, enables public IP connectivity
- **NAT Gateway** — Allows private subnets to initiate outbound internet (SNAT)
- **Route Tables** — Control routing per subnet
- **Elastic IP** — Static public IPv4

### 🔥 Q: Security Groups vs NACLs?

| Feature | Security Groups | NACLs |
|---------|----------------|-------|
| Level | Instance (ENI) | Subnet |
| State | **Stateful** (return traffic auto-allowed) | **Stateless** (must allow both directions) |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated | Rules evaluated in number order |
| Default | Deny all inbound, allow all outbound | Allow all |

**Example:**
```
Security Group (SG):
  Inbound:
    - Allow TCP 443 from 0.0.0.0/0
    - Allow TCP 22 from 10.0.0.0/16
  Outbound:
    - Allow all (default)
  # Return traffic for 443 and 22 is AUTO-ALLOWED (stateful)

NACL (Network ACL):
  Inbound:
    100: Allow TCP 443 from 0.0.0.0/0
    200: Allow TCP 22 from 10.0.0.0/16
    *: Deny all
  Outbound:
    100: Allow TCP 1024-65535 to 0.0.0.0/0  ← Must explicitly allow return traffic
    *: Deny all
```

### ⭐ Q: VPC Peering vs Transit Gateway vs PrivateLink?

| Feature | VPC Peering | Transit Gateway | PrivateLink |
|---------|-------------|-----------------|-------------|
| **Use case** | Connect 2 VPCs | Hub-and-spoke (many VPCs) | Access services privately |
| **Routing** | Direct, non-transitive | Transitive (hub routes) | Service-specific, no CIDR overlap issues |
| **Scale** | 1:1, max 125 peerings per VPC | Thousands of VPCs + on-prem | Expose service to many VPCs |
| **Cost** | Data transfer only | Per attachment + data | Per endpoint-hour + data |
| **CIDR overlap** | Not allowed | Not allowed | **Allowed** (uses ENI) |

**VPC Peering:**
```
VPC-A (10.0.0.0/16) ←──► VPC-B (10.1.0.0/16)
Non-transitive: VPC-A ⇔ VPC-B ⇔ VPC-C does NOT mean VPC-A ⇔ VPC-C
```

**Transit Gateway (hub-and-spoke):**
```
        ┌─── VPC-A
        │
  [Transit GW] ──── VPC-B     ← All VPCs can reach each other
        │
        └─── VPC-C
```

**PrivateLink (service exposure):**
```
Provider VPC                Consumer VPC
┌────────────────┐         ┌────────────────┐
│  Service (NLB) │◄────────│  VPC Endpoint  │
│  10.1.0.0/16   │         │  10.2.0.0/16   │
└────────────────┘         └────────────────┘
  │                              │
  └──► aws.com/vpce-svc-xyz ◄───┘
       (Private DNS)
```

**When to use what:**
- **Peering:** Simple 2-VPC connection
- **Transit Gateway:** Many VPCs, hybrid connectivity (VPN/Direct Connect)
- **PrivateLink:** SaaS providers exposing services, no CIDR management

### ⭐ Q: How do you connect on-prem to cloud?

| Method | Use Case | Bandwidth | Latency | Cost |
|--------|----------|-----------|---------|------|
| **VPN (Site-to-Site)** | Quick setup, low traffic | Up to 1.25 Gbps (AWS) | Internet latency | Low |
| **Direct Connect / Dedicated Interconnect** | High throughput, low latency | 1-100 Gbps | Dedicated fiber, <10ms | High |
| **SD-WAN** | Multiple sites, failover | Varies | Optimized routing | Medium |

**AWS VPN:**
```
On-prem Router (IPsec) ←→ Virtual Private Gateway ←→ VPC
```

**AWS Direct Connect:**
```
On-prem ──► DX Location (fiber) ──► AWS Region ──► VPC
            (private connection, not internet)
```

---

## Trending Topics (2025-2026)

### ⭐ Q: What is zero-trust networking?

**Traditional model:** "Castle and moat" — trust everything inside the network perimeter.

**Zero-trust:** "Never trust, always verify" — authenticate and authorize EVERY request, regardless of location.

**Core principles:**
1. **Verify explicitly** — Use all available data (identity, device, location, workload)
2. **Least privilege** — Just-in-time, just-enough access
3. **Assume breach** — Minimize blast radius, segment access, encrypt everything

**Implementation:**

| Layer | Traditional | Zero-Trust |
|-------|------------|------------|
| **Network** | VPN to corp network | No VPN, identity-based access (BeyondCorp) |
| **Service-to-service** | Trust anything in VPC | mTLS between all services |
| **Authentication** | Once at VPN | Every request (JWT, mTLS) |
| **Authorization** | IP-based ACLs | Identity + context (RBAC, ABAC) |

**Technologies:**
- **BeyondCorp (Google)** — Zero-trust access proxy
- **Service Mesh (Istio, Linkerd)** — Automatic mTLS
- **SPIFFE/SPIRE** — Identity for workloads (not just users)
- **Zero-trust network access (ZTNA)** — Cloudflare Access, Zscaler

### 💡 Q: eBPF for networking — what's the hype?

**What is eBPF?** Run custom programs in the Linux kernel without modifying kernel code.

**Why it matters for networking:**
- **Performance** — Kernel-level packet processing, no userspace context switch
- **Flexibility** — Update network logic on the fly
- **Observability** — Deep packet inspection without overhead
- **Security** — Filter/block at kernel level

**Use cases (2025):**

| Tool | Purpose |
|------|---------|
| **Cilium** | K8s CNI + service mesh (sidecarless) |
| **Katran** | L4 load balancer (Facebook's open-source) |
| **bpftrace** | Dynamic tracing (network latency, packet drops) |
| **Falco** | Runtime security (detect suspicious network activity) |
| **Cloudflare's eBPF XDP** | DDoS mitigation at kernel level |

**Why it's trending:**
- **Service mesh without sidecars** — Cilium's sidecarless mesh = 50% less memory
- **Observability** — Replace tcpdump, strace with eBPF probes (less overhead)
- **Cloud adoption** — GKE uses Cilium by default, AWS/EKS has eBPF CNI options

**Interview angle:**
> "Have you worked with eBPF? How would you use it to debug a network issue?"
> 
> **Answer:** "I'd use `bpftrace` to trace TCP retransmits in real-time without restarting the app:
> ```bash
> bpftrace -e 'kprobe:tcp_retransmit_skb { @retransmits[comm] = count(); }'
> ```
> This shows which processes are retransmitting, helping identify congestion or packet loss."

### 💡 Q: Service mesh: Sidecar vs sidecarless — what's better in 2025?

**Sidecar model (Istio, Linkerd):**
```
Pod
├── App container
└── Sidecar proxy (Envoy)
    ├── mTLS
    ├── Routing
    └── Metrics
```

**Pros:**
- Mature, feature-rich (Istio = gold standard)
- Language-agnostic (proxy handles everything)
- Fine-grained control (per-pod config)

**Cons:**
- **Resource overhead** — 2x memory/CPU per pod
- Complexity — More containers to manage
- Latency — Extra hop through proxy

**Sidecarless model (Cilium, Ambient Mesh):**
```
Pod
└── App container
     │
     └─► eBPF hooks in kernel
         (mTLS, routing, metrics)
```

**Pros:**
- **Low overhead** — No sidecar container
- Faster (kernel-level, no extra hop)
- Simpler deployment

**Cons:**
- Less mature (Cilium service mesh GA'd in 2024)
- Feature parity catching up to Istio
- Requires kernel 5.x+ with eBPF support

**Which to choose?**

| Scenario | Recommendation |
|----------|----------------|
| **Production, need stability** | Istio (sidecar) |
| **Cost/performance critical** | Cilium (sidecarless) |
| **Simple L7 routing + mTLS** | Linkerd (sidecar, lighter than Istio) |
| **Hybrid** | Istio Ambient Mesh (sidecarless option, preview in 2025) |

**Interview answer:**
> "I'd evaluate based on workload. For high-scale, low-latency services (like real-time APIs), I'd pilot Cilium to avoid sidecar overhead. For complex traffic management (weighted canary, circuit breaking), I'd stick with Istio for maturity."

### 💡 Q: HTTP/3 and QUIC — production-ready in 2025?

**Adoption status (2025):**
- **Google:** 100% of traffic over QUIC
- **Facebook/Meta:** QUIC for mobile apps
- **Cloudflare, Fastly:** Enabled by default
- **AWS CloudFront:** HTTP/3 available (opt-in)
- **Browsers:** Chrome, Firefox, Safari, Edge = full support
- **Servers:** Nginx 1.25+, Caddy, LiteSpeed = production-ready

**When to enable:**
- **Mobile-heavy traffic** — Benefits most (connection migration on network change)
- **High-latency networks** — 0-RTT resumption = huge win
- **Video streaming** — No head-of-line blocking

**When NOT to enable (yet):**
- **UDP blocked** — Some corporate firewalls drop UDP/443
- **Debugging complexity** — Tooling (tcpdump, Wireshark) still maturing for QUIC

**How to enable:**

```nginx
# Nginx 1.25+
server {
    listen 443 quic reuseport;  # HTTP/3 over QUIC
    listen 443 ssl;              # HTTP/2 fallback

    ssl_protocols TLSv1.3;
    http3 on;

    add_header Alt-Svc 'h3=":443"; ma=86400';  # Advertise HTTP/3 support
}
```

**Check if site supports HTTP/3:**
```bash
curl -I --http3 https://cloudflare.com
# HTTP/3 200
```

---

## Scenario-Based Questions

### ⭐ Q: Your application is experiencing intermittent 502 errors. Troubleshoot.

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

### ⭐ Q: Latency increased for users in Europe but not in US. What happened?

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

### Books & Guides
- **Computer Networking: A Top-Down Approach** — Kurose & Ross (THE networking textbook)
- **High Performance Browser Networking** — Ilya Grigorik (free online: hpbn.co)
- **Beej's Guide to Network Programming** — Classic sockets programming guide
- **TCP/IP Illustrated** — W. Richard Stevens (deep protocol internals)

### Online Resources
- **Cloudflare Learning Center** — https://www.cloudflare.com/learning/ (best for CDN, DDoS, DNS)
- **Julia Evans' Networking Zines** — Bite-sized networking concepts (tcpdump, DNS, HTTP)
- **Brendan Gregg's Network Tools** — https://www.brendangregg.com/linuxperf.html
- **eBPF.io** — https://ebpf.io/what-is-ebpf/ (eBPF networking intro)
- **Cilium Blog** — https://cilium.io/blog/ (modern K8s networking)

### RFCs (Reference)
- **RFC 2616** — HTTP/1.1
- **RFC 7540** — HTTP/2
- **RFC 9000** — QUIC
- **RFC 8446** — TLS 1.3
- **RFC 1918** — Private IP address space

### Hands-On Practice
- **PacketLife.net Cheat Sheets** — OSPF, BGP, VLAN, subnetting
- **Cisco Packet Tracer** — Network simulation (free)
- **GNS3** — Network emulator for complex topologies
- **Wireshark University** — Packet analysis courses
