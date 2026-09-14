---
anti_patterns:
  - description: A server-rendered page reads browser-only state during render or patches divergent text with `suppressHydrationWarning`, so the first client render disagrees with server output and hydration either fails or silently papers over real mismatches.
    id: react.rendering.hydrating-over-divergent-output
    name: Hydrating over divergent output
    severity: warn
applies_when: a server-rendered tree contains values that can differ between server output and the first client render, and the treatment of that difference is being chosen
id: react.rendering.hydration-consistency
severity: warn
stage: implementation
tech_stack:
  - react
  - next
title: Keep the First Client Render Equal to Server Output
---

## When to apply

Apply when a component renders values that may legitimately differ between the server render and the browser's first render — clocks and relative times, locale and timezone formatting, random or per-instance identifiers, browser-only storage. Decide how each divergence is treated.

## Guidance

The default is to make the outputs equal. Do not read browser-only state during render; initialize from values the server knows, and move client-only reads behind an explicit boundary:

- If the value is decoration, gate it on mount: initialize state from the server-known value, read the browser value in an effect, and accept the brief default — that flash is the honest cost of the divergence.
- If the value is knowingly different but visually irrelevant — a rendered timestamp, a per-instance id — keep the divergence and mark only that element with `suppressHydrationWarning`. It silences the warning for that element's text and attributes; React still patches nothing, so use it only where the two contents are both acceptable, never to hide structural mismatch (different element trees, missing children).
- If the value must be correct before first paint — a theme class, an authenticated shell — the fix belongs outside React's render: set it with a script that runs before hydration (the pattern theme libraries use), and mark the affected attribute container with `suppressHydrationWarning` so hydration does not reset it. This is an advanced, narrow technique; the server and client must still render the same React output.

Investigate every hydration warning instead of suppressing it by pattern-matching: React reports the exact divergence, and a mismatch in one leaf often means browser-only code ran during render somewhere else. A suppress call is a reviewed decision about one known divergence, not a build flag.

## Anti-pattern

A page renders `lastSaved.toLocaleTimeString()` during SSR and hydration, and the fix applied across the codebase is `suppressHydrationWarning` on every element near a date. One suppress lands on a `<ul>` whose children actually differ between server and client — a real bug (a client-only list) now renders silently wrong with no warning anywhere.

## Why

Hydration assumes the client's first render reproduces the server's output; the browser adopts the server's DOM and attaches to it. Divergence forces React to either warn and repair (cost, noise, flicker) or be told to look away (`suppressHydrationWarning`), which is safe only where both contents are genuinely acceptable. Sorting divergences into "eliminate", "mark", and "move outside render" keeps each treatment matched to its cause.

## Exceptions and boundaries

- Streaming SSR shifts where the mismatch appears but not the rule: what a boundary's children render must match between server and client.
- `suppressHydrationWarning` is one level deep — it does not suppress warnings for that element's children, which is why it is safe only on leaves.
- Client-only subtrees that should never render on the server are a loading-boundary decision (see `react.async.suspense-boundary-scope`), not a suppression decision.
- This Practice does not cover error-decoded hydration repair APIs or framework nonce/CSP interactions with pre-hydration scripts.

## Example

A draft editor shows when the draft was last saved. Server and client disagree on the rendered time string (timezones differ), so the treatment is chosen per surface: the decoration version mount-gates and shows the neutral string first; the version that must show a real timestamp immediately marks the leaf as a known divergence.

```tsx
function SavedAt({ isoTimestamp }: { isoTimestamp: string }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);

  return (
    <span>
      {mounted
        ? `Saved at ${new Date(isoTimestamp).toLocaleTimeString()}`
        : "Saved"}
    </span>
  );
}

function SavedAtExact({ isoTimestamp }: { isoTimestamp: string }) {
  return (
    <span suppressHydrationWarning>
      {new Date(isoTimestamp).toLocaleTimeString()}
    </span>
  );
}
```

`SavedAt` renders identically on both sides ("Saved"), then upgrades after hydration — the flash is the accepted cost. `SavedAtExact` keeps the divergent text but contains the divergence to one leaf whose two renderings are both acceptable. Neither suppresses anything structural, so a real mismatch elsewhere on the page still warns.
