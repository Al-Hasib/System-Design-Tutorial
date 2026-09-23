# Study Notes: Database Indexing

## সংজ্ঞা (Definitions)

- **Index** — একটি table-এর এক বা একাধিক column-এর ওপর তৈরি একটি আলাদা, সংগঠিত data structure যা column value-কে সংশ্লিষ্ট row-এর physical অবস্থানে map করে, যা database-কে পুরো table স্ক্যান করা এড়াতে সাহায্য করে।
- **Full table scan (sequential scan)** — মিল খুঁজতে একটি table-এর প্রতিটি row পড়া; একটি index যা এড়াতে সাহায্য করে।
- **B-Tree / B+Tree** — একটি balanced, sorted tree structure। একটি B+Tree-তে, সব data pointer linked, sorted leaf node-এ থাকে, যা দক্ষ range scan সম্ভব করে। বেশিরভাগ relational database-এর (PostgreSQL, MySQL InnoDB, SQL Server, Oracle) default index type।
- **Hash index** — একটি index যা একটি hash function-এর মাধ্যমে একটি value-কে একটি bucket-এ map করে, গড়ে O(1) exact-match lookup দেয় কিন্তু কোনো ordering দেয় না।
- **Cardinality** — row সংখ্যার তুলনায় একটি column-এ থাকা distinct value-এর সংখ্যা। High cardinality (যেমন, email, user ID) indexing থেকে low cardinality (যেমন, boolean flag)-এর চেয়ে বেশি উপকার পায়।
- **Composite (multi-column) index** — একটি নির্দিষ্ট column ক্রমে একাধিক column-এর ওপর তৈরি একটি index।
- **Covering index** — একটি index যাতে একটি query-এর প্রয়োজনীয় সব column থাকে, ফলে query table-এ আলাদা lookup ছাড়াই সম্পূর্ণভাবে index থেকেই সন্তুষ্ট করা যায় (কখনো কখনো একে "index-only scan" বলা হয়)।
- **Write amplification** — underlying row পরিবর্তন হলেই প্রতিটি index update করতে হওয়ার কারণে সৃষ্ট অতিরিক্ত write কাজ।

## B-Tree বনাম Hash Index

| দিক | B-Tree (B+Tree) | Hash Index |
|---|---|---|
| Lookup type | Exact match এবং range | শুধুমাত্র Exact match |
| Range query (`<`, `>`, `BETWEEN`) | হ্যাঁ — linked sorted leaf-এর মাধ্যমে দক্ষ | না — সব entry স্ক্যান করতে হয় |
| Ordering / `ORDER BY` সমর্থন | হ্যাঁ — data sorted অবস্থায় সংরক্ষিত থাকে | না — hashing order নষ্ট করে দেয় |
| Prefix মিল (`LIKE 'abc%'`) | হ্যাঁ | না |
| Time complexity (lookup) | O(log n) | গড়ে O(1) |
| Insert/update/delete খরচ | O(log n), সাথে মাঝে মাঝে node split/rebalancing | গড়ে O(1), মাঝে মাঝে bucket resizing/rehashing |
| Storage overhead | মাঝারি | সাধারণত প্রতি entry-তে কম, কিন্তু hash collision অতিরিক্ত overhead যোগ করে |
| সাধারণ ব্যবহারক্ষেত্র | Primary key, foreign key, sorting, range filter, general-purpose query | Pure equality lookup (cache key, exact ID lookup) |
| উদাহরণ | PostgreSQL default index, MySQL InnoDB default index, SQL Server clustered/nonclustered index | PostgreSQL `USING hash` index, Redis-এর internal hash table, in-memory hash map (Python dict, Java `HashMap`) |

## Composite Index

- একাধিক column-এর ওপর একটি index, যেমন `(last_name, first_name)`।
- Column-এর **ক্রম গুরুত্বপূর্ণ**: index দক্ষভাবে column-গুলোর একটি **left-prefix**-এর ওপর filter করা query সমর্থন করে:
  - `WHERE last_name = 'Smith'` — দক্ষ (index ব্যবহার করে)।
  - `WHERE last_name = 'Smith' AND first_name = 'Jane'` — দক্ষ (সম্পূর্ণ index ব্যবহার করে)।
  - শুধুমাত্র `WHERE first_name = 'Jane'` — এই index দক্ষভাবে ব্যবহার করতে পারে না (এটি left-prefix নয়)।
- এমন query-র জন্য উপযোগী যা একটি column-এ filter করে এবং অন্য একটিতে sort করে, যেমন `(customer_email, created_at DESC)`।

## Covering Index

- একটি query দ্বারা উল্লেখিত প্রতিটি column অন্তর্ভুক্ত করে (`SELECT`, `WHERE`, এবং `ORDER BY` clause-এ)।
- Database-কে একটি "index-only scan" করতে দেয় — এটি কখনো table heap থেকে সম্পূর্ণ row fetch করতে হয় না।
- Read-heavy, latency-sensitive query-র জন্য চমৎকার, খরচ হলো একটি বড় index (আরও বেশি column সংরক্ষিত)।

## গুরুত্বপূর্ণ সংখ্যা / Complexity

- B-Tree lookup, insert, delete: **O(log n)** — লক্ষ লক্ষ row-বিশিষ্ট একটি table-এর জন্য, এটি সাধারণত মাত্র ৩-৪টি tree level অতিক্রম করে।
- Hash index lookup: গড়ে **O(1)** (ভারী hash collision-এর ক্ষেত্রে খারাপ হতে পারে)।
- Full table scan (কোনো index ছাড়া): **O(n)** — প্রতিটি row পরীক্ষা করতে হয়।
- বেশি index = বেশি write খরচ: প্রতিটি index indexed column-এ প্রতিটি `INSERT`/`UPDATE`/`DELETE`-এ প্রায় একটি অতিরিক্ত write যোগ করে।

## কোনটি কখন ব্যবহার করবেন — সারাংশ

- **একটি B-Tree ব্যবহার করুন** (নিরাপদ default) যখন আপনার প্রয়োজন: range query, sorting, prefix matching, বা একই column-এ মিশ্র query pattern। এটি বেশিরভাগ application query-কে কভার করে।
- **একটি hash index ব্যবহার করুন** যখন: access pattern সম্পূর্ণভাবে equality lookup (কখনো range নয়, কখনো sorting নয়), এবং আপনি দ্রুততম সম্ভব exact-match performance চান — সাধারণত caching layer এবং in-memory key-value lookup-এ, একটি general relational index হিসেবে নয়।
- **একটি composite index ব্যবহার করুন** যখন query-গুলো ধারাবাহিকভাবে একই column combination-এর ওপর একসাথে filter/sort করে; সবচেয়ে-selective/সবচেয়ে-বেশি-filter-করা থেকে সবচেয়ে কম-এর দিকে column সাজান।
- **একটি covering index ব্যবহার করুন** hot, latency-critical read query-র জন্য যেখানে অতিরিক্ত table lookup এড়ানো গুরুত্বপূর্ণ।
- **Indexing এড়িয়ে চলুন** write-heavy table, ছোট table, এবং low-cardinality column-এ আগ্রাসীভাবে — overhead প্রায়ই সামান্য read গতির তুলনায় মূল্যবান নয়।
