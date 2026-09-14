---
anti_patterns:
  - description: A component expects a changing parent default to update local state, or overwrites a user's local choice every time the parent source refreshes.
    id: react.state.confused-default-and-override
    name: Default value and local override share one state meaning
    severity: warn
applies_when: an editable value should follow a changing external default until the user makes a local choice, after which that choice should remain authoritative
id: react.state.fallback-until-overridden
severity: warn
stage: design
tech_stack:
  - react
title: Preserve a Reactive Default Until the User Overrides It
---

## When to apply

Apply when an editable control starts from a parent or server-provided value that may change, while a user's local edit should take precedence after it is made. Decide whether the parent owns every value, the component only needs a mount-time initial value, or the component needs a local override that is absent until the user acts.

## Guidance

Choose the ownership contract before choosing the state shape:

- If the parent must determine the displayed value on every render, use a controlled value and change callback; do not add a competing local copy.
- If the source matters only when this component instance first mounts, use it as the state initializer and accept that later source changes do not automatically replace that state.
- If the control should track a changing source until the user edits it, store “no local override” separately from an actual value and display the local value when present, otherwise the current source. Updating the source then remains visible only while no override exists.

When the source changes to a different entity, make the override's lifetime explicit. Key the editor by stable entity identity or reset its local override deliberately; otherwise an edit to one record can leak into the next. If every value in the domain—including `undefined`—is a valid user choice, use an explicit tagged state instead of overloading `undefined` as the sentinel.

Do not use an Effect to continually copy the source into local state. That erases the distinction between inherited value and user intent and introduces a second update path.

## Anti-pattern

A results control initializes `pageSize` from a workspace policy and then expects later policy changes to update it. `useState` uses its initializer only for the initial mount, so later defaults are ignored. Copying each new default through an Effect would instead risk overwriting an explicit local selection.

## Why

A default, a controlled value, and a local override have different owners and update rules. Representing them as one ordinary state value hides those rules. Keeping the override absent until user action lets the current source remain authoritative before that point, while an explicit override preserves the user's choice afterward.

## Exceptions and boundaries

- For a fully controlled input, the parent remains authoritative even after each edit; this Practice does not recommend a local fallback layer.
- For a value intended to initialize once, later prop changes being ignored may be correct. Name the prop accordingly and use a key or explicit reset if identity changes should create a new state lifetime.
- A local override may need conflict resolution or save/rollback behavior when the source refreshes. This Practice defines display precedence, not synchronization, persistence, or merge policy.
- This is not a rule about storing pure values that can be calculated from current props/state, and it does not prescribe Effect design generally.

## Example

A workspace policy chooses 25, 50, or 100 results per page. Until a user chooses a local page size, the control follows the current policy value. An explicit selection—including any valid numeric option—wins until the user returns to the workspace default.

```tsx
import { useState } from "react";

type PageSize = 25 | 50 | 100;
const pageSizes: PageSize[] = [25, 50, 100];

export function PageSizeSetting({
  workspaceDefault,
}: {
  workspaceDefault: PageSize;
}) {
  const [override, setOverride] = useState<PageSize | undefined>(undefined);
  const pageSize = override ?? workspaceDefault;

  function handleChange(value: string) {
    const nextSize = pageSizes.find((size) => String(size) === value);
    if (nextSize !== undefined) setOverride(nextSize);
  }

  return (
    <fieldset>
      <legend>Result settings</legend>
      <label>
        Results per page
        <select
          value={pageSize}
          onChange={(event) => handleChange(event.currentTarget.value)}
        >
          {pageSizes.map((size) => (
            <option key={size} value={size}>
              {size}
            </option>
          ))}
        </select>
      </label>
      <button type="button" onClick={() => setOverride(undefined)}>
        Use workspace default
      </button>
    </fieldset>
  );
}
```

When `workspaceDefault` changes while `override` is `undefined`, the select follows the policy. After the user picks a size, the local value wins across later policy updates until the user selects the workspace default again. If the active settings scope changes to a different workspace, reset or isolate the override at that identity boundary.