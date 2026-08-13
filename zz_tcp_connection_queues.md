# TCP Server Internals — Revision Notes

## Core mental model

The most important distinction:

> **The listening socket is NOT the socket used to communicate with each client.**

A listening socket stays in `LISTEN` state and produces separate **child sockets** for individual clients.

```text
Client
  |
  | SYN
  v
Listening Socket (:8080)
  |
  v
Pending handshake state
(request_sock)
  |
  | final ACK
  v
Established Child Socket
  |
  v
Accept Queue
  |
  | accept()
  v
Application
```

---

## 1. `socket()`

Example:

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
```

### What it does

Creates a kernel-managed socket and returns a **file descriptor** referring to it.

Conceptually:

```text
File Descriptor Table

0 -> stdin
1 -> stdout
2 -> stderr
3 -> Socket
```

The socket initially has no server port and is not listening.

### `socket()` does NOT

- bind a port
- start listening
- create an established connection
- begin the TCP handshake

Think:

> `socket()` = **create the communication endpoint**

---

## 2. `bind()`

Example:

```c
bind(fd, 0.0.0.0:8080);
```

`bind()` associates the socket with a local address/port.

Conceptually:

```text
Socket

State = CLOSED
Local IP = 0.0.0.0
Port = 8080
```

It still isn't listening.

### If a SYN arrives before `listen()`

If there is no listening endpoint for that port, the host commonly responds with:

```text
SYN -> RST
```

The client typically sees:

```text
Connection refused
```

This differs from a silently dropped packet, where the client may retransmit and eventually experience a timeout.

---

## 3. `listen()`

Example:

```c
listen(fd, 128);
```

This turns the socket into a **passive listening endpoint**.

Conceptually:

```text
Before:

Socket
State = CLOSED

        |
        | listen()
        v

After:

Socket
State = LISTEN
```

The kernel now knows:

> "Incoming SYNs for this endpoint should be handled as connection attempts."

The listening socket is associated with structures for:

1. Pending/incomplete handshakes
2. Completed connections waiting for the application

The exact Linux implementation is more sophisticated than the simple "two FIFO queues" diagram.

---

# 4. SYN Queue / Pending Handshake Side

Suppose clients send:

```text
C1 -> SYN
C2 -> SYN
C3 -> SYN
```

Conceptually:

```text
Listening Socket
      |
      v
Pending handshake state

request_sock(C1)
request_sock(C2)
request_sock(C3)
```

A `request_sock` represents **temporary connection state** while the TCP handshake is incomplete.

It is NOT the full application-facing TCP socket.

The kernel can store information such as:

- client/server addresses
- ports
- TCP sequence information
- connection state
- retransmission/timer information

---

# 5. TCP Three-Way Handshake

For a client C1:

```text
Client                    Server

  |                         |
  | -------- SYN ---------> |
  |                         |
  | <------ SYN+ACK ------- |
  |                         |
  | -------- ACK ---------> |
  |                         |
```

Before the final ACK:

```text
request_sock(C1)
```

After the final ACK:

```text
established child socket(C1)
```

The listening socket itself remains in `LISTEN`.

---

# 6. Why `request_sock` exists

The kernel should not immediately allocate a full TCP connection for every SYN.

Imagine:

```text
100,000 SYNs
```

Many may never complete their handshakes.

Creating a full connection for every SYN would waste significant resources.

Instead:

```text
SYN
 |
 v
Small temporary state
(request_sock)
 |
 | final ACK
 v
Full child TCP socket
```

This allows the kernel to handle incomplete connections more efficiently.

---

# 7. ACKs Do NOT Have To Complete FIFO

Suppose:

```text
Pending:

C1
C2
C3
C4
```

Now the ACK order is:

```text
C3
C1
C2
C4
```

C3 can complete first.

Why?

The kernel does not need to literally scan:

```text
C1 -> C2 -> C3
```

for every packet.

TCP packet information allows the kernel to locate the matching connection state efficiently.

So:

```text
ACK(C3)
   |
   v
Find C3's connection state
   |
   v
Complete C3
```

C1 and C2 don't have to complete first.

### Important distinction

A diagram called a "queue" is a **conceptual model**.

The actual kernel implementation uses optimized data structures and lookup mechanisms.

---

# 8. Child Socket

After the final ACK arrives for C3:

Before:

```text
Pending:

C1
C2
C3
C4
C5
```

After:

```text
Pending:

C1
C2
C4
C5
```

And:

```text
Completed / Accept side:

C3
```

C3 now has a **full established child TCP socket**.

The listening socket is still:

```text
LISTEN
```

This is critical.

---

# 9. Listening Socket vs Child Socket

Think of:

```text
                    Listening Socket
                         :8080
                          |
                       LISTEN
                          |
              +-----------+-----------+
              |           |           |
              v           v           v
          Child C1    Child C2    Child C3
         ESTABLISHED ESTABLISHED ESTABLISHED
```

The listening socket:

> Accepts connection attempts.

A child socket:

> Communicates with one specific client.

Therefore:

```text
1 listening socket
+
10,000 clients
=
1 listening socket + potentially 10,000 child sockets
```

---

# 10. Accept Queue

Once a connection is established, the child socket can wait for the application.

Conceptually:

```text
Accept Queue

C3
C6
C8
```

These are already-established connections.

They are waiting because the application hasn't retrieved them yet.

---

# 11. `accept()`

Application:

```c
int client_fd = accept(listen_fd, ...);
```

The important thing:

> `accept()` does NOT turn the listening socket into a client connection.

Instead, it retrieves an established child connection and gives the application a file descriptor referring to that child socket.

Before:

```text
Listening Socket
      |
      +-- Accept Queue
             |
             +-- C3
             +-- C6
             +-- C8
