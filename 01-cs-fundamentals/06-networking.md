# Networking Essentials

> **Tier:** T2 (working knowledge) · **Related:** [How the web works](../04-web-fundamentals/01-how-the-web-works.md), [Realtime and WebSockets](../04-web-fundamentals/09-realtime-websockets.md), [Caching and performance](../04-web-fundamentals/08-caching-and-performance.md), [Web security](../04-web-fundamentals/07-web-security-owasp.md), [System design fundamentals](../15-system-design/01-fundamentals.md), [Operating systems](05-operating-systems.md)
> **On your resume:** REST APIs and third-party integrations (PriceCharting) in Rails and Node, a React Native app used over mobile networks, and real-time features at Devntech; Web Development lab TA at PUCIT.

## Quick revision
- OSI has 7 layers (physical, data link, network, transport, session, presentation, application); the TCP/IP model the internet actually uses has 4 (link, internet, transport, application).
- IP addresses identify hosts; ports identify processes. A TCP connection is identified by source IP, source port, destination IP and destination port.
- IPv4 is 32-bit (`203.0.113.7`), IPv6 is 128-bit (`2001:db8::1`). Private ranges: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16. Loopback: 127.0.0.1 and ::1.
- NAT maps many private addresses to one public address, which is why a laptop on Wi-Fi cannot receive webhooks without a tunnel.
- TCP: connection-oriented, reliable, ordered, with flow and congestion control; three-way handshake (SYN, SYN-ACK, ACK). UDP: connectionless, no delivery guarantees, low overhead.
- DNS: browser cache, OS cache, recursive resolver, root, TLD, authoritative server; answers are cached for their TTL.
- Record types: A (IPv4), AAAA (IPv6), CNAME (alias to another name), MX (mail servers), TXT (verification, SPF, DKIM), NS (authoritative servers).
- HTTP/1.1: text, one request at a time per connection. HTTP/2: binary, multiplexed streams over one TCP connection, header compression. HTTP/3: HTTP over QUIC on UDP, no TCP head-of-line blocking, faster setup.
- TLS authenticates the server with a CA-signed certificate and negotiates symmetric session keys; TLS 1.3 needs one round trip.
- URL to page: DNS, TCP plus TLS (or QUIC), HTTP request, CDN or load balancer, reverse proxy, app server, database, response, browser rendering.
- WebSocket: an HTTP/1.1 `Upgrade` request answered with `101 Switching Protocols`, then full-duplex messages on the same connection.
- L4 load balancers route by IP and port; L7 load balancers route by HTTP details (host, path, headers) and usually terminate TLS.
- A reverse proxy (Nginx) fronts servers: TLS, static files, buffering, compression, rate limiting. A CDN caches content at edge locations near users.
- Tools: `ping`, `traceroute`/`tracert`, `dig`, `nslookup`, `curl -v`, `ss`/`netstat`, `lsof -i :3000`.

## 1. OSI vs TCP/IP layers

| OSI layer | TCP/IP layer | Data unit | Examples | Responsibility |
|---|---|---|---|---|
| 7 Application | Application | message | HTTP, DNS, SMTP, SSH, WebSocket | application semantics |
| 6 Presentation | Application | | encoding, compression, encryption (TLS is often placed here) | data representation |
| 5 Session | Application | | session setup and resumption | managing conversations |
| 4 Transport | Transport | segment (TCP), datagram (UDP) | TCP, UDP, QUIC (built on UDP) | process-to-process delivery, ports, reliability |
| 3 Network | Internet | packet | IPv4, IPv6, ICMP | addressing and routing across networks |
| 2 Data link | Link | frame | Ethernet, Wi-Fi, ARP | delivery on the local network, MAC addresses |
| 1 Physical | Link | bits | copper, fiber, radio | signals on the medium |

The OSI model is a teaching reference; real protocols map onto the simpler TCP/IP stack. TLS sits between TCP and HTTP and does not fit either model neatly, which is fine to say in an interview.

