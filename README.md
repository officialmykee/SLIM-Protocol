# SLIM-Protocol

---

**SLIM — Super Light Instant Messaging Protocol**

**Core Philosophy**
SLIM is not general purpose. It is purpose built for social networks running on resource constrained servers. Every decision optimizes for one thing — doing more with less.

---

**The Four Commandments of SLIM**

- No string ever travels in a packet
- No connection ever re-identifies itself
- No byte exists without purpose
- No allocation happens at runtime

---

**Packet Structure**

Every SLIM packet looks like this:

```
[ 1 byte  ] Control byte
[ 2 bytes ] Channel ID (topic or recipient)
[ 2 bytes ] Payload length
[ N bytes ] Payload

Total overhead: 5 bytes. Always.
```

---

**The Control Byte — Unpacked**

```
┌─────────────────────────────────────┐
│  7   6   5   4  │  3    2    1    0 │
│  Message Type   │ DEL  PRI  DIR  RSV│
└─────────────────────────────────────┘

Bits 7-4 → Message type (16 possible types)
Bit  3   → DEL: Delivery confirmation needed
Bit  2   → PRI: Priority (0=normal 1=urgent)
Bit  1   → DIR: Direction (0=client→broker 1=broker→client)
Bit  0   → RSV: Reserved
```

---

**SLIM Message Types**

```
0001 → HELLO       (client initiates connection)
0010 → WELCOME     (broker confirms, assigns session)
0011 → PUBLISH     (client sends a message)
0100 → NOTIFY      (broker pushes to client)
0101 → SUBSCRIBE   (client joins a channel)
0110 → UNSUBSCRIBE (client leaves a channel)
0111 → PING        (keepalive from client)
1000 → PONG        (keepalive response from broker)
1001 → ACK         (delivery confirmation)
1010 → BYE         (clean disconnect)
1011 → ERROR       (something went wrong)
1100 → RESERVED
1101 → RESERVED
1110 → RESERVED
1111 → RESERVED
```

---

**The Handshake — Happens Once, Never Repeated**

```
Client → Broker:
┌──────────────────────────────────┐
│ HELLO control byte               │
│ 4 bytes → User ID                │
│ 1 byte  → Auth token length      │
│ N bytes → Auth token             │
└──────────────────────────────────┘

Broker → Client:
┌──────────────────────────────────┐
│ WELCOME control byte             │
│ 2 bytes → Assigned session ID    │
│ 1 byte  → Status (OK or REJECT)  │
└──────────────────────────────────┘
```

After WELCOME, the broker knows this connection completely. Nothing is ever sent again to identify the user.

---

**Channel ID System**

No string topics. Ever. Everything is a 2 byte unsigned integer giving 65,535 possible channels.

```
Channel 1   → Global feed
Channel 2   → Direct messages
Channel 3   → Notifications
Channel 4   → Friend activity
Channel 5   → Post interactions
Channel 6   → System alerts
Channel 7   → Live presence (online/offline)
Channel 8   → Story updates
...
Channel 100+ → User defined / future use
```

PHP knows these. Your broker knows these. Nothing is negotiated at runtime.

---

**Packet Examples**

**Publishing a message to channel 2 (DMs):**
```
Control  → 0011 0000  (PUBLISH, no confirm, normal, client→broker)
Channel  → 0x00 0x02  (channel 2)
Length   → 0x00 0x12  (18 bytes payload)
Payload  → [recipient_id: 2 bytes][message: 16 bytes]

Total packet: 23 bytes
```

**Broker pushing a notification:**
```
Control  → 0100 0110  (NOTIFY, no confirm, urgent, broker→client)
Channel  → 0x00 0x03  (channel 3, notifications)
Length   → 0x00 0x06  (6 bytes payload)
Payload  → [event_type: 1 byte][actor_id: 4 bytes][object_id: 1 byte]

Total packet: 11 bytes
```

**A keepalive ping:**
```
Control  → 0111 0000
Channel  → 0x00 0x00  (not applicable, zeroed)
Length   → 0x00 0x00  (no payload)

Total packet: 5 bytes
```

---

**Per Connection Memory Layout**

This is where SLIM really earns its name:

