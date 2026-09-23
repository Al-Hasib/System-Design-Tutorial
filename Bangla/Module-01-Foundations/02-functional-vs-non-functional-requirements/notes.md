# নোট: Functional vs Non-Functional Requirements

## সংজ্ঞা

| ধরন | সংজ্ঞা | উত্তর দেয় | উদাহরণ |
|---|---|---|---|
| **Functional Requirement** | System-টি কী করবে — features এবং behaviors | "একজন user কী করতে পারে?" | একটি photo আপলোড করা, একজন user-কে follow করা, একটি post-এ like দেওয়া |
| **Non-Functional Requirement** | System সেই functions কতটা ভালোভাবে সম্পাদন করবে | "এটা কত দ্রুত/reliable/secure হতে হবে?" | 99.99% uptime, <300ms latency, 500M DAU |

## সাধারণ Non-Functional ক্যাটাগরি

| ক্যাটাগরি | এটি যে প্রশ্নের উত্তর দেয় |
|---|---|
| Performance/Latency | Responses কত দ্রুত হতে হবে? |
| Scalability | এখন এবং পরে, এটাকে কত traffic/data/users সামলাতে হবে? |
| Availability | কত শতাংশ uptime দরকার? |
| Consistency | Data পরিবর্তন কত দ্রুত সব জায়গায় প্রচারিত (propagate)/দৃশ্যমান হতে হবে? |
| Durability | নিশ্চিত হওয়া (confirmed) data কি কখনো হারাতে পারে? |
| Security | কী সুরক্ষিত রাখতে হবে, কার থেকে? |
| Cost | Infrastructure বাজেট কত? |

## Back-of-the-Envelope Estimation-এর পদ্ধতি

১. **Daily Active Users (DAU)** দিয়ে শুরু করুন।
২. প্রতিদিন প্রতি user-এর **actions** দিয়ে গুণ করুন → মোট দৈনিক requests।
৩. **86,400 seconds/day** দিয়ে ভাগ করুন → গড় requests per second (RPS)।
৪. গড় RPS-কে **2-3 গুণ** দিয়ে গুণ করুন → peak RPS (traffic সমানভাবে বণ্টিত হয় না)।
৫. Storage-এর জন্য: **items/day × প্রতি item-এর গড় আকার** → দৈনিক storage বৃদ্ধি।

### উদাহরণ হিসাব

- 100M DAU × 10 requests/day = 1B requests/day
- 1B / 86,400 ≈ গড়ে 11,600 RPS
- 11,600 × 2.5 ≈ ~29,000 RPS peak
- 10M photo uploads/day × 2MB = দৈনিক 20TB নতুন storage

## মূল তুলনামূলক উদাহরণ

| System | Functional Req | প্রধান Non-Functional Req |
|---|---|---|
| URL Shortener | URL সংক্ষিপ্ত করা, redirect করা | অতি-নিম্ন read latency, বিশাল read:write ratio |
| Banking System | টাকা transfer করা | Strong consistency, শূন্য data loss (durability) |

## দ্রুত পুনরালোচনার পয়েন্ট

- Functional = "কী"; Non-functional = "কতটা ভালোভাবে।"
- Design করার আগে সবসময় clarifying questions জিজ্ঞাসা করুন (scale, read/write ratio, consistency needs, global vs regional)।
- একই feature set + ভিন্ন non-functional requirements = ভিন্ন architecture।
- Capacity estimation-এর লক্ষ্য মোটামুটি হিসাব (নির্ভুল সংখ্যা নয়)।
</content>