## 2. Addressing: IP, ports and NAT

- **IPv4** addresses are 32 bits (about 4.3 billion, long exhausted). **IPv6** addresses are 128 bits written in hex groups, with `::` compressing runs of zeros.
- **CIDR** notation: `192.168.1.0/24` means the first 24 bits identify the network, leaving 256 addresses; `/32` is a single host. Cloud networks and security groups are written this way.
- **Private ranges** (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) are not routed on the public internet. `127.0.0.1` reaches the same machine; binding a server to `0.0.0.0` listens on all interfaces, which is why a Rails server inside Docker needs `-b 0.0.0.0`.
- **NAT** (network address translation) rewrites private source addresses and ports to a public IP on the way out and maps replies back. Inbound connections need port forwarding, a load balancer or a tunnel, which is why third-party webhooks cannot reach `localhost:3000` directly.
- **Ports** are 16-bit numbers (0 to 65535). Well known: 22 SSH, 25 SMTP, 53 DNS, 80 HTTP, 443 HTTPS. Common in your stack: 3000 (Rails and many Node dev servers), 5432 PostgreSQL, 3306 MySQL, 6379 Redis, 27017 MongoDB, 8081 (the React Native Metro bundler). Clients use temporary ephemeral ports.

## 3. TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Connection | handshake before data | none: just send datagrams |
| Reliability | acknowledgments and retransmission | none; the application decides what to do about loss |
| Ordering | in-order byte stream | datagrams may arrive out of order or duplicated |
| Flow and congestion control | yes | no |
| Header size | 20 bytes or more | 8 bytes |
| Head-of-line blocking | one lost segment stalls everything behind it | no |
| Typical uses | HTTP/1.1 and HTTP/2, database connections, SSH, email | DNS queries, voice and video calls, games, QUIC (HTTP/3) |

```text
Client                              Server
  | -------- SYN (seq = x) ---------> |
  | <-- SYN-ACK (seq = y, ack = x+1)- |
  | -------- ACK (ack = y+1) -------> |   connected after one round trip
  | ======== request / response ===== |
  | -------- FIN -------------------> |   each side sends FIN and acknowledges the other's
```

TCP achieves reliability with sequence numbers, acknowledgments, retransmissions and checksums; flow control stops a fast sender from overwhelming a slow receiver, and congestion control backs off when the network drops packets. The three-way handshake lets both sides agree on initial sequence numbers. Because every new connection costs at least one round trip (more with TLS), reusing connections matters: HTTP keep-alive, HTTP client connection pools and database connection pools.

## 4. DNS

Resolving `shop.example.com`:

1. The browser checks its DNS cache, then the operating system's cache and `hosts` file.
2. The OS asks the configured **recursive resolver** (ISP, company, or a public resolver such as 1.1.1.1 or 8.8.8.8).
3. On a cache miss, the resolver asks a **root server**, which refers it to the `.com` **TLD servers**.
4. The TLD servers refer it to the domain's **authoritative name servers** (its NS records).
5. The authoritative server answers with the record (an A record, or a CNAME the resolver then follows).
6. The resolver caches the answer for its **TTL** and returns it; the browser connects to the IP.

| Record | Maps | Example |
|---|---|---|
| A | name to IPv4 address | `shop.example.com. A 203.0.113.7` |
| AAAA | name to IPv6 address | `shop.example.com. AAAA 2001:db8::7` |
| CNAME | name to another name (an alias) | `www.example.com. CNAME shop.example.com.` |
| MX | domain to mail servers, with priority | `example.com. MX 10 mail.example.com.` |
| TXT | free text: domain verification, SPF, DKIM, DMARC | `example.com. TXT "v=spf1 include:_spf.mailer.example ~all"` |
| NS | zone to its authoritative name servers | `example.com. NS ns1.dns-provider.example.` |

