# অনুশীলন ও ইন্টারভিউ প্রশ্ন

**১. Event-driven architecture কীভাবে একটি প্রথাগত request-driven (synchronous API call) architecture থেকে ভিন্ন?**
Request-driven architecture-এ, একটি service অন্য service-এর API call করে এবং response-এর জন্য block/wait করে, যা tight temporal coupling তৈরি করে। Event-driven architecture-এ, একটি service একটি fact বর্ণনা করে event publish করে ("X ঘটেছে") এবং এগিয়ে যায়; যেকোনো সংখ্যক অন্যান্য services স্বাধীনভাবে এবং asynchronously প্রতিক্রিয়া দেখায়, producer এবং consumer-এর availability-র মধ্যে কোনো সরাসরি dependency ছাড়াই।

**২. Events সাধারণত past tense-এ কেন নামকরণ করা হয় (যেমন, "PlaceOrder"-এর বদলে "OrderPlaced")?**
কারণ একটি event ইতিমধ্যে ঘটে যাওয়া কিছুর একটি immutable fact প্রতিনিধিত্ব করে, কোনো request বা করার জন্য command নয়। "PlaceOrder" এমন একটি নির্দেশনা বোঝায় যা receiver-কে মানতে হবে; "OrderPlaced" শুধু একটি fact বলে যাতে listeners ঐচ্ছিকভাবে প্রতিক্রিয়া দেখাতে পারে, যা commands এবং events-এর মধ্যে মূল semantic পার্থক্য।

**৩. Event notification এবং event-carried state transfer-এর মধ্যে পার্থক্য ব্যাখ্যা করুন।**
Event notification একটি ন্যূনতম event পাঠায় (প্রায়ই শুধু একটি ID) এবং আশা করে consumer সম্পূর্ণ বিস্তারিত জানতে source service-এর API-তে call back করবে, যা events ছোট রাখে কিন্তু producer-এর availability-র উপর কিছু runtime coupling সংরক্ষণ করে। Event-carried state transfer একটি consumer-এর প্রয়োজনীয় সব data সরাসরি event payload-এ embed করে, callback dependency দূর করে বড়, সম্ভাব্য duplicate data-র বিনিময়ে।

**৪. Event sourcing কী, এবং এর একটি প্রধান সুবিধা এবং একটি প্রধান খরচ কী?**
Event sourcing একটি entity-র বর্তমান state-এ পৌঁছানো সম্পূর্ণ events-এর sequence সংরক্ষণ করে (একটি ledger-এর মতো) শুধু বর্তমান state সংরক্ষণ করার বদলে, সেই events replay করে বর্তমান state derive করে। সুবিধা: সম্পূর্ণ audit trail এবং যেকোনো সময়ের state পুনর্গঠনের ক্ষমতা। খরচ: বর্তমান state দক্ষতার সাথে query করার এবং tooling-এর জন্য উল্লেখযোগ্যভাবে বেশি জটিলতা (প্রায়ই CQRS এবং snapshotting প্রয়োজন)।

**৫. "Eventual consistency" কী এবং কেন এটি event-driven সিস্টেমের একটি অন্তর্নিহিত বৈশিষ্ট্য?**
Eventual consistency মানে হলো একটি event publish হওয়ার পরে, একটি সময়ের window থাকে যেখানে সিস্টেমের বিভিন্ন অংশ একই underlying data-র ভিন্ন, সাময়িকভাবে অসামঞ্জস্যপূর্ণ দৃষ্টিভঙ্গি ধরে রাখতে পারে, কারণ consumers events asynchronously এবং নিজেদের গতিতে process করে। এটি EDA-এর অন্তর্নিহিত কারণ সময়ে producers এবং consumers-কে decouple করা অনিবার্যভাবে সেই নিশ্চয়তা সরিয়ে দেয় যে একটি event-এর সব প্রভাব সর্বত্র তাৎক্ষণিকভাবে এবং atomically ঘটবে।

**৬. পরিস্থিতি: একজন customer support agent একটি গ্রাহক order প্লেস করার সাথে সাথেই সেটি দেখেন এবং "confirmed"-এর বদলে "processing" দেখতে পান, যদিও payment প্রকৃতপক্ষে এক মুহূর্ত আগেই সফল হয়েছে। কী ঘটছে, এবং আপনি এটি একজন non-technical stakeholder-কে কীভাবে ব্যাখ্যা করবেন?**
এটি eventual consistency window-এর একটি বাস্তব উদাহরণ: payment confirmation event প্রকাশ হয়ে গেছে কিন্তু order-status service এখনও এটি process করে তার view update করেনি। আপনি ব্যাখ্যা করতে পারেন যে সিস্টেমের বিভিন্ন অংশ খুবই সংক্ষিপ্ত বিলম্বে (প্রায়ই মিলিসেকেন্ড) স্বাধীনভাবে update হয়, এবং UI একটি "processing" indicator দেখাতে পারে বা ব্যবহারকারীর জন্য সেই ফাঁক পূরণ করতে একটি সংক্ষিপ্ত polling/refresh mechanism যোগ করতে পারে।

