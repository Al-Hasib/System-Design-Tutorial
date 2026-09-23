# Security Fundamentals: TLS, Encryption, AuthN/AuthZ & Firewalls

**কঠিনতার মাত্রা:** Intermediate/Advanced

## শেখার লক্ষ্য (Learning Objectives)

- Encryption এবং hashing-এর মধ্যে পার্থক্য বোঝা এবং কখন কোনটি ব্যবহার করা হয় তা ব্যাখ্যা করা।
- Symmetric বনাম asymmetric encryption ব্যাখ্যা করা এবং TLS কীভাবে দুটোকে একত্রে ব্যবহার করে তা বোঝা।
- Authentication এবং authorization-এর মধ্যে স্পষ্ট পার্থক্য করা, এবং প্রতিটির সাধারণ mechanism (sessions, JWTs, OAuth2/OIDC) বর্ণনা করা।
- mTLS ব্যাখ্যা করা এবং service mesh কীভাবে service-to-service ট্রাফিক সুরক্ষিত করতে এটি ব্যবহার করে তা বোঝা।
- একটি defense-in-depth strategy-তে firewall, network segmentation, এবং Web Application Firewall (WAF)-এর ভূমিকা বর্ণনা করা।
- সিস্টেম-ডিজাইন পর্যায়ে বোঝা যে কেন defense-in-depth (একাধিক স্বাধীন নিরাপত্তা স্তর) যেকোনো একক control-এর চেয়ে বেশি গুরুত্বপূর্ণ।

## স্ক্রিপ্ট (Script)

### Hook / Intro

এই কোর্সের এখন পর্যন্ত প্রতিটি টপিক নীরবে ধরে নিয়েছে যে নেটওয়ার্ক ডেটা পাঠানোর জন্য একটি নিরাপদ জায়গা। কিন্তু তা নয়। নেটওয়ার্ক পথে থাকা যেকেউ — একটি compromised router, একটি malicious Wi-Fi hotspot, একজন attacker যে আপনার fleet-এর একটি service ভেঙেছে — সম্ভাব্যভাবে ট্রাফিক পড়তে বা পরিবর্তন করতে পারে, একজন user-কে impersonate করতে পারে, অথবা একটি সিস্টেম থেকে আরেকটিতে laterally move করতে পারে। Security কোনো আলাদা specialty নয় যা system design-এর পরে জোড়া লাগানো হয়; এটি scalability বা availability-এর মতোই একটি first-class design constraint। আজ আমরা প্রতিটি backend engineer-এর জানা প্রয়োজন এমন fundamentals কভার করব: encryption, authentication এবং authorization-এর মধ্যে গুরুত্বপূর্ণ পার্থক্য, এবং network-level defense যা একটি layer ব্যর্থ হলেও সিস্টেমকে নিরাপদ রাখে।

### Encryption বনাম Hashing

এই দুটি প্রায়ই গুলিয়ে ফেলা হয়, তাই চলুন এগুলোকে পরিষ্কারভাবে আলাদা করি। **Encryption** reversible: একটি key ব্যবহার করে data রূপান্তরিত করা হয় যাতে সঠিক key থাকা যে কেউ এটিকে decrypt করে মূল data-তে ফিরিয়ে আনতে পারে — এটি ব্যবহার করা হয় যখন আপনার পরে মূল data *পুনরুদ্ধার* করা দরকার (data in transit, disk-এ data at rest)। **Hashing** one-way: এটি data-র একটি fixed-size fingerprint তৈরি করে যা মূল data-তে ফিরিয়ে আনা যায় না — এটি ব্যবহার করা হয় যখন আপনার শুধু কিছু মিলছে কিনা তা *যাচাই* করা দরকার, কখনো পুনরুদ্ধার নয়, এবং এই কারণেই password hash করা উচিত (bcrypt বা Argon2-এর মতো slow, salted algorithm দিয়ে), কখনো encrypt নয় — যদি আপনার database leak হয়, আপনি চান যে মূল password পুনরুদ্ধার করা computationally infeasible হোক, শুধু অসুবিধাজনক নয়।

