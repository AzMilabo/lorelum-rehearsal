---
anti_patterns:
  - description: A materially expensive pure initializer is passed as a value expression, so JavaScript recomputes it on every render even though React only uses the initial value for that mounted state.
    id: react.state.eagerly-recomputes-initial-state
    name: Expensive initial state is recalculated on every render
    severity: warn
applies_when: a component builds a materially expensive initial local-state value that should be created once for that mounted state lifetime
id: react.state.lazy-initializer
severity: warn
stage: implementation
tech_stack:
  - react
title: Lazy-Initialize Expensive Mount-Time State
---

## When to apply

Apply when a component needs an editable local copy initialized from a stable input, and producing that initial value requires meaningful parsing, normalization, indexing, or other pure work. Decide whether the work is truly mount-time initialization or whether the state should instead follow changing inputs.

## Guidance

Pass an initializer function to `useState` when the initial calculation is costly enough to matter:

```tsx
const [rules, setRules] = useState(() => buildEditableRules(initialRules));
```

Passing a calculated value, such as `useState(buildEditableRules(initialRules))`, evaluates that expression whenever the component function runs. React ignores later initial values for the existing state, but JavaScript has already paid to compute them. The function form lets React call the initializer when it initializes that state rather than on each ordinary render.

Keep the initializer pure: it must not mutate inputs, perform I/O, schedule updates, or rely on being invoked exactly once. React Strict Mode may call an initializer twice in development to help expose impurities. Avoid the function form for cheap values such as a primitive or a small literal; lazy initialization is a cost boundary, not a default style rule.

Treat the initializer's inputs as a snapshot for that component's state lifetime. A new prop value does not rerun the initializer. If the edited state belongs to a different document or entity, give the editor a stable identity boundary (for example, a `key`) or define an explicit state-transition/reset path. If the value must always reflect the current prop, do not copy it into local state just to use a lazy initializer; choose a controlled or reactive-override contract instead.

## Anti-pattern

A rule editor parses hundreds of expressions into token arrays as an argument to `useState`. Editing one checkbox rerenders the editor and repeats the full parse, while the resulting arrays are discarded because the component already has state. Replacing the expression with an impure initializer that reads storage or writes a cache is not a safe fix: initialization must remain pure and compatible with render behavior.

## Why

The argument expression to a Hook is evaluated by JavaScript during every component invocation. React's one-time use of the initial value does not make that expression lazy. Supplying a function separates the initialization calculation from subsequent renders, while the explicit lifetime boundary prevents the optimization from being mistaken for synchronization with changing props.

## Exceptions and boundaries

- If construction is cheap, prefer the simpler direct initial value.
- If a derived value should change whenever its inputs change, calculate it from current inputs or use an appropriate memoization decision; lazy state initialization intentionally does not track those changes.
- If the user must edit a snapshot of an input, decide separately how identity changes replace that snapshot. Do not reset it through an Effect merely because props changed.
- Initializers must be pure. Generate random identifiers, read browser-only storage, perform requests, or apply other side effects outside the initializer and pass a suitable result through the owning event, data-loading, or component-lifecycle path.

## Example

A rule editor turns the selected document's rule expressions into editable token lists. The document key defines when a new editor state lifetime begins; the initializer normalizes the selected document once per mount, and subsequent checkbox updates reuse that state.

```tsx
import { useState } from "react";

type RuleSource = {
  id: string;
  expression: string;
  enabled: boolean;
};

type EditableRule = RuleSource & { tokens: string[] };

type RuleDocument = {
  id: string;
  rules: readonly RuleSource[];
};

function tokenize(expression: string): string[] {
  return expression.match(/\b(?:and|or|not)\b|[A-Za-z_]\w*|\d+(?:\.\d+)?|==|!=|<=|>=|[()<>!]/gi) ?? [];
}

function buildEditableRules(source: readonly RuleSource[]): EditableRule[] {
  return source.map((rule) => ({
    ...rule,
    tokens: tokenize(rule.expression),
  }));
}

export function DocumentRuleEditor({ document }: { document: RuleDocument }) {
  return <RuleEditor key={document.id} initialRules={document.rules} />;
}

function RuleEditor({ initialRules }: { initialRules: readonly RuleSource[] }) {
  const [rules, setRules] = useState(() => buildEditableRules(initialRules));

  function toggleRule(id: string) {
    setRules((current) =>
      current.map((rule) =>
        rule.id === id ? { ...rule, enabled: !rule.enabled } : rule,
      ),
    );
  }

  return (
    <ul>
      {rules.map((rule) => (
        <li key={rule.id}>
          <label>
            <input
              type="checkbox"
              checked={rule.enabled}
              onChange={() => toggleRule(rule.id)}
            />
            {rule.tokens.join(" ")}
          </label>
        </li>
      ))}
    </ul>
  );
}
```

The key means switching to a different document creates a new editor instance; an ordinary rerender or same-document prop refresh does not rebuild the local editable state. If same-identity rule updates must merge into the editor, define that state transition explicitly instead of expecting the initializer to run again.

## Further reading

- [React `useState`](https://react.dev/reference/react/useState) — initial values, initializer functions, and Strict Mode purity checks.
- [Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state) — how component identity and keys delimit state lifetime.