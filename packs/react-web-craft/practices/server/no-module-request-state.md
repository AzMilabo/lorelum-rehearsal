---
anti_patterns:
  - description: A server render writes per-request or per-user data into a module-level variable, so concurrent requests can overwrite or read each other's values.
    id: react.server.module-state-per-request
    name: Request data kept in module scope
    severity: warn
applies_when: server-rendered code needs to share request-scoped data across components or helpers, and the sharing mechanism is being chosen
id: react.server.no-module-request-state
severity: warn
stage: implementation
tech_stack:
  - react
  - next
title: Keep Request Data Out of Module Scope
---

## When to apply

Apply when writing or reviewing server-side code — server components, SSR'd client components, route handlers, server actions — that shares a value such as the current user, a request ID, or a locale across parts of one request. Decide where that value lives.

## Guidance

Treat module scope on the server as process-wide shared memory. Concurrent renders run in the same process, so a module-level variable written by request A is read by request B, and the last writer wins: races, cross-request contamination, and one user's data appearing in another's response all follow from that.

Pass request data through the render tree instead — as arguments and props from the entry point, or through a request-scoped memo when several components need the same lookup:

```tsx
export default async function OrderPage({ orderId }: { orderId: string }) {
  const user = await getCurrentUser(token);
  const order = await getOrder(orderId);
  return <OrderSummary user={user} order={order} />;
}
```

Keep an explicit allowlist for what may live at module scope: immutable configuration read once, constants, and shared caches designed for cross-request reuse with correct keys and size bounds. The test is not "is it a global?" but "can this value ever differ between two concurrent requests?" — if it can, it does not belong in module scope. Framework request objects and request-scoped storage exist precisely to carry per-request context; use the framework's facility rather than hand-rolled globals.

## Anti-pattern

A dashboard page assigns the authenticated user to a module-level `currentUser` and a nested component reads it. Two overlapping requests interleave: request B overwrites `currentUser` between request A's assignment and A's render of the nested component, and A's page shows B's account name.

## Why

Request isolation is the server's most basic correctness boundary. Module state silently dissolves it because every request shares the module instance; the bug appears only under concurrency, in production, and as another user's data — the most expensive kind of defect to reproduce and the worst kind to ship.

## Exceptions and boundaries

- Immutable static configuration and code constants are safe at module scope; they cannot differ between requests.
- An intentionally shared cache is safe only with keys that include every axis the data varies on (user, tenant, locale) plus eviction and size bounds; designing one is its own decision, and per-request sharing needs no such machinery.
- This Practice concerns mutable request-scoped data; module-level singletons that hold no per-request state, such as a database client, are a different pattern with different constraints.
- Within-request duplication of a lookup is a performance decision handled by `react.server.request-dedup-cache`; this Practice governs where the result may live.

## Example

An order page resolves its request-scoped values at the entry point and passes them down. Every concurrent request renders against its own values.

```tsx
import { getCurrentUser } from "./session";

export default async function OrderPage({
  orderId,
  token,
}: {
  orderId: string;
  token: string | null;
}) {
  const user = await getCurrentUser(token);
  const order = await getOrder(orderId, user);
  return (
    <section>
      <OrderTitle order={order} />
      <OrderItems order={order} />
      <SupportContact user={user} />
    </section>
  );
}
```

`user` and `order` exist only inside this request's render tree; the only module-level objects involved are the imported functions and read-only configuration. Two overlapping requests for different orders cannot observe each other's data, because there is no shared variable to observe.
