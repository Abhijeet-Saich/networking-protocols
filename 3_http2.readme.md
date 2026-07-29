# HTTP/2 — Revision & Interview Notes

## 1. Why HTTP/2 Was Needed

HTTP/1.1 introduced **persistent connections**, allowing multiple HTTP requests to reuse the same TCP connection.

```text
HTTP/1.0

Request → TCP connection → Response → close
Request → TCP connection → Response → close


HTTP/1.1

        TCP connection
             │
Request ─────┼────→ Response
Request ─────┼────→ Response
Request ─────┼────→ Response
```

Persistent connections solved repeated connection establishment, but they did **not provide true multiplexing**.

### HTTP/1.1 Pipelining

Pipelining allowed:

```text
A →
B →
C →
```

without waiting for each response before sending the next request.

However, responses had to correspond in request order:

```text
← A
← B
← C
```

If A was slow while B and C were ready:

```text
A ──────────────────────────
B ─ ready ── WAITING ───────
C ─ ready ── WAITING ───────

Wire:

[A =========================][B][C]
```

This creates **HTTP/application-level head-of-line (HOL) blocking**.

Browsers historically worked around HTTP/1.1's limited concurrency by opening multiple TCP connections to the same origin.

```text
Browser
   │
   ├── TCP 1
   ├── TCP 2
   ├── TCP 3
   └── ...
```

HTTP/2 instead enables many concurrent HTTP exchanges to share a connection efficiently.

---

# 2. What Changed in HTTP/2?

HTTP semantics largely stayed the same:

* Methods: `GET`, `POST`, `PUT`, `DELETE`, etc.
* Status codes: `200`, `404`, `500`, etc.
* Headers
* Request/response bodies
* Cookies
* Authentication
* Caching
* Content negotiation

The major change was **how HTTP is represented and transported on the wire**.

```text
HTTP/1.1

HTTP semantics
      ↓
textual HTTP/1.1 messages
      ↓
TCP


HTTP/2

HTTP semantics
      ↓
Streams
      ↓
Binary Frames
      ↓
TCP
```

HTTP/2 therefore does not replace HTTP semantics. It introduces a new binary framing architecture underneath them.

---

# 3. HTTP/2 Is a Binary Framed Protocol

HTTP/1.1 uses textual syntax such as:

```http
GET /users HTTP/1.1
Host: api.example.com
Accept: application/json
```

HTTP/2 does not send requests using this textual HTTP/1.1 wire representation.

Communication is divided into **binary frames**.

A frame conceptually contains:

```text
┌───────────────────────────┐
│ Length                    │
│ Type                      │
│ Flags                     │
│ Stream ID                 │
├───────────────────────────┤
│ Payload                   │
└───────────────────────────┘
```

The HTTP/2 frame header is **9 bytes**.

The important idea is not simply that "binary is faster."

Binary framing provides structured boundaries and metadata that make features such as **streams and multiplexing** possible.

---

# 4. HEADERS and DATA Frames

Two important HTTP/2 frame types are:

```text
HEADERS
DATA
```

A request such as:

```http
POST /users
Content-Type: application/json

{"name":"AB"}
```

might conceptually become:

```text
Stream 1

HEADERS
├── :method: POST
├── :path: /users
├── :scheme: https
├── :authority: example.com
└── content-type: application/json

DATA
└── {"name":"AB"}
```

A response:

```text
HEADERS
├── :status: 200
└── content-type: application/json

DATA
└── {"id":42,"name":"AB"}
```

Large bodies can be split across multiple DATA frames:

```text
HEADERS S1
DATA S1
DATA S1
DATA S1
DATA S1
...
```

A DATA frame does **not** automatically mean the stream has ended.

`END_STREAM` can indicate that the sender has finished sending on that stream direction.

---

# 5. HTTP/2 Architecture

The central hierarchy is:

```text
HTTP/2 Connection
        │
        ├── Stream 1
        │      ├── Frames
        │      └── Frames
        │
        ├── Stream 3
        │      ├── Frames
        │      └── Frames
        │
        └── Stream 5
               └── Frames
```

Remember:

> **Connection → Streams → Frames**

---

# 6. Streams

A **stream** is an independent, bidirectional logical sequence of frames inside an HTTP/2 connection.

Typically, one request/response exchange happens on one stream.

```text
HTTP/2 Connection
│
├── Stream 1 → GET /users
├── Stream 3 → GET /orders
└── Stream 5 → GET /profile
```

These streams can all be active concurrently.

They are **not separate TCP connections**.

```text
Application

/users      /orders      /profile
   │           │            │
   ↓           ↓            ↓
Stream 1    Stream 3     Stream 5
      \        |        /
       \       |       /
        HTTP/2 framing
              │
              ↓
       ONE TCP connection
```

### Streams are logically concurrent

Don't describe HTTP/2 as giving each stream its own physical network channel.

Instead:

> Multiple streams can be active concurrently, while their frames are multiplexed onto the same TCP connection.

---

# 7. Stream IDs

Frames identify which stream they belong to using a **Stream ID**.

Example:

```text
Frame:
Type      = DATA
Stream ID = 3
Length    = 1000
```

