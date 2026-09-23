# Study Notes: Probabilistic Data Structures

## সংজ্ঞাসমূহ

- **Probabilistic data structure:** একটি structure যা একটি নির্দিষ্ট, ছোট পরিমাণ মেমরি ব্যবহার করে, একটি নিয়ন্ত্রিত, সীমাবদ্ধ error rate-এর বিনিময়ে, একটি query-এর আনুমানিক উত্তর দেয়।
- **Bloom filter:** একটি bit array + একাধিক hash function যা "এই এলিমেন্টটি কি সম্ভবত সেটে আছে?" এই প্রশ্নের উত্তর দেয় — কোনো false negative নেই, টিউনযোগ্য false-positive rate।
- **False positive:** Filter বলে "উপস্থিত" কিন্তু এলিমেন্টটি আসলে কখনো যোগ করা হয়নি (Bloom filter-এ সম্ভব, এলিমেন্টগুলোর মধ্যে hash collision-এর কারণে)।
- **False negative:** Filter বলে "উপস্থিত নেই" কিন্তু এলিমেন্টটি আসলে যোগ করা হয়েছিল (Bloom filter-এ অসম্ভব, নির্মাণগতভাবেই)।
- **HyperLogLog:** একটি structure যা hash-value-এর leading zero-এর পরিসংখ্যানের উপর ভিত্তি করে কয়েক কিলোবাইট মেমরি ব্যবহার করে একটি বিশাল stream-এর cardinality (ইউনিক আইটেমের সংখ্যা) অনুমান করে।
- **Count-Min Sketch:** counter-এর একটি 2D array + একাধিক hash function যা একমুখী (শুধুমাত্র বেশি-অনুমান) error সহ একটি stream-এ আইটেম frequency অনুমান করে।

## Bloom Filter-এর কার্যপ্রণালী

| অপারেশন | কীভাবে কাজ করে |
|---|---|
| এলিমেন্ট যোগ করা | k সংখ্যক independent hash function দিয়ে hash করুন; ফলস্বরূপ প্রতিটি bit position 1 সেট করুন |
| Membership যাচাই | একইভাবে hash করুন; যদি যেকোনো bit 0 হয় → নিশ্চিতভাবে উপস্থিত নেই। যদি সব bit 1 হয় → সম্ভবত উপস্থিত (একটি false positive হতে পারে) |
| টিউনিং | বড় bit array + বেশি hash function = কম false-positive rate, তবে তার বিনিময়ে বেশি মেমরি খরচ |

## তিনটি Structure-এর তুলনা

| Structure | কোন প্রশ্নের উত্তর দেয় | Error-এর ধরন | সাধারণ ব্যবহার |
|---|---|---|---|
| Bloom filter | এই এলিমেন্টটি কি সেটে আছে? | False positive সম্ভব; false negative অসম্ভব | ব্যয়বহুল লুকআপ এড়িয়ে যাওয়া (যেমন, LSM-tree SSTable check), স্কেলে dedup, crawler-এর "URL দেখা হয়েছে?" check |
| HyperLogLog | কতগুলো ইউনিক আইটেম? | যেকোনো দিকে ~1-2% আপেক্ষিক error | ইউনিক ভিজিটর গণনা, বিশাল স্কেলে ইউনিক ইভেন্ট গণনা (Redis `PFCOUNT`) |
| Count-Min Sketch | এই আইটেমটি কতবার ঘটেছে? | শুধুমাত্র বেশি-অনুমান (কখনও কম অনুমান করে না) | Top-K / trending আইটেম, স্কেলে আনুমানিক frequency গণনা |

## HyperLogLog-এর স্বজ্ঞা (Intuition)

- N সংখ্যক leading zero bit সহ একটি hash value মোটামুটি 1-in-2^N একটি ঘটনা।
- দীর্ঘ leading zero-এর run সহ hash value পর্যবেক্ষণ করা অনেক distinct আইটেম hash করার পরিসংখ্যানগত প্রমাণ।
- HyperLogLog আগত আইটেমগুলোকে bucket-এ ভাগ করে (যেমন, 16,384টি bucket-এ) এবং প্রতি bucket-এ সর্বোচ্চ leading-zero count ট্র্যাক করে, তারপর চূড়ান্ত অনুমানের জন্য একটি নির্দিষ্ট averaging formula দিয়ে সব bucket একত্রিত করে।
- ফলাফল: প্রকৃত সংখ্যা হাজার হোক বা বিলিয়ন হোক তা নির্বিশেষে, মাত্র কয়েক KB মেমরি ব্যবহার করে ~1-2% নির্ভুল cardinality অনুমান।

## Count-Min Sketch-এর কার্যপ্রণালী

| অপারেশন | কীভাবে কাজ করে |
|---|---|
| ঘটনা রেকর্ড করা | k সংখ্যক hash function দিয়ে আইটেম hash করুন (প্রতিটি একটি ভিন্ন row-এ ম্যাপ হয়); ফলস্বরূপ প্রতিটি cell-এর counter বাড়ান |
| Frequency অনুমান | একইভাবে hash করুন; যেসব counter-এ পৌঁছান তাদের সবার মধ্যে সর্বনিম্নটি নিন (collision শুধুমাত্র একটি counter ফুলিয়ে তুলতে পারে, কখনো কমাতে পারে না, তাই সর্বনিম্নটি সঠিকের সবচেয়ে কাছাকাছি) |

## মূল সংখ্যা / তথ্য

- Bloom filter Burton Howard Bloom দ্বারা 1970 সালে প্রবর্তিত হয়েছিল।
- HyperLogLog (Flajolet et al., 2007) সাধারণত বিলিয়ন পর্যন্ত cardinality-এর জন্য প্রায় 1.5 KB মেমরি ব্যবহার করে ~1-2% standard error অর্জন করে।
- Redis নেটিভভাবে HyperLogLog বাস্তবায়ন করে (`PFADD`, `PFCOUNT`, `PFMERGE`)।

## সারাংশ

- তিনটি structure-ই নির্ভুল ট্র্যাকিংয়ের তুলনায় একটি ছোট, গাণিতিকভাবে সীমাবদ্ধ error-এর বিনিময়ে নাটকীয়, নির্দিষ্ট মেমরি সাশ্রয় করে।
- Bloom filter: membership, কোনো false negative নেই। HyperLogLog: cardinality, ~1-2% error। Count-Min Sketch: frequency, শুধুমাত্র-বেশি-অনুমান error।
- যখন একটি আনুমানিক উত্তর একটি নির্ভুল উত্তরের মতোই একই বাস্তবসম্মত ব্যবসায়িক ফলাফল দেয় তখন এগুলো ব্যবহার করুন — কখনো আর্থিক ব্যালেন্সের মতো সঠিকতা-গুরুত্বপূর্ণ নির্ভুল মানের জন্য নয়।
