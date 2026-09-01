# Practice & Interview Questions

**1. Why does horizontal scaling require a load balancer?**
Horizontal scaling adds more servers to handle load, but clients need one stable entry point and traffic needs to be intelligently spread across the fleet. A load balancer provides that single entry point and distributes requests so no single server is overwhelmed while others sit idle.

**2. Compare round robin and least connections. When would you prefer one over the other?**
Round robin cycles through servers in fixed order regardless of current load — good when servers are identical and requests are roughly equal cost. Least connections routes to whichever server has the fewest active connections — better when request duration/cost varies significantly, since it avoids piling more work onto an already-busy server.

**3. What is the core difference between Layer 4 and Layer 7 load balancing?**
L4 operates at the transport layer, making decisions using only IP address and port, without understanding the request content. L7 operates at the application layer, parsing the actual HTTP request (path, headers, cookies) to make smarter, content-aware routing decisions.

**4. Why might a system choose an L4 load balancer despite its more limited routing intelligence?**
L4 balancers have much lower per-request CPU overhead since they don't parse application-layer content, making them ideal for extremely high-throughput or latency-sensitive scenarios, or for non-HTTP protocols where content-based routing isn't relevant.

**5. What is a health check, and what are the two general approaches to implementing one?**
A health check is a mechanism the load balancer uses to determine whether a backend server is able to serve traffic. Active health checks proactively probe a dedicated endpoint (e.g., `GET /health`) on a schedule; passive health checks infer health by observing real traffic patterns like error rates or timeouts.

**6. Explain sticky sessions and one drawback of implementing them via IP hashing.**
Sticky sessions route a given client's requests consistently to the same backend server, often needed when that server holds in-memory session state. Implementing this via IP hashing is fragile because many users can share a single public IP (e.g., behind NAT/corporate proxy), causing uneven load, and removing/adding a server reshuffles the hash mapping for many clients at once.

**7. A single load balancer instance is itself a single point of failure. How would you design around this?**
Run multiple redundant load balancer instances, using a mechanism like DNS round robin across them or a floating/virtual IP with a protocol like VRRP/keepalived for automatic failover, or rely on a cloud provider's managed load balancer service, which is inherently redundant across multiple availability zones.

**8. Design the load balancing setup for an API with two very different endpoint types: image uploads (large payload, slow) and simple lookups (small payload, fast). How would you route this traffic?**
Use an L7 load balancer that routes by path — e.g., `/uploads/*` to a pool of servers tuned/scaled for large, slow requests, and `/lookup/*` to a separate pool optimized for fast, high-throughput requests. This isolates the two traffic types so a burst of slow uploads doesn't starve fast lookup requests, and each pool can be scaled independently.

**9. What does "L4 routes connections, L7 routes requests" mean in practice?**
An L4 balancer makes one routing decision per TCP/UDP connection based on IP/port and then just forwards packets for that connection's lifetime. An L7 balancer can make a distinct routing decision for every individual HTTP request, even multiple requests multiplexed over the same underlying connection (e.g., with HTTP/2 keep-alive).

**10. Why is TLS termination often done at the load balancer rather than on each backend server?**
Terminating TLS at the L7 load balancer centralizes certificate management, reduces CPU load on backend servers (decryption happens once), and simplifies operational overhead — backend servers can then communicate over plain HTTP within a trusted internal network.

**11. What's a potential downside of weighted round robin compared to least connections?**
Weighted round robin uses static weights assigned ahead of time based on assumed server capacity; it doesn't adapt in real time if actual load or request cost varies, whereas least connections dynamically reacts to current server load.

**12. In an interview, how would you justify choosing least-connections over round-robin for a video transcoding service?**
Transcoding jobs vary wildly in duration depending on video length and resolution, so round robin could unevenly overload a server that happens to receive several long jobs in a row. Least connections actively accounts for how many jobs each server is currently processing, spreading load more fairly for highly variable-cost work.
