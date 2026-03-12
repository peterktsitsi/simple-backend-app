# How the HTTP Demo Works — Detailed Explanation

---
*oh this took a while*<br>
![stressed](https://github.com/peterktsitsi/images/blob/main/xxx-nasty.gif)
## Overview

- The demo is a single HTML file that makes a **real, live HTTP GET request** to `https://httpbin.org/get` — a free public API specifically built for testing HTTP
- It visualises every step of the request/response cycle as it actually happens, with simulated delays added before the real network call to make each protocol layer visible
- Everything shown (status codes, headers, response body) is **real data** returned by the server — nothing is faked

---

## 1. The Target: `httpbin.org`

- `httpbin.org` is a public HTTP testing service — its entire purpose is to receive requests and echo back useful information
- The endpoint `/get` accepts `GET` requests and responds with a JSON object describing exactly what it received: your headers, your IP address, the URL you requested, etc.
- This makes it ideal for learning because the response **teaches you about your own request**

---

## 2. HTML Structure (the UI)

- **URL Bar** — shows the HTTP method (`GET`) and the target URL; contains the "Send Request" button that triggers everything
- **Network Diagram** — the 💻 → 🌐 visual with two animated "pipe" lines between browser and server; one line for the outgoing request (blue), one for the incoming response (green)
- **Request Panel** — displays the HTTP request headers that were sent (method, host, accept, user-agent, etc.)
- **Response Panel** — displays the HTTP response headers that came back (status code, content-type, server, round-trip time)
- **Event Timeline** — a live log of each network step with timestamps in milliseconds
- **Response Body** — the raw JSON payload returned by the server, pretty-printed

---

## 3. JavaScript: The `sendRequest()` Function

- This is the core function, called when you click the button
- It uses `async/await` so it can pause between steps without freezing the browser
- A `running` flag prevents you from firing multiple requests at once
- `performance.now()` is used to record the start time, so every timeline event can be stamped with `+Xms` (milliseconds since the button was clicked)

### Step-by-step inside `sendRequest()`:

- **Reset** — clears the timeline, panels, and response body back to their default placeholder states before each new request
- **DNS lookup event** — added immediately to the timeline; in reality the browser resolves `httpbin.org` to an IP address at this point (your OS checks its cache, then asks a DNS resolver)
- **`await sleep(200)`** — a 200ms artificial pause to make the DNS step visible before moving on
- **TCP connection event** — TCP is the transport protocol underneath HTTP; a 3-way handshake (SYN → SYN-ACK → ACK) establishes a reliable connection
- **`await sleep(150)`** — another pause
- **TLS handshake event** — because the URL is `https://`, TLS encryption is negotiated before any HTTP data is exchanged; the client and server agree on encryption keys
- **`await sleep(100)`** — another pause
- **"Sending HTTP GET request" event** — at this point `renderRequest()` is called to populate the Request Panel, and `animatePipe('req')` fires the blue dot animation across the pipe
- **`fetch(URL)`** — this is the actual real network call; JavaScript's built-in `fetch` API sends the HTTP request and returns a Promise that resolves when the server responds
- **`performance.now() - t0`** — measures the true round-trip time from when `fetch` was called to when the response headers arrived
- **Response events** — after `fetch` resolves, `renderResponse()` populates the Response Panel with the real headers, and `animatePipe('res')` fires the green dot animation
- **`res.json()`** — reads and parses the response body from a JSON string into a JavaScript object; this is a second async step because the body may arrive in chunks after the headers
- **`JSON.stringify(data, null, 2)`** — converts the parsed object back to a nicely indented string for display in the Response Body panel

---

## 4. The `sleep()` Helper

- `sleep(ms)` is defined as: `new Promise(r => setTimeout(r, ms))`
- It creates a Promise that resolves after `ms` milliseconds, so `await sleep(200)` simply pauses execution for 200ms
- This is purely for **educational pacing** — it lets each protocol step appear on screen before the next one begins, making the sequence readable
- The actual network call has no artificial delay; `fetch()` runs at real network speed

---

## 5. The Animated Pipe

- The pipe between the two nodes has two `<div>` elements styled as thin horizontal lines
- Each line contains a small absolutely-positioned circle (`.pipe-packet`) that is hidden by default (`opacity: 0`)
- When `animatePipe('req')` is called, a CSS class `sending` is added to the pipe container
- This triggers the `@keyframes packetRight` animation on the blue dot — it slides from `left: -12px` to `left: 100%` over 600ms, simulating a packet travelling from browser to server
- When `animatePipe('res')` is called, the class `receiving` is added instead, triggering `@keyframes packetLeft` on the green dot — the packet travels right to left (server → browser)
- The class is removed after 700ms via `setTimeout` so the animation can re-trigger on the next request

---

## 6. The Event Timeline

- `addEvent(msg, state)` creates a new `<div>` and appends it to the timeline list
- Each event starts with `opacity: 0; transform: translateX(-6px)` — invisible and slightly offset
- On the next animation frame (`requestAnimationFrame`), the class `visible` is added, which transitions it to `opacity: 1; transform: translateX(0)` — a smooth slide-in effect
- The coloured dot on each event (green = done, blue = active, red = error) is rendered via a CSS `::before` pseudo-element
- `scrollIntoView()` keeps the latest event in view as the list grows

---

## 7. What the Real HTTP Headers Mean

### Request headers shown:
- `GET /get HTTP/1.1` — the method, path, and protocol version
- `Host: httpbin.org` — tells the server which domain is being requested (required in HTTP/1.1)
- `Accept: application/json` — tells the server the client prefers JSON responses
- `Accept-Encoding: gzip, deflate` — tells the server it can compress the response to save bandwidth
- `Connection: keep-alive` — requests that the TCP connection stay open for possible future requests
- `User-Agent: Mozilla/5.0` — identifies the software making the request

### Response headers shown:
- `HTTP/1.1 200 OK` — the status line; `200` means the request succeeded
- `Content-Type: application/json` — tells the client the body is JSON
- `Server: httpbin` — identifies the server software
- `Access-Control-Allow-Origin: *` — a CORS header that allows any webpage to read this response
- `Round-trip time` — calculated by the demo itself, not a real HTTP header; shows how long the full request took

---

## 8. Error Handling

- The entire `fetch` + parsing block is wrapped in a `try/catch`
- If the network is offline, the URL is wrong, or CORS blocks the request, the catch block fires
- It adds a red error event to the timeline and displays the error message in the response body panel
- The button is always re-enabled in the final lines of `sendRequest()` regardless of success or failure

---

## 9. What's Simulated vs. What's Real

| Thing | Real or Simulated? |
|---|---|
| DNS lookup | **Simulated** (the event is shown, but JS can't observe DNS directly) |
| TCP connection | **Simulated** (happens inside the browser engine, not visible to JS) |
| TLS handshake | **Simulated** (same — handled by the browser automatically) |
| HTTP request headers | **Real** (these are the actual headers `fetch` sends) |
| Network packet animation | **Simulated** (CSS animation for visualisation) |
| HTTP response status + headers | **Real** (read directly from the `Response` object) |
| Response body (JSON) | **Real** (parsed directly from the server's response) |
| Round-trip time | **Real** (measured with `performance.now()`) |
| Timeline timestamps | **Real** (measured from button click, with artificial sleeps included) |