The receiver therefore knows that those bytes belong to Stream 3.

This allows frames from different HTTP exchanges to be interleaved safely.

### Client-initiated streams

Client-initiated streams use odd IDs:

```text
1, 3, 5, 7, 9...
```

### Server-initiated streams

Server-initiated streams use even IDs:

```text
2, 4, 6, 8...
```

Historically, server-initiated streams were particularly relevant to HTTP/2 server push.

Server push has seen limited adoption and modern browsers have largely moved away from it.

### Stream 0

Stream ID `0` is reserved for **connection-level frames**.

It is not a normal request/response stream.

---

# 8. Multiplexing

Multiplexing is the core HTTP/2 performance feature.

> HTTP/2 multiplexing allows frames from multiple independent streams to be interleaved over one HTTP/2 connection.

Suppose:

```text
Stream 1 → 50 MB response
Stream 3 → 2 KB response
Stream 5 → 10 KB response
```

HTTP/1.1-style sequential transfer could look like:

```text
[S1 =================================][S3][S5]
```

HTTP/2 can interleave frames:

```text
[S1][S1][S3][S5][S1][S5][S1]...
```

Stream 3 can therefore finish while Stream 1 is still active.

The receiver uses Stream IDs to reconstruct:

```text
Stream 1 → frames belonging to S1
Stream 3 → frames belonging to S3
Stream 5 → frames belonging to S5
```

---

# 9. HTTP/1.1 vs HTTP/2 Multiplexing

### HTTP/1.1

Concurrency was often achieved using multiple TCP connections.

```text
Browser

├── TCP 1 → HTTP exchanges
├── TCP 2 → HTTP exchanges
├── TCP 3 → HTTP exchanges
└── ...
```

### HTTP/2

Many concurrent streams can share a connection.

```text
Browser

└── TCP connection
       │
       └── HTTP/2 connection
              │
              ├── Stream 1
              ├── Stream 3
              ├── Stream 5
              ├── Stream 7
              └── ...
```

HTTP/2 therefore reduces the need to obtain concurrency by opening many TCP connections.

This does **not** mean a browser is guaranteed to use exactly one TCP connection for an entire website in every real-world situation.

---

# 10. How HTTP/2 Solves HTTP/1.1 HOL Blocking

HTTP/1.1 pipelining:

```text
Requests:

A →
B →
C →

Responses:

← A
← B
← C
```

If A is slow:

```text
A ─────────────────────────
B ─ ready but waiting
C ─ ready but waiting
```

HTTP/2 gives each exchange an independent stream:

```text
A → Stream 1
B → Stream 3
C → Stream 5
```

Frames can be interleaved:

```text
[S1][S3][S5][S1][S3][S1]...
```

Therefore Stream 3 can complete before Stream 1.

HTTP/2 solves the **HTTP/1.1 application/protocol-level HOL problem** through:

```text
Streams
   +
Frames
   +
Multiplexing
```

---

# 11. HTTP/2 Still Has TCP HOL Blocking

This is one of the most important interview distinctions.

HTTP/2 runs over TCP:

```text
HTTP/2 Streams
       ↓
HTTP/2 Frames
       ↓
TCP byte stream
       ↓
IP
```

TCP provides an **ordered byte stream**.

Suppose HTTP/2 has frames belonging to:

```text
[S1][S3][S5][S1]
```

and some TCP data is lost:

```text
TCP:

A ✓
B ✗   ← missing
C ✓
D ✓
```

Even if later bytes contain data for Streams 3 or 5, TCP cannot simply expose the byte stream beyond the missing gap to HTTP/2.

It must recover the missing data first.

Therefore unrelated HTTP/2 streams can be delayed.

This is **TCP-level head-of-line blocking**.

### Key distinction

```text
HTTP/1.1

Application/protocol HOL
        +
TCP HOL
```

```text
HTTP/2

HTTP-level HOL greatly addressed by multiplexing
        +
TCP HOL still exists
```

This limitation was one motivation behind HTTP/3 using QUIC instead of TCP.

---

# 12. Frames Are NOT TCP Packets

Never assume:

```text
HTTP/2 frame = TCP packet
```

HTTP/2 operates above TCP.

```text
HTTP/2
   │
 Frames
   ↓
TCP
   │
byte stream
   ↓
IP
```

For example, HTTP/2 may create:

```text
[Frame A][Frame B][Frame C]
```

Transport segmentation could conceptually look like:

```text
TCP segment 1:
[Frame A][part of Frame B]

TCP segment 2:
[rest of Frame B][part of Frame C]

TCP segment 3:
[rest of Frame C]
```

HTTP/2 reconstructs its frame boundaries from the TCP byte stream.

TCP does not understand:

```text
HEADERS
DATA
Stream 1
Stream 3
```

Those are HTTP/2 concepts.

---

# 13. Streams vs TCP Connections

A stream is **not a socket or TCP connection**.

Wrong:

```text
Stream 1 → TCP 1
Stream 3 → TCP 2
Stream 5 → TCP 3
```

Correct:

