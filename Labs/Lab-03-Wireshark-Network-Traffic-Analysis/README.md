# Lab 03 — Windows Firewall, Ports, and Reachability

**Platform:** Oracle VirtualBox | Windows 11 Enterprise VM  
**Date:** April 2026  
**Author:** Dylan McDougal

---

## Key Takeaway

> A port can be listening and explicitly allowed by Windows Firewall, yet still remain unreachable from another machine if the network path does not exist.

---

## Objective

This lab was designed to isolate and test three conditions that are frequently conflated in Windows networking troubleshooting:

1. A service is **listening** on a port
2. Windows Firewall **allows** inbound traffic to that port
3. Another machine can **actually reach** the service over the network

Each condition is necessary but not sufficient on its own. The goal was to prove that gap through controlled testing.

---

## Environment

| Component | Value |
|---|---|
| Guest OS | Windows 11 Enterprise (Evaluation) |
| Virtualization Platform | Oracle VirtualBox |
| Initial Adapter Mode | NAT |
| NAT-mode Guest IP | 10.0.2.15 |
| Later Adapter Mode | Bridged Adapter |
| Bridged-mode Guest IP | 192.168.4.22 |
| Test Port | TCP 8080 |
| Built-in Ports Observed | 135 (RPC Endpoint Mapper), 139, 445 (SMB) |

---

## Lab Progression

### Phase 1 — Baseline: What is the VM already listening on?

Ran `netstat` to identify services in a LISTENING state and examined the difference between `0.0.0.0` (all interfaces) and `127.0.0.1` (loopback only).
```powershell
netstat -ano | findstr LISTENING
```
TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       1084
TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
TCP    0.0.0.0:5040           0.0.0.0:0              LISTENING       1272
TCP    127.0.0.1:49669        0.0.0.0:0              LISTENING       4324
Mapped PID 4 to confirm it was the System process (core Windows networking):
```powershell
tasklist /svc /FI "PID eq 4"
```
Image Name   PID   Services
=========    ===   ========
System       4     N/A
Confirmed the guest IP via `ipconfig`:
```powershell
ipconfig
```
Ethernet adapter Ethernet:
IPv4 Address. . . . . . . . . . . : 10.0.2.15
Subnet Mask . . . . . . . . . . . : 255.255.255.0
Default Gateway . . . . . . . . . : 10.0.2.2
The `10.0.2.15` address is the standard VirtualBox NAT guest address. This is a private IP assigned internally by VirtualBox — the host has no direct route to it.

---

### Phase 2 — Local vs. Remote Reachability in NAT Mode

Tested TCP 445 from inside the VM (loopback):
```powershell
Test-NetConnection -ComputerName 10.0.2.15 -Port 445
```
ComputerName     : 10.0.2.15
RemoteAddress    : 10.0.2.15
RemotePort       : 445
TcpTestSucceeded : True
Tested the same port from the host — result was **failure**. This confirmed the critical distinction: **listening does not equal remote reachability.**

---

### Phase 3 — Custom Listener on TCP 8080

Created a PowerShell TCP listener on port 8080 to eliminate SMB-specific variables and test with a clean, controlled service:
```powershell
$l = [System.Net.Sockets.TcpListener]::new([Net.IPAddress]::Any, 8080)
$l.Start()
```

Confirmed the listener was bound to all interfaces, not loopback:
```powershell
netstat -ano | findstr :8080
```
TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       9804
Local test from inside the VM — **succeeded.**  
Host test with VM still in NAT mode — **failed.**

---

### Phase 4 — Firewall Rule Test (NAT Mode)

Checked for an existing allow rule on TCP 8080 — none found. Created one:
```powershell
New-NetFirewallRule -DisplayName "Allow TCP 8080 Lab" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow
```

Retested from the host with the firewall rule in place — **still failed.**

This isolated the root cause: the firewall was not the primary blocker. The network path from the host to the NAT guest simply did not exist.

Removed the temporary rule and confirmed host-to-guest failure was unchanged — consistent with that conclusion.

---

### Phase 5 — Network Mode Change: NAT → Bridged Adapter

