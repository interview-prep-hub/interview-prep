# Realtime: Polling, SSE and WebSockets

> **Tier:** T2 (working knowledge) · **Related:** [Networking essentials](../01-cs-fundamentals/06-networking.md), [How the web works](01-how-the-web-works.md), [Action Cable, Mailer and Active Storage](../05-ruby-on-rails/15-actioncable-mailer-activestorage.md), [Hotwire: Turbo and Stimulus](../05-ruby-on-rails/08-hotwire-turbo-stimulus.md), [Push notifications](../08-react-native-expo/05-push-notifications.md), [Redis](../12-databases/10-redis.md), [System design fundamentals](../15-system-design/01-fundamentals.md)
> **On your resume:** push notifications, events, tasks and split bills in the Coloc resident app; the Colochousing Rails 7 app; Hotwire (Turbo Streams) training at Visnext.

## Quick revision
- **Short polling:** the client asks every N seconds. Simple and wasteful; latency up to N.
- **Long polling:** the server holds the request until there is data or a timeout, then the client asks again at once. Near realtime over plain HTTP.
- **SSE (Server-Sent Events):** one long HTTP response streaming `text/event-stream` messages from server to client, with built-in reconnection and `Last-Event-ID`. Text only, one direction.
- **WebSockets:** an HTTP/1.1 `Upgrade` handshake, then a full-duplex framed connection (`ws://`, `wss://`) carrying text or binary with tiny per-message overhead.
- Choose polling for infrequent updates and job status, SSE for server-to-client feeds (notifications, progress, LLM token streaming), WebSockets for low-latency two-way traffic (chat, collaboration, presence, games).
- Frames: text, binary, ping, pong, close. Ping/pong detects dead peers and keeps proxies and NATs from dropping idle connections.
- Scaling: each connection is pinned to one server; broadcast across servers through a pub/sub backplane (Redis for Action Cable and Socket.IO); mind load balancer idle timeouts and file descriptor limits.
- **Sticky sessions** are needed when one session spans several HTTP requests, such as Socket.IO's long-polling transport.
- **Socket.IO** is its own protocol on top of WebSocket: reconnection, acknowledgements, **rooms** (server-side groups) and **namespaces** (separate channels with their own handlers and auth).
- **Turbo Streams** over Action Cable push server-rendered HTML fragments from Rails (`turbo_stream_from`, `broadcasts_to`).
- **Mobile:** sockets only live while the app is in the foreground; push notifications (APNs, FCM, Expo push) reach backgrounded apps, and the app refetches on resume.
- **Presence** = heartbeats plus expiring keys in Redis; **typing indicators** are throttled, ephemeral and never stored.
- Authenticate at connection time (cookies plus an `Origin` check, or a short-lived ticket), authorize every subscription, and disconnect when access ends.
- Reconnect with exponential backoff and jitter, resubscribe, then fetch what was missed.
- Delivery is at-most-once by default. For reliability add persisted events with sequence numbers and replay, dedupe by event ID, and treat messages as hints to refetch from the API.

## 1. The options compared

| | Short polling | Long polling | SSE | WebSockets |
|---|---|---|---|---|
| Direction | client pulls | client pulls; server answers when ready | server to client | both ways |
| Transport | repeated HTTP requests | held HTTP requests | one long HTTP response | upgraded TCP connection |
| Latency | up to the interval | near realtime | near realtime | lowest |
| Overhead | high: full requests, mostly empty answers | medium | low | lowest per message |
| Reconnection | not needed | the client loop | built into `EventSource` | build it yourself or use a library |
| Payload | anything | anything | UTF-8 text | text and binary |
| Proxies and firewalls | trivial | fine, watch timeouts | fine, disable buffering | need `Upgrade` support and long idle timeouts |
| Good for | job status, dashboards every 30 to 60 s | fallback transport, simple notifications | notifications, feeds, progress, LLM token streams | chat, collaboration, presence, live cursors, games |