```text
Stream 1 ─┐
Stream 3 ─┼── HTTP/2 framing ── ONE TCP connection
Stream 5 ─┘
```

Streams provide independence at the **HTTP layer**.

They do not provide independent TCP transport.

---

# 14. HTTP/1.1 vs HTTP/2 Architecture

### HTTP/1.1

```text
Application
     ↓
HTTP request/response
     ↓
Textual HTTP/1.1 representation
     ↓
TCP
```

Limited multiplexing capabilities led browsers to use multiple connections for concurrency.

### HTTP/2

```text
Application
     ↓
HTTP semantics
     ↓
Streams
     ↓
Frames
     ↓
Multiplexing
     ↓
TCP
```

The major architectural shift is:

> **HTTP/2 introduces streams and binary framing so many independent HTTP exchanges can be multiplexed over a shared connection.**

---

# 15. Production Consideration: HTTP/2 May End at a Proxy

Seeing HTTP/2 from a client doesn't necessarily mean your Node.js server receives HTTP/2.

For example:

```text
Browser
   │
 HTTP/2
   ↓
CDN / Load Balancer
   │
 HTTP/1.1
   ↓
Node.js
```

Or:

```text
Client
   │
  h2
   ↓
Reverse Proxy
   │
 HTTP/1.1
   ↓
Application
```

When debugging HTTP/2 behavior, identify which network hops actually use HTTP/2.

---

# 16. Interview Misconceptions

### “HTTP/2 sends requests in parallel.”

Better:

> HTTP/2 allows multiple streams to be active concurrently and multiplexes their frames over a shared connection.

---

### “Each HTTP/2 stream has a TCP connection.”

Wrong.

Many streams can share one TCP connection.

---

### “HTTP/2 completely eliminates HOL blocking.”

Wrong.

HTTP/2 addresses HTTP/1.1 application-level HOL blocking, but TCP-level HOL remains.

---

### “An HTTP/2 frame is a TCP packet.”

Wrong.

Frames belong to HTTP/2. TCP transports bytes and may segment them differently.

---

### “HTTP/2 changed GET, POST, status codes, etc.”

Mostly wrong.

HTTP semantics remain largely unchanged. HTTP/2 primarily changed the wire representation and connection architecture.

---

### “Binary means HTTP/2 converts JSON into a binary format.”

Wrong.

The HTTP protocol framing is binary. Your application body can still be JSON, HTML, images, Protobuf, etc.

---

# 17. Senior Interview Answers

### Why was HTTP/2 needed if HTTP/1.1 had persistent connections?

Persistent connections removed the need to establish a new TCP connection for every request, but HTTP/1.1 still lacked true multiplexing. Pipelining allowed multiple outstanding requests but response ordering caused application-level HOL blocking. HTTP/2 introduced streams and binary frames so multiple HTTP exchanges can make progress concurrently over a shared connection.

### What is an HTTP/2 stream?

An HTTP/2 stream is an independent, bidirectional logical sequence of frames within an HTTP/2 connection. A request and its response normally use the same stream.

### What is an HTTP/2 frame?

A frame is the basic protocol unit of HTTP/2 communication. Frames contain metadata such as their type, flags, length and Stream ID, allowing frames from different streams to be interleaved.

### What is HTTP/2 multiplexing?

Multiplexing is the ability to interleave frames belonging to multiple independent streams over a shared HTTP/2 connection.

### Why can Stream 3 finish before Stream 1?

HTTP/2 responses aren't required to complete according to request order. Frames identify their streams explicitly, so Stream 3's frames can be transmitted and completed while Stream 1 remains active.

### Does HTTP/2 solve HOL blocking?

It solves the HTTP/1.1 application/protocol-level HOL problem through multiplexing, but TCP HOL blocking remains because all streams share TCP's ordered byte stream.

---

# 18. Final Mental Model

```text
                 APPLICATION

       /users     /orders     /profile
          │          │           │
          ↓          ↓           ↓

      Stream 1   Stream 3    Stream 5
          │          │           │
          ↓          ↓           ↓

       HEADERS     HEADERS      HEADERS
       DATA        DATA         DATA
       DATA

          \           |           /
           \          |          /
            ───── Multiplex ─────

[S1][S3][S1][S5][S3][S1][S5]

                   │
                   ↓

             TCP byte stream

                   │
                   ↓

                Network
```

Remember:

> **Connection → Streams → Frames**

and:

> **Frames from multiple streams → interleaved → one shared connection**

and most importantly:

> **Streams are independent at the HTTP layer, but share the TCP transport underneath.**

---

# Topics To Continue Next Time

We intentionally stopped before covering:

* HPACK header compression
* Static and dynamic HPACK tables
* HTTP/2 flow control
* Stream-level flow control
* Connection-level flow control
* `WINDOW_UPDATE`
* Stream prioritization and practical caveats
* HTTP/2 connection reuse
* TLS + ALPN and HTTP/2 negotiation
* Does HTTP/2 require TLS?
* `SETTINGS`
* `RST_STREAM`
* `GOAWAY`
* Production debugging
* `curl --http2 -v`
* HTTP/2 with Node.js
* Final senior-backend interview round
