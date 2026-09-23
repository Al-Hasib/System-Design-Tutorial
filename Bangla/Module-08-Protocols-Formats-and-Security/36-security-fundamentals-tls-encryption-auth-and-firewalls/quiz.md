# অনুশীলন ও ইন্টারভিউ প্রশ্ন (Practice & Interview Questions)

**১. Encryption এবং hashing-এর মধ্যে মূল পার্থক্য কী, এবং কেন password encrypt না করে hash করা উচিত?**
Encryption reversible — key থাকা যে কেউ মূল data পুনরুদ্ধার করতে পারে। Hashing one-way এবং reverse করা যায় না। Password hash করা উচিত (bcrypt বা Argon2-এর মতো slow, salted algorithm দিয়ে) যাতে database leak হলেও, মূল password পুনরুদ্ধার করা computationally infeasible হয় — সেগুলো encrypt করার মানে হলো decryption key পাওয়া যে কেউ সরাসরি প্রতিটি password পুনরুদ্ধার করতে পারবে।

**২. TLS কেন শুধুমাত্র একটির পরিবর্তে asymmetric এবং symmetric encryption উভয়ই ব্যবহার করে?**
Asymmetric encryption key-distribution সমস্যা সমাধান করে (একটি public key প্রকাশ্যে share করা যায়) কিন্তু একটি সম্পূর্ণ session-এর data encrypt করার জন্য খুব ধীর। Symmetric encryption দ্রুত কিন্তু উভয় পক্ষের আগে থেকেই একটি secret key share করা প্রয়োজন। TLS handshake-এর সময় সংক্ষেপে asymmetric crypto ব্যবহার করে নিরাপদে একটি random symmetric key-তে সম্মত হয়, তারপর actual session data-র জন্য দ্রুত symmetric encryption ব্যবহার করে।

**৩. Authentication এবং authorization-এর মধ্যে পার্থক্য ব্যাখ্যা করুন, এবং এটি কীভাবে HTTP status code-এর সাথে মিলে যায়।**
Authentication যাচাই করে একটি party কে; authorization যাচাই করে সেই (ইতিমধ্যে-identified) party কী করার অনুমতি পায়। একটি 401 Unauthorized response মানে authentication ব্যর্থ হয়েছে বা অনুপস্থিত (server জানে না আপনি কে)। একটি 403 Forbidden response মানে authentication সফল হয়েছে কিন্তু authenticated party requested action সম্পাদন করার অনুমতি পায় না।

**৪. Server-side session-এর তুলনায় authentication-এর জন্য JWT ব্যবহার করলে কী trade-off আসে?**
JWT self-contained এবং signed, তাই যেকোনো service একটি central session store-এ round trip ছাড়াই সেগুলো verify করতে পারে — stateless, horizontally-scaled সিস্টেমের জন্য ভালো। Trade-off হলো revocation: যেহেতু token নিজেই identity-র প্রমাণ এবং expire না হওয়া পর্যন্ত valid, তাই এর natural expiry-র আগে একটি নির্দিষ্ট token তাৎক্ষণিকভাবে invalidate করা server-side session-এর চেয়ে কঠিন, যেখানে আপনি শুধু session record delete করতে পারেন।

**৫. mTLS কী, এবং এটি সাধারণ HTTPS browsing-এ ব্যবহৃত TLS থেকে কীভাবে আলাদা?**
সাধারণ HTTPS/TLS-এ, শুধুমাত্র server client-কে তার identity প্রমাণ করতে একটি certificate প্রদর্শন করে। mTLS (mutual TLS) উভয় পক্ষকে certificate প্রদর্শন করতে বলে — client-ও server-কে তার identity প্রমাণ করে — যা service-to-service communication-এর জন্য service-দের একে অপরের identity cryptographically verify করার উপায়।

**৬. Istio বা Linkerd-এর মতো একটি service mesh সাধারণত একটি microservices architecture জুড়ে mTLS কীভাবে implement করে?**
প্রতিটি service instance-এর পাশে deploy করা sidecar proxy-র মাধ্যমে, যা প্রতিটি service-to-service call-এর জন্য স্বয়ংক্রিয়ভাবে mTLS handshake এবং certificate management পরিচালনা করে, তাই individual service-দের নিজেদের mTLS implement করতে হয় না — mesh transparently infrastructure layer-এ mutual authentication এবং encryption প্রয়োগ করে।

