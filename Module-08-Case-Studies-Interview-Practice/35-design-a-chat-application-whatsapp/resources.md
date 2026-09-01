# Further Reading & References

## Protocols & Standards
- [WebSocket (Wikipedia)](https://en.wikipedia.org/wiki/WebSocket) — overview of the WebSocket protocol used for the persistent client-gateway connections in this design.
- [RFC 6455 — The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455) — the official IETF specification for WebSockets.
- [XMPP (Wikipedia)](https://en.wikipedia.org/wiki/XMPP) — an open messaging/presence protocol historically used by early chat systems (including WhatsApp in its early architecture).
- [Signal Protocol (Wikipedia)](https://en.wikipedia.org/wiki/Signal_Protocol) — the end-to-end encryption protocol used by WhatsApp and Signal; relevant background for the encryption trade-offs discussed in the quiz.

## Infrastructure
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/) — official docs for the distributed pub/sub log used as the message queue / routing layer between connection gateways.
- [Redis Documentation](https://redis.io/docs/latest/) — official docs for Redis, useful for presence tracking, pub/sub fan-out, and caching layers referenced in this design.

## Engineering Background
- [Meta Engineering Blog](https://engineering.fb.com/) — Meta's (WhatsApp's parent company) engineering blog, useful for browsing real-world posts on messaging infrastructure at scale.
