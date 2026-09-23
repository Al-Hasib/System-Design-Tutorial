# Study Notes: Zero-Downtime Deployments & Database Migrations

## সংজ্ঞা

- **Rolling deployment:** একবারে অল্প কয়েকটা করে পুরনো instance-কে নতুন instance দিয়ে replace করা, যাতে পুরো সময় জুড়ে মোট ready-instance pool মোটামুটি স্থির থাকে।
- **Blue-green deployment:** নতুন version নিয়ে একটা সম্পূর্ণ দ্বিতীয় environment চালানো, তারপর যাচাই হয়ে গেলে সব traffic একবারে atomically সুইচ করা।
- **Canary deployment:** প্রথমে নতুন version-কে traffic-এর একটা ছোট percentage-এ ছাড়া, metric দেখতে দেখতে ধীরে ধীরে সেটা বাড়ানো।
- **Expand-contract pattern:** একটা multi-step database migration discipline — আগে নতুন schema structure যোগ করা, সেটা ব্যবহার করার জন্য code/data migrate করা, তারপর নিরাপদ হলে তবেই পুরনো structure সরানো।

## Deployment Strategy তুলনা

| Strategy | Infra খরচ | Rollback গতি | খারাপ deploy-এর blast radius | জটিলতা |
|---|---|---|---|---|
| Rolling | কম (বিদ্যমান capacity পুনরায় ব্যবহার করে) | ধীর (ক্রমান্বয়ে পুনরায়-rollout) | rollout এগোনোর সাথে সাথে বাড়ে | কম — প্রায়ই platform-এর default |
| Blue-green | বেশি (সাময়িকভাবে ২x environment) | দ্রুত (router আবার সুইচ করে) | প্রতিটি cutover-এ সব-অথবা-কিছুই না | মাঝারি |
| Canary | কম-মাঝারি | দ্রুত (ramp থামানো/rollback করা যায়) | শুরুতে ছোট, সীমাবদ্ধ | মাঝারি-উচ্চ (metric-পর্যবেক্ষণ/automation দরকার) |

## কেন পুরনো ও নতুন version একসাথে থাকতে হয়

- প্রতিটি zero-downtime strategy-তে এমন একটা window থাকে যেখানে পুরনো ও নতুন version একসাথে traffic serve করে (rolling: পুরো rollout জুড়ে; blue-green: cutover যাচাইয়ের সময় সংক্ষিপ্তভাবে; canary: পুরো ramp জুড়ে)।
- এর মানে deploy-এর সময় API/message/schema পরিবর্তন সেই window-এর জন্য backward-compatible হতে হবে — additive, ধ্বংসাত্মক নয় (Module 8-এর Protocol Buffers schema evolution-এরই প্রতিধ্বনি)।

## Expand-Contract Pattern (Database Migrations)

| ধাপ | কী ঘটে | কীভাবে deploy হয় |
|---|---|---|
| ১. Expand | নতুন column/table/index যোগ করা; পুরনো code সেটা উপেক্ষা করে | শুধু-migration deploy, কোনো app code পরিবর্তন নেই |
| ২. Migrate | নতুন app code পুরনো ও নতুন — উভয় structure-এই write করে; পুরনো data backfill করা হয় | App code deploy |
| ৩. Contract | পুরোপুরি rollout ও backfill হয়ে গেলে, পুরনো structure-এ write করা বন্ধ; read সম্পূর্ণভাবে নতুনটায় সরে যায় | App code deploy |
| ৪. Cleanup | এখন-অব্যবহৃত পুরনো structure drop করা | শুধু-migration deploy, পরে |

- একটা atomic ধাপে কখনো একটা column-এর নাম বদলাবেন না/মুছবেন না এবং একই সাথে application code-কে সেটার দিকে সুইচ করবেন না — rollout-এর মাঝখানে তখনও চলমান পুরনো-version pod-গুলো সাথে সাথেই ভেঙে পড়বে।

## গুরুত্বপূর্ণ সংখ্যা/তথ্য

- Kubernetes Deployments default-ভাবে একটা rolling update strategy ব্যবহার করে, যা `maxSurge`/`maxUnavailable` parameter দিয়ে configure করা যায়, যা নিয়ন্ত্রণ করে rollout-এর মাঝখানে কতগুলো অতিরিক্ত/unavailable pod অনুমোদিত।
- Canary rollout প্রায়ই progressive-delivery tool (যেমন, Argo Rollouts, Flagger) দিয়ে automate করা হয়, যেগুলো metric দেখে এবং নিজে থেকেই traffic percentage এগিয়ে নেয় বা auto-rollback করে।

## সারসংক্ষেপ

- সব zero-downtime strategy ready-instance pool-কে শূন্যের উপরে রাখে কিন্তু কিছু window-এর জন্য পুরনো ও নতুন version-কে একসাথে থাকতে হয় — সেই window-এর সময় backward-compatible পরিবর্তন প্রয়োজন।
- Rolling = সস্তা কিন্তু rollback ধীর; blue-green = দ্রুত rollback কিন্তু সাময়িকভাবে দ্বিগুণ infrastructure খরচ; canary = সবচেয়ে ছোট blast radius কিন্তু সবচেয়ে ধীর, সবচেয়ে বেশি operational জড়িত rollout।
- Database schema পরিবর্তনেও একই সহাবস্থান discipline দরকার, যা expand-contract হিসেবে formal করা হয়েছে: যোগ করো, migrate করো, তারপর সরাও — একসাথে সব নয়।
