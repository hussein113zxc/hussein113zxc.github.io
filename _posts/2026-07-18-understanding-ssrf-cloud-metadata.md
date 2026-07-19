---
layout: post
title: "Understanding SSRF: From URL Fetchers to Cloud Metadata"
date: 2026-07-18 12:00:00 +0000
categories: web-security ssrf
---

Server-Side Request Forgery (SSRF) is one of the highest-impact vulnerability
classes in modern web applications — and one of my main focus areas. This post is a
practical, defender-oriented walkthrough of *why* SSRF matters and *how* it
escalates from a harmless-looking feature into a critical finding.

> ⚖️ Everything here is general, educational material. Any real testing should only
> ever be performed against systems you are explicitly authorized to test.

## What is SSRF?

SSRF happens when an application can be tricked into making HTTP (or other protocol)
requests to a destination the attacker controls or chooses. Instead of the *user*
reaching a resource, the *server* does — often from a trusted network position.

The danger isn't the request itself; it's **where the server sits**. A server that
can reach internal services, admin panels, or cloud metadata endpoints becomes a
powerful proxy in the attacker's hands.

## Where SSRF Hides

In practice, SSRF tends to live behind features that fetch remote content:

- **URL parameters** that load a preview, avatar, or thumbnail
- **Webhooks & integrations** that call out to a user-supplied endpoint
- **Import-from-URL** features (documents, feeds, images)
- **PDF / screenshot generators** that render a given URL

Any of these can become an SSRF sink if the destination isn't strictly validated.

## Why Cloud Metadata Is the Classic Target

Most cloud providers expose an **instance metadata service** on a link-local
address — `169.254.169.254`. From inside the instance, that endpoint can return
configuration and, critically, **temporary credentials** for the instance's role.

- **AWS:** `http://169.254.169.254/latest/meta-data/`
- **GCP:** `http://169.254.169.254/computeMetadata/v1/`
- **Azure:** `http://169.254.169.254/metadata/instance`

If an SSRF can reach this endpoint, a "fetch a URL" bug can turn into leaked cloud
credentials — the difference between a Medium and a Critical.

## The Escalation Ladder

A typical SSRF impact chain looks like this:

1. **Confirm** the server makes a request to a destination you control.
2. **Pivot inward** — can it reach `localhost`, internal IP ranges, or private
   services?
3. **Reach metadata** — can it hit the metadata endpoint?
4. **Extract & escalate** — retrieve credentials or internal data, then assess how
   far that access reaches.

Each rung raises severity. The best reports demonstrate the *highest safe rung*
without ever touching real customer data.

## Why Naive Defenses Fail

Teams often add protections that look sufficient but aren't:

- **Blocklists** of internal IPs miss encodings, IPv6, and alternate notations.
- **Allowlists** can be defeated if redirects are followed to an off-list host.
- **Validate-once** logic checks the URL at submission time but not at fetch time —
  leaving a gap that time-of-check/time-of-use techniques exploit.

The robust fix is defense-in-depth: validate at request time, disable redirects (or
re-validate them), block the metadata range at the network layer, and require
authentication headers on metadata (IMDSv2-style).

## Testing Responsibly

When hunting SSRF in a bug bounty program:

- Stay strictly **in scope**.
- Use a **proof-of-concept** that demonstrates impact without exfiltrating real
  data.
- Capture clean evidence (request/response) with timestamps.
- Report clearly, with a concrete remediation.

## Closing

SSRF is a reminder that the *server's network position* is part of your attack
surface. A one-line "fetch this URL" feature, placed on an instance with metadata
access, can be one of the most valuable findings in a program — which is exactly why
it's worth understanding deeply.

*More writeups on GraphQL and IDOR/BOLA coming soon.*