Encryption-এর মধ্যে দুটি পরিবার আছে। **Symmetric encryption** (AES আজকের standard) encrypt এবং decrypt উভয়ের জন্য একটি shared secret key ব্যবহার করে — এটি দ্রুত, কিন্তু উভয় পক্ষের কাছে আগে থেকেই একই key থাকতে হবে, যা একটি chicken-and-egg সমস্যা তৈরি করে: প্রথমে আপনি নিরাপদে key কীভাবে share করবেন? **Asymmetric encryption** (RSA, elliptic-curve cryptography) একটি mathematically-linked key pair ব্যবহার করে: একটি public key যা যে কারো কাছে থাকতে পারে, এবং একটি private key যা শুধু owner-এর কাছে থাকে। Public key দিয়ে encrypt করা data শুধু private key দিয়েই decrypt করা যায়। এটি symmetric encryption-এর চেয়ে অনেক ধীর, কিন্তু এটি key-distribution সমস্যা সমাধান করে — আপনি আপনার public key প্রকাশ্যে publish করতে পারেন।

### TLS: দুটোকে একত্রে ব্যবহার

এই কারণেই TLS (যা আমরা video 6-এ HTTPS-এর জন্য স্পর্শ করেছিলাম) দুটোই ব্যবহার করে। TLS handshake-এর সময়, asymmetric encryption শুধু ততক্ষণ ব্যবহার করা হয় যতক্ষণ server-কে (তার certificate-এর মাধ্যমে) authenticate করা এবং নিরাপদে একটি fresh, random symmetric key সম্মত হওয়ার জন্য প্রয়োজন — slow-but-safe পদ্ধতি ব্যবহার করে key-distribution সমস্যা সমাধান করে। এরপর actual session-এর প্রতিটি byte সেই negotiated key দিয়ে দ্রুত symmetric encryption ব্যবহার করে। আপনি asymmetric crypto-র key-distribution নিরাপত্তা এবং symmetric crypto-র গতি উভয়ই পান, পুরো, সম্ভাব্যভাবে বড়, data transfer-এর জন্য asymmetric encryption-এর খরচ না দিয়ে।

### Authentication বনাম Authorization

যদি এই video থেকে একটি পার্থক্যই মনে রাখেন, তাহলে এটিই রাখুন, কারণ interviewer-রা এটি ক্রমাগত জিজ্ঞাসা করে এবং production incident ঘটে কারণ engineer-রা এগুলো গুলিয়ে ফেলে: **Authentication (AuthN)** "আপনি কে?" এই প্রশ্নের উত্তর দেয় — identity যাচাই করা। **Authorization (AuthZ)** "আপনি কী করার অনুমতি পান?" এই প্রশ্নের উত্তর দেয় — permission চেক করা, এবং এটি শুধুমাত্র authentication ইতিমধ্যে কে জিজ্ঞাসা করছে তা প্রতিষ্ঠিত করার *পরেই* অর্থপূর্ণ হয়। এটি সরাসরি video 6-এর HTTP status code-এর সাথে মিলে যায়: 401 Unauthorized আসলে মানে "authentication ব্যর্থ হয়েছে, আমি জানি না আপনি কে"; 403 Forbidden মানে "আমি জানি আপনি কে, এবং আপনি এটি করার অনুমতি পান না।"

সাধারণ authentication mechanism: **session-based auth**, যেখানে server একটি opaque session ID ইস্যু করে যা একটি cookie-তে সংরক্ষিত থাকে এবং actual session state server-side (অথবা Redis-এর মতো একটি shared store-এ) রাখে; **JWT (JSON Web Tokens)**, self-contained signed token যা identity claim সরাসরি token-এর মধ্যেই বহন করে, যা যেকোনো service-কে central session store-এ round trip ছাড়াই signature যাচাই করতে দেয় — stateless, horizontally-scaled architecture-এ কার্যকর, তবে খরচ হলো immediate revocation আরও কঠিন হয়ে যায় কারণ token expire না হওয়া পর্যন্ত valid থাকে; এবং **OAuth 2.0 / OpenID Connect**, delegated authorization এবং identity-র জন্য standard — OAuth2 একজন user-কে তার password share না করেই একটি third-party app-কে তার data-তে সীমিত access দিতে দেয়, এবং OIDC OAuth2-এর authorization flow-এর উপরে standardized identity (এই user কে) স্তর যোগ করে। এটি সেই "Sign in with Google" pattern যা আপনি শত শত বার ব্যবহার করেছেন।

### Service-to-Service Communication সুরক্ষিত করা

