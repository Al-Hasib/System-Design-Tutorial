# Client-Server Architecture & How the Internet Works (DNS, IP, TCP/IP, HTTP)

**কঠিনতার মাত্রা:** Beginner

## শেখার লক্ষ্য (Learning Objectives)

- client-server model ব্যাখ্যা করা এবং এটি peer-to-peer architecture থেকে কীভাবে আলাদা তা বোঝা।
- আপনি যখন একটি browser-এ URL টাইপ করে Enter চাপেন, তখন ধাপে ধাপে কী ঘটে তা বর্ণনা করা।
- DNS কী এবং কেন এটি domain name-কে IP address-এ রূপান্তর করে তা বোঝা।
- IP address কী এবং TCP/IP protocol suite-এর মূল ভূমিকা বোঝা।
- HTTP কী, request/response কীভাবে গঠিত হয়, এবং এটি পুরো stack-এর কোথায় বসে তা বোঝা।

## স্ক্রিপ্ট (Script)

### Hook / ভূমিকা

আপনি browser-এ "google.com" টাইপ করেন, Enter চাপেন, এবং সেকেন্ডের এক ভগ্নাংশ পরেই একটি page হাজির হয়ে যায়। সেই একটিমাত্র, সাধারণ কাজটি আসলে একাধিক protocol, বিশ্বজুড়ে ছড়িয়ে থাকা server, এবং networking-এর একাধিক layer জড়িত এক অবিশ্বাস্য ঘটনার শৃঙ্খল শুরু করে দেয় — এবং সবকিছু এক সেকেন্ডেরও অনেক কম সময়ে সম্পন্ন হয়ে যায়। আপনি যদি এমন সব system design করতে চান যা internet-এ চলে, তাহলে এই pipeline-টি আপনাকে বুঝতেই হবে, কারণ আপনার system-এ আসা প্রতিটি request এই পথ ধরেই আসে। চলুন, click থেকে response পর্যন্ত পুরো পথটি খুঁজে বের করি।

### Client-Server Model

প্রথমে, মৌলিক architecture pattern: **client-server**। এই model-এ, একজন **client** — আপনার browser, আপনার phone-এর app, বা অন্য কোনো backend service — একটি request শুরু করে। একটি **server** — অন্য কোথাও থাকা একটি machine (বা machine-এর একটি বহর) — request-এর জন্য অপেক্ষা করে, সেগুলো process করে, এবং একটি response ফেরত পাঠায়। client জিজ্ঞাসা করে; server উত্তর দেয়। এটি অসম (asymmetric) এবং এটি ইচ্ছাকৃতভাবেই এমন: server যে চালায় সে-ই তা provision, maintain, এবং scale করে, অন্যদিকে client হতে পারে এমন বিভিন্ন ধরনের device যেগুলোর ওপর আপনার কোনো নিয়ন্ত্রণ নেই।

এটি **peer-to-peer (P2P)** architecture থেকে ভিন্ন, যেখানে প্রতিটি participant অন্য সবার কাছে একই সাথে client এবং server উভয়ই — যেমন BitTorrent, যেখানে আপনি অন্য user-দের কাছ থেকে একটি file-এর অংশ download করেন এবং একইসাথে তাদের কাছে upload-ও করেন। এই course-এ আমরা যেসব system design করব তার বেশিরভাগই client-server model ব্যবহার করে, কারণ এটি নিয়ন্ত্রিতভাবে বোঝা, secure করা, এবং scale করা সহজ।

### ধাপ ১: DNS — ঠিকানা খুঁজে বের করা

আপনি যখন "google.com" টাইপ করেন, তখন আপনার computer আসলে জানে না সেটি কোথায় আছে। internet-এর computer-গুলো একে অপরকে খুঁজে পায় **IP address** নামক সাংখ্যিক ঠিকানা (যেমন 142.250.190.14) দিয়ে, মানুষের বোঝার মতো নাম দিয়ে নয়। তাই প্রথম ধাপ হলো "google.com"-কে একটি IP address-এ রূপান্তর করা — আর সেটাই করে **DNS**, অর্থাৎ Domain Name System।

DNS-কে internet-এর জন্য একটি phone book হিসেবে ভাবুন। আপনার computer একটি DNS resolver-কে জিজ্ঞাসা করে, "google.com-এর IP address কী?" সেই resolver প্রথমে একটি local cache পরীক্ষা করে — হতে পারে আপনার browser বা operating system সাম্প্রতিক কোনো visit থেকে উত্তরটি ইতিমধ্যে জানে। যদি না জানে, তাহলে এটি একটি DNS server-এর chain-কে জিজ্ঞাসা করে: প্রথমে একটি root server, তারপর ".com" domain-এর জন্য দায়ী একটি server, এবং শেষে "google.com"-এর জন্য authoritative server, যেটি প্রকৃত IP address ফেরত দেয়। এই পুরো lookup সাধারণত কয়েক মিলিসেকেন্ড সময় নেয় এবং একে দ্রুত রাখতে প্রতিটি layer-এ ব্যাপকভাবে cache করা হয়, কারণ প্রতিটি request-এই এই পুরো lookup করলে তা ভীষণ ধীর হয়ে যেত।

