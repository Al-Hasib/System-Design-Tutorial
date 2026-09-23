# কুইজ: পরবর্তী ইন্টারভিউ প্রশ্নাবলি

[README.md](README.md)-এর প্রাথমিক ডিজাইনের পর একজন ইন্টারভিউয়ার যেসব প্রশ্ন জিজ্ঞাসা করতে পারেন তার অনুশীলন। মডেল উত্তর পড়ার আগে নিজে উত্তর দেওয়ার চেষ্টা করুন।

**১. একটি ভিডিও হঠাৎ viral হয়ে যায় এবং কয়েক মিনিটের মধ্যে ট্রাফিক 100x বেড়ে যায়। আপনি এটি কীভাবে সামলাবেন?**
CDN-এর request coalescing-এর উপর নির্ভর করুন যাতে একটি edge-এ concurrent cache miss গুলো origin storage-কে stampede না করে বরং একটি একক origin fetch-এ একত্র হয়। view-velocity সংকেত ব্যবহার করে সক্রিয়ভাবে অতিরিক্ত edge cache pre-warm করুন এবং edge ও durable storage-এর মধ্যে একটি origin-shield layer যোগ করুন যা fan-in শুষে নেয়। Load balancer-এর পেছনে যেকোনো stateless read-path সার্ভিস (metadata API) autoscale করুন।

**২. ব্যবহারকারীর অভিজ্ঞতা ক্ষতিগ্রস্ত না করে কীভাবে transcoding cost কমাবেন?**
প্রতিটি ভিডিওর জন্য প্রতিটি rendition/codec সমন্বয় আগে থেকে তৈরি করবেন না। আপলোডের সময় শুধু সবচেয়ে বেশি অনুরোধ করা rendition গুলো (যেমন, H.264-এ 360p/720p) pre-transcode করুন, এবং বিরল গুলো (4K, AV1) প্রথম অনুরোধে lazily তৈরি করুন, ফলাফল পরে cache করে রাখুন। এছাড়াও transcoding queue-কে prioritize করুন যাতে জনপ্রিয় বা trending আপলোড আগে প্রসেস হয়।

**৩. Video-on-demand (VOD)-এর পাশাপাশি live streaming কীভাবে সাপোর্ট করবেন?**
Live streaming VOD-এর কিছু গ্যারান্টি ছেড়ে দিয়ে খুব কম end-to-end latency পায়: একটি সম্পূর্ণ ফাইল transcode করার পরিবর্তে, আপনি একটি ক্রমাগত stream ছোট segment-এ (প্রতিটি কয়েক সেকেন্ড) transcode করেন যেভাবে এটি আসে, একটি ক্রমাগত-বর্ধমান manifest প্রকাশ করেন, এবং সাথে সাথে segment গুলো CDN edge-এ পুশ করেন। পরে re-transcode করার জন্য কোনো "raw master" থাকে না, retry budget আরও কড়া হয়, এবং প্রোটোকল প্রায়ই low-latency HLS/DASH extension বা sub-second latency use case-এর জন্য WebRTC-কে পছন্দ করে।

**৪. কোন rendition গুলো pre-generate করবেন এবং কোনগুলো on demand তৈরি করবেন তা কীভাবে সিদ্ধান্ত নেবেন?**
পর্যবেক্ষিত request distribution-এর উপর ভিত্তি করে সিদ্ধান্ত নিন: প্রতি ডিভাইস/নেটওয়ার্ক মিশ্রণে আসলে কোন resolution/codec অনুরোধ করা হচ্ছে তা instrument করুন, এবং সেই rendition গুলো pre-generate করুন যা বেশিরভাগ playback অনুরোধ কভার করে (প্রায়ই H.264-এ 360p/480p/720p বেশিরভাগ কভার করে)। Long-tail rendition (4K, পুরনো codec, অস্বাভাবিক aspect ratio) on-demand তৈরি করা হয় এবং cache করা হয়, যেহেতু বেশিরভাগ ভিডিও কখনো সেই সেটিংসে দেখা হয় না।

**৫. বড় ভিডিও ফাইলের জন্য আপলোড resumability কীভাবে সামলাবেন?**
Upload-কে একটি resumable upload প্রোটোকল সহ fixed-size chunk-এ বিভক্ত করুন (যেমন, একটি upload session ID প্লাস প্রতি-chunk checksum/offset, S3 multipart upload API বা tus প্রোটোকলের মতো)। সংযোগ বিচ্ছিন্ন হলে, ক্লায়েন্ট জিজ্ঞাসা করে সার্ভারের কাছে ইতিমধ্যে কোন chunk গুলো আছে এবং পুরো ফাইল আবার শুরু করার পরিবর্তে পরবর্তী অনুপস্থিত chunk থেকে পুনরায় শুরু করে।

