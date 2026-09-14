---
anti_patterns:
  - description: A React date-selection component uses optional fields for single dates and ranges, so callers can provide incomplete or conflicting selection data and callbacks.
    id: react.composition.optional-props-for-exclusive-modes
    name: Optional props for mutually exclusive modes
    severity: warn
applies_when: one React TypeScript component API supports mutually exclusive modes with different required props or callbacks, and its boolean or optional fields let callers describe a combination the UI cannot support
id: react.composition.model-mutually-exclusive-props
severity: warn
stage: design
tech_stack:
  - react
  - typescript
title: Model Mutually Exclusive React Props with a Discriminated Union
---

## When to apply

Apply when one public React TypeScript component API must represent mutually exclusive modes that require different props or callbacks, and boolean or optional fields let callers describe a combination the UI does not support. First confirm that one public component surface is still appropriate: the modes may require different data and behavior, but should not need independently owned state, separate providers, unrelated composition, or distinct extension points. If they do, expose named components instead of forcing both variants through one union.

## Guidance

Give each mode a literal discriminator such as `mode: "single"` or `mode: "range"`. Define one union member per mode and require that mode's inputs and callbacks. Use `?: never` for known cross-mode props that should reject supplied values, including values arriving through typed object spreads. Narrow on the discriminator with an exhaustive `switch`; share only rendering that genuinely belongs to both modes. Keep independent options such as `disabled` outside the mode union.

`?: never` is a compile-time constraint, not runtime validation. With `strictNullChecks` enabled and `exactOptionalPropertyTypes` disabled, an optional `never` property still accepts explicitly supplied `undefined`; non-`undefined` conflicting values are rejected, but property absence is not guaranteed. `exactOptionalPropertyTypes` requires `strictNullChecks` and tightens assignments only in projects whose `tsconfig` enables it; a reusable component library cannot require its consumers to use that option. Use the discriminator as the runtime mode signal. If runtime behavior depends on conflicting properties being absent, validate or normalize the input at the boundary rather than trusting the type.

Assume the automatic JSX runtime (`jsx: "react-jsx"`) and the corresponding React JSX type declarations. With the classic transform, include React in scope as required by the project.

```tsx
type SingleDateProps = {
  mode: "single";
  selectedDate: Date;
  onSelect: (date: Date) => void;
  startDate?: never;
  endDate?: never;
  onSelectRange?: never;
};

type DateRangeProps = {
  mode: "range";
  startDate: Date;
  endDate: Date;
  onSelectRange: (range: { start: Date; end: Date }) => void;
  selectedDate?: never;
  onSelect?: never;
};

export type DateSelectionProps = SingleDateProps | DateRangeProps;

function assertNever(_value: never): never {
  throw new Error("Unsupported date selection mode");
}

export function DateSelection(props: DateSelectionProps) {
  switch (props.mode) {
    case "single":
      return (
        <section>
          <p>Selected date: {props.selectedDate.toLocaleDateString()}</p>
          <button type="button" onClick={() => props.onSelect(props.selectedDate)}>
            Confirm date
          </button>
        </section>
      );
    case "range":
      return (
        <section>
          <p>
            Selected range: {props.startDate.toLocaleDateString()} – {props.endDate.toLocaleDateString()}
          </p>
          <button
            type="button"
            onClick={() => props.onSelectRange({ start: props.startDate, end: props.endDate })}
          >
            Confirm range
          </button>
        </section>
      );
    default:
      return assertNever(props);
  }
}
```

The `mode` field narrows the props for each branch. The single-date variant cannot provide a usable range, while the range variant requires both endpoints and its range callback. The `switch` makes an added union member a type-checking concern; its fixed fallback error is only a defensive failure for values that bypass the TypeScript contract, not a substitute for validating external input.

## Anti-pattern

A calendar component accepts `mode?`, `selectedDate?`, `startDate?`, `endDate?`, `onSelect?`, and `onSelectRange?`, with every field optional. A caller can provide only one range endpoint, omit the callback required by the selected mode, or combine a single date with range-only values. The implementation must handle a missing mode or incomplete data and guard reads of optional fields, instead of receiving a variant whose required data is statically known.

## Why

Independent optional fields describe many combinations, including states the UI cannot use. A tagged union ties each mode to its required inputs and callback, so TypeScript narrows the props by the discriminator and catches invalid combinations at checked call sites. An exhaustive `switch` also makes a newly added mode visible to the implementation.

## Exceptions and boundaries

Use separate named components when modes need independent state ownership, separate providers, substantially different composition, or distinct extension points; a union should not force unrelated UI through one public component. TypeScript does not validate JSON, URL state, JavaScript callers, or other untrusted runtime input, so parse or validate those values before constructing `DateSelectionProps`. Do not infer the mode from whether a cross-mode property exists; branch on the explicit discriminator. See the `exactOptionalPropertyTypes` caveat above when checked assignments must also reject explicitly supplied `undefined`.

## Example

A scheduling interface offers a single-day selection and a date-range selection through the same calendar surface. Both variants share the calendar's public component boundary, but each needs different selection data and a different callback. The code above is a compact prop-contract example; a production calendar still needs to implement date navigation, selection, and validation.