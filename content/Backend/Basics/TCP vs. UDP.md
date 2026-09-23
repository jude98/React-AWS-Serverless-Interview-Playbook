# TCP vs. UDP

## Key Concepts

- **Layer 4 Protocols:** Both Transmission Control Protocol (TCP) and User Datagram Protocol (UDP) operate at the Transport Layer of the OSI / TCP-IP stack, multiplexing network traffic using port numbers.

- **TCP is Connection-Oriented:** Establishes a stateful virtual circuit via a three-way handshake (`SYN` $\rightarrow$ `SYN-ACK` $\rightarrow$ `ACK`) before sending application data, and terminates via a four-way handshake (`FIN` / `ACK`).

- **UDP is Connectionless:** Operates on a "fire-and-forget" model with zero handshake overhead; packets (datagrams) are sent directly to the destination IP and port without prior negotiation.

- **Reliability vs. Latency:** TCP guarantees ordered, in-sequence delivery without loss or duplication via acknowledgments (ACKs), sequence numbers, and retransmissions. UDP provides no delivery guarantees—packets can arrive out of order, duplicated, or not at all.

- **Flow & Congestion Control:** TCP implements windowing (sliding window) and congestion avoidance algorithms (CUBIC, BBR) to throttle transmission to match receiver buffer capacity and network saturation. UDP has zero built-in flow/congestion control.

- **Framing Model:** TCP is a continuous byte stream (no application-level message boundaries; requires framing protocols like length-prefixing or delimiters). UDP preserves discrete datagram boundaries (one `send()` corresponds to one `recv()`).

## Common Interview Questions

- What are the fundamental trade-offs between TCP and UDP?

- Walk me through the TCP 3-way handshake and 4-way teardown. What is the purpose of the `TIME_WAIT` state?

- How does TCP ensure ordered delivery and reliability under the hood?

- What is Head-of-Line (HoL) blocking in TCP, and how does HTTP/3 (QUIC) solve it using UDP?

- When would you deliberately choose UDP over TCP in a production backend architecture?

- What are the header sizes of TCP and UDP, and what does the overhead difference imply for throughput?

## Strong Answers / Talking Points

### 1. Structural & Operational Comparison

|**Feature**|**TCP (Transmission Control Protocol)**|**UDP (User Datagram Protocol)**|
|---|---|---|
|**Connection State**|Stateful; connection established before data transfer.|Stateless; datagrams dispatched independently.|
|**Reliability**|Guaranteed delivery (retransmission via timeouts and duplicate ACKs).|Best-effort; dropped packets are lost unless handled at application layer.|
|**Ordering**|Strictly guaranteed via sequence numbers.|No ordering guarantee; packets may arrive out of sequence.|
|**Data Boundary**|**Byte stream** (data chunks merge/fragment arbitrarily).|**Datagram / Message** (explicit packet boundaries preserved).|
|**Header Overhead**|Minimum **20 bytes** (can reach up to 60 bytes with options).|Fixed **8 bytes** (Source Port, Dest Port, Length, Checksum).|
|**Flow / Congestion Control**|Yes (Sliding window, Slow Start, Congestion Avoidance, Fast Retransmit).|None. Transmits as fast as the application socket pushes data.|
|**Speed / Latency**|Higher latency (handshake RTTs + retransmission pauses).|Low latency / real-time (zero connection delay, zero retransmission delay).|

### 2. Deep Dive: Handshake, Teardown & Edge States

- **3-Way Handshake (`SYN` $\rightarrow$ `SYN-ACK` $\rightarrow$ `ACK`):**

    - Synchronizes Initial Sequence Numbers (ISNs) securely between client and server to prevent collision with stale segments from previous connections.

    - Adds a minimum of 1 full RTT overhead before application payload (HTTP/1.1 and HTTP/2) can transit.

