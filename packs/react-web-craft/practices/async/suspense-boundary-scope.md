---
anti_patterns:
  - description: A page component awaits its slowest data source before returning any JSX, so header, navigation, and fast sections cannot render or stream until it resolves.
    id: react.async.shell-blocked-by-slowest-subtree
    name: Whole shell awaits one slow data source
    severity: warn
applies_when: a server component tree waits on one slow data source even though only one subtree's output depends on it
id: react.async.suspense-boundary-scope
severity: warn
stage: design
tech_stack:
  - react
  - next
title: Put Slow Subtrees Behind Their Own Suspense Boundaries
---

## When to apply

Apply when one server component's data need is much slower than the rest of a page's. Decide which part of the interface that data feeds and where the Suspense boundary that isolates it belongs.

## Guidance

An await in an async component delays everything its output feeds; an await at the top of a page delays the whole shell. Move the slow fetch into the child that needs it and wrap that child in `<Suspense>`. The shell renders immediately — and, under a streaming setup such as the Next.js App Router, streams to the client — while the slow child shows its fallback until its data lands.

Boundary placement is the decision. A boundary around the whole page makes even fast content wait behind one fallback; a boundary per slow subtree lets each fallback resolve on its own schedule and keeps fast, critical content in the initial response. A page usually wants the second shape: fast, layout-critical, or SEO-relevant content stays unwrapped, and only genuinely slow subtrees sit behind boundaries.

Not awaiting in the parent is also what starts sibling fetches concurrently — the component structure is the parallelism mechanism; there is no need for a parent-level `Promise.all` over children the parent does not consume. When several consumers need the same data, start the promise once and read it with React 19's `use()` inside each consumer under one boundary, so the request happens once and they fall back together.

Choose with the product, not only the profiler. A boundary buys a faster first paint at the cost of content arriving later inside a fallback, which can shift layout; that trade-off is a UX decision to make deliberately per surface, not a default.

## Anti-pattern

A product page awaits the reviews feed before returning any JSX. The title, price, and navigation depend on none of it, yet they all wait for the slowest source; the browser shows nothing until reviews resolve.

## Why

A Suspense boundary converts "this subtree's data" into "this subtree's timing": the shell is no longer gated by the slowest fetch, and each wrapped subtree resolves independently. Keeping fast content outside boundaries preserves it in the initial response, which matters for perceived speed and for crawlers.

## Exceptions and boundaries

- Layout-critical data — content whose size or position determines the surrounding structure — belongs in the shell; hiding it behind a fallback causes reflow.
- SEO-critical above-the-fold content should remain in the initial response rather than arrive through a fallback.
- For cheap queries, fallback churn can cost more than the wait would; a boundary is a cost boundary, not a default wrapper.
- If a slow subtree must also start before the page renders (prefetching during navigation), that is a framework navigation decision this Practice does not cover.
- This Practice decides where timing is isolated in the component tree. Single-function sequencing is `react.async.parallel-independent-work` and `react.async.await-after-cheap-work`.

## Example

A product page keeps the fast catalog read in the shell — it is above the fold and SEO-relevant — and isolates the two slow panes behind their own boundaries. The declared helpers stand in for the data layer.

```tsx
import { Suspense } from "react";

type ProductSummary = { name: string; price: string };

declare function getProductSummary(id: string): Promise<ProductSummary>;
declare function getProductReviews(id: string): Promise<{ items: string[] }>;
declare function getCompatibilityChecks(id: string): Promise<{ rows: string[] }>;

async function ProductHead({ id }: { id: string }) {
  const product = await getProductSummary(id);
  return (
    <header>
      <h1>{product.name}</h1>
      <p>{product.price}</p>
    </header>
  );
}

async function ReviewsPane({ id }: { id: string }) {
  const reviews = await getProductReviews(id);
  return (
    <section>
      {reviews.items.map((item) => (
        <p key={item}>{item}</p>
      ))}
    </section>
  );
}

async function CompatibilityPane({ id }: { id: string }) {
  const checks = await getCompatibilityChecks(id);
  return (
    <aside>
      {checks.rows.map((row) => (
        <p key={row}>{row}</p>
      ))}
    </aside>
  );
}

export function ProductPage({ id }: { id: string }) {
  return (
    <main>
      <ProductHead id={id} />
      <Suspense fallback={<p>Loading reviews…</p>}>
        <ReviewsPane id={id} />
      </Suspense>
      <Suspense fallback={<p>Checking compatibility…</p>}>
        <CompatibilityPane id={id} />
      </Suspense>
    </main>
  );
}
```

`ProductPage` itself never awaits, so the head data ships in the initial response while each pane resolves behind its own fallback. The two panes' fetches also start concurrently, because no parent awaits them before rendering.
