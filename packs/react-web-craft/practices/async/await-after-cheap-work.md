---
anti_patterns:
  - description: A handler awaits a remote flag or dataset before a cheap local check that would have skipped the branch, paying the I/O cost on paths that never use the value.
    id: react.async.io-before-its-guard
    name: Async work started before its guard
    severity: warn
applies_when: an async function branches on both cheap local conditions and awaited values, and the order that decides whether I/O is needed at all is being chosen
id: react.async.await-after-cheap-work
severity: warn
stage: implementation
tech_stack:
  - react
title: Run Cheap Checks Before Starting Async Work
---

## When to apply

Apply when an async function both can skip work based on cheap local data — props, already-loaded records, request metadata, configuration booleans — and awaits I/O for other conditions. Decide the order in which guards and awaits run.

## Guidance

Default order: cheap and synchronous first, awaited last. Two shapes cover most cases:

- A compound condition such as `remoteFlag && cheapCondition`: evaluate `cheapCondition` first. Only when it passes should the code await `remoteFlag`; on the failing side, the request is never started.
- Early returns: check local states — an already-closed ticket, a disabled feature, missing request metadata — before fetching the permissions, configuration, or flags used only past those checks.

Defer each await into the branch that consumes its value. An await at the top of a function body runs for every caller even when the only consumer is a branch most callers never enter; deferring removes that cost from every path that exits earlier.

Keep the original order when reordering would change behavior: when the "cheap" condition is itself expensive or I/O-bound; when the condition is computed from the awaited value; or when side effects must run in a fixed order, such as writing an audit note before a state transition. Deferral is an ordering decision, not a license to change semantics.

## Anti-pattern

A close-ticket handler awaits the audit-service feature flag before checking the ticket's local status. Most invocations arrive for already-closed tickets, so the flag request is paid and discarded on the dominant path; only the rare open-ticket path ever reads it.

## Why

I/O started on a path that discards it is pure latency on that path and pure load on the remote service. Guards that are local and synchronous cost nanoseconds; placing them first removes the I/O from every path they fail on, which matters most when the skipped branch is the common one or the deferred operation is expensive.

## Exceptions and boundaries

- "Cheap" means local and synchronous: in-memory fields, request metadata, literals from configuration already loaded. A condition that hits the network or a database is not cheap, however short it usually is.
- If the early exit itself needs data only the I/O provides, the await is the gate; do not reorder past it.
- Deferral must not change side-effect order observable elsewhere, and it must not strand a value that a concurrently started operation already depends on.
- This Practice is single-function ordering. Starting independent operations together is `react.async.parallel-independent-work`; isolating a slow subtree's timing in a component tree is `react.async.suspense-boundary-scope`.

## Example

Closing a support ticket checks the ticket's local status and a local feature flag before touching the network; the permission lookup runs only on the close path; the audit-note template is fetched only inside the branch that needs it.

```ts
type Ticket = { id: string; status: "open" | "closed"; channel: string };
type Session = { userId: string };

declare function getPermissions(userId: string): Promise<{ canClose: boolean }>;
declare function getAuditNoteTemplate(channel: string): Promise<string>;
declare function appendAuditNote(ticketId: string, note: string): Promise<void>;
declare function transitionToClosed(ticketId: string, userId: string): Promise<void>;

export async function closeTicket(
  ticket: Ticket,
  session: Session,
  options: { closingEnabled: boolean; requireAuditNote: boolean },
) {
  if (ticket.status === "closed") {
    return { closed: false as const, reason: "already-closed" as const };
  }
  if (!options.closingEnabled) {
    return { closed: false as const, reason: "disabled" as const };
  }

  const permissions = await getPermissions(session.userId);
  if (!permissions.canClose) {
    return { closed: false as const, reason: "forbidden" as const };
  }

  if (options.requireAuditNote) {
    const template = await getAuditNoteTemplate(ticket.channel);
    await appendAuditNote(ticket.id, template);
  }

  await transitionToClosed(ticket.id, session.userId);
  return { closed: true as const };
}
```

On the dominant already-closed path the function performs zero I/O. An ordinary close performs exactly the two required requests, plus a third only when policy demands the audit note — and the audit note still precedes the transition, so the required side-effect order is preserved.
