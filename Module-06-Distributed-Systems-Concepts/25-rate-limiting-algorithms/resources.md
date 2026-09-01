# Further Reading & References

## Official Docs

- [Stripe API Rate Limits](https://docs.stripe.com/rate-limits) — how Stripe enforces per-account request limits and the `429`/`Retry-After` response contract.
- [Amazon API Gateway: Throttle API Requests](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html) — steady-state rate and burst capacity configuration (token bucket model exposed directly).
- [Nginx: Rate Limiting with ngx_http_limit_req_module](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html) — leaky-bucket-style request rate limiting and the `burst` parameter.
- [Cloudflare: Rate Limiting Rules](https://developers.cloudflare.com/waf/rate-limiting-rules/) — edge-level distributed rate limiting concepts and configuration.

## Papers

- No single canonical academic paper defines these algorithms as commonly taught; they originate from network traffic-shaping standards (see ITU-T and ATM Forum leaky bucket specifications referenced in the Wikipedia articles below) and have been adapted into API rate limiting practice.

## Further Reading

- [Wikipedia: Token Bucket](https://en.wikipedia.org/wiki/Token_bucket) — background on the token bucket traffic-shaping algorithm.
- [Wikipedia: Leaky Bucket](https://en.wikipedia.org/wiki/Leaky_bucket) — background on the leaky bucket traffic-shaping algorithm and its relation to token bucket.
