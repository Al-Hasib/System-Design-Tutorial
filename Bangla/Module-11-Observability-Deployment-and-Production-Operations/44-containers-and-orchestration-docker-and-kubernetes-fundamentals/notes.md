# স্টাডি নোটস: Containers & Orchestration

## সংজ্ঞাসমূহ (Definitions)

- **Container:** host-এর kernel share করা একটি OS-স্তরের virtualized একক, যার নিজস্ব isolated filesystem, process namespace, এবং network stack আছে (Linux namespaces/cgroups-এর মাধ্যমে)।
- **Virtual machine (VM):** একটি সম্পূর্ণ virtualized কম্পিউটার যার নিজস্ব OS kernel-সহ, যা একটি hypervisor-এর উপর চলে।
- **Container image:** একটি application, এর runtime, এবং dependency-র একটি অপরিবর্তনীয়, প্যাকেজড স্ন্যাপশট, যা একটি Dockerfile থেকে build হয় এবং যেকোনো জায়গায় অভিন্নভাবে চলে।
- **Registry:** container image-এর জন্য একটি storage/distribution service (যেমন, Docker Hub, প্রাইভেট registry)।
- **Orchestrator:** একটি system (যেমন, Kubernetes) যা অনেক machine জুড়ে অনেক container schedule, scale, heal, এবং পরিচালনা করে।
- **Pod:** Kubernetes-এ ক্ষুদ্রতম deployable একক — এক বা একাধিক নিবিড়ভাবে সংযুক্ত container যা একসাথে schedule হয়, একটি network namespace share করে।
- **Deployment:** একটি Kubernetes object যা কাঙ্ক্ষিত অবস্থা বর্ণনা করে (যেমন, "image X-এর 5টি replica"); Kubernetes ক্রমাগত বাস্তবতাকে এই অবস্থার দিকে reconcile করে।
- **Service:** একটি স্থিতিশীল network পরিচয়/ঠিকানা যা একটি Deployment-এর বর্তমানে healthy pod-গুলোর যে কোনোটির দিকে route করে।
- **Liveness probe:** একটি health check যা নির্ধারণ করে একটি container-কে kill করে পুনরায় চালু করা উচিত কিনা।
- **Readiness probe:** একটি health check যা নির্ধারণ করে একটি pod বর্তমানে ট্রাফিক গ্রহণ করা উচিত কিনা।

## Containers বনাম Virtual Machines

| দিক | Virtual Machine | Container |
|---|---|---|
| Virtualize করে | সম্পূর্ণ কম্পিউটার + নিজস্ব OS kernel | শুধুমাত্র OS-স্তরে (host kernel share করে) |
| আকার | গিগাবাইট | মেগাবাইট |
| শুরু হওয়ার সময় | কয়েক দশ সেকেন্ড থেকে মিনিট | মিলিসেকেন্ড থেকে কয়েক সেকেন্ড |
| Isolation শক্তি | শক্তিশালী (পৃথক kernel) | দুর্বল (share করা kernel) |
| সাধারণ ব্যবহার | বিভিন্ন OS চালানো, শক্তিশালী multi-tenant isolation | বিশ্বস্ত application code-এর অনেক instance প্যাকেজ ও চালানো |

## Kubernetes-এর মূল Object

| Object | দায়িত্ব |
|---|---|
| Pod | ক্ষুদ্রতম deployable একক; এক বা একাধিক co-located container |
| Deployment | কাঙ্ক্ষিত replica সংখ্যা/অবস্থা; বাস্তব অবস্থাকে মেলাতে reconcile করে |
| Service | Deployment-এর healthy pod জুড়ে load-balance করা একটি স্থিতিশীল ঠিকানা |
| Liveness probe | একটি hung/crashed container শনাক্ত করে → restart trigger করে |
| Readiness probe | একটি pod ট্রাফিকের জন্য প্রস্তুত কিনা শনাক্ত করে → Service routing নিয়ন্ত্রণ করে, restart নেই |

## একটি Orchestrator আসলে যা প্রদান করে

- **Scheduling:** resource-এর ভিত্তিতে কোন machine কোন container চালাবে তা ঠিক করা।
- **Scaling:** load-এর ভিত্তিতে instance যোগ/বাদ দেওয়া (horizontal scaling, স্বয়ংক্রিয়)।
- **Self-healing:** crash হওয়া container পুনরায় চালু করা; একটি machine fail করলে পুনরায় schedule করা।
- **Service discovery/load balancing:** কোন নির্দিষ্ট instance বিদ্যমান তা নির্বিশেষে স্থিতিশীল addressing।
- **Rolling updates:** পুরনো instance-গুলোকে ধীরে ধীরে নতুন দিয়ে প্রতিস্থাপন করা (zero-downtime deploy-এর পেছনের মেকানিজম)।

## মূল সংখ্যা/তথ্য (Key Numbers / Facts)

- Docker প্রায় ২০১৩ সাল থেকে মূলধারার backend ব্যবহারের জন্য container-কে জনপ্রিয় করে তোলে; অন্তর্নিহিত Linux kernel feature (namespaces, cgroups) আগে থেকেই ছিল।
- Kubernetes-এর উৎপত্তি Google-এ (Borg-এর মতো internal system-এর ভিত্তিতে) এবং এটি ২০১৪ সালে open-source করা হয়েছিল।
- Container image layer-এ layer-এ build হয়; অপরিবর্তিত layer-গুলো cache করা হয় এবং পুনরায় ব্যবহার করা হয়, যা build এবং pull-কে দ্রুত করে।

## সারসংক্ষেপ (Summary)

- Container-গুলো VM-এর তুলনায় হালকা, দ্রুত-শুরু হওয়া বিকল্প, দক্ষতার বিনিময়ে কিছুটা isolation শক্তি বিসর্জন দেয় — আজকের backend service deploy করার ডিফল্ট একক।
- Image-গুলো deployment-কে reproducible এবং portable করে তোলে, "works on my machine" নির্মূল করে।
- Kubernetes Pod, Deployment, এবং Service-এর মাধ্যমে container-এর জন্য scheduling, scaling, healing, এবং traffic routing স্বয়ংক্রিয় করে, liveness/readiness probe ব্যবহার করে জানতে যে আসলে কী healthy।
</content>
