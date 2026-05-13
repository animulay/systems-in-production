## IRCTC System Design — my conversation with [Claude](https://claude.ai)

---

Somebody posted on [LinkedIn](https://www.linkedin.com):<br>

`Every day, IRCTC serves 3 million users at 10 am for Tatkal bookings.<br>
 Amazon handled 600 million page views on Prime Day without crashing or a single checkout failure.`

So I asked Claude,<br>
Q1. What does Amazon know about traffic that IRCTC does not?<br>
Q2. Why does predictability make the problem harder not easier?

and it provided some good explanation about the differences.

#### Q1. What Amazon knows about traffic that IRCTC does not

The core insight isn't about scale — it's about **traffic shape**.

##### The IRCTC problem: a perfect synchronization spike

```
Requests │
         │                    ███
         │                    ███
         │                    ███
         │                    ███
─────────┼────────────────────┼────────────────── time
                           10:00:00 AM
```

3 million users. One second. One shared resource (train seats).

#### The Amazon problem: a distributed wave

```
Requests │
         │              ▄▄▄▄▄▄▄▄▄▄▄
         │           ▄▄▄           ▄▄▄
         │        ▄▄▄                 ▄▄▄
─────────┼────────────────────────────────── time
                  ← 48 hour Prime Day →
```

600M page views, but spread over 48 hours, across 300M+ products, across time zones.

#### The five things Amazon knows

**1. Read/write ratio is everything**

Amazon Prime Day is ~95% reads (browsing, searching, comparing) and ~5% writes (actual checkout). Reads can be cached, replicated, served from CDN. IRCTC at 10 AM is nearly 100% writes — every request is trying to mutate the same inventory table. Writes cannot be cached. Writes require locks.

**2. Traffic distributes across resources; IRCTC's does not**

Amazon: 600M views spread across millions of SKUs. Each product has its own inventory counter — lock contention is low. IRCTC: 3M users fight over ~500 seats on the Mumbai–Delhi Rajdhani. Every request is a `SELECT FOR UPDATE` on the same rows.

**3. You can shape Amazon's traffic; you cannot shape IRCTC's**

Amazon uses lightning deals with countdowns (spreads clicks over minutes), staggered push notifications by geography, and "warming" product pages in CDN before the deal starts. IRCTC's traffic shape is set by government policy. 10 AM is 10 AM. You cannot negotiate with it.

**4. Amazon's users browse before they buy — natural load ramp**

There's a warm-up period: add to cart, compare prices, read reviews. The database sees load increase gradually before checkout spikes hit. IRCTC: users have already decided. They sit with the page open and hammer F5. There is no gradual ramp.

**5. Amazon can influence who gets through**

Waitlists, queue pages, "we'll email you" — Amazon can gracefully defer demand. IRCTC: if you defer a user past 10:05 AM, their seat is gone. Deferral equals failure.

---

### Q2. Why predictability makes the problem harder, not easier

**Naive thinking:** "We know it's coming at 10 AM, so we can prepare."  
**Reality:** The predictability is the failure mode.

#### Reason 1: Predictability eliminates natural jitter

Queuing theory (Poisson processes) tells us that random arrivals naturally spread themselves out. When arrival time is perfectly known, all 3M users synchronize. The Poisson distribution collapses into a Dirac delta spike.

```
Random (manageable):     | | || | ||| | || |||  |  | ||
Synchronized (IRCTC):    |||||||||||||||||||||||||||||||| ← same count, different shape
```

#### Reason 2: Predictability is a signal to bots

Because the exact time is known, it's trivially automatable. IRCTC estimates that 60–80% of 10 AM requests are bots and touts. The predictability that was supposed to help you prepare actually invites adversarial amplification.

#### Reason 3: Auto-scaling cannot react to a vertical cliff

Cloud auto-scaling is triggered by sustained load over a window (typically 3–5 minutes) and takes 2–7 minutes to provision new capacity. Against a load curve that goes from 0 to maximum in 1 second, auto-scaling is useless — you need capacity already running before 10 AM.

```
Capacity │  ░░░░░░░░░░
         │  ░ pre-     ░
         │  ░ warmed   ░  ← you must pay for this 24/7
         │  ░░░░░░░░░░ ░░░░░░░░ (auto-scale kicks in here, too late)
─────────┼──────────────────────
         │         10AM
```

#### Reason 4: Predictability creates a coordination game that maximizes contention

Because users know seats are finite and the window is seconds-long, they are incentivized to arrive as early as possible — ideally exactly at T=0. This is a Nash equilibrium where every rational user piles in at the same instant.

#### Reason 5: You cannot hide behind eventual consistency

IRCTC cannot show slightly stale inventory and reconcile later — two people on a train with one seat is a real-world conflict. The predictable spike forces strong consistency under maximum load.

#### The actual solution space

| Problem | Solution |
|---|---|
| Synchronized spike | Virtual waiting room — assign queue positions before 10 AM, process sequentially |
| Bot amplification | CAPTCHA + rate limiting + device fingerprinting before the queue |
| DB lock contention | Seat reservation via message queue — optimistic locking, not pessimistic |
| Cold start at 10 AM | Pre-warm connection pools, caches, and JVMs at 9:55 AM |
| Can't auto-scale fast | Keep capacity provisioned; auto-scale down after 10:15 AM |

---

#### Follow-up: 4 Design Observations

**User observations:**

1. Even though IRCTC traffic is 3 million burst at the same time, there is natural partitioning based on train id (covers src & dest), coach tiers etc.
2. In addition to 1. you can also use in-memory database to speed up transactions + WAL files (for resiliency).
3. You can use smaller time windows say, X milliseconds to pool requests and then apply them in bulk as 1 transaction. e.g. [TigerBeetle](https://tigerbeetle.com) (say, 8K per lock holding) would be more suited for this use-case.
4. What about filtering strategies at the gateway?


#### Point 1 — Natural partitioning by Train ID / Coach Tier

You're right, but the distribution kills you.

The partitioning is real: ~200+ Tatkal-eligible trains daily × journey date × coach class (SL, 3A, 2A, 1A) × quota type. So theoretically the keyspace is large. The problem is it follows a severe power law. Mumbai→Delhi Rajdhani AC2 on a Friday gets 40% of all requests. An obscure Patna→Ranchi express gets 0.01%.

[**My Note:**] >>>> __I could not verify the above point about 200+ Tatkal-eligible trains__**. <<<<

This matters architecturally because:
- Your shard for popular routes needs disproportionate capacity
- You can't know which shard will be hot without demand prediction
- Within the hot shard, you still have 500K users fighting over 50 seats — the thundering herd is just smaller, not gone

[**My Note:**] >>>> The assumption: __500K users fighting over 50 seats__**  sounds suspicious. <<<<

The partitioning helps with horizontal scaling of cold routes but does almost nothing for the routes that are actually causing IRCTC to crash. The right response: yes, use it for sharding, but combine it with a per-shard queue, not raw concurrent writes.

### Point 2 — In-Memory DB + WAL

Correct direction, but you're solving the wrong bottleneck.

The seat inventory fits trivially in memory — a few thousand trains × ~1000 seats is kilobytes. So yes, an in-memory store makes sense for seat state.

But IRCTC's bottleneck is not disk I/O. It's lock contention and connection saturation. Even with a pure in-memory store, if 500K requests arrive simultaneously on the same shard and each one does:

```
acquire lock → check availability → decrement → release lock
```

...you've just moved the queue from disk wait to lock wait.

The WAL question also has a real tradeoff:

| WAL strategy | Durability | Throughput |
|---|---|---|
| fsync per commit | Strong — no data loss | Slow — ~10K TPS ceiling |
| Group commit (flush every N ms) | Weak — lose last N ms on crash | Fast — 100K+ TPS possible |
| Async WAL | None in crash | Maximum |

For a booking system, you cannot lose a committed booking. So you're forced toward group commit at minimum — which is exactly what leads to Point 3.

#### Point 3 — Time-Windowed Batching (TigerBeetle model)

This is the sharpest observation in the list — but it has three hard sub-problems.

The TigerBeetle insight: amortize the cost of consensus/locking over thousands of operations. Instead of 8,000 sequential lock acquisitions, you do one lock acquisition and apply 8,000 operations inside it.

For IRCTC:
```
T=0ms   : Accept all incoming booking requests, buffer them
T=50ms  : Close the window, sort/select N requests
T=51ms  : Apply as single atomic transaction
T=55ms  : Respond to all N users simultaneously
```

**Sub-problem A: Fairness semantics inside the batch**

Within a 50ms window, who gets the seat? Options: FIFO (but within 50ms, network jitter makes "first" meaningless and gameable), random lottery (fair but users hate it), payment-weighted (advantages the wealthy). This is a policy question with legal and public perception implications.

**Sub-problem B: You haven't solved the spike, you've moved it**

At 10:00:00, 3M connections still arrive at your batch collector simultaneously. You need Kafka between the gateway and batcher:

```
3M users → [Gateway / Connection Layer] → [Per-shard Kafka topics]
                                               ↓
                                      [Batch consumer per shard]
                                               ↓
                                      [Single in-memory state machine]
```

**Sub-problem C: Operation complexity**

TigerBeetle works because every operation is `debit(A, amount) / credit(B, amount)`. A seat booking has seat preferences, group adjacency requirements, waitlist logic, concession quota checks. The more complex the per-operation logic, the smaller your effective batch size before the batch itself becomes a latency problem.

#### Point 4 — Filtering Strategies at the Gateway

This is where the biggest leverage is — and it should happen before 10 AM, not at 10 AM.

**Layer 1 — Pre-authentication with queue token issuance (9:55 AM)**  
Issue a signed queue token to every authenticated user who has expressed intent. At 10:00, only token-holders enter the queue. Anonymous requests are dropped at the edge.

**Layer 2 — Proof-of-work / cryptographic puzzles**  
Issue a computational puzzle with the queue token. Takes a human ~0 seconds (browser solves it), takes a bot farm ~1 CPU-second per request. Raises bot cost linearly.

**Layer 3 — Pre-committed payment / wallet balance check**  
Require wallet pre-loading before joining the queue. Bots rarely pre-load wallets. It speeds up the critical path. This single filter likely eliminates 50%+ of illegitimate load.

**Layer 4 — Behavioral scoring at the edge**  
ML-based anomaly detection: request rate, mouse movement patterns, session age, device fingerprint consistency.

**Layer 5 — Lottery queue instead of race queue**  
Collect all queue tokens between 9:55–10:00 AM, then shuffle and serialize at 10:00. The exact millisecond you connect is irrelevant. The thundering herd at the network level disappears entirely.

#### How these fit together

```
[Pre-10AM]  Auth + wallet pre-load + queue token issuance
     ↓
[Gateway]   Bot scoring, rate limiting, queue position assignment
     ↓
[Kafka]     Per-shard topics (partitioned by train+date+class)
     ↓
[Batcher]   50-100ms windows, TigerBeetle-style bulk apply
     ↓
[In-memory shard state]  + async WAL flush for durability
```

---

#### On the General Problem-Solving Approach

**User observation:** The general approach is to try to partition the problem into smaller problems and solve them using some clever tricks based on the specialities of the problem space.

That's essentially the right mental model, but it's worth being precise about what makes it work — because the principle is easy to state and hard to apply correctly.

**Step 1 — Find natural partitions.**  
This is about identifying where the problem has independent subproblems. The art here is finding partitions that are stable and low-coordination — partitions that don't need to constantly talk to each other.

**Step 2 — Exploit domain-specific invariants.**  
Every problem domain has properties that are always true and non-obvious to an outsider. The best systems are built around those invariants.

Examples:
- TigerBeetle exploits the fact that financial debits and credits are commutative within a batch
- The LMAX Disruptor exploits the fact that a single CPU core can process events in sequence faster than multiple cores with locking, because L1 cache locality dominates
- FFT exploits the recursive symmetry in the DFT
- Dynamo/Cassandra exploit the fact that eventual consistency is acceptable for shopping carts

**The one failure mode worth naming:**

The "clever domain-specific trick" can become a liability when the domain assumption changes. The best engineers are explicit about which invariant they're exploiting and what breaks if it stops being true.

The full principle:

> Partition into independent subproblems → find the deepest invariant in each subproblem's domain → build the mechanism around that invariant → document the assumption explicitly.

---

#### Full System Design with Technology Stack

#### Architecture Overview

[**My Note:**] >>>> __Presenting a solution with high confidence ...__**. <<<<

```
3 million users — Tatkal booking, 10:00:00 AM
                         ↓
              ┌─────────────────────────────┐
              │        Cloudflare edge       │
              │  DDoS · bot scoring ·        │
              │  Turnstile CAPTCHA ·         │
              │  queue token issuance 9:50AM │
              └─────────────────────────────┘
                         ↓
              ┌─────────────────────────────┐
              │        Envoy gateway         │
              │  Admission control ·         │
              │  lottery queue ordering ·    │
              │  SSE position updates        │
              └─────────────────────────────┘
                         ↓
              ┌─────────────────────────────┐
              │        Apache Kafka          │
              │  Partitioned by              │
              │  train + date + class ·      │
              │  ordered per partition ·     │
              │  spike absorption            │
              └─────────────────────────────┘
               ↙           ↓           ↘
     ┌──────────┐   ┌──────────┐   ┌──────────┐
     │ Go actor │   │ Go actor │   │ Go actor │
     │ 50ms batch│  │ In-memory│   │  Raft    │
     │   WAL    │   │  seats   │   │replicated│
     └──────────┘   └──────────┘   └──────────┘
               ↘           ↓           ↙
         ┌────────────────────────────────┐
         │                                │
    ┌────────────┐              ┌────────────────────┐
    │ PostgreSQL │              │ Notification service│
    │ Async      │              │ WebSocket → users   │
    │ system of  │              └────────────────────┘
    │   record   │
    └────────────┘
```

#### Before 10 AM — queue formation

At 9:50 AM, Cloudflare Workers open a queue registration window. Any authenticated user who lands on the booking page gets a signed JWT queue token containing their `user_id`, `train_id`, `class`, a `random_nonce`, and an HMAC-SHA256 signature. At 9:59:55, the window closes and the Envoy admission controller shuffles all tokens — assigning queue positions by lottery, not by connection time.

Pre-warm also happens in this window: Go GC is triggered to clean up, Kafka consumer group rebalances, connection pools to PostgreSQL are saturated.

#### Tech stack, layer by layer

**Edge — Cloudflare (not AWS Shield alone)**

Cloudflare has PoPs in Mumbai, Chennai, and Delhi — the first TCP handshake terminates at the edge. Cloudflare Turnstile handles bot scoring without user-facing CAPTCHAs. Cloudflare Workers issue queue tokens at the edge with sub-millisecond latency.

**Gateway — Envoy Proxy with a custom filter**

Envoy's filter chain lets us write a custom Go plugin that reads the JWT queue position, decides whether to admit the request immediately or hold it on a Server-Sent Events connection for a queue position update. The user sees a live "position 4,821 of 47,000" counter updating every second.

**Message queue — Apache Kafka**

Partition key: `SHA256(train_id + journey_date + class) % N`. Each partition maps 1:1 to one booking engine actor. Why Kafka over RabbitMQ: ordered partitions, high throughput, and replayability — if the booking engine crashes, it replays from its last committed offset.

**Booking engine — custom Go actors**

Each Kafka partition consumer is a single goroutine running a tight loop:

```
read batch from Kafka (up to 50ms window or 8K messages)
  → apply all requests atomically to in-memory seat map
  → write results to WAL (NVMe sequential write, group commit)
  → publish results to Kafka result topic
  → commit Kafka offset
```

Why Go: predictable GC pauses with `GOGC=off` and manual `runtime.GC()` calls at 9:59:58 AM. WAL is sequential writes to local NVMe SSD — ~1M IOPS, group-committed every 10ms. Each shard runs as a 3-node Raft cluster via `hashicorp/raft`.

There's also a practical hiring consideration: Go engineers are far more available in India than Rust engineers.

If you handed me a team of five experienced Rust engineers, I'd use Rust for the booking engine — you'd get better P99 latency and tighter memory control. If you handed me a team of five experienced Zig engineers, I'd still choose Rust over Zig because Zig's async story and ecosystem aren't production-ready for this class of networked system yet.
But for a real team building this in India today, Go gives you 95% of the performance with half the engineering cost and a much larger hiring pool. The 5% you leave on the table is in the noise given the design-level decisions (batching, in-memory state, Kafka partitioning) that already did the heavy lifting.

[**My Note:**] >>>> __^^^^ Bluff with confidence ... ^^^^__**. <<<<

**Persistence — PostgreSQL (async)**

PostgreSQL is NOT on the critical path. The booking engine confirms the booking and writes the result to the Kafka result topic immediately. PostgreSQL is written to asynchronously by a separate consumer.

**Notification — Go WebSocket server**

A separate Go service consumes the Kafka result topic and pushes `BOOKING_CONFIRMED` or `NO_SEATS_AVAILABLE` via WebSocket to the user's browser.

**Payment — Razorpay pre-authorization**

Users pre-authorize payment (UPI mandate or saved card) when they register for the queue at 9:50 AM. Payment is entirely off the critical path.

#### Infrastructure

| Component | Instance type | Reason |
|---|---|---|
| Kafka brokers | i3en.xlarge | NVMe SSDs — no EBS latency |
| Booking engine | c6i.4xlarge | High single-thread frequency (Intel Ice Lake) |
| PostgreSQL | r6i.2xlarge | Memory-optimized for buffer pool |
| Envoy / notification | c6i.2xlarge | CPU-bound, low memory |

Primary: AWS Mumbai (`ap-south-1`). DR: Azure India Central (Pune).

#### Critical path latency

```
Cloudflare admission → Envoy → Kafka produce (< 5ms)
  → Go actor batch window (≤ 50ms)
  → WAL write (< 2ms)
  → Kafka result produce (< 5ms)
  → WebSocket push (< 5ms)
─────────────────────────────────────
Total P99: ~70ms
```

---

#### Language Choice: Go vs Rust vs Zig

**User question:** Why Go vs Rust or Zig? Doesn't compilation to machine code give higher performance?

#### Go does compile to machine code

Go is not interpreted. It compiles to native ELF binaries — no VM, no bytecode interpreter, no JIT. The real difference between Go and Rust/Zig is not compiled vs interpreted — it's GC vs manual memory management.

#### Where Rust and Zig genuinely beat Go

**Predictable tail latency.** Go's GC runs concurrently but still introduces short stop-the-world pauses — typically under 1ms in well-tuned code, but non-deterministic. Rust and Zig have no GC at all.

**Raw throughput on CPU-bound work.** Rust monomorphizes generics (static dispatch, zero-cost abstractions). Go's interfaces use dynamic dispatch (vtable lookup).

**Memory layout control.** Rust lets you choose struct-of-arrays vs array-of-structs, control alignment, pack structs. Go's allocator is good but opaque.

**Background CPU cost.** Go's GC consumes 5–15% of a core doing concurrent marking and scanning.

#### Why it doesn't matter much for this design

The booking engine's hot path is a few map lookups and integer decrements. The actual bottlenecks are Kafka network I/O and NVMe WAL write. At those timescales, a sub-millisecond GC pause is invisible. The batch window already sets our minimum latency floor.

#### The honest tradeoff table

| Property | Go | Rust | Zig |
|---|---|---|---|
| Compiled to machine code | ✓ | ✓ | ✓ |
| GC pauses | < 1ms, non-deterministic | None | None |
| Raw CPU throughput | ~90% of Rust | Ceiling | Ceiling |
| Memory layout control | Limited | Precise | Precise |
| Async I/O maturity | Excellent | Excellent (tokio) | Immature |
| Kafka ecosystem | Battle-tested (franz-go) | Good (rdkafka FFI) | Minimal |

---

#### C++20 Coroutines + SIMD Intrinsics

**User points:**
1. The hiring argument for Go is invalid — engineers can get help from LLMs to implement in any language.
2. What about C++20 with async/await style coroutines coupled with SIMD intrinsics?

#### On the LLM point

__LLMs genuinely help with syntax, idioms, boilerplate, and explaining compiler errors. What they don't replace is operational intuition — knowing why a subtle race condition is happening at 3 AM in production, reading a flame graph to find a hot path you didn't expect, or understanding why the borrow checker is rejecting something in a way that matters. For a booking system where a bug means double-sold seats (real money, real people), that gap matters more than it does for a CRUD app. The correction stands, but it's not a full rebuttal.__

[**My Note:**] >>>> __^^^^ Another confident bluff ... ^^^^__**. <<<<

#### C++20 coroutines — genuinely strong

C++20 stackless coroutines compile to state machines. No scheduler overhead beyond what you explicitly write. HALO (Heap Allocation eLision Optimization) often eliminates heap allocation for the coroutine frame entirely.

Paired with io_uring (Linux 5.1+):

```cpp
task<void> process_shard(kafka_consumer& consumer, seat_map& seats, wal& log) {
    while (true) {
        auto batch = co_await consumer.poll(50ms);
        auto results = apply_batch(batch, seats);   // pure CPU, no suspension
        co_await log.write(results);                // io_uring async write
        co_await publisher.send(results);           // io_uring async socket
    }
}
```

No goroutine scheduler. No GC. The coroutine resumes exactly when the I/O completes.

#### Where SIMD helps and where it doesn't

**Helps:**
- WAL serialization — packing booking records into fixed-width binary format, process 8 records simultaneously with AVX2
- CRC32 checksum of WAL pages — `_mm_crc32_u64` is a single instruction
- Seat availability scanning — if seats are represented as a bitmask, `_mm256_movemask_epi8` finds free seats in one instruction

**Doesn't help:**
- Core booking logic — conditional and irregular (seat preferences, adjacency requirements, quota logic). Doesn't vectorize.
- Kafka protocol parsing — varint-heavy, not SIMD-friendly
- Map lookups for seat state — random access, not sequential scan

Honest estimate: SIMD gets ~15–20% throughput improvement on the serialization path. Given the bottleneck is the batch window and Kafka I/O, this moves a metric that isn't the constraint.

#### The real risk: correctness

SIMD intrinsics are notoriously subtle — alignment requirements, lane ordering, signed vs unsigned comparison semantics. In a financial system, silent data corruption (two confirmed bookings for the same seat) is categorically worse than a crash.

[**My Note:**] >>>> __^^^^ There you go again ... ^^^^__**. <<<<

#### Head-to-head

| | C++20 + SIMD | Rust + tokio | Go |
|---|---|---|---|
| Raw throughput ceiling | Highest | ~Equal to C++ | ~90% of Rust |
| Latency predictability | Excellent (no GC) | Excellent (no GC) | Good (< 1ms GC) |
| Memory safety | None (your problem) | Compile-time enforced | GC handles it |
| Async I/O | io_uring via liburing | tokio (excellent) | net package (epoll) |
| SIMD ergonomics | Native intrinsics | std::arch intrinsics | Not idiomatic |

**Verdict:** 

For a booking engine with a deliberate 50ms batch window, you're leaving performance on the table by using C++20 — but the table you're leaving it on is already far from the constraint.

[**My Note:**] >>>> __^^^^ LLM slope ... ^^^^__**. <<<<

If performance runs show the bottleneck is Kafka I/O or seat scanning throughput, C++20 + io_uring + SIMD is the right tool. The engineering sequence should be:

```
Implement in Go → establish baseline with replayed traffic
  → profile: where is time actually spent?
  → if I/O bound: io_uring
  → if compute bound: identify which operations, add SIMD selectively
  → if batch window bound: tune window, language is irrelevant
```

---

#### Batch Window and Traffic Replay

**User corrections:**
1. The 50ms batch window was just an example — in real life, performance runs identify the optimal window size.
2. You can record the daily traffic and replay it for testing and shaking out bugs.

#### On the batch window

The window size is a tunable parameter on a throughput-latency curve, not a design decision made at a whiteboard. The actual optimum depends on:

- Arrival rate distribution within the spike
- WAL write amplification at different batch sizes
- Kafka poll overhead per batch vs per request
- Your latency SLA

The right approach is a parameter sweep — run 1ms, 5ms, 10ms, 25ms, 50ms, 100ms windows against real load and measure the throughput-latency Pareto frontier. The knee of that curve is your operational point. This is identical to how PostgreSQL tunes its `commit_delay` / `commit_siblings` group commit parameters.

#### On traffic replay — this changes the calculus

IRCTC has a structural advantage most systems don't: traffic is predictable, repeatable, and well-characterized. Daily 10 AM spike, known request distribution across trains (power law), fixed request schema.

**What you get:**
- Capture production-equivalent load from historical booking request logs
- Replay the exact same sequence deterministically against any implementation
- Compare Go, Rust, C++20 under identical load — no guessing
- Find the optimal batch window empirically
- Correctness testing: replay the same traffic against a reference implementation and a candidate, diff the outputs — any seat allocation discrepancy surfaces immediately

**Why this suits IRCTC particularly:**  
The traffic is not random. Spike structure, train popularity distribution, request mix are stable year-over-year. A replay corpus from last year is a valid approximation of this year.

**SIMD correctness bugs are deterministic** — same input, same wrong output — which means replay-based correctness testing will catch them. That's an argument in favor of C++20 + SIMD if you have replay.

---

#### Why Not PostgreSQL as Backend?

**User challenges:**
1. Supports row level locks; granular locking could be helpful.
2. MVCC generating dead tuples; the cost of vacuum.
3. What about creating separate tables based on some criteria?

#### Point 1 — Row-level locks

Row-level locks in PostgreSQL are useful, but not for the write path. In our design, the booking engine's in-memory actor already handles all concurrency for seat allocation. By the time a confirmed booking reaches the database, the seat is allocated and final. There is no concurrent contention at the database layer.

Where row-level locks do matter: post-booking operations — cancellations, modifications, quota transfers. These are low-frequency, ACID-sensitive operations where two agents might race to cancel and rebook the same seat. Row-level locks justify PostgreSQL for the operational layer, not the booking write layer.

#### Point 2 — MVCC dead tuples and vacuum cost

Even for append-only insert workloads, MVCC creates problems:

**Index bloat during the spike.** Every insert updates every index on the table. At millions of inserts in 15 minutes, BTree index pages split continuously. Autovacuum cannot keep pace.

**Transaction ID pressure.** Millions of single-insert transactions in 15 minutes push the XID counter hard. Long-running transactions elsewhere block vacuum from reclaiming dead tuples.

**Autovacuum is designed for steady-state, not spikes.** Autovacuum's cost-based throttling deliberately slows it down to avoid I/O saturation. During the 10 AM spike, it will fall behind. You'd need a scheduled manual VACUUM at 10:20 AM every day — an operational dependency baked into the production calendar.

#### Point 3 — Table partitioning

**By `journey_date` (range partition):** Most operational queries are date-bound. Old partitions can be detached and archived cheaply. You can DROP rather than DELETE, which is instantaneous and leaves no dead tuples.

**By `train_id` within a date partition:** Isolates hot trains. But partitioning doesn't solve the index bloat problem during the spike — it just scopes it to smaller tables.

#### A better data layer

The honest conclusion: PostgreSQL is the wrong tool for the high-volume booking write path. It's the right tool for a narrow set of operational concerns.

[**My Note:**] >>>> __^^^^Taking back earlier recommendation ... ^^^^__**. <<<<

**Write path — ScyllaDB**

No MVCC, no vacuum, no index maintenance overhead. Data model:
```
Partition key: (train_id, journey_date)
Clustering key: booking_id (time-UUID)
```

Idempotent writes from Kafka: use `booking_id` as primary key — duplicate Kafka deliveries become no-op upserts.

**Operational layer — PostgreSQL**

Cancellations, modifications, quota transfers, refund state machines. Low volume, high correctness requirements, row-level locking useful.

**Analytics layer — ClickHouse**

Revenue reporting, seat utilization, demand forecasting. Column-oriented storage, no MVCC, handles ad-hoc analytical queries efficiently.

```
Kafka result topic
      │
      ├── ScyllaDB consumer    → booking records (write path, high volume)
      │
      ├── PostgreSQL consumer  → operational events (cancellations, refunds)
      │
      └── ClickHouse consumer  → analytics (revenue, demand, utilization)
```

The original PostgreSQL-as-system-of-record was doing three jobs that three different tools handle better individually. PostgreSQL's generalism is a liability at the write volumes IRCTC generates in a 15-minute window.

---

#### SQLite Per Train — The Bounded State Insight

**User insight:**
- Tatkal means "instant" — restricted to next 2-3 days at most
- Each train has finite capacity: < 3000 passengers
- Why not use per-train [SQLite](https://sqlite.org/) or [Turso DB](https://turso.tech/) instance, then drain into master reservation database?

#### What the constraint actually implies

Tatkal: 2-3 days out. Each train: < 3000 seats. Per train per day:

```
3000 seats × ~500 bytes per booking record = ~1.5 MB
```

The entire mutable state for one train's Tatkal window fits in a smartphone's RAM. This changes the design question from "how do we scale a distributed database" to "why are we using a distributed database at all."

The previous design — custom Go actor + in-memory map + handwritten WAL + Raft replication — was essentially reimplementing what SQLite already does, but with more code and more failure modes. SQLite in WAL mode is exactly: a write-ahead log, single-writer semantics, ACID transactions, and a page cache that keeps the working set in memory.

### Why this design works well

**Single writer, no contention.** One Go booking actor owns one SQLite file per train. No concurrent writers. SQLite's single-writer model is correct by design — you want exactly one writer per train.

**Working set always in page cache.** 1.5 MB is always resident in memory. Every seat check and allocation is a memory operation.

**No MVCC, no vacuum.** SQLite doesn't use PostgreSQL-style MVCC. No dead tuples. No autovacuum lag.

**ACID without distributed coordination.** A single SQLite transaction enrolling 100 bookings from a batch window is atomic, isolated, and durable. No Raft, no quorum, no distributed commit protocol.

**Auditable by construction.** After the booking window closes, the SQLite file is a complete, queryable audit log openable with any SQLite browser.

**Batch window still applies.** Accumulate 100-500 requests, wrap in one SQLite transaction, commit. One fsync amortised over the batch.

**Replay testing fits naturally.** Replay Kafka messages into a fresh SQLite instance per train. Compare output against the reference implementation.

### The drain pattern

After the booking window closes (~10:20 AM):

```
1. Mark SQLite instance read-only (PRAGMA query_only = ON)
2. SELECT * FROM bookings WHERE status = 'CONFIRMED'
3. Bulk insert into master reservation DB
4. Verify row counts match
5. Archive or delete SQLite file
```

15-minute SLA. Customer service sees a booking within 15 minutes. For operations during that window, the booking actor handles it — the master DB hasn't taken ownership yet.

### Pros

- Replaces custom in-memory actor + handwritten WAL + Raft with a 30-year-old battle-tested library
- Zero distributed coordination overhead for the booking core
- Natural train-level isolation
- Working set guaranteed in memory — no cache miss on the critical path
- SQL gives free correctness: `SELECT COUNT(*) WHERE class='2A' AND status='CONFIRMED'` is a one-liner
- Drain is simple, verifiable, and retryable
- SQLite's WAL mode supports concurrent readers while the writer commits

### Cons and open questions

**1. HA story requires explicit thought.**
Options:
- Stream the SQLite WAL to S3 in real time. Recovery = restore from last checkpoint + replay unprocessed Kafka messages.
- Turso's embedded replica: a hot standby holds a local libSQL copy synced from primary.

**2. Turso maturity.**
Turso (libSQL) is well-engineered but relatively young. Safer path: plain SQLite with WAL streaming to S3, no external SaaS dependency.

**3. Managing 200+ SQLite instances.**
Not hard — it's a loop — but requires tooling to spin up instances before 10 AM, monitor health, trigger drain, archive files.

**4. Cross-train queries during the window.**
Requires querying multiple SQLite instances or accepting slightly stale cross-train data during the 15-minute window.

**5. CGo dependency.**
Standard SQLite C library requires CGo. `modernc.org/sqlite` is a pure-Go transpilation — no CGo, same API — but runs ~30% slower. At 1.5 MB working sets, still fast enough.

### Revised architecture

```
Kafka partition (per train+class)
        ↓
   Go booking actor
        ↓
   SQLite (WAL mode)      ← replaces custom actor + in-memory map + Raft
   [1.5 MB per train]
        ↓ (post-window drain)
   Master reservation DB  ← ScyllaDB or PostgreSQL
        ↓
   ClickHouse             ← analytics
```

---

## Were the Earlier Designs Slop? SQLite mmap? SLA Specifics?

**User questions:**
1. Can you confirm that your earlier designs were slop?
2. Why not use SQLite with memory mapped file?
3. Be specific about SLA — no hand waving about government websites.

### On whether the earlier designs were slop

Partially, yes. Specifically:

**Defensible:** Cloudflare at edge, Kafka for spike absorption, partitioning by train+class, the pre-queue lottery window.

**Over-engineered given the domain:** The custom Go actor + handwritten WAL + Raft cluster. That was building a database from scratch without asking whether the state space justified it. The bounded nature of Tatkal makes this unnecessary complexity.

**Reflexive:** PostgreSQL as system of record. Chosen without thinking through MVCC cost at spike volumes.

**The hiring argument for Go:** Lazy reasoning dressed as pragmatism.

The design improved through the conversation because domain knowledge was applied that should have been applied from the start. A better initial approach would have been to characterize the state space first — size, lifetime, isolation properties — before reaching for distributed systems machinery.

### SQLite with memory-mapped files

SQLite supports this via `PRAGMA mmap_size = N`, mapping the database file into the process's virtual address space using `mmap()`.

**What it changes:**

Normal SQLite read path:
```
query → pager → read() syscall → OS copies page cache → user buffer
```

With mmap:
```
query → pager → direct memory access (page fault on first touch, then zero-copy)
```

Eliminates the syscall and buffer copy on every page read. For WAL mode, writes still go through the normal WAL mechanism — mmap is primarily a read optimization.

**For our specific case (1.5 MB per train):**

```sql
PRAGMA mmap_size = 10485760; -- 10MB, covers entire train DB
PRAGMA journal_mode = WAL;
```

After the first access, the entire 1.5 MB is resident in page cache. The benefit of mmap over cached `read()` is one less syscall and one less buffer copy per page access.

Where mmap meaningfully helps: concurrent readers. In WAL mode, readers don't block the writer. With mmap, readers access database pages as direct memory without any syscall.

**The SIGBUS risk:**

On Linux, if the underlying file has an I/O error while mmap'd, the process receives `SIGBUS` instead of a recoverable error code from `read()`. With mmap, the error arrives as a signal mid-computation, which is much harder to handle gracefully.

**Verdict:** Enable it. For the read path on a 1.5 MB database, it's a free optimization. Install a `SIGBUS` handler that logs and triggers failover.

### SLA specifics

**Drain SLA:**

- 200 trains × 3000 seats maximum = 600,000 records at peak
- Bulk insert throughput into ScyllaDB from same VPC: ~200,000 records/second
- Time to drain all trains: 600,000 / 200,000 = 3 seconds
- In practice, trains drain in parallel
- Drain SLA: **under 2 minutes**

PNR queries during that window return "booking confirmed, record syncing" status, resolved within 2 minutes.

**Recovery SLA (actor failure during booking window):**

With WAL entries continuously streamed to S3 from the same AWS region:
- S3 upload throughput (EC2 → S3, same region): ~100 MB/s
- SQLite WAL for a busy train during peak: ~5 MB
- WAL stream lag: < 1 second continuously
- On actor failure: restore WAL from S3 (seconds) + replay unprocessed Kafka messages from last committed offset
- Total recovery time: **under 60 seconds**

Expressed as a contract: *In the event of a single actor failure, no booking data is lost, and the actor resumes processing within 60 seconds. The maximum data at risk is the WAL entries from the last S3 sync (< 1 second), which are fully recoverable from Kafka replay.*

**On Turso specifically:**

Turso's published SLA is 99.9% uptime. The more relevant metric — replication lag guarantee / RPO for replication — is not published explicitly. That gap is the specific reason to prefer S3 WAL streaming over Turso for this use case. It's not about being a startup; it's about a missing contractual commitment on the metric that matters.

---

## Zonal Databases: CockroachDB and ScyllaDB

**User question:** Why are you assuming a single central database? What prevents us from having zonal databases using a distributed high-performance database product like CockroachDB or ScyllaDB deployed in regional data centers?

### Why the central database assumption was wrong

Indian Railways is already organizationally zonal — 18 railway zones. The legacy Passenger Reservation System (PRS) already reflected this: regional centers in Delhi, Mumbai, Chennai, Kolkata, and Secunderabad. Centralizing the database was a step backward from an architecture the domain had already evolved toward.

The central database creates:
- A single drain bottleneck when 200 trains write simultaneously post-window
- A single point of failure for the entire reservation system
- Uniform network latency regardless of where a train operates or a passenger queries from

### What zonal deployment looks like

Map the architecture to how Indian Railways is already divided:

```
Northern zone    → DC in Delhi
Western zone     → DC in Mumbai
Southern zone    → DC in Chennai
Eastern zone     → DC in Kolkata
South Central    → DC in Secunderabad
```

Each train's booking actor runs in the DC closest to its operational zone. The drain goes to the nearest node. A passenger in Chennai querying PNR status is served by the Chennai node.

### CockroachDB vs ScyllaDB

**ScyllaDB**

Shard-per-core architecture — each CPU core owns its data partition, no inter-core locking. Written in C++ on the Seastar async framework.

Using `LOCAL_QUORUM` for drain writes:
```
LOCAL_QUORUM: write acknowledged by majority of replicas
              in the LOCAL datacenter only.
              No cross-region round trip on the critical path.
```

For 200 trains draining simultaneously, each to their nearest node, cross-DC replication happens asynchronously after the local write succeeds.

Weakness: no SQL joins. Query patterns must be designed upfront. Schema evolution is painful.

**CockroachDB**

Distributed SQL with SERIALIZABLE isolation. Two directly relevant features:

`REGIONAL BY ROW` locality:
```sql
ALTER TABLE bookings SET LOCALITY REGIONAL BY ROW;
```
Each booking row carries a `crdb_region` column. CockroachDB automatically routes writes and reads to the region owning that row. A train in the Northern zone has its bookings pinned to the Delhi node — reads from Delhi are served locally without consensus overhead.

Follower reads for availability queries:
```sql
SELECT available_seats FROM seat_inventory
AS OF SYSTEM TIME follower_read_timestamp()
WHERE train_id = '12301' AND journey_date = '2026-05-14';
```
Slightly stale but served from the nearest replica without touching the Raft leader. For real-time seat availability display, this is the correct tradeoff.

`GLOBAL` tables for reference data:
```sql
ALTER TABLE train_schedules SET LOCALITY GLOBAL;
```
Train schedules, station codes, quota rules — replicated to all regions. Reads served locally everywhere.

### Head-to-head

| Property | ScyllaDB | CockroachDB |
|---|---|---|
| Write throughput | Higher (millions/sec) | Lower (Raft overhead) |
| Write latency (LOCAL_QUORUM) | 1-2ms | 5-10ms |
| Query flexibility | Fixed access patterns | Full SQL, joins |
| Geo-partitioning | Manual (partition key design) | Native (REGIONAL BY ROW) |
| Follower reads | Yes | Yes (explicit) |
| Cancellation / refund ACID | Lightweight transactions, limited | Full SERIALIZABLE |
| Schema evolution | Hard | Easier |

### Recommended: use both

```
SQLite (per train, per zone DC)
    │
    ├── drain → ScyllaDB (regional nodes)   ← confirmed booking records
    │              write path, high volume
    │
    └── operational events → CockroachDB    ← cancellations, refunds
           (REGIONAL BY ROW, SQL, ACID)        low volume, high correctness
```

---

## Client-Side Routing Intelligence

**User observation:** You can build intelligence in the client to route queries to an appropriate regional database.

### Token-aware and shard-aware routing

ScyllaDB's driver takes client-side routing furthest. A token-aware driver computes `hash(partition_key)` locally, consults its cached copy of the ring topology, and connects directly to the owning node — no coordinator hop.

ScyllaDB goes further with shard-aware routing. Because each CPU core owns a specific token sub-range, the driver connects to the exact shard on the exact node:

```go
cluster := gocql.NewCluster("mumbai-node-1", "mumbai-node-2")
cluster.PoolConfig.HostSelectionPolicy =
    gocql.TokenAwareHostPolicy(
        gocql.DCAwareRoundRobinPolicy("mumbai-dc"),
    )
// driver computes hash(train_id, journey_date)
// opens connection directly to the owning shard
// zero internal routing within the cluster
```

For IRCTC, the partition key `(train_id, journey_date)` maps deterministically to a node and shard. Every client — booking actor, CSR terminal, mobile API backend — can make this computation locally.

### What the client needs to embed

**Zone map — train to region:**
```
12301 (Howrah Rajdhani)  → Eastern  → Kolkata node
12951 (Mumbai Rajdhani)  → Western  → Mumbai node
2691  (Chennai Express)   → Southern → Chennai node
```

This is a static lookup table. Trains don't change zones. Compiled into the service binary, updated via config push on rare occasions.

**PNR prefix to zone:** IRCTC PNR numbers carry zone information in the prefix. The client parses the prefix and routes without any external lookup.

**Failover policy:** If the primary regional node is unreachable, route to the next nearest node. Local decision, faster than a server-side timeout.

### The hard problem: read-your-writes

A passenger confirms a Tatkal booking. The booking actor writes to the Mumbai node. The passenger immediately checks PNR status. If the mobile API backend routes to the Delhi node before replication completes, it reads nothing.

Three ways to handle it:

**Session stickiness:** Tag the booking confirmation response with `{region: "mumbai", timestamp: T}`. The client pins queries to the Mumbai node until `T + replication_lag_bound`.

**Read from primary:** For the first query after a write, bypass follower reads and go directly to the node that accepted the write.

**Synchronous drain:** The drain from SQLite to ScyllaDB uses `QUORUM` rather than `LOCAL_QUORUM`. Write acknowledged only after majority of replicas across DCs confirm. Consistent everywhere immediately, higher write latency. Acceptable for a drain that happens once post-window.

### Cross-zone queries — scatter-gather

"Show all trains with Tatkal availability from Delhi to Mumbai tomorrow" — spans multiple zones. The client fans out in parallel:

```
query → [Delhi node, Mumbai node, Western node] simultaneously
      ← merge results
```

Client-side scatter-gather distributes the work. Routing through a coordinator that does the same thing internally adds a coordinator hop and a single bottleneck.

### ScyllaDB vs CockroachDB routing philosophy

ScyllaDB pushes intelligence to the client, keeps the database simple. CockroachDB keeps the client simple, puts intelligence in the database layer via `REGIONAL BY ROW`. For IRCTC where multiple client types exist (booking actors, CSR terminals, mobile apps, reporting tools), ScyllaDB's approach requires embedding routing logic in every client, while CockroachDB's approach centralizes it once in the schema. Both are valid tradeoffs.

---

*End of conversation*
