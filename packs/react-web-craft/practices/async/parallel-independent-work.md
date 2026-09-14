---
anti_patterns:
  - description: An async server function awaits unrelated I/O operations one after another, so its latency is the sum of all requests even though no result depends on an earlier one.
    id: react.async.sequential-independent-fetches
    name: Independent requests awaited in sequence
    severity: warn
applies_when: an async function performs several I/O operations whose results do not depend on one another, and a decision is needed about starting them together or in sequence
id: react.async.parallel-independent-work
severity: warn
stage: implementation
tech_stack:
  - react
title: Start Independent Async Work Together
---

## When to apply

Apply while writing or reviewing an async function that performs more than one request or other I/O operation. Decide, operation by operation, whether its input depends on another operation's result, and sequence only true dependents.

## Guidance

Classify before sequencing. An operation is dependent only if it consumes another operation's result as input; everything else is independent and can start immediately. Independent operations start together with `Promise.all`. The settled results come back in call order, not completion order.

`Promise.all` is all-or-nothing: it rejects as soon as one input rejects, and the batch's other results are dropped. When the caller can make use of partial results, use `Promise.allSettled` and inspect each status instead.

A dependency does not force serializing unrelated work. Create a dependent operation's promise from its input's promise as soon as the input is started, and include the chained promise in the same final `Promise.all`:

```ts
const customerPromise = getCustomer(customerId);
const paymentMethodsPromise = customerPromise.then((customer) =>
  getPaymentMethods(customer.id),
);

const [customer, taxRegion, template, paymentMethods] = await Promise.all([
  customerPromise,
  getTaxRegion(request.countryCode),
  getDefaultInvoiceTemplate(),
  paymentMethodsPromise,
]);
```

The dependent lane costs only its own extra leg; the independent lanes run alongside it.

Keep fan-out bounded. For per-item work over a long list, cap concurrency instead of starting every request at once; "together" means together per lane, not unbounded. `Promise.all` overlaps waiting on I/O; it does not parallelize CPU work, and requests that share one connection pool or single-threaded resource may serialize anyway. Treat the win as likely for independent network requests, not as a universal multiplier.

## Anti-pattern

An invoice builder awaits the customer, then the tax region, then the invoice template, then the payment methods in sequence. No result depends on an earlier one, so total latency is the sum of four round trips where the same data costs the slowest single lane.

## Why

Each `await` placed before unrelated work is a barrier those operations wait behind. Starting independent operations together collapses the wall-clock time of the group to its slowest member, and starting a dependent operation at its input's resolution keeps chains from becoming full barriers.

## Exceptions and boundaries

- If every operation genuinely feeds the next, the sequence is the dependency graph; this Practice does not ask to parallelize true dependents.
- `Promise.all`'s fail-fast behavior is usually correct for building one response; choose `Promise.allSettled` only when the caller has defined behavior for partial results.
- Third-party rate limits or per-request cost can make deliberate sequencing cheaper than maximal concurrency; cap or sequence explicitly when a quota exists.
- This Practice stays at the single-function level. Restructuring a component tree so slow data does not block the page shell is the `react.async.suspense-boundary-scope` decision, and per-item follow-up chaining over a list is `react.async.chain-nested-item-fetches`.

## Example

An invoice assembly service needs the customer, the tax region for the request, and the default template independently; payment methods depend on the customer. The chained promise starts the moment the customer resolves, so the payment-methods lane adds only its own leg. The declared helpers stand in for the data layer.

```ts
type Customer = { id: string; name: string };
type TaxRegion = { code: string; rate: number };
type InvoiceTemplate = { id: string; headerHtml: string };
type PaymentMethod = { id: string; label: string };

declare function getCustomer(id: string): Promise<Customer>;
declare function getTaxRegion(countryCode: string): Promise<TaxRegion>;
declare function getDefaultInvoiceTemplate(): Promise<InvoiceTemplate>;
declare function getPaymentMethods(customerId: string): Promise<PaymentMethod[]>;

export async function buildInvoiceSummary(customerId: string, countryCode: string) {
  const customerPromise = getCustomer(customerId);
  const paymentMethodsPromise = customerPromise.then((customer) =>
    getPaymentMethods(customer.id),
  );

  const [customer, taxRegion, template, paymentMethods] = await Promise.all([
    customerPromise,
    getTaxRegion(countryCode),
    getDefaultInvoiceTemplate(),
    paymentMethodsPromise,
  ]);

  return { customer, taxRegion, template, paymentMethods };
}
```

Wall-clock cost is now the slowest single lane — `max(customer + paymentMethods, taxRegion, template)` — instead of the sum of four legs. If a partial invoice were still renderable when one lane fails, the final await would become `Promise.allSettled` over the same four promises.
