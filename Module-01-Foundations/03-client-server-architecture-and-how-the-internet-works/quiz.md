# Practice & Interview Questions

1. **Describe the client-server model in one or two sentences.**
   Clients initiate requests to servers, and servers process those requests and return responses. The roles are asymmetric — clients don't typically serve requests to other clients.

2. **How does client-server architecture differ from peer-to-peer?**
   In client-server, roles are fixed and asymmetric (clients request, servers respond). In peer-to-peer, every node can act as both a client and a server to other nodes, as seen in systems like BitTorrent.

3. **What problem does DNS solve, and why is it necessary?**
   Computers route traffic using numerical IP addresses, but humans prefer memorable names like "google.com." DNS translates domain names into IP addresses so users don't need to memorize numbers.

4. **Walk through the DNS resolution process at a high level.**
   The browser/OS first checks its local cache; if not found, it queries a DNS resolver, which in turn may query a root server, then a TLD server (e.g., for ".com"), then the authoritative name server for the specific domain, which returns the IP address.

5. **Explain the TCP three-way handshake.**
   The client sends a SYN packet to initiate a connection; the server responds with a SYN-ACK acknowledging and agreeing to connect; the client sends a final ACK. After this exchange, the connection is established and data can be transmitted.

6. **What is the difference between the responsibilities of IP and TCP?**
   IP handles addressing and routing — getting packets from source to destination across networks. TCP adds reliability on top of IP: ordering packets correctly, detecting loss, and retransmitting missing packets.

7. **What does an HTTP request typically consist of?**
   A method (e.g., GET, POST), a path (e.g., `/users/123`), headers (metadata like content type or auth tokens), and optionally a body (data being submitted).

8. **What's the difference between HTTP and HTTPS?**
   HTTPS is HTTP layered on top of TLS (Transport Layer Security), which encrypts the data in transit and allows the client to verify the server's identity via certificates — HTTP alone sends data in plaintext.

9. **Scenario: A user reports that a website is "slow to load for the first time but fast on reload." What networking concept likely explains this?**
   DNS caching and/or TCP connection reuse (keep-alive) — the first visit pays the cost of a full DNS lookup and TCP/TLS handshake, while subsequent visits can reuse cached DNS results and/or an already-open connection.

10. **Why is TCP described as "connection-oriented," and what's the trade-off of that design?**
    TCP requires establishing a connection (via the three-way handshake) before sending data, and maintains state about the connection to guarantee ordered, reliable delivery. The trade-off is added latency upfront (the handshake) in exchange for reliability guarantees.

11. **Scenario: You're designing a system where occasional data loss is acceptable but speed is critical (e.g., live video streaming). Would TCP's reliability guarantees always be the best fit? What might this hint at?**
    Not always — this is a hint toward protocols like UDP that trade reliability for lower latency, since retransmitting old video frames is often less useful than simply moving on to the next frame. (UDP itself is out of scope for this beginner video but worth flagging as a forward-looking consideration.)

12. **What HTTP status code range generally indicates client errors, and what range indicates server errors?**
    4xx status codes (e.g., 404 Not Found) indicate client errors; 5xx status codes (e.g., 500 Internal Server Error) indicate server errors.