উপরের সবকিছু একজন client-এর আপনার সিস্টেমে authenticate করা কভার করে। কিন্তু একটি microservices architecture-এর ভেতরে (Module 7 মনে করুন), service-দেরও একে অপরকে trust করতে হয়। **mTLS (mutual TLS)** TLS handshake-কে বাড়িয়ে দেয় যাতে *উভয়* পক্ষ certificate প্রদর্শন করে — client server-কে তার identity প্রমাণ করে এবং server client-কে তার identity প্রমাণ করে, শুধু server authenticate করার পরিবর্তে যেমন সাধারণ HTTPS-এ হয়। এটি ঠিক সেই mechanism যা Istio এবং Linkerd-এর মতো service mesh (video 31-এ সংক্ষেপে উল্লেখ করা হয়েছিল) sidecar proxy-র মাধ্যমে automate করে: প্রতিটি service-to-service call স্বয়ংক্রিয়ভাবে mTLS-এ wrap করা হয়, তাই একজন attacker আপনার network-এ foothold পেলেও, তারা সহজে অন্য কোনো service-কে impersonate করতে বা internal ট্রাফিক eavesdrop করতে পারে না — প্রতিটি internal call-এর জন্য এখনো একটি valid, verified certificate প্রয়োজন।

### Firewalls এবং Network Segmentation

Network layer-এ, একটি **firewall** rule-এর ভিত্তিতে একটি নির্দিষ্ট resource-এ কী ট্রাফিক পৌঁছাতে পারবে তা নিয়ন্ত্রণ করে — source/destination IP, port, protocol। Cloud environment-এ এটি সাধারণত security group বা network ACL-এর রূপ নেয়, যা আপনাকে বলতে দেয় "শুধুমাত্র load balancer port 443-এ application server-এ পৌঁছাতে পারবে; শুধুমাত্র application server port 5432-এ database-এ পৌঁছাতে পারবে" — একই network-এ থাকলেও আর কিছু পৌঁছাতে পারবে না। এটি হলো **network segmentation**: আপনার infrastructure-কে বিভিন্ন trust level-এর zone-এ ভাগ করা, যাতে একটি zone-এ (যেমন একটি public-facing web tier) কোনো compromise স্বয়ংক্রিয়ভাবে আরও sensitive zone-এ (যেমন আপনার database বা internal admin tool) access না দেয়। একটি **Web Application Firewall (WAF)** একটি স্তর উপরে কাজ করে, HTTP ট্রাফিক নিজেই পরিচিত attack pattern-এর জন্য পরীক্ষা করে — SQL injection চেষ্টা, cross-site scripting payload — এবং সেগুলো আপনার application code-এ পৌঁছানোর আগেই block করে, যা আপনার application-এর নিজস্ব input validation-কে replace না করে complement করে।

### Defense in Depth

এই video-র সবকিছুর পেছনে একীভূতকারী নীতি হলো **defense in depth**: কোনো একক control নিখুঁত ধরে নেওয়া হয় না, তাই আপনি একাধিক স্বাধীন defense স্তরে সাজান যাতে একটি ব্যর্থ হলে সম্পূর্ণ compromise না হয়। TLS transit-এ ট্রাফিক encrypt করে, কিন্তু আপনি এখনো at rest password hash করেন যদি database নিজেই কখনো exposed হয়। একটি firewall আপনার database-এ কী পৌঁছাতে পারে তা সীমিত করে, কিন্তু আপনি এখনো প্রতিটি request-এ authentication এবং authorization প্রয়োজন করেন যদি firewall rule কখনো ভুলভাবে configure করা হয়। mTLS internal service ট্রাফিক সুরক্ষিত করে, কিন্তু আপনি এখনো service level-এ প্রতিটি request validate এবং authorize করেন, network perimeter পার হয়ে যাওয়া যেকোনো কিছু trust করার পরিবর্তে — একটি পদ্ধতি যাকে প্রায়ই "zero trust" বলা হয়। এই স্তরগুলোর কোনোটিই optional নয় কারণ আরেকটি বিদ্যমান; প্রতিটি নির্দিষ্টভাবে সেই দৃশ্যপটের জন্য আছে যেখানে একটি ভিন্ন স্তর ব্যর্থ হয়।

### Real-World Example

