# অধ্যয়ন নোট: Batch বনাম Stream Processing

## মূল সংজ্ঞাসমূহ

- **Batch processing**: একটা নির্দিষ্ট সময় ধরে সংগৃহীত bounded, finite dataset একসাথে, একটা শুরু ও শেষ সহ একটা discrete job হিসেবে প্রসেস করা।
- **Stream processing**: একটা unbounded, ক্রমাগত ডেটার প্রবাহ, event by event (অথবা ছোট micro-batch-এ), যেভাবে আসে সেভাবে প্রসেস করা।
- **Windowing**: একটা অসীম stream-কে finite chunk-এ ভাগ করার কৌশল যাতে aggregate গণনা (counts, averages, ইত্যাদি) করা যায়।
- **Event time**: একটা event আসলে কখন ঘটেছে তার timestamp।
- **Processing time**: system আসলে কখন সেই event প্রসেস করেছে তার timestamp।
- **Watermark**: stream processing-এর একটা mechanism যা অনুমান করে system event time-এ কতটা পিছিয়ে থাকা ডেটা এখনো পেতে পারে, এবং সম্ভাব্য late arrival সত্ত্বেও কখন একটা window চূড়ান্ত করা হবে তা সিদ্ধান্ত নিতে ব্যবহৃত হয়।

## Batch বনাম Stream তুলনা

| দিক | Batch Processing | Stream Processing |
|---|---|---|
| Data scope | Bounded, finite dataset | Unbounded, continuous data |
| Latency | বেশি (মিনিট থেকে ঘণ্টা/দিন) | কম (মিলিসেকেন্ড থেকে সেকেন্ড) |
| Throughput efficiency | অনেক বেশি (bulk optimization) | প্রতি-event efficiency কম, কিন্তু ক্রমাগতভাবে scalable |
| Complexity | কম (সহজ mental model) | বেশি (ordering, late data, windowing) |
| সাধারণ tools | Hadoop MapReduce, Apache Spark (batch mode), Apache Hive | Kafka Streams, Apache Flink, Spark Structured Streaming |
| উদাহরণ use case | রাতের financial report, সাপ্তাহিক ML model retraining, payroll processing | Fraud detection, real-time dashboard, alerting, live recommendations |
| Data freshness | সংজ্ঞানুসারেই stale (batch interval-এর মতোই পুরনো) | Near real-time |

## Windowing-এর ধরন

| Window ধরন | বর্ণনা | উদাহরণ |
|---|---|---|
| Tumbling | Fixed-size, non-overlapping, পরপর windows | "প্রতি ৬০-সেকেন্ড window-এ অর্ডার" |
| Sliding | Fixed-size windows যা overlap করে, ছোট interval-এ পুনরায় গণনা করা হয় | "শেষ ৬০ সেকেন্ড, প্রতি ১০ সেকেন্ডে আপডেট হয়" |
| Session | Activity অনুযায়ী events গ্রুপ করে, নিষ্ক্রিয়তার ফাঁকের পর বন্ধ হয় | "৩০ মিনিট idle থাকার পর user session শেষ হয়" |

## Lambda বনাম Kappa Architecture

| দিক | Lambda Architecture | Kappa Architecture |
|---|---|---|
| Approach | Parallel batch layer (accurate, ধীর) + speed layer (দ্রুত, আনুমানিক); serving time-এ ফলাফল merge করা | একটা একক stream-processing pipeline; সব ডেটা (historical সহ) একটা replayable stream হিসেবে ধরা |
| Codebases | দুটো আলাদা codebase/logic path maintain করতে হয় | একটা codebase |
| Reprocessing | Batch layer পর্যায়ক্রমে পুনরায় গণনা/সংশোধন করে | পুনরায় গণনার জন্য stream শুরু থেকে replay করা |
| Complexity | বেশি operational burden (দুটো system) | সহজ, কিন্তু পর্যাপ্ত retention সহ একটা log-based system (যেমন, Kafka) দরকার |
| যা এটাকে সম্ভব করে | Traditional batch (Hadoop) + streaming systems পাশাপাশি | Kafka-র মতো durable, replayable logs |

## দ্রুত সারসংক্ষেপ

- Batch বেছে নিন যখন ঘণ্টা/দিনের latency গ্রহণযোগ্য এবং আপনি বড়, bounded গণনার জন্য সর্বোচ্চ throughput/efficiency চান।
- Stream বেছে নিন যখন সেকেন্ডের মধ্যে প্রতিক্রিয়া দেখানো দরকার এবং ordering, late data, এবং windowing-এর বাড়তি complexity সহ্য করতে পারেন।
- অনেক বাস্তব system (যেমন, Spotify, Uber) একই product-এর মধ্যে ভিন্ন use case-এর সাথে মিলিয়ে দুটোই ব্যবহার করে।
- একটা durable, replayable log (Kafka-র মতো) যদি ইতিমধ্যেই stack-এর অংশ হয়, তাহলে Lambda-র চেয়ে Kappa architecture ক্রমশ বেশি পছন্দ করা হচ্ছে, কারণ এটা দুটো parallel codebase maintain করা এড়ায়।
