# Study Notes: Distributed Transactions — 2PC ও Saga

## সংজ্ঞাসমূহ

- **Distributed transaction**: একটি logical কাজের একক যা একটি অনির্ভরযোগ্য network-এর মাধ্যমে coordinate করা দুই বা ততোধিক স্বাধীনভাবে ব্যর্থ হতে পারা resource — আলাদা database, service, বা message broker — জুড়ে অবশ্যই atomic (all-or-nothing) দেখাতে হবে।
- **Two-Phase Commit (2PC)**: একটি blocking atomic commitment protocol যেখানে একটি coordinator সব participant-কে vote দিতে বলে (Prepare phase), তারপর সর্বসম্মত ফলাফলের ভিত্তিতে সব participant-কে commit বা abort করার নির্দেশ দেয় (Commit phase)।
- **Saga**: একটি sequence of local transaction, প্রতিটি নিজের service/database-এ স্বাধীনভাবে committed, যেখানে যেকোনো step-এ একটি failure পূর্বে সম্পন্ন হওয়া প্রতিটি step-এর জন্য compensating transaction ট্রিগার করে, বিপরীত ক্রমে চালানো হয়।
- **Compensating transaction**: একটি আলাদা, ইচ্ছাকৃতভাবে ডিজাইন করা business operation যা পূর্বে committed একটি local transaction-এর প্রভাবকে semantically undo করে (যেমন, "payment refund" undo করে "charge card")। এটি প্রকৃত rollback নয় — মূল প্রভাব ইতিমধ্যে জগতের কাছে দৃশ্যমান/committed ছিল।
- **Orchestration (saga)**: একটি central orchestrator component explicitly প্রতিটি service-এর local transaction চালু করে এবং failure-এর সময়, explicitly প্রয়োজনীয় compensating transaction চালু করে।
- **Choreography (saga)**: প্রতিটি service domain event প্রকাশ করে এবং শোনে, প্রতিক্রিয়ায় স্বাধীনভাবে নিজের local transaction বা compensation চালানোর সিদ্ধান্ত নেয় — কোনো central coordinator নেই।
- **XA**: X/Open standard interface যা একটি transaction manager-কে একাধিক XA-compliant resource manager (database, message broker) জুড়ে 2PC coordinate করতে দেয়।

## তুলনা টেবিল: 2PC বনাম Saga

| মাত্রা | Two-Phase Commit (2PC) | Saga |
|---|---|---|
| Atomicity | প্রকৃত atomic commit — সব participant commit করে অথবা কেউই করে না | কোনো প্রকৃত atomicity নেই — compensation-এর মাধ্যমে eventual consistency; একটি সাময়িক "আংশিকভাবে প্রয়োগ করা" state বিদ্যমান |
| Isolation | সংরক্ষিত — চূড়ান্ত সিদ্ধান্ত না হওয়া পর্যন্ত participant-রা lock ধরে রাখে | সংরক্ষিত নয় — committed local transaction থেকে intermediate state অন্যদের কাছে দৃশ্যমান |
| Availability | হ্রাসপ্রাপ্ত — Prepare-এর পরে coordinator crash সব participant-কে অনির্দিষ্টকালের জন্য block করে (blocking protocol) | উচ্চ — প্রতিটি local transaction সাথে সাথেই commit হয় এবং resource release করে; failure asynchronously সামলানো হয় |
| Failure handling | Coordinator পুনরুদ্ধার না হওয়া পর্যন্ত (অথবা একটি heuristic/manual সিদ্ধান্ত না নেওয়া পর্যন্ত) participant-রা lock ধরে block হয়ে থাকে | সম্পন্ন হওয়া step-গুলোকে semantically undo করতে compensating transaction চালানো হয়; প্রতিটি step-এর জন্য design করতে হবে |
| জটিলতা | Protocol/infrastructure জটিলতা (XA-compliant resource, transaction manager প্রয়োজন); সরল application code | Application-স্তরের জটিলতা (compensation design করতে হবে, ordering, duplication, আংশিক compensation failure সামলাতে হবে) |
| Latency | বেশি — synchronous, প্রতিটি participant-এর কাছে network round trip জুড়ে lock ধরে রাখে | কম / non-blocking — local commit সাথে সাথেই ঘটে; সামগ্রিক saga সম্পূর্ণভাবে settle হতে বেশি সময় নিতে পারে কিন্তু block করে না |
| সেরা ব্যবহার-ক্ষেত্র | একটি বিশ্বস্ত boundary-র মধ্যে অল্প সংখ্যক tightly-coupled participant (যেমন, monolith + ২টি DB, স্বল্পস্থায়ী ledger operation) | Loosely-coupled microservices, স্বাধীনভাবে deploy/মালিকানাধীন service, দীর্ঘস্থায়ী business process |