### ধাপ ২: TCP/IP দিয়ে সংযোগ স্থাপন

এখন আপনার browser-এর কাছে একটি IP address আছে, এবং এটিকে সেই machine-এর সাথে প্রকৃতপক্ষে একটি নির্ভরযোগ্য connection স্থাপন করতে হবে। এখানেই **TCP/IP** protocol suite কাজে আসে।

**IP (Internet Protocol)** addressing এবং routing পরিচালনা করে — এটি আপনার machine থেকে গন্তব্য machine পর্যন্ত পৃথক পৃথক data packet পৌঁছে দেওয়ার জন্য দায়ী, পথে অনেক intermediate router পার হয়ে। কিন্তু শুধু IP-এর নিজে থেকে packet-গুলো সঠিক ক্রমে পৌঁছাবে, বা আদৌ পৌঁছাবে কিনা, তার নিশ্চয়তা দেয় না।

এখানেই **TCP (Transmission Control Protocol)**-এর ভূমিকা। TCP, IP-এর ওপরে বসে reliability যোগ করে: এটি data-কে packet-এ ভাগ করে, সেগুলোর নম্বর দেয়, প্রতিটি packet পাওয়া গেছে কিনা তা নিশ্চিত করে (হারিয়ে যাওয়া কোনো packet পুনরায় পাঠায়), এবং অপর প্রান্তে সেগুলোকে সঠিক ক্রমে পুনরায় জোড়া লাগায়। কোনো data পাঠানোর আগে, TCP একটি **three-way handshake** সম্পন্ন করে — client একটি SYN (synchronize) packet পাঠায়, server একটি SYN-ACK দিয়ে সাড়া দেয়, এবং client একটি ACK দিয়ে নিশ্চিত করে। এই handshake সম্পন্ন হওয়ার পরই connection-কে "established" ধরা হয়, এবং প্রকৃত data প্রবাহিত হতে শুরু করে।

এই কারণেই TCP-কে "connection-oriented and reliable" বলা হয় — এটি সামান্য কিছু আগাম latency-এর (handshake) বিনিময়ে একটি শক্তিশালী নিশ্চয়তা দেয় যে আপনার data অক্ষত ও সঠিক ক্রমে পৌঁছাবে। এটি এমন যেকোনো কিছুর জন্য অত্যন্ত গুরুত্বপূর্ণ যেখানে গতির চেয়ে সঠিকতা বেশি গুরুত্বপূর্ণ, যেমন একটি webpage load করা বা একটি file transfer করা।

### ধাপ ৩: HTTP — একই ভাষায় কথা বলা

একটি TCP connection স্থাপিত হওয়ার পর, আপনার browser এখন webpage-এর জন্য একটি প্রকৃত request পাঠাতে পারে। এটি করা হয় **HTTP (HyperText Transfer Protocol)** ব্যবহার করে — একটি application-level protocol যা request এবং response-এর *format* নির্ধারণ করে, এবং আমরা মাত্র যে TCP connection তৈরি করলাম তার ওপরে চলে।

একটি HTTP request-এ থাকে একটি **method** (যেমন GET কিছু retrieve করার জন্য, অথবা POST কিছু submit করার জন্য), একটি **path** (যেমন `/search`), **header** (metadata, যেমন client কী ধরনের content type গ্রহণ করে, বা authentication token), এবং ঐচ্ছিকভাবে একটি **body** (পাঠানো data, POST request-এ সাধারণ)। server এই request process করে এবং একটি HTTP **response** ফেরত পাঠায়, যেখানে থাকে একটি **status code** (যেমন সফলতার জন্য 200, not found-এর জন্য 404, বা server error-এর জন্য 500), header, এবং একটি body — সাধারণত HTML, JSON, বা অন্য কোনো data।

আধুনিক web traffic সাধারণত **HTTPS** ব্যবহার করে, যা হলো encryption-এর জন্য **TLS (Transport Layer Security)**-এর ওপর স্তরীভূত HTTP — অর্থাৎ client ও server-এর মধ্যে পাঠানো সবকিছু এলোমেলো করে দেওয়া হয় যাতে আড়িপাতাকারীরা তা পড়তে না পারে, এবং উভয় পক্ষ যাচাই করতে পারে তারা কার সাথে কথা বলছে। আমরা HTTP, REST API, এবং status code নিয়ে আরও গভীরে যাব Module 2-তে।

