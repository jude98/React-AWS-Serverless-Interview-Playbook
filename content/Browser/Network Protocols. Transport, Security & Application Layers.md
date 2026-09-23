# Network Protocols: Transport, Security & Application Layers

## Key Concepts

> [!summary] Protocol Layer Architecture
> 
> Modern web communication is organized into hierarchical abstraction layers (OSI / TCP-IP models):
> 
>   
> 
> - **Transport Layer (L4)**: Governs host-to-host data delivery via **TCP** (reliable, ordered, connection-oriented) or **UDP** (lightweight, connectionless, low-latency).
>     
>       
>     
> - **Security Layer (L4.5 / L5)**: **TLS/SSL** establishes an encrypted, authenticated session on top of the transport layer before application data is sent.
>     
>       
>     
> - **Application Layer (L7)**: High-level protocols like **HTTP/1.1**, **HTTP/2**, **HTTP/3**, and **SSH** that define message syntax, semantics, and resource retrieval.
>     
>       
>     

> [!abstract] Transport Layer: TCP vs. UDP
> 
>   
> 
> - **TCP (Transmission Control Protocol)**: Establishes a virtual circuit via a **3-Way Handshake (SYN, SYN-ACK, ACK)**. Guarantees in-order packet delivery, error-checking, acknowledgments, retransmissions, flow control (sliding window), and congestion control.
>     
>       
>     
> - **UDP (User Datagram Protocol)**: "Fire-and-forget" connectionless datagram delivery. No handshake, no delivery guarantees, no retransmissions, and no ordering. Offers minimal protocol overhead and zero head-of-line blocking at the transport layer.
>     
>       
>     

> [!info] Security: TLS/SSL vs. SSH
> 
>   
> 
> - **TLS (Transport Layer Security)**: Successor to SSL (Secure Sockets Layer, now deprecated). Secures application traffic (like HTTP $\to$ HTTPS) using public-key cryptography (X.509 PKI certificates) to authenticate servers and negotiate ephemeral symmetric session keys.
>     
>       
>     
> - **SSH (Secure Shell)**: An application/transport protocol designed specifically for secure remote server administration, shell access, tunneling, and file transfer (`sftp`/`scp`). Uses asymmetric host keys and user authentication (public/private key pairs or passwords), bypassing public Certificate Authorities (CAs) by default via a trust-on-first-use (TOFU) or enterprise certificate authority model.
>     
>       
>     

> [!tip] HTTP vs. HTTPS
> 
>   
> 
> - **HTTP**: Application-layer protocol transmitted in **plaintext** over port 80. Susceptible to packet-sniffing, man-in-the-middle (MITM) tampering, and session hijacking.
>     
>       
>     
> - **HTTPS**: Standard HTTP layered directly over an encrypted **TLS tunnel** over port 443. Guarantees **Confidentiality** (data encryption), **Integrity** (message tampering detection via HMACs), and **Authentication** (proves server identity via trusted CAs).
>     
>       
>     

> [!danger] HTTP Evolution: HTTP/1.1 vs. HTTP/2.0
> 
>   
> 
> - **HTTP/1.1**: Text-based protocol. Suffers from **Application-Level Head-of-Line (HoL) Blocking** on a single TCP connection because requests must be served in the exact order they were sent. Browsers mitigate this by opening up to 6 concurrent TCP connections per domain.
>     
>       
>     
> - **HTTP/2.0**: Binary-framed protocol. Introduces **Multiplexing** (multiple concurrent bidirectional request/response streams over a **single TCP connection**), HPACK header compression, stream prioritization, and Server Push.
>     
>       
>     

## Common Interview Questions

- "What is the difference between TCP and UDP? When would you choose UDP over TCP?"

- "What is the concrete difference between HTTP and HTTPS? What extra latency does HTTPS introduce?"

- "What is TLS/SSL, and how does the hybrid encryption model (asymmetric + symmetric) work during the TLS handshake?"

- "Compare SSH and TLS: How do their trust models and typical use cases differ?"

- "What is Head-of-Line (HoL) blocking, and how does HTTP/2 solve it at the application layer while remaining vulnerable to it at the transport layer?"

- "How does HTTP/2 multiplexing work over a single TCP connection compared to HTTP/1.1?"

- "What is HPACK in HTTP/2, and why was header compression necessary?"

## Deep Dive & Talking Points

### 1. TCP vs. UDP: Mechanics & Trade-Offs

