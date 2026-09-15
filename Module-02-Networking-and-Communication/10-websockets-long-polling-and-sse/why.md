# Why This Topic Matters: WebSockets, Long Polling & Server-Sent Events

> **In one sentence:** HTTP was designed so the client always speaks first, which makes "tell me the instant something happens" surprisingly hard — and the workarounds people invent by instinct are the expensive ones.

## The World Before This Idea

You're building a chat app. A message arrives on the server. How does the recipient's browser find out?

The instinctive answer is polling: ask the server every two seconds, "anything new?" It works, and its costs are brutal. With 100,000 connected users polling every 2 seconds, that's 50,000 requests per second — and roughly 99% of them return "nothing new." You're paying full HTTP request cost (connection handling, TLS, headers, auth check, database query) for an empty answer, over and over. And the user still sees up to 2 seconds of delay.

Tighten the interval to get better latency and the load rises linearly. Loosen it to cut load and the app feels broken. There is no setting that is both cheap and responsive, because polling is structurally wrong for this problem.

## The Problems It Solves

### 1. Wasted work on empty polls
**What you see:** Enormous request volume, high infrastructure cost, and dashboards full of 200-with-nothing responses.

**Why it happens:** The client has no way to know whether there's news, so it must ask constantly.

**How push solves it:** With SSE or WebSockets, the connection stays open and the server sends data only when data exists. 100,000 idle users cost you 100,000 mostly-idle connections instead of 50,000 requests per second. Modern event-loop servers hold idle connections cheaply; that's the whole trade.

### 2. Latency floors you can't optimize away
**What you see:** Notifications feel sluggish no matter how fast the backend gets.

**Why it happens:** With a 2-second poll interval, average delay is 1 second and worst case is 2 — entirely independent of server speed.

**How push solves it:** Delivery happens as soon as the event exists. Latency drops to network round-trip time, which is what "real time" actually means in practice.

### 3. Client-to-server streams, not just server-to-client
**What you see:** Typing indicators, collaborative cursors, and multiplayer game input need frequent small messages *upward*, and each one as a separate HTTP request carries hundreds of bytes of headers for a few bytes of payload.

**Why it happens:** Request/response has fixed per-message overhead, paid in both directions.

**How WebSockets solve it:** One persistent, full-duplex connection with framing overhead measured in single-digit bytes. Both sides send whenever they want. This is the case where WebSockets are clearly correct and SSE is not enough.

### 4. Reaching for the heavy tool when a light one fits
**What you see:** A team adopts WebSockets for a live-updating dashboard, then discovers they need to handle reconnection, heartbeats, backpressure, sticky routing across a server fleet, and proxies that don't understand the protocol.

**Why it happens:** "Real-time" gets treated as one requirement with one answer.

**How this topic solves it:** It separates the cases. If data only flows server-to-client — notifications, live scores, progress bars, streaming LLM tokens — Server-Sent Events is plain HTTP, works through every proxy, and reconnects automatically with almost no code. Knowing that saves teams months of accidental complexity.

## The Price You Pay

Persistent connections change how you operate a system:

- **Stateful servers.** A WebSocket is pinned to one server. Deploys disconnect users; scaling out doesn't rebalance existing connections; load balancers need connection-aware handling. This undoes some of the simplicity that statelessness bought you.
- **Cross-server fan-out.** If user A is on server 1 and user B is on server 3, delivering A's message to B requires a Pub/Sub backbone (usually Redis or Kafka) between servers.
- **Connection limits and memory.** Each open connection costs file descriptors and buffer memory. A million connections is an infrastructure project, not a config change.
- **Infrastructure friction.** Some corporate proxies and older load balancers mishandle long-lived connections, so production systems need fallback paths and aggressive heartbeats to detect dead connections.
- **Polling is sometimes right.** For infrequent updates and small user counts, a simple poll is less code, less operational risk, and perfectly adequate. Don't build a push system for a page that updates hourly.

## When You Need It — and When You Don't

| Use | When |
|---|---|
| **Short polling** | Updates are infrequent, user count is modest, simplicity wins |
| **Long polling** | You need near-real-time but must work on legacy infrastructure |
| **SSE** | Server-to-client only: notifications, feeds, live metrics, token streaming |
| **WebSockets** | Genuine bidirectional, high-frequency traffic: chat, collaboration, games, trading |

## Why This Shows Up in Interviews

Chat, notifications, live feeds, collaborative editing, and ride tracking are all standard design prompts, and each hinges on this decision. The answer interviewers want isn't "WebSockets" — it's the reasoning: which direction does data flow, how frequently, how many concurrent connections, and therefore which mechanism. The best candidates then raise the hard part unprompted: connection state across a fleet, and the Pub/Sub layer needed to route a message to whichever server holds the recipient's socket.

## How It Connects

Persistent connections reintroduce the statefulness that **horizontal scaling** worked to eliminate, which is why they depend on **Pub/Sub** for cross-server delivery and complicate **load balancing** and **zero-downtime deployments**. They're central to the **chat application** and **ride-sharing** case studies, and the choice interacts directly with **transport protocols** and **API gateway** behavior.

**Next:** [SQL vs NoSQL](../../Module-03-Databases-and-Storage/11-sql-vs-nosql/why.md) — moving from how data travels to where it lives.
