# ADR-0004: Choose a retry and dead-letter topic policy

| Field | Value |
|---|---|
| **Status** | Proposed (academic case study) |
| **Date** | 2026-10-27 |
| **Deciders** | [Pelu Jeremiah] (student architect) |
| **Depends on** | [ADR-0001](adopt-consumer-proxy.md), [ADR-0002](choose-at-least-once-delivery.md), [ADR-0003](define-per-key-ordering-mode.md) |
| **Related SRS** | [define-consumer-proxy-requirements.md](../srs/define-consumer-proxy-requirements.md): FR-06, FR-07, FR-08, FR-09, NFR-P3 |

> **Honesty note.** All timeouts, delays, attempt counts and retention periods below are **[Assumption]**
> starting values chosen for the study, to be tuned by measurement. They are not Uber's settings.
> Uber Engineering has also published an article on reprocessing and dead-letter queues with Kafka; read it and
> compare before citing it as supporting evidence.

## Context

- **Need.** When a service fails or is slow on a message, the proxy must not stall the commit path (ADR-0001), must not lose the message (ADR-0002), and must not retry forever.
- **Link to FR-08/FR-09.** A message that blocks the commit path for too long must be moved out of the way.
- **Link to ADR-0002.** Deduplication windows in consumers must be sized from the **maximum automatic redelivery window**, which this ADR defines.
- **Forces.** Bounded delay; protection of failing downstream services from retry storms; enough information for engineers to diagnose and replay failures; compatibility with per-key ordering (ADR-0003).

## Decision

**1. Classify failures.** The service's reply is one of: `ACK`, `NACK_RETRYABLE`, `NACK_PERMANENT`. A processing timeout counts as `NACK_RETRYABLE`. `NACK_PERMANENT` goes straight to the dead-letter topic.

**2. Retry with delay tiers (mode `NONE`).** Failed messages are published to retry topics with increasing delay:

| Tier | Topic | Delay before redelivery (default) |
|---|---|---|
| 1 | `<topic>.retry.1` | 10 s |
| 2 | `<topic>.retry.2` | 1 min |
| 3 | `<topic>.retry.3` | 10 min |

- Maximum attempts: **4 in total** (the original plus three retries). Configurable.
- Per-message processing timeout: **30 s**. Configurable.
- Delays use **jitter** (random variation of ±20 %) to avoid synchronized retry bursts.
- Once a message is published to a retry topic, it is considered handled for offset purposes, so the commit path moves on (this is what removes the blocking).

**3. Retry in place (mode `PER_KEY`).** To keep order (ADR-0003), the proxy retries in place with the same delays while the key stays blocked, then dead-letters after the last attempt.

**4. Dead-letter topic.** After the last attempt or on `NACK_PERMANENT`, the message goes to `<topic>.dlq` with headers: stable message ID, original topic, partition and offset, attempt count, last error, first-failure and last-failure timestamps. Default retention: **14 days**.

**5. Head-of-line blocking mitigation.** When FR-08 detects a blocked message, the proxy moves it to the next retry tier (or to the DLQ if attempts are exhausted), which counts as an attempt.

**6. Replay.** An authorised engineer can replay DLQ messages through the admin API. Replayed messages keep their original message ID and carry `replay=true`.

**7. Maximum automatic redelivery window.** With the defaults, the window is about

`4 × 30 s (timeouts) + 10 s + 60 s + 600 s (delays) ≈ 790 s ≈ 13 minutes`

plus jitter. Consumers using a deduplication store should keep IDs for **at least this window plus a safety margin**. Replays are deliberate human actions, so a consumer whose dedupe window is shorter than the DLQ retention must treat `replay=true` messages as possibly already applied.

**8. Alerting.** The proxy exposes retry count, DLQ count and HOL events (FR-13). Alerts fire when the DLQ rate exceeds a configurable threshold.

## Consequences

**Positive**
- A poison or slow message is removed from the commit path within a bounded time (supports NFR-P3).
- Backoff and jitter protect struggling downstream services.
- Every failed message is diagnosable and replayable.
- The redelivery window is explicit, so idempotency guidance (ADR-0002, OI-04) has a concrete number to work from.

**Negative**
- More topics to operate per source topic (three retry topics and one DLQ).
- Retry tiers add latency for messages that fail temporarily.
- Retried messages are processed out of order relative to later messages in mode `NONE`.
- The DLQ needs ownership; unattended DLQs hide data problems.

**Risks**
- **Retry amplification.** Many failures at once can multiply load. Mitigation: backoff, jitter, per-group rate limits (FR-11).
- **Silent data loss in practice.** If nobody watches the DLQ, messages expire after 14 days. Mitigation: alerting and an owner per consumer group.
- Wrong failure classification (retrying a permanent error wastes capacity).

**Follow-up**
- Tune delays, attempts and retention by measuring real failure patterns.
- Write DLQ runbook and ownership rules.
- Update the idempotency guidance (SRS OI-04) with the 13-minute figure once the defaults are final.

**Review.** One month after any trial, review retry rates, DLQ contents and replay outcomes against the SRS evaluation plan (section 5).

## Alternatives considered

| Option | Why not chosen |
|---|---|
| Retry forever in place | Brings back head-of-line blocking. |
| One retry topic with a fixed delay | Simpler, but no backoff and a higher risk of retry storms. |
| Drop messages after failure (no DLQ) | Loses data; conflicts with ADR-0002. |
| Delegate all retry logic to each service | Duplicated effort and inconsistent behaviour across teams (the problem ADR-0001 aims to remove). |

## Revision history

| Date | Change | Author |
|---|---|---|
| 2026-10-27 | Created | [Pelu Jeremiah] |