Push notifications (section 8) cover apps that are not open at all. WebRTC handles peer-to-peer audio, video and data, and WebTransport is an emerging HTTP/3-based option.

## 2. Polling and long polling

```ts
// Short polling with TanStack Query v5: poll every 3 s until the export finishes
const { data } = useQuery({
  queryKey: ["export", exportId],
  queryFn: () => fetchExport(exportId), // returns Promise<{ status: string; downloadUrl?: string }>
  refetchInterval: (query) => (query.state.data?.status === "completed" ? false : 3000),
});
```

Polling is underrated: it is stateless, cacheable, works through every proxy and survives deploys. It pairs naturally with the 202-plus-status-resource pattern from [REST API design](04-rest-api-design.md).

**Long polling:** the server holds each request until a new event arrives or about 25 to 30 seconds pass (then returns 204), and the client immediately sends the next request with a cursor (`?since=<last_event_id>`) so nothing falls between requests. Every waiting client occupies a connection, and on thread-per-request servers such as Puma, a thread too, unless the server handles waiting asynchronously.

## 3. Server-Sent Events

```text
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

retry: 5000

id: 41
event: notification
data: {"title":"Rent is due tomorrow"}

: keep-alive comment, sent every 15 to 30 seconds

id: 42
event: bill.created
data: {"id":981,"amountCents":4500}
```

Each message is a block of `field: value` lines ended by a blank line; `retry` sets the reconnection delay in milliseconds, `id` is remembered and sent back as `Last-Event-ID` when the browser reconnects, and lines starting with `:` are comments used as heartbeats.

```ts
// Express-style handler; req.user is set by your auth middleware
app.get("/events", (req, res) => {
  res.set({ "Content-Type": "text/event-stream", "Cache-Control": "no-cache", Connection: "keep-alive" });
  res.flushHeaders();

  function send(e: { id: number; type: string; data: unknown }) {
    res.write(`id: ${e.id}\nevent: ${e.type}\ndata: ${JSON.stringify(e.data)}\n\n`);
  }

  const lastId = Number(req.get("Last-Event-ID") ?? 0);       // sent by the browser on reconnect
  for (const e of missedEventsSince(req.user.id, lastId)) send(e);

  const unsubscribe = subscribe(req.user.id, send);           // for example a Redis pub/sub listener
  const heartbeat = setInterval(() => res.write(": ping\n\n"), 20_000);
  req.on("close", () => { clearInterval(heartbeat); unsubscribe(); });
});
```

```ts
const source = new EventSource("/events", { withCredentials: true }); // cookies only: no custom headers
source.addEventListener("notification", (e) => showToast(JSON.parse(e.data)));
source.onerror = () => {
  // the browser reconnects on its own, honoring retry and sending Last-Event-ID
};
```

- `EventSource` only does GET and cannot set headers, so authenticate with cookies or a short-lived ticket in the URL. For POST requests with headers (for example a chat endpoint), read a `fetch` response body as a stream (`response.body.getReader()`) and parse the event format yourself.
- **Streaming LLM output** is the classic modern use: LLM APIs such as OpenAI's stream tokens as SSE, and your backend re-streams them to the browser so text appears as it is generated.
- **Proxies** must not buffer the stream: Nginx `proxy_buffering off;` or an `X-Accel-Buffering: no` response header, plus a long read timeout; compression middleware can buffer too.
- Under HTTP/1.1 browsers allow about six connections per origin and each SSE stream holds one, so many open tabs can starve other requests; HTTP/2 multiplexing removes the limit.
- Hono has a `streamSSE` helper; Rails can stream with `ActionController::Live`, where each open stream occupies a Puma thread.

## 4. WebSockets: handshake, frames, ping/pong

The client sends an HTTP/1.1 `GET` with `Upgrade: websocket`, `Connection: Upgrade`, `Sec-WebSocket-Key` and `Sec-WebSocket-Version: 13` (the browser adds `Origin`). The server answers `101 Switching Protocols` with `Sec-WebSocket-Accept`, a hash derived from the key that proves it speaks the protocol. From then on the same TCP connection carries **frames** in both directions; the raw exchange is shown in [Networking essentials](../01-cs-fundamentals/06-networking.md).

