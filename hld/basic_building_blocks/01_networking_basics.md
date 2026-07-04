

> Everything above is HTTP; everything below is IP. Know both layers well enough to reason about latency, failure modes, and where to put a proxy.

## The Layers (OSI simplified)

```
Application    HTTP, gRPC, DNS, TLS-terminated       (what your code sees)
Transport      TCP (reliable), UDP (unreliable)      (ports, streams/datagrams)
Network        IP (v4/v6), routing, ICMP             (hosts across networks)
Link           Ethernet, Wi-Fi, MAC                  (hop-to-hop)
```

Only 3 layers matter in interviews: **IP, TCP/UDP, and the app protocol on top**.

## IP

- **IPv4**: 32-bit, ~4.3B addresses. Exhausted years ago → NAT everywhere.
- **IPv6**: 128-bit, effectively infinite. Adoption growing (Google reports ~45%).
- **CIDR** (`10.0.0.0/16`): the `/16` = first 16 bits are network → 65,536 host addresses.
- **Private ranges**: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — not routable on the public internet.
- **NAT**: home router / cloud NAT gateway rewrites source IP so many private hosts share one public IP.

## TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Connection | 3-way handshake (SYN, SYN-ACK, ACK) | Fire-and-forget |
| Reliability | Retransmits lost packets | None |
| Ordering | Preserves order | Out of order possible |
| Flow control | Yes (window) | No |
| Congestion control | Yes | No |
| Header size | 20-60 bytes | 8 bytes |
| Best for | HTTP, DB, most apps | DNS, VoIP, gaming, QUIC |

### TCP Handshake & Termination
```
Client    SYN(seq=x)          →     Server
Client    ←    SYN-ACK(seq=y, ack=x+1)
Client    ACK(ack=y+1)         →     Server        (connection open)

... data ...

Client    FIN                  →     Server
Client    ←    ACK
Client    ←    FIN
Client    ACK                  →     Server
```

Practical costs: handshake = 1 RTT before data. HTTPS adds another 1-2 RTT for TLS. In a 100 ms RTT link, that's 200-300 ms before your first byte. Keep-alive + TLS session resumption + HTTP/3 (0-RTT) fight this.

### TCP Backpressure & Head-of-Line Blocking
- Window size limits inflight bytes.
- If a segment is lost, everything after it waits — bad for multiplexed streams. HTTP/2 improved this at the app layer; HTTP/3 (over QUIC/UDP) fixes it end-to-end.

## DNS

Translates `api.example.com` → `52.10.20.30`.

**Resolution walk:**
```
Browser cache → OS resolver cache → Recursive resolver (ISP or 8.8.8.8)
  → Root nameserver → TLD nameserver (.com) → Authoritative (example.com)
```
- **TTL** on each record; low TTL = fast propagation, high resolver load.
- **Record types**: A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), TXT (SPF/verification), SRV (service).
- **Latency**: fresh resolution 20-100 ms; cached < 1 ms.

**DNS as a load balancer**: return multiple A records (round-robin), or GeoDNS (Route 53 latency-based routing) to send users to the nearest region.

## TLS (HTTPS = HTTP over TLS)

**Handshake (TLS 1.3, 1-RTT):**
```
Client → Server: ClientHello (supported ciphers, key share)
Server → Client: ServerHello (chosen cipher, cert, key share, Finished)
Client → Server: Finished
--- encrypted app data ---
```

- **TLS 1.2**: 2 RTTs. TLS 1.3: 1 RTT. Resumed session: 0 RTT.
- **Certificates**: server proves identity via a chain rooted at a trusted CA. Browsers ship with ~150 root CAs.
- **PKI**: private key stays on server; public cert distributed. Compromised private key → revoke cert (OCSP / CRL).
- **mTLS**: both sides present certs; used in service meshes and B2B APIs.
- **SNI**: TLS extension telling the server which hostname you want — enables one IP to host many HTTPS sites.

**Practical**: terminate TLS at the LB or edge (CDN) so backends don't have to; use HTTPS internally when crossing zones.

## Latency Numbers Every SDE Should Memorize

| Operation | Latency |
|-----------|---------|
| L1 cache | 0.5 ns |
| L2 cache | 7 ns |
| RAM access | 100 ns |
| SSD random read | 100 µs |
| HDD seek | 10 ms |
| Datacenter round-trip (same DC) | 0.5 ms |
| Cross-region round-trip (US East ↔ US West) | 60-80 ms |
| Cross-ocean (US ↔ Europe) | 80-120 ms |
| US ↔ India | 200-250 ms |
| DNS lookup (cold) | 20-100 ms |
| TLS 1.3 handshake | 1 RTT |
| TCP handshake | 1 RTT |

**Rule of thumb**: RAM is 10⁶× faster than SSD, which is 10⁶× faster than a cross-continent RTT. Data locality matters.

## Interview Q&A

**Q: Why is UDP used for DNS?**
- Small payloads (< 512 bytes usually) fit in a single datagram → no need for stream setup.
- Latency-critical; TCP handshake would double lookup time.
- Falls back to TCP for large responses (DNSSEC, large answers).

**Q: What's the practical difference between TCP and UDP for real apps?**
- Video call: UDP — a lost frame is a blip, retransmit would be stale.
- Bank transaction: TCP — need every byte, in order.
- Modern trend: HTTP/3 uses UDP + QUIC to combine TCP's reliability with UDP's flexibility.

**Q: Why does HTTPS "feel slow" on cold connections?**
- DNS (100ms) + TCP (1 RTT) + TLS (1-2 RTT) = 3-4 round-trips before data.
- Fix: keep-alive, TLS session resumption, HTTP/2 multiplexing, HTTP/3, CDN edge termination.

**Q: How does NAT break with peer-to-peer?**
- Behind NAT you don't have a public IP. Fix with STUN (learn your public IP), TURN (relay), ICE (protocol combining them). WebRTC does this.

**Q: What's the maximum size of a UDP datagram vs TCP segment?**
- UDP: theoretically 65535 bytes, but IP MTU (1500 on Ethernet) fragments larger ones. Best practice: keep under 1400 bytes.
- TCP: stream, no fixed size; broken into segments by MSS (~1460 bytes on Ethernet).

**Q: What's the difference between a port and a socket?**
- Port: a 16-bit number the OS uses to demultiplex packets (0-65535). Well-known < 1024, ephemeral 32768-60999.
- Socket: `(protocol, src_ip, src_port, dst_ip, dst_port)` — a unique 5-tuple identifying a connection. One port can serve millions of concurrent sockets (each with different remote endpoint).

## Gotchas
- **DNS TTL vs cache**: OS or app may ignore TTL and cache forever (Java's default was infinity — hit us hard when IPs change). Set `networkaddress.cache.ttl=30`.
- **TIME_WAIT** exhaustion: closed TCP connections linger 2×MSL (~60s). High-QPS client → runs out of ephemeral ports. Use `SO_REUSEADDR`, keep-alive, or move to HTTP/2.
- **Head-of-line blocking**: bane of HTTP/1.1 pipelines and even HTTP/2 over lossy links. HTTP/3 fixes with per-stream retransmit.
- **MTU mismatch** causing silent fragmentation; DF (don't fragment) bit + Path MTU Discovery.