- **TTL** controls caching. Before a migration, lower the TTL (for example to 300 seconds), wait for the old TTL to expire, switch the record, then raise it again. "DNS propagation" is mostly caches expiring.
- A CNAME cannot coexist with other records at the same name, so the zone apex (`example.com`) cannot be a CNAME; DNS providers offer ALIAS or flattening features instead.
- DNS mostly uses UDP port 53 and falls back to TCP for large responses.
- A CNAME is a DNS alias, not an HTTP redirect: the browser still shows the original name.

## 5. HTTP/1.1 vs HTTP/2 vs HTTP/3

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | QUIC over UDP |
| Format | text | binary frames | binary frames |
| Concurrency | one request at a time per connection; browsers open several connections per host | many multiplexed streams over one connection | multiplexed streams that are independent at the transport level |
| Header compression | none | HPACK | QPACK |
| Head-of-line blocking | at the HTTP level | gone at the HTTP level, but one lost TCP packet still stalls every stream | gone: a lost packet affects only its own stream |
| Encryption | optional (HTTPS is HTTP over TLS) | TLS in practice (browsers require it) | always: TLS 1.3 is built into QUIC |
| Connection setup | TCP handshake, then TLS | same | combined transport and TLS handshake in one round trip; 0-RTT resumption |
| Extras | keep-alive | stream prioritization; server push existed but browsers dropped it | connection migration: switching from Wi-Fi to mobile data keeps the connection |

HTTP semantics (methods, status codes, headers, caching rules) are the same in all three versions; only the wire format and transport change. HTTP/2 and HTTP/3 are usually enabled at the CDN, load balancer or reverse proxy, while app servers such as Puma often speak HTTP/1.1 behind them. HTTP/3 helps most on lossy mobile networks, which is where React Native app users often are. See [REST API design](../04-web-fundamentals/04-rest-api-design.md) for the semantics.

## 6. TLS and certificates

A simplified TLS 1.3 handshake:

1. **ClientHello:** supported versions and cipher suites, an ephemeral key share (Diffie-Hellman public value), and **SNI**, the hostname, so one IP address can serve many certificates.
2. **ServerHello:** the chosen cipher and the server's key share. Both sides now derive the same session keys. The server sends its **certificate chain** and a signature proving it holds the certificate's private key, then Finished.
3. The **client verifies** the chain up to a trusted root CA in its trust store, checks the hostname and validity dates, and sends Finished. From here on, application data is encrypted with fast symmetric ciphers.

TLS 1.3 completes in one round trip (TLS 1.2 needed two), and resumed sessions can send 0-RTT data with replay caveats. Asymmetric cryptography authenticates the server and agrees on keys; symmetric cryptography protects the bulk data.

A **certificate** binds a public key to domain names and is signed by a certificate authority; browsers trust a chain from the leaf certificate through intermediates to a preinstalled root. Let's Encrypt issues free, short-lived certificates renewed automatically through the ACME protocol. Classic production incidents: expired certificates, a missing intermediate certificate, hostname mismatch and wrong system clocks. HTTPS gives confidentiality and integrity in transit and authenticates the server; it does not fix application vulnerabilities. HSTS tells browsers to use HTTPS only for a domain.

## 7. What happens when you type a URL

For `https://shop.example.com/listings?page=2`:

1. **Parse and check caches:** the browser splits the URL into scheme, host, port (443 by default for HTTPS), path and query, applies HSTS (force HTTPS), and may answer from its HTTP cache or a service worker.
2. **DNS:** resolve the hostname to an IP address (section 4), often a CDN edge address.
3. **Connect:** TCP three-way handshake plus TLS handshake, or a single QUIC handshake for HTTP/3. An existing connection may be reused.
4. **Request:** `GET /listings?page=2` with headers such as `Host`, `Cookie`, `Accept` and `User-Agent`.
5. **Edge:** a CDN may serve cached content; otherwise the request goes to the origin's load balancer.
6. **Load balancer and reverse proxy:** TLS is usually terminated here; a healthy app server is chosen, and Nginx (or similar) forwards the request to Puma or a Node process.
7. **Application:** middleware (logging, sessions, authentication), routing, the controller or handler, database queries over a connection pool, maybe a Redis cache, then HTML or JSON rendering.
8. **Response:** status line, headers (`Content-Type`, `Cache-Control`, `Set-Cookie`) and a body, usually compressed.
9. **Render:** the browser parses HTML into the DOM and CSS into the CSSOM, builds the render tree, lays out and paints; it fetches discovered CSS, JavaScript and images (repeating steps 2 to 8, often over the same connection), runs JavaScript (React hydrates or mounts) and makes further API calls.
10. **Afterwards:** connections stay open for reuse, and caches make the next visit faster.

