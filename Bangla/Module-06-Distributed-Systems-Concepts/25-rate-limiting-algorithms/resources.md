# আরও পড়াশোনা ও রেফারেন্স (Further Reading & References)

## অফিসিয়াল ডকুমেন্টেশন (Official Docs)

- [Stripe API Rate Limits](https://docs.stripe.com/rate-limits) — Stripe কীভাবে প্রতি-account request limit প্রয়োগ করে এবং `429`/`Retry-After` response contract।
- [Amazon API Gateway: Throttle API Requests](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html) — steady-state rate এবং burst capacity কনফিগারেশন (token bucket মডেল সরাসরি উন্মুক্ত)।
- [Nginx: Rate Limiting with ngx_http_limit_req_module](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html) — leaky-bucket-ধরনের request rate limiting এবং `burst` প্যারামিটার।
- [Cloudflare: Rate Limiting Rules](https://developers.cloudflare.com/waf/rate-limiting-rules/) — edge-level distributed rate limiting ধারণা এবং কনফিগারেশন।

## গবেষণাপত্র (Papers)

- এই অ্যালগরিদমগুলো সংজ্ঞায়িত করার মতো কোনো একক প্রামাণ্য একাডেমিক গবেষণাপত্র নেই যা সাধারণত শেখানো হয়; এগুলোর উৎপত্তি network traffic-shaping মান থেকে (নিচের Wikipedia নিবন্ধগুলোতে উল্লিখিত ITU-T এবং ATM Forum leaky bucket নির্দিষ্টকরণ দেখুন) এবং API rate limiting অনুশীলনে অভিযোজিত হয়েছে।

## আরও পড়াশোনা (Further Reading)

- [Wikipedia: Token Bucket](https://en.wikipedia.org/wiki/Token_bucket) — token bucket traffic-shaping অ্যালগরিদমের পটভূমি।
- [Wikipedia: Leaky Bucket](https://en.wikipedia.org/wiki/Leaky_bucket) — leaky bucket traffic-shaping অ্যালগরিদম এবং token bucket-এর সাথে এর সম্পর্কের পটভূমি।
