---
anti_patterns:
  - description: A page statically imports large modules that only conditional or rare paths use, so every visitor downloads them before the first interaction whether or not they are ever opened.
    id: react.bundle.initial-bundle-carries-deferrable-code
    name: Deferrable code shipped in the initial bundle
    severity: warn
applies_when: a large module, component, or third-party library is needed only after the initial render or only on a conditional path, and the loading mechanism is being chosen
id: react.bundle.defer-heavy-loads
severity: warn
stage: implementation
tech_stack:
  - react
  - next
title: Defer Heavy Non-Critical Loads Until They Are Needed
---

## When to apply

Apply when a module, component, or third-party library is large relative to the page and is not needed for the initial paint or first interaction — an editor, a charting library, a map widget, an export dialog, an analytics script. Decide what loads up front and what loads when its feature is actually reached.

## Guidance

Defer two things at once: download cost and execution cost. A statically imported module joins the initial graph whether or not its feature renders; a dynamically imported one becomes a separate chunk fetched when first rendered or first called:

```tsx
const ReportBuilder = dynamic(() => import("./report-builder"), {
  loading: () => <p>Loading report builder…</p>,
});
```

Use the framework's component loader for components (in Next.js, `next/dynamic`), plain `import()` for data and non-component modules, and the framework's script-loading strategy for third-party scripts — analytics and chat widgets are usually scripts, not React components, and their loading strategy is separate from component code splitting. Note the trade-offs honestly: `ssr: false` keeps a component out of the server render (it must then be used from a client component in the App Router), so the delivered HTML will not contain it; that is usually right for widgets and wrong for content.

Keep import paths statically analyzable. A bundler cannot split what it cannot see: prefer an explicit map of loader functions over composing a path string, and prefer literal file-system paths in server code, where over-broad paths also widen Next.js output file tracing:

```ts
const REPORT_LOADERS = {
  sales: () => import("./reports/sales"),
  usage: () => import("./reports/usage"),
} as const;

const loadReport = REPORT_LOADERS[reportKind];
```

Correct one common misconception while deferring: a runtime guard such as `typeof window !== "undefined"` prevents a module from executing on the server; it does not remove the module from the server bundle. Bundle membership is decided by static imports and loader flags, not by runtime checks.

## Anti-pattern

A settings page statically imports a full charting library that only the admin-only usage dialog renders. Every visitor — including the ninety-nine percent who never open the dialog — downloads and parses the library before the page becomes interactive.

## Why

The initial bundle competes with the user's first impression; deferrable code in it is paid by everyone for the benefit of few. A dynamic import turns that cost into a separate chunk fetched exactly when the feature is reached, and explicit loader maps keep the bundler's reachable set narrow instead of forcing conservative over-inclusion.

## Exceptions and boundaries

- Do not defer code on the critical rendering or first-interaction path; a deferred critical component trades one cost for a worse one.
- Small modules are not worth splitting: the extra chunk request can cost more than the bytes saved; this is a cost boundary, not a style rule.
- `typeof window` guards and `ssr: false` answer different questions (execution versus inclusion); this Practice does not replace the browser-API and SSR boundary decisions of the rendering category.
- Deciding when a deferred chunk should be fetched ahead of its click is the `react.bundle.preload-on-intent` decision; excluding barrel-file entry points from imports is deferred in this Pack pending build-output evidence.

## Example

A settings page renders an admin-only usage section; the charting library lives behind the dynamic boundary and is fetched only when an admin expands it. Most visitors never download it.

```tsx
"use client";

import dynamic from "next/dynamic";

const UsageCharts = dynamic(() => import("./usage-charts"), {
  loading: () => <p>Loading charts…</p>,
});

export function SettingsPage({ isAdmin }: { isAdmin: boolean }) {
  return (
    <main>
      <h1>Settings</h1>
      <ProfileForm />
      {isAdmin && (
        <details>
          <summary>Usage</summary>
          <UsageCharts />
        </details>
      )}
    </main>
  );
}
```

The charting library ships in the `usage-charts` chunk, fetched when `UsageCharts` first renders — for most visitors, never. If several report kinds competed for the same slot, they would hang off a loader map like the one above, each entry a literal `() => import(...)` so the bundler can enumerate the split points instead of over-including a directory.
