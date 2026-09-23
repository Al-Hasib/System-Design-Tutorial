# আরও পড়ার জন্য ও References

## Protocols ও Standards
- [WebSocket (Wikipedia)](https://en.wikipedia.org/wiki/WebSocket) — এই design-এ persistent client-gateway connections-এর জন্য ব্যবহৃত WebSocket protocol-এর একটি overview।
- [RFC 6455 — The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455) — WebSockets-এর জন্য official IETF specification।
- [XMPP (Wikipedia)](https://en.wikipedia.org/wiki/XMPP) — একটি open messaging/presence protocol যা ঐতিহাসিকভাবে প্রাথমিক chat systems (WhatsApp-এর প্রাথমিক architecture-সহ) ব্যবহার করত।
- [Signal Protocol (Wikipedia)](https://en.wikipedia.org/wiki/Signal_Protocol) — WhatsApp এবং Signal ব্যবহৃত end-to-end encryption protocol; quiz-এ আলোচিত encryption trade-offs-এর জন্য প্রাসঙ্গিক background।

## Infrastructure
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/) — connection gateways-এর মধ্যে message queue / routing layer হিসেবে ব্যবহৃত distributed pub/sub log-এর জন্য official docs।
- [Redis Documentation](https://redis.io/docs/latest/) — Redis-এর জন্য official docs, এই design-এ উল্লেখিত presence tracking, pub/sub fan-out, এবং caching layers-এর জন্য উপযোগী।

## Engineering Background
- [Meta Engineering Blog](https://engineering.fb.com/) — Meta-র (WhatsApp-এর parent company) engineering blog, স্কেলে messaging infrastructure নিয়ে বাস্তব-জগতের posts ব্রাউজ করার জন্য উপযোগী।
