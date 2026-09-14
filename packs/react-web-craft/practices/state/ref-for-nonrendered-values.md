---
anti_patterns:
  - description: A value is moved from state to a ref because it changes often even though the rendered interface must show each change.
    id: react.state.ref-hides-rendered-change
    name: Ref used for a value that must update the UI
    severity: warn
applies_when: event or imperative logic needs non-rendered mutable bookkeeping to persist across React renders, and the value is not being used to produce visible output or signal work to an Effect
id: react.state.ref-for-nonrendered-values
severity: warn
stage: design
tech_stack:
  - react
title: Use Refs for Non-Rendered Per-Instance Bookkeeping
---

## When to apply

Apply when a later event or imperative operation needs bookkeeping that survives renders. Decide whether the value itself belongs in rendered output or is private per-instance memory that the UI does not need to display.

## Guidance

Use state when a change should update rendered output. Use a ref when private bookkeeping must persist between renders and a later event or imperative operation will read it, but rendering does not need the bookkeeping value. Changing `ref.current` does not schedule a render, so a ref cannot by itself keep visible output in sync.

Choose by the value's role, not only its update frequency. A frequently changing value that drives what users see still needs an update mechanism that produces the intended UI. Do not switch it to a ref merely to suppress renders, and do not write DOM styles from an event handler as a substitute for deciding how the interface should render. Do not replace a state flag with a ref just to make an Effect run: ref mutation schedules no render for the Effect to observe. A command caused by a user event belongs in that event handler; render-driven external synchronization needs a render-visible input.

## Anti-pattern

An upload percentage is stored in a ref to avoid renders, then JSX reads `progressRef.current` expecting the visible progress label to advance. Updating the ref alone leaves the rendered label at its last committed value.

## Why

React state participates in rendering; a ref is mutable storage that persists for the component instance without notifying React when it changes. Confusing those jobs either causes needless renders for private bookkeeping or leaves visible UI stale.

## Exceptions and boundaries

- Use state, an external animation system, or another explicitly chosen rendering mechanism when changing data must be visible. This Practice does not prescribe animation architecture or promise a performance improvement.
- Refs also hold DOM nodes and are used for imperative APIs; this Practice addresses only the state-versus-ref choice for non-DOM values.
- A ref used inside a long-lived event listener to read changing values is a separate closure/Effect decision. This Practice does not recommend latest-value refs or Effect Events.

- A value that is not displayed now but later affects a user-visible action may still be appropriate in a ref if its lifetime and intended read time are explicit.

## Example

A telemetry button suppresses duplicate local pings that arrive within one second. The last accepted instant affects a later event decision but is never shown in the interface, so it can live in a ref. This is only a per-component-instance sampling guard, not a server-side rate limit.

```tsx
import { useRef } from "react";

export function CheckpointButton({
  recordCheckpoint,
}: {
  recordCheckpoint: () => void;
}) {
  const lastRecordedAtRef = useRef<number | null>(null);

  function handleClick() {
    const now = performance.now();
    const previous = lastRecordedAtRef.current;
    if (previous !== null && now - previous < 1000) return;

    lastRecordedAtRef.current = now;
    recordCheckpoint();
  }

  return (
    <button type="button" onClick={handleClick}>
      Record checkpoint
    </button>
  );
}
```

Changing `lastRecordedAtRef.current` does not redraw the button. If the user must see a cooldown, store the visible cooldown state in React state as well; the ref alone cannot update rendered text or disabled controls.