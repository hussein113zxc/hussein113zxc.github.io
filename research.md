---
layout: page
title: Research &amp; Approach
permalink: /research/
---

I research **web application security** and participate in **authorized bug bounty
programs**, with a focus on how authorization and business-logic assumptions break
down in modern APIs.

## What I focus on

- **GraphQL** — authorization bypass, batching abuse, introspection exposure
- **SSRF** — cloud metadata access and SSRF-to-cloud escalation paths
- **IDOR / BOLA** — broken object-level authorization
- **Authentication** — JWT and OAuth implementation flaws
- **Access control** — horizontal and vertical privilege escalation
- **Business logic** — workflow and price-manipulation flaws

## How I work

Testing happens strictly inside the authorized scope of official programs, following
each program's rules of engagement. The goal is to demonstrate an issue safely —
without touching real user data — and to write it up so a triage team can reproduce
and act on it quickly.

Findings stay private under each program's disclosure policy. What I publish here is
the *general* technique: how a class of bug arises, how it escalates in impact, and
how to defend against it.

## Published research

Technical, defensive-oriented writeups covering the vulnerability classes above —
walking through the mechanics, the escalation path, and the mitigations.

**[Read all writeups &rarr;](/)**

## Elsewhere

- **HackerOne:** [aizen-sosuke](https://hackerone.com/aizen-sosuke)
- **GitHub:** [hussein113zxc](https://github.com/hussein113zxc)