- **Frames** have a small header and a payload, with opcodes for text, binary, continuation, close, ping and pong. Client-to-server frames are masked, which protects intermediaries from cache-poisoning tricks.
- **Ping/pong:** either side may ping and the other must answer with a pong. Servers ping every 20 to 30 seconds to detect dead clients (a phone that loses signal never sends a close frame) and to keep load balancers and NAT gateways from cutting idle connections. Browsers answer pings automatically, but the JavaScript API cannot send them, so clients detect dead servers with application-level heartbeats.
- **Close codes:** 1000 normal, 1001 going away (tab closed, server restarting), 1006 abnormal (no close frame; reported locally, never sent), 1008 policy violation (for example failed auth), 1011 server error.

```ts
const ws = new WebSocket("wss://realtime.example.com/socket");
ws.onopen = () => ws.send(JSON.stringify({ type: "subscribe", channel: "house:12" }));
ws.onmessage = (event) => handleMessage(JSON.parse(event.data));
ws.onclose = (event) => console.log("closed", event.code, event.reason); // reconnect: section 11
```

WebSocket messages have no built-in structure, so define an envelope (`{ type, channel, id, data }`), version it and validate every incoming message like any API input. Server options: `ws` in Node, Socket.IO, Action Cable in Rails, Django Channels, or managed services such as Pusher, Ably and AWS API Gateway WebSockets.

## 5. Scaling realtime

```text
 clients ---> realtime server A ---+
 clients ---> realtime server B ---+---> Redis pub/sub <--- web app or Sidekiq job publishes "bill.created" on house:12
 clients ---> realtime server C ---+
 Each server subscribes to the channels its own clients need and forwards messages to those sockets.
```

