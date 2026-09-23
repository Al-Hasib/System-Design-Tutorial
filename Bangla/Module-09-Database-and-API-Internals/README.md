# Module 9: Database & API Internals

Module 3 আপনাকে শিখিয়েছে *কখন* একটা relational database বনাম NoSQL store বেছে নেবেন, এবং Module 8 covered করেছে service-গুলো কীভাবে wire format নিয়ে একমত হয়। এই module উভয় দিকেই আরও এক ধাপ গভীরে যায়। Database দিক থেকে, আমরা খুলে দেখব "ACID" আর "index" storage-engine level-এ আসলে কী মানে: একটা database কীভাবে concurrent transaction-গুলোর মধ্যে isolation বলবৎ করে, এবং underlying data structure — B-tree নাকি LSM-tree — কীভাবে ঠিক করে একটা database read-এর জন্য নাকি write-এর জন্য optimized। API দিক থেকে, আমরা দেখব GraphQL-কে REST-এর একটা genuine alternative হিসেবে, শুধু একটা buzzword হিসেবে না, এবং বুঝব ঠিক কোন সমস্যা এটা সমাধান করে যা REST structurally পারে না। এসবের কোনোটাই একটা beginner system design interview pass করার জন্য দরকার নেই, কিন্তু এটাই ঠিক সেই ধরনের depth যা "আমি vocabulary জানি" আর "আমি বুঝি কেন এই vocabulary আছে"-এর মধ্যে পার্থক্য তৈরি করে।

## এই Module-এর Video সমূহ

| # | Title | Description | Link |
|---|-------|-------------|------|
| 37 | Transaction Isolation Levels & Concurrency Control: Locking vs. MVCC | দুইটা transaction একইসাথে একই row-তে হাত দিলে আসলে কী ঘটে — isolation level, locking, এবং MVCC কীভাবে reader-কে writer-এর কাছে block হওয়া থেকে আটকায়। | [37-transaction-isolation-levels-and-concurrency-control](37-transaction-isolation-levels-and-concurrency-control/README.md) |
| 38 | LSM Trees vs. B-Trees: Storage Engine Internals | আপনার database-এর index-এর নিচে থাকা data structure, এবং কেন write-heavy NoSQL store traditional relational database-এর চেয়ে একেবারে ভিন্ন একটা structure বেছে নেয়। | [38-lsm-trees-vs-b-trees-storage-engine-internals](38-lsm-trees-vs-b-trees-storage-engine-internals/README.md) |
| 39 | GraphQL: A Query-Based Alternative to REST | Over-fetching/under-fetching সমস্যা যা REST structurally সমাধান করতে পারে না, এবং GraphQL-এর single, client-driven query endpoint কীভাবে এটা সমাধান করে — সাথে এর নিজস্ব real trade-off। | [39-graphql-a-query-based-alternative-to-rest](39-graphql-a-query-based-alternative-to-rest/README.md) |
