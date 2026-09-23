# Study Notes: Web Server Internals

## সংজ্ঞাসমূহ (Definitions)

- **Concurrency:** একাধিক request-কে সময়ে ওভারল্যাপিং হিসেবে হ্যান্ডেল করা (অগত্যা একইসাথে execute হওয়া নয়)।
- **Parallelism:** একাধিক request সত্যিকার অর্থে একই মুহূর্তে execute হওয়া, সাধারণত একাধিক CPU core জুড়ে।
- **Blocking I/O:** একটি I/O operation (disk read, network call, DB query) শেষ হওয়ার জন্য অপেক্ষা করার সময় একটি thread সম্পূর্ণভাবে execution থামিয়ে দেয়।
- **Non-blocking I/O:** একটি thread একটি I/O operation জারি করে এবং অন্য কাজ চালিয়ে যায়, operation সম্পন্ন হওয়ার notification পেলেই শুধু আবার শুরু করে।
- **Event loop:** একটি একক (বা একটি ছোট pool) thread যা ক্রমাগত একটি queue থেকে সম্পন্ন হওয়া I/O event/callback তুলে execute করে, কখনো কোনো একটি operation-এ block হয় না।
- **C10K problem:** একটি server-এ 10,000+ concurrent connection হ্যান্ডেল করার ঐতিহাসিক চ্যালেঞ্জ (1999 সালের একটি প্রবন্ধে নামকরণ করা), যা one-thread/process-per-connection architecture গুলো দক্ষভাবে করতে পারত না।

## Concurrency Model গুলোর তুলনা

| Model | Concurrency-এর একক | Connection প্রতি memory খরচ | যতদূর scale করে | উদাহরণ |
|---|---|---|---|---|
| Process-per-request | OS process | উচ্চ (সম্পূর্ণ process + memory space) | কয়েকশো | পুরনো CGI |
| Thread-per-request | OS/lightweight thread | মাঝারি (thread stack, প্রায়ই 1MB+) | কয়েক হাজার | Apache (prefork/worker), ঐতিহ্যবাহী Java servlet container |
| Event loop / async non-blocking I/O | ভাগাভাগি করা thread(গুলো)-তে Callback/coroutine | কম (idle connection-এর জন্য কোনো dedicated thread নেই) | দশ হাজারেরও বেশি | Nginx, Node.js, Python asyncio, Netty (Java) |

## Static বনাম Dynamic Content

| দিক | Static content | Dynamic content |
|---|---|---|
| উদাহরণ | Image, CSS, JS bundle, prebuilt HTML | API response, personalized page, DB-backed data |
| প্রতি request কাজ | কিছুই না — byte পড়ে পাঠিয়ে দাও | বাস্তব computation: DB query, business logic |
| সবচেয়ে ভালো serve হয় | CDN edge / reverse proxy cache থেকে | Application server থেকে |
| Cacheability | অত্যন্ত cacheable, দীর্ঘ TTL | কখনো কখনো cacheable (cache-aside, স্বল্প TTL), অনেক সময় নয় |

## মূল Trade-off গুলো

- Thread-per-request: সহজ mental model (blocking, synchronous code লেখা), কিন্তু I/O-এর জন্য অপেক্ষা করার সময় প্রতিটি blocked thread memory/scheduler সময় নষ্ট করে।
- Event loop: I/O-bound concurrency অত্যন্ত ভালোভাবে scale করে, কিন্তু একটি একক CPU-bound task সেই loop ভাগাভাগি করা *প্রতিটি* connection-কে block করে দেয় — CPU-heavy কাজ অবশ্যই একটি worker pool বা আলাদা service-এ offload করতে হবে।
- বেশিরভাগ production stack hybrid: request-এর জন্য async I/O handling, CPU-bound বা blocking legacy কাজের জন্য একটি নির্দিষ্ট সীমার worker/thread pool দ্বারা সমর্থিত।

## মূল সংখ্যা/তথ্য

- "C10K problem" প্রবন্ধ (Dan Kegel, 1999) 10,000 সমসাময়িক connection হ্যান্ডেল করাকে thread-per-connection server-এর ক্রমবর্ধমান scalability bottleneck হিসেবে চিহ্নিত করেছিল।
- একটি সাধারণ OS thread stack default-এ প্রায় 1MB memory সংরক্ষণ করে — 10,000 thread মানে কোনো প্রকৃত কাজ শুরু হওয়ার আগেই শুধু stack space-এ প্রায় ~10GB।
- Nginx-এর default worker model একটি ছোট, নির্দিষ্ট সংখ্যক worker process ব্যবহার করে (প্রায়ই CPU core সংখ্যার সাথে মিলিয়ে), প্রতিটি নিজস্ব event loop চালায় যা হাজার হাজার connection হ্যান্ডেল করে।

## সারাংশ

- একটি web server-এর কাজ সবসময় accept → read → process → write; performance নির্ভর করে "process" ধাপটি concurrency-র অধীনে কীভাবে scale করে তার উপর।
- Thread/process-per-request model গুলো OS overhead-এর কারণে কম কয়েক হাজার connection-এর বেশি scale করে না (C10K problem); event-loop/async model গুলো idle, অপেক্ষারত একটি connection-এর জন্য কখনো একটি thread উৎসর্গ না করে অনেক বেশি scale করে।
- Static content একটি CDN/edge থেকে serve করা উচিত এবং কখনোই application concurrency logic পর্যন্ত পৌঁছানো উচিত নয় — app server-এর capacity dynamic, request-নির্দিষ্ট কাজের জন্য রেখে দিয়ে।