Interviewers usually want this skeleton plus depth in one area of your choice. Browser rendering details are in [How the web works](../04-web-fundamentals/01-how-the-web-works.md).

## 8. WebSockets

```text
GET /cable HTTP/1.1
Host: app.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the `101` response, the same TCP connection carries framed messages in both directions until either side closes it; `wss://` runs it over TLS. Use cases: chat, live notifications, collaborative editing and dashboards. In Rails, Action Cable provides WebSocket channels (mounted at `/cable` by default) and powers Turbo Stream broadcasts; in Node, `ws` and Socket.IO are common (Socket.IO adds its own protocol, rooms and fallbacks). Alternatives: short polling, long polling and Server-Sent Events (one-way server-to-client streaming over plain HTTP).

Operational concerns: load balancers must allow the upgrade and long-lived connections (idle timeouts), multiple servers need a shared pub/sub backend (such as Redis for Action Cable) to broadcast to every client, connections must be authenticated when they open, clients should reconnect with backoff, and mobile apps lose sockets in the background, so offline users need push notifications instead.

## 9. Load balancers, reverse proxies and CDNs

| | L4 load balancer | L7 load balancer |
|---|---|---|
| Sees | IP addresses, ports, TCP or UDP | full HTTP: host, path, headers, cookies |
| Routing unit | a connection | a request: `/api` to one pool, other paths to another, routing by host name |
| TLS | usually passed through | usually terminated (decrypts, inspects, re-encrypts if needed) |
| Extras | very fast, protocol-agnostic | header rewriting, cookie-based sticky sessions, WebSocket handling, web application firewall, rate limiting |
| Examples | AWS Network Load Balancer, HAProxy in TCP mode | AWS Application Load Balancer, Nginx, HAProxy in HTTP mode, Cloudflare |

Balancing algorithms include round robin, least connections and IP hash, and health checks take failing instances out of rotation.

A **forward proxy** acts on behalf of clients (a corporate proxy); a **reverse proxy** acts on behalf of servers. Nginx in front of Puma terminates TLS, serves static and precompiled assets, buffers slow clients so app threads are not tied up, compresses responses, limits request sizes and can rate-limit. Apps generated by Rails 8 also run Thruster, a lightweight proxy in front of Puma, for similar duties.

A **CDN** is a network of edge caches close to users. It serves static assets (and cacheable pages or API responses) with low latency, reduces origin load and absorbs traffic spikes and some attacks. You control it with `Cache-Control` headers (`public, max-age=31536000, immutable` for fingerprinted assets), cache keys and purges. See [Caching and performance](../04-web-fundamentals/08-caching-and-performance.md).

## 10. Debugging tools

```bash
ping api.example.com                        # reachable? round-trip time (ICMP may be blocked)
traceroute api.example.com                  # hop-by-hop path (tracert on Windows)
dig shop.example.com A +short               # just the DNS answer
dig +trace shop.example.com                 # follow resolution from the root servers down
nslookup shop.example.com                   # simple lookup, also available on Windows
curl -I https://shop.example.com            # response headers only (HEAD request)
curl -v https://shop.example.com/api/listings   # DNS, connection, TLS, request and response headers
curl -X POST https://api.example.com/orders -H "Content-Type: application/json" -d '{"sku":"A1"}'
lsof -i :3000                               # which process holds port 3000 ("address already in use")
ss -tlnp                                    # listening TCP sockets on Linux (netstat -ano on Windows)
```