Changed the VM adapter from NAT to Bridged. After reboot, the guest received a LAN-facing address:
```powershell
ipconfig
```
Ethernet adapter Ethernet:
IPv4 Address. . . . . . . . . . . : 192.168.4.22
Subnet Mask . . . . . . . . . . . : 255.255.255.0
Default Gateway . . . . . . . . . : 192.168.4.1
Recreated the 8080 listener and retested from the host:
```powershell
Test-NetConnection -ComputerName 192.168.4.22 -Port 8080
```
ComputerName     : 192.168.4.22
RemoteAddress    : 192.168.4.22
RemotePort       : 8080
TcpTestSucceeded : True
**Succeeded.** Same service, same port, same firewall state. The only change was the network topology.

---

## Results by Test Condition

| Condition | Listener State | Firewall State | Host Reachability |
|---|---|---|---|
| NAT mode, no firewall rule | Listening on 0.0.0.0:8080 | No rule for 8080 | **Failed** |
| NAT mode, with firewall rule | Listening on 0.0.0.0:8080 | Allow rule created | **Still failed** |
| NAT mode, rule removed | Listening on 0.0.0.0:8080 | Rule removed | **Still failed** |
| Bridged mode | Listening on 0.0.0.0:8080 | No special rule | **Succeeded** |

---

## Analysis

The decisive variable was network topology, not service state or firewall policy.

In NAT mode, the guest's `10.0.2.15` address is a VirtualBox-internal address. The host has no direct route to it. Inbound connections from the host to the guest are blocked at the virtualization layer — adding a Windows Firewall allow rule does not fix a missing network path. Those are two separate decision points in the traffic flow.

After switching to Bridged Adapter mode, the guest joined the same LAN as the host (`192.168.4.22`). The host now had a direct path. No additional firewall changes were required — the listener and the existing default policy were already sufficient once the path existed.

This pattern reflects a common real-world troubleshooting failure: engineers confirm a service is running and a firewall rule exists, then escalate when the service is still unreachable, without checking whether a valid network path actually exists between source and destination.

---

## Key Concepts Reinforced

- **Listening is a host-local condition.** It tells you the service is running, not that it is externally accessible.
- **Firewall policy is one decision point, not the whole path.** A packet can pass firewall inspection and still fail if the routing or network topology doesn't support the connection.
- **Reachability requires the full stack:** valid addressing, correct topology, a listening service, and any required policy allowance — all four.
- **NAT and Bridged networking are fundamentally different** for host-to-guest inbound testing. NAT hides the guest behind a translated address with no inbound path by default.
- **Controlled variable isolation** — changing one thing at a time — is what makes root cause identification reliable rather than accidental.

---

## SOC and Blue Team Relevance

This troubleshooting model — *Is it listening? Is policy allowing it? Is the path actually there?* — is the same sequence a SOC analyst works through when triaging connectivity-based alerts or investigating suspected lateral movement.

A port scan showing TCP 445 or 3389 as open on an internal host means the service responded. It does not tell you whether that host is legitimately reachable from the segment the scan originated from, whether a firewall policy should have blocked it, or whether the path exists because of a misconfiguration. This lab's methodology — isolating service state, firewall state, and network path independently — is directly applicable to that kind of investigation.

Understanding why a connection succeeds or fails at each layer is also foundational to reading firewall logs, IDS alerts, and SIEM events accurately. An alert that fires on an attempted connection to TCP 8080 carries different weight depending on whether the host is NAT-isolated or directly LAN-accessible.

---

## Commands Reference

| Command | Purpose |
|---|---|
| `netstat -ano \| findstr LISTENING` | Show all listening ports with owning PIDs |
| `tasklist /svc /FI "PID eq 4"` | Map a PID to its associated service or process name |
| `ipconfig` | Confirm IP address and adapter configuration |
| `Test-NetConnection -ComputerName <IP> -Port <Port>` | Test TCP reachability to a specific host and port |
| `$l = [System.Net.Sockets.TcpListener]::new(...)` | Create a custom PowerShell TCP listener for testing |
| `New-NetFirewallRule ...` | Create an inbound Windows Firewall allow rule |

---

*Part of the [SOC Lab Journey](https://github.com/dylan-mcdougal/SOC-Lab-Journey) — building toward SOC analyst and blue team roles through hands-on lab work, Network+, and Security+ certification.*
