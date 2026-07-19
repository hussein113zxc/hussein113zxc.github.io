---
layout: post
title: "OAuth Security: Common Pitfalls and How to Test Them"
date: 2026-07-12 12:00:00 +0000
categories: web-security authentication
---

OAuth 2.0 powers "Log in with…" buttons and countless API integrations, but it is a
*delegation* framework, not a drop-in authentication solution. The specification leaves
many decisions to implementers, and those decisions are where things go wrong. A single
loose parameter check can turn a convenient login flow into an account-takeover
primitive. This post surveys the most common OAuth pitfalls from a defensive angle and
outlines how to reason about them responsibly.

> This is general, educational material. Any hands-on testing must only ever be carried
> out against applications and accounts you are explicitly authorized and in-scope to
> test.

## A Quick Refresher on the Authorization Code Flow

In the recommended authorization code flow, several parties interact: the **resource
owner** (the user), the **client** (the application), and the **authorization server**
(the identity provider). Simplified, it looks like this:

```http
GET /authorize?response_type=code
    &client_id=web-app
    &redirect_uri=https://client.example.com/callback
    &scope=openid%20email
    &state=8f14e45fcea167a5 HTTP/1.1
Host: auth.example.com
```

The user authenticates and consents, the authorization server redirects back to
`redirect_uri` with a short-lived `code`, and the client exchanges that code for tokens
over a back-channel request. Every pitfall below is a way this handshake can be
subverted.

## Pitfall 1: Weak redirect_uri Validation

The `redirect_uri` is where the authorization server sends the code, which makes its
validation critical. If matching is loose, an attacker can cause the code to land on a
destination they control. Common weaknesses include:

- **Substring or prefix matching** instead of exact matching, allowing
  `https://client.example.com.attacker.example/`.
- **Open paths** where any path under the domain is accepted, combined with an open
  redirect on the client to bounce the code onward.
- **Loose subdomain rules** that accept `https://anything.example.com/`.

The defense is **exact, allowlisted redirect URIs**. Registered values should be
compared in full — scheme, host, port, and path — with no wildcard interpretation.

```text
Registered:  https://client.example.com/callback
Accept:      https://client.example.com/callback
Reject:      https://client.example.com/callback/../evil
Reject:      https://client.example.com.attacker.example/callback
```

## Pitfall 2: Missing or Ignored state (CSRF)

The `state` parameter binds an authorization response to the browser session that
started it. If the client omits it or fails to verify it on return, an attacker can
initiate their own OAuth flow and trick a victim into completing it — potentially
linking the victim's session to the attacker's account, or vice versa. This is CSRF
against the OAuth flow itself.

`state` should be a **high-entropy, unguessable value** tied to the user's session,
generated before redirecting and strictly verified when the response comes back. A
missing or unvalidated `state` is one of the most common and impactful OAuth findings.

## Pitfall 3: Authorization Code Leakage

The code is a bearer secret for a brief window, and several channels can leak it:

- **Referer headers.** If the callback page loads third-party resources, the full URL —
  including the code — may be sent in the `Referer` to those hosts.
- **Browser history and logs.** Codes in URLs can persist in history, proxies, and
  server logs.
- **Open redirects.** Chained with weak `redirect_uri` handling, an open redirect can
  forward the code off-domain.

Codes must be **single-use and short-lived**, and the authorization server should reject
any attempt to redeem a code twice. The client should also consume the code immediately
and avoid loading untrusted content on the callback page.

## Pitfall 4: Missing PKCE

Proof Key for Code Exchange (PKCE) protects the code exchange, and it is now recommended
for *all* client types, not just mobile apps. The client generates a random
`code_verifier`, sends its hash as `code_challenge` on the authorization request, and
presents the original verifier when redeeming the code:

```http
GET /authorize?...&code_challenge=E9Melhoa2Ow...&code_challenge_method=S256
```

Without PKCE, a stolen code can be exchanged by anyone who obtains it. With PKCE, the
code is useless without the matching verifier, which never travels over the front
channel. Absent or improperly enforced PKCE (for example, accepting `plain` instead of
`S256`, or not checking the verifier at all) is worth examining closely.

## Pitfall 5: Scope and Consent Handling

Scopes define what the token can do. Problems arise when a client requests more than it
needs, when the authorization server fails to constrain granted scopes to what the user
approved, or when scope escalation is possible during token refresh. From a defensive
view, request **least privilege**, validate that returned tokens carry only the expected
scopes, and never widen scope silently on refresh.

## Pitfall 6: Insecure Identity Binding and Account Takeover

Using OAuth for login means mapping an external identity to a local account, and the
mapping logic is a frequent source of takeover bugs. Two recurring mistakes:

- **Trusting an unverified email.** If a provider returns an `email` that isn't marked
  verified, and the client links accounts by email, an attacker who controls an
  unverified address at the provider can claim someone else's local account.
- **Linking by email alone.** Accounts should be bound to a **stable, provider-issued,
  unique identifier** (such as `sub`), not to a mutable email address.

```json
// Prefer binding on a stable identifier, and require verification
{ "sub": "prov|a1b2c3", "email": "user@example.com", "email_verified": true }
```

Always confirm `email_verified` before trusting it, and key the local account on the
immutable subject identifier.

## How to Test OAuth Safely

A methodical pass through an OAuth implementation typically checks:

1. **redirect_uri handling** — does it enforce exact matches, or can the destination be
   shifted?
2. **state** — is it present, unpredictable, and verified on return?
3. **PKCE** — is it required and correctly validated?
4. **Code hygiene** — is the code single-use, short-lived, and kept off untrusted pages?
5. **Scope** — are granted scopes constrained to what was consented?
6. **Identity binding** — is linking based on a verified, stable identifier?

Test one variable at a time, using only your own accounts, and confirm real impact
rather than stopping at a suspicious response.

## Defensive Guidance

- **Exact-match redirect URIs** from a registered allowlist.
- **Require and verify `state`** for every flow.
- **Enforce PKCE with S256** across client types.
- **Make codes single-use and short-lived**; reject reuse.
- **Constrain scopes** to least privilege and validate them server-side.
- **Bind accounts to verified, immutable identifiers**, never to unverified email.

## Testing Responsibly

- Stay strictly **in scope** and use **test accounts you control** on both the client
  and the provider side.
- Demonstrate impact with a **minimal proof-of-concept** — never take over a real user's
  account to prove a point.
- Capture clean evidence (the full request/response chain) and report with a concrete,
  actionable remediation.

## Conclusion

OAuth is secure only when the implementer gets the details right, so treat every
parameter in the flow as a decision that must be validated on the server — not a
convenience to be trusted.
