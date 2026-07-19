---
layout: post
title: "JWT Attacks: Algorithm Confusion, Weak Secrets, and Claim Abuse"
date: 2026-07-13 12:00:00 +0000
categories: web-security authentication
---

JSON Web Tokens are everywhere: session replacements, API credentials, single
sign-on assertions. Their appeal is that they are self-contained and verifiable
without server-side state. That same property is their weakness — if a service
verifies a token incorrectly, an attacker who can craft or tamper with tokens may
be able to impersonate anyone. This post walks through the main JWT attack surfaces
from a defensive standpoint, so you can recognize and close them in your own systems.

> This is general, educational material. Any hands-on testing must only be performed
> against applications you are explicitly authorized and in-scope to test. Do not test
> tokens or credentials you do not own.

## JWT Structure in Brief

A signed JWT (technically a JWS) has three Base64URL-encoded parts joined by dots:
`header.payload.signature`. The header declares the signing algorithm, the payload
carries claims, and the signature binds them together.

```json
// header
{ "alg": "HS256", "typ": "JWT" }

// payload
{ "sub": "1024", "role": "user", "exp": 1789000000 }
```

Every attack below comes down to one question: **can the client influence what the
server accepts as a valid signature?** If yes, the token stops being a trust anchor.

## Attack Surface 1: The Algorithm Header

### The "none" Algorithm

The JWT specification includes an `alg` value of `none`, meaning "unsecured" — no
signature at all. If a verification library honors it, an attacker can strip the
signature, set `"alg": "none"`, and submit an unsigned token with any claims:

```json
{ "alg": "none", "typ": "JWT" }
.
{ "sub": "1024", "role": "admin", "exp": 1789000000 }
.
```

A correctly configured verifier must **reject `none`** whenever it expects a signed
token. The defensive rule is to never let the token itself dictate whether it should
be verified.

### Algorithm Confusion (Asymmetric to Symmetric)

This is the subtle one. Asymmetric algorithms like RS256 sign with a *private* key and
verify with a *public* key. Symmetric algorithms like HS256 use a single shared secret
for both. The public key, by design, is not secret.

The confusion arises when a verifier selects its algorithm based on the token's own
`alg` header and calls a generic `verify(token, key)` routine. If the server intends
RS256 and holds the RSA public key, an attacker can:

1. Change the header to `"alg": "HS256"`.
2. Forge claims in the payload.
3. Compute an HMAC-SHA256 signature using the **public key bytes as the HMAC secret**.

Because the server "verifies" HS256 using that same public key, and the public key is
known, the forged signature checks out. The fix is to **pin the expected algorithm
server-side** and never derive it from attacker-controlled input:

```text
// Vulnerable pattern
verify(token, key)                  // algorithm taken from the header

// Safer pattern
verify(token, key, { algorithms: ["RS256"] })   // algorithm enforced by the server
```

## Attack Surface 2: Weak or Guessable Secrets

HMAC-signed tokens are only as strong as their secret. If a service signs with a short,
dictionary-based, or reused key, that secret can be recovered offline. An attacker who
captures a single valid token can run it against a wordlist locally — no traffic to the
target — testing candidate secrets until one reproduces the signature:

```text
# Conceptual offline check against a captured token
for candidate in wordlist:
    if hmac_sha256(header + "." + payload, candidate) == signature:
        found(candidate)
```

Once the secret is known, the attacker can sign arbitrary tokens with any claims. The
defenses are straightforward: use a **long, random, high-entropy secret** (treat it
like any cryptographic key), store it in a secrets manager rather than source control,
rotate it if exposure is suspected, and prefer asymmetric signing for tokens that cross
trust boundaries.

## Attack Surface 3: Claim Abuse

Even with a sound signature, weak *validation of claims* undermines security.

### Missing or Ignored Expiry

If a service doesn't check `exp`, tokens live forever, so a token leaked once (via
logs, a referer header, or shared caches) never stops working. Related claims matter
too: `nbf` (not before), `iat` (issued at), `iss` (issuer), and `aud` (audience) should
all be validated. A token minted for one audience should not be accepted by another.

### Identity and Privilege Claims

Claims like `sub`, `role`, `scope`, or `tenant_id` often drive authorization. If any of
them can be altered — because of a signature weakness above — an attacker rewrites their
identity or privilege level:

```json
{ "sub": "1024", "role": "admin", "tenant_id": "victim-org" }
```

The token is the wrong place to make the *final* authorization decision blindly. Where
possible, cross-check sensitive claims against server-side records, and never trust a
claim you would not trust as a raw query parameter.

### Header Injection: kid, jku, and jwk

The header can carry key-selection fields. The `kid` (key ID) tells the server which
key to use; if it is passed unsanitized into a file path or database lookup, it becomes
a path-traversal or injection vector. Fields like `jku` (a URL to fetch keys) and `jwk`
(an embedded key) are more dangerous still: if honored, an attacker points the server
at a key *they* control and signs tokens that validate perfectly.

```json
{ "alg": "RS256", "kid": "../../dev/null", "jku": "https://attacker.example/keys" }
```

Servers should ignore attacker-supplied key sources entirely, treat `kid` as an opaque
lookup against a fixed allowlist, and never fetch verification keys from a URL contained
in the token.

## Defensive Checklist

- **Pin the algorithm** on the server; reject `none` and any unexpected `alg`.
- **Separate key material by algorithm** so a public key can never be used as an HMAC
  secret.
- **Use high-entropy secrets** for HMAC and manage them like the sensitive keys they
  are.
- **Validate every relevant claim**: `exp`, `nbf`, `iss`, `aud`, and identity fields.
- **Never trust header-supplied key sources** (`jku`, `jwk`); constrain `kid` to a
  known set.
- **Keep token lifetimes short** and support revocation for high-value sessions.

## Testing Responsibly

When assessing JWT handling under a bug bounty or engagement:

- Work only with **tokens issued to your own test accounts**, strictly in scope.
- Demonstrate impact with a **minimal proof-of-concept** — for example, showing a
  self-signed token is accepted — without accessing other users' data.
- Record the exact token, request, and response as evidence, then report with a clear
  remediation.

## Conclusion

Nearly every JWT vulnerability reduces to the same root cause — letting the token decide
how it should be trusted — so the strongest defense is to make those decisions on the
server and never hand them to the client.
