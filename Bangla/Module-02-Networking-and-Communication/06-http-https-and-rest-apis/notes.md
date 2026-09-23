# Study Notes: HTTP/HTTPS & REST APIs

## Definitions

- **HTTP (HyperText Transfer Protocol):** Application-layer, stateless, request-response protocol যা clients এবং servers-এর মধ্যে communication-এর জন্য ব্যবহৃত হয়, TCP/IP-এর উপর তৈরি।
- **Stateless:** Server একটি client-এর আগের requests-এর কোনো memory রাখে না; প্রতিটি request-এর সাথে তার প্রয়োজনীয় সব context (cookies, tokens, ইত্যাদি) বহন করতে হয়।
- **TLS (Transport Layer Security):** একটি cryptographic protocol যা server-কে authenticate করে, keys negotiate করে, এবং transit-এ data encrypt করে। HTTPS = HTTP over TLS।
- **REST (Representational State Transfer):** একটি architectural style যেখানে system state resources হিসেবে model করা হয়, প্রতিটি একটি URL দ্বারা চিহ্নিত, standard HTTP methods-এর মাধ্যমে manipulate করা হয়।
- **Idempotent operation:** এমন একটি operation যা যতবারই প্রয়োগ করা হোক না কেন একই result তৈরি করে।

## HTTP Methods

| Method | উদ্দেশ্য | Idempotent? | Safe (side effect নেই)? | সাধারণ ব্যবহার |
|--------|---------|-------------|--------------------------|--------------|
| GET | Resource retrieve করা | হ্যাঁ | হ্যাঁ | Data fetch করা |
| POST | Resource তৈরি করা / action trigger করা | না | না | Order তৈরি করা, form submit করা |
| PUT | Resource সম্পূর্ণভাবে replace করা | হ্যাঁ | না | Full update |
| PATCH | Resource আংশিকভাবে update করা | না (সাধারণত) | না | Partial update |
| DELETE | Resource সরানো | হ্যাঁ | না | Item remove করা |

## Status Code Ranges

| Range | অর্থ | সাধারণ codes |
|-------|---------|---------------|
| 1xx | Informational | 100 Continue |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved Permanently, 302 Found, 304 Not Modified |
| 4xx | Client error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Server error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

- **401 বনাম 403:** 401 = authenticated নয় (তুমি কে?)। 403 = authenticated কিন্তু authorized নয় (আমি তোমাকে চিনি, তুমি এটা করতে পারবে না)।

## HTTP বনাম HTTPS

| দিক | HTTP | HTTPS |
|--------|------|-------|
| Encryption | নেই (cleartext) | TLS-encrypted |
| Server identity verification | না | হ্যাঁ (certificate) |
| Default port | 80 | 443 |
| Latency | কম (handshake নেই) | সামান্য বেশি (TLS handshake), session resumption / HTTP/2+ দ্বারা কমানো হয় |
| Production-এ ব্যবহার | Sensitive data-এর জন্য গ্রহণযোগ্য নয় | Standard/required |

## HTTP Protocol Version Notes

- **HTTP/1.1:** একই সময়ে প্রতি connection-এ একটি request (pipelining খুব কমই ব্যবহৃত হয়); keep-alive reconnect overhead কমায়।
- **HTTP/2:** একটি single TCP connection-এর উপর একাধিক requests/responses multiplex করে; header compression (HPACK)।
- **HTTP/3:** TCP-এর বদলে QUIC (UDP-based)-এর উপর চলে, TCP head-of-line blocking এড়িয়ে; দ্রুত connection setup।

## REST Principles

- Resources হলো nouns, URLs দ্বারা চিহ্নিত (`/users/5`), path-এ verbs নয় (`/getUser`)।
- Actions HTTP methods-এর মাধ্যমে প্রকাশ করা হয়, URL-এর মাধ্যমে নয়।
- Stateless: প্রতিটি request self-contained।
- যেখানে উপযুক্ত সেখানে Cacheable (GET responses, `Cache-Control`, `ETag`)।
- Uniform interface: পুরো API জুড়ে consistent conventions।
- Relationships প্রকাশ করতে resources nest করা যেতে পারে: `/users/5/orders`।

## Good REST API Design Checklist

- Collections-এর জন্য plural nouns: `/user` নয়, `/users`।
- Filtering/sorting/pagination-এর জন্য query params ব্যবহার করুন: `/orders?status=shipped&page=2&limit=20`।
- API version করুন: `/v1/users` বা একটি `Accept`/custom header।
- সঠিক status codes ফেরত দিন, সবসময় একটি error field সহ 200 নয়।
- Unsafe operations-এর জন্য (যেমন POST) যেগুলোকে retries সহ্য করতে হবে, সেগুলোর জন্য `Idempotency-Key` header ব্যবহার করুন।

## Key Numbers / Facts

- Default HTTP port: 80। Default HTTPS port: 443।
- TLS handshake সাধারণত 1-2টি network round trips যোগ করে (TLS 1.3-এ কম, যা 1-RTT এবং 0-RTT resumption সমর্থন করে)।
- REST আনুষ্ঠানিকভাবে Roy Fielding-এর 2000 সালের PhD dissertation-এ বর্ণিত হয়েছিল।

## Summary

- HTTP হলো client-server communication-এর ভাষা: stateless, request/response, method + status code driven।
- HTTPS একটি ছোট handshake overhead-এর বিনিময়ে TLS-এর মাধ্যমে confidentiality এবং server authentication যোগ করে।
- REST HTTP methods ব্যবহার করে CRUD-style operations-কে resource URLs-এর উপর map করে, যা predictable, cacheable, ভালোভাবে গঠিত APIs তৈরি করে।
