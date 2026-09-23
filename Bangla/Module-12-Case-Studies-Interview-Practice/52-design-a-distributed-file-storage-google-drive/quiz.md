# Quiz: Follow-up Interview Questions

মূল design-এর পরে interviewer যেসব প্রশ্ন জিজ্ঞেস করতে পারেন সেগুলোর practice। Model answer-গুলো ইচ্ছাকৃতভাবে সংক্ষিপ্ত -- একটি real interview-তে সেগুলো গলায় বিস্তারিত বলুন।

**১. দুটো device concurrent-ভাবে একই file edit করলে আপনি কীভাবে সামলাবেন, বিশেষ করে যদি একটি offline ছিল?**
Metadata service দ্বারা track করা file version number (বা vector clock) ব্যবহার করে conflict শনাক্ত করুন -- যদি একটি device একটি নতুন version commit করার চেষ্টা করে যার "based on" version বর্তমান head-এর সাথে আর মেলে না, সেটাই conflict। নির্বিচারে binary content-এর একটি অনিরাপদ automatic merge করার চেষ্টা না করে, উভয় version রাখুন: জয়ী write (যেমন, timestamp দিয়ে last-writer-wins)-কে canonical file হিসেবে apply করুন, এবং অন্যটিকে user manually reconcile করার জন্য একটি "conflicted copy" হিসেবে save করুন, Dropbox-এর approach-এর মতো।

**২. আপনি কীভাবে efficient delta sync implement করবেন যাতে একটি ছোট edit-এর পর পুরো file আবার upload করতে না হয়?**
File-গুলোকে fixed-size chunk-এ (যেমন, 4 MB) ভাগ করুন এবং প্রতিটি chunk hash করুন। Edit-এর সময়, শুধু যেসব chunk-এর content আসলে পরিবর্তিত হয়েছে সেগুলোর নতুন hash হয়; client তার local chunk-hash list-কে শেষবার sync হওয়া version-এর সাথে diff করে এবং শুধু ভিন্ন chunk-গুলোই upload/download করে। এটাই rsync-এর rolling-checksum delta encoding-এর একই principle, fixed-size, content-addressed chunk-এর জন্য অভিযোজিত।

**৩. বিভিন্ন user-এর upload করা identical file আপনি কীভাবে deduplicate করবেন?**
যেহেতু chunk-গুলো তাদের hash দিয়ে keyed একটি content-addressable store-এ store থাকে, যদি দুইজন user byte-identical file (বা common chunk share করা file) upload করে, chunk hash-গুলো বিদ্যমান entry-র সাথে মিলে যায় এবং upload সম্পূর্ণভাবে skip হয়ে যায় -- metadata service শুধু বিদ্যমান chunk-গুলোতে একটি reference যোগ করে। এটি user-দের মধ্যে transparent-ভাবে কাজ করে কারণ store global, per-user নয়।

**৪. একটি অবিশ্বস্ত network connection-এর উপর 50 GB file upload আপনি কীভাবে সামলাবেন?**
Client-side-এ file chunk করুন এবং প্রতিটি chunk আলাদাভাবে per-chunk acknowledgment সহ upload করুন। Client কোন chunk acknowledge হয়েছে তা persist করে রাখে; disconnect/retry-এর সময়, এটি পুরো transfer আবার শুরু না করে শেষ unacknowledged chunk থেকে resume করে। চূড়ান্ত "commit" ধাপ (metadata service-এ ordered chunk-hash list লেখা) শুধু তখনই ঘটে যখন সব chunk store হয়েছে বলে নিশ্চিত হয়, তাই একটি আংশিক upload কখনো corrupt "complete" file হয়ে যায় না।

**৫. আপনি sharing/permission model কীভাবে design করবেন?**
Metadata service-এ প্রতি file/folder-এর একটি access-control list (ACL) store করুন: (principal, role) জোড়ার একটি list, যেখানে role হলো owner/editor/viewer। Folder share-এর ক্ষেত্রে, permission tree-র নিচের দিকে inherit হয় যদি না স্পষ্টভাবে override করা হয়, read time-এ নিকটতম explicit grant পর্যন্ত উপরে গিয়ে resolve করা হয়। Permission check chunk-hash list ফেরত দেওয়ার বা upload commit অধিকার দেওয়ার আগে metadata service-এ ঘটে, তাই blob storage নিজেই permission-agnostic থাকে।

**৬. Storage cost বিস্ফোরিত না হয়ে আপনি কীভাবে file version history পরিচালনা করবেন?**
যেহেতু storage chunk-based, একটি নতুন version সাধারণত শুধু অল্প কিছু নতুন chunk (পরিবর্তিত অংশ) যোগ করে যখন আগের version থেকে অপরিবর্তিত chunk reference counting-এর মাধ্যমে পুনরায় ব্যবহার করা হয় -- তাই N version store করা N-টি পূর্ণ copy-র চেয়ে অনেক কম খরচ করে। এর উপরে, retention policy প্রয়োগ করুন: retained version-এর সংখ্যা বা তাদের retention window (যেমন, 30-180 দিন) সীমিত করুন, এবং paid tier-কে দীর্ঘ history রাখতে দিন।

