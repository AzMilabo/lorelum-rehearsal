---
anti_patterns:
  - description: A component accumulates mode booleans such as `isEditing` and `isThread`, so the set of supported combinations doubles with each flag and the render body dissolves into conditional branches no caller can predict.
    id: react.composition.boolean-mode-flags
    name: Boolean mode flags on one component
    severity: warn
applies_when: one component's API grows booleans or optional fields to select between modes that need different data, sections, or actions
id: react.composition.explicit-variants-over-flags
severity: warn
stage: design
tech_stack:
  - react
  - typescript
title: Prefer Explicit Component Variants to Boolean Mode Flags
---

## When to apply

Apply when a component is asked to support a new mode — create, edit, forward, reply — and the proposed change is another boolean prop plus the branches it needs. Decide whether the modes should be separate named components sharing parts.

## Guidance

Each mode boolean doubles the states a component can be in, and the modes usually need different data anyway (`dmId` versus `channelId`). Split the modes into named variants that each take exactly what they need, and let them share the pieces that are genuinely common:

```tsx
export function ThreadComposer({ channelId }: { channelId: string }) {
  return (
    <ComposerFrame>
      <ComposerInput />
      <AlsoSendToChannelField channelId={channelId} />
      <ComposerActions submitLabel="Post" />
    </ComposerFrame>
  );
}

export function EditComposer({ messageId }: { messageId: string }) {
  return (
    <ComposerFrame>
      <ComposerInput />
      <ComposerActions submitLabel="Save" cancel />
    </ComposerFrame>
  );
}
```

The shared parts stay shared — a frame, an input, an action row — while each variant's signature documents what it does and requires. The reader of a call site sees `<EditComposer messageId=... />` instead of reverse-engineering a truth table.

Two boundaries keep this from becoming dogma. First, splitting is not the only honest shape: when the modes are genuinely variations of one public surface, a discriminated-union props contract gives the same per-mode data checking inside one component — see `react.composition.model-mutually-exclusive-props` for that contract. The split decision is about composition and data, the union decision about a retained single API. Second, independent options are not modes: `disabled`, `compact`, or `showAvatar` are facets that combine freely and belong as props; only mutually exclusive modes earn variants.

## Anti-pattern

A message composer takes `isThread`, `isEditing`, and `isForwarding` plus the optional IDs each mode needs. Every mode combination compiles, three of them are nonsense, the render body is a branch per flag, and adding "schedule for later" mode means touching all of them.

## Why

Variants push the mode choice to the type level and the call site, where wrong combinations fail to compile or fail to be written at all; booleans push it into runtime branches where every combination is representable. Shared parts keep the variants from forking into copies, so the split costs composition, not duplication.

## Exceptions and boundaries

- A single-mode component should not be pre-split into variants for imagined futures; the split earns its keep when a second mode with different data actually arrives.
- Variant families need deliberate shared-part design; without it, splitting degrades into copy-paste, which is worse than flags.
- Cross-variant state that outlives one variant's mount (drafts, selections) is a state-ownership decision — see `react.state.share-one-owner` — not a props shape.
- This Practice does not prescribe the shared-parts mechanism (compound parts, context provider, plain props); choose the simplest one that keeps the variants honest.

## Example

The variants share `ComposerFrame`, `ComposerInput`, and `ComposerActions`. Adding a scheduled mode later is one more variant — not a new flag rippling through every existing branch:

```tsx
function ScheduledComposer({ channelId, runAt }: { channelId: string; runAt: Date }) {
  return (
    <ComposerFrame>
      <ComposerInput />
      <ScheduleFields runAt={runAt} />
      <ComposerActions submitLabel="Schedule" cancel />
    </ComposerFrame>
  );
}
```

Call sites read the mode from the component name — `<ThreadComposer channelId=... />`, `<EditComposer messageId=... />`, `<ScheduledComposer channelId=... runAt=... />` — and each takes exactly its own data. A scheduled post has a `runAt`; the boolean version of this component would have carried it as an optional prop that meant nothing outside one mode, and the empty-check for it would have lived in a branch.