### সবকিছু একত্রে

তাহলে শুরু থেকে শেষ পর্যন্ত পুরো ধারাবাহিকতা এরকম: আপনি একটি URL টাইপ করেন, আপনার browser (client) DNS-কে domain-টিকে IP address-এ resolve করতে বলে, আপনার machine সেই IP-তে থাকা server-এর সাথে একটি নির্ভরযোগ্য connection স্থাপনের জন্য একটি TCP three-way handshake সম্পন্ন করে, HTTPS হলে তার ওপর TLS encryption যোগ করে, এবং তারপর একটি HTTP request পাঠায়। server request process করে — হয়তো পেছনে নিজের database এবং cache-এর সাথে কথা বলে — এবং একটি HTTP response ফেরত পাঠায়, যেটি আপনার browser render করে যে page আপনি দেখেন তা তৈরি করে। এই সবকিছু, সম্ভবত হাজার হাজার মাইল physical cable এবং একাধিক router পেরিয়ে, সাধারণত এক সেকেন্ডেরও অনেক কম সময়ে সম্পন্ন হয়।

### বাস্তব-জগতের উদাহরণ

একটি news website load করলে কী ঘটে তা বিবেচনা করুন। DNS domain-টিকে একটি load balancer-এর পেছনে থাকা সম্ভাব্য অনেক server-এর একটিতে resolve করে (একটি concept যা আমরা Module 2-তে কভার করব)। TCP load balancer যে server বেছে নেয় তার সাথে একটি connection স্থাপন করে। একটি HTTPS request পাঠানো হয়, server article content আনে — হয়তো প্রতিবার database-এ না গিয়ে একটি cache থেকে — এবং HTML, CSS, এবং JavaScript ফেরত দেয়। এরপর আপনার browser সেই HTML-এ উল্লিখিত image, font, এবং script-এর জন্য আরও *অনেক* HTTP request পাঠায়, যার প্রতিটি সম্ভবত একই DNS-থেকে-TCP-থেকে-HTTP pipeline-এর মধ্য দিয়ে যায়, যদিও দক্ষতার জন্য সাধারণত ইতিমধ্যে-স্থাপিত connection পুনরায় ব্যবহার করা হয়।

### পুনরালোচনা (Recap)

চলুন পুনরালোচনা করি। Client-server architecture request-শুরুকারী (client) এবং request-প্রক্রিয়াকারী (server)-কে আলাদা করে। একটি webpage পাওয়ার জন্য তিনটি প্রধান ধাপ জড়িত: DNS একটি domain name-কে IP address-এ resolve করে; TCP/IP একটি three-way handshake-এর মাধ্যমে client ও server-এর মধ্যে একটি নির্ভরযোগ্য connection স্থাপন করে; এবং HTTP সেই connection-এর ওপর দিয়ে বিনিময় হওয়া প্রকৃত request ও response-এর গঠন নির্ধারণ করে। এই course-এ আপনি যে system-ই design করুন না কেন, তা এই stack-এর ওপরই বসে।

### এরপর কী

একক request কীভাবে client থেকে server পর্যন্ত যায় তা এখন যেহেতু আমরা বুঝেছি, স্বাভাবিক পরবর্তী প্রশ্ন হলো: যখন *একটিমাত্র* server আপনার সব traffic সামলাতে যথেষ্ট নয়, তখন কী হয়? পরবর্তী video-তে, আমরা একটি system scale করার দুটি মৌলিক কৌশল নিয়ে কথা বলব — vertical scaling এবং horizontal scaling — এবং কেন প্রায় প্রতিটি বড় system-কেই শেষমেশ horizontal পথ বেছে নিতে হয়। সেখানে দেখা হচ্ছে।

## মূল বিষয়সমূহ (Key Takeaways)

- Client-server architecture: client-রা request শুরু করে, server সেগুলো process করে ও সাড়া দেয়; এটি peer-to-peer-এর বিপরীত।
- DNS মানুষের বোধগম্য domain name-কে সাংখ্যিক IP address-এ রূপান্তর করে, অনেকটা একটি phone book-এর মতো।
- TCP/IP একটি three-way handshake (SYN, SYN-ACK, ACK)-এর মাধ্যমে machine-এর মধ্যে data-এর নির্ভরযোগ্য, ক্রমানুসারে delivery নিশ্চিত করে, যেখানে IP addressing/routing এবং TCP reliability সামলায়।
- HTTP একটি application-level protocol যা request (method, path, header, body) এবং response (status code, header, body)-এর format নির্ধারণ করে; HTTPS এর ওপর TLS encryption যোগ করে।
- সম্পূর্ণ request path (DNS → TCP handshake → HTTP request/response) এই course-এ আপনি যে system-ই design করবেন তার ভিত্তি তৈরি করে।
