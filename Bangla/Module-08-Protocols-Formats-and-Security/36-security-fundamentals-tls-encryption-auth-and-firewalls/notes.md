# Study Notes: Security Fundamentals

## সংজ্ঞা (Definitions)

- **Encryption:** একটি key ব্যবহার করে data-র reversible রূপান্তর, যাতে শুধুমাত্র সঠিক key-র holder-রা এটি পড়তে পারে। Confidentiality-র জন্য ব্যবহৃত।
- **Hashing:** One-way রূপান্তর যা একটি fixed-size fingerprint তৈরি করে; reverse করা যায় না। Verification-এর জন্য ব্যবহৃত (যেমন, password storage, integrity check)।
- **Symmetric encryption:** Encryption এবং decryption উভয়ের জন্য ব্যবহৃত একটি shared secret key (যেমন, AES)। দ্রুত, কিন্তু নিরাপদ key distribution প্রয়োজন।
- **Asymmetric encryption:** একটি public/private key pair; public key দিয়ে encrypted data শুধুমাত্র private key দিয়ে decrypt করা হয় (যেমন, RSA, ECC)। ধীর, কিন্তু key distribution সমাধান করে।
- **Authentication (AuthN):** একটি party *কে* তা যাচাই করা।
- **Authorization (AuthZ):** একটি authenticated party *কী* করার অনুমতি পায় তা যাচাই করা।
- **mTLS (mutual TLS):** TLS বর্ধিত করা হয়েছে যাতে client এবং server উভয়ই certificate প্রদর্শন করে, একে অপরকে authenticate করে (শুধু server নয়, যেমন সাধারণ HTTPS-এ)।
- **Firewall:** একটি network control যা rule-এর ভিত্তিতে (source/destination IP, port, protocol) ট্রাফিক allow/block করে।
- **WAF (Web Application Firewall):** পরিচিত attack pattern (SQLi, XSS, ইত্যাদি) এর জন্য HTTP ট্রাফিক পরীক্ষা করে এবং application layer-এ তাদের block করে।
- **Defense in depth:** একাধিক স্বাধীন security control স্তরে সাজানো যাতে একটি ব্যর্থ হলে সম্পূর্ণ compromise না হয়।

## Encryption বনাম Hashing

| | Encryption | Hashing |
|---|---|---|
| Reversible? | হ্যাঁ (key দিয়ে) | না |
| উদ্দেশ্য | Confidentiality (পরে মূল ফিরে পাওয়া) | Verification/integrity (মূল ফেরত দরকার নেই) |
| সাধারণ ব্যবহার | Data in transit (TLS), data at rest | Password storage, checksum, digital signature |
| উদাহরণ algorithm | AES (symmetric), RSA/ECC (asymmetric) | bcrypt, Argon2 (password); SHA-256 (integrity) |

## Symmetric বনাম Asymmetric Encryption

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | একটি shared secret key | Public/private key pair |
| গতি | দ্রুত | ধীর (অনেক বেশি computation) |
| Key distribution সমস্যা | হ্যাঁ — প্রথমে নিরাপদে key share করতে হবে | না — public key প্রকাশ্যে share করা যায় |
| উদাহরণ algorithm | AES | RSA, ECC |
| TLS-এ যেখানে ব্যবহৃত | Actual session data encrypt করতে | Server authenticate করতে এবং symmetric key exchange করতে |

## AuthN বনাম AuthZ

| | Authentication | Authorization |
|---|---|---|
| উত্তরযুক্ত প্রশ্ন | আপনি কে? | আপনি কী করতে পারেন? |
| Failure status code | 401 Unauthorized | 403 Forbidden |
| অন্যটির উপর নির্ভরশীল? | প্রথমে ঘটে | Authentication ইতিমধ্যে সফল হয়ে থাকা প্রয়োজন |
| সাধারণ mechanism | Sessions, JWT, OAuth2/OIDC login | Roles/permissions চেক, scopes, policy engines |

## Authentication Mechanism

| Mechanism | এটি কীভাবে কাজ করে | Trade-off |
|---|---|---|
| Session-based | Server একটি opaque session ID (cookie) ইস্যু করে; state server-side সংরক্ষিত (যেমন, Redis) | সহজ instant revocation; horizontally scale করতে shared session store প্রয়োজন |
| JWT | Self-contained signed token identity claim বহন করে | কোনো server-side lookup প্রয়োজন নেই (stateless, সহজে scale হয়); expiry-র আগে revoke করা কঠিন |
| OAuth 2.0 | Delegated authorization-এর জন্য standard (password share না করে সীমিত access দেওয়া) | Third-party access-এর জন্য industry standard ("Sign in with X") |
| OpenID Connect (OIDC) | OAuth2-এর উপরে identity layer | "এই user কে" তা standardize করে, শুধু "এই app কী access করতে পারে" নয় |

## Network এবং Application Layer Defense

| Control | Layer | উদ্দেশ্য |
|---|---|---|
| Firewall / Security Group | Network | কোন source কোন destination-এ পৌঁছাতে পারবে তা সীমিত করে (IP/port/protocol) |
| Network segmentation | Network | Infrastructure-কে trust zone-এ ভাগ করে যাতে একটিতে breach সবগুলোতে access না দেয় |
| mTLS / Service Mesh | Transport | Internal service-দের মধ্যে mutual authentication + encryption |
| WAF | Application (HTTP) | App code-এ পৌঁছানোর আগে পরিচিত attack pattern (SQL injection, XSS) block করে |

## মূল সংখ্যা / তথ্য (Key Numbers / Facts)

- একই পরিমাণ data-র জন্য AES (symmetric) সাধারণত RSA (asymmetric)-এর চেয়ে বহুগুণ দ্রুত — এই কারণেই TLS handshake-এর সময় শুধুমাত্র সংক্ষেপে asymmetric crypto ব্যবহার করে।
- bcrypt এবং Argon2 ইচ্ছাকৃতভাবে ধীর (tunable work factor) বিশেষভাবে leaked password hash brute-force করাকে computationally ব্যয়বহুল করার জন্য।
- OAuth 2.0 ২০১২ সালে RFC 6749 হিসেবে চূড়ান্ত হয়েছিল; OpenID Connect এর উপরে স্তর যোগ করে।

## সারাংশ (Summary)

- মূল ফেরত দরকার হলে encrypt করুন (confidentiality); শুধু match যাচাই করা দরকার হলে hash করুন (কখনো plaintext password সংরক্ষণ করবেন না)।
- TLS asymmetric encryption (key exchange/server authentication) কে symmetric encryption (fast session encryption)-এর সাথে একত্রিত করে।
- Authentication (কে) এবং authorization (কী) আলাদা এবং ধারাবাহিক — 401 বনাম 403।
- mTLS/service mesh service-to-service trust সুরক্ষিত করে; firewall/segmentation এবং WAF network এবং HTTP layer সুরক্ষিত করে।
- কোনো একক control একা trusted নয় — defense in depth ধরে নেয় যে কোনো একটি স্তর ব্যর্থ হতে পারে।
