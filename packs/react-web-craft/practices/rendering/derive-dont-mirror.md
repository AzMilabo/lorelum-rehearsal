---
anti_patterns:
  - description: A value computed from current props and state is copied into another piece of state and refreshed by an effect, adding an extra render per change and a version that can drift from its inputs.
    id: react.rendering.mirroring-derived-value
    name: Derived value mirrored into state
    severity: warn
applies_when: a render value can be computed from current props or state, and a decision is needed between deriving it and storing it
id: react.rendering.derive-dont-mirror
severity: warn
stage: implementation
tech_stack:
  - react
title: Derive Render Values; Do Not Mirror Props into State
---

## When to apply

Apply when you catch a component storing a value in state that is fully determined by current props or state — and typically re-writing it from an effect. Decide whether the value has an independent life of its own or is just a calculation wearing a state coat.

## Guidance

If the value is a pure function of current props and state, compute it during render and delete the state and the effect:

```tsx
const totalWithTax = subtotal + subtotal * taxRate;
```

The stored version is worse in every direction: it renders once behind the change (the effect runs after paint), it can drift when a code path updates the inputs without re-running the mirror, and it doubles the state to keep in sync.

Three cases do look like mirroring but are not, and each has its own decision:

- The value starts from a prop but the user edits it: that is an editable draft with its own lifecycle; initialize from the prop and own it thereafter.
- The value must reset when an identity changes (a form bound to a different record): reset explicitly with a `key` at the identity boundary, not by watching props in an effect.
- The computation is expensive: derive anyway and memoize the calculation if measurement says so — memoization is an optimization on a correct shape, never the fix for a wrong one.

The same test applies to effects that "react" to prop changes by setting state: an effect may synchronize with external systems, but the props-to-state direction inside one render tree is what this Practice removes.

## Anti-pattern

An invoice editor keeps `total` in state and an effect recomputes it from line items and tax rate on every change. Toggling a line item paints the old total for one frame; a second developer adds a bulk-import path that sets the items without firing the effect, and the displayed total silently diverges from the items it summarizes.

## Why

A stored derived value creates a second authority for one fact, and the effect that maintains it runs after the render the user sees. Deriving during render keeps a single authority, makes the value impossible to drift from its inputs, and removes the extra render — the state that remains is exactly the state the user or an external system owns.

## Exceptions and boundaries

- Values with genuine user ownership or independent lifetimes stay in state; this Practice does not shrink state, it removes duplicated state.
- Deriving values during render must stay pure: no side effects, no writes, even for expensive derivations.
- Memoizing a derivation with `useMemo` trades memory and dependency hygiene for computation; whether that trade pays is a measured, Compiler-aware decision outside this Practice.
- Reading and adjusting DOM in effects for layouts React cannot express is external-system synchronization, not value mirroring.

## Example

An invoice summary derives both figures from its inputs during render. There is no second state to fall out of sync and no effect to forget.

```tsx
function InvoiceSummary({ lines, taxRate }: { lines: InvoiceLine[]; taxRate: number }) {
  const subtotal = lines.reduce((sum, line) => sum + line.amount, 0);
  const totalWithTax = subtotal + subtotal * taxRate;

  return (
    <dl>
      <dt>Subtotal</dt>
      <dd>{subtotal}</dd>
      <dt>Total with tax</dt>
      <dd>{totalWithTax}</dd>
    </dl>
  );
}
```

Adding, removing, or editing a line updates both figures in the same render that shows the change. If profiling ever shows the reduction matters on very large invoices, `useMemo` wraps this exact derivation — the shape of the code does not change, only its caching.
