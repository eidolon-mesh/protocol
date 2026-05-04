# **Vellum Protocol Specification (Wisp v1 Extended)**

**Version:** 1.1 (Namespace Isolated)

**Base Protocol:** Wisp v1 (WebSocket over TCP)

**Purpose:** A unified multiplexing protocol that handles standard proxy streams (TCP/UDP) alongside native Zero-Trust Layer 3 mesh routing.

## **1\. Packet Structure Overview**

All Vellum packets follow the standard Wisp framing format to ensure the Aether Edge Router can parse them with zero-copy efficiency.

Every packet begins with a **1-byte Packet Type** and a **4-byte unsigned integer Stream ID** (Little-Endian).

\[TYPE (1 Byte)\] \[STREAM\_ID (4 Bytes)\] \[PAYLOAD (...)\]

*Note: For connectionless Vellum extensions (like mesh Datagrams), the STREAM\_ID is set to 0x00000000.*

## **2\. Standard Wisp v1 Packets (The Proxy Plane: 0x01 \- 0x0F)**

These packets are fully compatible with standard Wisp v1 clients and are used when a client wants to browse the normal internet via Aether as a proxy.

### **0x01 CONNECT**

Initiates a new TCP or UDP stream to a remote host.

* **Format:** \[0x01\] \[STREAM\_ID (4)\] \[STREAM\_TYPE (1)\] \[PORT (2)\] \[HOSTNAME (UTF-8)\]  
* **Stream Type:** 0x01 (TCP), 0x02 (UDP)

### **0x02 DATA**

Sends a chunk of data over an established stream.

* **Format:** \[0x02\] \[STREAM\_ID (4)\] \[RAW\_BYTES\]

### **0x03 CONTINUE**

Flow control. Tells the sender they are allowed to send more DATA packets.

* **Format:** \[0x03\] \[STREAM\_ID (4)\] \[BUFFER\_REMAINING (4)\]

### **0x04 CLOSE**

Tears down an active stream.

* **Format:** \[0x04\] \[STREAM\_ID (4)\] \[REASON\_CODE (1)\]  
* **Reason Codes:** 0x01 (Normal), 0x02 (Connection Refused), 0x03 (Timeout).

## **3\. Vellum Extended Packets (The Mesh Plane: 0x10 \- 0x1F)**

These packets are custom-built for Project Eidolon. By reserving the 0x1x namespace, we guarantee zero collisions if the upstream Wisp protocol ever adds 0x05 or 0x06.

### **0x10 INFO (The Negotiation Plane)**

Sent immediately upon WebSocket connection to negotiate Eidolon-specific capabilities. Standard Wisp clients will not send this, allowing Aether to instantly identify advanced clients.

* **Format:** \[0x10\] \[STREAM\_ID=0\] \[EXTENSIONS\_JSON\]

### **0x11 SIGNAL (The Control Plane)**

Used for network orchestration between Shard (Client) and Aether (Server), bypassing the proxy flow.

* **Format:** \[0x11\] \[STREAM\_ID=0\] \[SIGNAL\_TYPE (1)\] \[JSON\_PAYLOAD\]  
* **Signal Types:**  
  * 0x01 (IP\_ASSIGN): Server assigns the client its fd00:: IPv6 address.  
  * 0x02 (ICE\_CANDIDATE): WebRTC SDP passing for future local P2P.

### **0x12 DATAGRAM (The Data Plane)**

The core workhorse of the Eidolon mesh. Carries fully End-to-End Encrypted (E2EE) raw IP packets between two mesh nodes.

* **Format:** \[0x12\] \[STREAM\_ID=0\] \[DEST\_IPV6 (16)\] \[ENCRYPTED\_PAYLOAD\]  
* **Routing Logic:** Aether reads the 16-byte Destination IPv6 address. It looks up the associated client in its Arc\<RwLock\<HashMap\>\> and directly dumps the packet into the destination's WebSocket.

### **0x13 HANDSHAKE (The Cryptographic Plane)**

Facilitates the Diffie-Hellman Key Exchange (X25519) between two nodes *before* they can send 0x12 DATAGRAM traffic.

* **Format:** \[0x13\] \[STREAM\_ID=0\] \[DEST\_IPV6 (16)\] \[X25519\_PUBLIC\_KEY (32)\]

## **4\. Example: The E2EE Mesh Handshake Flow**

1. **Client A** constructs a 0x13 HANDSHAKE packet: \[0x13\] \[00 00 00 00\] \[fd 00 ... B\] \[A\_PUBLIC\_KEY\]  
2. **Client A** sends this to **Aether** over WSS.  
3. **Aether** reads 0x13, slices out the destination IP (fd00::B), looks up Client B's WSS channel, and forwards the packet exactly as-is.  
4. **Client B** receives the 0x13, calculates the shared secret, and sends a 0x13 back through Aether to Client A.  
5. Keys are negotiated. **Client A** encrypts an ICMP Echo Request and sends it via 0x12 DATAGRAM.