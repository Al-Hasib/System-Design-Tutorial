# Event-Driven Architecture

**কঠিনতার মাত্রা:** Intermediate/Advanced

## শেখার লক্ষ্যসমূহ

- event-driven architecture (EDA) সংজ্ঞায়িত করা এবং request-driven architecture থেকে এর পার্থক্য বোঝা
- EDA-এর মূল building block গুলো বোঝা: events, event producers, event consumers, event brokers/buses
- event notification, event-carried state transfer, এবং event sourcing শৈলীগুলোর তুলনা করা
- EDA-এর সুবিধা (scalability, decoupling, resilience) এবং চ্যালেঞ্জ (complexity, eventual consistency, debugging) চিহ্নিত করা
- একটি বাস্তবসম্মত multi-service সিস্টেম ডিজাইন পরিস্থিতিতে EDA চিন্তাভাবনা প্রয়োগ করা

## স্ক্রিপ্ট

### ভূমিকা

একটি newsroom-এর সাথে একটি phone tree-র তুলনা করে দেখুন। একটি phone tree-তে, যদি আপনি দশজন মানুষকে কিছু জানাতে চান, তাহলে আপনি প্রথম জনকে কল করেন, সে পরের জনকে কল করে, সে আবার পরের জনকে — আর যদি মাঝপথে কোনো একটি লিঙ্ক অনুপস্থিত থাকে, তাহলে পুরো chain ভেঙে যায়। একটি newsroom-এ, একজন reporter শুধু গল্পটি প্রকাশ করে দেয়। প্রতিটি subscriber — প্রতিটি পাঠক যার subscription আছে — নিজের সময়ে, নিজের channel-এর মাধ্যমে খবরটি জানতে পারে, reporter-কে প্রত্যেককে আলাদা করে জানাতে হয় না। Event-driven architecture হলো newsroom মডেলটি সফটওয়্যার সিস্টেমে প্রয়োগ করা: services একে অপরকে সরাসরি call করে response-এর জন্য অপেক্ষা করার বদলে, তারা ঘটে যাওয়া বিষয়ে তথ্য প্রকাশ করে, যাকে বলা হয় events, এবং অন্য services স্বাধীনভাবে প্রতিক্রিয়া জানাতে পারে। আজ আমরা আগে আলোচনা করা individual patterns থেকে zoom out করে পুরো architectural philosophy-টি দেখব।

### Request-Driven বনাম Event-Driven

আমাদের বেশিরভাগই request-driven সিস্টেম তৈরি করে শুরু করি: Service A, Service B-এর API call করে, response-এর জন্য অপেক্ষা করে, এবং B যা রিটার্ন করে তার ভিত্তিতে এগিয়ে যায়। এটি বোঝা সহজ — এটি synchronous, linear, এবং trace করা সহজ। কিন্তু এটি tight temporal coupling তৈরি করে: A এখন নির্ভরশীল থাকে B এর ঠিক সেই মুহূর্তে available, দ্রুত এবং সঠিক থাকার উপর, যখন A-এর প্রয়োজন হয়।

Event-driven architecture এটিকে উল্টে দেয়। Service A, Service B-কে কিছু করতে বলার এবং অপেক্ষা করার বদলে, Service A ঘোষণা করে "X ঘটেছে" — একটি event — এবং জানে না বা পাত্তা দেয় না কে এতে প্রতিক্রিয়া দেখাবে, বা কখন। এটি শুধু গত বার আলোচনা করা pub-sub mechanism নয়; এটি আপনার সিস্টেমের control flow ডিজাইন করার একটি সম্পূর্ণ ভিন্ন পদ্ধতি। একটি event-driven সিস্টেমে, এরপর কী ঘটবে তার sequence কোনো একটি service-এ hard-code করা থাকে না — এটি উদ্ভূত হয় কোন services কোন events শুনছে তার উপর ভিত্তি করে। এটি একটি গভীর পরিবর্তন: আপনার সিস্টেমের আচরণ command-এর একটি chain না হয়ে facts-এর প্রতি প্রতিক্রিয়ার একটি graph হয়ে ওঠে।

### মূল Building Blocks

একটি event-driven সিস্টেমের কয়েকটি মূল অংশ থাকে। একটি **event** হলো একটি immutable রেকর্ড যা প্রমাণ করে যে কিছু ঘটেছে — "OrderPlaced," "PaymentProcessed," "UserDeactivated।" past tense লক্ষ্য করুন; events facts বর্ণনা করে, requests নয়। একটি command থেকে ভিন্ন, যা বলে "এটা করো," একটি event বলে "এটা ঘটেছে," এবং listener-এর সিদ্ধান্ত থাকে সে প্রতিক্রিয়া দেখাবে কিনা এবং কীভাবে দেখাবে।