একটি payments feature end-to-end বিবেচনা করুন। Mobile app HTTPS (TLS, video 6 থেকে) দিয়ে আপনার API gateway-তে connect হয়, যা user-কে authenticate করতে একটি JWT validate করে এবং "charge card" action অনুমতি দেওয়ার আগে তার authorization চেক করে — token invalid হলে 401, authenticated কিন্তু এই নির্দিষ্ট account charge করার অনুমতি না থাকলে 403। Gateway request-টি mTLS-এর মাধ্যমে একটি payments microservice-এ forward করে, তাই payments service যাচাই করতে পারে যে এই call সত্যিই trusted gateway থেকে এসেছে এবং কোনো অসম্পর্কিত internal service ভেঙে ফেলা attacker থেকে নয়। একটি security group নিশ্চিত করে যে শুধুমাত্র payments service — cluster-এ আর কিছু নয় — tokenized card data ধারণকারী database-এ পৌঁছাতে পারে, এবং সেই data at rest encrypted। এবং gateway-র সামনে একটি WAF স্পষ্ট injection চেষ্টা block করে এগুলোর যেকোনো একটিতে পৌঁছানোর আগেই। প্রতিটি স্তর ধরে নেয় যে অন্যগুলো ব্যর্থ হতে পারে, এবং তাদের কোনোটিই একা "the" security control নয়।

### Recap

Encryption reversible (confidentiality-র জন্য ব্যবহৃত), hashing one-way (verification-এর জন্য ব্যবহৃত, যেমন password)। TLS slow-but-key-distributable asymmetric encryption-কে fast symmetric encryption-এর সাথে একত্রিত করে। Authentication প্রতিষ্ঠা করে আপনি কে; authorization প্রতিষ্ঠা করে আপনি কী করার অনুমতি পান — এগুলো ভিন্ন প্রশ্ন, যা 401 বনাম 403-এ mapped। mTLS TLS-কে বাড়িয়ে দেয় যাতে service-রা পারস্পরিকভাবে একে অপরকে authenticate করে, যা service mesh automate করে। Firewall এবং network segmentation network level-এ কী কী পৌঁছাতে পারে তা সীমিত করে, এবং একটি WAF HTTP layer-এ পরিচিত attack pattern block করে। এর কোনোটিই বিচ্ছিন্নভাবে কাজ করে না — defense in depth মানে প্রতিটি স্তর ধরে নেয় যে অন্যগুলো ব্যর্থ হতে পারে।

### What's Next

এটি Module 8 বন্ধ করে দেয় — আমরা এখন transport layer থেকে message format এবং security পর্যন্ত সম্পূর্ণভাবে চলে গেছি। এখান থেকে, Module 9 আরও দুটি বিষয়ে গভীরে যায় যা আমরা এখন পর্যন্ত শুধুমাত্র conceptual level-এ কভার করেছি: একটি database আসলে কীভাবে concurrent transaction-এর মধ্যে isolation প্রয়োগ করে, এবং এর storage engine কীভাবে physically disk-এ data persist করে — সাথে REST-এর একটি সত্যিকারের বিকল্প হিসেবে GraphQL-এর দিকে একটি নজর।

## মূল বিষয়গুলো (Key Takeaways)

- Encryption reversible (confidentiality-র জন্য); hashing one-way (verification-এর জন্য) — সবসময় password hash করুন, কখনো শুধু encrypt করবেন না।
- Symmetric encryption দ্রুত কিন্তু একটি pre-shared key প্রয়োজন; asymmetric encryption key distribution সমাধান করে কিন্তু ধীর — TLS সংক্ষেপে asymmetric crypto ব্যবহার করে একটি symmetric key exchange করতে, তারপর সেই fast symmetric key দিয়ে session encrypt করে।
- Authentication (আপনি কে?) এবং authorization (আপনি কী করতে পারেন?) আলাদা ধাপ — যথাক্রমে HTTP 401 বনাম 403-এ mapped — এবং authorization শুধুমাত্র authentication-এর পরেই অর্থপূর্ণ।
- Sessions, JWTs, এবং OAuth2/OIDC হলো client authenticate করার এবং access delegate করার standard mechanism; mTLS হলো service-রা কীভাবে একে অপরকে authenticate করে, সাধারণত একটি service mesh দ্বারা automated।
- Firewalls/security group network segmentation প্রয়োগ করে, এবং একটি WAF পরিচিত HTTP-layer attack block করে — কিন্তু defense in depth মানে কোনো একক স্তর একা trusted নয়; প্রতিটি স্তর ধরে নেয় যে অন্যটি ব্যর্থ হতে পারে।
