# অনুশীলনী ও Interview প্রশ্ন

1. **এক বা দুই বাক্যে client-server model বর্ণনা করুন।**
   Client-রা server-এর কাছে request শুরু করে, এবং server সেই request-গুলো process করে ও response ফেরত পাঠায়। ভূমিকাগুলো অসম (asymmetric) — client-রা সাধারণত অন্য client-দের request সার্ভ করে না।

2. **Client-server architecture peer-to-peer থেকে কীভাবে আলাদা?**
   Client-server-এ ভূমিকা নির্দিষ্ট এবং অসম (client request করে, server সাড়া দেয়)। Peer-to-peer-এ প্রতিটি node অন্য node-দের কাছে client এবং server উভয় হিসেবেই কাজ করতে পারে, যেমনটা BitTorrent-এর মতো system-এ দেখা যায়।

3. **DNS কোন সমস্যা সমাধান করে, এবং কেন এটি প্রয়োজনীয়?**
   Computer-রা traffic route করে সাংখ্যিক IP address ব্যবহার করে, কিন্তু মানুষ "google.com"-এর মতো মনে রাখার সহজ নাম পছন্দ করে। DNS domain name-কে IP address-এ রূপান্তর করে যাতে user-দের সংখ্যা মনে রাখতে না হয়।

4. **উচ্চ পর্যায়ে DNS resolution প্রক্রিয়াটি বর্ণনা করুন।**
   Browser/OS প্রথমে তার local cache পরীক্ষা করে; না পাওয়া গেলে, এটি একটি DNS resolver-কে query করে, যেটি পরিবর্তে একটি root server, তারপর একটি TLD server (যেমন, ".com"-এর জন্য), তারপর নির্দিষ্ট domain-এর জন্য authoritative name server-কে query করতে পারে, যেটি IP address ফেরত দেয়।

5. **TCP three-way handshake ব্যাখ্যা করুন।**
   Client একটি connection শুরু করার জন্য একটি SYN packet পাঠায়; server acknowledge করে এবং connect করতে সম্মত হয়ে একটি SYN-ACK দিয়ে সাড়া দেয়; client একটি চূড়ান্ত ACK পাঠায়। এই বিনিময়ের পর, connection স্থাপিত হয় এবং data transmit করা যায়।

6. **IP এবং TCP-এর দায়িত্বের মধ্যে পার্থক্য কী?**
   IP addressing এবং routing সামলায় — network জুড়ে packet-কে উৎস থেকে গন্তব্যে পৌঁছে দেওয়া। TCP, IP-এর ওপর reliability যোগ করে: packet-কে সঠিকভাবে ক্রমবদ্ধ করা, loss শনাক্ত করা, এবং হারিয়ে যাওয়া packet পুনরায় পাঠানো।

7. **একটি HTTP request সাধারণত কী কী নিয়ে গঠিত হয়?**
   একটি method (যেমন, GET, POST), একটি path (যেমন, `/users/123`), header (content type বা auth token-এর মতো metadata), এবং ঐচ্ছিকভাবে একটি body (submit করা data)।

8. **HTTP এবং HTTPS-এর মধ্যে পার্থক্য কী?**
   HTTPS হলো TLS (Transport Layer Security)-এর ওপর স্তরীভূত HTTP, যা transit-এ থাকা data encrypt করে এবং client-কে certificate-এর মাধ্যমে server-এর identity যাচাই করতে দেয় — শুধু HTTP data plaintext-এ পাঠায়।

9. **পরিস্থিতি: একজন user রিপোর্ট করলেন একটি website "প্রথমবার লোড হতে ধীর কিন্তু reload করলে দ্রুত।" কোন networking concept সম্ভবত এটি ব্যাখ্যা করে?**
   DNS caching এবং/অথবা TCP connection reuse (keep-alive) — প্রথম visit-এ একটি সম্পূর্ণ DNS lookup এবং TCP/TLS handshake-এর খরচ দিতে হয়, যেখানে পরবর্তী visit-গুলো cache করা DNS ফলাফল এবং/অথবা ইতিমধ্যে-খোলা একটি connection পুনরায় ব্যবহার করতে পারে।

10. **কেন TCP-কে "connection-oriented" বলা হয়, এবং এই design-এর trade-off কী?**
    TCP data পাঠানোর আগে একটি connection স্থাপন (three-way handshake-এর মাধ্যমে) করা প্রয়োজন, এবং ক্রমানুসারে, নির্ভরযোগ্য delivery নিশ্চিত করতে connection সম্পর্কে state বজায় রাখে। Trade-off হলো reliability guarantee-র বিনিময়ে আগাম যোগ হওয়া latency (handshake)।

11. **পরিস্থিতি: আপনি এমন একটি system design করছেন যেখানে মাঝেমধ্যে data loss গ্রহণযোগ্য কিন্তু গতি গুরুত্বপূর্ণ (যেমন, live video streaming)। TCP-এর reliability guarantee কি সবসময় সবচেয়ে উপযুক্ত হবে? এটি কীসের ইঙ্গিত দিতে পারে?**
   সবসময় নয় — এটি UDP-এর মতো protocol-এর দিকে একটি ইঙ্গিত, যা latency কমানোর বিনিময়ে reliability ছেড়ে দেয়, কারণ পুরনো video frame পুনরায় পাঠানোর চেয়ে পরবর্তী frame-এ এগিয়ে যাওয়া প্রায়ই বেশি কার্যকর। (UDP নিজেই এই beginner video-র আওতার বাইরে, তবে ভবিষ্যতের একটি বিবেচনা হিসেবে উল্লেখ করার মতো।)

12. **কোন HTTP status code range সাধারণত client error নির্দেশ করে, এবং কোন range server error নির্দেশ করে?**
    4xx status code (যেমন, 404 Not Found) client error নির্দেশ করে; 5xx status code (যেমন, 500 Internal Server Error) server error নির্দেশ করে।
