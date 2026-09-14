---
anti_patterns:
  - description: A JSX conditional guards rendering with `&&` on a numeric or string expression whose falsy domain includes renderable values, so `0` or `NaN` appears in the interface.
    id: react.rendering.falsy-left-operand-rendered
    name: Falsy left operand rendered by `&&`
    severity: warn
applies_when: a conditional render uses `&&` on an expression that can be `0`, `NaN`, or another falsy value that is not boolean
id: react.rendering.boolean-render-guards
severity: warn
stage: implementation
tech_stack:
  - react
title: Make Render Conditions Actually Boolean
---

## When to apply

Apply while writing or reviewing a conditional render of the form `{value && <Thing />}`. Decide whether the left operand's falsy domain contains only `false` and `undefined`, or also values JSX would render as content.

## Guidance

`&&` returns its left operand when falsy, and JSX renders numbers — so `{count && <Badge />}` shows the text `0` whenever the count is zero. When the condition's type is not genuinely boolean, reduce it to one first:

```tsx
{error.code > 0 && <ErrorBanner code={error.code} />}
```

An explicit comparison, `Boolean(...)`, or a ternary with an explicit `null` branch all work; the decision is that the author stated what the zero case means instead of leaving it to coercion. Empty strings and `NaN` follow the same rule: `NaN` renders as text, and an empty string renders an empty text node where `null` renders nothing at all.

Do not blanket-ban `&&`: when the left operand is already boolean — a flag, a comparison, a predicate call — `{isOpen && <Panel />}` is idiomatic and correct. The reviewable question is only "can this expression be a falsy value that renders?", which is a type question first and a style question never.

## Anti-pattern

An error banner is guarded by `{error.code && <Banner ... />}` where `code` is `0` on success and a nonzero code on failure. On every success path the page renders a stray `0` next to the form, because zero is falsy but still a number JSX displays.

## Why

JSX renders any child it is given, and `&&` passes the left operand through when it is falsy. Numbers and strings therefore leak into the tree as content. Collapsing the condition to a real boolean makes the render decision and the displayed value independent, so the zero case renders nothing rather than rendering zero.

## Exceptions and boundaries

- Genuinely boolean conditions need no change; adding `Boolean()` around a `bool` is noise.
- If zero is a meaningful display value, the ternary form makes that intent explicit — this Practice does not say zero must never render, only that its rendering must be chosen.
- TypeScript narrows some of this: a typed `boolean` left operand cannot leak a number, so the failure mode lives at untyped boundaries, API responses, and arithmetic results.
- This Practice governs render conditions only; it does not cover truthiness in non-JSX logic, where `if (items.length)` may be perfectly clear.

## Example

A search results header shows the match count only when there is at least one result. The comparison makes the zero case explicit; the previous `resultCount && ...` shape rendered the text `0` under the heading whenever the result set came back empty.

```tsx
function ResultsHeader({ resultCount }: { resultCount: number }) {
  return (
    <header>
      <h1>Results</h1>
      {resultCount > 0 && (
        <p>
          {resultCount} match{resultCount === 1 ? "" : "es"}
        </p>
      )}
    </header>
  );
}
```

An empty result set now renders the heading alone. If the product decision were to show "0 matches" instead, the ternary form would say so — the point of the Practice is that the zero case's meaning appears in the code rather than falling out of a coercion.
