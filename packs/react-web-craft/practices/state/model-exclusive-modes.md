---
anti_patterns:
  - description: Several booleans encode one mutually exclusive workflow phase, allowing combinations such as “saving and saved” or “idle with an error.”
    id: react.state.independent-flags-for-exclusive-modes
    name: Independent flags for mutually exclusive modes
    severity: warn
applies_when: multiple state flags describe mutually exclusive modes of the same UI flow and permit contradictory combinations
id: react.state.model-exclusive-modes
severity: warn
stage: design
tech_stack:
  - react
  - typescript
title: Represent Mutually Exclusive UI Modes with One State
---

## When to apply

Apply when several booleans or optional fields jointly describe one workflow phase—for example, a save can be idle, in progress, complete, or failed. Decide whether these values may truly coexist or whether the current state shape permits impossible combinations.

## Guidance

If exactly one mode is active at a time, represent it with one status value. In TypeScript, a discriminated union can attach data that belongs only to a particular mode, such as an error message to the failed variant. Transition by replacing the whole state value so the next mode and its associated data change together.

Keep independent facts separate when they can be true at the same time and have different owners or lifetimes. Do not merge unrelated toggles merely to reduce the number of state variables, and do not use this Practice to decide whether a value can be derived from other state.

## Anti-pattern

A label manager keeps `isCreating` and `editingLabelId` as separate state values. Nothing prevents both from being active, so the UI can open a create-label editor and a rename editor at the same time even though the product allows only one label editor.

## Why

Independent flags can encode every combination, including combinations the product does not allow. One tagged mode can make “closed,” “creating,” and “renaming this label” exclusive, while carrying an ID only for the rename mode.

## Exceptions and boundaries

- Keep separate state for facts that can coexist, such as “email alerts enabled” and “SMS alerts enabled.” Those are independent choices, not alternative phases.
- This Practice concerns the shape of one state concept. It does not require `useReducer`, a state-machine library, or a particular transition architecture.
- It does not address redundant render-derived values, prop synchronization, request deduplication, or races between concurrent operations.
- If a workflow has complex transitions or side effects, model those separately; a union makes valid states explicit but does not by itself enforce every transition rule.

## Example

A label manager supports either creating a label or renaming one existing label, never both editors at once. One tagged state stores the active mode and the data needed for that mode.

```tsx
import { useState } from "react";

type Label = { id: string; name: string };
type EditorState =
  | { kind: "closed" }
  | { kind: "create"; name: string }
  | { kind: "rename"; id: string; name: string };

type LabelManagerProps = {
  labels: Label[];
  createLabel: (name: string) => void;
  renameLabel: (id: string, name: string) => void;
};

export function LabelManager({
  labels,
  createLabel,
  renameLabel,
}: LabelManagerProps) {
  const [editor, setEditor] = useState<EditorState>({ kind: "closed" });

  function updateName(name: string) {
    setEditor((current) =>
      current.kind === "closed" ? current : { ...current, name },
    );
  }

  function saveEditor() {
    if (editor.kind === "create") createLabel(editor.name);
    if (editor.kind === "rename") renameLabel(editor.id, editor.name);
    setEditor({ kind: "closed" });
  }

  return (
    <section>
      <button
        type="button"
        onClick={() => setEditor({ kind: "create", name: "" })}
      >
        New label
      </button>
      <ul>
        {labels.map((label) => (
          <li key={label.id}>
            {label.name}{" "}
            <button
              type="button"
              onClick={() =>
                setEditor({ kind: "rename", id: label.id, name: label.name })
              }
            >
              Rename {label.name}
            </button>
          </li>
        ))}
      </ul>
      {editor.kind !== "closed" && (
        <fieldset>
          <legend>{editor.kind === "create" ? "New label" : "Rename label"}</legend>
          <label>
            Name
            <input
              value={editor.name}
              onChange={(event) => updateName(event.currentTarget.value)}
            />
          </label>
          <button type="button" onClick={saveEditor}>
            Save
          </button>
          <button
            type="button"
            onClick={() => setEditor({ kind: "closed" })}
          >
            Cancel
          </button>
        </fieldset>
      )}
    </section>
  );
}
```

When `editor.kind` is `closed`, it has no name or label ID. A `create` mode has a draft name but no ID; a `rename` mode carries the ID it will update. If creating and renaming were both allowed at once, they would be independent modes and should not be combined this way.