[🏠 Back to Main Notes](../../README.md) • [💻 CS Theory CN Module](../../CS%20Theory/03-computer-networks.md)

# Computer Networks (CN) - Interview Questions & Answers

Refer to the complete, updated 2026 Master Guide in:
👉 **[03. Computer Networks (CN) Master Guide](../../CS%20Theory/03-computer-networks.md)**
👉 **[07. CS Theory Interview Mastery](../../CS%20Theory/07-cs-theory-interview-mastery.md)**

---

## 🎯 Quick Top Interview Highlights

### 1. What happens when you enter a URL in the browser?
- **DNS Lookup:** Resolves domain name $\rightarrow$ IP address (Browser Cache $\rightarrow$ OS Cache $\rightarrow$ Resolver $\rightarrow$ Root $\rightarrow$ TLD $\rightarrow$ Authoritative).
- **TCP 3-Way Handshake:** `SYN` $\rightarrow$ `SYN-ACK` $\rightarrow$ `ACK`.
- **TLS 1.3 Handshake:** Diffie-Hellman key exchange for symmetric session key.
- **HTTP Request / Response:** Client sends `GET`, Server returns payload (HTML/JSON).
- **Browser Rendering:** DOM Tree $\rightarrow$ CSSOM $\rightarrow$ Render Tree $\rightarrow$ Layout $\rightarrow$ Paint.

### 2. Difference between TCP and UDP?
- **TCP:** Connection-oriented, guarantees packet delivery & order, has flow & congestion control. Used for Web (HTTP/1,2), Databases, SSH.
- **UDP:** Connectionless, no delivery guarantees, minimal latency header (8 bytes). Used for DNS, Video Streaming, Gaming, and HTTP/3 (QUIC).

### 3. HTTP/1.1 vs HTTP/2 vs HTTP/3?
- **HTTP/1.1:** Head-of-Line blocking at application level; sequential requests.
- **HTTP/2:** Multiplexing over 1 TCP connection, binary framing, HPACK header compression.
- **HTTP/3:** Runs on **QUIC (UDP)**, eliminates TCP-level Head-of-Line blocking with independent streams and 0-RTT connection resumption.