**৭. একটি firewall/security group এবং একটি Web Application Firewall (WAF)-এর মধ্যে পার্থক্য কী?**
একটি firewall বা security group network layer-এ কাজ করে, IP, port, এবং protocol-এর ভিত্তিতে কোন source কোন destination-এ পৌঁছাতে পারে তা নিয়ন্ত্রণ করে। একটি WAF application (HTTP) layer-এ কাজ করে, request content নিজেই SQL injection বা cross-site scripting-এর মতো পরিচিত attack pattern-এর জন্য পরীক্ষা করে এবং application code-এ পৌঁছানোর আগেই সেগুলো block করে।

**৮. Network segmentation কী, এবং এটি কেন একটি breach-এর blast radius সীমিত করে?**
Network segmentation infrastructure-কে বিভিন্ন trust level-এর zone-এ ভাগ করে (যেমন, public web tier, internal service, database tier), যা firewall/security group rule দ্বারা প্রয়োগ করা হয় যাতে শুধুমাত্র নির্দিষ্ট, প্রত্যাশিত ট্রাফিক zone boundary অতিক্রম করতে পারে। যদি একজন attacker একটি zone (যেমন একটি public-facing service) compromise করে, segmentation তাদের সেই boundary নিয়ন্ত্রণকারী rule ভঙ্গ না করে সরাসরি আরও sensitive zone-এ (যেমন database) পৌঁছাতে বাধা দেয়।

**৯. "Defense in depth" ব্যাখ্যা করুন এবং কেন একটি সিস্টেমের শুধুমাত্র একটি শক্তিশালী security control-এর উপর নির্ভর করা উচিত নয়।**
Defense in depth মানে একাধিক স্বাধীন security control স্তরে সাজানো যাতে কোনো একক failure point সম্পূর্ণ compromise-এর দিকে না নিয়ে যায়। প্রতিটি control ব্যর্থ হতে পারে বা ভুলভাবে configure হতে পারে — database নিজেই compromised হলে এবং password hash না করা হলে TLS সাহায্য করে না; একটি firewall rule ভুলভাবে configure হতে পারে, তাই "internal" ট্রাফিকের জন্যও authentication/authorization চেক গুরুত্বপূর্ণ থাকে। প্রতিটি স্তর এমনভাবে ডিজাইন করা হয় যাতে অন্য একটি স্তর যা মিস করতে পারে তা ধরতে পারে।

**১০. দৃশ্যপট: একজন attacker আপনার cluster-এর internal network-এর ভেতরে একটি microservice compromise করে। এই video থেকে কোন দুটি control তারা পরে কী করতে পারে তা সীমিত করবে, এবং কীভাবে?**
mTLS/service mesh — compromised service এখনো অন্য service-দের impersonate করতে পারবে না বা এমন service-দের মধ্যে ট্রাফিক decrypt করতে পারবে না যেখানে সে একটি প্রকৃত পক্ষ নয়, যেহেতু প্রতিটি call-এর জন্য একটি valid certificate প্রয়োজন। Network segmentation/security groups — compromised service সম্ভবত এখনো শুধুমাত্র তার নিজস্ব security group rule যেগুলো অনুমতি দেয় সেই নির্দিষ্ট destination-এই পৌঁছাতে পারবে, পুরো internal network নয়, যা lateral movement সীমিত করে।

**১১. OAuth 2.0 এবং OpenID Connect (OIDC) প্রতিটি কী সমাধান করে, এবং এগুলো একে অপরের সাথে কীভাবে সম্পর্কিত?**
OAuth 2.0 delegated authorization সমাধান করে — একজন user-কে তার password share না করেই একটি third-party app-কে তার data-তে সীমিত access দিতে দেয়। OIDC হলো OAuth2-এর উপরে তৈরি একটি identity layer যা *কে* এই user তা প্রতিষ্ঠা করার (authentication) উপায়ও standardize করে, যে কারণে "Sign in with Google" flow OAuth2-এর authorization mechanics-এর উপরে OIDC ব্যবহার করে।

**১২. সত্য নাকি মিথ্যা: একবার ট্রাফিক আপনার internal network-এর ভেতরে (perimeter firewall পার হয়ে), service-দের মধ্যে authentication/authorization চেক এড়িয়ে যাওয়া নিরাপদ।**
মিথ্যা — এটি ঠিক সেই ধারণা যার বিরুদ্ধে "zero trust" এবং defense-in-depth যুক্তি দেয়। একটি perimeter firewall ভুলভাবে configure করা বা bypass করা যেতে পারে, এবং একজন attacker যে কোনো একটি single internal service compromise করে সে স্বয়ংক্রিয়ভাবে অন্য প্রতিটি service-এ trusted access পাওয়া উচিত নয়। Internal call-গুলো এখনো authenticated হওয়া উচিত (যেমন, mTLS-এর মাধ্যমে) এবং স্বাধীনভাবে authorized হওয়া উচিত।