**Event producers** তাদের domain-এ উল্লেখযোগ্য কিছু ঘটলে এই events emit করে। **Event consumers** প্রাসঙ্গিক events-এ subscribe করে এবং প্রতিক্রিয়া দেখায় — তাদের নিজস্ব data update করে, side effects trigger করে, অথবা নিজেরাই নতুন events emit করে, যা প্রতিক্রিয়ার chain তৈরি করে। এবং এই সবকিছুকে একত্র করে একটি **event broker** বা **event bus** — প্রায়শই Kafka, RabbitMQ, বা একটি cloud-native service — যা প্রকৃত delivery, buffering, এবং fan-out পরিচালনা করে, যা আমরা গত দুটি video-তে আলোচনা করেছি।

### Events-এর তিনটি ধরন

জানা দরকার যে EDA-তে "event"-এর একটিই শৈলী নেই। **Event notification** সবচেয়ে হালকা — event শুধু বলে যে কিছু ঘটেছে এবং হয়তো একটি ID অন্তর্ভুক্ত থাকে, আর consumer-এর যদি আরও বিস্তারিত প্রয়োজন হয়, তাহলে সে source service-এর API-তে call back করে। এতে events ছোট থাকে কিন্তু কিছুটা coupling পুনরায় প্রবেশ করে, কারণ consumers এখনও producer-এর API পরে available থাকার উপর নির্ভরশীল থাকে।

**Event-carried state transfer** আরও এগিয়ে যায় — event নিজেই consumer-এর প্রয়োজনীয় সব প্রাসঙ্গিক data বহন করে, তাই তাকে কখনো call back করতে হয় না। উদাহরণস্বরূপ, একটি "OrderPlaced" event-এ সম্পূর্ণ order details, customer ID, এবং item list অন্তর্ভুক্ত থাকতে পারে, তাই shipping service-কে order service-এ query করার দরকারই পড়ে না। এটি decoupling এবং resilience সর্বোচ্চ করে কিন্তু এর মানে events আরও বেশি data বহন করে এবং সেই data যুক্তিসঙ্গতভাবে fresh এবং consistent রাখার বিষয়ে সতর্কভাবে চিন্তা করতে হয়।

**Event sourcing** সবচেয়ে architecturally উচ্চাভিলাষী: একটি entity-র শুধু বর্তমান state সংরক্ষণ করার বদলে, আপনি সেই state পর্যন্ত পৌঁছানো সম্পূর্ণ events-এর sequence সংরক্ষণ করেন, এবং বর্তমান state সেগুলো replay করে derive করা হয়। এটিকে একটি bank balance-এর বদলে একটি bank ledger হিসেবে ভাবুন — আপনি balance overwrite করেন না, প্রতিটি transaction রেকর্ড করেন, এবং balance একটি computed view। এটি আপনাকে একটি সম্পূর্ণ audit trail এবং যেকোনো সময়ের state পুনর্গঠনের ক্ষমতা দেয়, তবে querying এবং tooling-এ উল্লেখযোগ্য জটিলতার বিনিময়ে।

### সুবিধা এবং চ্যালেঞ্জ

EDA-এর সুবিধাগুলো গত দুই video-তে আমরা যা তৈরি করেছি তারই প্রতিধ্বনি: services স্বাধীনভাবে scale করে, একটি consumer-এর failure producer বা অন্য consumers-কে ধ্বংস করে না, এবং নতুন functionality শুধু বিদ্যমান events-এ subscribe করার মাধ্যমে যোগ করা যায় — upstream services-এ কোনো পরিবর্তন প্রয়োজন হয় না। বহু autonomous team সহ বড় organization-এর জন্য এটি একটি বিশাল ব্যাপার।

কিন্তু EDA বিনামূল্যে আসে না। সবচেয়ে বড় চ্যালেঞ্জ হলো **eventual consistency**: কারণ প্রতিক্রিয়াগুলো asynchronously ঘটে, একটি সময়ের window থাকে যেখানে সিস্টেমের বিভিন্ন অংশের বাস্তবতার ভিন্ন দৃষ্টিভঙ্গি থাকে। একজন customer support agent যদি একটি order প্লেস হওয়ার দুইশ মিলিসেকেন্ড পরে সেটি দেখে, তাহলে কি inventory ইতিমধ্যে decrement হয়েছে? confirmation email কি পাঠানো হয়েছে? আপনাকে আপনার সিস্টেম — এবং আপনার product প্রত্যাশা — এই lag-কে ঘিরে ডিজাইন করতে হবে, এটি নেই এমন ভান করার বদলে।

