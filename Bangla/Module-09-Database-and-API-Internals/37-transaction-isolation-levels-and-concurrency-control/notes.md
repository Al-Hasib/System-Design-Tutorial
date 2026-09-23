# Study Notes: Transaction Isolation Levels & Concurrency Control

## Definitions

- **Dirty read:** অন্য কোনো transaction-এর uncommitted (এবং সম্ভবত পরে rollback হয়ে যাওয়া) পরিবর্তন read করা।
- **Non-repeatable read:** একটি transaction-এর মধ্যে একই row দুইবার read করে ভিন্ন value পাওয়া, কারণ মাঝখানে অন্য একটি transaction একটি পরিবর্তন commit করেছে।
- **Phantom read:** একই query পুনরায় চালিয়ে ভিন্ন সেট row পাওয়া, কারণ মাঝখানে অন্য একটি transaction মিলে যাওয়া row insert/delete করেছে।
- **Pessimistic locking:** কোনো ডেটা স্পর্শ করার আগেই সেটির ওপর একটি lock নেওয়া, যা অন্য transaction-গুলোকে অপেক্ষা করতে বাধ্য করে।
- **Optimistic Concurrency Control (OCC):** কোনো lock ছাড়াই এগিয়ে যাওয়া, শুধু commit-এর সময় conflict চেক করা (সাধারণত একটি version/timestamp তুলনার মাধ্যমে), এবং conflict হলে retry করা।
- **MVCC (Multi-Version Concurrency Control):** প্রতিটি row-এর একাধিক version রাখা যাতে reader-রা concurrent writer-দের block না করেই একটি consistent snapshot দেখতে পায়।
- **Deadlock:** দুই (বা ততোধিক) transaction প্রত্যেকেই এমন একটি lock ধরে রাখে যা অন্যটির প্রয়োজন, এবং কেউই এগোতে পারে না; database এটি detect করে একটিকে abort করে।

## The Four SQL Isolation Levels

| Level | Dirty reads | Non-repeatable reads | Phantom reads | Notes |
|---|---|---|---|---|
| Read Uncommitted | সম্ভব | সম্ভব | সম্ভব | সবচেয়ে দুর্বল; খুব কমই ব্যবহৃত হয় |
| Read Committed | প্রতিরোধ করা হয় | সম্ভব | সম্ভব | PostgreSQL, SQL Server, Oracle-এ default |
| Repeatable Read | প্রতিরোধ করা হয় | প্রতিরোধ করা হয় | সম্ভব (স্পেক অনুযায়ী; কিছু engine যেকোনোভাবে প্রতিরোধ করে) | MySQL/InnoDB-এ default |
| Serializable | প্রতিরোধ করা হয় | প্রতিরোধ করা হয় | প্রতিরোধ করা হয় | সবচেয়ে শক্তিশালী; সর্বোচ্চ coordination খরচ, সবচেয়ে বেশি abort/retry |

## Locking vs. Optimistic Concurrency Control

| | Pessimistic Locking | Optimistic Concurrency Control |
|---|---|---|
| ধারণা | Conflict হওয়ার সম্ভাবনা বেশি | Conflict বিরল |
| প্রক্রিয়া | কাজ করার আগে lock নেওয়া | স্বাধীনভাবে কাজ করা, commit-এর সময় version/timestamp চেক করা |
| Conflict বিরল হলে খরচ | অপচয়িত অপেক্ষা/contention | ন্যূনতম — সস্তা চেক, বিরল retry |
| Conflict সাধারণ হলে খরচ | প্রয়োজনীয় — অপচয়িত কাজ প্রতিরোধ করে | ব্যয়বহুল — ঘন ঘন abort ও retry |
| ঝুঁকি | Deadlock (detection/resolution প্রয়োজন) | Application-level retry logic প্রয়োজন |
| উপযুক্ত ক্ষেত্র | Hot, high-contention row (যেমন, inventory count) | সাধারণ application-এ বেশিরভাগ row, বেশিরভাগ সময় |

## MVCC in Practice

- প্রতিটি row-এর একাধিক version থাকতে পারে, যেগুলোকে সেই transaction দিয়ে ট্যাগ করা হয় যা সেগুলো তৈরি করেছে।
- একটি reader তার নিজের transaction/statement শুরুর সময়ের সাথে সামঞ্জস্যপূর্ণ version দেখে — কখনো একটি concurrent writer-এর জন্য block হয় না।
- Writer-দের এখনও একই row-এর অন্য writer-দের সাথে coordinate করতে হয় (দুটি conflicting "বিজয়ী" থাকতে পারে না)।
- কোনো active transaction-এর আর প্রয়োজন না থাকলে পুরোনো version-গুলো পরিষ্কার করতে হয় (যেমন, PostgreSQL-এর `VACUUM`); এটি উপেক্ষা করলে table bloat ঘটে।
- MVCC-এর অধীনেও, hot/high-stakes row-গুলোর জন্য এখনও স্পষ্ট pessimistic locking প্রয়োজন হতে পারে (যেমন, `SELECT ... FOR UPDATE`) যেখানে একটি race সত্যিই সহ্য করা যায় না।

## Key Numbers / Facts

- PostgreSQL-এর default isolation level হলো Read Committed; MySQL/InnoDB-এর default হলো Repeatable Read।
- `SELECT ... FOR UPDATE` হলো একটি MVCC database-এর মধ্যে pessimistic row-level locking জোর করার standard SQL পদ্ধতি।
- PostgreSQL-এ Serializable isolation implement করা হয় Serializable Snapshot Isolation (SSI)-এর মাধ্যমে, যা সম্পূর্ণভাবে block করার বদলে conflict detect করে এবং transaction abort করে।

## Summary

- সুরক্ষা ছাড়া concurrency নির্দিষ্ট, সুপরিচিত anomaly তৈরি করে: dirty read, non-repeatable read, phantom read।
- Isolation level কোন anomaly প্রতিরোধ করা হবে তা throughput/coordination খরচের বিনিময়ে trade-off করে — সচেতনভাবে বেছে নিন, framework-এর default-এ না পরীক্ষা করে রেখে দেবেন না।
- Pessimistic locking এবং optimistic concurrency control হলো isolation কার্যকর করার দুটি engineering কৌশল; MVCC-ই হলো যেভাবে বেশিরভাগ বাস্তব database row versioning-এর মাধ্যমে reader/writer blocking এড়িয়ে দক্ষতার সাথে এটি implement করে।
