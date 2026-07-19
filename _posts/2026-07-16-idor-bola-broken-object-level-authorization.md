---
layout: post
title: "IDOR & BOLA: Breaking Object-Level Authorization"
date: 2026-07-16 12:00:00 +0000
categories: web-security access-control
---

Insecure Direct Object Reference (IDOR) — known in API circles as Broken Object-Level
Authorization (BOLA) — is one of the most common and highest-impact classes of access
control flaw. It's conceptually simple, which is exactly why it shows up everywhere:
an application exposes a reference to a data object but forgets to confirm that the
current user is actually allowed to touch it. This post is a defender-oriented look at
why these bugs persist and how to reason about them methodically.

> ⚖️ Everything here is general, educational material. Any real testing should only
> ever be performed against systems you are explicitly authorized to test and that
> fall within a program's defined scope.

## IDOR and BOLA: The Same Root Cause

The two terms describe the same underlying problem from different eras. "IDOR" comes
from classic web application testing; "BOLA" is the name used in API security
guidance, where it consistently ranks as the number-one API risk. Both mean:

> The application uses a client-supplied identifier to locate an object, but never
> verifies that the authenticated caller is authorized for *that specific object*.

Authentication answers "who are you?" Object-level authorization answers "are you
allowed to access *this* thing?" IDOR/BOLA is what happens when the first question is
enforced and the second is skipped.

## Anatomy of the Vulnerability

Consider an endpoint that returns an invoice:

```http
GET /api/v1/invoices/49281 HTTP/1.1
Host: app.example.com
Authorization: Bearer <valid-session-token>
```

The request is authenticated — the token is valid. The vulnerability appears if the
server fetches invoice `49281` and returns it *without* checking that the invoice
belongs to the caller. Change the number to `49282` and, if another customer's
invoice comes back, you have a textbook IDOR.

The fix conceptually is one line of logic: the query should be scoped to the owner —
"fetch invoice 49282 **where owner = current_user**" — so an object that isn't yours
simply isn't found.

## Common Object Reference Patterns

### Sequential integers

The easiest to spot. IDs like `1024`, `1025`, `1026` invite enumeration and make the
impact obvious: an attacker can walk the entire range.

### UUIDs and "unguessable" IDs

Random identifiers such as `9f1c2a7e-...` raise the bar for *guessing*, but they do
not provide authorization. UUIDs leak through search results, referrer headers, shared
links, logs, and other endpoints. Treating an unguessable ID as a secret is
**security by obscurity** — if the object comes back without an ownership check, it's
still BOLA.

### Encoded or hashed references

Identifiers wrapped in Base64, or a predictable hash, only add a decoding step:

```text
b3JkZXItMTAwMg==   ->   order-1002
```

Decoding, incrementing, and re-encoding is trivial. Encoding is not authorization
either.

## Beyond Simple ID Swaps

The most interesting object-level flaws hide beyond the obvious `GET` with an integer.

### HTTP method changes

Read access might be authorized correctly while `PUT`, `PATCH`, or `DELETE` on the
same object is not:

```http
DELETE /api/v1/documents/8842 HTTP/1.1
Host: app.example.com
Authorization: Bearer <valid-session-token>
```

Always consider each method independently — a resource can be safe to read but unsafe
to modify.

### Parameter pollution and arrays

APIs that accept a single ID may behave differently when handed several. Supplying an
array or duplicated parameter can bypass a check that only validates the first value:

```json
{ "userId": ["me", "victim"] }
```

### Nested and secondary identifiers

Ownership is sometimes checked on the primary object but not on a **nested** one. A
request scoped to your own account may still accept someone else's object id in a
sub-field:

```http
POST /api/v1/teams/1002/members HTTP/1.1
Content-Type: application/json

{ "invitedUserId": 5567, "documentId": 90114 }
```

Here `documentId` might reference a document you don't own, quietly attaching it to a
context you *do* control.

## A Methodical Testing Approach

Object-level authorization testing is fundamentally a **two-account exercise**:

1. **Create two accounts** you control — call them Account A and Account B.
2. **Generate objects** in Account B (orders, files, messages) and record their
   identifiers.
3. **Authenticate as Account A** and replay B's requests, substituting B's object
   identifiers while keeping A's session.
4. **Inspect the response.** If A receives B's data — or successfully modifies it —
   object-level authorization is broken.

Work through every object type and every method the application exposes, not just the
first endpoint you find. A clean matrix — object type, identifier, method, expected
result, actual result — makes coverage and evidence far easier.

## Why It Persists

If the fix is essentially one ownership check, why is IDOR still everywhere?

- **Distributed logic.** Authorization is spread across many handlers; one new
  endpoint that forgets the check reopens the door.
- **Framework defaults.** Many ORMs make "find by id" effortless and "find by id
  scoped to owner" a deliberate extra step.
- **Trust in obscurity.** Teams assume random or encoded IDs are enough.
- **Microservice hand-offs.** An internal service may trust an upstream identifier
  that was never authorization-checked at the boundary.

## Defensive Patterns

- **Enforce ownership at the data layer.** Scope every lookup to the authenticated
  principal so unauthorized objects are simply not returned.
- **Centralize authorization.** A shared, testable checkpoint beats per-handler logic
  that's easy to forget.
- **Check every method and every nested reference**, not just top-level reads.
- **Don't rely on unguessable IDs** for security; use them for uniqueness, not
  access control.
- **Add automated tests** that assert Account A cannot reach Account B's objects, and
  run them in CI so regressions are caught early.

## Testing Responsibly

- Use **only accounts you own** to demonstrate cross-user access — never pivot into a
  real customer's data.
- Prove the issue with the **smallest possible** example (one object, one swap).
- Record request/response pairs and timestamps as clean evidence.
- Report with a concrete remediation and, where possible, the affected object types.

## Closing

IDOR and BOLA endure because authentication is easy to remember and per-object
authorization is easy to forget — which is why the durable fix is to make ownership
checks a default of the data layer rather than an afterthought in each handler.
