# ADR-0001: Adopt a consumer proxy between Kafka and consumer services

| Field | Value |
|---|---|
| **Status** | Proposed (academic case study) |
| **Date** | 2026-10-25 |
| **Deciders** | [Pelu Jeremiah] (student architect) |
| **Related SRS** | [define-consumer-proxy-requirements.md](../srs/define-consumer-proxy-requirements.md): FR-01 to FR-14, NFR-P1 to NFR-SEC1 |

> **Honesty note.** Uber publicly reports building a push-based consumer proxy for Kafka. This ADR
> reconstructs the architectural reasoning behind that kind of decision for study purposes. It does
> **not** claim to be Uber's internal ADR, and it does not claim access to Uber's internal numbers.

## Context

- **[Fact]** In the standard Kafka consumer-group model, each partition is read by at most one consumer
  in a group, and progress is tracked by offsets. Within a partition, messages are processed and committed in order.
- **[Fact]** Uber's engineering blog says it hit partition scalability, head-of-line blocking and
  multi-language support problems, and that its push-based consumer proxy became the primary way to read
  Kafka in pub-sub use cases, with over 1,000 consumer services onboarded.
- **Problem (our analysis).** One slow or repeatedly failing message blocks every message behind it in the
  same partition (head-of-line blocking). Parallelism is capped by the partition count, and every service
  embeds its own Kafka client and retry logic.
- **Root cause.** The partition is both the unit of ordering and the unit of parallelism, offsets commit
  sequentially, and consumption logic lives in every service.
- **Forces.** Keep ordering where it matters; avoid message loss; avoid forcing every team to repartition;
  support several programming languages; limit operational risk.

## Decision

We will introduce a **consumer proxy layer** between Kafka and consumer services. The proxy will:

1. read from Kafka on behalf of services and push messages to them over gRPC;
2. require a per-message acknowledgment and track offsets independently of processing order,
   committing only the highest offset below which every message is acknowledged or dead-lettered;
3. detect head-of-line blocking and move blocked messages to a retry or dead-letter topic;
4. offer an **optional per-key ordering mode** **[Proposal, our extension; not claimed for Uber]**.

## Consequences

**Positive**
- Throughput can scale beyond the partition count.
- A single bad message no longer stalls a whole partition.
- Services drop Kafka client code and can be written in any language.
- Retry and dead-letter policy is centralised.

**Negative**
- An extra network hop adds latency **[Assumption]**.
- Out-of-order processing weakens default ordering; consumers must tolerate it or use per-key mode.
- Delivery is at-least-once, so consumers must be idempotent.
- Higher implementation complexity (offset tracking, rebalancing).

**Risks**
- The proxy becomes a critical component and a potential single point of failure; it must be replicated.
- Misconfiguration could affect many consumer services at once.

**Follow-up ADRs (to write)**
- [ADR-0002](choose-at-least-once-delivery.md): Choose at-least-once delivery with idempotent consumers.
- [ADR-0003](define-per-key-ordering-mode.md): Define the per-key ordering mode.
- [ADR-0004](choose-retry-and-dead-letter-policy.md): Choose retry and dead-letter topic policy.

**Review.** Compare the real behaviour against the evaluation criteria in the SRS (section 5) one month after any trial.

## Alternatives considered

| Option | Why not chosen (or what to verify) |
|---|---|
| Add more partitions | More broker and rebalancing overhead; cannot fix one blocked message inside a partition. |
| Thread pool inside each service | Pushes complex offset tracking into every team and language; does not centralise retry or dead-letter handling. |
| Kafka-native queue-style consumption (share groups, KIP-932) | Promising for per-message acknowledgment. **[Verify its current maturity in the Kafka version in use before relying on this.]** |

## Revision history

| Date | Change | Author |
|---|---|---|
| 2026-10-25 | Created | [Pelu Jeremiah] |
| 2026-10-27 | Follow-up ADR links added (ADR-0003, ADR-0004) | [Pelu Jeremiah] |