```

After:

```text
accept()
   |
   v
client_fd -> Child Socket(C3)
```

The listening socket remains:

```text
LISTEN
```

---

# 12. Complete Lifecycle

Memorize this:

```text
socket()
   |
   v
Socket created
   |
bind()
   |
   v
Local IP/port assigned
   |
listen()
   |
   v
LISTEN
   |
   | SYN
   v
Temporary handshake state
(request_sock)
   |
   | SYN+ACK
   |
   | final ACK
   v
Established child socket
   |
   v
Accept Queue
   |
accept()
   |
   v
Application receives child socket FD
```

---

# 13. Worked Example

Initial:

```text
Listening sockets = 1

Pending request_sock:
C1
C2
C3
C4
C5

Established child sockets = 0

Accept Queue = empty
```

C3 sends the final ACK first.

Result:

```text
Listening sockets = 1

Pending request_sock:
C1
C2
C4
C5

Established child sockets:
C3

Accept Queue:
C3
```

The listening socket remains:

```text
LISTEN
```

If C6 now sends only a SYN:

```text
Pending:
C1
C2
C4
C5
C6
```

C6 does **not** immediately become an established child socket.

It must complete the handshake first.

---

# 14. `listen(fd, backlog)` and backlog

Example:

```c
listen(fd, 128);
```

Do NOT simplify this to:

> "128 is the size of the SYN queue."

That's not an accurate Linux mental model.

The backlog is primarily associated with the number of completed connections waiting to be accepted, while pending handshake capacity is governed by additional kernel behavior/settings such as `tcp_max_syn_backlog`.

The exact behavior depends on the OS/kernel version and configuration.

---

# 15. Production Problems

## Slow application / accept queue pressure

If the application doesn't accept connections quickly enough:

```text
Established connections
        |
        v
Accept Queue
        |
        | fills
        v
Pressure / limits
```

This can cause connection failures or increased latency depending on the circumstances.

---

## SYN floods

An attacker can send huge numbers of SYNs without completing handshakes.

Conceptually:

```text
SYN
SYN
SYN
SYN
SYN
...
```

The kernel must protect resources used for pending connections.

Linux has mechanisms such as **SYN cookies** to help defend against SYN-flood scenarios.

---

## Connection refused vs timeout

### Refused

Usually an active response such as:

```text
RST
```

The client often immediately gets:

```text
Connection refused
```

### Timeout

Often means packets are being silently dropped somewhere:

```text
Client
  |
  | SYN
  v
Firewall / network
  X
```

The client retransmits and eventually times out.

---

# 16. Node.js Mental Model

For:

```js
const net = require("net");

const server = net.createServer((socket) => {
    console.log("client connected");
});

server.listen(8080);
```

Conceptually:

```text
server.listen(8080)
        |
        v
Listening endpoint
        |
        v
Linux TCP stack
        |
        | SYN
        v
Pending handshake state
        |
        | final ACK
        v
Established child socket
        |
        v
accept()
        |
        v
Node/libuv
        |
        v
JavaScript socket object
        |
        v
"connection" callback
```

Node.js hides most of this complexity.

The exact internal implementation depends on the Node.js/libuv/Linux versions, so this diagram is an architectural mental model rather than a literal source-code trace.

---

# 17. The Most Important Insights

### Insight 1

**A listening socket is not a connected socket.**

### Insight 2

**One listening socket can create many child sockets.**

### Insight 3

**A SYN does not immediately create a full established socket.**

### Insight 4

**`request_sock` represents temporary handshake state.**

### Insight 5

**The final ACK allows the connection to become established and a child socket to become available to the accept side.**

### Insight 6

**`accept()` retrieves a child socket; it does not modify the listening socket into a connected socket.**

### Insight 7

**The conceptual "SYN queue" should not be imagined as a simple FIFO array that must be scanned from the front for every ACK.**

---

# 18. Rapid-Fire Self Test

Try answering these without looking above.

1. What does `socket()` create?
2. What does `bind()` do?
3. What changes after `listen()`?
4. Are the connection queues global or associated with a listening socket?
5. What is `request_sock`?
6. Does a SYN immediately create an established socket?
7. What happens after the final ACK?
8. Does the listening socket become `ESTABLISHED`?
9. What does `accept()` return?
10. If C3's ACK arrives before C1's ACK, can C3 complete first?
11. Can one listening socket have thousands of child sockets?
12. Why isn't `listen(fd, 128)` simply "SYN queue size = 128"?

---

# One-Line Memory Hook

> **Create → Name → Listen → Temporary handshake → Establish child → Accept child**

```text
socket()
   ↓
bind()
   ↓
listen()
   ↓
SYN
   ↓
request_sock
   ↓
final ACK
   ↓
child socket
   ↓
accept queue
   ↓
accept()
   ↓
application
```

---

## Next Topic

The natural next step is to trace:

```text
Client SYN
    ↓
Linux NIC
    ↓
network stack
    ↓
TCP processing
    ↓
listening socket lookup
    ↓
request_sock
    ↓
SYN+ACK
    ↓
final ACK
    ↓
child socket
    ↓
accept queue
    ↓
libuv
    ↓
Node.js
    ↓
server.on("connection")
```

That will connect **TCP theory → Linux kernel → Node.js internals** into one complete picture.
