# Notes: Scalability Basics — Vertical vs Horizontal Scaling

## Definition

**Scalability**: resource যোগ করে ক্রমবর্ধমান load (ব্যবহারকারী, request, data) সামলানোর একটি সিস্টেমের ক্ষমতা, আদর্শভাবে performance degradation বা একটি সম্পূর্ণ redesign ছাড়াই।

## Vertical vs Horizontal Scaling

| দিক | Vertical Scaling (Scale Up) | Horizontal Scaling (Scale Out) |
|---|---|---|
| পদ্ধতি | একটি machine-এ আরও CPU/RAM/storage যোগ করা | Pool-এ আরও machine যোগ করা |
| উপমা | একটি বড় truck-এ upgrade করা | একটি fleet-এ আরও van যোগ করা |
| জটিলতা | কম — app code প্রায়ই অপরিবর্তিত থাকে | বেশি — load balancing, statelessness প্রয়োজন |
| সীমা (Ceiling) | কঠিন physical/cost সীমা | কার্যত সীমাহীন |
| Fault tolerance | দুর্বল — single point of failure | ভালো — একটি ব্যর্থ হলে অন্য machine টিকে থাকে |
| Cost curve | Non-linear; top-tier hardware অসামঞ্জস্যপূর্ণভাবে ব্যয়বহুল | Scale-এ প্রায়ই বেশি cost-effective; redundancy যোগ করে |
| প্রয়োজন | অতিরিক্ত কিছুই না | Load balancer, stateless service, shared data store |

## Prerequisites for Horizontal Scaling

1. **Load balancer** — server-গুলোতে request বিতরণ করে (Module 2)।
2. **Stateless application server** — যেকোনো server যেকোনো request সামলাতে পারে।
3. **Shared/external data store** — সব server-এর জন্য accessible database বা cache (Module 3-4)।

## Rule of Thumb

> "Vertical scaling আপনাকে সময় কিনে দেয়; horizontal scaling আপনাকে একটি ভবিষ্যৎ কিনে দেয়।"

সাধারণ বৃদ্ধির pattern: vertical দিয়ে শুরু (শুরুর দিকে সহজ, সস্তা) → একটি একক machine-এর সীমায় পৌঁছালে বা redundancy প্রয়োজন হলে horizontal-এ স্থানান্তর।

## Quick Revision Bullets

- Scalability = সিস্টেম পুনর্লিখন না করে resource যোগ করে বৃদ্ধি সামলানো।
- Vertical = বড় machine; সহজ কিন্তু সীমাবদ্ধ এবং ভঙ্গুর (SPOF)।
- Horizontal = আরও machine; scalable এবং resilient কিন্তু operationally জটিল।
- Horizontal scaling-এর জন্য statelessness + load balancing + shared data storage প্রয়োজন।
- Real system সাধারণত উভয়ই ব্যবহার করে, সময়ের সাথে vertical থেকে horizontal-এ স্থানান্তরিত হয়।