```
┌─────────────────────────────────┐
│     SLIM Connection Record      │
├─────────────────────────────────┤
│ 4 bytes → User ID               │
│ 2 bytes → Session ID            │
│ 4 bytes → Socket FD             │
│ 2 bytes → Channel subscriptions │
│           (bitmap, 16 channels) │
│ 2 bytes → State flags           │
│ 2 bytes → Last ping timestamp   │
├─────────────────────────────────┤
│ Total: 16 bytes per connection  │
└─────────────────────────────────┘
```

At 16 bytes per connection:
```
10,000 connections = 160KB of connection state
```

That is essentially nothing.

---

**Broker Internal Architecture**

```
┌─────────────────────────────────────────────┐
│                SLIM Broker                  │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │     Epoll Event Loop (1 thread)     │    │
│  └────────────────┬────────────────────┘    │
│                   ↓                         │
│  ┌─────────────────────────────────────┐    │
│  │   Packet Parser (zero allocation)   │    │
│  └────────────────┬────────────────────┘    │
│                   ↓                         │
│  ┌─────────────────────────────────────┐    │
│  │  Channel Router (array, O(1) lookup)│    │
│  └────────────────┬────────────────────┘    │
│                   ↓                         │
│  ┌─────────────────────────────────────┐    │
│  │  Delivery Engine (pointer passing,  │    │
│  │  zero copy)                         │    │
│  └─────────────────────────────────────┘    │
│                                             │
│  ┌──────────────┐  ┌─────────────────────┐  │
│  │ Memory Pool  │  │ Offline Queue       │  │
│  │ (pre-alloc   │  │ (append only log    │  │
│  │ at startup)  │  │ for missed msgs)    │  │
│  └──────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────┘
```

---

**Memory Budget On 512MB VPS**

```
┌──────────────────────────────────────────┐
│         SLIM Memory Budget               │
├──────────────────────────────────────────┤
│ Broker baseline code       →  2MB        │
│ Pre-allocated packet pool  →  8MB        │
│ 10,000 connection records  →  160KB      │
│ Channel routing tables     →  1MB        │
│ Offline message queue      →  5MB        │
│ Read/write socket buffers  →  10MB       │
│ OS and overhead            →  3MB        │
├──────────────────────────────────────────┤
│ Total SLIM usage           →  ~29MB      │
│ Remaining for your app     →  ~483MB     │
└──────────────────────────────────────────┘
```

Fits comfortably under your 30MB target.

---

**Deployment Architecture**

```
Internet
    ↓
[ Nginx ]  ←  handles TLS, HTTP, static files
    ↓
[ SLIM Broker ]  ←  pure C++, Unix socket
    ↑
[ PHP App ]  ←  talks to SLIM via Unix socket
    ↑
[ MySQL/DB ]  ←  normal database layer
```

PHP never handles real time connections directly. It just tells SLIM "send this to user 42" and SLIM handles the rest instantly.

---

**PHP To SLIM Communication**

PHP sends a raw binary packet to SLIM over a Unix socket:

```php
$socket = stream_socket_client('unix:///var/run/slim.sock');
$packet = pack('CnnN', 0x30, 3, 6, $userId) . $payload;
fwrite($socket, $packet);
```

No HTTP. No JSON. No overhead. Just bytes on a socket.

---

**Performance Targets SLIM Is Designed To Hit**

```
┌──────────────────────────────────────────────┐
│            SLIM Performance Targets          │
├──────────────────────────────────────────────┤
│ Concurrent connections  →  5,000 to 10,000   │
│ Broker RAM baseline     →  under 30MB        │
│ Message routing latency →  under 1ms         │
│ CPU at normal load      →  under 10%         │
│ Packet overhead         →  5 bytes always    │
│ Per connection cost     →  16 bytes RAM      │
│ Throughput              →  100,000 msg/sec   │
└──────────────────────────────────────────────┘
```

---

**What SLIM Deliberately Excludes**

```
✗ String topics          → numeric channel IDs only
✗ Built in TLS           → Nginx handles this
✗ QoS levels             → fire and forget or confirm, nothing more
✗ Wildcard subscriptions → explicit channel IDs only
✗ Clustering             → single broker, single VPS
✗ Dynamic allocation     → memory pool only
✗ Thread per connection  → epoll single thread only
```

Every exclusion is intentional. Every exclusion saves resources.

---

**What SLIM Gives You In Return**

```
✓ Predictable memory usage at any connection count
✓ Sub millisecond message delivery
✓ Runs comfortably alongside your PHP app
✓ Simple enough to fully understand and maintain
✓ Purpose built for exactly your social network use case
✓ Room to grow without changing the core protocol
```

---
