---
layout: post
title: "Race Conditions: When Timing Breaks Web Application Logic"
date: 2026-07-10 12:00:00 +0000
categories: web-security race-conditions
---

Most web vulnerabilities are about *what* you send. Race conditions are about *when*
you send it. They live in the tiny gap between an application checking a fact and
acting on it — and in that gap, carefully timed requests can make an app do things
its own rules were supposed to prevent. This post is a defender-oriented look at how
timing breaks logic and how to close those gaps.

> ⚖️ Everything here is general, educational material. Any real testing should only
> ever be performed against systems you are explicitly authorized to test, and
> strictly within a program's defined scope.

## What a Race Condition Is

A race condition occurs when the correctness of a system depends on the sequence or
timing of events that are not properly controlled. In web applications, that usually
means two or more requests are processed concurrently and interfere with each other
in a way a single request never could.

The classic shape is **time-of-check to time-of-use (TOCTOU)**: the application
checks a condition, then performs an action based on that check — but the state can
change in between. If several requests all pass the check *before any of them
performs the action*, the invariant the check was protecting is quietly violated.

```text
Request A:  read balance (100)  ----+
Request B:  read balance (100)  --+ |
                                  | |  both saw 100, both proceed
Request A:  deduct 100 -> 0       | |
Request B:  deduct 100 -> ?       +-+  second deduction should never have happened
```

## A Simple Mental Model: the Window

Every state-changing operation has a **window** — the interval between reading state
and committing the result. Under normal, sequential traffic the window is invisible.
Send enough requests close enough together, and multiple executions can overlap
inside that window, each one believing it is acting alone.

The wider the window and the weaker the locking, the easier the collision. Operations
that read, decide, and write across several steps (especially across multiple service
calls) tend to have the widest windows.

## Where Timing Breaks Logic

Race conditions matter most where a rule is meant to be enforced *exactly once* or
*up to a limit*. A few common patterns:

### Limit and Quota Bypass

Anywhere the app enforces "you may do this N times," concurrency can push past N. If
a coupon is single-use, a check limits one action per account, or an invite grants a
fixed number of seats, overlapping requests can each pass the "under the limit" check
before any of them records its usage.

```http
POST /apply-coupon    HTTP/1.1
{ "code": "WELCOME" }

# The same request fired many times in parallel. If each one validates the
# coupon before any of them marks it consumed, the discount can apply repeatedly.
```

### Balance and "Double Spend"

Financial and points-like features are the textbook case. Two withdrawals or
transfers that both read the same starting balance can both be approved, letting the
total effect exceed what the balance should allow. The ledger ends up inconsistent
because two operations shared one snapshot of the truth.

### One-Time Redemptions

Gift codes, referral bonuses, email-verification tokens, and "claim your reward"
flows are supposed to be redeemable once. If redemption is not atomic, several
concurrent claims can each succeed against the same token before it is marked used.

### State Confusion Across Steps

Some races are not about counting at all. Firing an action while an object is
mid-transition — for example, changing a resource's owner while another request
reads it — can produce a state the application never intended, occasionally exposing
data or capabilities across a boundary.

## Why These Are Hard to Spot

Race conditions are invisible to most testing because functional tests are
sequential: send one request, check one response, move on. The bug only appears
under **overlap**, which normal test suites rarely create. They can also be
intermittent — the same set of requests may collide one time and not the next — so a
non-deterministic result is itself a signal worth investigating rather than
dismissing.

Conceptually, exercising a suspected race means dispatching multiple equivalent
requests so they arrive as close together as possible, minimizing the timing jitter
between them, and then inspecting whether an invariant (a balance, a count, a
one-time flag) was violated. The goal of a responsible tester is only to
*demonstrate* that the invariant can break, never to repeatedly exploit it.

## Why Naive Defenses Fail

Teams often add controls that look sufficient but do not address concurrency:

- **Check-then-act in application code.** Reading a value, comparing it, and writing
  a new value in separate steps is the very pattern that races exploit. Without a
  lock or atomic operation, two threads interleave freely.
- **Rate limiting.** Throttling is about *volume over time*, not *simultaneity*. A
  small burst that still fits under the limit can be enough to collide.
- **Client-side guards.** Disabling a button after one click does nothing against
  requests crafted outside the browser.
- **"It's fast enough."** A narrow window is still a window. Assuming requests never
  overlap is an assumption an attacker gets to test.

## Building Real Defenses

The reliable fixes push correctness down to a layer that can enforce it atomically:

1. **Atomic database operations.** Let the datastore do the check and the change in
   one statement, so no two operations can share a stale snapshot.

   ```sql
   -- Only succeeds if the code has not been consumed yet; the DB serializes this.
   UPDATE coupons
   SET consumed = TRUE
   WHERE code = ? AND consumed = FALSE;
   -- Then act only if exactly one row was affected.
   ```

2. **Database constraints.** Unique constraints and conditional updates make illegal
   states impossible to persist, even if application logic slips. A `UNIQUE` index on
   "one redemption per user per code" turns a race into a rejected duplicate.

3. **Locking.** Row-level locks (for example, `SELECT ... FOR UPDATE`) or
   application-level locks serialize access to the contested resource so the window
   closes.

4. **Idempotency keys.** Have clients attach a unique key per logical operation and
   have the server record it, so a repeated or duplicated request is recognized and
   collapsed into a single effect.

5. **Transactions with the right isolation.** Wrap multi-step logic in a transaction
   at an isolation level appropriate to the invariant, so concurrent transactions
   cannot both commit conflicting results.

The unifying principle: never let a decision and the action based on it drift apart
across an unprotected gap. Collapse "check" and "act" into a single atomic step, or
guard them with a lock, and the timing advantage disappears.

## Testing Responsibly

When investigating race conditions inside an authorized program:

- Stay strictly **in scope** and respect any rules on automated or high-volume
  traffic.
- Use your **own test accounts and disposable artifacts** (test coupons, test
  balances) so no real users or funds are affected.
- Stop at **proof**: demonstrating a single invariant violation is enough. Do not
  repeat the effect to accumulate value.
- Capture clear evidence — the concurrent requests, the before/after state, and
  timestamps — and report with a concrete, atomicity-focused remediation.

## Closing

Race conditions are a reminder that a rule is only as strong as the moment it is
enforced; if the check and the action can be pried apart, timing becomes the exploit,
and the fix is to make that gap impossible to slip through.

*More writeups on CORS misconfiguration and access control coming soon.*
