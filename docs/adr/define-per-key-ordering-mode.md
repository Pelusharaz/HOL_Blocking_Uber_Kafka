# ADR-0003: Define an optional per-key ordering mode

| Field | Value |
|---|---|
| **Status** | Proposed (academic case study) |
| **Date** | 2026-10-27 |
| **Deciders** | [Pelu Jereremiah] (student architect) |
| **Depends on** | [ADR-0001](adopt-consumer-proxy.md), [ADR-0002](choose-at-least-once-delivery.md) |
| **Related SRS** | [define-consumer-proxy-requirements.md](../srs/define-consumer-proxy-requirements.md): FR-04, FR-10, NFR-C1 |

> **Honesty note.** The per-key ordering mode is **my own extension [Proposal]**. It is **not** claimed to be
> implemented by Uber, and every number or default below is an assumption to be validated by measurement.

## Context

- **[Fact]** Kafka guarantees order only within a partition. With the default partitioner, messages that share
  a key go to the same partition, so they are stored in order (as long as the partition count does not change).
- **Effect of ADR-0001.** The proxy processes messages out of order to avoid head-of-line blocking, so the
  partition-wide ordering that consumers used to get from sequential consumption is no longer provided.
- **Need [Assumption, illustrative].** Some consumers need order only among related events, for example a
  `TripCompleted` event must not be applied before `TripStarted` for the *same* trip. They do not need order
  between different trips (assumption A-01).
- **Forces.** Preserve order where it matters; do not bring back partition-wide blocking; keep memory use
  bounded; keep the delivery contract simple for service teams.

## Decision

Each consumer group will have an `ordering` setting with two values:

| Mode | Behaviour |
|---|---|
| `NONE` (**default**) | Messages are dispatched concurrently with no ordering guarantee (ADR-0001). |
| `PER_KEY` | At most **one message per Kafka record key** is in flight at a time. Later messages with the same key wait in a per-key queue in offset order. Messages with different keys are processed in parallel. |

Detailed rules **[Proposal]**:

1. **Key.** The Kafka record key is the ordering key. Messages without a key are treated as `NONE`, and a warning metric is emitted.
2. **Failure inside a key.** In `PER_KEY` mode, retries for a failing message are done **in place** by the proxy
   (with backoff) while the key stays blocked, rather than through a retry topic, because a retry topic would
   reorder messages (see [ADR-0004](choose-retry-and-dead-letter-policy.md)).
3. **Bounded blocking.** After the maximum attempts, the message goes to the dead-letter topic and the key is released.
   The next message for that key is marked with a header `ordering-gap=true` so the consumer knows an earlier message was skipped.
4. **Backpressure.** Limits apply to waiting messages per key and in total. When a limit is reached, the proxy pauses
   fetching from the affected partition instead of growing memory without bound.
5. **Rebalance.** After a rebalance or crash, in-flight state is rebuilt by redelivering from the committed offset in offset order, which preserves per-key order (duplicates handled as in ADR-0002).

## Consequences

**Positive**
- Teams that need ordering get it for the data that needs it, without partition-wide blocking.
- A slow message blocks only its own key, so the "blast radius" of head-of-line blocking shrinks from a whole partition to a single key.
- The default stays `NONE`, so most consumers keep full parallelism.

**Negative**
- **Hot keys.** A key with a very high message rate is processed serially and can become a bottleneck.
- The proxy now holds per-key state in memory, which raises complexity and memory cost.
- Blocking at key level still exists, so a permanently failing message delays its key until attempts are exhausted.
- Ordering is not preserved across a dead-lettered message (`ordering-gap` is only a signal).

**Risks**
- Memory pressure or pausing of partitions under skewed key distributions.
- Consumers may assume `PER_KEY` gives global order; documentation must say it does not.

**Follow-up**
- Refine requirement FR-10 with acceptance tests: zero ordering violations per key; throughput of other keys unaffected by one blocked key.
- Measure the real key skew of the target topics before choosing queue limits.

**Review.** One month after any trial, compare ordering violations, hot-key delay and memory use with the SRS evaluation plan (section 5).

## Alternatives considered

| Option | Why not chosen |
|---|---|
| No ordering support; consumers use sequence numbers or version checks | Valid and simple for the proxy, but shifts complexity to every team; still offered as the recommended approach when order matters rarely. |
| Keep partition-level ordering (the current model) | Reintroduces head-of-line blocking, which is the problem being solved. |
| Single partition for global order | Removes all parallelism; unacceptable at scale. |

## Revision history

| Date | Change | Author |
|---|---|---|
| 2026-10-27 | Created | [Pelu Jeremiah] |
