# Worked example: duplicate job runs fixed with a Redis lock

A generic web app ("a small invoicing app") runs a scheduled job, `send_due_invoices`, on every worker instance. After scaling from one worker to three, some customers received the same invoice email two or three times. Mode: `teach`. Notes: `on`. Language: English.

---

## 1. User prompt

> Customers are getting duplicate invoice emails since we added more workers. Fix it.

## 2. Before implementing (agent)

> Quick prediction before I start: why might more workers cause duplicates? (Answer or say go.)
>
> **Plan:** The scheduler likely fires `send_due_invoices` on all 3 workers at the same minute, and each one reads the same "due" invoices before any marks them sent (check-then-act race). I'll (1) wrap the job in a Redis lock so only one worker runs it per tick, and (2) mark each invoice as sent atomically so a retry can't resend it.

> **User:** go

The agent continues without waiting further.

## 3. During the work (agent's scratch list, not shown live)

- Confirmed: scheduler config has no "single instance" setting; logs show 3 runs at 09:00:00.
- Tried first: `if not redis.get(key): redis.set(key, 1)`. Rejected: same race (two workers can both see "not set").
- Chosen: `SET lock:send_due_invoices <token> NX PX 300000`, released with a compare-and-delete Lua script.
- Opened: Redis `SET` command docs; Redis distributed-locks pattern page.
- Verified: new test spawns 3 concurrent job runs against a test Redis → exactly 1 email per invoice. Full test suite passes.

## 4. After implementing: the walkthrough (agent, full format)

**What changed**
- `jobs/send_due_invoices.py`: the job now tries to take a lock first; if another worker holds it, it exits quietly.
- `lib/locks.py` (new): `acquire_lock()` and `release_lock()`. Release only deletes the lock if it still holds *our* token.
- `models/invoice.py`: `mark_sent()` now does a conditional update (`... WHERE id = ? AND sent_at IS NULL`) and returns whether it actually changed a row; the job only emails if it did.
- `tests/test_send_due_invoices.py`: new test running the job 3 times concurrently.

**Why:** With three workers, the scheduler started the job three times at the same moment. Each run read the same "due" list before any run marked an invoice as sent: a classic check-then-act race. Nothing made "check" and "act" a single atomic step.

**Alternatives I rejected**
- Run the scheduler on one worker only: removes the race but creates a single point of failure and a config that's easy to break later.
- `GET` then `SET` in Redis: still two steps, so the same race moves into Redis.
- An in-process mutex: works only inside one process; the workers are separate processes.
- A database advisory lock: a valid choice; Redis won because all workers already share it and the job has no DB transaction to hang it on.