A systematic "the site or API is down" check: does the name resolve (`dig`)? Is the host reachable (`ping`, `traceroute`)? Does the TLS handshake succeed and what status comes back (`curl -v`)? Then read the server, proxy and application logs.

## Common pitfalls
- Claiming HTTP/2 removed head-of-line blocking entirely; it still exists at the TCP level, which HTTP/3 fixes.
- Confusing a CNAME (DNS alias) with an HTTP redirect (301/302).
- Forgetting DNS caching (TTL) when switching servers, then debugging "random" traffic to the old host.
- Binding a development server to `127.0.0.1` inside a container and wondering why it is unreachable.
- Assuming HTTPS hides which site a user visits: DNS queries and SNI can reveal hostnames.
- Putting WebSockets behind a load balancer without upgrade support or with short idle timeouts.
- Calling UDP "bad because unreliable"; it is the right choice when timeliness beats completeness.
- Expecting third-party webhooks to reach a laptop behind NAT without a tunnel.

## How to talk about this in an interview
- You assisted the Web Development lab at PUCIT, so the request/response cycle, DNS and HTTP basics should be fluent. Practice the URL walkthrough out loud in under two minutes, then be ready to go deeper on whichever step the interviewer picks.
- Connect networking to integration work: timeouts and retries for third-party APIs, connection reuse, TLS errors, rate limits; and to mobile: unreliable networks, reconnecting sockets, push notifications for backgrounded apps.
- > **Fill in:** one networking problem you debugged at Devntech (for example a failing third-party API call, a webhook that could not reach your environment, or a WebSocket that kept disconnecting) and which tools you used.
- > **Fill in:** how traffic reaches your Rails apps in production (CDN, load balancer, Nginx or Thruster, Puma), if you know it, so you can describe the path concretely.

## Interview questions

**Q1. What happens when you type a URL into the browser and press Enter?**
The browser parses the URL and checks HSTS and its caches, resolves the hostname through DNS, opens a TCP connection and performs a TLS handshake (or one QUIC handshake for HTTP/3), and sends an HTTP request. The request may be answered by a CDN or pass through a load balancer and reverse proxy to an app server, which runs middleware, routing, controller code and database queries. The response comes back with headers and a body, and the browser builds the DOM and CSSOM, lays out, paints, and fetches further resources.
*Follow-up:* Where can caching short-circuit this? Browser cache, service worker, DNS caches, CDN, reverse proxy and application caches.

**Q2. TCP vs UDP?**
TCP is connection-oriented and gives reliable, ordered delivery with flow and congestion control, at the cost of handshakes and head-of-line blocking. UDP sends independent datagrams with no guarantees and minimal overhead. Use TCP for web pages, APIs and databases; UDP for DNS, real-time voice, video and games, and as the base for QUIC.
*Follow-up:* If UDP is unreliable, how does HTTP/3 work? QUIC implements reliability, ordering and congestion control per stream on top of UDP.

**Q3. Explain the TCP three-way handshake. Why three steps?**
The client sends SYN with its initial sequence number, the server replies SYN-ACK with its own sequence number and an acknowledgment, and the client sends ACK. Three messages are the minimum for both sides to announce and confirm their sequence numbers, proving each direction works.
*Follow-up:* How is a connection closed? Each side sends FIN and acknowledges the other's FIN.

**Q4. What are the OSI layers, and where do HTTP, TLS, TCP and IP sit?**
Physical, data link, network, transport, session, presentation and application. HTTP is application layer, TLS sits between transport and application (often called presentation), TCP is transport, and IP is network. The TCP/IP model collapses these into link, internet, transport and application.
*Follow-up:* Where does a MAC address belong? The data link layer.

**Q5. How does DNS resolution work, and what is the difference between an A record and a CNAME?**
The resolver checks caches, then walks from root servers to TLD servers to the domain's authoritative servers, and caches the answer for its TTL. An A record maps a name directly to an IPv4 address; a CNAME maps a name to another name, which must then be resolved.
*Follow-up:* Why can't the root domain be a CNAME? A CNAME cannot coexist with the SOA and NS records every zone apex needs, so providers offer ALIAS or flattening.

