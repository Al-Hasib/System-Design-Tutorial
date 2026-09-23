# Practice ও Interview Questions

1. **System design কী, আপনার নিজের ভাষায় বলুন।**
   একটি software system-এর components (clients, servers, databases, caches, queues, ইত্যাদি) কীভাবে একসাথে খাপ খায় এবং interact করে scale, performance, এবং reliability সম্পর্কিত requirements পূরণ করার জন্য তা সংজ্ঞায়িত করার প্রক্রিয়া — কোনো একটি single component-এর implementation লেখার বিপরীতে।

2. **System design একটি algorithm লেখা বা coding interview problem থেকে কীভাবে আলাদা?**
   Algorithm/coding problems-এ সাধারণত স্পষ্ট inputs/outputs সহ একটি optimal, verifiable উত্তর থাকে। System design problems open-ended, অস্পষ্ট requirements স্পষ্ট করা প্রয়োজন, এবং একটি একক "সঠিক" সমাধানের পরিবর্তে trade-offs ও reasoning-এর গুণমানের উপর ভিত্তি করে বিচার করা হয়।

3. **Companies কেন interviews-এ system design skills পরীক্ষা করে?**
   কারণ প্রকৃত engineering কাজ — বিশেষ করে senior levels-এ — শুধু ভালোভাবে specified tickets implement করা নয়, বরং features এবং services কীভাবে architect করা উচিত তা ঠিক করা প্রয়োজন। Interview সেই decision-making process-টিকে simulate করে।

4. **বেশিরভাগ large-scale systems-এ common তিনটি core building block-এর নাম বলুন।**
   এর যেকোনো তিনটি: client, server, database, cache, load balancer, message queue।

5. **Bricklayer/architect analogy ব্যবহার করে, coding এবং system design-এর মধ্যে পার্থক্য ব্যাখ্যা করুন।**
   একজন bricklayer একটি brick (একটি function/module) সঠিকভাবে বসানোর উপর ফোকাস করেন। একজন architect সামগ্রিক structure ঠিক করেন — ঘরগুলো কীভাবে সংযুক্ত থাকে, building চাপ কীভাবে সামলায়, utilities কীভাবে প্রবাহিত হয় — যা একটি system-এ services, data stores, এবং communication patterns কীভাবে খাপ খায় তা ঠিক করার সমতুল্য।

6. **System design-এ খুব কমই একটি একক "সঠিক" উত্তর থাকে কেন?**
   কারণ প্রতিটি design choice-এ trade-offs জড়িত (cost, complexity, latency, consistency, availability) যা requirements এবং constraints-এর উপর নির্ভর করে, এবং এগুলো পরিস্থিতি ভেদে ভিন্ন হয় — একটি context-এর জন্য একটি ভালো design অন্য context-এর জন্য ভুল হতে পারে।

7. **Scenario: আপনার interviewer আপনাকে "একটি URL shortener design করো" বলেছেন। কোনো box আঁকার আগে আপনার একদম প্রথম পদক্ষেপ কী হওয়া উচিত?**
   Requirements স্পষ্ট করা — প্রত্যাশিত scale (প্রতিদিন কতগুলো URL shorten হয়, read/write ratio), URL কি expire হয়, analytics দরকার কিনা, ইত্যাদি — একটি architecture প্রস্তাব করার আগে (এটি পরের video-তে আরও বিস্তারিত হবে)।

8. **এই course system design শেখার জন্য দুটি প্রধান কারণ কী দেয়?**
   (১) এটি mid/senior roles-এর জন্য technical interviews-এর একটি standard অংশ, এবং (২) এটি সেইসব engineer-দের জন্য একটি মূল real-world skill যারা তাদের career-এ বেড়ে ওঠার সাথে সাথে architecture decisions নিতে হয়।

9. **এই course-এর 12টি module ক্রমানুসারে তালিকাভুক্ত করুন।**
   Foundations; Networking & Communication; Databases & Storage; Caching & Content Delivery; Messaging & Asynchronous Systems; Distributed Systems Concepts; Architecture Patterns; Protocols, Formats & Security; Database & API Internals; Distributed Coordination & Scale Techniques; Observability, Deployment & Production Operations; Case Studies/Interview Practice।

10. **সত্য নাকি মিথ্যা: System design-এর জন্য একটি নির্দিষ্ট programming language গভীরভাবে জানা প্রয়োজন।**
    মিথ্যা — system design মূলত language-agnostic; এটা architecture এবং trade-offs নিয়ে, যদিও implementation knowledge feasibility নিয়ে reasoning করতে সাহায্য করতে পারে।

11. **Scenario: একজন junior engineer বলেন "আমার system design দরকার নেই, আমার শুধু ভালো code লিখতে হবে।" আপনি কীভাবে জবাব দেবেন?**
    ভালো code লেখা প্রয়োজনীয় কিন্তু যথেষ্ট নয় — systems যত বড় হয়, components কীভাবে interact করে, scale করে, এবং reliable থাকে সে সম্পর্কিত decisions সাফল্যের প্রধান factor হয়ে ওঠে, এবং সেই decisions সম্পর্কে reasoning করতেই system design আপনাকে শেখায়।

12. **এই course networking, databases, বা distributed systems-এর আগে "Foundations" দিয়ে কেন শুরু হয়?**
    কারণ requirements gathering, client-server communication, scalability, এবং reliability-এর মতো concepts প্রতিটি পরবর্তী module জুড়ে reference করা prerequisites — অন্য কোথাও শুরু করলে বারবার পিছিয়ে গিয়ে basic vocabulary ব্যাখ্যা করা প্রয়োজন হতো।
