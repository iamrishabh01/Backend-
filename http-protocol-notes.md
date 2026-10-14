# Fundamentals of HTTP for Backend Engineers

Notes based on *"5. Understanding HTTP for backend engineers, where it all starts"* by Sriniously.

---

## 1. Core Principles of HTTP

### Statelessness
- **Definition:** HTTP has no memory of past interactions. Each request is completely self-contained and carries all information necessary for the server to process it (URI, method, headers, payload, authentication).
- **The Request-Response Cycle:** The server receives a request, executes processing, returns a response, and immediately terminates context of that interaction. Subsequent requests are treated as brand-new events.
- **Architectural Advantages:**
  - **Simplicity:** Servers do not require persistent in-memory session tables, connection tracking locks, or synchronization overhead.
  - **Horizontal Scalability:** Incoming traffic can be freely balanced across arbitrary backend instances without session affinity (sticky sessions).
  - **Fault Tolerance:** If an application instance fails or restarts, ongoing client flows are not broken because the client provides state on every request.
- **State Emulation:** Modern applications simulate stateful user experiences (e.g., shopping carts, authentication sessions) by passing persistent identifiers via cookies, authorization headers, or tokens (such as JWTs) on each request.

### Client-Server Model
- **Unidirectional Initiation:** HTTP follows a strict client-initiated model. The client (browser, mobile app, API caller) sends a request, and the server returns a response. The server cannot independently initiate communication in standard HTTP.
- **HTTP vs. HTTPS:** HTTPS is conceptually identical to HTTP at the application layer, but the transport stream is wrapped with TLS/SSL encryption and certificate-based identity verification.

---

## 2. Networking Context & Protocol Evolution

HTTP operates at **Layer 7 (Application Layer)** of the OSI model and relies on transport layer protocols to handle delivery and packet ordering.

```
+--------------------------------------------+
| OSI Layer 7: Application Layer (HTTP/HTTPS)|
+--------------------------------------------+
                     |
                     v
+--------------------------------------------+
| OSI Layer 4: Transport Layer (TCP / UDP)   |
+--------------------------------------------+
```

### Protocol Evolution Summary

| Version | Transport Protocol | Key Mechanism | Characteristics & Trade-offs |
| :--- | :--- | :--- | :--- |
| **HTTP/1.0** | TCP | One connection per request | Creates and tears down a full TCP connection (3-way handshake) for every single asset. Highly inefficient for multi-resource pages. |
| **HTTP/1.1** | TCP | Persistent connections (`keep-alive`) | Reuses an established TCP connection across multiple sequential requests. Introduced standard chunked transfer encoding and caching headers. |
| **HTTP/2.0** | TCP | Binary framing & multiplexing | Interleaves multiple streams over a single TCP connection. Adds HPACK header compression and server push. Limited by TCP-level head-of-line (HoL) blocking on packet loss. |
| **HTTP/3.0** | QUIC (over UDP) | Native multiplexed streams over UDP | Eliminates TCP head-of-line blocking. Enables 0-RTT connection resumption and resilient connection migration across IP networks. |

---

## 3. Anatomy of an HTTP Message

### Request Message Structure
```http
POST /api/v1/users HTTP/1.1
Host: api.example.com
User-Agent: Mozilla/5.0
Authorization: Bearer <token>
Content-Type: application/json
Content-Length: 48

{"name": "Alex", "email": "alex@example.com"}
```
1. **Request Line:** HTTP Method (`POST`), Resource URI (`/api/v1/users`), and Protocol Version (`HTTP/1.1`).
2. **Request Headers:** Metadata specifying client preferences, authentication, host, and body characteristics.
3. **Blank Line (`\r\n`):** A required empty line separating headers from the payload.
4. **Request Body:** Optional data payload sent to the server.

### Response Message Structure
```http
HTTP/1.1 200 OK
Date: Wed, 14 Oct 2026 10:10:00 GMT
Content-Type: application/json; charset=UTF-8
Content-Length: 35

{"status": "success", "id": 1024}
```
1. **Status Line:** Protocol Version (`HTTP/1.1`), 3-digit Status Code (`200`), and Reason Phrase (`OK`).
2. **Response Headers:** Metadata providing server context, date, content details, and caching instructions.
3. **Blank Line (`\r\n`):** Header-payload boundary.
4. **Response Body:** Data or error payload returned to the client.

---

## 4. HTTP Headers

Headers act like shipping labels on parcels: intermediaries and destination servers can inspect routing, encoding, and authorization directives without having to unpack or decode the message payload.

### Categories of Headers
- **Request Headers:** Provide client context and capabilities (`User-Agent`, `Authorization`, `Accept`).
- **General Headers:** Apply to both requests and responses (`Date`, `Connection`, `Cache-Control`).
- **Representation Headers:** Describe payload encoding and structure (`Content-Type`, `Content-Length`, `Content-Encoding`, `ETag`).
- **Security Headers:** Protect clients and servers against web application vulnerabilities:
  - `Strict-Transport-Security` (HSTS): Enforces HTTPS connections to prevent protocol downgrade attacks.
  - `Content-Security-Policy` (CSP): Whitelists valid resource origins to mitigate Cross-Site Scripting (XSS).
  - `X-Frame-Options`: Restricts embedding in `<frame>`, `<iframe>`, or `<object>` to prevent clickjacking.
  - `X-Content-Type-Options: nosniff`: Prevents MIME-type sniffing by browsers.
  - `Set-Cookie` Directives: Enforces `HttpOnly` (blocks JavaScript access) and `Secure` (HTTPS only transmission).

