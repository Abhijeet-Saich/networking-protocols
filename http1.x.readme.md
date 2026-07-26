# HTTP/1.x — Backend Engineer Notes

## 1. What is HTTP?

HTTP is an **application-layer protocol** that defines how a client and server communicate using requests and responses.

A request:

```http
GET /users/42 HTTP/1.1
Host: example.com
Accept: application/json
```

A response:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id":42,"name":"John"}
```

HTTP defines **how these messages are structured and what they mean**.

For HTTP/1.1, communication traditionally looks like:

```text
Application
    ↓
HTTP
    ↓
TCP
    ↓
IP
```

HTTP and TCP have different responsibilities:

```text
HTTP → requests, responses, methods, headers, status codes

TCP  → reliable, ordered byte stream

IP   → routing data between machines
```

TCP does not understand `GET`, `POST`, headers, JSON, etc. It transports bytes.

---

# 2. HTTP Request Structure

A complete request can look like:

```http
POST /users HTTP/1.1
Host: api.example.com
Accept: application/json
Content-Type: application/json
Content-Length: 15

{"name":"Sam"}
```

Its structure is:

```text
Request Line
Headers
Blank Line
Optional Body
```

### Request Line

```http
POST /users HTTP/1.1
```

contains:

```text
POST       → HTTP method
/users     → request target
HTTP/1.1   → protocol version
```

The method describes the intended operation.

The target identifies the resource.

For:

```text
https://example.com/users/42?active=true
```

the request might contain:

```http
GET /users/42?active=true HTTP/1.1
Host: example.com
```

---

# 3. Important HTTP Headers

Headers are metadata:

```text
Header-Name: value
```

### Host

```http
Host: example.com
```

Identifies the hostname the request targets.

One server/IP can host several domains:

```text
              Server
                 │
       ┌─────────┼─────────┐
       │         │         │
 example.com  shop.com  api.com
```

`Host` lets the server determine which logical website/service should receive the request.

It is required in HTTP/1.1 requests.

---

## Accept vs Content-Type

This distinction is important.

```http
Accept: application/json
```

means:

> I want/can accept a JSON response.

While:

```http
Content-Type: application/json
```

means:

> The body in this HTTP message is JSON.

Example:

```http
POST /users HTTP/1.1
Content-Type: application/json
Accept: application/json

{"name":"John"}
```

Think:

```text
Content-Type → what I am sending

Accept       → what I want to receive
```

---

# 4. Blank Line and Body

HTTP headers are followed by a blank line:

```http
POST /users HTTP/1.1
Host: example.com
Content-Type: application/json

{"name":"John"}
```

The blank line marks:

```text
END OF HEADERS
```

HTTP/1.x uses CRLF line endings:

```text
\r\n
```

Therefore the end of the header section is represented by:

```text
\r\n\r\n
```

Anything after that can belong to the message body according to the applicable framing rules.

---

# 5. HTTP Response

A response follows a similar structure:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 25

{"id":42,"name":"John"}
```

Structure:

```text
Status Line
Headers
Blank Line
Optional Body
```

The status line:

```http
HTTP/1.1 200 OK
```

contains:

```text
HTTP/1.1 → protocol version
200      → status code
OK       → reason phrase
```

---

# 6. Why HTTP Needs Message Framing

This is an important HTTP/1 concept.

HTTP/1.1 commonly runs over TCP.

TCP provides a **continuous ordered stream of bytes**.

It doesn't inherently say:

```text
Response A ends here.
Response B begins here.
```

Imagine:

```text
TCP byte stream

████████████████████████████████████████
```

HTTP must determine how those bytes form messages:

```text
████████████ | ████████ | ███████████
 Response A    Response B   Response C
```

This is called **message framing**.

Two mechanisms we studied are:

```text
Content-Length
Transfer-Encoding: chunked
```

---

# 7. Content-Length

Example:

```http
Content-Length: 500
```

means:

> The body contains 500 bytes.

The receiver can:

```text
parse headers
     ↓
Content-Length: 500
     ↓
read 500 bytes
     ↓
body finished
```

Important: `Content-Length` represents **bytes**, not characters.

For example:

```js
const value = "😀";

value.length;
// 2

Buffer.byteLength(value);
// 4
```

HTTP cares about the transmitted byte length.

---

# 8. Chunked Transfer Encoding

Sometimes the server wants to start sending a response before it knows the final response size.

For example:

```js
res.write("part 1");

await getMoreData();

res.write("part 2");

await getMoreData();

res.write("part 3");

res.end();
```

The server cannot necessarily provide:

```http
Content-Length: ???
```

without first generating and buffering the entire response.

HTTP/1.1 can instead use:

```http
Transfer-Encoding: chunked
```

Conceptually:

```text
chunk
chunk
chunk
chunk
END
```

A simplified wire example:

```text
4\r\n
Wiki\r\n
5\r\n
pedia\r\n
0\r\n
\r\n
```