## Saga Failure-Handling বিবেচ্য বিষয়সমূহ

- **Idempotent compensation**: Compensating transaction (এবং forward step) একাধিকবার execute করা নিরাপদ হতে হবে, কারণ distributed messaging-এ retry এবং at-least-once delivery সাধারণ ব্যাপার।
- **Ordering guarantee**: Event/step অসামঞ্জস্যপূর্ণ ক্রমে আসতে পারে বা ডুপ্লিকেট হতে পারে; saga logic (orchestrator বা প্রতিটি choreography participant)-কে এটি সহ্য করতে হবে অথবা explicitly sequence করতে হবে — যেমন, saga ID এবং step versioning ব্যবহার করে।
- **Semantic lock**: যেহেতু isolation হারিয়ে যায়, একটি application-স্তরের "semantic lock" ব্যবহার করুন — যেমন, একটি record-কে `PENDING`/`reserved` চিহ্নিত করা — যাতে অন্যান্য transaction জানে যে in-flight saga state-কে চূড়ান্ত হিসেবে ধরা যাবে না, dirty-read-ধরনের anomaly কমাতে।
- **Compensation failure**: একটি compensating transaction নিজেই ব্যর্থ হতে পারে (যেমন, refund API ডাউন) — backoff-সহ retry, dead-letter handling, এবং সম্ভাব্য manual/ops হস্তক্ষেপ path প্রয়োজন।
- **Non-reversible step**: কিছু action (যেমন, একটি physical package পাঠানো, একটি ইমেইল পাঠানো) সত্যিকারভাবে compensate করা যায় না — শুধুমাত্র প্রশমিত করা যায় (যেমন, "ক্ষমা চাওয়ার ইমেইল পাঠানো," "return label জারি করা")। Saga এমনভাবে design করুন যাতে irreversible step সবার শেষে ঘটে।
- **Timeout**: প্রতিটি local transaction step-এর একটি timeout থাকা উচিত যার পরে saga failure ধরে নেয় এবং অনির্দিষ্টকাল অপেক্ষা না করে compensation শুরু করে।
- **Observability**: যেহেতু logic step/event জুড়ে ছড়িয়ে থাকে, saga-র কী ঘটেছে তা পুনর্নির্মাণ করতে শক্তিশালী tracing/correlation ID প্রয়োজন, বিশেষত choreography-based saga-র ক্ষেত্রে।

## দ্রুত Interview Revision পয়েন্ট

- 2PC = শক্তিশালী atomicity + isolation, কিন্তু Prepare-এর পরে coordinator ব্যর্থ হলে blocking এবং availability-সীমাবদ্ধ।
- Saga = কোনো cross-service isolation/atomicity নেই, কিন্তু উচ্চ availability; rollback নয়, compensating transaction ব্যবহার করে।
- Compensating transaction ≠ rollback: এটি একটি নতুন, সামনের দিকে এগিয়ে যাওয়া operation যা ইতিমধ্যে দৃশ্যমান হওয়া প্রভাবগুলোকে undo করে।
- Choreography = event-driven, বিকেন্দ্রীভূত, ট্রেস করা কঠিন; Orchestration = central coordinator, বোঝা এবং monitor করা সহজ।
- XA হলো standard-ভিত্তিক mechanism যা বেশিরভাগ প্রকৃত 2PC implementation ব্যবহার করে।
- বাস্তব system (Temporal, Camunda, AWS Step Functions) state persistence হাতে-কলমে তৈরি না করে saga implement করতে durable execution প্রদান করে।
- সাধারণ নিয়ম: একটি trust boundary-র মধ্যে কয়েকটি tightly-coupled resource-এর জন্য 2PC; loosely-coupled microservices-এর জন্য Saga।
