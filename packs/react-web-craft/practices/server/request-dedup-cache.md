---
anti_patterns:
  - description: Several components in one server render each perform the same authentication or database lookup, multiplying latency and backend load although one lookup would serve the whole request.
    id: react.server.duplicate-request-lookups
    name: Request-scoped lookups repeated within a render
    severity: warn
applies_when: one server render calls the same request-scoped async work, such as session or record lookup, from more than one component or helper
id: react.server.request-dedup-cache
severity: warn
stage: implementation
tech_stack:
  - react
  - next
title: Deduplicate Request-Scoped Lookups with React cache
---

## When to apply

Apply when a server render needs the same request-scoped value — the authenticated user, a tenant setting, a record that several components display — and more than one component or helper fetches it independently. Decide where the single per-request lookup lives.

## Guidance

Wrap the lookup in React's `cache()`. Within one server request, every call to the cached function shares one execution and one result; the cache lives exactly for the request and nothing survives it:

```ts
import { cache } from "react";

export const getCurrentUser = cache(async (token: string | null) => {
  if (!token) return null;
  return loadUserByToken(token);
});
```

Match the arguments to the cache's equality. `cache()` compares arguments with `Object.is`: primitives hit, freshly built object literals always miss. Pass primitive keys, or pass one stable reference around if an object argument is unavoidable.

Do not reach for `cache()` where the framework already deduplicates: in Next.js, identical `fetch` calls in one request are memoized automatically, so `cache()` earns its keep on non-fetch work — database and ORM queries, authentication checks, file reads, expensive pure computation. And do not expect it to share anything across requests: deduplication across requests, users, or instances is a different decision with invalidation and isolation consequences that `cache()` deliberately does not make.

## Anti-pattern

A layout fetches the session user, the page fetches it again for the heading, and a sidebar component fetches it a third time for the avatar. One page render issues three identical database queries; under load, that multiplies backend work by component count.

## Why

A server render is one logical unit of work, but its component tree is many independently written pieces; without a per-request memo, each piece re-derives shared inputs. `cache()` scopes the memo to the request precisely so sharing happens inside the request and isolation is preserved across requests and users.

## Exceptions and boundaries

- If the lookup runs exactly once per request anyway, `cache()` adds ceremony without benefit; this is a cost boundary.
- Arguments that cannot be expressed as primitives or stable references defeat the cache; reshape the signature rather than passing fresh object literals.
- Cached values are frozen for the request's lifetime: use it for data the request treats as constant, not for values a later mutation must immediately re-read.
- Module-level or cross-request caching is a separate decision with its own invalidation, tenancy, and memory design; see `react.server.no-module-request-state` for the isolation boundary that motivates keeping per-request data out of module scope.

## Example

Two independent server components on one account page each need the authenticated user. Both call the same `cache()`-wrapped lookup; within the request it loads once. The declared helpers stand in for the data layer.

```tsx
import { getCurrentUser } from "./session";

export async function AccountHeader({ token }: { token: string | null }) {
  const user = await getCurrentUser(token);
  return <header>{user?.name}</header>;
}

export async function BillingNotice({ token }: { token: string | null }) {
  const user = await getCurrentUser(token);
  return user?.plan === "trial" ? <p>Trial ends soon.</p> : null;
}
```

One render of a page composing both components issues one `loadUserByToken` call, no matter how many components need the user. A second request — even the same user reloading — runs the lookup again, because nothing survives the request.