**৬. Netflix-এর মতো একটি প্ল্যাটফর্মের জন্য কোন DRM/content protection বিবেচনা গুরুত্বপূর্ণ?**
লাইসেন্সযুক্ত কনটেন্টের জন্য সাধারণত segment এনক্রিপ্ট করতে হয় (যেমন, HLS-এর জন্য AES-128 বা DASH-এর জন্য Common Encryption/CENC) এবং DRM সিস্টেমের (Widevine, PlayReady, FairPlay) সাথে একীভূত করতে হয় যাতে শুধু অনুমোদিত, লাইসেন্সপ্রাপ্ত player গুলো কনটেন্ট decrypt এবং playback করতে পারে। এতে playback-এর আগে একটি license-server round trip যোগ হয় এবং key rotation strategy প্রয়োজন হয়, কিন্তু user-generated কনটেন্টের বদলে লাইসেন্সযুক্ত/premium কনটেন্টের জন্য এটি একটি কঠোর প্রয়োজনীয়তা।

**৭. একটি write bottleneck তৈরি না করে view counter কীভাবে নির্ভুল রাখবেন?**
প্রতি view-তে primary metadata store-এ একটি row synchronously increment করবেন না। এর পরিবর্তে, view event গুলো একটি streaming pipeline-এ পাঠান, সেগুলোকে সময়ের window-এ (যেমন, প্রতি কয়েক সেকেন্ডে) aggregate করুন, এবং batched increment গুলো sharded metadata store-এ flush করুন। এটি hot ভিডিওতে অনেক গুণ কম write-এর বিনিময়ে সামান্য counter staleness মেনে নেয়।

**৮. Startup latency (প্রথম frame-এর সময়) কীভাবে কমাবেন?**
প্রাথমিক segment গুলো ছোট রাখুন যাতে playback শুরু হওয়ার আগে player-কে একটি রক্ষণশীল (কম/মাঝারি) bitrate-এর মাত্র কয়েক সেকেন্ড ডাউনলোড করতে হয়। নিকটতম CDN edge থেকে manifest এবং প্রথম segment সার্ভ করুন, এবং predictive pre-fetching বিবেচনা করুন (যেমন, একজন ব্যবহারকারী একটি ভিডিওর উপর hover করলে বা select করলে manifest এবং প্রথম segment pre-fetch করা) নেটওয়ার্ক round-trip time লুকাতে।

**৯. Encoding-এর জন্য H.264, VP9, এবং AV1-এর মধ্যে কীভাবে নির্বাচন করবেন?**
H.264-এর সার্বজনীন hardware decode সাপোর্ট আছে, তাই এটি compatibility-র জন্য একটি প্রয়োজনীয় baseline, বিশেষত পুরনো ডিভাইসে। VP9 এবং AV1 উল্লেখযোগ্যভাবে ভালো compression দেয় (সমান quality-তে কম bitrate), যা স্কেলে storage এবং CDN egress cost কমায়, কিন্তু encode করতে বেশি CPU খরচ হয় এবং কম সার্বজনীন hardware decode সাপোর্ট থাকে। বিশাল egress cost-সহ প্ল্যাটফর্মগুলো (যেমন YouTube/Netflix) একাধিক codec-এ encode করে এবং ক্লায়েন্ট যেটি সাপোর্ট করে তার মধ্যে সেরাটি সার্ভ করে।

**১০. Database sharding strategy কীভাবে "একটি channel-এর সব ভিডিও তালিকাভুক্ত করুন"-এর মতো metadata query-কে প্রভাবিত করে?**
যদি আপনি শুধুমাত্র `videoId` hash দিয়ে shard করেন, তাহলে একটি channel-এর সব ভিডিও তালিকাভুক্ত করতে shard জুড়ে একটি scatter-gather query প্রয়োজন হয়, যা ধীর। একটি সাধারণ প্রতিকার হলো একটি secondary index বা একটি আলাদা mapping table (channelId -> videoId-এর তালিকা) বজায় রাখা যা নিজেই `channelId` দিয়ে shard করা, যাতে channel-scoped query একটি একক shard-এ আঘাত করে, অন্যদিকে `videoId` দিয়ে পৃথক ভিডিও lookup প্রাথমিক sharding scheme ব্যবহার করে।

**১১. একটি transcoding worker কাজের মাঝপথে crash করলে কী হয়?**
যেহেতু pipeline queue-driven, transcoding এবং storage write সফল না হওয়া পর্যন্ত job message acknowledge/commit হয় না। যদি একটি worker crash করে, message আবার দৃশ্যমান হয়ে যায় (queue visibility timeout বা consumer group rebalance-এর মাধ্যমে) এবং অন্য একটি worker সেটি তুলে নেয়। Transcoding idempotent হওয়া উচিত (একটি unique output path-এ লেখা, বা deterministically overwrite করা) যাতে একটি retry হওয়া job state নষ্ট না করে।

**১২. একাধিক audio track বা subtitle কীভাবে সাপোর্ট করবেন?**
এগুলোকে অতিরিক্ত, স্বাধীনভাবে-encode করা এবং স্বাধীনভাবে-cache করা stream হিসেবে বিবেচনা করুন যা একই manifest দ্বারা রেফারেন্স করা হয় — HLS/DASH manifest স্বাভাবিকভাবেই video rendition-এর পাশাপাশি একাধিক audio/subtitle track সাপোর্ট করে, এবং player video bitrate পরিবর্তনের মতোই track নির্বাচন/পরিবর্তন করে, একটি আলাদা delivery pipeline ছাড়াই।
</content>
