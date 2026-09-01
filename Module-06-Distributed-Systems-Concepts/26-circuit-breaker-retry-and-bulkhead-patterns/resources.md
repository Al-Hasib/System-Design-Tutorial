# Further Reading & References

## Official Docs

- [Resilience4j Documentation](https://resilience4j.readme.io/docs) — official docs for the lightweight fault-tolerance library offering Circuit Breaker, Retry, Bulkhead, and Rate Limiter decorators for the JVM.
- [Netflix Hystrix Wiki (GitHub)](https://github.com/Netflix/Hystrix/wiki) — Netflix's original circuit breaker library that popularized the pattern in microservices; the project is now in maintenance mode, with resilience4j widely recommended as its successor.
- [Microsoft Azure Architecture Center — Circuit Breaker Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker) — canonical pattern reference covering states, thresholds, and implementation considerations.
- [Microsoft Azure Architecture Center — Bulkhead Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead) — canonical pattern reference for isolating resources into independent pools.
- [Istio — Circuit Breaking](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/) — how to configure outlier detection and connection pool limits for circuit breaking at the service mesh layer.

## Papers

- [Google SRE Book — Chapter 22: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/) — foundational treatment of how cascading failures happen and how to design defenses against them.

## Further Reading

- [AWS Builders' Library — Timeouts, Retries, and Backoff with Jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — the well-known deep dive on why naive exponential backoff still causes thundering herds, and how jitter fixes it.
