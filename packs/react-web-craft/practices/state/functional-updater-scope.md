---
anti_patterns:
  - description: A callback switches to a functional updater to remove one state dependency, then also omits other changing props or state that the callback still reads.
    id: react.state.updater-hides-other-dependencies
    name: Updater mistaken for a live read of every capture
    severity: warn
applies_when: a useCallback uses a functional state updater to remove the state it updates from its dependency list, but the callback still reads another reactive value
id: react.state.functional-updater-scope
severity: warn
stage: implementation
tech_stack:
  - react
title: A Functional Updater Does Not Refresh Other Captures
---

## When to apply

Apply while refactoring a `useCallback` to pass an updater function so the callback no longer reads the state value it updates. Decide whether other props, state, or render-local values read by that callback must remain dependencies. This Practice narrows one dependency decision; it is not a general guide to every stale-closure problem.

## Guidance

Use an updater when the next value must be calculated from the pending value of that same state variable. The updater receives that state value; it does not make the surrounding callback run again or refresh the other values captured when that callback was created. Keep the updater pure: return the next state without mutating the existing value or causing side effects.

When a `useCallback` also reads another reactive value, keep that value in its dependency list. Remove only the state dependency that the callback no longer reads because its updater receives the pending state. If callback identity does not need to be stable, do not add `useCallback` solely to manage dependencies.

Decide which moment a captured value is supposed to represent. A workspace ID passed or captured when the user starts an action may intentionally identify that action. If the callback must follow the currently rendered workspace, make that value reactive in the callback or pass the intended ID explicitly; an updater for the message list cannot choose that policy for you.

## Anti-pattern

A memoized “mark active workspace read” callback updates `messages` through `setMessages(current => ...)`, then uses an empty dependency list even though the updater also compares each message with `activeWorkspaceId`. The updater gets the latest pending `messages` value, but `activeWorkspaceId` is still the value captured by the callback's render. After switching workspaces, the callback can continue marking messages for the old workspace.

## Why

A state updater supplies the pending value for one state variable. A memoized callback is still the function created by a particular render, and its other reactive reads do not become live just because one state read moved into an updater. Omitting those dependencies can preserve old behavior when the callback is called after those values change.

## Exceptions and boundaries

- If the callback reads no changing reactive value other than the state being updated, the updater can remove that state's dependency. Stable setters and module-scope helpers do not need to be added as dependencies.
- If the next state is a replacement already known from the event or another source, pass that value directly; an updater is not required just because it is available.
- If the operation should use the value from the moment it was initiated, capture or pass that value deliberately. If it should use a later value, arrange for the callback to receive or read that later value through the appropriate data flow. Do not treat “latest” as automatically correct.
- If `useCallback` is used, its dependency list must reflect the other reactive values the callback reads; stable function identity does not make those values live. A stable event handler that must read a changing value through a ref or another mechanism is a different decision; this Practice does not prescribe a “latest value” workaround.

## Example

The callback below updates a message list from its pending state and also filters by the active workspace. Its dependency list therefore includes `activeWorkspaceId`; the updater only replaces the need to read `messages` from the render snapshot. This TSX sample assumes the automatic JSX runtime; follow the JSX runtime configured by the project.

```tsx
import { useCallback, useState } from "react";

type Message = {
  id: string;
  workspaceId: string;
  read: boolean;
};

const initialMessages: Message[] = [
  { id: "m-1", workspaceId: "north", read: false },
  { id: "m-2", workspaceId: "south", read: false },
];

export function WorkspaceInbox({
  activeWorkspaceId,
}: {
  activeWorkspaceId: string;
}) {
  const [messages, setMessages] = useState(initialMessages);

  const markActiveWorkspaceRead = useCallback(() => {
    setMessages((currentMessages) =>
      currentMessages.map((message) =>
        message.workspaceId === activeWorkspaceId
          ? { ...message, read: true }
          : message,
      ),
    );
  }, [activeWorkspaceId]);

  return (
    <section>
      <button type="button" onClick={markActiveWorkspaceRead}>
        Mark this workspace read
      </button>
      <ul>
        {messages.map((message) => (
          <li key={message.id}>
            {message.workspaceId}: {message.read ? "read" : "unread"}
          </li>
        ))}
      </ul>
    </section>
  );
}
```

If this callback filtered only by a fixed operation argument and the pending `messages` value, it could receive the workspace ID as an argument and avoid capturing that prop. Here the callback's contract is specifically “mark the workspace active for this render,” so the workspace ID remains a dependency.