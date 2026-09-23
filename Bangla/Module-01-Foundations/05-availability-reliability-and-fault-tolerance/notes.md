# নোটস: Availability, Reliability, Redundancy & Fault Tolerance

## মূল সংজ্ঞাসমূহ (Core Definitions)

| Term | সংজ্ঞা |
|---|---|
| **Availability** | একটি সিস্টেম কতটা সময় সচল থাকে এবং request-এর জবাব দিতে সক্ষম থাকে তার % |
| **Reliability** | সিস্টেম তার নির্ধারিত কাজ সঠিকভাবে এবং ধারাবাহিকভাবে সময়ের সাথে সম্পন্ন করে |
| **Redundancy** | ব্যাকআপ/ডুপ্লিকেট component থাকা যাতে একটির ব্যর্থতা সিস্টেমকে ব্যর্থ না করে |
| **Fault Tolerance** | Component ব্যর্থ হওয়া সত্ত্বেও সিস্টেম স্বয়ংক্রিয়ভাবে সঠিকভাবে কাজ চালিয়ে যায় |
| **Single Point of Failure (SPOF)** | এমন যেকোনো একটি component যার ব্যর্থতা পুরো সিস্টেমকে ধ্বসিয়ে দেয় |

Availability উত্তর দেয় "এটা কি চালু আছে?" Reliability উত্তর দেয় "চালু থাকা অবস্থায় এটা কি সঠিকভাবে কাজ করছে?"

## The Nines টেবিল

| Availability | ডাকনাম | বছরে Downtime | দিনে Downtime |
|---|---|---|---|
| 99% | Two nines | ~3.65 দিন | ~14.4 মিনিট |
| 99.9% | Three nines | ~8.76 ঘণ্টা | ~1.44 মিনিট |
| 99.99% | Four nines | ~52.6 মিনিট | ~8.6 সেকেন্ড |
| 99.999% | Five nines | ~5.26 মিনিট | ~0.86 সেকেন্ড |

**সাধারণ নিয়ম:** প্রতিটি অতিরিক্ত nine ≈ 10x কম downtime নির্দেশ করে, যার জন্য আনুপাতিকভাবে আরও পরিশীলিত redundancy/infrastructure প্রয়োজন।

## Fault Tolerance কৌশলসমূহ

| কৌশল | এটি যা করে |
|---|---|
| Replication | ডেটা/সার্ভিসের একাধিক কপি যাতে একটি কপি হারালে ডেটা না হারায় (Module 3) |
| Failover | ব্যর্থ component থেকে স্বয়ংক্রিয়ভাবে সুস্থ ব্যাকআপে ট্র্যাফিক পুনর্নির্দেশ করে |
| Health Checks | পর্যায়ক্রমিক স্বয়ংক্রিয় প্রোব যা unhealthy instance শনাক্ত করে, failover ট্রিগার করে |
| Graceful Degradation | non-critical component-এর ব্যর্থতা সম্পূর্ণ ব্যর্থতার বদলে কার্যকারিতা কমিয়ে দেয় |

## Scalability-র সাথে সম্পর্ক (ভিডিও ৪)

- Horizontal scaling (একাধিক সার্ভার) স্বাভাবিক সাইড-ইফেক্ট হিসেবে redundancy প্রদান করে।
- একটি একক, এমনকি অত্যন্ত শক্তিশালী vertically-scaled সার্ভারও সবসময় একটি SPOF।

## দ্রুত পুনরালোচনার বুলেট (Quick Revision Bullets)

- Availability ≠ Reliability: available কিন্তু ভুল উত্তর = unreliable।
- nines টেবিলটি মুখস্থ রাখুন — interviewer-রা প্রায়ই একটি শতাংশকে downtime-এ রূপান্তর করতে বলেন।
- SPOF = single point of failure; সবসময় জিজ্ঞেস করুন "এই একটা জিনিস মারা গেলে কী হবে?"
- Fault tolerance = redundancy + স্বয়ংক্রিয় শনাক্তকরণ (health checks) + স্বয়ংক্রিয় recovery (failover)।
- Horizontal scaling এবং fault tolerance একে অপরকে শক্তিশালী করে।
</content>
