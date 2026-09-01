# Practice & Interview Questions

**1. What are the four steps every web server performs for each request?**
Accept the connection, read and parse the request, process it (whatever work is needed to produce a response), and write the response back.

**2. What is the C10K problem, and why did it force a change in web server design?**
It's the historical difficulty of handling 10,000+ concurrent connections on a single server. Thread-per-connection and process-per-connection models cost too much memory and OS scheduling overhead per connection to scale that far, which pushed the industry toward event-loop, non-blocking I/O architectures that don't dedicate a thread to each idle, waiting connection.

**3. Explain the difference between blocking and non-blocking I/O.**
Blocking I/O halts the calling thread entirely until the operation (e.g., a DB query) completes. Non-blocking I/O issues the operation and lets the thread continue doing other work, being notified (via a callback or event) only once the operation finishes.

**4. Why can a single CPU-bound task "block the event loop," and why is that dangerous?**
An event loop typically runs on one (or a few) threads that also need to keep dispatching I/O callbacks for every other connection. If one piece of work runs a long synchronous computation instead of yielding, that thread can't service any other connection until it finishes — stalling every request the loop was handling, not just the slow one.

**5. Why should static assets like images and CSS files not be served directly by your application server's request-handling code?**
They require no per-request computation — they're the same bytes every time — so routing them through the application's concurrency model wastes capacity that should be reserved for dynamic, request-specific work. They belong at a CDN or reverse-proxy edge, which can serve (and cache) them far more cheaply and closer to the user.

**6. Compare thread-per-request and event-loop models on how they handle a slow database query.**
Thread-per-request: the dedicated thread blocks and does nothing else until the query returns, tying up that thread's memory and scheduling slot the whole time. Event loop: the query is issued non-blockingly, the loop moves on to service other connections, and a callback resumes this request only once the query result arrives — no thread sits idle waiting.

**7. What's a practical way to handle a genuinely CPU-heavy task (e.g., PDF generation, image resizing) in an event-loop-based server without hurting other requests?**
Offload it to a separate worker pool, background job queue, or dedicated service (this is exactly the message-queue/async-processing pattern from Module 5) so the CPU-heavy work runs outside the event loop thread and doesn't block other connections' I/O callbacks.

**8. Why does Nginx typically run a small, fixed number of worker processes rather than one thread per connection?**
Each Nginx worker process runs its own event loop capable of handling many thousands of non-blocking connections concurrently, so a handful of workers (often matching CPU core count) can serve far more simultaneous connections than a one-thread-per-connection model, while also allowing genuine parallelism across CPU cores.

**9. Scenario: Your Node.js API's response times spike badly under load even though CPU usage looks low and the database is healthy. What's a plausible cause tied to today's topic?**
A blocking or long-running synchronous operation somewhere in the request path (e.g., synchronous JSON parsing of a huge payload, a tight synchronous loop, or a blocking library call) is stalling the event loop, delaying every other in-flight request even though the database and overall CPU utilization look fine.

**10. True or False: An event-loop architecture eliminates the need for any threads or worker processes at all.**
False. Most production systems are hybrids — an async event loop handles I/O-bound work efficiently, but CPU-bound or blocking work is still offloaded to a separate thread pool, worker process pool, or background job system so it doesn't stall the event loop.
