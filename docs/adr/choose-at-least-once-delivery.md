# ADR-0002: Choose at-least-once delivery with idempotent consumers

| Field | Value |
|---|---|
| **Status** | Proposed (academic case study) |
| **Date** | 2026-10-26 |
| **Deciders** | [Pelu Jeremiah] (student architect) |
| **Depends on** | [ADR-0001: Adopt a consumer proxy](adopt-consumer-proxy.md) |
| **Related SRS** | [define-consumer-proxy-requirements.md](../srs/define-consumer-proxy-requirements.md): FR-03, FR-05, FR-06, FR-14, NFR-R1, NFR-C1 |

> **Honesty note.** This ADR records a design choice for our proposed consumer proxy. It does **not** claim
> that Uber uses this exact delivery contract; Uber's internal choice was not verified for this study.

## Context

- **[Fact]** In Kafka, when a consumer commits its offset determines the delivery guarantee. Committing
  *after* processing gives at-least-once (a crash before the commit causes redelivery); committing *before*
  processing gives at-most-once (a crash can lose messages). Kafka's exactly-once features (idempotent
  producer, transactions) cover reading from and writing to Kafka, not side effects in external systems
  such as databases or third-party APIs. See the Kafka documentation on delivery semantics.
- **Our design (ADR-0001).** The proxy processes messages out of order and commits only the highest
  contiguous acknowledged offset (FR-05). Messages acknowledged after a gap are therefore not yet committed,
  and will be redelivered if the proxy crashes or a partition is reassigned (FR-14).
- **Other sources of redelivery.** Service timeouts, nacks and retries (FR-06) deliver the same message
  more than once by design.
- **Forces.** No message may be lost; the proxy must stay simple enough to operate; services call external
  systems where a distributed exactly-once guarantee is not available; many teams will write consumers in different languages.

## Decision

We will provide **at-least-once delivery** and require consumers to be **idempotent**.

1. The proxy commits offsets only after messages are acknowledged or dead-lettered (as in FR-05). It never commits before processing.
2. Every delivery carries a **stable message ID** that stays the same across redeliveries and retries.
   **[Proposal]** The ID is built from topic, partition and original offset; copies in retry or dead-letter topics keep the original ID in a header.
3. Consumers must tolerate duplicates, using one or more of:
   - naturally idempotent operations (for example, upsert or "set to value" rather than "add one");
   - a deduplication store keyed by message ID, kept at least as long as the maximum redelivery window **[Assumption]**;
   - idempotency keys on calls to downstream APIs.
4. The proxy will **not** deduplicate on behalf of services. Deduplication needs state and knowledge of each service's side effects.
5. Exactly-once delivery is **out of scope**.

## Consequences

**Positive**
- No acknowledged or unacknowledged message is lost (supports NFR-R1).
- No distributed transaction between Kafka and external systems.
- The proxy stays stateless with respect to business logic, which keeps it easier to scale and replace.

**Negative**
- Duplicates will happen, so every consumer team carries the idempotency burden.
- Deduplication stores add cost and operational work, and need a retention policy.
- Testing is harder: teams must test duplicate and out-of-order delivery.

**Risks**
- Services with non-idempotent side effects (for example, sending a payment or an email) could act twice. Mitigation: require idempotency keys for such calls and review them before onboarding.
- An under-sized deduplication window lets late duplicates through.

**Follow-up**
- Write idempotency guidance for service teams (SRS open issue OI-04).
- ADR-0004 (retry and dead-letter policy) must state the maximum redelivery window that sizes the deduplication window.

**Review.** One month after any trial, count duplicate deliveries and duplicated side effects, and compare with the SRS evaluation plan (section 5).

## Alternatives considered

| Option | Why not chosen |
|---|---|
| At-most-once (commit before processing) | Messages can be lost on a crash; unacceptable for events such as trips and payments. |
| Exactly-once via Kafka transactions | Applies to Kafka-to-Kafka read-process-write; does not cover external side effects, and adds complexity. |
| Proxy-side deduplication | Requires shared state, becomes another critical component, and cannot know which side effects are safe to repeat. |

## Revision history

| Date | Change | Author |
|---|---|---|
| 2026-10-25 | Created | [pelu Jeremiah] |
