[🏠 Back to CS Theory Index](./README.md) • [⬅️ Prev: Database Systems](./02-database-management-systems.md) • [Next: Distributed Systems ➡️](./04-distributed-systems.md)

<div align="center">
  <h1>03. Computer Networks (CN)</h1>
  <p><b>OSI & TCP/IP Models, DNS, HTTP/1.1 to HTTP/3 (QUIC), TLS 1.3, TCP Flow/Congestion, & WebSockets</b></p>
</div>

---

## 📑 Module Index
- [1. Network Layering Models](#1-network-layering-models)
  - [The 7-Layer OSI Model vs 5-Layer TCP/IP Architecture](#osi-vs-tcp-ip)
  - [Encapsulation & Protocol Data Units (PDUs)](#encapsulation--pdus)
- [2. Application Layer Protocols](#2-application-layer-protocols)
  - [DNS Resolution Lifecycle (Recursive vs Iterative)](#dns-resolution)
  - [HTTP/1.1 vs HTTP/2 vs HTTP/3 (QUIC over UDP)](#http-evolution)
- [3. Transport Layer: TCP & UDP](#3-transport-layer-tcp--udp)
  - [TCP vs UDP Core Differences](#tcp-vs-udp)
  - [TCP 3-Way Handshake & 4-Way Connection Teardown](#tcp-handshake-and-teardown)
  - [TCP Flow Control (Sliding Window) & Congestion Control (CUBIC/BBR)](#flow-and-congestion-control)
- [4. Security & Cryptography](#4-security--cryptography)
  - [Symmetric vs Asymmetric Encryption](#cryptography-basics)
  - [TLS 1.3 Handshake (1-RTT & 0-RTT Resumption)](#tls-13-handshake)
- [5. Network & Routing Layers](#5-network--routing-layers)
  - [IPv4 vs IPv6, Subnetting (CIDR), NAT & ARP](#ip-addressing--nat)
- [6. Real-Time Communication Protocols](#6-real-time-communication-protocols)
  - [WebSockets vs Server-Sent Events (SSE) vs gRPC vs WebRTC](#real-time-protocols)

---

## 1. Network Layering Models

### OSI vs TCP/IP

```
      OSI 7-LAYER MODEL                                TCP/IP 5-LAYER SUITE
  ┌───────────────────────┐                          ┌───────────────────────┐
  │ 7. Application Layer  │ ──┐                      │                       │
  │ 6. Presentation Layer │ ──┼────────────────────► │ 5. Application Layer  │ (HTTP, DNS, SSH, gRPC)
  │ 5. Session Layer      │ ──┘                      │                       │
  ├───────────────────────┤                          ├───────────────────────┤
  │ 4. Transport Layer    │ ───────────────────────► │ 4. Transport Layer    │ (TCP, UDP, QUIC)
  ├───────────────────────┤                          ├───────────────────────┤
  │ 3. Network Layer      │ ───────────────────────► │ 3. Network Layer      │ (IP, ICMP, BGP, OSPF)
  ├───────────────────────┤                          ├───────────────────────┤
  │ 2. Data Link Layer    │ ───────────────────────► │ 2. Data Link Layer    │ (Ethernet, Wi-Fi, MAC)
  ├───────────────────────┤                          ├───────────────────────┤
  │ 1. Physical Layer     │ ───────────────────────► │ 1. Physical Layer     │ (Cables, Fiber, Radio)
  └───────────────────────┘                          └───────────────────────┘
```

- **Protocol Data Units (PDUs):**
  - Application $\rightarrow$ **Data / Payload**
  - Transport $\rightarrow$ **Segment** (TCP) / **Datagram** (UDP)
  - Network $\rightarrow$ **Packet** (IP)
  - Data Link $\rightarrow$ **Frame** (MAC)
  - Physical $\rightarrow$ **Bits** ($0, 1$)

---

## 2. Application Layer Protocols

### DNS Resolution Lifecycle

```
Client ──► [ Local DNS Resolver ] ──► [ Root Server (.) ]
                  │             ◄── (Returns .com TLD IP)
                  ├──► [ TLD Server (.com) ]
                  │             ◄── (Returns authoritative nameserver IP)
                  └──► [ Authoritative Nameserver (example.com) ]
                                ◄── (Returns A Record IP: 93.184.216.34)
```

- **A Record:** Maps domain $\rightarrow$ IPv4 address.
- **AAAA Record:** Maps domain $\rightarrow$ IPv6 address.
- **CNAME:** Canonical name alias (maps subdomain $\rightarrow$ another domain).
- **MX Record:** Mail Exchange server routing.

---

### HTTP Evolution

```
┌─────────────────┬─────────────────┬────────────────────────────────────────────────────────┐
│ Protocol        │ Transport       │ Core Features & Fixes                                  │
├─────────────────┼─────────────────┼────────────────────────────────────────────────────────┤
│ **HTTP/1.1**    │ TCP             │ Keep-Alive connections; suffers from **Head-of-Line (HoL) Blocking** at application layer.│
├─────────────────┼─────────────────┼────────────────────────────────────────────────────────┤
│ **HTTP/2**      │ TCP             │ Binary framing, **Multiplexing** (multiple requests over 1 TCP connection), Header Compression (HPACK), Server Push.│
├─────────────────┼─────────────────┼────────────────────────────────────────────────────────┤
│ **HTTP/3**      │ **QUIC (UDP)**  │ Eliminates TCP-level HoL blocking; zero-RTT connection resumption; built-in TLS 1.3 encryption.│
└─────────────────┴─────────────────┴────────────────────────────────────────────────────────┘
```

---

## 3. Transport Layer: TCP & UDP

```
┌───────────────────────────────────────────────┬───────────────────────────────────────────────┐
│               TCP (Transmission Control)      │            UDP (User Datagram)                │
├───────────────────────────────────────────────┼───────────────────────────────────────────────┤
│ • Connection-oriented (3-Way Handshake)       │ • Connectionless (Fire and forget)            │
│ • Guaranteed delivery (Acks & Retransmits)    │ • No delivery guarantees (Packet loss allowed)│
│ • In-order delivery & Byte-stream oriented    │ • Out-of-order packet arrival possible        │
│ • Congestion control & Flow control           │ • Lightweight; minimal header (8 bytes)       │
│ • Use Case: Web (HTTP/1,2), DBs, SSH, Email   │ • Use Case: DNS, Video Streaming, Gaming, VoIP│
└───────────────────────────────────────────────┴───────────────────────────────────────────────┘
```

---

### TCP 3-Way Handshake & 4-Way Teardown

```
         ESTABLISHING CONNECTION (3-Way)                    CLOSING CONNECTION (4-Way)
      CLIENT                      SERVER                 CLIENT                      SERVER
        │                           │                      │                           │
        │─── SYN (seq = x) ────────►│                      │─── FIN (seq = u) ────────►│
        │                           │                      │                           │
        │◄── SYN-ACK (ack=x+1,seq=y)│                      │◄── ACK (ack = u+1) ───────│
        │                           │                      │                           │
        │─── ACK (ack = y+1) ──────►│                      │◄── FIN (seq = v) ─────────│
        │                           │                      │                           │
   [ ESTABLISHED ]             [ ESTABLISHED ]             │─── ACK (ack = v+1) ──────►│
                                                           │                           │
                                                      [ TIME_WAIT (2*MSL) ]        [ CLOSED ]
```

---

### Flow Control vs Congestion Control

1. **Flow Control (Protects the Receiver):**
   - Receiver sends its available buffer space in the **Window Size field (TCP Sliding Window)**. Sender never transmits more bytes than the receiver's advertised window.
2. **Congestion Control (Protects the Network):**
   - **Slow Start:** Exponential window growth ($1 \rightarrow 2 \rightarrow 4 \rightarrow 8 \dots$) until threshold `ssthresh`.
   - **Congestion Avoidance:** Linear window growth ($+1$ MSS per RTT).
   - **Modern Algorithms:** **BBR (Bottleneck Bandwidth and RTT)** by Google uses model-based rate pacing instead of packet-loss indicators.

---

## 4. Security & Cryptography

### TLS 1.3 Handshake

TLS 1.3 reduced handshake latency from 2-RTT (TLS 1.2) to **1-RTT (and 0-RTT on resumption)**:

```
CLIENT                                                          SERVER
  │                                                               │
  │─── ClientHello (Supported Ciphers + Key Share diffie-hellman)─►│
  │                                                               │
  │◄── ServerHello (Selected Cipher + Server Key Share + Cert)───│
  │                                                               │
 [ Compute Symmetric Session Key ]               [ Compute Symmetric Session Key ]
  │                                                               │
  │◄═══════════════ Encrypted Application Data (HTTP) ═══════════►│
```

---

## 5. Network & Routing Layers

### IPv4 vs IPv6 & Subnetting

- **IPv4:** 32-bit address (`192.168.1.1` $\approx$ 4.3 Billion addresses — exhausted).
- **IPv6:** 128-bit hexadecimal address (`2001:0db8:85a3:0000:0000:8a2e:0370:7334` $\approx 3.4 \times 10^{38}$ addresses).
- **CIDR (Classless Inter-Domain Routing):** `192.168.1.0/24` means the first 24 bits are Network prefix, leaving $32 - 24 = 8$ bits for host addresses ($2^8 - 2 = 254$ usable hosts).

---

## 6. Real-Time Communication Protocols

| Technology | Communication Direction | Underlying Protocol | Best Use Case |
| :--- | :--- | :--- | :--- |
| **HTTP Polling** | Unidirectional (Client $\rightarrow$ Server pull) | TCP / HTTP | Legacy dashboards with low refresh rate. |
| **Server-Sent Events (SSE)** | Unidirectional (Server $\rightarrow$ Client push) | HTTP/2 or HTTP/1.1 | **LLM Token Streaming (ChatGPT)**, live score feeds. |
| **WebSockets** | **Full-Duplex** (Bi-directional persistent) | TCP (Starts via HTTP Upgrade) | **Real-Time Chat**, multiplayer games, trading desks. |
| **gRPC** | Bi-directional streaming | HTTP/2 + Protocol Buffers | **Microservice-to-microservice** low-latency RPCs. |
| **WebRTC** | Peer-to-Peer Bi-directional | UDP / SRTP | Audio/Video calling (Zoom, Google Meet). |

---

[➡️ Continue to Module 04: Distributed Systems & Cloud](./04-distributed-systems.md)
