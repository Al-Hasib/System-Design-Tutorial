# Study Notes: Forward Proxy vs Reverse Proxy

## Definitions

- **Proxy:** An intermediary server that relays requests and responses between a client and a destination server.
- **Forward proxy:** A proxy that sits in front of clients and represents them to the outside world; the destination server sees the proxy, not the real client.
- **Reverse proxy:** A proxy that sits in front of servers and represents them to the outside world; the client sees the proxy, not the real backend servers.

## Forward Proxy vs Reverse Proxy Comparison

| Aspect | Forward Proxy | Reverse Proxy |
|--------|----------------|-----------------|
| Who deploys/configures it | The client side (e.g., a company for its employees) | The server side (e.g., a company for its own backend) |
| Who it hides | The client, from the server | The server (and its topology), from the client |
| What the other party sees | Server sees only the proxy's identity | Client sees only the proxy's identity |
| Typical uses | Content filtering, anonymity, bypassing geo-restriction, client-side caching, monitoring outbound traffic | TLS termination, load balancing, caching responses, hiding internal architecture, compression, security (WAF) |
| Example products | Corporate proxy servers, Squid, VPN services | NGINX, HAProxy, Envoy, cloud load balancers (ALB), Cloudflare |
| Client awareness | Client explicitly configured to use it | Client is unaware a proxy/backend fleet exists |

## Relationship to Load Balancing

- A load balancer operating at Layer 7 is, technically, a reverse proxy with load-distribution algorithms as one of its features.
- Reverse proxy is the broader category; load balancing, TLS termination, caching, and compression are common functions it performs.
- Not every reverse proxy load-balances (some just proxy to a single backend for caching/security), and not every load balancer is L7 (L4 balancers don't do full proxying of application content in the same way).

## Common Reverse Proxy Responsibilities

- TLS/SSL termination
- Caching static or frequently-requested content
- Compression (gzip/br)
- Load balancing across backend pool
- Hiding internal network topology and real server IPs
- Request/response rewriting, header injection
- Basic security filtering (rate limiting, WAF rules)

## Common Forward Proxy Responsibilities

- Content filtering / access control for outbound traffic
- Anonymizing client identity from destination servers
- Caching frequently requested external resources
- Logging/monitoring outbound traffic for compliance
- Bypassing network/geographic restrictions

## Summary

- One sentence to remember: **forward proxy protects the client, reverse proxy protects the server.**
- Both are intermediaries; the difference is entirely about which side deploys them and whose identity they shield.
- Load balancers are typically a specialized case of reverse proxy, not a separate concept.
