---
layout: post
title: "GraphQL Security: From Introspection to Broken Authorization"
date: 2026-07-17 12:00:00 +0000
categories: web-security graphql
---

GraphQL gives clients a flexible, self-describing way to ask for exactly the data
they need. That same flexibility reshapes the security model: a single endpoint, a
strongly typed schema, and client-driven queries create an attack surface that
behaves very differently from a traditional REST API. This post is a
defender-oriented tour of how GraphQL security issues develop — starting with
introspection and ending with the broken authorization flaws that tend to matter
most.

> ⚖️ Everything here is general, educational material. Any real testing should only
> ever be performed against systems you are explicitly authorized to test and that
> fall within a program's defined scope.

## A Different Kind of Attack Surface

A REST API usually spreads its functionality across many URLs and HTTP verbs. GraphQL
collapses that into a **single endpoint** — commonly `/graphql` — where the *body* of
the request decides what happens. Three operation types exist:

- **Queries** read data.
- **Mutations** change state.
- **Subscriptions** stream updates over a persistent connection.

Behind the schema, each field is backed by a **resolver** — a function that fetches or
computes that field's value. This matters enormously for security: authorization is
only as strong as the checks inside those resolvers. A missing check on one nested
field can expose data even when the parent query looks harmless.

## Introspection: Reading the Map

GraphQL ships with a built-in introspection system that lets a client ask the server
to describe its own schema. For a tester this is the equivalent of being handed the
full API documentation:

```graphql
query {
  __schema {
    types {
      name
      fields {
        name
        args { name }
      }
    }
  }
}
```

The response enumerates every type, field, argument, and mutation. From there you can
see exactly which objects exist, which fields are sensitive, and which mutations
change state.

Disabling introspection in production is reasonable defense-in-depth, but treat it as
**obfuscation, not a control**. Field-suggestion error messages ("Did you mean
`email`?") and known client queries can rebuild much of the schema even when
introspection is off. The real security boundary is authorization, not schema
secrecy.

## Object-Level Authorization: The Core Risk

The most common high-impact GraphQL flaw is **broken object-level authorization**
(the GraphQL flavor of IDOR/BOLA). Many schemas expose objects by identifier:

```graphql
query {
  order(id: "1024") {
    id
    total
    shippingAddress
  }
}
```

If the resolver fetches the order by `id` but never verifies that the *authenticated
user* owns order `1024`, then changing the identifier walks straight into another
customer's data. Global-ID patterns (`node(id: "...")`) can widen this further,
because a single `node` field may resolve many object types through one code path.
The lesson for defenders is that **every object resolver must re-check ownership**,
regardless of how the object was reached.

## Field-Level Authorization

Authorization is not only about whole objects — individual fields carry different
sensitivity:

```graphql
query {
  me {
    displayName      # safe for the owner
    email            # more sensitive
    role             # authorization-relevant
  }
}
```

A frequent mistake is enforcing access at the top-level query but letting nested
resolvers return privileged fields without their own checks. A `user` object reached
through a public `comments` list, for example, might still expose an `email` or
`phoneNumber` field if that field lacks its own guard. Sensitive fields deserve
explicit, field-level authorization.

## Mutations and State Change

Mutations deserve special attention because they *write*:

```graphql
mutation {
  updateUserRole(userId: "77", role: "ADMIN") {
    id
    role
  }
}
```

If role assignment is enforced only in the UI, or if the resolver trusts a
client-supplied `role` without checking the caller's privileges, this becomes
privilege escalation. Defenders should treat every mutation as a trust boundary and
validate both *who* is calling and *what* they are allowed to change.

## Aliasing and Batching

GraphQL lets a client request the same field multiple times using **aliases**:

```graphql
query {
  a: order(id: "1001") { total }
  b: order(id: "1002") { total }
  c: order(id: "1003") { total }
}
```

A single HTTP request now performs many lookups. If rate limiting counts requests
rather than operations, aliasing can quietly defeat it — turning one request into a
high-volume enumeration or brute-force attempt (for example, against a
resource-guessing or one-time-code field). Array-based query batching, where an array
of operations is posted at once, has the same effect. Robust designs apply limits at
the *operation* and *field* level, not just per HTTP request.

## Query Depth and Cost (Denial of Service)

Because clients compose their own queries, they can also compose *expensive* ones.
Cyclic relationships make this vivid:

```graphql
query {
  user {
    posts {
      author {
        posts {
          author { posts { id } }
        }
      }
    }
  }
}
```

Each level multiplies the work the server must do. Without guardrails, a compact query
can trigger enormous database load. Mature GraphQL deployments defend with **query
depth limits**, **cost/complexity analysis**, **pagination caps**, and **execution
timeouts** so that no single query can exhaust resources.

## Building Robust Authorization

Pulling the defensive themes together:

- **Enforce authorization in resolvers**, close to the data, not in the transport
  layer or the client.
- **Re-check ownership on every object**, including nested and `node`-resolved ones.
- **Guard sensitive fields individually**, not just top-level entry points.
- **Treat mutations as trust boundaries** and never trust client-supplied role or
  ownership arguments.
- **Limit operations and cost**: depth limits, complexity budgets, pagination caps,
  and sensible timeouts.
- **Disable introspection in production** as defense-in-depth, while relying on real
  authorization for security.

## Testing Responsibly

When examining GraphQL in an authorized program:

- Confirm the endpoint and operations are **in scope** before probing.
- Use **your own test accounts** to demonstrate cross-tenant access rather than
  touching real users' data.
- Keep proof-of-concept queries **minimal** — enough to show impact, no more.
- Capture clean request/response evidence with timestamps, and pair every finding with
  a concrete remediation.

## Closing

GraphQL doesn't create new authorization principles — it just moves the enforcement
point into resolvers and rewards anyone who forgets it, which is exactly why
disciplined, per-object, per-field access control is the heart of GraphQL security.