**৭. এই system-এ strong এবং eventual consistency-র মধ্যে trade-off কী, এবং কোথায় কোনটা প্রয়োগ করবেন?**
CAP/PACELC অনুযায়ী: metadata-র জন্য (file tree, permission, "বর্তমান version কী"), আমরা Raft/Paxos-backed write-এর মাধ্যমে strong consistency বেছে নিই, কারণ file tree-র একটি inconsistent view data loss বা stale permission expose করার অর্থ হতে পারে। Blob/chunk storage-এর জন্য, আমরা eventual consistency বেছে নিই, কারণ chunk immutable এবং content-addressed -- পড়ার জন্য কোনো "stale value" নেই, শুধু একটি chunk সব জায়গায় available হওয়ার আগে replication lag আছে -- তাই আমরা এর বদলে availability এবং write throughput-কে অগ্রাধিকার দিই।

**৮. Exabyte scale-এ আপনি কীভাবে storage cost কমাবেন?**
তিনটি lever: (১) content-addressable chunk storage-এর মাধ্যমে deduplication, যা identical content-এর redundant copy দূর করে; (২) storage tiering, ~90 দিন access না হওয়া file সস্তা cold/archival storage class-এ move করা; (৩) version-history retention সীমাবদ্ধ করা যাতে পুরনো, superseded chunk-গুলো, কোনো live version আর তাদের reference না করলে, একসময় garbage-collect করা যায়।

**৯. একটি client কীভাবে জানে কখন ক্রমাগত poll না করে update fetch করতে হবে?**
একটি version commit হলেই metadata service একটি event (যেমন, `file.updated`) publish করে, event-driven pub/sub architecture ব্যবহার করে। Notification service subscribe করে এবং একটি persistent connection (WebSocket/long-poll)-এর মাধ্যমে user-এর অন্যান্য online device-এ একটি হালকা "কিছু পরিবর্তন হয়েছে, metadata check করো" signal push করে, যারপর সেই device-গুলো শুধু metadata diff এবং missing chunk pull করে। এটি push-based এবং near-real-time, ক্রমাগত polling-এর চেয়ে অনেক সস্তা।

**১০. একটি upload acknowledge করার আগে প্রতিটি chunk প্রতিটি region-এ synchronously replicate করলেই বা কী সমস্যা?**
এটা প্রতি write-এ durability guarantee সর্বোচ্চ করবে কিন্তু availability এবং latency ধ্বংস করবে -- একটি ধীর বা unreachable replica প্রতিটি upload block করে দেবে। এর বদলে, durability-র জন্য একটি quorum write (যেমন, 3-এর মধ্যে 2 replica)-এর পর acknowledge করুন, এবং বাকি region/replica-তে asynchronously replicate করুন, availability এবং throughput-এর বিনিময়ে blob layer-এর জন্য সংক্ষিপ্ত eventual consistency মেনে নিন -- chunk storage-কে system-এর "AP-leaning" দিক হিসেবে treat করার সাথে সামঞ্জস্যপূর্ণ।

**১১. আপনি ধীর "list folder" operation ছাড়াই খুব বড় folder (যেমন, 100,000 file) কীভাবে সমর্থন করবেন?**
Metadata service-এ folder listing paginate করুন এবং (parent_folder_id, name) দিয়ে একটি covering index সহ index করুন, যাতে একটি listing query একটি shard-এ একটি bounded range scan হয় (যেহেতু shard key user/owner ID, একজন user-এর পুরো folder tree সাধারণত একটি shard-এ থাকে)। একটি listing call-এ পূর্ণ chunk-hash list ফেরত দেওয়া এড়িয়ে চলুন -- শুধু তখনই সেগুলো ফেরত দিন যখন একটি নির্দিষ্ট file open/sync করা হয়।

**১২. এই design real-time collaborative co-editing (Google Docs-এর মতো) যোগ করলে কীভাবে পরিবর্তিত হবে?**
Character-by-character concurrent edit-এর জন্য file-level chunk sync অনেক বেশি coarse। আপনার একটি operation-based model দরকার হবে -- Operational Transformation (OT) বা CRDT -- যেখানে sync-এর একক একটি edit operation, একটি file chunk নয়, এবং একটি central sequencing service (বা peer-to-peer CRDT merge) conflicted copy-তে fallback না করে automatically concurrent operation resolve করে। এই design-এর metadata/versioning এবং storage-tiering অংশগুলো তখনও underlying document snapshot এবং history-র জন্য প্রযোজ্য থাকবে।
