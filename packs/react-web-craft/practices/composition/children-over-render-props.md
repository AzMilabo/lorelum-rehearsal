---
anti_patterns:
  - description: A component exposes one `renderX` callback prop per slot, so callers compose structure through unfamiliar callback signatures while the component could have accepted children and slot elements directly.
    id: react.composition.render-props-for-static-slots
    name: Render props for static slots
    severity: warn
applies_when: a component's composition API is being designed and a choice is needed between children/slot elements and render-function props
id: react.composition.children-over-render-props
severity: warn
stage: design
tech_stack:
  - react
title: Compose with Children; Reserve Render Props for Data-Bound Slots
---

## When to apply

Apply when designing a component others will fill with content — a frame, a card, a toolbar, a dialog. Decide which parts of its API are structure the caller already has, and which are slots the component must fill with its own data.

## Guidance

Structure passes as children. What the caller can write as elements at the call site — headers, actions, sections — should travel as `children` or named slot props containing elements, not as functions the component calls:

```tsx
<Card>
  <Card.Header title="Billing" />
  <PlanSummary plan={plan} />
  <Card.Footer>
    <Button onClick={openPortal}>Manage subscription</Button>
  </Card.Footer>
</Card>
```

Render functions earn their indirection only when the slot is data-bound: the component holds information the caller needs to render with — each row of a table, each item of a virtualized list, measured dimensions — and the callback is the delivery mechanism:

```tsx
<DataTable rows={invoices} renderRow={(invoice) => <InvoiceRow key={invoice.id} invoice={invoice} />} />
```

The reviewable question per prop: does the caller need anything from inside the component to render this? If no, take an element. A `renderHeader?: () => ReactNode` whose result never touches internal state is a callback impersonating a prop, and it costs readability and inlining for nothing. Naming follows the split — slot props are element props; data-bound ones are `render*` taking the data as a parameter, not zero-argument thunks.

## Anti-pattern

A toolbar component declares `renderLeft`, `renderRight`, and `renderOverflow` — all zero-argument. Callers study the component's signature to compose plain buttons, each usage wraps static elements in arrows, and none of the three slots can ever receive toolbar state, because the component never passes any.

## Why

Children are declarative, tree-shaped, and already familiar; render functions add a call layer that pays for itself only when the component supplies data per invocation. Keeping element props and render props distinct also makes the component's contract self-describing: element props say "bring structure", render props say "I will call you with my data".

## Exceptions and boundaries

- Zero-argument render props are occasionally justified for lazy mounting (deferring a heavy subtree until the slot renders); the deferral motive should be stated, and suspense boundaries usually express it better.
- Compound-component APIs (`<Toolbar.Left />`) are the shared-context evolution of slot elements; they add machinery that single-use slots do not need.
- Data-bound slots need key discipline and stable callbacks like any list rendering; this Practice covers the API shape, not list rendering semantics.
- Wrapping children to inject context is provider design, not a render prop; see `react.state.share-one-owner` for the ownership decision underneath.

## Example

A page header takes structure as elements and leaves one data-bound slot for what only the table knows. The declared `Button` and `FilterInput` stand in for real components.

```tsx
function PageHeader({ title, actions, children }: {
  title: string;
  actions?: React.ReactNode;
  children?: React.ReactNode;
}) {
  return (
    <header className="page-header">
      <h1>{title}</h1>
      {children}
      <div className="page-header-actions">{actions}</div>
    </header>
  );
}

<PageHeader
  title="Invoices"
  actions={<Button onClick={newInvoice}>New invoice</Button>}
>
  <FilterInput value={filter} onChange={setFilter} />
</PageHeader>
```

`actions` and `children` are elements: the caller composes real buttons and inputs with no callback layer. A render prop would earn its place here only if the header had to hand the caller something internal — say, which actions overflowed into a menu — and then the signature would say so: `renderOverflow?: (items: Action[]) => ReactNode`, data as a parameter, not a zero-argument thunk.
