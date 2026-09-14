---
anti_patterns:
  - description: A page head accumulates speculative hints for origins and resources it may never use while a blocking script tag delays first paint, so the browser spends its budget on guesses and waits anyway.
    id: react.rendering.speculative-hints-blocking-script
    name: Speculative hints with a blocking script
    severity: warn
applies_when: a page declares resource hints or loads third-party and inline scripts, and the priority each one deserves is being chosen
id: react.rendering.resource-hints-scripts
severity: warn
stage: implementation
tech_stack:
  - react
  - next
title: Match Resource Hints and Script Loading to Real Need
---

## When to apply

Apply when a page or layout declares resource hints (`preload`, `preconnect`, DNS hints) or loads scripts that are not part of the framework bundle. Decide, per resource, how early the browser should start working on it.

## Guidance

Hints are a ladder, and each rung costs more — spend the higher rungs only on certainty:

- `prefetchDNS` for a domain you will probably contact; `preconnect` for one you will certainly contact soon (an API origin, a font CDN). Browsers keep only a few connections warm, so a head full of preconnects evicts itself.
- `preload` a resource the current page will definitely use and the parser would otherwise discover late — a critical font, a hero image. React DOM exposes these as functions callable during server render (`preload`, `preinit`, `preloadModule`), which lets the hint ship with the HTML instead of after discovery.
- `preinit` for a stylesheet or script that must load and execute early; it is the strongest, so reserve it for the genuinely critical.

Scripts follow the same priority logic. A classic `<script src>` blocks parsing; `defer` downloads in parallel and executes in order after parsing, `async` executes as soon as it arrives with no order guarantee. Choose by dependency, not habit: DOM- or script-dependent code takes `defer`; independent counters and tags take `async`. In Next.js, use the framework's script component and its strategies (`afterInteractive` for most third-party tags, `lazyOnload` for anything that can wait for idle) instead of raw tags, because the framework coordinates them with hydration.

Every hint is a request to jump the queue, and queues are shared: review them when the page's critical path changes, and delete hints for resources that are no longer critical.

## Anti-pattern

A marketing page's head preconnects to eight third-party origins, preloads two hero images that already have `fetchpriority` handling, and still loads its analytics tag as a plain blocking script. The browser burns connections on origins the page may never call, while the one thing a visitor actually waits for — first paint — waits for a tag that needed `async`.

## Why

The browser starts working on resources only when it learns about them, and it serializes speculative work per origin. Hints exist to move certainty earlier; scripts' `defer`/`async` exist to stop independent code from standing in front of content. Matching each resource to the rung its certainty earns is what makes first paint fast; over-hinting is how a page pays for guesses and still blocks.

## Exceptions and boundaries

- A framework router often prefetches its own routes and fonts; do not hand-hint what the framework already emits — duplicate hints waste connections.
- `preload` for cross-origin resources needs correct `crossorigin` matching or the fetch is refetched and the hint is wasted; verify the attribute matches the eventual consumer.
- Preloading deferred feature code on intent is a different decision with different signals; see `react.bundle.preload-on-intent`.
- This Practice does not set a caching or CDN strategy; hints change discovery timing, not cacheability.

## Example

A documentation site declares exactly what it is certain about: one search-origin connection, the display font, the critical stylesheet — and its analytics tag waits for idle.

```tsx
import { preconnect, preload, preinit } from "react-dom";
import Script from "next/script";

export default function DocsLayout({ children }: { children: React.ReactNode }) {
  preconnect("https://search.docs.example.com");
  preload("/fonts/docs.woff2", {
    as: "font",
    type: "font/woff2",
    crossOrigin: "anonymous",
  });
  preinit("/styles/critical.css", { as: "style" });

  return (
    <html>
      <body>
        {children}
        <Script src="https://analytics.example.com/collector.js" strategy="lazyOnload" />
      </body>
    </html>
  );
}
```

The resource functions ship their hints with the HTML, so the font and stylesheet start loading before the parser would find them; the analytics tag — independent and non-critical — neither blocks parsing nor competes with hydration. Nothing speculative occupies the head: no preconnects to origins the page might never call, no preloads for below-the-fold imagery the browser will discover in time anyway.
