# Lab 01 — VirtualBox NAT and Bridged Networking

**Category:** Networking Fundamentals  
**Environment:** Ubuntu 24.04 LTS VM in VirtualBox  
**Date Completed:** March 2026  
**Status:** ✅ Complete

---

## Lab Objective

Compare VirtualBox NAT and Bridged Adapter networking modes by observing how the guest VM's IP address, subnet, and network visibility change between modes. Verify findings using live command output and document directional ping behavior to understand what each mode allows and restricts.

---

## Environment

| Component | Details |
|-----------|---------|
| VM | Ubuntu 24.04 LTS in Oracle VirtualBox |
| Host OS | Windows 11 |
| Mode 1 | Adapter 1 — NAT |
| Mode 2 | Adapter 1 — Bridged Adapter |

---

## Commands Used

| Purpose | Command |
|---------|---------|
| View IP address, subnet mask, MAC address | `ip a` |
| Verify default gateway | `ip route` |
| Test internet connectivity from guest | `ping google.com` |
| Ping host from guest (Bridged mode) | `ping <host IP>` |
| Ping guest from host (NAT mode — expected failure) | `ping <guest IP>` |
| Ping guest from host (Bridged mode) | `ping <guest IP>` |
| Verify DNS resolution | `nslookup google.com` |

---

## Key Findings

### 1. NAT Mode — Guest IP and Addressing

In NAT mode, VirtualBox assigned the guest a private IP in the `10.0.2.0/24` range:

- Guest IP: `10.0.2.15`
- Subnet mask: `255.255.255.0`
- Default gateway: `10.0.2.2`
- DNS server: `10.0.2.3`

These addresses are internal to VirtualBox — no other device on the home network sees them.

### 2. Bridged Mode — Guest IP and Addressing

In Bridged mode, the guest received an IP address from the home network's DHCP server directly:

- Guest IP: address on the `192.168.x.x` home network range
- Subnet mask matched the home network
- Default gateway matched the home router

The guest appeared as a fully independent device on the local network — indistinguishable from a physical machine.

### 3. MAC Address Differences Between Modes

Each adapter mode produced a different MAC address for the guest's virtual NIC. In NAT mode, VirtualBox uses a virtualized MAC. In Bridged mode, the guest NIC gets its own MAC address visible to the physical network.

### 4. Directional Ping Behavior

| Test | NAT Mode | Bridged Mode |
|------|----------|-------------|
| Guest → internet | ✅ Success | ✅ Success |
| Guest → host | ✅ Success (via virtual NAT gateway) | ✅ Success |
| Host → guest | ❌ Failed (host cannot reach 10.0.2.15) | ✅ Success |
| Host → guest explanation | NAT hides the guest; no inbound route exists | Guest has a real network address the host can reach directly |

The NAT ping failure was **expected and correct** — it confirmed that NAT provides one-way access by design. The guest can initiate outbound connections, but nothing on the outside can initiate inbound connections to the guest.

### 5. DNS Resolved Correctly in Both Modes

`nslookup google.com` returned valid IP addresses in both modes, confirming DNS was functioning correctly via VirtualBox's virtual DNS relay (`10.0.2.3` in NAT, and the home router in Bridged).

---

## NAT vs Bridged — Side-by-Side Comparison

| Feature | NAT | Bridged |
|---------|-----|---------|
| Guest IP range | VirtualBox internal (10.0.2.x) | Real network (192.168.x.x) |
| IP assigned by | VirtualBox built-in DHCP | Home router DHCP |
| Internet access | ✅ Yes | ✅ Yes |
| Host can reach guest | ❌ No | ✅ Yes |
| Guest visible on LAN | ❌ No | ✅ Yes |
| Best for | Safe isolated lab work | Labs requiring LAN visibility |
| Security posture | More isolated | Less isolated |

---

## What I Learned

- NAT mode hides the guest behind VirtualBox's virtual router. The guest can reach out, but nothing can reach in — similar to how home routers protect devices behind NAT.
- Bridged mode makes the VM a full participant on the physical network, with its own DHCP-assigned IP visible to all other devices.
- A failed ping is not always a problem — in NAT mode, the host failing to ping the guest is the **expected and correct behavior**, confirming isolation is working.
- MAC addresses change between adapter modes because VirtualBox generates different virtual NICs for each mode.
- The `ip a` command (Linux) and `ipconfig /all` (Windows) are the first tools to reach for when verifying network configuration.

---

## Why This Matters for SOC Work

- Understanding NAT isolation is foundational for lab safety — keeping malware samples or attack tools isolated from the rest of a network.
- SOC analysts regularly deal with NAT when correlating logs — a private IP in one log and a public IP in another may represent the same device.
- Knowing which network mode a VM is in determines what traffic is visible and where captures need to be taken.
- Directional connectivity awareness (who can reach whom) is essential for firewall rule analysis and network segmentation decisions.

---

## Tools Used

- Oracle VirtualBox
- Ubuntu 24.04 LTS VM
- Built-in Linux commands: `ip a`, `ip route`, `ping`, `nslookup`
