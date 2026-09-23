# Study Notes: Multi-Region Architecture & Disaster Recovery

## সংজ্ঞা (Definitions)

- **Region:** একটি ভৌগোলিকভাবে স্বতন্ত্র, শারীরিকভাবে বিচ্ছিন্ন infrastructure অবস্থান, যাতে একটিতে ঘটা failure অন্যটিতে ছড়িয়ে না পড়ে।
- **Active-passive (active-standby):** একটি region live traffic পরিবেশন করে; দ্বিতীয় একটি region replicated data নিয়ে প্রস্তুত থাকে কিন্তু failover না হওয়া পর্যন্ত production traffic পরিবেশন করে না।
- **Failover:** active region fail করার পর একটি standby region-এ traffic redirect করার প্রক্রিয়া।
- **Active-active:** একাধিক region একই সাথে live production traffic পরিবেশন করে, সাধারণত নিকটতম region-এ route করা হয়।
- **RTO (Recovery Time Objective):** একটি disaster এবং সিস্টেম আবার অনলাইনে ফিরে আসার মধ্যে সর্বোচ্চ গ্রহণযোগ্য সময়।
- **RPO (Recovery Point Objective):** সর্বোচ্চ গ্রহণযোগ্য data loss-এর পরিমাণ, সময়ে পরিমাপ করা হয় (যেমন, "সর্বোচ্চ 5 মিনিটের write")।

## Active-Passive বনাম Active-Active

| দিক | Active-Passive | Active-Active |
| --------------------- | --------------------------------------------- | ---------------------------------------------------------- |
| Traffic পরিবেশন | একবারে একটি region | সব region একই সাথে |
| Idle capacity খরচ | Standby region বেশিরভাগ সময় অব্যবহৃত | নেই — সব capacity প্রকৃত কাজ করে |
| Failover delay | প্রকৃত (DNS propagation, warm-up, verification) | নেই — fail over করার মতো কোনো একক active region নেই |
| Consistency চ্যালেঞ্জ | সরল — একবারে একটি write region | Cross-region write-এ CAP/PACELC trade-off মোকাবিলা করতে হয় |
| সেরা মানানসই | কম RTO সহনশীলতা গ্রহণযোগ্য (মিনিট) | খুব কম RTO প্রয়োজনীয়তা, প্রায়-শূন্য downtime |

## RTO / RPO Architecture চালিত করে

| RPO প্রয়োজনীয়তা | Replication স্ট্র্যাটেজি |
| ------------------------ | ----------------------------------------------------------------------- |
| শূন্য data loss | Synchronous cross-region replication (প্রতিটি write-এ প্রকৃত latency খরচ) |
| কয়েক মিনিট গ্রহণযোগ্য | Asynchronous replication (দ্রুততর, প্রতিদিনের জন্য সাশ্রয়ী) |

| RTO প্রয়োজনীয়তা | Architecture |
| --------------- | --------------------------------------------------------- |
| সেকেন্ড | Active-active (কোনো failover delay নেই) |
| মিনিট | Automated failover সহ Active-passive |
| ঘণ্টা | Backup থেকে documented manual recovery গ্রহণযোগ্য হতে পারে |

## মূল নীতি (Key Principle)

- RTO এবং RPO হলো **ব্যবসায়িক সিদ্ধান্ত**, শক্তিশালী guarantee-র খরচ (latency, infrastructure খরচ, engineering জটিলতা) সেই নির্দিষ্ট সিস্টেমের downtime/data loss-এর প্রকৃত ব্যবসায়িক খরচের বিপরীতে ওজন করে।
- একই কোম্পানির মধ্যে ভিন্ন ভিন্ন সিস্টেমের যুক্তিসঙ্গতভাবে খুব ভিন্ন RTO/RPO লক্ষ্য এবং architecture থাকতে পারে (যেমন, internal analytics dashboard বনাম customer payment processing)।

## মূল সংখ্যা / তথ্য (Key Numbers / Facts)

- প্রতিটি major cloud provider (AWS, GCP, Azure)-এর ঐতিহাসিকভাবে region-level outage হয়েছে — multi-region ঝুঁকি কাল্পনিক নয়।
- "শূন্য RPO" (synchronous replication, কোনো data loss নেই) অর্জনযোগ্য কিন্তু region-এর মধ্যে দূরত্ব/network condition-এর সমানুপাতিক প্রকৃত write latency যোগ করে।

## সারসংক্ষেপ (Summary)

- Single-region redundancy মেশিন/data-center failure-এর বিরুদ্ধে সুরক্ষা দেয়, region-level failure-এর বিরুদ্ধে নয় — multi-region architecture সেই ফাঁকটি মোকাবিলা করে।
- Active-passive সরল, idle-capacity খরচ এবং failover delay সহ; active-active উভয়ই দূর করে কিন্তু একটি স্পষ্ট cross-region consistency trade-off বাধ্য করে।
- RTO এবং RPO হলো নির্দিষ্ট, ইচ্ছাকৃতভাবে নির্বাচিত সংখ্যা যা নির্ধারণ করে একটি নির্দিষ্ট সিস্টেমের প্রকৃতপক্ষে কোন architecture এবং replication স্ট্র্যাটেজি প্রয়োজন — খরচকে ব্যবসায়িক প্রভাবের বিপরীতে ওজন করে নির্ধারিত, "যতটা সম্ভব শক্তিশালী"-তে ডিফল্ট করা নয়।
