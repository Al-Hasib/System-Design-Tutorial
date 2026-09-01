# Study Notes: Web Server Internals

## Definitions

- **Concurrency:** Handling multiple requests as overlapping in time (not necessarily simultaneously executing).
- **Parallelism:** Multiple requests genuinely executing at the same instant, typically across multiple CPU cores.
- **Blocking I/O:** A thread halts execution entirely while waiting for an I/O operation (disk read, network call, DB query) to finish.
- **Non-blocking I/O:** A thread issues an I/O operation and continues doing other work, resuming only when notified the operation completed.
- **Event loop:** A single (or small pool of) thread(s) that continuously pulls completed I/O events/callbacks off a queue and executes them, never blocking on any one operation.
- **C10K problem:** The historical challenge (named in a 1999 essay) of handling 10,000+ concurrent connections on one server, which one-thread/process-per-connection architectures could not do efficiently.

## Concurrency Models Compared

| Model | Unit of concurrency | Memory cost per connection | Scales to | Examples |
|---|---|---|---|---|
| Process-per-request | OS process | High (full process + memory space) | Hundreds | Old CGI |
| Thread-per-request | OS/lightweight thread | Medium (thread stack, often 1MB+) | Low thousands | Apache (prefork/worker), traditional Java servlet containers |
| Event loop / async non-blocking I/O | Callback/coroutine on shared thread(s) | Low (no dedicated thread per idle connection) | Tens of thousands+ | Nginx, Node.js, Python asyncio, Netty (Java) |

## Static vs Dynamic Content

| Aspect | Static content | Dynamic content |
|---|---|---|
| Examples | Images, CSS, JS bundles, prebuilt HTML | API responses, personalized pages, DB-backed data |
| Work per request | None — read bytes and send | Real computation: DB queries, business logic |
| Best served from | CDN edge / reverse proxy cache | Application server |
| Cacheability | Highly cacheable, long TTLs | Sometimes cacheable (cache-aside, short TTL), often not |

## Key Trade-offs

- Thread-per-request: simple mental model (write blocking, synchronous code), but each blocked thread wastes memory/scheduler time while waiting on I/O.
- Event loop: scales I/O-bound concurrency extremely well, but a single CPU-bound task blocks *every* connection sharing that loop — must offload CPU-heavy work to a worker pool or separate service.
- Most production stacks are hybrid: async I/O handling for requests, backed by a bounded worker/thread pool for CPU-bound or blocking legacy work.

## Key Numbers / Facts

- The "C10K problem" essay (Dan Kegel, 1999) framed handling 10,000 simultaneous connections as the emerging scalability bottleneck of thread-per-connection servers.
- A typical OS thread stack reserves on the order of 1MB of memory by default — 10,000 threads could mean ~10GB just in stack space before any actual work happens.
- Nginx's default worker model uses a small, fixed number of worker processes (often matching CPU core count), each running its own event loop handling thousands of connections.

## Summary

- A web server's job is always accept → read → process → write; performance is about how "process" scales under concurrency.
- Thread/process-per-request models don't scale past low thousands of connections due to OS overhead (the C10K problem); event-loop/async models scale much further by never dedicating a thread to an idle, waiting connection.
- Static content should be served from a CDN/edge and never reach application concurrency logic at all — leaving app server capacity for dynamic, request-specific work.