The numbers specify chunk sizes in **hexadecimal**.

The final zero-size chunk:

```text
0
```

signals that the chunked body has finished.

### Why chunked encoding?

It allows the sender to:

```text
generate data
     ↓
send chunk
     ↓
generate more data
     ↓
send chunk
     ↓
...
finish
```

without knowing the complete body size beforehand.

---

# 9. The Connection Problem

Imagine a webpage needs:

```text
index.html
style.css
app.js
logo.png
```

If every resource required a new connection:

```text
TCP connection
→ request HTML
→ response
→ close

TCP connection
→ request CSS
→ response
→ close

TCP connection
→ request JS
→ response
→ close
```

we repeatedly pay connection-establishment overhead.

This motivated connection reuse.

---

# 10. HTTP/1.1 Persistent Connections

HTTP/1.1 makes persistent connections the default.

Instead of:

```text
TCP → request → response → close

TCP → request → response → close
```

we can have:

```text
ONE TCP CONNECTION

Request A ─────>
          <──── Response A

Request B ─────>
          <──── Response B

Request C ─────>
          <──── Response C
```

The TCP connection is reused for multiple HTTP exchanges.

You may see:

```http
Connection: keep-alive
```

but persistence is already the default behavior in HTTP/1.1.

To explicitly signal closure:

```http
Connection: close
```

### Why framing matters here

If the connection remained open but HTTP had no way to determine body boundaries, the client wouldn't know when one response finished.

So concepts like:

```text
persistent connections
        +
message framing
```

are closely related.

---

# 11. HTTP/1.1 Pipelining

Persistent connections reduce connection setup cost, but requests may still be sequential:

```text
send A
   ↓
wait for A
   ↓
receive A
   ↓
send B
```

If network latency is significant, waiting before sending every next request wastes time.

HTTP/1.1 pipelining allows:

```text
Request A ─────>
Request B ─────>
Request C ─────>

          <──── Response A
          <──── Response B
          <──── Response C
```

The client can send B without waiting for A's response.

But there's an important limitation.

Responses must correspond to the request order.

Suppose:

```text
A → takes 5 seconds
B → takes 10 ms
C → takes 20 ms
```

Even if B and C are fast:

```text
A → processing....................

B → finished ✓
C → finished ✓
```

their responses cannot simply jump ahead of A's response on the pipelined connection.

This creates:

## Head-of-Line Blocking

A slow earlier response can delay later responses.

This is **HTTP/1.1 application/protocol-level HOL blocking**.

---

# 12. Multiple TCP Connections

Browsers historically worked around HTTP/1.1's limited parallelism by opening multiple TCP connections to the same origin.

Instead of:

```text
ONE CONNECTION

A → 5 sec
B → waiting
C → waiting
```

we can have:

```text
TCP #1 → A → 5 sec

TCP #2 → B → 10 ms

TCP #3 → C → 20 ms
```

Now B and C don't have to sit behind A on the same HTTP connection.

But this creates another tradeoff:

```text
more TCP connections
       ↓
more connection setup
more sockets
more client/server state
more resource usage
```

So multiple connections improve parallelism but introduce additional overhead.

---

# 13. Core Limitation of HTTP/1.1

What we would ideally like is:

```text
ONE CONNECTION

Request A ─────>
Request B ─────>
Request C ─────>

<──── part B
<──── part A
<──── part C
<──── more B
<──── more C
<──── more A
```

In other words:

> Multiple independent request/response streams progressing concurrently over one connection.

HTTP/1.1 does not provide this kind of **true multiplexing**.

This becomes one of the major motivations for HTTP/2.

---

# 14. HTTP HOL vs TCP HOL

These are different problems.

### HTTP/1.1 HOL

```text
Request A → slow
Request B → fast
Request C → fast
```

B/C can be delayed behind A because of HTTP response ordering.

This happens at the HTTP/protocol level.

### TCP HOL

TCP guarantees ordered byte delivery.

Suppose:

```text
Data 1 ✓
Data 2 ✗ lost
Data 3 ✓
```

TCP needs to recover the missing data before the application can receive the ordered stream past that gap.

So remember:

```text
HTTP HOL
→ HTTP response ordering

TCP HOL
→ ordered byte-stream delivery
```

---

# 15. Safe and Idempotent Methods

Two important HTTP method properties:

### Safe

The intended operation does not request modification of server state.

### Idempotent

Repeating the same operation has the same intended effect as performing it once.

For the methods we covered:

| Method | Safe | Idempotent |
| ------ | ---- | ---------- |
| GET    | Yes  | Yes        |
| POST   | No   | No         |
| PUT    | No   | Yes        |
| DELETE | No   | Yes        |

### GET

```http
GET /users/42
```

Reading repeatedly does not request state modification.

Safe + idempotent.

### POST

```http
POST /orders

{"productId":42}
```

Could create:

```text
request #1 → order 101
request #2 → order 102
```