- **TCP Guarantees**:

    - **Connection-Oriented**: 3-Way Handshake establishes sequence numbers.

    - **Reliability & Ordering**: If packet #2 is lost, the receiver buffers packet #3 and holds delivery to the application until packet #2 is retransmitted.

    - **Flow & Congestion Control**: Dynamically scales transmission rate using sliding window algorithms (e.g., CUBIC, BBR) to avoid overwhelming the network or receiver buffer.

    - **Trade-off**: High connection setup latency ($1\text{ RTT}$) and packet retransmission delays.

- **UDP Characteristics**:

    - Zero handshake ($0\text{ RTT}$ startup).

    - 8-byte header overhead (compared to TCP's minimum 20-byte header).

    - Packets can arrive out of order, duplicated, or not at all.

    - **Use Cases**: Real-time gaming, VoIP (Zoom, Discord), DNS queries, WebRTC media streams, and **HTTP/3 (QUIC)**.

### 2. TLS/SSL & The Hybrid Encryption Pattern

- Modern cryptography avoids using purely asymmetric or purely symmetric encryption:

    - **Asymmetric Encryption (RSA, ECC)**: Highly secure for initial key exchange without prior shared secrets, but computationally slow and CPU-heavy.

    - **Symmetric Encryption (AES-GCM, ChaCha20-Poly1305)**: Blazing fast and efficient on modern CPUs with hardware acceleration, but requires both parties to possess the same shared key securely.

- **The Hybrid Solution**:

    1. The **TLS Handshake** uses asymmetric cryptography (Diffie-Hellman / ECDHE) and digital certificates to authenticate the server and safely negotiate a temporary secret.

    2. Once the shared secret is established, the connection switches entirely to **symmetric encryption** for bulk data transfer.

### 3. SSH vs. TLS

|**Dimension**|**TLS (Transport Layer Security)**|**SSH (Secure Shell)**|
|---|---|---|
|**Primary Domain**|Web browsers, REST/GraphQL APIs, microservices|Remote terminal access, SFTP, port forwarding|
|**Trust Model**|Centralized Public Key Infrastructure (PKI) with Root Certificate Authorities (Let's Encrypt, DigiCert)|Trust On First Use (TOFU) or pinned public keys in `~/.ssh/authorized_keys`|
|**Client Auth**|Usually anonymous clients (server authenticated); optional mTLS|Key-pair authentication (`id_ed25519`) or password|
|**Port**|443 (HTTPS), 853 (DoT), 587 (SMTPS)|22 (Default)|

### 4. HTTP/1.1 vs. HTTP/2.0

- **The Problem in HTTP/1.1**:

    - Pipelining was unreliable and disabled by default. If a client requested an image and a script over a single TCP connection, the script response could not be returned until the image was fully transmitted (**Application HoL Blocking**).

    - Workarounds: Domain sharding (splitting assets across `static1.cdn.com`, `static2.cdn.com`), CSS sprite sheets, asset inlining.

- **The HTTP/2 Solution**:

    - **Binary Framing Layer**: Replaces plain text with binary frames (`HEADERS`, `DATA`, `SETTINGS`).

    - **Multiplexing**: Streams are split into discrete, numbered frames that interleave over a single connection and reassemble at the receiver.

    - **HPACK Compression**: Headers are no longer sent repeatedly in plaintext. A dynamic shared lookup table compresses headers by sending only diffs/indices across requests.

- **The HTTP/2 Transport Catch (Why HTTP/3 exists)**:

    - HTTP/2 multiplexes streams over **one TCP connection**.

    - If a single TCP packet is dropped at the IP layer, TCP halts the _entire_ connection until the missing packet is retransmitted. This causes **Transport-Level Head-of-Line Blocking**, stalling all multiplexed streams simultaneously. (HTTP/3 fixes this by running over UDP via QUIC).

## Code Snippets / Architectural Comparisons

### 1. HTTP/1.1 Plaintext vs. HTTP/2 Binary Frames

HTTP

```text
/* HTTP/1.1: Human-Readable Plaintext Format */
GET /api/v1/users HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: application/json
Cookie: session_id=xyz123...
(Headers sent in full plaintext on EVERY single request)
```

```text
/* HTTP/2: Binary Framing Layer */
+-----------------------------------------------+
| Length (24)  | Type (8)     | Flags (8)       |
+-----------------------------------------------+
| R (1)        | Stream Identifier (31)          |
+-----------------------------------------------+
| Frame Payload (e.g., HEADERS, DATA) ...       |
+-----------------------------------------------+

Stream 1 (GET /api/users)    ──[HEADERS Frame]──────────[DATA Frame]──>
Stream 3 (GET /styles.css)   ─────[HEADERS Frame]──[DATA Frame]───────>
(Interleaved over a SINGLE TCP socket without waiting)
```

### 2. Node.js Raw TCP vs. UDP Server Implementation

```javascript
// ==========================================
// 1. TCP Server (Reliable, Stream-oriented)
// ==========================================
import net from "node:net";

const tcpServer = net.createServer((socket) => {
  console.log("Client connected via TCP 3-Way Handshake");

  socket.on("data", (data) => {
    console.log("Received ordered byte stream:", data.toString());
    socket.write("TCP ACK + Response");
  });

  socket.on("end", () => console.log("TCP connection closed cleanly"));
});

tcpServer.listen(8080, () => console.log("TCP Server listening on 8080"));

// ==========================================
// 2. UDP Server (Connectionless, Datagram-oriented)
// ==========================================
import dgram from "node:dgram";

const udpServer = dgram.createSocket("udp4");

udpServer.on("message", (msg, rinfo) => {
  // No connection established; sender info (IP/port) arrives with each packet
  console.log(`Received UDP packet from ${rinfo.address}:${rinfo.port}: ${msg.toString()}`);

  const reply = Buffer.from("UDP Datagram Reply");
  udpServer.send(reply, rinfo.port, rinfo.address);
});

udpServer.bind(8081, () => console.log("UDP Server listening on 8081"));
```

### 3. Establishing HTTPS with Native Node.js TLS

```javascript
import https from "node:https";
import fs from "node:fs";

// HTTPS requires SSL/TLS Certificate and Private Key
const options = {
  key: fs.readFileSync("server-key.pem"),
  cert: fs.readFileSync("server-cert.pem")
};

const server = https.createServer(options, (req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("Secured with TLS 1.3\n");
});

server.listen(443, () => {
  console.log("HTTPS Server listening on port 443");
});
```

## Comparison Matrices

### Transport Layer: TCP vs. UDP

|**Feature**|**TCP (Transmission Control Protocol)**|**UDP (User Datagram Protocol)**|
|---|---|---|
|**Connection State**|Connection-oriented (3-way handshake)|Connectionless (no handshake)|
|**Reliability**|Guaranteed delivery (retransmissions)|Best-effort (packets can drop)|
|**Packet Ordering**|Strictly in-order (sequence numbers)|No order guarantee|
|**Header Overhead**|20–60 bytes|**8 bytes**|
|**Congestion Control**|Yes (scales window based on latency/drop)|None (transmits at application rate)|
|**Speed / Latency**|Slower due to handshakes and ACKs|**Fastest, near zero overhead**|
|**Common Protocols**|HTTP/1.1, HTTP/2, SSH, FTP, SMTP|DNS, VoIP, WebRTC, HTTP/3 (QUIC)|

### Application Layer: HTTP/1.1 vs. HTTP/2.0

|**Feature**|**HTTP/1.1**|**HTTP/2.0**|
|---|---|---|
|**Data Format**|Plaintext ASCII|**Binary Framing**|
|**Concurrency Model**|Sequential per TCP socket (up to 6 connections)|**True Multiplexing (Single TCP connection)**|
|**Header Handling**|Redundant plaintext headers on every request|**Compressed via HPACK (indexed tables)**|
|**Head-of-Line Blocking**|Present at the HTTP application layer|**Solved at application layer** (persists at TCP layer)|
|**Server Capabilities**|Response only upon request|**Server Push** (proactively push cacheable assets)|
|**Transport Protocol**|TCP|TCP|

## Related Topics

- [[What Happens When You Enter a URL in the Browser. The End-to-End Lifecycle|What Happens When You Enter a URL in the Browser: The End-to-End Lifecycle]]

- [[Client-Side Browser Storage. Mechanisms, Architecture & Security|Client-Side Browser Storage: Mechanisms, Architecture & Security]]

- [[Web Security & Identity Architecture. SOP, XSS, CSRF & Token Lifecycles|Frontend Web Security: XSS, CSRF, CORS & CSP]]

- [[Cross-Tab Communication in Modern Browsers. Mechanisms, Architecture & Trade-Offs|WebSockets, Server-Sent Events (SSE) & Real-Time Architectures]]

## Tags

#fullstack #interview #networking #http #https #tcp #udp #tls #ssl #ssh #http2

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups