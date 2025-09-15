# Streaming Delivery Patterns over HTTP

This document summarizes practical server→client streaming options for web applications, focusing on long‑lived delivery of incremental events or updates. It captures the previously outlined 1–15 points (expanded slightly for standalone clarity).

## 1. Core Options Overview

| Pattern | Direction | Framing | Browser Support | Typical Use Cases | Complexity |
|---------|-----------|---------|-----------------|-------------------|------------|
| Chunked Streaming GET (NDJSON) | Server → Client | Arbitrary (newline JSON) | Universal (fetch/XHR) | Change feeds, logs, progressive results | Low/Moderate |
| Server-Sent Events (SSE) | Server → Client | Text lines (`data:`) | Native `EventSource` | Notifications, progress, low-frequency events | Low |
| WebSockets | Full Duplex | Binary/Text frames | Native `WebSocket` | Chat, collaborative editing, bidirectional RPC | Moderate |
| gRPC Streaming | Uni/Bidirectional | Protobuf frames over HTTP/2 | Browsers (grpc-web variant) | Strongly typed APIs, microservices | Moderate/High |

## 2. When to Choose Which
- **Chunked GET**: One-way stream, simplest infra (no upgrades), custom framing OK, easy to fallback to polling.
- **SSE**: One-way, need auto-reconnect & simple browser API with minimal code.
- **WebSockets**: Need client → server interaction or high-frequency low-latency messages.
- **gRPC Streaming**: Typed contracts across polyglot services or needing bidi semantics plus tooling.

## 3. Chunked Streaming GET (NDJSON Pattern)
A long-lived HTTP GET using `Transfer-Encoding: chunked` (HTTP/1.1) or data frames (HTTP/2). Server flushes each event immediately.

Example event framing (newline delimited JSON / NDJSON):
```
{"type":"update","seq":1,"data":{...}}
{"type":"update","seq":2,"data":{...}}
```

Key traits:
- Plain HTTP; plays nicely with proxies and existing auth.
- Client must explicitly stream-read (no full buffering).
- Heartbeats (e.g., `{"type":"heartbeat"}` or just `\n`) prevent idle timeouts.

## 4. Server-Sent Events (SSE)
SSE uses a text/event-stream content type and `EventSource` in browsers.
```
data: {"type":"notice","message":"Build complete"}

```
Features:
- Built-in automatic retry with `Last-Event-ID` support.
- Only server→client text; no binary.
- Minimal code but custom metadata fields limited (define your own JSON inside `data:`).

## 5. WebSockets
Full duplex, persistent connection after Upgrade handshake.
- Frames can be text or binary.
- Good for interactive sessions (collab editing, presence, chat).
- Requires explicit heartbeat/ping for some infrastructures.
- More likely to be blocked by restrictive proxies than plain GET.

## 6. Backpressure & Flow Control
| Mechanism | Backpressure Strategy |
|-----------|-----------------------|
| Chunked GET | Reader must drain quickly; otherwise server blocked or connection reset |
| SSE | Similar to chunked GET (text stream) |
| WebSockets | Native framing + TCP flow control; application-level queue still needed |
| gRPC Streaming | Flow control windows + application-level buffering |

## 7. Heartbeats / Keep-Alive
Problem: Proxies close seemingly idle connections.
- **Chunked/SSE**: Send lightweight heartbeat every 15–30s.
- **WebSockets**: Use ping/pong frames (library-provided) or application-level JSON heartbeat.
- **Timeout detection**: Clients track last event timestamp; reconnect if stale.

## 8. Reconnection Strategies
| Strategy | Chunked GET | SSE | WebSocket |
|----------|-------------|-----|-----------|
| Auto retry | Manual | Built-in (`EventSource`) | Manual |
| Resume position | Custom `?since=ID` param | `Last-Event-ID` header, server may replay | Custom protocol (send last seq after open) |
| Exponential backoff | Recommended | Polyfilled if customizing | Recommended |

## 9. Framing Choices (Chunked Streaming)
- **NDJSON** (newline-delimited JSON): Easiest to parse.
- **Length-Prefixed JSON**: Write `<len>\n<json>` for robustness against embedded newlines.
- **JSON Array with commas**: Harder; incomplete final JSON until close.
- **Binary**: Base64 in JSON or switch to WebSocket/gRPC.

## 10. Example Implementations
### Server (Go)
```go
func stream(w http.ResponseWriter, r *http.Request) {
  w.Header().Set("Content-Type", "application/json")
  flusher, _ := w.(http.Flusher)
  enc := json.NewEncoder(w)
  ctx := r.Context()
  for ev := range events { // events: <-chan interface{}
    if ctx.Err() != nil { return }
    enc.Encode(ev)
    flusher.Flush()
  }
}
```
### Client (Browser Fetch)
```js
const resp = await fetch('/stream');
const reader = resp.body.getReader();
let buf = '', dec = new TextDecoder();
while (true) {
  const {value, done} = await reader.read();
  if (done) break;
  buf += dec.decode(value, {stream:true});
  const lines = buf.split('\n');
  buf = lines.pop();
  for (const line of lines) if (line.trim()) handle(JSON.parse(line));
}
```

## 11. Feature Comparison Snapshot
| Feature | Chunked GET | SSE | WebSocket |
|---------|-------------|-----|-----------|
| Direction | 1-way | 1-way | 2-way |
| Binary | No (unless encoded) | No | Yes |
| Auto-reconnect | Manual | Yes | Manual |
| Backpressure detail | Minimal | Minimal | Better |
| Event ordering | Sequential | Sequential | Depends on app |
| Proxy friendly | High | High | Medium |
| Simplicity | High | High | Medium |

## 12. Performance Considerations
- Coalesce small events if overhead dominates (e.g., batch every 50ms).  
- Avoid huge JSON objects—stream incremental updates instead of snapshots.  
- For high fan-out, consider broker (Redis pub/sub, NATS) behind your HTTP fan-out layer.

## 13. Observability & Metrics
Track per connection:
- Bytes sent, events sent
- Active streams gauge
- Average events per second
- Reconnect rate
- Latency from event creation to flush (server side)

## 14. Security & Auth
- Use the same auth (Bearer tokens, cookies) as other GET endpoints.
- Ensure tokens outlive stream; refresh by reconnect.
- Sanitize/validate events at the boundary; treat stream as a public API surface.

## 15. Choosing Quickly (Decision Cheat Sheet)
| Need | Pick |
|------|------|
| One-way, minimal setup, custom JSON framing | Chunked GET |
| One-way, browser with built-in reconnect | SSE |
| Bidirectional interaction | WebSocket |
| Strong typing + bidi + microservices | gRPC streaming |
| Legacy infra/proxy skepticism | Chunked GET or SSE |

---
**Summary**: Use chunked streaming GET or SSE by default for simple push; escalate to WebSockets only when you truly need client → server messaging or binary efficiency. gRPC streaming adds typed contracts when you control both ends.


## 16. Scenario: Kafka → Go Microservice → Browser (Server-Sent Events)

Assumptions:
* Source of truth events arrive on Kafka topic(s) (e.g. `notifications`) with at-least-once delivery.
* Go microservice (in Kubernetes on AWS) consumes as a single consumer group with N goroutines (one per assigned partition).
* Each browser session must receive only its authorized subset (user or tenant scoping) with low latency (<250ms typical end-to-end target).
* Ingress layer: nginx ingress controller or ALB → nginx sidecar → service pod (optional layering). We focus on nginx ingress annotations.

### 16.1 Architecture Flow
Kafka → Go consumer group → in-process dispatcher (fan-out hub) → SSE HTTP handler (`/notifications/sse`) → Browser `EventSource`.

### 16.2 Event Envelope
Canonical JSON (prior to SSE framing):
```json
{
  "id": "topic:partition:offset",  
  "ts": 1737000123456,
  "type": "notification",
  "userId": "u123",
  "data": { "title": "Build complete", "severity": "info" }
}
```
SSE frame emitted as lines (id is optional but recommended):
```
id: topic:3:10452
data: {"id":"topic:3:10452","ts":1737000123456,"type":"notification","userId":"u123","data":{"title":"Build complete","severity":"info"}}

```

### 16.3 Go Consumer & Dispatcher Sketch
```go
type Event struct { ID, UserID, Type string; TS int64; Data any }

// Per session queue (bounded)
type Session struct { ch chan *Event; lastSeen string }

var sessions sync.Map // key sessionID -> *Session

func kafkaLoop(msgs <-chan kafkaMsg) {
  for m := range msgs {
    ev := normalize(m)
    // Single pass filter + fan-out
    sessions.Range(func(_, v any) bool {
      s := v.(*Session)
      if authorized(ev, s) {
        select { case s.ch <- ev: default: handleOverflow(s) }
      }
      return true
    })
  }
}
```

Overflow policy options: drop oldest (ring), drop newest, or disconnect client. Document chosen behavior (e.g. disconnect to maintain freshness guarantees).

### 16.4 SSE HTTP Handler (Go)
```go
func sseHandler(w http.ResponseWriter, r *http.Request) {
  user := authenticate(r)
  if user == nil { http.Error(w, "unauth", http.StatusUnauthorized); return }
  w.Header().Set("Content-Type", "text/event-stream")
  w.Header().Set("Cache-Control", "no-cache")
  w.Header().Set("Connection", "keep-alive")
  w.Header().Set("X-Accel-Buffering", "no") // nginx buffering off
  flusher, _ := w.(http.Flusher)

  lastID := r.Header.Get("Last-Event-ID")
  sess := registerSession(user.ID, lastID) // may trigger small replay from buffer
  defer unregisterSession(sess)

  ctx := r.Context()
  heartbeat := time.NewTicker(20 * time.Second)
  defer heartbeat.Stop()
  for {
    select {
    case <-ctx.Done(): return
    case <-heartbeat.C:
      io.WriteString(w, ":keep-alive\n\n")
      flusher.Flush()
    case ev := <-sess.ch:
      // Write id (optional but enables Last-Event-ID)
      io.WriteString(w, "id: "+ev.ID+"\n")
      b, _ := json.Marshal(ev)
      io.WriteString(w, "data: "+string(b)+"\n\n")
      flusher.Flush()
    }
  }
}
```

### 16.5 Replay Strategy (Last-Event-ID)
Maintain an in-memory ring buffer (e.g., slice of last 500 events) keyed by ID. On connect with `Last-Event-ID`, scan forward until head; if missing (gap due to eviction) start live only (document potential missed notifications and offer manual refresh endpoint if critical).

### 16.6 Browser Client (SSE)
```js
const es = new EventSource('/notifications/sse');
const seen = new Set();
es.onmessage = e => {
  const ev = JSON.parse(e.data);
  if (seen.has(ev.id)) return; // dedupe (rare if at-least-once dup)
  seen.add(ev.id);
  display(ev);
};
es.onerror = () => {
  // Let built-in retry happen; optionally show a disconnected indicator after a grace period
};
```

### 16.7 Ingress / nginx Settings (SSE)
Add annotations:
```
nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
nginx.ingress.kubernetes.io/configuration-snippet: |
  proxy_buffering off;
  proxy_cache off;
  add_header X-Accel-Buffering "no";
```
If using ALB: increase idle timeout: `alb.ingress.kubernetes.io/load-balancer-attributes: idle_timeout.timeout_seconds=300`.

### 16.8 Metrics & Observability (SSE)
* notifications_sse_active_connections
* notifications_events_sent_total{type}
* notifications_sse_heartbeats_total
* notifications_sse_disconnects_total{reason}
* kafka_consumer_lag (per partition)
Tracing span: (Kafka receive) → (enqueue) → (flush write) with attributes: event.id, user.count.

### 16.9 Pros / Cons (SSE Scenario)
Pros: Minimal client code, native retries, simple framing, easy replay with Last-Event-ID.  
Cons: Text-only, one-way, custom metadata fields limited to JSON payload, replay buffer complexity if large gaps.

---

## 17. Scenario: Kafka → Go Microservice → Browser (Chunked NDJSON Streaming)

Assumptions mostly mirror SSE scenario; differences focus on framing & reconnect protocol.

### 17.1 Differences from SSE
* Content-Type: `application/x-ndjson` (or `application/json` if preferred).
* No implicit retry: client must implement reconnect with exponential backoff (e.g., 1s, 2s, 4s up to 30s).
* Resume marker passed via query parameter `?since=<eventID>` (or `?offset=` tuple) rather than `Last-Event-ID` header.

### 17.2 Server Handler (Go NDJSON)
```go
func ndjsonHandler(w http.ResponseWriter, r *http.Request) {
  user := authenticate(r)
  if user == nil { http.Error(w, "unauth", http.StatusUnauthorized); return }
  w.Header().Set("Content-Type", "application/x-ndjson")
  w.Header().Set("Cache-Control", "no-cache")
  w.Header().Set("Connection", "keep-alive")
  w.Header().Set("X-Accel-Buffering", "no")
  flusher, _ := w.(http.Flusher)

  since := r.URL.Query().Get("since")
  sess := registerSession(user.ID, since) // may push replay events onto queue first
  defer unregisterSession(sess)

  ctx := r.Context()
  heartbeat := time.NewTicker(20 * time.Second)
  defer heartbeat.Stop()
  for {
    select {
    case <-ctx.Done(): return
    case <-heartbeat.C:
      io.WriteString(w, "\n") // blank line heartbeat
      flusher.Flush()
    case ev := <-sess.ch:
      b, _ := json.Marshal(ev)
      io.WriteString(w, string(b)+"\n")
      flusher.Flush()
    }
  }
}
```

### 17.3 Browser Client (NDJSON)
```js
async function connect(since) {
  const resp = await fetch('/notifications/stream' + (since ? ('?since=' + encodeURIComponent(since)) : ''));
  const reader = resp.body.getReader();
  const dec = new TextDecoder();
  let buf = '', lastID = since, reconnecting = false;
  while (true) {
    const {done, value} = await reader.read();
    if (done) break;
    buf += dec.decode(value, {stream:true});
    const lines = buf.split('\n');
    buf = lines.pop();
    for (const line of lines) {
      if (!line.trim()) continue;
      const ev = JSON.parse(line);
      lastID = ev.id;
      display(ev);
    }
  }
  // Reconnect with backoff
  let delay = 1000;
  setTimeout(() => connect(lastID), delay);
}
connect();
```

### 17.4 Replay Logic
Identical buffer concept as SSE. If `since` not found (evicted), server can return `410 Gone` or custom 204 with header `X-Replay-Not-Possible: true` prompting client to do a full refresh fetch of latest snapshot + new stream without since.

### 17.5 Ingress / nginx Settings (NDJSON)
Same as SSE (disable buffering, long timeouts). Ensure no unintended gzip for tiny chunks (can add `gzip off;` if latency spikes observed due to compression buffering).

### 17.6 Metrics & Observability (NDJSON)
* notifications_stream_active_connections
* notifications_stream_events_sent_total
* notifications_stream_reconnects_total
* notifications_stream_bytes_total
* notifications_stream_heartbeats_total

### 17.7 Pros / Cons (NDJSON Scenario)
Pros: Flexible framing, easy non-browser consumption, explicit control over reconnect + resume semantics.  
Cons: More client code (no native retry), need to define resume protocol, potential for subtle parsing bugs.

### 17.8 Decision Guidance
Start with SSE for browser-only audiences needing minimal code + auto retry. Adopt NDJSON version when you introduce CLI tools, other languages, or need framing changes (length-prefix, batching) without data: semantics.

---
**Scenario Summary**: Both SSE and NDJSON share the same internal dispatcher & replay buffer. SSE optimizes integration speed; NDJSON optimizes protocol flexibility and multi-client reuse.