- A socket lives on the server that accepted it. A background job on another machine cannot reach it directly, so it publishes to a **backplane** (Redis pub/sub, NATS, Postgres `LISTEN/NOTIFY`), and every realtime server relays the message to its local subscribers. Action Cable uses Redis (or Solid Cable's database table, the Rails 8 default); Socket.IO uses `@socket.io/redis-adapter`.
- **Sticky sessions** route every request of one session to the same server. They are required when a session spans many HTTP requests (Socket.IO's long-polling transport); a pure WebSocket is one long-lived connection and does not need them.
- **Capacity:** every connection holds memory and a file descriptor (raise `ulimit -n`), servers must be event-driven rather than thread-per-connection, and load balancers need WebSocket support with idle timeouts above your ping interval (an AWS Application Load Balancer defaults to 60 seconds).
- **Deploys** drop every connection and all clients reconnect at once, so jittered backoff and graceful draining matter.
- Split realtime servers from the main app as connection counts grow, or use a managed service. Remember that Redis pub/sub is fire-and-forget: a server that is restarting misses messages (section 12).

## 6. Socket.IO: rooms and namespaces

```ts
import { createServer } from "node:http";
import { Server } from "socket.io";

const httpServer = createServer();
const io = new Server(httpServer, { cors: { origin: "https://app.example.com" } });

io.use(async (socket, next) => {
  try {
    socket.data.user = await verifyToken(socket.handshake.auth.token); // reject unauthenticated sockets
    next();
  } catch {
    next(new Error("unauthorized"));
  }
});

io.on("connection", (socket) => {
  socket.on("house:join", async (houseId: string, ack: (ok: boolean) => void) => {
    const allowed = await isMember(socket.data.user.id, houseId); // authorize every room join
    if (allowed) socket.join(`house:${houseId}`);
    ack(allowed);
  });
});

// elsewhere, after a bill is created:
// io.to(`house:${houseId}`).emit("bill:created", { id: bill.id, amountCents: bill.amountCents });
httpServer.listen(3001);
```

```ts
import { io } from "socket.io-client";

const socket = io("https://realtime.example.com", { auth: { token } });
socket.emit("house:join", houseId, (ok: boolean) => { if (!ok) console.warn("not a member"); });
socket.on("bill:created", () => queryClient.invalidateQueries({ queryKey: ["bills", houseId] }));
```

- Socket.IO is not plain WebSocket: it has its own protocol (Engine.IO underneath) that can start with HTTP long polling and upgrade to WebSocket, so a plain WebSocket client cannot talk to a Socket.IO server.
- **Rooms** are server-side groups: a socket can join many, `io.to(room).emit(...)` broadcasts to members, and clients cannot join rooms themselves, so the server decides (and authorizes) membership.
- **Namespaces** split one connection into separate channels (`io.of("/admin")`), each with its own handlers, rooms and middleware, such as admin-only auth.
- Extras: acknowledgements (callbacks), automatic reconnection with backoff, client-side buffering of emits while disconnected (they replay on reconnect, which may surprise you), and `volatile` emits for data that may be dropped, such as typing events.
- Across several nodes, use the Redis adapter, plus sticky sessions if long polling stays enabled.

## 7. Turbo Streams over WebSockets in Rails

```erb
<%# app/views/houses/show.html.erb: subscribe this page to the house's stream over Action Cable %>
<%= turbo_stream_from @house %>
<div id="bills"><%= render @house.bills %></div>
```

```ruby
class Bill < ApplicationRecord
  belongs_to :house
  # after commits, render the bill partial and push append, replace or remove actions to the house's stream
  broadcasts_to :house, inserts_by: :prepend
end
```

The server renders HTML once and pushes `<turbo-stream action="prepend" target="bills">` fragments, so there is no client-side state or JSON API to maintain. Turbo 8 adds `broadcasts_refreshes`, which tells subscribed pages to refresh and morph in the changes. `turbo_stream_from` uses a signed stream name, so clients cannot subscribe to streams they were never shown, but render it only on pages the user may see. Details: [Hotwire](../05-ruby-on-rails/08-hotwire-turbo-stimulus.md) and [Action Cable](../05-ruby-on-rails/15-actioncable-mailer-activestorage.md).

## 8. Mobile: push notifications vs in-app realtime

iOS and Android suspend backgrounded apps, so sockets die soon after the user leaves the app. Push notifications (APNs on iOS, FCM on Android, both wrapped by Expo's push service) are the only reliable way to reach a backgrounded or closed app. A sensible design for an app like Coloc:
1. **Foreground:** keep open screens fresh with a socket, SSE or polling, or simply refetch when the screen gains focus.
2. **Background or closed:** send a push for events that matter ("New bill: Internet, 45 EUR").
3. **On resume or notification tap:** refetch from the API (TanStack Query's focus manager driven by `AppState`), because anything sent while disconnected was missed.

Push delivery is best-effort: messages can be delayed, collapsed or disabled by the user, and silent background pushes are throttled by the OS. Keep sensitive data out of payloads and never treat push as the source of truth ([Push notifications](../08-react-native-expo/05-push-notifications.md)).

## 9. Presence and typing indicators

```text
SET presence:user:42 online EX 30                         # refreshed by a heartbeat every 15 s; expiry means offline
ZADD presence:house:12 1792000000 user:42                 # per-house sorted set scored by last-seen time
ZRANGEBYSCORE presence:house:12 1791999970 +inf           # members seen in the last 30 s
ZREMRANGEBYSCORE presence:house:12 -inf 1791999970        # prune stale members
```

- Count connections per user (several tabs and devices), and announce "offline" only when the last one disappears, after a short grace period so brief reconnects do not flap.
- Broadcast joins and leaves to the room; show "last seen" from the stored timestamp.
- **Typing indicators:** the client sends `typing` at most every 2 to 3 seconds while the user types and `stopped` on send or blur; receivers show "Sara is typing..." and hide it automatically after about 5 seconds without updates. They are ephemeral and best-effort: never persisted, fine to drop.

## 10. Authenticating WebSocket connections

- The browser `WebSocket` constructor cannot set custom headers, so the common options are:
  - **Cookies:** the handshake is an HTTP request, so same-site session cookies travel automatically. Then check `Origin` against an allowlist, because WebSockets are not covered by CORS: any site can open a socket to your server with the user's cookies (cross-site WebSocket hijacking).
  - **A short-lived, single-use ticket** fetched from an authenticated HTTP endpoint and passed in the URL or the first message, so long-lived tokens never land in URLs and logs.
  - **A token in the first message** (close the socket if it does not arrive within a few seconds), or Socket.IO's `auth` payload.
- **Authorize every subscription,** not just the connection: joining `house:12` requires membership of house 12.
- **Re-check over time:** tokens expire and memberships change, so disconnect a user's sockets on logout or loss of access (keep an index from user to sockets).
- **Action Cable:** `ApplicationCable::Connection#connect` finds the user (usually from a signed or encrypted cookie), sets it with `identified_by :current_user`, and calls `reject_unauthorized_connection` otherwise; channels authorize in `subscribed`.
- Rate limit messages per connection and validate every payload: a socket is an API endpoint.

## 11. Reconnection with backoff

```ts
let lastEventId = 0;

function connect(attempt = 0) {
  const ws = new WebSocket("wss://realtime.example.com/socket");
  ws.onopen = () => {
    attempt = 0;                                                   // reset the backoff after a success
    ws.send(JSON.stringify({ type: "resume", since: lastEventId })); // ask for anything missed
  };
  ws.onmessage = (e) => {
    const msg = JSON.parse(e.data);
    lastEventId = msg.id;
    handleMessage(msg);
  };
  ws.onclose = (e) => {
    if (e.code === 1000) return;                                   // closed on purpose
    const delay = Math.min(30_000, 1_000 * 2 ** attempt) * (0.5 + Math.random() / 2); // capped, jittered
    setTimeout(() => connect(attempt + 1), delay);
  };
}
```

- Exponential backoff with jitter stops thousands of clients from reconnecting in lockstep after a deploy; cap the delay and reset it after success.
- After reconnecting: re-authenticate, resubscribe, and fetch what was missed (a last event ID, a `since` timestamp, or a plain refetch of the screen's data).
- Show the connection state ("Reconnecting...") and disable or queue actions that need the socket.
- React to network changes (browser `online`/`offline` events, NetInfo in React Native) and reconnect when the app returns to the foreground. The Socket.IO client and Action Cable's consumer handle backoff for you; a plain `WebSocket` does not.

## 12. Ordering and delivery guarantees

- TCP guarantees ordered, reliable delivery **within one connection** only. Across reconnects, servers or producers there are no guarantees.
- The default is **at-most-once**: anything published while a client is disconnected, or while a server misses a Redis pub/sub message, is gone.
- For **at-least-once**, persist events (a table, Redis Streams, Kafka) with a per-channel sequence number; clients remember the last sequence they processed and request a replay after reconnecting (SSE's `Last-Event-ID` is this idea built in); optionally acknowledge messages.
- Replays create duplicates, so dedupe by event ID and keep handlers idempotent. "Exactly once" in practice means at-least-once delivery plus idempotent processing.
- Ordering across producers needs a single sequencer per stream (a database sequence, Redis Stream IDs, a Kafka partition); order across different streams is not guaranteed and rarely needed.
- The pragmatic pattern for CRUD apps: send small invalidation hints ("bill 981 changed") and refetch from the API, which stays the source of truth; refetch-on-reconnect heals anything missed.
- Redis pub/sub has no persistence or acknowledgements; Redis Streams persist entries, support consumer groups and replay from an ID ([Redis](../12-databases/10-redis.md)).

## Common pitfalls
- Reaching for WebSockets when polling every 30 seconds or SSE would do.
- A single realtime server with no backplane, so broadcasts reach only some users once you scale out.
- Authenticating the connection but not each subscription or room join.
- Accepting WebSocket handshakes without checking `Origin` when using cookie auth.
- Long-lived tokens in WebSocket URLs, which end up in logs.
- Load balancer idle timeouts shorter than the heartbeat interval, causing constant reconnects.
- Reconnecting without backoff and jitter, then stampeding the server after every deploy.
- Assuming messages sent while a client was offline will arrive; never refetching after reconnect.
- Proxies buffering SSE streams, so events arrive in bursts or only at the end.
- Expecting sockets to work while a mobile app is backgrounded instead of using push.
- Putting sensitive data in push payloads.

## How to talk about this in an interview
- The Coloc resident app shipped push notifications for features such as events, tasks, split bills and rent payments: explain foreground vs background delivery, refetching on resume, and why push payloads stay small and non-sensitive.
- From your Rails and Hotwire background (Visnext training, the Colochousing Rails 7 app), Turbo Streams over Action Cable is the Rails way to push server-rendered HTML.
- If you have not run WebSockets in production, say so plainly and then walk through a design: transport choice, backplane, authentication and authorization per subscription, reconnection with backoff, and recovering missed messages.
- Always justify the transport from the requirement (direction, latency, scale, client type) rather than defaulting to WebSockets.

> **Fill in:** whether any Devntech feature used in-app realtime (Action Cable, Turbo Streams, polling) and how it was built, or how you would add it to the Coloc app.

> **Fill in:** how push notifications were triggered in the Coloc app (Sidekiq jobs, Expo push service, which events) and how the app refreshed data after a notification tap.

## Interview questions

**Q1. Polling vs long polling vs SSE vs WebSockets: how do they compare, and when do you use each?**
Short polling repeats requests on an interval: simple and cacheable but wasteful and laggy. Long polling holds each request until data arrives: near realtime over plain HTTP, but it ties up connections. SSE streams server-to-client text events over one HTTP response with automatic reconnection. WebSockets give full-duplex, low-overhead messaging. Polling for job status and slow dashboards, SSE for notifications and token streaming, WebSockets for chat, collaboration and presence.
*Follow-up:* Which is easiest to scale? Polling, because it is stateless and works with ordinary HTTP caching and load balancing.

**Q2. How does the WebSocket handshake work?**
The client sends an HTTP/1.1 GET with `Upgrade: websocket`, `Connection: Upgrade`, a random `Sec-WebSocket-Key` and the protocol version; the server replies `101 Switching Protocols` with `Sec-WebSocket-Accept` derived from the key. The TCP connection then carries WebSocket frames in both directions until one side sends a close frame.
*Follow-up:* What does `wss://` add? TLS, exactly like HTTPS; always use it in production.

**Q3. When would you choose SSE over WebSockets?**
When data flows only from server to client: notifications, live feeds, progress updates, streaming LLM output. SSE is plain HTTP, reconnects and resumes automatically with `Last-Event-ID`, and passes through proxies easily. Choose WebSockets when the client must also send frequent low-latency messages or binary data.
*Follow-up:* What limits SSE? GET only, no custom headers from `EventSource`, text only, and the per-origin connection limit under HTTP/1.1.

**Q4. How do you scale WebSockets across multiple servers?**
Connections are pinned to the server that accepted them, so add a pub/sub backplane (Redis pub/sub, NATS) through which any process publishes and every realtime server relays to its local sockets. Raise file descriptor limits, use event-driven servers, set load balancer idle timeouts above the heartbeat interval, and handle reconnect storms after deploys with jittered backoff.
*Follow-up:* What does Action Cable use? Redis, or Solid Cable's database table in Rails 8 apps.

**Q5. Why does Socket.IO need sticky sessions?**
Its long-polling transport sends a series of separate HTTP requests that must all reach the server holding that session's state. With WebSocket-only transport the session is a single connection, so sticky sessions are unnecessary; broadcasting across nodes still needs the Redis adapter.
*Follow-up:* How do you configure WebSocket-only? Set `transports: ["websocket"]` on the client, accepting the loss of the polling fallback.

**Q6. Socket.IO rooms vs namespaces?**
Rooms are server-side groups of sockets used for targeted broadcasts (`io.to("house:12").emit(...)`); clients cannot join them directly, so the server authorizes joins. Namespaces are separate channels on one connection (`/admin`) with their own handlers, rooms and middleware.
*Follow-up:* Rooms in your design? One room per house or organization, joined only after a membership check.

**Q7. How do you authenticate and authorize WebSocket connections?**
Authenticate during the handshake with a same-site cookie (and an `Origin` allowlist check, since WebSockets bypass CORS), a short-lived single-use ticket, or a token in the first message or Socket.IO's `auth` payload. Then authorize every subscription or room join, and disconnect sockets when the user logs out or loses access.
*Follow-up:* What is cross-site WebSocket hijacking? A malicious page opening a socket to your server with the victim's cookies, which an `Origin` check prevents.

**Q8. How do you handle reconnection and messages missed while offline?**
Reconnect with capped exponential backoff and jitter, reset after success, re-authenticate and resubscribe, then recover missed data: replay from the last event ID or sequence number if events are persisted, or simply refetch the screen's data from the API. Show the connection state in the UI.
*Follow-up:* Why jitter? Without it, every client disconnected by a deploy reconnects at the same instant.

**Q9. What delivery and ordering guarantees do WebSockets give?**
Ordered, reliable delivery within a single connection, nothing more: messages sent while a client was disconnected are lost (at-most-once). Reliable delivery needs persisted events with sequence numbers, replay on reconnect and dedupe by event ID; ordering across producers needs one sequencer per stream.
*Follow-up:* What is the simplest robust pattern? Messages as invalidation hints plus refetching from the API.

**Q10. How would you implement presence and typing indicators?**
Presence: clients heartbeat every 15 seconds, the server refreshes an expiring Redis key or a sorted set scored by last-seen time, counts connections per user across tabs and devices, and broadcasts joins and leaves with a grace period. Typing: throttled `typing` events that receivers expire after a few seconds, sent as volatile, never stored.
*Follow-up:* Why a grace period before "offline"? Brief reconnects would otherwise make users flicker offline and online.

**Q11. Push notifications vs in-app realtime in a mobile app?**
In-app realtime (sockets, SSE, polling) only works while the app is in the foreground, because the OS suspends background apps. Push notifications through APNs and FCM (or Expo's push service) reach backgrounded and closed apps, but they are best-effort and not a data channel, so the app refetches from the API when it resumes or a notification is tapped.
*Follow-up:* What should a push payload contain? A short message and identifiers to deep-link with, not sensitive data.

**Q12. What are ping and pong frames for?**
Health checks at the protocol level: one side pings and the other must pong. Servers use them to detect dead clients that vanished without a close frame and to keep proxies, load balancers and NATs from closing idle connections. Browser JavaScript cannot send pings, so clients add application-level heartbeats.
*Follow-up:* What happens if the ping interval exceeds the load balancer's idle timeout? The balancer silently cuts connections, causing constant reconnects.

**Q13. How would you add live updates to a Rails page?**
With Hotwire: `turbo_stream_from @house` in the view subscribes over Action Cable, and `broadcasts_to :house` (or explicit `broadcast_*` calls after commit) pushes rendered partials as Turbo Stream actions; Turbo 8's `broadcasts_refreshes` triggers morphing page refreshes instead. Run Action Cable with Redis or Solid Cable in production and authorize streams.
*Follow-up:* Where does broadcasting happen for heavy pages? In background jobs, using the `_later` broadcast variants so requests stay fast.

## Further reading
- [MDN: The WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) and [MDN: Server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [Rails guide: Action Cable overview](https://guides.rubyonrails.org/action_cable_overview.html)
- [RFC 6455: The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