### Core Header Capabilities
1. **Extensibility:** Custom headers (such as `X-Request-ID` or `Traceparent`) can be added without altering the protocol specification.
2. **Remote Control Directives:** Headers allow peers to negotiate content formats (`Accept`), validate cache validity (`If-None-Match`), and manage connection persistence (`Connection: keep-alive`).

---

## 5. HTTP Methods & Idempotency

Methods define the semantic operation to be executed on a target resource.

### Method Comparison Matrix

| Method | Target Action | Payload / Body | Idempotent? | Safe? |
| :--- | :--- | :---: | :---: | :---: |
| **GET** | Retrieve resource state | No | **Yes** | **Yes** |
| **POST** | Create subordinate resource or process action | Yes | **No** | No |
| **PUT** | Replace target resource in its entirety | Yes | **Yes** | No |
| **PATCH** | Apply partial modifications / deltas | Yes | Usually No | No |
| **DELETE** | Remove the target resource | Optional | **Yes** | No |
| **OPTIONS** | Query communication options / allowed methods | No | **Yes** | **Yes** |

### Safety vs. Idempotency
- **Safe Methods (`GET`, `OPTIONS`, `HEAD`):** Read-only operations that do not modify backend server state.
- **Idempotent Methods (`GET`, `PUT`, `DELETE`):** Executing the identical request multiple times results in the exact same state on the server as executing it once.
  - *Example (`DELETE`):* Deleting `/items/10` once removes the item. Subsequent calls do not alter the database state; the item remains deleted.
- **Non-Idempotent Methods (`POST`):** Repeating the request produces additional side-effects (e.g., creating duplicate user or payment records).

---

## 6. Same-Origin Policy (SOP) & CORS

### Same-Origin Policy (SOP)
A fundamental browser security boundary preventing scripts running under one origin (`protocol://domain:port`) from reading data from another origin without explicit permission.

### Cross-Origin Resource Sharing (CORS)
CORS defines how servers can instruct browsers to permit cross-origin access.

#### 1. Simple Requests
- **Requirements:** Standard methods (`GET`, `POST`, `HEAD`), simple headers, and standard content types (`text/plain`, `multipart/form-data`, `application/x-www-form-urlencoded`).
- **Mechanism:** The browser sends an `Origin` header. If the server responds with a matching `Access-Control-Allow-Origin` header, the browser delivers the response to the script.

#### 2. Pre-flight Requests
Triggered whenever a request includes:
- Non-simple HTTP methods (`PUT`, `PATCH`, `DELETE`).
- Non-simple content types (notably **`application/json`**).
- Custom headers (such as `Authorization`).

```
Browser (Client)                                Backend Server
       |                                                |
       | ------ 1. OPTIONS (Pre-flight request) ------> |
       |        Access-Control-Request-Method: PUT      |
       |        Access-Control-Request-Headers: auth    |
       |                                                |
       | <----- 2. 204 No Content (Approval) ---------- |
       |        Access-Control-Allow-Origin: ...        |
       |        Access-Control-Allow-Methods: ...       |
       |        Access-Control-Max-Age: 86400           |
       |                                                |
       | ====== 3. Actual Request (PUT /users/1) =====> |
       |                                                |
       | <===== 4. Actual Response (200 OK) =========== |
```
- **`Access-Control-Max-Age`:** Specifies the duration (in seconds) the browser can cache the pre-flight response, reducing latency by avoiding redundant `OPTIONS` round trips.

---

## 7. HTTP Status Codes

### 1xx: Informational
- `100 Continue`: Initial headers accepted; client should proceed to send the request body.
- `101 Switching Protocols`: Server agrees to switch communication protocols (e.g., upgrading to WebSockets).

### 2xx: Success
- `200 OK`: Standard successful request.
- `201 Created`: Request succeeded and a new resource was created (recommended for `POST`).
- `204 No Content`: Request succeeded, but the response intentionally contains no body (common for `DELETE` or `OPTIONS`).

### 3xx: Redirection
- `301 Moved Permanently`: The resource has permanently shifted to a new URI.
- `302 Found`: Temporary redirection; client should continue using original URI for future calls.
- `304 Not Modified`: Cached representation is still fresh; saves network transfer by returning no payload.

### 4xx: Client Error
- `400 Bad Request`: Malformed syntax, invalid JSON, or failed input validation.
- `401 Unauthorized`: Missing or invalid authentication credentials.
- `403 Forbidden`: Authenticated identity lacks permission to access the resource.
- `404 Not Found`: Target URI does not exist on the server.
- `405 Method Not Allowed`: Target resource exists but does not support the requested HTTP method.
- `409 Conflict`: Request conflicts with current resource state (e.g., unique key violation).
- `429 Too Many Requests`: Client has exceeded rate limit thresholds.

### 5xx: Server Error
- `500 Internal Server Error`: Unhandled exception or unexpected server failure.
- `501 Not Implemented`: Server does not support functionality required to fulfill request.
- `502 Bad Gateway`: Reverse proxy or gateway received an invalid response from upstream backend.
- `503 Service Unavailable`: Server is temporarily overloaded or undergoing maintenance.
- `504 Gateway Timeout`: Reverse proxy did not receive a timely response from upstream application.

---

## 8. HTTP Caching Foundations

- **Objective:** Reuse previously fetched representations to reduce latency, save bandwidth, and minimize server workload.
- **`Cache-Control` Directives:**
  - `max-age=<seconds>`: Freshness lifetime of the cached response.
  - `no-cache`: Forces cache to revalidate with the server using conditional headers before reusing.
  - `no-store`: Prohibits storing the response anywhere (critical for private/sensitive transactions).
- **Validation with `ETag`:** The server returns an entity tag (`ETag`) representing resource state. On subsequent requests, the client passes `If-None-Match: <etag>`. If unchanged, the server responds with `304 Not Modified` and an empty body.
