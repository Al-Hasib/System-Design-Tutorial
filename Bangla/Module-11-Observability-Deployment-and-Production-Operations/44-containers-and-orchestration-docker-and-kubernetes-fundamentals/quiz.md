# অনুশীলন ও ইন্টারভিউ প্রশ্ন (Practice & Interview Questions)

**১. একটি container এবং একটি virtual machine-এর মধ্যে মৌলিক আর্কিটেকচারাল পার্থক্য কী?**
একটি virtual machine একটি সম্পূর্ণ কম্পিউটারকে virtualize করে, যার নিজস্ব পূর্ণাঙ্গ OS kernel-সহ, যা একটি hypervisor-এর উপর চলে। একটি container শুধুমাত্র OS স্তরে virtualize করে — এটি host-এর kernel share করে কিন্তু namespaces এবং cgroups-এর মতো kernel feature-এর মাধ্যমে নিজস্ব isolated filesystem, process namespace, এবং network stack পায়।

**২. Container-গুলো virtual machine-এর চেয়ে এত বেশি দ্রুত কেন শুরু হয়?**
একটি VM-কে তার নিজস্ব kernel-সহ একটি সম্পূর্ণ guest operating system boot করতে হয়, যা প্রকৃত সময় নেয়। একটি container একেবারেই তার নিজস্ব kernel বহন করে না — এটি host-এর ইতিমধ্যে চলমান kernel share করে — তাই একটি container শুরু করা একটি সাধারণ process শুরু করার কাছাকাছি, যা মিনিটের বদলে মিলিসেকেন্ড থেকে কয়েক সেকেন্ড সময় নেয়।

**৩. একটি container image কোন সমস্যার সমাধান করে, এবং কীভাবে?**
এটি "works on my machine" সমাধান করে — যে পরিস্থিতিতে একটি application বিভিন্ন environment জুড়ে অসামঞ্জস্যপূর্ণ dependency সংস্করণ বা configuration-এর কারণে ভিন্নভাবে আচরণ করে। একটি image application-টিকে, এর সঠিক runtime, এবং এর সমস্ত dependency-কে একটি অপরিবর্তনীয় স্ন্যাপশটে প্যাকেজ করে, যা একবার build হয় এবং যেখানেই একটি container runtime বিদ্যমান সেখানে অভিন্নভাবে চলে, তাই "আশা করি এই environment মিলবে" এমন অনুমানের প্রয়োজন হয় না।

**৪. Kubernetes-এর মতো একটি orchestrator-এর তিনটি দায়িত্ব তালিকাভুক্ত করুন।**
এর যেকোনো তিনটি: scheduling (কোন machine কোন container চালাবে তা ঠিক করা), scaling (load-এর ভিত্তিতে instance যোগ/বাদ দেওয়া), self-healing (crash হওয়া container পুনরায় চালু করা, machine fail হলে পুনরায় schedule করা), service discovery/load balancing (পরিবর্তনশীল instance জুড়ে স্থিতিশীল addressing), অথবা rolling updates (পুরনো instance-গুলোকে ধীরে ধীরে নতুন দিয়ে প্রতিস্থাপন করা)।

**৫. একটি Kubernetes Pod কী, এবং এটিকে একটি একক container-এর পরিবর্তে "ক্ষুদ্রতম deployable একক" হিসেবে কেন বর্ণনা করা হয়?**
একটি Pod হলো এক বা একাধিক নিবিড়ভাবে সংযুক্ত container যা একই machine-এ একসাথে schedule হয়, একটি network namespace share করে। এটি ক্ষুদ্রতম deployable একক কারণ Kubernetes Pod স্তরে schedule এবং পরিচালনা করে — যদিও বাস্তবে বেশিরভাগ pod-এ শুধু একটি application container থাকে, এই abstraction প্রয়োজন হলে নিবিড়ভাবে সংযুক্ত helper container-গুলোকে একসাথে অবস্থান করতে দেয়।

**৬. একটি Kubernetes Deployment আসলে কী করে?**
এটি pod-এর একটি সেটের কাঙ্ক্ষিত অবস্থা বর্ণনা করে — যেমন, "এই container image-এর 5টি replica চলমান থাকা উচিত" — এবং Kubernetes ক্রমাগত প্রকৃত চলমান অবস্থাকে সেই কাঙ্ক্ষিত অবস্থার দিকে reconcile করে, প্রয়োজন অনুযায়ী pod পুনরায় চালু করে বা পুনরায় schedule করে যদি বাস্তবতা এর থেকে বিচ্যুত হয় (যেমন, একটি pod crash করে বা একটি machine fail করে)।