**Concepts and tools**
- **Race condition (check-then-act):** two actors check the same state, both decide to act, both act.
- **Atomic operation:** `SET ... NX` checks and sets in one indivisible Redis command ([Redis `SET` docs](https://redis.io/docs/latest/commands/set/)).
- **Lock TTL and owner token:** `PX` expires the lock if a worker crashes; the token stops a slow worker from deleting someone else's lock ([Redis distributed locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)).
- **Idempotency:** the conditional `mark_sent()` makes "send invoice" safe to repeat even if the lock ever fails.

**Verified by:** `pytest tests/test_send_due_invoices.py` → 1 email per invoice across 3 concurrent runs; full suite green.

**Learn next (~20 min):** Remove the token check from `release_lock()`, make the job sleep longer than the TTL in a test, and watch a second worker get its lock deleted.

**Check yourself**
1. Why did `GET` then `SET` not fix the problem?
2. What happens if the job takes 6 minutes but the lock TTL is 5?
3. Name one other place in a typical web app where the same check-then-act bug could appear.

_Reply with your answers (short is fine), or say `answers` to see them._

## 5. User answers

> 1. because another worker can set it between my get and set
> 2. the lock expires and another worker starts the job too
> 3. not sure

## 6. Grading (agent)

1. ✅ Correct: the gap between the two commands is the race window.
2. 🟡 Partial: right, a second run starts. Also: when the first worker finishes it would delete the *new* lock without the token check; with it, the conditional `mark_sent()` still prevents duplicate emails.
3. ❌ Missing, which is fine to say. Model answer: "register user if email not taken", "reserve the last item in stock", "apply a coupon only once". Each checks state and then writes based on it. Fix with a unique constraint, a conditional update, or a lock.

**Learning state:** race condition / atomic operation: `exposure` → `assisted` (correct explain-why answer). Lock TTL: stays `exposure` (partial). Idempotency: stays `exposure` (not tested).

## 7. Learning note written

`docs/learning/2026/03/2026-03-14-duplicate-invoice-job-lock.md`:

```markdown
---
type: learning-note
date: 2026-03-14
depth: deep
concepts: [race-condition, atomic-operation, distributed-lock, idempotency]
tools: [redis, pytest]
learning_state: assisted
ai_assisted: true
---

# Duplicate invoice emails after scaling workers

## Problem and context
After scaling from 1 to 3 workers, the scheduled `send_due_invoices` job ran on all of them at once and some customers got the same invoice 2-3 times.

## Root cause
All workers read the same "due" invoices before any marked them sent (check-then-act). No step was atomic.

## What changed
- Job takes a Redis lock (`SET NX PX` + owner token) and exits if it can't.
- `mark_sent()` is a conditional update; email is sent only if it changed a row.
- Concurrency test: 3 parallel runs → 1 email per invoice.

## Approaches considered
| Approach | Result | Why |
|---|---|---|
| Scheduler on one worker | rejected | single point of failure |
| Redis GET then SET | failed | still two steps, same race |
| In-process mutex | rejected | doesn't cross processes |
| DB advisory lock | valid, not chosen | Redis already shared by all workers |
| SET NX PX + token, plus idempotent mark | chosen | atomic, crash-safe, defence in depth |

## Concepts
- [[Race Condition]]: check-then-act by two actors.
- [[Atomic Operation]]: check and write in one indivisible step.
- [[Distributed Lock]]: TTL for crashes, token for safe release.
- [[Idempotency]]: repeating the action has no extra effect.

## Verification
Concurrency test and full suite pass. Not covered: Redis failover during a run.

## Sources
- https://redis.io/docs/latest/commands/set/
- https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/

## Learn next
- Break the token check on purpose and watch the failure (see chat).

## Self-check
1. Why did GET then SET not fix it?
2. What happens if the job outlives the lock TTL?
3. Name another check-then-act bug in a typical web app.

> [!answer]- Answers
> 1. Another worker can act between the two commands; only a single atomic command closes the window.
> 2. A second worker gets the lock and starts; without an owner token the first worker would later delete the second's lock. Idempotent `mark_sent()` still prevents duplicate emails.
> 3. Unique signup email, last-item stock reservation, single-use coupons.
```

Index line appended to `docs/learning/index.md`:

```
- 2026-03-14 [[2026-03-14-duplicate-invoice-job-lock]] — duplicate invoice job fixed with Redis lock + idempotent mark — concepts: race-condition, atomic-operation, distributed-lock, idempotency — state: assisted
```

---

## Same task in `brief` mode (compact format, notes off)

> **Plan:** All 3 workers run the invoice job at once and read the same due list (check-then-act race). Adding a Redis lock plus an idempotent "mark sent".
>
> *(work happens)*
>
> **What changed**
> - `jobs/send_due_invoices.py`: takes a Redis lock first; other workers skip.
> - `lib/locks.py`: lock helpers with TTL and owner token.
> - `models/invoice.py`: email is sent only if "mark sent" actually updated the row.
>
> **Why:** the check ("is it due?") and the act ("send + mark") weren't atomic, so 3 workers all passed the check.
>
> **Concept:** **Atomic operation**: `SET NX` checks and sets in one step, so only one worker wins.
>
> **Verified by:** `pytest tests/test_send_due_invoices.py` → 1 email per invoice across 3 concurrent runs.
>
> **Check yourself:**
> 1. Why wouldn't `GET` then `SET` have fixed this?
>
> _Reply with your answer, or say `answers`._

## Same task in `drill` mode (target concept: atomic lock acquisition)

> **Plan:** same as above. I'll wire the job, the conditional `mark_sent()`, and a failing concurrency test. **You** write `acquire_lock(key, ttl_ms)` in `lib/locks.py`.
>
> Done = `pytest tests/test_locks.py` passes (3 concurrent callers → exactly one gets `True`).
>
> Hints on request: **L1** look up the options of Redis `SET`. **L2** one command can set a key only if it doesn't exist and give it an expiry. **L3** pseudocode: `set key = token, only if absent, expire after ttl_ms; return whether it was set`.
