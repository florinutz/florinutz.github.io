---
layout: default
title: Message Deduplication - A Complete Guide
---

# Message Deduplication: A Complete Guide

## Table of Contents

- [1. Why Duplicates Exist](#1-why-duplicates-exist)
  - [1.1 The Fundamental Problem: At-Least-Once Delivery](#11-the-fundamental-problem-at-least-once-delivery)
  - [1.2 Where Duplicates Are Born](#12-where-duplicates-are-born)
  - [1.3 Why Not Just Prevent Them?](#13-why-not-just-prevent-them)
- [2. What Deduplication Actually Is](#2-what-deduplication-actually-is)
  - [2.1 The Core Abstraction](#21-the-core-abstraction)
  - [2.2 Identity: What Makes a Message "The Same"?](#22-identity-what-makes-a-message-the-same)
  - [2.3 The Two Fundamental Modes](#23-the-two-fundamental-modes)
- [3. Data Structures for Dedup State](#3-data-structures-for-dedup-state)
  - [3.1 Hash Map (The Obvious Choice)](#31-hash-map-the-obvious-choice)
  - [3.2 Bloom Filters (Probabilistic)](#32-bloom-filters-probabilistic)
  - [3.3 Cuckoo Filters](#33-cuckoo-filters)
  - [3.4 Key-Value Stores with TTL](#34-key-value-stores-with-ttl)
  - [3.5 Sorted Sets / Skip Lists](#35-sorted-sets--skip-lists)
  - [3.6 Comparison Matrix](#36-comparison-matrix)
- [4. Time-Windowed Deduplication](#4-time-windowed-deduplication)
  - [4.1 Why Not Just Dedup Forever?](#41-why-not-just-dedup-forever)
  - [4.2 Expiry Strategies](#42-expiry-strategies)
  - [4.3 Wall Clock vs Event Time](#43-wall-clock-vs-event-time)
  - [4.4 The Edge at Window Boundaries](#44-the-edge-at-window-boundaries)
- [5. The Trade-Off Space](#5-the-trade-off-space)
  - [5.1 Memory vs Accuracy](#51-memory-vs-accuracy)
  - [5.2 Latency vs Throughput](#52-latency-vs-throughput)
  - [5.3 Exactness vs Efficiency](#53-exactness-vs-efficiency)
  - [5.4 Simplicity vs Durability](#54-simplicity-vs-durability)
  - [5.5 The CAP Angle (Distributed Systems)](#55-the-cap-angle-distributed-systems)
- [6. Dedup in Stream Processing (The Production Context)](#6-dedup-in-stream-processing-the-production-context)
  - [6.1 Dedup as a Pipeline Stage](#61-dedup-as-a-pipeline-stage)
  - [6.2 The Two-Phase Pattern](#62-the-two-phase-pattern)
  - [6.3 Batch vs Per-Message Dedup](#63-batch-vs-per-message-dedup)
  - [6.4 What Happens When Dedup Fails?](#64-what-happens-when-dedup-fails)
- [7. How GlassFlow Does It](#7-how-glassflow-does-it)
  - [7.1 Architecture](#71-architecture)
  - [7.2 The Badger KV Approach](#72-the-badger-kv-approach)
  - [7.3 Processor Chain Pattern](#73-processor-chain-pattern)
  - [7.4 Design Decisions Worth Noting](#74-design-decisions-worth-noting)
- [8. Edge Cases and Gotchas](#8-edge-cases-and-gotchas)
- [9. The Assignment Through This Lens](#9-the-assignment-through-this-lens)
- [10. Further Reading](#10-further-reading)

---

## 1. Why Duplicates Exist

### 1.1 The Fundamental Problem: At-Least-Once Delivery

In any system where messages travel between components, there are three delivery guarantees you can aim for:

| Guarantee | Meaning | Trade-off |
|-----------|---------|-----------|
| **At-most-once** | Fire and forget. Message may be lost. | Fast, simple. Data loss acceptable. |
| **At-least-once** | Retry until acknowledged. Message may arrive more than once. | No data loss, but duplicates. |
| **Exactly-once** | Each message processed exactly one time. | The holy grail. Extremely hard/expensive. |

**Exactly-once delivery is essentially impossible** in distributed systems without cooperation between producer, broker, and consumer. What people call "exactly-once" is almost always "at-least-once delivery + deduplication" — which gives you **exactly-once _processing_** semantics.

This is the reason deduplication exists: **it's the mechanism that bridges at-least-once delivery to exactly-once processing.**

### 1.2 Where Duplicates Are Born

Duplicates aren't bugs — they're a natural consequence of reliable systems:

**Producer retries:** A producer sends a message, the network drops the ACK (not the message). The producer assumes failure and retries. The broker now has two copies.

```
Producer ──msg──> Broker  (received, stored)
Producer <──X──── Broker  (ACK lost in network)
Producer ──msg──> Broker  (retry — now stored twice)
```

**Consumer reprocessing:** A consumer processes a message, crashes before committing its offset/ACK. On restart, it re-reads the same message.

```
Consumer reads msg → processes → [CRASH before ACK]
Consumer restarts  → reads same msg again → processes again
```

**Partition rebalancing:** In Kafka-style systems, when consumers join/leave a group, partitions get reassigned. Offsets may roll back slightly, causing reprocessing of recent messages.

**Network partitions and split-brain:** Two instances of a producer both think they're the leader. Both produce the same logical message.

**Application-level retries:** An HTTP API times out. The caller retries. The first request actually succeeded — you've now triggered the same action twice.

### 1.3 Why Not Just Prevent Them?

You could try:

- **Kafka idempotent producers** — eliminate producer-side duplicates within a session. Doesn't help with consumer-side duplicates.
- **Exactly-once transactions (Kafka EOS)** — works within Kafka-to-Kafka pipelines. Falls apart the moment you involve an external system (a database, an API call).
- **TCP-style sequence numbers** — requires tight coupling between producer and consumer. Doesn't scale to multi-producer, multi-consumer topologies.

The reality: **prevention is partial at best.** Every production system that cares about correctness needs deduplication as a safety net.

---

## 2. What Deduplication Actually Is

### 2.1 The Core Abstraction

At its simplest, dedup is a stateful filter:

```
for each message:
    key = extract_identity(message)
    if key in seen_set:
        drop(message)       # duplicate
    else:
        seen_set.add(key)
        emit(message)       # first occurrence
```

Three questions define every dedup implementation:
1. **What is the identity key?** (how do you know two messages are "the same")
2. **How do you store the seen set?** (data structure / persistence)
3. **How long do you remember?** (forever vs time-bounded)

### 2.2 Identity: What Makes a Message "The Same"?

This is more subtle than it looks.

**Option A: Natural key (a field in the payload)**
- E.g., `message.id`, `order.order_id`, `event.transaction_ref`
- Simple, meaningful, business-aligned
- Requires trusting the producer to populate it correctly
- What the assignment asks for

**Option B: Content hash**
- Hash the entire message body (SHA-256, xxHash, etc.)
- No trust required — identical content = identical hash
- Problem: two semantically identical messages with different timestamps or metadata would hash differently
- Problem: hash collisions (astronomically rare but theoretically possible)

**Option C: Composite key**
- Combine multiple fields: `(user_id, event_type, timestamp_bucket)`
- More precise but more complex
- Common in analytics pipelines

**Option D: Broker-assigned ID**
- Kafka offset, NATS message ID, SQS message dedup ID
- Tied to infrastructure, not business logic
- What GlassFlow uses (NATS `Nats-Msg-Id` header)

**The choice matters because it defines what "duplicate" means to your system.** Two messages with the same `order_id` but different `status` fields — are they duplicates? Depends entirely on which field you chose as the key.

### 2.3 The Two Fundamental Modes

**Permanent deduplication (infinite window)**
- You remember every key you've ever seen
- A message with a previously-seen key is _always_ dropped
- Simple semantics, but **unbounded memory growth**
- Appropriate when: the key space is finite/bounded, or you genuinely need global uniqueness

**Time-windowed deduplication (finite window)**
- You remember keys for a configurable duration (e.g., 5 minutes, 1 hour)
- After the window expires, the same key is treated as new
- Bounded memory, but a duplicate arriving after the window expires will slip through
- Appropriate when: duplicates arrive within a known time bound (retries, reprocessing)

Most production systems use time-windowed dedup because **the failure modes that produce duplicates are temporally bounded.** A Kafka consumer rebalance creates duplicates within seconds, not days. A producer retry happens within minutes, not weeks.

---

## 3. Data Structures for Dedup State

### 3.1 Hash Map (The Obvious Choice)

```go
seen := map[string]time.Time{}  // key → first-seen timestamp
```

| Aspect | Detail |
|--------|--------|
| Lookup | O(1) average |
| Memory | ~80-120 bytes per entry in Go (key string + time + map overhead) |
| False positives | Zero — exact matching |
| False negatives | Zero — if it's there, you find it |
| Persistence | None — lost on restart |
| TTL support | Manual (sweep goroutine or lazy check on access) |

**When to use:** Small to medium key spaces, ephemeral processes, CLI tools, prototypes. This is what the assignment expects for the basic version.

**The problem at scale:** 100M unique keys * ~100 bytes = ~10 GB of RAM. Not tenable for long-running services with high cardinality.

### 3.2 Bloom Filters (Probabilistic)

A Bloom filter is a space-efficient probabilistic set. It can tell you:
- "Definitely NOT in the set" (no false negatives)
- "Probably in the set" (configurable false positive rate)

```
Insert "abc":  hash1("abc")=3, hash2("abc")=7 → set bits 3,7
Check "xyz":   hash1("xyz")=3, hash2("xyz")=5 → bit 5 not set → definitely new
Check "abc":   hash1("abc")=3, hash2("abc")=7 → both set → probably seen
```

| Aspect | Detail |
|--------|--------|
| Lookup | O(k) where k = number of hash functions |
| Memory | ~10 bits per element for 1% FP rate (vs ~800 bits for hash map) |
| False positives | Yes — configurable (0.1%, 1%, etc.) |
| False negatives | **Never** — if you added it, the filter will always say "probably yes" |
| Deletion | Not supported (standard Bloom filter) |
| TTL | Not native — need rotating Bloom filters |

**The false positive direction matters for dedup.** A false positive means "we think we've seen this, but we haven't" → we drop a message we shouldn't. This is **data loss**. So Bloom filters for dedup need very low FP rates, which eats into their space advantage.

**Rotating Bloom filters for TTL:** Use N bloom filters, one per time bucket. Check all N for membership. When the oldest bucket expires, clear it and reuse. This gives you approximate time-windowed dedup.

### 3.3 Cuckoo Filters

An improvement over Bloom filters that supports deletion:

| Aspect | Detail |
|--------|--------|
| Lookup | O(1) |
| Memory | Slightly better than Bloom for FP rates < 3% |
| False positives | Yes, but lower than Bloom for same memory |
| Deletion | Supported (unlike Bloom) |
| TTL | Possible via deletion |

Relevant when you need probabilistic dedup WITH the ability to expire individual keys.

### 3.4 Key-Value Stores with TTL

This is what GlassFlow uses (Badger), and what production systems typically reach for:

**Embedded KV stores (Badger, BoltDB, LevelDB, RocksDB):**

| Aspect | Detail |
|--------|--------|
| Lookup | O(log n) — LSM tree or B-tree |
| Memory | Configurable — hot data in RAM, cold on disk |
| Persistence | Yes — survives restart |
| TTL | Native in Badger; manual in others |
| Capacity | Limited by disk, not RAM |

**External KV stores (Redis, Memcached):**

| Aspect | Detail |
|--------|--------|
| Lookup | O(1) for Redis hash |
| TTL | Native (`SETEX`, `EXPIRE`) |
| Persistence | Optional (Redis RDB/AOF) |
| Shared | Multiple consumers can share dedup state |
| Trade-off | Network hop per lookup, operational overhead |

The choice between embedded vs external depends on whether dedup state needs to be shared across processes.

### 3.5 Sorted Sets / Skip Lists

Used when you need to efficiently expire by time range:

```
sorted_set: { (timestamp, key), (timestamp, key), ... }
```

You can do range deletions: "delete all entries with timestamp < now - window". Useful when you want to control cleanup precisely rather than relying on per-key TTLs.

### 3.6 Comparison Matrix

| Structure | Memory | Exactness | TTL | Persistence | Best For |
|-----------|--------|-----------|-----|-------------|----------|
| `map[string]T` | High | Exact | Manual | No | Small scale, CLIs, tests |
| Bloom filter | Very low | Probabilistic | Rotation | No | Massive scale, loss-tolerant |
| Cuckoo filter | Low | Probabilistic | Via delete | No | When you need delete + space |
| Badger/RocksDB | Disk-backed | Exact | Native/manual | Yes | Production stream processing |
| Redis | RAM (remote) | Exact | Native | Optional | Shared state across consumers |

---

## 4. Time-Windowed Deduplication

### 4.1 Why Not Just Dedup Forever?

Three reasons:

1. **Memory/storage grows without bound.** If you see 1M unique keys per hour and never forget, after a week you're tracking 168M keys. After a month, 720M. Eventually you OOM or fill the disk.

2. **Legitimate re-use of keys.** A sensor might cycle through device IDs. A user might submit the same form legitimately after a cooling period. Permanent dedup would silently drop valid data.

3. **The duplicate threat is temporally bounded.** Network retries happen within seconds. Consumer rebalances within minutes. There's no mechanism that creates duplicates weeks later.

### 4.2 Expiry Strategies

**Strategy 1: Per-key TTL**
Each key gets a timestamp or TTL when stored. Expired keys are either lazily evicted (checked on access) or actively swept.

```go
type entry struct {
    expiresAt time.Time
}
seen := map[string]entry{}

func isDuplicate(key string, now time.Time, window time.Duration) bool {
    if e, ok := seen[key]; ok && now.Before(e.expiresAt) {
        return true  // seen within window
    }
    seen[key] = entry{expiresAt: now.Add(window)}
    return false
}
```

- Pro: Precise per-key expiry.
- Con: Need cleanup mechanism for expired keys. Lazy-only check means expired keys stay in memory until accessed again (which for dedup might be never).

**Strategy 2: Background sweeper goroutine**
Periodically scan the map and delete expired entries.

```go
go func() {
    ticker := time.NewTicker(sweepInterval)
    for range ticker.C {
        now := time.Now()
        mu.Lock()
        for k, e := range seen {
            if now.After(e.expiresAt) {
                delete(seen, k)
            }
        }
        mu.Unlock()
    }
}()
```

- Pro: Memory stays bounded.
- Con: Full scan is O(n). Sweep interval introduces a lag — entries live slightly beyond their TTL.

**Strategy 3: Time-bucketed maps (generation approach)**
Maintain multiple maps, one per time bucket. Check all buckets. Drop the oldest bucket when it expires.

```go
// With 1-hour window and 10-minute buckets:
buckets := [6]map[string]bool{}  // 6 buckets of 10 min each
currentBucket := 0

// Every 10 minutes: clear oldest bucket, advance pointer
// To check: scan all 6 buckets
```

- Pro: O(1) cleanup (just clear one bucket). No per-key overhead.
- Con: Granularity limited by bucket size. Entries may live up to `window + bucket_size`.

**Strategy 4: Delegate to the storage engine**
Use Badger/Redis TTL and let the engine handle expiry. This is what GlassFlow does.

- Pro: No application-level cleanup code.
- Con: Coupled to a specific storage engine.

### 4.3 Wall Clock vs Event Time

A subtle but important distinction:

**Wall clock time:** The TTL is based on when the dedup service _processes_ the message. Simple, deterministic from the service's perspective.

**Event time:** The TTL is based on a timestamp _within the message_. Correct for out-of-order events, but introduces complexity — what if event time is in the future? What if it's way in the past?

For the assignment and most practical dedup, **wall clock time is fine.** Event-time dedup is a deep rabbit hole (watermarks, late arrivals, etc.) that stream processing frameworks like Flink deal with.

### 4.4 The Edge at Window Boundaries

Consider: message arrives at T=0, window=60s. At T=59, a duplicate arrives — correctly dropped. At T=61, another duplicate arrives — it passes through because the window expired.

This is **by design, not a bug.** But it means:

- Window size should exceed the maximum expected duplicate delay
- In practice: if retries have a 30s timeout, a 5-minute window is very safe
- The assignment specifies 1s to 1 week — the wide range hints they want you to think about this

---

## 5. The Trade-Off Space

### 5.1 Memory vs Accuracy

This is the fundamental tension:

```
More memory = exact dedup (hash map of all keys)
Less memory = probabilistic dedup (Bloom filter)
```

At small scale (< 1M keys), just use a map. At large scale (billions of keys), you need to decide: is a 0.01% false positive rate (= 0.01% data loss) acceptable? For some analytics pipelines, yes. For financial transactions, no.

### 5.2 Latency vs Throughput

**Per-message dedup** adds latency to every message but keeps throughput predictable.

**Batch dedup** accumulates messages, deduplicates within and across the batch, then emits. Higher throughput (fewer store lookups), but adds latency equal to the batch window.

GlassFlow batches up to 50,000 messages or 100ms — a throughput-optimized choice.

### 5.3 Exactness vs Efficiency

| Approach | Guarantees | Cost |
|----------|-----------|------|
| Exact dedup (hash map) | No false positives, no false negatives | O(n) memory |
| Probabilistic dedup (Bloom) | No false negatives, rare false positives | O(1) memory* |
| Content-hash dedup | No false negatives*, extremely rare collisions | O(n) memory, hash computation cost |

*Bloom filters are technically O(n) but with a constant ~10 bits per element vs ~800+ for a hash map.

### 5.4 Simplicity vs Durability

**In-memory (map):** Simple, fast, zero dependencies. Lose all state on crash/restart → duplicates will pass through until the map is repopulated.

**On-disk (Badger, SQLite):** Survives restarts, bounded memory. More complex, I/O latency, compaction overhead.

**External (Redis):** Shared across instances, can survive individual crashes. Network latency, operational overhead, another thing to monitor.

For a CLI tool reading STDIN: in-memory is perfect. For a long-running service: you need persistence.

### 5.5 The CAP Angle (Distributed Systems)

When you have multiple instances of a dedup service (horizontal scaling), you face a distributed state problem:

- **Shared nothing:** Each instance has its own dedup state. A message routed to instance A, then its duplicate routed to instance B — duplicate passes through. Solution: consistent routing (partition by dedup key).
- **Shared state (Redis):** All instances check the same store. Consistent but introduces a bottleneck and a SPOF.
- **Replicated state:** Each instance replicates state to others. Consistency lag means brief windows where duplicates pass.

This is why Kafka's consumer groups use partition assignment — each partition is processed by exactly one consumer, making dedup a local problem.

---

## 6. Dedup in Stream Processing (The Production Context)

### 6.1 Dedup as a Pipeline Stage

In mature stream processing systems, dedup isn't a standalone service — it's one stage in a processing pipeline:

```
Source → [Ingest] → [Filter] → [Dedup] → [Transform] → [Sink]
```

Each stage is a processor implementing a common interface. This is the middleware/chain-of-responsibility pattern. Messages flow through, each processor can:
- Pass messages through unchanged
- Drop messages (filter, dedup)
- Modify messages (transform)
- Add messages to a dead letter queue (error handling)
- Fail fatally (stop the pipeline)

### 6.2 The Two-Phase Pattern

A critical design pattern for dedup in pipelines:

**Phase 1: Read (FilterDuplicates)**
- Check the dedup store
- Separate messages into "new" and "duplicate"
- Do NOT write to the store yet

**Phase 2: Commit (SaveKeys)**
- Only called AFTER downstream processing succeeds
- Persist the keys of successfully processed messages

**Why two phases?** Imagine you write keys immediately in Phase 1, then the downstream sink fails. The messages are NAK'd and will be redelivered. But now the dedup store thinks they're duplicates and drops them. **You've lost data.**

The two-phase approach ensures: if downstream fails, keys aren't committed, so redelivered messages pass through dedup again. This is exactly what GlassFlow does.

### 6.3 Batch vs Per-Message Dedup

**Per-message:**
```go
for msg := range input {
    if !seen[msg.Key] {
        seen[msg.Key] = true
        output <- msg
    }
}
```

**Batched:**
```go
batch := collectBatch(input, maxSize, maxWait)
unique := filterDuplicates(batch, store)
processBatch(unique)
commitKeys(unique, store)
```

Batching amortizes the cost of store operations. Checking 1000 keys in one Badger transaction is much faster than 1000 individual lookups.

### 6.4 What Happens When Dedup Fails?

Three strategies:

1. **Fail open:** If you can't check dedup state, let messages through. Risk: duplicates reach downstream. Used when data loss is worse than duplicates.

2. **Fail closed:** If you can't check dedup state, drop/NAK messages. Risk: data doesn't flow. Used when duplicates are dangerous (financial transactions).

3. **Dead letter queue (DLQ):** Messages that can't be deduplicated go to a separate queue for manual review or retry.

GlassFlow uses fail-closed: dedup errors return `FatalError`, messages are NAK'd for redelivery.

---

## 7. How GlassFlow Does It

### 7.1 Architecture

```
Kafka → Ingestor → NATS JetStream → [Filter → Dedup → Transform] → Sink → ClickHouse
                                            ↓
                                      Badger KV (dedup state)
```

The dedup component runs as a separate process role (`dedup`), reading from NATS and writing back to NATS. It's horizontally scaled by partitioning NATS subjects.

### 7.2 The Badger KV Approach

```go
// Write: store key with TTL, empty value
entry := badger.NewEntry([]byte(msgID), []byte{}).WithTTL(ttl)

// Read: just check if key exists
_, err := txn.Get([]byte(msgID))
if errors.Is(err, badger.ErrKeyNotFound) {
    // Not a duplicate
}
```

Key design choices:
- **Empty values** — only the key existence matters, not what's stored. Minimizes disk usage.
- **Native TTL** — Badger handles expiry and compaction. No application-level cleanup code.
- **Embedded** — no network hop. Dedup latency is just disk I/O (often memory-mapped).

### 7.3 Processor Chain Pattern

```go
type Processor interface {
    ProcessBatch(ctx context.Context, batch ProcessorBatch) ProcessorBatch
}

// Chain: Filter → Dedup → Transform
chain := ChainProcessors(middlewares, transformProcessor)
```

Each processor receives a batch, returns a batch. Processors can attach `CommitFn` functions that run only after the full chain succeeds. The DLQ middleware wraps processors to catch failed messages.

### 7.4 Design Decisions Worth Noting

| Decision | Rationale |
|----------|-----------|
| Uses NATS message ID, not payload field | Decouples dedup identity from business logic |
| Messages without ID always pass through | Fail-open for unidentified messages — data availability over strictness |
| Two-phase commit (filter, then save) | Prevents data loss on downstream failure |
| Batch size 50K / 100ms flush | Throughput-optimized for high-volume streams |
| Badger with nil logger | Silent in production — observability via OpenTelemetry instead |
| `NoopProcessor` when dedup disabled | Avoids nil checks throughout the chain — null object pattern |

---

## 8. Edge Cases and Gotchas

**1. The field doesn't exist in the message.**
What do you do when the dedup field (e.g., `"id"`) is missing from a JSON message? Options:
- Treat as unique (pass through) — GlassFlow's approach
- Treat as error (drop or DLQ)
- Use a hash of the full message as fallback
The assignment doesn't specify. Your choice here shows judgment.

**2. The field value is not a simple type.**
`{"id": {"nested": "object"}}` — deduplicating by `id` means stringifying a JSON object. Different serializations of the same object could produce different strings. Be explicit about how you normalize the key.

**3. Numeric types in JSON.**
JSON has no integer type. `{"id": 1}` and `{"id": 1.0}` may parse differently depending on your JSON library. In Go, `json.Unmarshal` into `interface{}` turns all numbers into `float64`. So `1` becomes `1` as float64, which stringifies as `1` — consistent, but worth being aware of.

**4. Concurrent access to the seen set.**
If you ever parallelize message processing (goroutines per line), the map needs synchronization (`sync.Mutex` or `sync.Map`). For a STDIN line-reader, this isn't needed — but mentioning it in NOTES.md shows awareness.

**5. Memory growth with permanent dedup.**
With no TTL and an unbounded stream, your map grows forever. In a 3-hour exercise, this is fine. In production, this is a memory leak. Mentioning this trade-off is important.

**6. What "deduplication period" means for ordering.**
If messages arrive: A(t=0), B(t=1), A(t=5) with a 10s window — second A is dropped. But if A(t=0), B(t=1), A(t=15) with a 10s window — second A passes through. The ordering of output depends on input timing.

**7. Graceful vs ungraceful shutdown.**
STDIN can end with EOF (graceful) or the process can be killed. If you buffer messages (batching), ungraceful shutdown means losing the buffer. For the assignment (no batching, line-by-line), this isn't an issue.

**8. Empty lines, malformed JSON, whitespace.**
Real-world input is messy. How you handle non-JSON lines (skip silently? write to stderr? exit with error?) is a minor but telling design choice.

---

## 9. The Assignment Through This Lens

The assignment is a simplified dedup scenario:

| Production concern | Assignment scope |
|--------------------|-----------------|
| Kafka/NATS streams | STDIN/STDOUT |
| Broker-assigned IDs | Payload field name (CLI arg) |
| Badger KV with TTL | `map[string]time.Time` is sufficient |
| Two-phase commit | Not needed (no downstream failure risk) |
| Batch processing | Line-by-line is fine |
| Distributed state | Single process |
| Persistence across restarts | Not needed (ephemeral CLI) |
| Billions of keys | Trivial scale |

What remains are the **core concepts:**
1. Extract a key from a JSON message by field name
2. Track seen keys with optional time window
3. Emit only first-seen messages

The simplicity is the point. They want to see how you structure even simple code, not whether you can build Kafka.

**What distinguishes a strong submission:**
- Clean separation: arg parsing, dedup logic, I/O
- Correct TTL: lazy or active expiry, not just a map with timestamps you never check
- Edge cases acknowledged: missing fields, malformed JSON, memory growth
- Testable: dedup logic separated from I/O so it can be unit tested
- Commit progression: scaffold → basic dedup → TTL → edge cases → tests → NOTES.md

---

## 10. Further Reading

- **Designing Data-Intensive Applications** (Kleppmann) — Chapter 11 on stream processing covers exactly-once semantics and dedup
- **Kafka: The Definitive Guide** — Chapter on idempotent producers and exactly-once semantics
- **Bloom Filters by Example** — https://llimllib.github.io/bloomfilter-tutorial/
- **Badger KV design doc** — explains the LSM tree + value log architecture that makes TTL efficient
- **NATS JetStream dedup** — NATS has built-in message dedup using `Nats-Msg-Id` headers, which is what GlassFlow leverages
- **The Two Generals' Problem** — the theoretical impossibility result underpinning why exactly-once delivery is impossible