**৭. একটি Kubernetes Service কী প্রদান করে, এবং Pod-গুলো ঘন ঘন তৈরি ও ধ্বংস হতে পারে বলে এটি কেন প্রয়োজনীয়?**
একটি Service একটি একক, স্থিতিশীল network ঠিকানা প্রদান করে যা বর্তমানে যে pod-গুলো healthy এবং একটি Deployment-এর অংশ তাদের দিকে route করে। যেহেতু পৃথক pod-গুলোকে যেকোনো সময় পুনরায় schedule, পুনরায় চালু, বা scale up/down করা যেতে পারে (প্রত্যেকে সম্ভাব্যভাবে একটি নতুন internal ঠিকানা পেয়ে), অন্য service-গুলোর একটি স্থিতিশীল ঠিকানা প্রয়োজন যা কল করার জন্য, যা অন্তর্নিহিত pod-গুলোর সাথে পরিবর্তিত হয় না — এটিই ঠিক যা একটি Service প্রদান করে।

**৮. একটি liveness probe এবং একটি readiness probe-এর মধ্যে পার্থক্য ব্যাখ্যা করুন, এবং প্রতিটি ব্যর্থ হলে কী ঘটে।**
একটি liveness probe পরীক্ষা করে একটি container এখনও কার্যকর কিনা (hung বা deadlocked নয়); যদি এটি ব্যর্থ হয়, Kubernetes container-টিকে kill করে এবং পুনরায় চালু করে। একটি readiness probe পরীক্ষা করে একটি pod বর্তমানে ট্রাফিক পরিচালনা করতে সক্ষম কিনা (যেমন, এটি হয়তো জীবিত কিন্তু এখনও warm up হচ্ছে); যদি এটি ব্যর্থ হয়, pod-টিকে শুধু Service-এর routing থেকে সরিয়ে দেওয়া হয় যতক্ষণ না এটি আবার pass করে — কোনো restart ঘটে না, শুধু ট্রাফিক থেকে সাময়িক বর্জন।

**৯. পরিস্থিতি: একটি নতুন pod শুরু হওয়া শেষ করে কিন্তু request সঠিকভাবে পরিবেশন করার আগে একটি in-memory cache warm করতে 10 সেকেন্ড প্রয়োজন। এই ভিডিও থেকে কোন Kubernetes মেকানিজম নিশ্চিত করে যে এটি সেই warm-up-এর সময় ট্রাফিক গ্রহণ করে না, এবং কীভাবে?**
একটি readiness probe — যা শুধুমাত্র cache warm-up সম্পূর্ণ হলে "ready" রিপোর্ট করার জন্য configured (যেমন, একটি internal health endpoint চেক করে যা এটি প্রতিফলিত করে)। ততক্ষণ পর্যন্ত, pod-টি তার readiness check-এ ব্যর্থ হয় এবং Service-এর routing থেকে বাদ পড়ে, তাই অকালে কোনো ট্রাফিক এতে পৌঁছায় না, pod-টিকে পুনরায় চালু না করেই।

**১০. সত্য নাকি মিথ্যা: যেহেতু container-গুলো host-এর kernel share করে, তারা virtual machine-এর মতোই ঠিক একই মাত্রার নিরাপত্তা isolation প্রদান করে।**
মিথ্যা। host kernel share করার মানে হলো container-গুলোর VM-এর তুলনায় স্বাভাবিকভাবেই দুর্বল isolation আছে — একটি kernel-স্তরের vulnerability তাত্ত্বিকভাবে container সীমানা অতিক্রম করার জন্য exploit করা যেতে পারে, যা VM সীমানার ক্ষেত্রে অনেক বেশি কঠিন, যাদের সম্পূর্ণ পৃথক kernel রয়েছে। এটি আপনার নিজের বিশ্বস্ত application code-এর অনেক instance চালানোর জন্য একটি গ্রহণযোগ্য trade-off, সমতুল্য isolation শক্তির দাবি নয়।
</content>
