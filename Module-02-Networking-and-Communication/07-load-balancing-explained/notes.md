# Study Notes: Load Balancing

## Definitions

- **Load balancer:** A component that sits in front of a pool of servers and distributes incoming traffic across them based on an algorithm.
- **Health check:** A periodic probe (e.g., HTTP GET to `/health`) the load balancer uses to determine if a backend server is available.
- **Sticky session:** Routing all requests from a given client to the same backend server, usually to preserve in-memory session state.
- **Single point of failure (SPOF):** A component whose failure takes down the whole system; load balancers must be made redundant to avoid becoming one.

## Load Balancing Algorithms

| Algorithm | How it works | Best for | Weakness |
|-----------|---------------|----------|----------|
| Round robin | Requests cycle through servers in fixed order | Homogeneous servers, uniform request cost | Ignores actual server load |
| Weighted round robin | Round robin, but higher-capacity servers get more turns | Heterogeneous server capacity | Static weights don't adapt to real-time load |
| Least connections | New request goes to server with fewest active connections | Variable request cost/duration | Slightly more overhead to track state |
| Weighted least connections | Least connections, weighted by server capacity | Heterogeneous servers + variable request cost | More complex to tune |
| IP hash | Hash of client IP determines server | Need session stickiness without cookies | Uneven load if client IPs are skewed; reshuffling on server changes |
| Least response time | Combines connection count and observed latency | Latency-sensitive services | More overhead to measure |
| Consistent hashing | Maps requests/keys to servers on a hash ring, minimal remapping on changes | Caching layers, sharded backends | More complex to implement (covered in Module 6) |

## Layer 4 vs Layer 7 Load Balancing

| Aspect | Layer 4 (Transport) | Layer 7 (Application) |
|--------|----------------------|--------------------------|
| OSI Layer | Transport (TCP/UDP) | Application (HTTP/HTTPS/gRPC, etc.) |
| Visibility | IP address + port only | Full request: URL, headers, cookies, body |
| Routing intelligence | Cannot route by content | Can route by path, header, cookie, etc. |
| Performance | Very fast, low CPU overhead | More CPU overhead (parses each request) |
| TLS termination | Typically passes through | Can terminate TLS on behalf of backends |
| Example tech | AWS Network Load Balancer, IPVS | NGINX, HAProxy (L7 mode), AWS ALB, Envoy |
| Use case | High-throughput, non-HTTP protocols, extreme low latency | Most modern web/API traffic, microservices routing |

Rule of thumb: **L4 routes connections. L7 routes requests.**

## Health Checks

- Active checks: load balancer proactively pings a health endpoint on a schedule.
- Passive checks: load balancer observes real traffic (e.g., error rates, timeouts) to infer health.
- Unhealthy servers are removed from rotation until they pass checks again — enables self-healing traffic routing.

## Avoiding SPOF in the Load Balancing Layer

- Run multiple load balancer instances.
- Use DNS round robin across LB instances, or a floating/virtual IP with VRRP/keepalived.
- Prefer a managed, multi-AZ load balancer service (e.g., cloud provider ALB/NLB) which is redundant by design.

## Summary

- Load balancing is what makes horizontal scaling actually work in practice.
- Algorithm choice depends on server homogeneity, request cost variability, and session requirements.
- L4 vs L7 is a tradeoff between raw speed and routing intelligence — most modern systems favor L7 for HTTP traffic.
- Health checks + redundant load balancers are what make failures invisible to end users.
