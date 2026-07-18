# Platform Events — Active Reading Kit (for a senior architect)

**Source:** Salesforce *Platform Events Developer Guide*, v67.0 (Summer '26) — `platform_events.pdf`
**You:** 11 yrs Salesforce, read this doc multiple times.
**Problem you're actually solving:** re-reading hasn't produced durable, design-ready mastery.
**Fix:** stop reading to absorb; read to *verify*. Retrieve first, then read only to close gaps.

---

## Why the previous reads didn't stick

Re-reading familiar text creates the **illusion of fluency**: it feels easy → brain says "known" → you never actually test recall. Recognition ≠ recall ≠ design-under-pressure. The lever isn't another pass; it's **retrieval** (pull it from memory), **spacing** (revisit over days), and **desirable difficulty** (make yourself struggle before you look).

You already own the concepts. What's still soft is almost certainly the **precise contract**: delivery/ordering guarantees, publish-behavior semantics, retry mechanics, exact allocations, replay/retention windows, subscriber-context limits. That's what wins design reviews — and it's exactly what linear reading skims.

---

## The method — 4 moves per topic

For each topic below:

1. **Blank-page it.** Before opening the PDF, write your answer to the interrogation questions from memory. Cite the number if you think you know it.
2. **Read to verify, not to cover.** Open only the relevant pages. Confirm or correct what you wrote. Mark each item: ✅ knew it · ⚠️ fuzzy · ❌ wrong.
3. **Capture the delta.** Only the ⚠️/❌ items go into your durable sheet (below). Ignore the ✅ — don't re-read what you proved you know.
4. **Space it.** Re-attempt the ❌ questions from memory 2 days later, then a week later. No re-reading in between.

> Read for the **edges**, not the core. Skip the concept chapters (pp. 1–5) and the standard-object catalog (pp. 143–657, lookup only). Spend your attention on **Considerations (114–121)**, **Allocations (132–143)**, **Error Codes (143)**, **Publish behavior (6–10)**, **Apex publish/subscribe + retries (14–31, 42–63)**, **Pub/Sub API**, and **Testing (97–104)**.

---

## The durable artifact you're building

One file: `ARCHITECT_DECISION_SHEET.md`. Not notes — a **defensible one-pager** you'd bring to a design review. Sections:

- **Guarantees** (delivery, ordering, durability) — one line each, exact.
- **Allocations & limits** — the numbers, with the edition/qualifier.
- **Publish semantics** — after-commit vs immediate, rollback behavior, running user/context.
- **Subscribe semantics** — batch size, retries, resume/checkpoint, poison-message handling.
- **Decision matrix** — Platform Events vs CDC vs Streaming/PushTopic vs Outbound Msg vs Pub/Sub API, with the *one* discriminator that picks each.
- **Gotchas** — the things that bite in production.

If you can't fill a line from memory, that's your reading target. When this sheet is complete and you can reproduce it blank, you've mastered the doc.

---

## Interrogation bank — answer from memory FIRST, then verify

These are the questions a design review will actually ask. Attempt each cold. Then find the page that confirms/corrects it.

### A. Delivery, ordering & durability
- [ ] Delivery guarantee: at-least-once, at-most-once, or exactly-once? What does that force you to design for?
- [ ] Are events ordered? Within one `publish()` call? Across calls? Per subscriber?
- [ ] Do all subscribers share one event stream/replay position, or does each track its own?
- [ ] Event retention window (high-volume)? What happens to `ReplayId` after it expires?
- [ ] Standard-volume vs high-volume platform events — differences in retention, delivery, allocations?

### B. Publish semantics
- [ ] "Publish After Commit" vs "Publish Immediately": when exactly does each dispatch?
- [ ] Transaction rolls back after `EventBus.publish()` — is the event delivered? For which publish behavior?
- [ ] What does `Database.SaveResult` from `EventBus.publish()` actually tell you (and not tell you)?
- [ ] Can you get a publish confirmation/callback? How, and what are its limits?
- [ ] Which governor limits does publishing consume (DML? a separate limit)?

### C. Subscriber context (Apex trigger)
- [ ] Which user runs the subscriber trigger, and what does that restrict?
- [ ] Default event batch size per trigger execution? How do you reduce it and why would you?
- [ ] `EventBus.RetryableException`: max retries? Over what time window? What happens after the max?
- [ ] `setResumeCheckpoint()` / resume mechanics — what problem does it solve, and its constraints?
- [ ] Poison message: how do you stop an infinite retry loop by design?
- [ ] `EventBus.TriggerContext` — `lastError`, `retries`, `resumeCheckpoint`: when do you read each?

### D. Idempotency & correlation
- [ ] `EventUuid` vs `ReplayId` — which for dedup, which for replay/position? Why?
- [ ] Given at-least-once delivery, what's your idempotency strategy at the subscriber?

### E. Allocations & limits (know the numbers)
- [ ] Daily publish allocation? Delivery allocation (Pub/Sub / CometD / empApi)? Are publish and delivery counted separately?
- [ ] Max event message size?
- [ ] Concurrency limits on subscriptions / clients?
- [ ] How do add-on event allocations / usage-based entitlements factor in?

### F. Pub/Sub API
- [ ] Why gRPC + Avro schema fetch vs plain REST? What do you gain?
- [ ] Managed vs unmanaged subscription — who tracks the replay position, and the trade-off?
- [ ] Flow control: how does the client govern how many events it receives?

### G. Testing
- [ ] `Test.getEventBus().deliver()` — what does it force, and why can't you test subscribers without it?
- [ ] How do you test a *retried* event and assert on the retry count/behavior?

### H. Decision boundaries (the architect questions)
- [ ] Platform Events vs **Change Data Capture** — the one discriminator.
- [ ] Platform Events vs **Streaming API / PushTopic** — the one discriminator.
- [ ] Platform Events vs **Outbound Messages / callouts** for external integration.
- [ ] When is a platform event the *wrong* tool?

---

## Scenario drills — design cold, then mine the doc for constraints

Write a full design for each *before* reading. Then read Considerations + Allocations and annotate what your design got wrong (usually a limit or a delivery guarantee).

1. **High-throughput ingestion:** external system fires 5M events/day into Salesforce; a trigger updates records. Sizing, allocations, batch/retry, idempotency, failure handling?
2. **Ordered financial updates:** downstream must apply updates in order and never double-apply. What does the platform guarantee, and what must *you* build?
3. **Long-outage recovery:** a Pub/Sub subscriber is down 5 days, then reconnects. What's recoverable, what's lost, and how do you design around retention?
4. **Fan-out:** one event, five independent subscribers with different SLAs and one that's flaky. Isolation, retries, and blast-radius control?

---

## Cadence

Not "read 143 pages." Instead: **one topic (A–H) per session** — blank-page it, verify against its pages, log only the deltas. ~8 focused sessions to convert the whole doc into your Decision Sheet, plus spaced re-attempts of the ❌ items. You'll spend most time *writing from memory and checking*, very little *reading* — which is the point.