দ্বিতীয় চ্যালেঞ্জ হলো **জটিলতা এবং debuggability**। যখন আচরণ একটি single call stack-এর বদলে event producers এবং consumers-এর একটি জালের মধ্য থেকে উদ্ভূত হয়, তখন কয়েক ডজন services জুড়ে "এটা কেন ঘটল" trace করার জন্য distributed tracing, correlation IDs, এবং event schemas-এ গুরুতর বিনিয়োগ প্রয়োজন হয়। যেসব team এই বিনিয়োগ ছাড়াই EDA গ্রহণ করে তারা প্রায়ই এমন সিস্টেম পায় যা technically decoupled কিন্তু বাস্তবে debug করা অসম্ভব।

### বাস্তব-জগতের উদাহরণ

ভাবুন Netflix-এর মতো একটি কোম্পানি কীভাবে একজন ব্যবহারকারীর একটি episode শেষ করা পরিচালনা করে। সেই একটি fact — "playback completed" — স্বাধীন প্রতিক্রিয়ার একটি cascade trigger করে: recommendation engine viewing history update করে পরবর্তীতে কী suggest করবে তা পরিমার্জন করে, "continue watching" row update হয়, একটি billing/usage analytics service licensing calculations-এর জন্য watch time log করে, এবং একটি autoplay-next-episode feature সক্রিয় হয়। এগুলোর কোনোটিই কোনো একটি service-এর কোডে একসাথে chain করা নেই। playback service শুধু একটি event emit করে, এবং বিভিন্ন priority ও release schedule সহ এই দলগুলোর সমষ্টি নিজেদের শর্তে প্রতিক্রিয়া জানায়। যদি recommendation engine redeploy হচ্ছে এবং সাময়িকভাবে unavailable থাকে, তাহলে playback একেবারেই প্রভাবিত হয় না — সে ফিরে এলে শুধু event catch up করে নেবে।

### পুনরালোচনা

চলুন সংক্ষেপ করি। Event-driven architecture একটি design philosophy যেখানে সিস্টেমের আচরণ services একে অপরকে সরাসরি call করে অপেক্ষা করার বদলে events তৈরি ও প্রতিক্রিয়া দেখানোর মধ্য থেকে উদ্ভূত হয়। Events হলো ইতিমধ্যে ঘটে যাওয়া বিষয়ে immutable facts। তিনটি ধরন আছে: হালকা event notification, সমৃদ্ধ event-carried state transfer, এবং সবচেয়ে উচ্চাভিলাষী, event sourcing, যেখানে event log নিজেই source of truth। EDA আপনাকে scalability, resilience, এবং team autonomy দেয়, কিন্তু বিনিময়ে eventual consistency এবং অতিরিক্ত সিস্টেম জটিলতার মূল্য দিতে হয় যার জন্য শক্তিশালী observability practices প্রয়োজন।

### এরপর কী

আমরা এখন events সরানোর "কীভাবে" এবং সেগুলো ঘিরে ডিজাইন করার "কেন" আলোচনা করেছি। পরবর্তী video-তে, আমরা একটি নির্দিষ্ট এবং খুবই ব্যবহারিক সিদ্ধান্ত দেখব যা event-driven এবং data-heavy সিস্টেমে ক্রমাগত দেখা যায়: আপনি কি আপনার data batches-এ, একটি schedule অনুযায়ী প্রক্রিয়া করবেন, নাকি ক্রমাগত একটি stream হিসেবে, তা এসে পৌঁছানোর মুহূর্তেই? আমরা batch এবং stream processing-কে মুখোমুখি তুলনা করব। সেখানে দেখা হবে।

## মূল শিক্ষণীয় বিষয়

- Event-driven architecture (EDA) request-driven design-কে উল্টে দেয়: services একে অপরকে সরাসরি call করে অপেক্ষা করার বদলে facts (events) ঘোষণা করে।
- Events হলো immutable, past-tense রেকর্ড যা কিছু ঘটার প্রমাণ দেয়, যা একটি event broker/bus-এর মাধ্যমে স্বাধীনভাবে produce এবং consume করা হয়।
- তিনটি event শৈলী: event notification (পাতলা, callback প্রয়োজন হতে পারে), event-carried state transfer (self-contained), এবং event sourcing (event log-ই source of truth)।
- EDA-এর সুবিধা: স্বাধীন scaling, individual service failure-এর প্রতি resilience, এবং upstream services স্পর্শ না করেই নতুন functionality সহজে যোগ করা।
- EDA-এর খরচ: eventual consistency (services জুড়ে সাময়িক অসামঞ্জস্যপূর্ণ দৃষ্টিভঙ্গি) এবং উচ্চতর সিস্টেম জটিলতা যার জন্য শক্তিশালী observability এবং event contracts প্রয়োজন।
</content>
