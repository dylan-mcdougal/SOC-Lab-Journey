# Lab 03 — Wireshark Network Traffic Analysis

**Platform:** Oracle VirtualBox | Ubuntu 24.04 LTS VM  
**Date:** March 2026  
**Author:** Dylan McDougal

---

## Key Takeaway

> Normal traffic has a recognizable structure. DNS resolves before data flows, TCP handshakes precede every reliable connection, and TLS makes application data unreadable at the packet level — understanding that baseline is what makes abnormal traffic detectable.

---

## Objective

Capture and analyze live network traffic using Wireshark inside an Ubuntu VM. The goal was to observe real packet-level activity across multiple OSI model layers — DNS resolution, ICMP ping behavior, TCP handshake sequences, and HTTPS/TLS negotiation — and build a baseline understanding of what normal traffic looks like in a controlled environment.

---

## Environment

| Component | Value |
|---|---|
| Guest OS | Ubuntu 24.04 LTS |
| Virtualization Platform | Oracle VirtualBox |
| Network Interface | enp0s3 (NAT adapter) |
| Wireshark Version | 4.2.2 |
| Additional Tools | Firefox, Linux Terminal |
| Network Mode | NAT (10.0.2.15) |

---

## Lab Progression

### Phase 1 — Capture Setup

Launched Wireshark and selected the `enp0s3` interface — Ubuntu's primary virtual ethernet adapter in NAT mode. Started a live capture before generating any traffic to ensure the full sequence of each protocol exchange would be captured from the start.

```bash
wireshark &
```

Selected `enp0s3` from the interface list and began capture.

---

### Phase 2 — DNS Traffic

Ran `ping google.com` from the terminal to trigger DNS resolution and ICMP traffic simultaneously.

```bash
ping -c 4 google.com
```

```
PING google.com (173.194.219.100) 56(84) bytes of data.
64 bytes from 173.194.219.100: icmp_seq=1 ttl=255 time=14.3 ms
64 bytes from 173.194.219.100: icmp_seq=2 ttl=255 time=13.8 ms
64 bytes from 173.194.219.100: icmp_seq=3 ttl=255 time=14.1 ms
64 bytes from 173.194.219.100: icmp_seq=4 ttl=255 time=13.9 ms
```

Wireshark captured DNS query packets before any ICMP traffic appeared — confirming that name resolution must complete before the ping can send. The VM sent a standard query to the DNS server at `10.0.2.3`, which responded with `173.194.219.100` for `google.com`.

Confirmed the same resolution behavior with `nslookup`:

```bash
nslookup github.com
```

```
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   github.com
Address: 140.82.113.4
```

DNS operates at Layer 7 and uses UDP at Layer 4 — both visible in Wireshark's protocol column.

---

### Phase 3 — ICMP Traffic

Applied a Wireshark display filter to isolate ping traffic:

```
icmp
```

Alternating Echo Request and Echo Reply packets were visible between `10.0.2.15` (VM) and `173.194.219.100` (Google). Four requests and four replies matched the `-c 4` flag used in the ping command.

TTL values observed:
- VM outbound packets: TTL 64 (standard Linux default)
- Google responses: TTL 255 (consistent with network infrastructure)

ICMP operates at Layer 3 and carries no port numbers — distinguishing it from TCP and UDP traffic.

---

### Phase 4 — TCP Three-Way Handshake

Applied a Wireshark filter to isolate SYN packets:

```
tcp.flags.syn == 1
```

Multiple TCP handshake sequences were visible. Each connection followed the expected pattern:

1. **SYN** — VM sends connection request from a high ephemeral port to destination port 443
2. **SYN-ACK** — Remote server acknowledges and responds
3. **ACK** — VM confirms, connection established

This sequence repeated for every new HTTPS connection during the Firefox browsing session, confirming that TCP establishes a reliable path before any application data is transmitted.

---

### Phase 5 — HTTPS and TLS Negotiation

Applied a filter for HTTPS traffic:

```
tcp.port == 443
```

Following each TCP handshake, TLS negotiation packets were visible in sequence:

- Client Hello
- Server Hello
- Certificate
- Server Key Exchange
- Client Key Exchange

Remote servers observed included Google (`34.160.144.191`) and Cloudflare infrastructure (`151.101.113.91`) serving GitHub content. After TLS negotiation completed, all subsequent data was encrypted and unreadable in the capture — the expected and correct behavior of HTTPS.

---

### Phase 6 — OSI Layer Inspection in a Single Packet

Clicked into individual packets to examine the full layer stack in Wireshark's detail panel. A single captured packet revealed all layers represented simultaneously:

| Layer | What Was Visible |
|---|---|
| Layer 2 — Data Link | Ethernet II frame with source and destination MAC addresses |
| Layer 3 — Network | IPv4 header with source/destination IP addresses and TTL |
| Layer 4 — Transport | TCP header with port numbers, sequence numbers, and flags |
| Layer 7 — Application | Protocol identifier (DNS, TLS, HTTP) |

---

## OSI Layers Observed

| Layer | Evidence in Capture |
|---|---|
| Layer 7 — Application | DNS queries and responses, HTTPS protocol activity |
| Layer 6 — Presentation | TLS Client Hello, Server Hello, Certificate exchange |
| Layer 4 — Transport | TCP SYN, SYN-ACK, ACK sequences on port 443; UDP on port 53 |
| Layer 3 — Network | IP addresses in all packets, ICMP ping traffic, TTL values |
| Layer 2 — Data Link | Ethernet II MAC addresses in packet detail panel |

---

## Analysis

DNS resolution consistently preceded ICMP and TCP traffic in every test — confirming that name resolution is a dependency, not a parallel process. An analyst looking at a host's traffic should expect to see DNS queries before outbound connections; their absence or unusual destination can indicate hardcoded IPs or DNS tunneling.

The TCP handshake sequences were uniform and predictable across all HTTPS connections. Deviations from that pattern — incomplete handshakes, unexpected SYN floods, or RST packets — are the kind of anomalies that generate IDS alerts and require triage.

TLS negotiation made all HTTPS payload data unreadable at the packet level, which is the correct behavior. However, the metadata remained visible: server IPs, timing, packet volume, and certificate exchange details. Traffic analysis at this level can identify suspicious destinations even when payload content is encrypted.

---

## SOC and Blue Team Relevance

Establishing a baseline of normal traffic behavior is a prerequisite for identifying abnormal traffic. This lab documented what that baseline looks like in practice: DNS before data, handshake before payload, TLS making application data opaque while metadata remains visible.

In a SOC context, Wireshark is used for deep packet inspection during incident response — examining specific hosts or sessions flagged by a SIEM or IDS alert. The filters used here (`icmp`, `tcp.flags.syn == 1`, `tcp.port == 443`) are the same building blocks used to isolate suspicious traffic during an investigation. The ability to read a packet capture and distinguish expected protocol behavior from anomalous behavior is a direct analyst skill.

---

## Filters Used

| Filter | Purpose |
|---|---|
| `icmp` | Isolate ping request and reply traffic |
| `tcp.flags.syn == 1` | Show only TCP SYN packets to view handshake initiations |
| `tcp.port == 443` | Filter for HTTPS traffic |
| `dns` | Isolate DNS query and response packets |

---

*Part of the [SOC Lab Journey](https://github.com/dylan-mcdougal/SOC-Lab-Journey) — building toward SOC analyst and blue team roles through hands-on lab work, Network+, and Security+ certification.*