Not inherently idempotent.

### PUT

```http
PUT /users/42

{"name":"John"}
```

Repeated requests:

```text
Alex → John
John → John
John → John
```

Changes state, so not safe.

But repeating it has the same intended effect, so idempotent.

### DELETE

```http
DELETE /users/42
```

Repeated:

```text
exists → doesn't exist

doesn't exist → doesn't exist
```

Not safe, but idempotent.

---

# 16. Idempotency Keys

Consider:

```text
Client                     Server

POST /payments ───────────>
                           charge ₹5,000
                           SUCCESS

       X <──────────────── response lost
```

The client receives a timeout.

Important:

```text
timeout ≠ operation failed
```

The payment may already have succeeded.

Blindly retrying the POST could create another payment.

A client can provide:

```http
POST /payments
Idempotency-Key: abc123
```

The server associates that key with the logical operation.

If the request is retried:

```http
POST /payments
Idempotency-Key: abc123
```

the server recognizes that it has already processed that operation and avoids creating another payment.

### Concurrency detail

Don't simply:

```text
check key
process payment
store key
```

because two concurrent requests could both observe the key as absent.

The idempotency key should be claimed/checked **atomically**, commonly using a database uniqueness constraint, transaction, or equivalent coordination mechanism.

---

# 17. HTTP Status Codes

Think in categories:

```text
1xx → informational
2xx → success
3xx → redirection
4xx → client/request problem
5xx → server problem
```

Useful codes:

```text
200 OK
201 Created
204 No Content

301 Moved Permanently
302 Found
304 Not Modified

400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
429 Too Many Requests

500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

---

## 401 vs 403

### 401

Authentication is missing or invalid.

```text
401 → Who are you?
```

### 403

The client is authenticated but doesn't have permission.

```text
403 → I know who you are,
      but you cannot do this.
```

Therefore:

```text
401 → Authentication
403 → Authorization
```

---

# 18. 500 vs 502 vs 503 vs 504

Imagine:

```text
Client
   ↓
Load Balancer / Nginx
   ↓
Node API
   ↓
Database / Service
```

### 500 — Internal Server Error

The server/application encountered an unexpected internal failure.

```text
Node API
   ↓
exception/error
   ↓
500
```

### 502 — Bad Gateway

A gateway/proxy failed to get a valid response from an upstream service.

```text
Client → Nginx → Node
                   X

Client ← 502 ← Nginx
```

### 503 — Service Unavailable

The service is temporarily unable to handle the request.

Examples:

```text
overloaded
maintenance
temporary capacity issue
```

### 504 — Gateway Timeout

The gateway waited too long for the upstream service.

```text
Client → Load Balancer → Node
              │
              │ waiting...
              │ waiting...
              │
            timeout
              ↓
             504
```

Remember:

```text
500 → internal application/server failure
502 → bad/failed upstream response
503 → temporarily unavailable
504 → upstream took too long
```

---

# 19. curl Commands

Inspect HTTP/1.1:

```bash
curl --http1.1 -v http://example.com/
```

Verbose output:

```text
> → request sent
< → response received
* → curl/network information
```

Custom headers:

```bash
curl --http1.1 -v \
  -H "Accept: application/json" \
  http://example.com/
```

POST JSON:

```bash
curl --http1.1 -v \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"name":"Alex","age":25}' \
  https://httpbin.org/post
```

Use `--http1.1` because HTTPS servers may otherwise negotiate HTTP/2 with curl.

---

# 20. Final Mental Model

The important story is:

```text
HTTP/1.1 runs over TCP
        ↓
TCP provides an ordered byte stream
        ↓
HTTP needs message boundaries
        ↓
Content-Length / chunked encoding
        ↓
Creating TCP connections has cost
        ↓
HTTP/1.1 reuses connections
        ↓
Persistent connections
        ↓
Need better utilization
        ↓
Pipelining
        ↓
Response ordering creates HOL blocking
        ↓
Browsers use multiple TCP connections
        ↓
More parallelism
but more connection overhead
        ↓
Fundamental HTTP/1.1 limitation:
no true multiplexing
        ↓
Motivation for HTTP/2
```

## Interview Checklist

You should now be able to explain:

* HTTP vs TCP
* HTTP request and response structure
* `Host`
* `Accept` vs `Content-Type`
* `Content-Length`
* Why chunked encoding exists
* Persistent HTTP/1.1 connections
* HTTP pipelining
* HTTP/1.1 HOL blocking
* HTTP HOL vs TCP HOL
* Why browsers used multiple TCP connections
* Why HTTP/1.1 lacks true multiplexing
* Safe vs idempotent methods
* Why POST retries can be dangerous
* Idempotency keys
* `401` vs `403`
* `500`, `502`, `503`, `504`

The most important architectural chain to remember is:

**TCP connection cost → persistent connections → pipelining → HOL blocking → multiple connections → lack of multiplexing → HTTP/2.**
