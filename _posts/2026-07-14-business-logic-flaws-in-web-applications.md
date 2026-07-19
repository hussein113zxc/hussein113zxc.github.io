---
layout: post
title: "Hunting Business Logic Flaws in Web Applications"
date: 2026-07-14 12:00:00 +0000
categories: web-security business-logic
---

Business logic flaws are the vulnerabilities that scanners almost never find. There
is no malformed payload, no injection, no obviously "bad" character — just a valid
request that the application was never designed to handle. Because the traffic looks
legitimate, these bugs slip past automated tooling and land squarely in the domain of
the patient human tester. This post is a defender-oriented walkthrough of how logic
flaws arise, the categories worth knowing, and a repeatable way to reason about them.

> Everything here is general, educational material intended to help builders and
> testers reason about their own systems. Any real testing must only ever be
> performed against targets you are explicitly authorized and in-scope to test.

## What Makes Logic Flaws Different

Most vulnerability classes have a signature. SQL injection has quote characters and
error strings; XSS has script tags; SSRF has outbound requests. Business logic flaws
have none of that. They are failures of *intended behavior* — the code does exactly
what it was written to do, but the rules it enforces don't match the rules the
business actually needs.

That distinction matters. To find a logic flaw you first have to understand what the
feature is *for*: what a legitimate user is supposed to be able to do, in what order,
under what constraints. Only then can you spot the sequence of valid-looking requests
that quietly violates one of those assumptions.

## Common Categories of Business Logic Flaws

### Price and Parameter Manipulation

Whenever a value that should be authoritative is accepted from the client, trouble
follows. A classic pattern is a checkout endpoint that trusts a submitted price or
total instead of recomputing it server-side:

```http
POST /api/cart/checkout HTTP/1.1
Host: shop.example.com
Content-Type: application/json

{
  "item_id": "SKU-1042",
  "quantity": 1,
  "unit_price": 199.00
}
```

If the server bills `unit_price` as sent rather than looking it up from its own
catalog, an attacker simply changes the number. The same idea applies to currency
fields, tax flags, shipping tiers, and "membership level" parameters that unlock
pricing.

### Workflow and Step Sequencing Bypass

Multi-step flows — registration, checkout, KYC, password reset — usually assume the
steps happen in order. If step 3 doesn't verify that steps 1 and 2 actually completed,
a tester can jump straight to it:

```http
POST /api/onboarding/step3-activate HTTP/1.1
Host: app.example.com
Content-Type: application/json

{ "account_id": "9f2a", "plan": "enterprise" }
```

When each step trusts that the previous one enforced its checks, skipping ahead can
bypass payment, identity verification, or approval gates entirely. State should be
validated on the server at every transition, not inferred from the fact that a request
arrived.

### Over-Trusting Client-Controlled State

Hidden form fields, cookies, JWT claims, and mobile-app parameters are all
client-controlled, no matter how "internal" they look. A field like
`"role": "user"` or `"is_verified": false` is an invitation to flip it. The rule is
simple: any security decision must be re-derived from trusted server-side state
(the authenticated session, the database), never read back from data the client can
edit.

### Quantity, Limits, and Numeric Edge Cases

Logic that assumes "reasonable" input often breaks at the boundaries. Negative
quantities can create credits instead of charges; extremely large values can overflow
or trigger integer wraparound; zero can bypass a "must purchase at least one" rule.

```http
POST /api/cart/add HTTP/1.1
Host: shop.example.com
Content-Type: application/json

{ "item_id": "SKU-1042", "quantity": -3 }
```

If adding a negative quantity subtracts from the order total, the checkout becomes a
refund machine. Always test the minimum, maximum, zero, and negative cases explicitly.

### Discount, Coupon, and Referral Abuse

Promotional systems are dense with implicit rules: one coupon per order, one signup
bonus per person, referrals only for genuinely new users. Each unstated rule is a
candidate flaw. Can two coupons stack? Can the same code be replayed? Can a referral
loop pay a user for inviting themselves through disposable email addresses? These bugs
have direct financial impact, which is exactly why programs care about them.

## Race Conditions Amplify Logic Flaws

Many logic checks are written as read-then-write: read the current balance or
remaining uses, decide, then update. If two requests interleave between the read and
the write, both can pass the same check. That is how a single gift card gets redeemed
twice or a one-time discount applies several times. Sending a small burst of identical
requests in parallel is a standard way to probe for this class — and the fix is
server-side atomicity (locks, transactions, or conditional updates), not faster
validation.

## A Methodology for Finding Logic Flaws

A structured approach beats random tampering:

1. **Map the intended flow.** Walk the feature as a normal user first. Write down the
   steps, the values that move between them, and the rules you think are being
   enforced.
2. **Identify trust boundaries.** For each value, ask: does the server recompute this,
   or trust what I send? Client-trusted values are your primary hunting ground.
3. **Question every assumption.** What if this step is skipped? Repeated? Done out of
   order? What if the number is negative, zero, or enormous?
4. **Test one variable at a time.** Change a single thing per request so you can
   attribute any change in behavior precisely.
5. **Confirm real impact.** A changed response is interesting; a changed *outcome* —
   a lower charge, an unlocked tier, a bypassed gate — is the finding.

## Defensive Guidance

For teams building these systems, a few principles prevent most logic flaws:

- **Treat all client input as untrusted**, including hidden fields, headers, and
  token claims. Re-derive prices, roles, and entitlements server-side.
- **Enforce state machines on the server.** Each step should verify that the
  prerequisites genuinely completed for *this* user and session.
- **Make critical operations atomic.** Use database transactions or conditional
  updates so concurrent requests can't both pass a one-time check.
- **Validate numeric ranges** explicitly, rejecting negatives and absurd magnitudes
  where they make no sense.
- **Write abuse cases alongside test cases.** For every rule ("one per customer"),
  add a test that tries to break it.

## Testing Responsibly

Logic flaws often touch money, accounts, and orders, so restraint matters even more
than usual:

- Stay strictly **in scope** and within the program's rules.
- Use **test accounts and minimal proof-of-concepts** — demonstrate that a discount
  *could* be doubled without actually draining a real system.
- Never manipulate other users' data or real financial records to prove a point.
- Capture clean request/response evidence and describe the business impact plainly,
  along with a concrete remediation.

## Conclusion

Business logic flaws reward the tester who understands the application better than the
next automated tool ever could — find the rule nobody wrote down, and you've found the
bug nobody else will.
