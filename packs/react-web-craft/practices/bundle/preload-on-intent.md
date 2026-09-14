---
anti_patterns:
  - description: Deferred chunks are preloaded indiscriminately on mount, or not at all, so preload traffic competes with the current page or the deferred code is always paid as a wait at click time.
    id: react.bundle.preload-without-intent-signal
    name: Preloading without an intent signal
    severity: warn
applies_when: code or data has been deferred into a separate chunk, and a decision is needed about whether and when to fetch it ahead of the request that uses it
id: react.bundle.preload-on-intent
severity: warn
stage: implementation
tech_stack:
  - react
  - next
title: Preload Deferred Code When User Intent Signals It
---

## When to apply

Apply when a feature's code lives behind a dynamic import and users reach it through a visible, predictable step — a button, a tab, a wizard's next stage. Decide whether the chunk should start loading at some signal before the click, and which signal earns that head start.

## Guidance

Rank signals by strength and cost. An explicit step toward the feature (a key press, completing the previous wizard stage, a feature flag known to enable it) is the strongest; pointer and focus intent (`onMouseEnter`, `onFocus`) buys a small head start cheaply; unconditional preload on mount is the weakest and usually wrong. Preload the few chunks the current surface actually leads to, not a catalog:

```tsx
const preloadCheckout = () => {
  void import("./checkout-flow");
};

export function CheckoutEntry() {
  return (
    <button
      onMouseEnter={preloadCheckout}
      onFocus={preloadCheckout}
      onClick={() => void import("./checkout-flow").then((m) => m.openCheckout())}
    >
      Go to checkout
    </button>
  );
}
```

The preload call is free of bookkeeping: a module's `import()` promise is cached, so the click-time call and the earlier preload share one in-flight or completed fetch. Give the click handler its own `import()` call rather than stashing shared state.

Know what the framework already does. Next.js prefetches linked routes (production `<Link>` prefetches on viewport entry), so intent-based preloading earns its keep mainly for dynamically imported chunks and non-link targets — exactly the code the defer decision split off. Preloading is bandwidth spent in advance: wrong guesses compete with the current page's own bytes, so treat the preload list as a small, measured decision, not a default.

## Anti-pattern

A dashboard mounts and immediately preloads every panel's chunk — twelve heavy modules — because one of them might be opened. On metered and mobile connections the preloads strangle the dashboard's own data fetching; the panels most users open are no faster than without preloading.

## Why

Deferral moves cost from everyone to the feature's users; preloading on intent gives those users the download back while they are still deciding, so the click resolves against an already-arrived chunk. The signal, not the mount event, is what separates a head start from a tax.

## Exceptions and boundaries

- If the deferred feature is one tap from every session and small enough to be instant anyway, skip preloading; this is a cost boundary.
- Do not preload to compensate for deferring critical-path code; fix the deferral decision instead.
- Preloading code is not prefetching data: cached module code still fetches its own data on use, and data prefetching has its own staleness and privacy considerations outside this Practice.
- On the server, preload is meaningless — the guard question is execution, not inclusion; see `react.bundle.defer-heavy-loads`.

## Example

A checkout wizard renders each stage from its own deferred chunk and preloads the next stage's chunk as soon as the user lands on the current one — an explicit step, the strongest signal. The two heavy stages are deferred; `AddressForm` is a small local component that stays in the main bundle, because splitting it would cost more than it saves.

```tsx
"use client";

import { useEffect } from "react";
import dynamic from "next/dynamic";

const ReviewStage = dynamic(() => import("./review-stage"));
const PaymentStage = dynamic(() => import("./payment-stage"));

export function CheckoutWizard({ stage }: { stage: "address" | "review" | "payment" }) {
  useEffect(() => {
    if (stage === "address") void import("./review-stage");
    if (stage === "review") void import("./payment-stage");
  }, [stage]);

  if (stage === "address") return <AddressForm />;
  if (stage === "review") return <ReviewStage />;
  return <PaymentStage />;
}
```

While the user fills the address form, the review chunk is already arriving; crossing to the next stage renders against a warm chunk instead of a spinner. The preloads stay bounded — one chunk per step, triggered by the step the user is actually on.
