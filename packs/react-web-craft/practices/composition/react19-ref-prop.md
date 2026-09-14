---
anti_patterns:
  - description: A codebase forwards refs through `forwardRef` wrappers (or strips them entirely) by habit, so ref support is inconsistent and React 19 upgrades leave pointless indirections that new code copies.
    id: react.composition.ref-forwarding-by-habit
    name: Ref forwarding decided by habit instead of React line
    severity: warn
applies_when: a component should let its caller reach an underlying DOM node or imperative handle, and the ref-passing mechanism is being chosen for a target React version
id: react.composition.react19-ref-prop
severity: warn
stage: implementation
tech_stack:
  - react
title: Pass ref as a Prop on React 19; Keep forwardRef for Older Lines
---

## When to apply

Apply when writing or reviewing a component that needs to expose its DOM node or an imperative handle to callers — inputs that deserve focus, dialogs that need positioning. Decide how `ref` travels, given the React version line the project runs.

## Guidance

On React 19, `ref` is an ordinary prop: declare it, pass it through, no wrapper:

```tsx
function TextField({ ref, label, ...props }: InputProps & { ref?: React.Ref<HTMLInputElement> }) {
  return (
    <label>
      <span>{label}</span>
      <input ref={ref} {...props} />
    </label>
  );
}
```

`forwardRef` is not wrong on React 19 — it still works — but new components gain nothing from it, and its indirection (a second function boundary, a separate type parameter) is now pure ceremony. Treat it as a migration decision, not a style war:

- A project on React 19: write `ref`-as-prop for new components; migrate `forwardRef` wrappers opportunistically where the file is already being touched.
- A project supporting React 18 and 19 from one source line, or publishing a library with a 18-compatible peer range: keep `forwardRef`, which works on both; do not ship 19-only ref props from that line.
- Never hand-roll the alternative — passing a `domNodeRef`-style custom prop alongside a `ref` requirement — when one of the two built-in mechanisms covers the version line.

The same version-line thinking governs the rest of the ref surface: imperative handles via `useImperativeHandle` follow the identical mechanism as the `ref` they attach to, so the decision above covers them together.

## Anti-pattern

A component library pins React 18 compatibility, but a contributor "modernizes" a text input to React 19 ref-as-prop and removes its `forwardRef`. On React 18 every call site's `ref` silently stops reaching the input — focus management breaks with no type error at the call site and no runtime crash, just dead behavior.

## Why

The two mechanisms are version-locked: ref-as-prop is invisible to React 18, and `forwardRef` is noise on React 19. Choosing by the project's supported React line — and keeping that line explicit in the library's peer dependencies — is what keeps ref support working and uniform while the ecosystem finishes its 19 migration.

## Exceptions and boundaries

- Components that never expose a node or handle should not accept a ref at all; this Practice decides the mechanism, not whether the capability exists.
- Higher-order components and composition utilities that must be agnostic to the wrapped component's version may need to support both mechanisms during a migration window; scope that shim tightly and delete it when the floor rises to 19.
- Reading another component's DOM node through a grabbed ref — as opposed to accepting one passed down — is a component-boundary decision this Practice does not cover.

## Example

A search panel focuses its combobox the moment it opens. The component exposes its input on the React 19 line; the call site neither knows nor cares how the ref traveled.

```tsx
import { useEffect, useRef } from "react";

function Combobox({ ref, label, ...rest }: InputProps & { ref?: React.Ref<HTMLInputElement> }) {
  return (
    <label>
      <span>{label}</span>
      <input ref={ref} role="combobox" {...rest} />
    </label>
  );
}

function SearchPanel() {
  const queryRef = useRef<HTMLInputElement>(null);
  useEffect(() => queryRef.current?.focus(), []);

  return <Combobox ref={queryRef} label="Search" />;
}
```

On a library line that still supports React 18, the same component is written with `forwardRef` and the call site is byte-for-byte identical — which is exactly why the mechanism choice stays inside the component and its version line, and never leaks into callers.