**৭. Synchronous, request-driven সিস্টেমের তুলনায় event-driven সিস্টেমে debugging কেন কঠিন, এবং কী tools এটি প্রশমিত করে?**
একটি synchronous সিস্টেমে, একটি single call stack execution-এর পুরো flow দেখায়, যা কারণ ও প্রভাব trace করা সহজ করে তোলে। EDA-তে, আচরণ অনেক স্বাধীনভাবে deploy করা producers এবং consumers থেকে উদ্ভূত হয় যারা events-এ প্রতিক্রিয়া দেখায়, তাই অনুসরণ করার জন্য কোনো একক call stack নেই। correlation/causation IDs সহ distributed tracing, কেন্দ্রীভূত event logging, এবং সুনথিভুক্ত event schemas পরবর্তীতে services জুড়ে causal chain পুনর্গঠন করতে দিয়ে এটি প্রশমিত করে।

**৮. Event-driven architecture কীভাবে একটি বড় organization-এ স্বাধীন team autonomy সমর্থন করে?**
কারণ producers-দের জানার প্রয়োজন নেই কে তাদের events consume করছে, নতুন teams কোনো code পরিবর্তন, coordination, বা producing team থেকে deployment ছাড়াই বিদ্যমান events-এ subscribe করে features তৈরি করতে পারে। এটি teams-কে স্বাধীনভাবে functionality যোগ করতে, পরীক্ষা করতে, এবং deploy করতে দেয়, যা একটি request-driven সিস্টেমে অনেক কঠিন যেখানে caller-কে প্রতিটি downstream service-এর সাথে স্পষ্টভাবে integrate করতে হয়।

**৯. "Event-carried state transfer" data freshness সম্পর্কিত কী ঝুঁকি নিয়ে আসে, এবং আপনি কীভাবে এটি সমাধান করতে পারেন?**
কারণ publish-এর সময় ধরা event snapshot যদি consumer সংরক্ষণ করে এবং পরে source data পরিবর্তিত হয় তাহলে তা পুরনো হয়ে যেতে পারে, শুধুমাত্র event payload-এর উপর নির্ভরশীল consumers পুরনো data-র উপর কাজ করে শেষ করতে পারে। এটি সাধারণত underlying entity পরিবর্তিত হলেই update/versioned events অন্তর্ভুক্ত করে, অথবা event data-কে কঠোর currency প্রয়োজন এমন কোনো কিছুর জন্য একটি live source of truth হিসেবে না দেখে "এই সময় পর্যন্ত সত্য ছিল" একটি fact হিসেবে বিবেচনা করে সমাধান করা হয়।

**১০. আপনি কখন ইচ্ছাকৃতভাবে একটি simple synchronous request-এর পক্ষে event-driven architecture এড়িয়ে যাবেন?**
যখন caller-এর সত্যিকারভাবে এগিয়ে যাওয়ার আগে একটি তাৎক্ষণিক, নিশ্চিত ফলাফল প্রয়োজন — উদাহরণস্বরূপ, checkout সম্পূর্ণ করার অনুমতি দেওয়ার আগে payment authorization যাচাই করা — একটি synchronous call বোঝা সহজ এবং eventual consistency-র জটিলতা এড়িয়ে যায়। EDA সবচেয়ে ভালো side effects, notifications, এবং workflows-এর জন্য সংরক্ষিত যেগুলোর প্রাথমিক ব্যবহারকারী-মুখী operation block করার প্রয়োজন নেই।

**১১. একটি "correlation ID" কী এবং event-driven সিস্টেমে এটি কেন গুরুত্বপূর্ণ?**
একটি correlation ID হলো একটি unique identifier যা একটি প্রাথমিক request বা event-এ সংযুক্ত থাকে এবং তা থেকে উদ্ভূত প্রতিটি পরবর্তী event এবং service call জুড়ে প্রচারিত হয়। এটি গুরুত্বপূর্ণ কারণ এটি engineers-দের অনেক স্বাধীনভাবে প্রতিক্রিয়াশীল services জুড়ে একটি business process-এর সম্পূর্ণ causal chain পুনর্গঠন এবং trace করতে দেয়, যা একটি asynchronous, decoupled architecture-এ অন্যথায় খুবই কঠিন।

**১২. Schema-evolution দৃষ্টিকোণ থেকে event notification বনাম event sourcing-এর coupling ঝুঁকি তুলনা করুন।**
Event notification events ছোট (প্রায়ই শুধু একটি ID এবং type), তাই schema পরিবর্তন কম ঝুঁকিপূর্ণ কারণ evolve করার মতো খুব কম payload আছে — কিন্তু consumers বিস্তারিত জানার জন্য এখনও producer-এর API contract-এর সাথে coupled থাকে। Event sourcing events স্থায়ী source of truth, তাই তাদের schema অত্যন্ত সাবধানে version করতে হবে কারণ আপনার হয়তো বছর পরে পুরনো events replay করার প্রয়োজন হতে পারে — একটি খারাপভাবে পরিকল্পিত schema পরিবর্তন historical state সঠিকভাবে পুনর্গঠনের ক্ষমতা ভেঙে দিতে পারে।
</content>
