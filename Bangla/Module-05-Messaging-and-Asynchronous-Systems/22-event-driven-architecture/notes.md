# অধ্যয়ন নোট: Event-Driven Architecture

## মূল সংজ্ঞাসমূহ

- **Event-Driven Architecture (EDA)**: একটি architectural style যেখানে সিস্টেমের components প্রাথমিকভাবে সরাসরি, synchronous calls-এর মাধ্যমে না হয়ে events (ঘটে যাওয়া বিষয়ে facts) তৈরি ও consume করে যোগাযোগ করে।
- **Event**: একটি immutable রেকর্ড যা প্রমাণ করে কিছু ঘটেছে, সাধারণত past tense-এ নামকরণ করা হয় (যেমন, `OrderPlaced`, `PaymentProcessed`)।
- **Event producer**: তার domain-এ উল্লেখযোগ্য কিছু ঘটলে events emit করে।
- **Event consumer**: প্রাসঙ্গিক events-এ subscribe করে এবং প্রতিক্রিয়া দেখায়।
- **Event broker / event bus**: infrastructure (Kafka, RabbitMQ, cloud pub-sub, ইত্যাদি) যা producers থেকে consumers-এ events পরিবহন করে।

## Request-Driven বনাম Event-Driven

| দিক | Request-Driven | Event-Driven |
|---|---|---|
| যোগাযোগ | Synchronous call, response-এর জন্য অপেক্ষা করে | Asynchronous, fire-and-react |
| Coupling | Tight temporal coupling (উভয়কেই একই সময়ে up থাকতে হয়) | সময়, স্থান, এবং control flow-তে loose coupling |
| Control flow | স্পষ্ট, linear, trace করা সহজ (call stack) | উদ্ভূত, producers/consumers জুড়ে বিতরণকৃত |
| Failure-এর প্রভাব | callee down থাকলে caller block/fail হয় | Producer প্রভাবিত হয় না; consumer পরে catch up করে |
| নতুন functionality যোগ করা | প্রায়ই caller পরিবর্তন করতে হয় | নতুন consumer শুধু subscribe করে; upstream পরিবর্তন নেই |

## Events-এর তিনটি ধরন

| শৈলী | বর্ণনা | Tradeoff |
|---|---|---|
| Event notification | পাতলা event, ন্যূনতম data (যেমন, শুধু একটি ID); consumer বিস্তারিত জানতে source-এ call back করে | ছোট events, কিন্তু producer-এর API-এর উপর coupling/availability dependency পুনরায় প্রবেশ করায় |
| Event-carried state transfer | Event-এ consumer-এর প্রয়োজনীয় সব data থাকে | Decoupling/resilience সর্বোচ্চ করে; বড় payload, data freshness পরিচালনা করতে হয় |
| Event sourcing | Events-এর সম্পূর্ণ history-ই source of truth; বর্তমান state = events-এর replay | সম্পূর্ণ audit trail, time-travel/পুনর্গঠন ক্ষমতা; উল্লেখযোগ্য query/tooling জটিলতা |

## EDA-এর সুবিধা

- Producers এবং consumers-এর স্বাধীন scaling।
- Resilience: একটি ব্যর্থ/ধীর consumer producer বা অন্য consumers-কে প্রভাবিত করে না।
- Extensibility: নতুন features upstream services পরিবর্তন না করেই বিদ্যমান events-এ subscribe করে।
- অনেক autonomous teams/services সহ organization-এর জন্য স্বাভাবিক উপযোগিতা।

## EDA-এর চ্যালেঞ্জ

- **Eventual consistency**: সিস্টেমের বিভিন্ন অংশ একই data-র ভিন্ন দৃষ্টিভঙ্গি সাময়িকভাবে ধরে রাখতে পারে।
- **Debuggability/জটিলতা**: আচরণ একটি single call stack-এর বদলে producers/consumers-এর একটি জাল থেকে উদ্ভূত হয়; distributed tracing, correlation IDs, এবং সুনির্ধারিত event schemas প্রয়োজন।
- **Schema evolution**: একটি event-এর আকার পরিবর্তন করলে পুরনো fields-এর উপর নির্ভরশীল consumers ভেঙে যেতে পারে।
- **Testing জটিলতা**: একাধিক স্বাধীনভাবে deploy করা reactive services জুড়ে বিস্তৃত end-to-end flows test করা কঠিন।

## দ্রুত সারসংক্ষেপ

- EDA "call and wait"-কে "announce and react"-এ প্রতিস্থাপন করে।
- Consumers-এর কতটা data প্রয়োজন এবং producer-এর API-তে কতটা coupling সহ্য করতে পারেন তার ভিত্তিতে event শৈলী বেছে নিন।
- Event sourcing শক্তিশালী কিন্তু ভারী — এটি শুধুমাত্র সেসব domain-এর জন্য সংরক্ষণ করুন যেখানে audit trail/time-travel সত্যিকারের মূল্যবান (যেমন, financial ledgers)।
- আগে থেকেই observability-তে বিনিয়োগ করুন (tracing, correlation IDs, event schemas/versioning) — এই tools ছাড়া EDA-এর জটিলতা scale-এ অনিয়ন্ত্রিত হয়ে ওঠে।
</content>
