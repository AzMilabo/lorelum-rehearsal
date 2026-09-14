---
anti_patterns:
  - description: A "use server" mutation performs no authentication or authorization of its own, relying on middleware, layout, or page guards that a directly invoked endpoint bypasses.
    id: react.server.action-trusts-ui-guards
    name: Server Action trusting page-level guards
    severity: warn
applies_when: a server-side mutation is exposed through a server action, and the placement of its authentication and authorization checks is being decided
id: react.server.authorize-server-actions
severity: warn
stage: implementation
tech_stack:
  - react
  - next
title: Authorize Inside Every Server Action
---

## When to apply

Apply when writing or reviewing any function marked `"use server"`. Decide where its authentication (who is calling) and authorization (may this caller act on this target) checks live.

## Guidance

Treat every server action as a public endpoint, because it is one: the framework turns it into an addressable handler that any client can invoke with any arguments, regardless of what UI rendered it. Middleware, layout guards, and page-level checks protect rendering paths — they do not run for a directly posted action call. Each action therefore verifies its own contract before touching data:

```ts
"use server";

export async function archiveProject(projectId: string) {
  const session = await verifySession();
  if (!session) {
    throw new Error("Unauthenticated");
  }

  const project = await getProject(projectId);
  if (!project || !canAdminister(session.user, project)) {
    throw new Error("Forbidden");
  }

  await archiveProjectRecord(projectId);
  return { ok: true };
}
```

Three checks per action, in order: authenticate the caller; authorize this caller for this target — the target's own ownership or membership, not just a role; validate the input, because action arguments arrive from untrusted wire data no matter what types the client passed. Keep the checks inside the action itself rather than in a shared wrapper that some actions might bypass by accident; a wrapper is acceptable only when every mutation provably goes through it.

Do not let the UI imply security that the action does not enforce: a hidden or disabled button is a convenience, not a boundary. And do not treat this as unique to destructive operations — reads that leak per-user data through actions need the same scrutiny.

## Anti-pattern

An admin page checks the viewer's role in its layout, and its "use server" `archiveProject` action performs no checks of its own. The layout guard never runs when the action is invoked directly with a hand-crafted request, so any authenticated user archives any project.

## Why

The action's handler is reachable independently of the page that rendered it; its security boundary is wherever its own first check sits, and nowhere earlier. Placing authentication, authorization, and validation inside each action keeps the enforcement point identical to the execution point, which is the only placement an attacker cannot route around.

## Exceptions and boundaries

- A shared action wrapper or framework middleware may perform the authenticate step, but only when coverage of every action is guaranteed by construction; authorization against the specific target cannot be centralized because only the action knows its target.
- Input validation is input validation, not authorization: a valid-shaped ID the caller does not own still fails the ownership check.
- Framework-specific mechanisms (Next.js's guidance to treat actions as public endpoints) fix the general contract; this Practice does not replace a project's chosen auth library or error-reporting conventions.
- Rate limiting, auditing, and CSRF posture for action endpoints are deployment-level decisions outside this Practice.

## Example

An invite action checks all three legs inside itself: who is calling, whether that caller administers this specific space, and whether the wire input is well-formed. The declared helpers stand in for the project's data layer.

```ts
"use server";

declare function verifySession(): Promise<{ userId: string } | null>;
declare function getMembership(
  spaceId: string,
  userId: string,
): Promise<{ role: "owner" | "member" } | null>;
declare function insertInvite(input: {
  spaceId: string;
  email: string;
  invitedBy: string;
}): Promise<void>;

export async function inviteMember(spaceId: string, email: string) {
  const session = await verifySession();
  if (!session) throw new Error("Unauthenticated");

  const membership = await getMembership(spaceId, session.userId);
  if (!membership || membership.role !== "owner") {
    throw new Error("Forbidden");
  }

  if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(email)) {
    throw new Error("Invalid email");
  }

  await insertInvite({ spaceId, email, invitedBy: session.userId });
  return { ok: true };
}
```

The dialog that renders this action can be hidden, and the admin layout that shows it can be guarded — neither protects the endpoint. A member (not owner) calling it directly fails the ownership check; a well-formed email for a space the caller does not administer fails the same way.
