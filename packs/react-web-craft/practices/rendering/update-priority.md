---
anti_patterns:
  - description: An input's keystroke triggers a synchronous expensive filter or render, so the control the user is typing into stutters even though the expensive result could lag behind without harm.
    id: react.rendering.urgent-input-blocked-by-heavy-render
    name: Urgent input blocked by heavy follow-up render
    severity: warn
applies_when: one interaction produces both an urgent visual update and an expensive derived render or data follow-up, and their priorities are being chosen
id: react.rendering.update-priority
severity: warn
stage: implementation
tech_stack:
  - react
title: Split Urgent Input Updates from Slow Follow-Up Renders
---

## When to apply

Apply when a single interaction updates a control the user watches — text input, a slider, a selected tab — and also drives work too expensive to finish within one frame. Decide which part of the update is urgent and which part may lag.

## Guidance

Two mechanisms express the split, and the choice follows the shape of the lag:

- The lag is in rendering a derived view of an input value (filtering a large list, re-rendering an expensive chart): defer the value. The control keeps its own immediate state, and the deferred value trails it while React keeps the input responsive:

  ```tsx
  const [query, setQuery] = useState("");
  const deferredQuery = useDeferredValue(query);
  const results = useMemo(
    () => filterEntries(corpus, deferredQuery),
    [corpus, deferredQuery],
  );
  const isStale = query !== deferredQuery;
  ```

- The lag is in a state update the user does not need synchronously (switching a tab whose content is heavy, navigating): wrap the update in `startTransition`, or use `useTransition` when the UI should show that the follow-up is in flight.

Mark the lag with something honest — dimming the stale results, a subtle pending indicator on the tab — so the split is visible instead of looking broken. Do not reach for these tools outside the split: a transition does not make high-frequency updates cheap and is not a scroll throttle; a deferred value is not a network debounce and does not cancel a fetch; and transition `isPending` describes the deferred update's own lifecycle, so error handling and races for async follow-ups remain their own decisions.

## Anti-pattern

A pricing page's range slider writes its value to state, and every change re-filters and re-renders a grid of two hundred cards synchronously. Dragging the slider drops frames on the thumb itself — the one element that must track the pointer — even though the grid could happily lag a few frames behind.

## Why

Concurrent rendering lets React interrupt low-priority work, but only if the code says which updates can wait. Declaring the input update urgent and the derived work deferred keeps the watched control at full frame rate while the expensive part settles, and the stale marker tells the user the lag is deliberate.

## Exceptions and boundaries

- If the derived render is cheap enough to finish within the frame budget, adding deferral is ceremony; measure before reaching for it.
- If the expensive work is a network request rather than render cost, the split is about request lifecycle and caching, not render priority.
- If the follow-up changes what the user is looking at (not a lagging mirror but a navigation), decide with transitions at the navigation boundary; hidden-content timing is the `react.async.suspense-boundary-scope` decision.

## Example

A pricing page's range slider filters a grid of two hundred cards. The slider writes its own urgent state; the grid reads a deferred value and dims while it trails.

```tsx
function PriceExplorer({ products }: { products: Product[] }) {
  const [maxPrice, setMaxPrice] = useState(500);
  const deferredMaxPrice = useDeferredValue(maxPrice);

  const visible = useMemo(
    () => products.filter((product) => product.price <= deferredMaxPrice),
    [products, deferredMaxPrice],
  );
  const isStale = maxPrice !== deferredMaxPrice;

  return (
    <>
      <input
        type="range"
        min={50}
        max={500}
        value={maxPrice}
        onChange={(event) => setMaxPrice(Number(event.currentTarget.value))}
      />
      <div style={{ opacity: isStale ? 0.6 : 1 }}>
        <ProductGrid products={visible} />
      </div>
    </>
  );
}
```

The thumb tracks the pointer at full frame rate because its update never waits for the grid. The grid settles a frame or two behind, and the dimmed state tells the user the lag is deliberate filtering, not a freeze.
