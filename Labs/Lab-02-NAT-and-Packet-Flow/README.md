# Lab 02 — VirtualBox NAT and Packet Flow

**Category:** Networking Fundamentals  
**Environment:** Windows 11 Enterprise VM in VirtualBox (NAT mode)  
**Date Completed:** April 2026  
**Status:** ✅ Complete

---

## Lab Objective

Use a working NAT-based VirtualBox VM to understand how Windows decides where packets go, how it learns local next-hop hardware addresses, how to inspect active network sockets, and how the same guest traffic looks different when observed from the host because of NAT/PAT translation.

---

## Environment

| Component | Details |
|-----------|---------|
| VM | Windows 11 Enterprise in Oracle VirtualBox |
| Network mode | Adapter 1 — NAT |
| Guest IP | 10.0.2.15 |
| Guest gateway | 10.0.2.2 |
| Guest DNS | 10.0.2.3 |
| Host IP | 192.168.4.20 |

---

## Commands Used

| Purpose | Command |
|---------|---------|
| Verify NAT baseline guest addressing | `ipconfig /all` |
| Inspect the guest ARP cache | `arp -a` |
| Review guest route table and default route | `route print` |
| Observe path visibility through VirtualBox NAT | `tracert 8.8.8.8` |
| List guest sockets and owning PIDs | `netstat -ano` |
| Filter active guest connections only | `netstat -ano \| findstr ESTABLISHED` |
| Map a guest PID to a process name | `tasklist \| findstr 3140` |
| Map svchost.exe to its hosted service | `tasklist /svc \| findstr 3140` |
| Resolve a hostname using guest DNS | `nslookup google.com` |
| Find VirtualBox-related processes on the host | `tasklist \| findstr /i virtualbox` |
| Show established host-side connections | `netstat -ano \| findstr ESTABLISHED` |
| Isolate one VirtualBox VM process on the host | `netstat -ano \| findstr "13552"` |
| Check whether a VirtualBox process hosts services | `tasklist /svc \| findstr 13552` |

---

## Key Findings

### 1. ARP Showed the Local Next Hop

The guest ARP table included an entry for `10.0.2.2` with a physical MAC address, confirming the VM already had a valid local next hop for off-subnet traffic.

### 2. The Default Route Explained Packet Direction

The guest default route (`0.0.0.0/0`) pointed to gateway `10.0.2.2` via interface `10.0.2.15`. Any traffic outside the `10.0.2.0/24` subnet is handed off to the VirtualBox NAT gateway.

### 3. Traceroute Visibility Was Incomplete by Design

`tracert 8.8.8.8` showed the destination appearing as hop 1. This was not because Google was one router away — VirtualBox NAT was abstracting and hiding the intermediate hops.

### 4. Netstat Showed the Difference Between Listening and Established

- **LISTENING** — a local port is open and waiting for incoming connections
- **ESTABLISHED** — an active conversation is live
- One guest example: `10.0.2.15:49677 → 104.208.203.88:443` owned by PID 3140

### 5. PID-to-Service Mapping Made Traffic Meaningful

PID 3140 mapped to `svchost.exe`. Running `tasklist /svc` revealed the hosted service as **WpnService** (Windows Push Notification Service), turning a raw socket entry into a real explanation.

### 6. NAT/PAT in Action — Guest View vs Host View

| Viewpoint | Example Connection | What It Shows |
|-----------|-------------------|---------------|
| Inside the guest | `10.0.2.15:49677 → 104.208.203.88:443` | Guest sees its own private IP and original source port |
| On the host after NAT/PAT | `192.168.4.20:62849 → 104.208.203.88:443` | Host sees VirtualBox rewriting source IP and source port |

NAT/PAT rewrote **both** the source IP and the source port. The same traffic looks fundamentally different depending on where it is observed.

### 7. CLOSE_WAIT and Loopback Entries Added Socket State Context

A host-side `CLOSE_WAIT` entry showed a connection where the remote side closed first and the local side had not finished its teardown. Loopback UDP entries (`127.0.0.1 → 127.0.0.1`) showed VirtualBox communicating internally between helper processes on the same host.

### 8. DNS Tied the Whole Stack Together

`nslookup google.com` used DNS server `10.0.2.3` and returned both IPv4 and IPv6 answers. The "non-authoritative answer" result confirmed the DNS server was forwarding or caching the response rather than acting as Google's authoritative source.

---

## What I Learned

- A machine does not send off-subnet traffic directly to the final destination — it first builds a local Ethernet frame addressed to the **gateway's MAC address**.
- The **default route** (`0.0.0.0/0`) is the catch-all Windows uses when no more specific route exists.
- Traceroute shows what the network allows you to see, not always the full real path. NAT, firewalls, and virtual edges can hide intermediate hops.
- `netstat` is far more useful when combined with `tasklist` and `tasklist /svc` — raw socket entries alone don't explain which service owns the traffic.
- **NAT/PAT rewrites both the source IP and the source port.** The same conversation looks different depending on whether you observe it from inside the guest or from the host.
- TCP socket states: LISTENING (open door waiting), ESTABLISHED (active conversation), CLOSE_WAIT (remote closed first, local cleanup pending).

---

## Why This Matters for SOC Work

- Builds packet-flow intuition: local next hop → routing decision → socket ownership → translated egress.
- Explains why analysts see **different addresses and ports** depending on where traffic is captured.
- Reinforces the difference between **connectivity, observability, and attribution**. A connection can be active, but process and service mapping are required to understand it operationally.
- NAT awareness is critical for log correlation — a SOC analyst who doesn't account for NAT translation may misidentify the true source of traffic.

---

## Tools Used

- Oracle VirtualBox
- Windows 11 Enterprise (evaluation VM)
- Built-in Windows commands: `ipconfig`, `arp`, `route`, `tracert`, `netstat`, `tasklist`, `nslookup`