**Q6. Compare HTTP/1.1, HTTP/2 and HTTP/3.**
HTTP/1.1 is text-based and handles one request at a time per connection. HTTP/2 uses binary frames to multiplex many streams over one TCP connection with header compression, but TCP packet loss still stalls every stream. HTTP/3 runs over QUIC on UDP, so streams are independent, handshakes are faster, and connections survive network changes.
*Follow-up:* Do you need to change application code for HTTP/2? Usually not; semantics are identical and it is often enabled at the proxy or CDN.

**Q7. How does HTTPS work?**
TLS runs over the transport connection. In the handshake, client and server exchange key shares to derive symmetric session keys, and the server proves its identity with a certificate chain signed by a trusted CA plus a signature from its private key. After that, all HTTP data is encrypted and integrity-protected with symmetric ciphers.
*Follow-up:* What does SNI do? It sends the hostname in the ClientHello so one IP address can host many certificates.

**Q8. WebSockets vs polling vs Server-Sent Events?**
Polling repeatedly asks the server for updates, which is simple but wasteful and laggy. Server-Sent Events keep one HTTP response open for one-way server-to-client streaming. WebSockets upgrade an HTTP connection into a persistent, full-duplex channel for low-latency messages in both directions, at the cost of stateful connections that need load balancer support and reconnection logic.
*Follow-up:* How do you broadcast across several app servers? Use a shared pub/sub backend such as Redis.

**Q9. L4 vs L7 load balancers?**
An L4 load balancer routes connections using IP addresses and ports without reading HTTP, which is fast and protocol-agnostic. An L7 load balancer understands HTTP, so it can route by host, path or headers, terminate TLS, rewrite headers and keep sticky sessions with cookies.
*Follow-up:* Which do you need for path-based routing to different services? L7.

**Q10. Reverse proxy vs forward proxy, and why put Nginx in front of Puma?**
A forward proxy acts for clients reaching out; a reverse proxy acts for servers receiving traffic. Nginx in front of Puma terminates TLS, serves static assets, buffers slow clients so Puma threads stay free, compresses responses, enforces limits and can balance across app instances.
*Follow-up:* What is slow-client buffering? The proxy receives the full request and sends the full response at the client's pace, so the app server is busy only briefly.

**Q11. What is a CDN, and how do you control what it caches?**
A geographically distributed set of edge servers that cache content close to users, cutting latency and origin load. `Cache-Control` headers set cacheability and lifetime, fingerprinted asset names allow long `immutable` caching, and purges or new file names handle updates. Personalized responses should be marked `private` or not cached.
*Follow-up:* Why fingerprint asset file names? A changed file gets a new URL, so you can cache aggressively without serving stale assets.

**Q12. What are NAT and private IP ranges?**
Private ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) are reused inside networks and not routed on the internet. NAT on a router or cloud gateway rewrites private addresses to a public one for outgoing traffic and maps responses back. Inbound connections therefore need port forwarding, a load balancer or a tunnel.
*Follow-up:* Why does IPv6 reduce the need for NAT? Its address space is large enough for every device to have a public address.

**Q13. A production server cannot reach a third-party API. How do you debug it?**
Check DNS from the server (`dig`), then connectivity (`curl -v` shows whether the TCP connection or TLS handshake fails, or which HTTP status returns), then firewalls, security groups and egress rules, then the API's status page and rate limits. Compare with the same request from your machine and read application logs for timeouts and error bodies.
*Follow-up:* What makes integrations resilient to this? Timeouts, retries with backoff, circuit breakers and background jobs instead of synchronous calls in web requests.

## Further reading
- [MDN: HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [MDN: The WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
- [MDN glossary: DNS](https://developer.mozilla.org/en-US/docs/Glossary/DNS)
