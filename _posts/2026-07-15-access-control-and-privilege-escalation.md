---
layout: post
title: "Access Control Testing: Horizontal and Vertical Privilege Escalation"
date: 2026-07-15 12:00:00 +0000
categories: web-security access-control
---

Broken access control is consistently one of the most impactful categories in web
application security. It's rarely a single exotic bug — more often it's a pattern of
small enforcement gaps that let a user reach data or actions meant for someone else.
This post is a defender-oriented framework for thinking about access control, with a
focus on the two escalation directions every tester should keep in mind: **horizontal**
and **vertical**.

> ⚖️ Everything here is general, educational material. Any real testing should only
> ever be performed against systems you are explicitly authorized to test and that
> fall within a program's defined scope.

## What Access Control Really Enforces

Access control is the layer that decides *what an authenticated identity is allowed to
do*. It sits on top of authentication (proving who you are) and depends on it, but the
two fail independently: a perfectly authenticated user can still be handed powers they
should never have. Sound access control answers three questions on **every** request:

- **Who** is making this request?
- **What** resource or function are they trying to reach?
- **Are they permitted** to reach it, right now, in this state?

When the third question is skipped, assumed, or enforced only in the interface, the
door opens.

## Horizontal vs. Vertical Escalation

### Horizontal escalation

Moving *sideways* to another account at the **same** privilege level — one standard
user reading or modifying another standard user's data. This is the access control
framing of the object-level flaws (IDOR/BOLA) that plague per-user resources like
profiles, orders, and messages.

### Vertical escalation

Moving *upward* to a **higher** privilege level — a regular user performing
administrative or moderator actions. Vertical escalation usually targets
**functions** (the API equivalent is Broken Function-Level Authorization) rather than
individual records: reaching an admin endpoint, invoking a privileged mutation, or
flipping a role.

Keeping these two axes explicit is useful because they demand different test data:
horizontal testing needs two same-level accounts, while vertical testing needs a
low-privilege account probing high-privilege functionality.

## Where Access Control Breaks Down

### Function-level access and forced browsing

A classic vertical flaw is an admin function that hides its link from non-admins but
never actually enforces the restriction server-side:

```http
GET /admin/users/export HTTP/1.1
Host: app.example.com
Cookie: session=<standard-user-session>
```

If a standard user's session receives the export, the control was **presentation, not
enforcement**. The same applies to "hidden" API routes discovered in JavaScript
bundles or documentation.

### Client-side enforcement only

Interfaces frequently disable buttons or hide menus based on role. That improves UX,
but attackers don't use the interface — they send requests directly. Any check that
lives solely in the browser can be bypassed by replaying the underlying request:

```http
POST /api/v1/users/1200/promote HTTP/1.1
Host: app.example.com
Content-Type: application/json
Cookie: session=<standard-user-session>

{ "role": "administrator" }
```

If the server honors this without confirming the caller is already an administrator,
the "you don't see the button" defense collapses.

### Parameter- and header-based roles

Some applications trust the client to state its own privileges. Watch for role or
tenant hints that travel in requests and can be tampered with:

```http
GET /api/v1/reports HTTP/1.1
X-Account-Role: user      ->      X-Account-Role: admin
```

or a body/cookie field such as `"isAdmin": false`. Authorization must be derived from
trusted server-side session state, never from a value the client can rewrite.

### State- and step-dependent gaps

Multi-step workflows can enforce access on step one but not step three, or leave an
object editable after it should be locked. Access control is not only *who* and
*what*, but also *when* — the current state of the object matters.

## A Structured Testing Methodology

Ad-hoc poking finds shallow bugs; a **matrix** finds systematic ones. Enumerate the
sensitive functions and resources, then test each against every role you can obtain:

| Function / Resource | Anonymous | User A | User B (peer) | Admin |
|---------------------|-----------|--------|---------------|-------|
| View own profile    | deny      | allow  | allow (own)   | allow |
| View others' orders | deny      | deny   | deny          | allow |
| Admin user export   | deny      | deny   | deny          | allow |

For each cell, the actual behavior should match the intended one. The interesting
findings are the mismatches — a "deny" that returns data, or an "allow (own)" that
also serves someone else's record.

A practical workflow:

1. **Map roles and functions.** Identify every privilege tier and every sensitive
   action or resource.
2. **Obtain accounts** at each tier that you are authorized to use — ideally two peers
   plus a low-privilege account.
3. **Capture baseline requests** for each function while acting as the *intended* role.
4. **Replay across identities.** Send high-privilege requests with low-privilege
   sessions, and swap peer object identifiers between same-level accounts.
5. **Compare responses** against the matrix and record every deviation.

## Multi-Step and Context-Dependent Flaws

Some of the most valuable findings aren't a single unauthorized request but a
**sequence**. An object created in a draft state might remain modifiable after
approval; a token issued for one workflow step might be accepted at another; an
invitation flow might let a low-privilege user attach themselves to a resource they
shouldn't reach. Testing these means thinking like a state machine — asking not just
"can I call this?" but "can I call this *out of order* or *after it should be locked*?"

## Defensive Patterns

- **Deny by default.** Every resource and function starts closed and is explicitly
  opened to authorized roles.
- **Enforce on the server, every time.** UI hiding is UX, never security.
- **Derive privileges from trusted session state**, not from client-supplied roles,
  headers, or hidden fields.
- **Centralize authorization** in a shared, well-tested component so new endpoints
  inherit the checks instead of reinventing them.
- **Cover both axes and object state**: horizontal (peer data), vertical (privileged
  functions), and the *when* of stateful workflows.
- **Test continuously.** Automated checks that assert each role's boundaries catch
  regressions long before release.

## Testing Responsibly

- Test with **accounts and roles you are authorized to hold**; never escalate into a
  real user's or administrator's live data.
- Demonstrate impact with the **least intrusive** proof possible.
- Keep clean request/response evidence and timestamps.
- Pair each finding with a clear, actionable remediation.

## Closing

Access control fails quietly — one unchecked function or one peer-swappable identifier
at a time — so the strongest defense is a deny-by-default posture, enforced on the
server and verified continuously across every role.
