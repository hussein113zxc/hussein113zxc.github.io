---
layout: post
title: "Cross-Site Scripting (XSS): A Practical Guide to Reflected, Stored, and DOM"
date: 2026-07-11 12:00:00 +0000
categories: web-security xss
---

Cross-Site Scripting (XSS) is one of the oldest and most stubborn vulnerability
classes on the web, and it still shows up constantly in modern applications. This
post is a practical, defender-oriented walkthrough of what XSS actually is, the
three main variants you will meet, and why the "obvious" fixes so often fall short.

> ⚖️ Everything here is general, educational material. Any real testing should only
> ever be performed against systems you are explicitly authorized to test, and
> strictly within a program's defined scope.

## What XSS Really Is

At its core, XSS is a failure to keep **data** and **code** apart. An application
takes input from somewhere — a query string, a form field, a stored comment — and
places it into an HTML page without neutralizing it. The browser then has no way to
tell that a chunk of text was meant to be *displayed* rather than *executed*, so it
runs any script it finds.

The consequence is that an attacker's JavaScript executes in the victim's browser,
inside the victim's session, on the trusted origin. That means it can read the DOM,
make authenticated requests, exfiltrate session data reachable to script, and
generally act as the user. The impact scales with what the application lets a logged
in user do.

## The Three Faces of XSS

XSS is usually split into three families based on *where the untrusted data lives*
and *how it reaches the page*.

### Reflected XSS

In reflected XSS, the payload travels in the request and is immediately echoed back
in the response. Nothing is stored server-side. Consider a search page that prints
the user's query:

```html
<!-- Server takes ?q= and drops it straight into the page -->
<p>You searched for: Aizen Sosuke shoes</p>
```

If the server does not encode that value, a crafted query becomes live markup:

```
https://example.test/search?q=<script>alert(document.domain)</script>
```

```html
<p>You searched for: <script>alert(document.domain)</script></p>
```

Because it needs the victim to follow a crafted link, reflected XSS is typically
delivered via a URL. It is still serious — a single click can be enough.

### Stored XSS

Stored (or persistent) XSS is the more dangerous cousin. The payload is saved by the
application — in a comment, profile field, support ticket, or product review — and
served to everyone who later views that content.

```json
POST /api/comments
{ "body": "<script>/* runs for every visitor of this thread */</script>" }
```

No lure is required. Any user who loads the affected page executes the script. When
stored XSS lands somewhere high-traffic, like a shared feed or an admin dashboard, a
single injection can reach a large population, including privileged staff.

### DOM-Based XSS

DOM-based XSS never needs the server to reflect anything. The vulnerability lives
entirely in client-side JavaScript that reads attacker-controllable input (a
**source**) and writes it into a dangerous **sink** without sanitization:

```javascript
// Source: the URL fragment. Sink: innerHTML.
const name = decodeURIComponent(location.hash.slice(1));
document.getElementById("greeting").innerHTML = "Hello, " + name;
```

A fragment like `#<img src=x onerror=alert(1)>` is written directly into the DOM and
executes. Because the payload can sit in the fragment (after the `#`), it may never
even reach the server logs — which makes DOM XSS easy to miss in server-side
reviews.

## Sources, Sinks, and Why They Matter

Thinking in terms of sources and sinks is the fastest way to reason about XSS.

- **Sources** are anywhere untrusted data enters: `location`, `document.referrer`,
  `window.name`, `postMessage` data, query and form values, stored records.
- **Sinks** are anywhere that data can become code: `innerHTML`, `outerHTML`,
  `document.write`, `eval`, `setAttribute` for event handlers, `src`/`href` on
  certain elements, and framework equivalents like `dangerouslySetInnerHTML`.

A bug exists when a source can flow to a sink without safe handling in between. This
model also explains why the fix belongs at the sink: that is the last place you
control before the browser interprets the value.

## Why Naive Filters Fail

The instinct is to "filter out `<script>`." Attackers have decades of practice
defeating that. A few reasons blocklists lose:

- **Event handlers need no script tag:** `<img src=x onerror=alert(1)>` or
  `<svg onload=alert(1)>` execute without ever writing `<script>`.
- **Encoding and case tricks:** mixed case, HTML entities, and URL encoding slip
  past exact-string matching.
- **Context confusion:** a value that is safe inside visible text can be dangerous
  inside an attribute, a URL, or a `<script>` block.

Stripping keywords produces brittle rules that break legitimate input while still
leaving bypasses. Escaping and correct output handling are what actually work.

## Context Is Everything

The single most important idea in XSS defense is that **the right encoding depends on
where the value is placed.** The same string needs different treatment in each of
these positions:

```html
<!-- HTML body context: encode < > & " ' -->
<div>USER_VALUE</div>

<!-- Attribute context: encode and always quote the attribute -->
<input value="USER_VALUE">

<!-- JavaScript context: this is hostile ground; avoid it -->
<script>var x = "USER_VALUE";</script>

<!-- URL context: validate the scheme, reject javascript: -->
<a href="USER_VALUE">link</a>
```

HTML-encoding a value is correct for the body context but does not make it safe
inside a `javascript:` URL or an inline script. Injecting untrusted data into a
script context is best avoided entirely; pass it as data (for example via a
`data-` attribute the script reads) instead of concatenating it into code.

## Building Real Defenses

A layered approach holds up far better than any single control:

1. **Context-aware output encoding.** Encode at the point of output, matched to the
   context. Let your templating engine do it and confirm auto-escaping is on.
2. **Prefer safe sinks.** Use `textContent` instead of `innerHTML` when inserting
   text. In frameworks, avoid the "raw HTML" escape hatches unless you sanitize.
3. **Sanitize rich HTML with a vetted library.** If users must submit formatted
   HTML, run it through a maintained sanitizer with a strict allowlist rather than
   hand-rolled regex.
4. **Content Security Policy (CSP).** A strong CSP — ideally nonce-based, without
   `unsafe-inline` — turns many injections into non-events by refusing to run
   inline or unauthorized scripts. Treat it as defense-in-depth, not a primary fix.
5. **Protect session cookies.** Set `HttpOnly` so script cannot read them, plus
   `Secure` and a sensible `SameSite`. This limits what a successful XSS can steal.
6. **Validate input on the way in.** Type, length, and format checks reduce the
   surface, though they never replace output encoding.

```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-r4nd0m';
  object-src 'none'; base-uri 'none'
```

## Testing Responsibly

When looking for XSS inside an authorized program:

- Stay strictly **in scope** and honor the program's rules.
- Use a **harmless proof of concept** — something like `alert(document.domain)` or a
  benign console log is enough to prove execution. Never deploy payloads that steal
  real user data or persist beyond your own test account.
- For stored XSS, prefer a value that only your own test view renders, so you do not
  affect other users.
- Capture clean evidence (request, response, and where it executes) with timestamps,
  and report with a concrete, context-aware remediation.

## Closing

XSS endures because it is a boundary problem — data crossing into code — and that
boundary appears in dozens of small places across an app; the fix is to encode for
the exact context, every time, and back it with CSP and hardened cookies.

*More writeups on race conditions and CORS misconfiguration coming soon.*
