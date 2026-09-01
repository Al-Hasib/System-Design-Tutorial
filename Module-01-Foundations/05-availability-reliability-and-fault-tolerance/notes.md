# Notes: Availability, Reliability, Redundancy & Fault Tolerance

## Core Definitions

| Term | Definition |
|---|---|
| **Availability** | % of time a system is up and able to respond to requests |
| **Reliability** | System performs its intended function correctly and consistently over time |
| **Redundancy** | Having backup/duplicate components so failure of one doesn't fail the system |
| **Fault Tolerance** | System continues operating correctly automatically despite component failures |
| **Single Point of Failure (SPOF)** | Any one component whose failure takes down the entire system |

Availability answers "is it up?" Reliability answers "does it work correctly when it's up?"

## The Nines Table

| Availability | Nickname | Downtime per Year | Downtime per Day |
|---|---|---|---|
| 99% | Two nines | ~3.65 days | ~14.4 minutes |
| 99.9% | Three nines | ~8.76 hours | ~1.44 minutes |
| 99.99% | Four nines | ~52.6 minutes | ~8.6 seconds |
| 99.999% | Five nines | ~5.26 minutes | ~0.86 seconds |

**Rule of thumb:** each additional nine ≈ 10x less downtime, requiring proportionally more sophisticated redundancy/infrastructure.

## Fault Tolerance Techniques

| Technique | What It Does |
|---|---|
| Replication | Multiple copies of data/services so losing one copy doesn't lose data (Module 3) |
| Failover | Automatically redirects traffic from failed component to healthy backup |
| Health Checks | Periodic automated probes that detect unhealthy instances, trigger failover |
| Graceful Degradation | Non-critical component failure reduces functionality instead of causing total failure |

## Relationship to Scalability (Video 4)

- Horizontal scaling (multiple servers) provides redundancy as a natural side effect.
- A single, even very powerful, vertically-scaled server is always a SPOF.

## Quick Revision Bullets

- Availability ≠ Reliability: available but wrong answers = unreliable.
- Memorize the nines table — interviewers often ask you to translate a percentage into downtime.
- SPOF = single point of failure; always ask "what if this one thing dies?"
- Fault tolerance = redundancy + automatic detection (health checks) + automatic recovery (failover).
- Horizontal scaling and fault tolerance reinforce each other.
