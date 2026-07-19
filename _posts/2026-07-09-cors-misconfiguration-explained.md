---
layout: post
title: "CORS Misconfiguration: How a Header Becomes a Vulnerability"
date: 2026-07-09 12:00:00 +0000
categories: web-security cors
---

Cross-Origin Resource Sharing (CORS) is one of those mechanisms that quietly sits in
the response headers and is easy to get subtly wrong. A single reflected value or one
`true` in the wrong place can turn a protective browser feature into a data-leak
channel. This post is a defender-oriented walkthrough of what CORS does, how
misconfigurations arise, and how to configure it safely.

> ⚖️ Everything here is general, educational material. Any real testing should only
> ever be performed against systems you are explicitly authorized to test, and
> strictly within a program's defined scope.

## The Same-Origin Policy, Briefly

The browser's **Same-Origin Policy (SOP)** is a foundational security boundary.
Scripts running on one origin — the tuple of scheme, host, and port — can send
requests to another origin, but by default they cannot *read* the responses. That is
what stops a random site from quietly reading your authenticated data from another
application in your browser.

CORS exists because that boundary is sometimes too strict. Legitimate applications
often need a front-end on one origin to call an API on another. CORS is the
controlled, opt-in way for a server to say "these specific other origins are allowed
to read my responses."

## What CORS Actually Does

CORS is a **server-driven relaxation** of the SOP. The browser advertises the calling
origin, and the server answers with headers describing what is permitted. The browser
then enforces the result.

For requests that are not "simple," the browser first sends a **preflight** — an
`OPTIONS` request — asking whether the real request is allowed:

```http
OPTIONS /api/account HTTP/1.1
Origin: https://app.example.test
Access-Control-Request-Method: GET
```

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.test
Access-Control-Allow-Methods: GET, POST
```

The header that does the heavy lifting is **`Access-Control-Allow-Origin` (ACAO)**.
It names which origin may read the response. Get its value wrong and you have handed
out read access you never intended.

## How a Header Becomes a Vulnerability

The interesting failures are not "CORS is on." They are subtle mistakes in *how the
allowed origin is decided.*

### Reflecting the Origin

The most common dangerous pattern is a server that takes whatever `Origin` the
request carried and echoes it straight back:

```http
# Request from an arbitrary site
Origin: https://attacker-controlled.test
```

```http
# Server reflects it verbatim
Access-Control-Allow-Origin: https://attacker-controlled.test
Access-Control-Allow-Credentials: true
```

This effectively allowlists *everyone*, because every origin sees its own value
reflected. Reflection usually happens when a developer wants to "support many
front-ends" and takes a shortcut instead of maintaining a real allowlist.

### Trusting the `null` Origin

Some servers explicitly allow the literal value `null`:

```http
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
```

The `null` origin is not a safe sentinel. It is produced by several ordinary
situations — sandboxed iframes, some redirects, and documents loaded from certain
non-standard contexts — which means "allow null" can be reachable by content the
application does not control. Treating `null` as trusted is a frequent misstep.

### Sloppy Host Matching

When a team does try to build an allowlist, weak string matching reopens the door:

- **Suffix checks** like "does the origin end with `example.test`?" also match
  `notexample.test` or `example.test.attacker.test`.
- **Prefix or `startsWith` checks** match `example.test.attacker.test` when they only
  meant to match the real domain.
- **Naive wildcards** for "any subdomain" can be satisfied by an attacker-registered
  look-alike or by a subdomain the org does not fully control.

Each of these is an origin-parsing bug wearing an allowlist costume.

## The Credentials Amplifier

CORS misconfiguration becomes genuinely serious when it is paired with credentials.
The header **`Access-Control-Allow-Credentials: true`** tells the browser it may send
cookies (and read the response) on cross-origin requests. That is what allows a
malicious page to make requests *as the logged-in victim* and read the answer.

There is one important guardrail built into browsers: the wildcard and credentials
**cannot be combined.** A response of `Access-Control-Allow-Origin: *` together with
`Access-Control-Allow-Credentials: true` is rejected by the browser. This is exactly
why **reflecting the origin** is the pattern attackers look for — it is the way to
get a *specific* allowed origin (so credentials are permitted) while still effectively
allowing anyone.

```http
# Rejected by the browser — wildcard cannot carry credentials:
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true

# Accepted, and dangerous if the origin was reflected without validation:
Access-Control-Allow-Origin: https://attacker-controlled.test
Access-Control-Allow-Credentials: true
```

When an endpoint returns authenticated, user-specific data and is reachable this way,
a page on an untrusted origin can read that data out of the victim's session. That is
the difference between a cosmetic header issue and a real information-disclosure
finding.

## Why It Matters

Picture an API endpoint that returns the current user's profile, tokens, or account
details, protected only by a session cookie. If its CORS policy reflects the origin
and allows credentials, then any site the victim visits can script a request to that
endpoint, receive the authenticated response, and forward it onward. The user did
nothing but browse to the wrong page. No XSS on the target was required — the misplaced
trust was entirely in the response headers.

## Why Naive Configurations Fail

The recurring theme is that **CORS decisions are made from attacker-influenced
input.** The `Origin` header is supplied by the caller, so any logic that reflects it,
loosely matches it, or trusts special values like `null` is trusting data it should be
validating. "Allow the origins I recognize" is only as strong as the matching code
behind it, and string operations are a notoriously leaky way to compare origins.

## Building Real Defenses

A safe CORS posture comes down to a few firm rules:

1. **Use a strict allowlist of full origins.** Compare the incoming `Origin` against
   an exact set of known-good values (scheme, host, and port), not substrings.

   ```text
   allowed = { "https://app.example.test", "https://admin.example.test" }
   if request.origin in allowed:
       set Access-Control-Allow-Origin: <that exact origin>
   else:
       omit the header entirely
   ```

2. **Never reflect the origin unchecked**, and never treat `null` as trusted.
3. **Keep wildcards and credentials apart.** Only send
   `Access-Control-Allow-Credentials: true` for a vetted, specific origin — never
   alongside `*`, and never for endpoints that do not truly need cross-origin
   credentialed access.
4. **Scope the relaxation.** Do not apply a permissive policy blanket-wide; limit it
   to the endpoints that genuinely require it, and constrain allowed methods and
   headers.
5. **Send `Vary: Origin`.** When the allowed origin depends on the request, this
   prevents a shared cache from serving one origin's permissive response to another.
6. **Do not rely on CORS as access control.** CORS governs cross-origin *reads* in
   the browser; sensitive endpoints still need real authentication and authorization
   on the server.

## Testing Responsibly

When reviewing CORS behavior inside an authorized program:

- Stay strictly **in scope** and use your **own test accounts**.
- Confirm the misconfiguration with a **benign check** — for example, observing that
  an arbitrary `Origin` is reflected and that credentials are permitted — without
  exfiltrating another person's data.
- Capture the request and response headers as clean evidence, with timestamps.
- Report clearly, and pair the finding with a concrete allowlist-based remediation.

## Closing

CORS misconfigurations are a lesson in trusting the wrong input: the `Origin` header
is attacker-influenced, so the only safe policy is to match it against an exact
allowlist and keep credentials far away from anything permissive.

*More writeups on authentication and access-control flaws coming soon.*