- **Connection Teardown & `TIME_WAIT`:**

    - Active closer enters `TIME_WAIT` (typically 2 * Maximum Segment Lifetime, or 2MSL = 60–120 seconds).

    - _Why it matters:_ (1) Ensures the final `ACK` is reliably received by the peer (if lost, peer re-sends `FIN`), and (2) Prevents delayed/wandering packets from an old connection from corrupting a newly spawned connection using the identical 4-tuple (`src_ip, src_port, dst_ip, dst_port`).

### 3. The Head-of-Line (HoL) Blocking Problem & Modern UDP

> [!IMPORTANT] The HTTP/2 vs. HTTP/3 Interview Pivot
>
> An essential modern interview talking point: HTTP/2 multiplexed multiple streams over a **single TCP connection**, meaning a single dropped packet stalls **all** multiplexed streams until TCP retransmits it (Transport-level Head-of-Line blocking).
>
> **HTTP/3 solves this by switching to QUIC over UDP**: QUIC implements its own independent stream-level packet loss recovery in user space. If packet drops occur in Stream A, Stream B continues processing without stalling.


### 4. Real-World Decision Matrix: When to Use What

- **Choose TCP when correctness and complete data integrity are non-negotiable:**

    - _Protocols:_ HTTP/1.1, HTTP/2, WebSockets, SSH, FTP, SMTP, Database wire protocols (PostgreSQL, MySQL).

    - _Use Cases:_ Financial transactions, file uploads/downloads, REST/GraphQL APIs, distributed database replication streams.

- **Choose UDP when timeliness beats completeness, or when building custom transport logic:**

    - _Protocols:_ DNS queries, DHCP, NTP, WebRTC (media channels), QUIC (HTTP/3), RTP/RTSP.

    - _Use Cases:_ Real-time multiplayer gaming state sync (latest player coordinates invalidate old dropped ones), live video/audio conferencing, metrics collection/telemetry (StatsD/syslog where logging agent failure must not block app threads).

## Code Snippets / Examples

```typescript
import dgram from "node:dgram";
import net from "node:net";

// ============================================================================
// 1. TCP Server (Stream-Oriented, Stateful, Requires Framing)
// ============================================================================
const tcpServer = net.createServer((socket) => {
  // Handshake already completed here; virtual stream connection established
  socket.on("data", (chunk: Buffer) => {
    // Note: chunk is arbitrary slice of byte stream, NOT guaranteed full message
    console.log(`TCP chunk received from ${socket.remoteAddress}:${socket.remotePort}`);
    socket.write("ACK: " + chunk.toString());
  });

  socket.on("close", () => {
    console.log("TCP connection cleanly closed via FIN/ACK handshake");
  });
});
tcpServer.listen(8080);

// ============================================================================
// 2. UDP Socket (Datagram-Oriented, Stateless, Fire-and-Forget)
// ============================================================================
const udpSocket = dgram.createSocket("udp4");

udpSocket.on("message", (msg: Buffer, rinfo: dgram.RemoteInfo) => {
  // Discrete message received; no connection state, no handshake occurred
  console.log(`UDP datagram of ${msg.length} bytes from ${rinfo.address}:${rinfo.port}`);

  // Replying requires explicitly providing target address and port every time
  const response = Buffer.from("PONG");
  udpSocket.send(response, rinfo.port, rinfo.address);
});

udpSocket.bind(8081);
```

## Related Topics

- [[Network Protocols. Transport, Security & Application Layers|OSI Model vs TCP-IP Stack]]

- [[HTTP Fundamentals|HTTP Evolution (HTTP-1.1 vs HTTP-2 vs HTTP-3 / QUIC)]]

- [[Cross-Tab Communication in Modern Browsers. Mechanisms, Architecture & Trade-Offs|WebSockets vs WebRTC]]

- [[Network Protocols. Transport, Security & Application Layers|Socket Programming & Network I-O Multiplexing (epoll, kqueue)]]

- [[HTTP Fundamentals|Head-of-Line Blocking]]

## Tags

#fullstack #interview #networking #system-design #protocols #tcp #udp

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
