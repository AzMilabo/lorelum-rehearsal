---
anti_patterns:
  - description: A list fan-out awaits every first-level item before starting any second-level follow-up, so the slowest item delays the follow-up of every other item.
    id: react.async.two-stage-list-waterfall
    name: Two-stage batches for nested list data
    severity: warn
applies_when: a fan-out over a list needs a per-item follow-up request that depends on its own item's result
id: react.async.chain-nested-item-fetches
severity: warn
stage: implementation
tech_stack:
  - react
title: Chain Each Item's Follow-Up Fetch Inside Its Own Promise
---

## When to apply

Apply when loading a list where each item needs a second request derived from that item — a detail record, an enrichment, a sub-resource. Decide whether follow-ups start per item or only after the whole first batch resolves.

## Guidance

Chain inside the item. Map each identifier to one promise that resolves the item and continues to its follow-up, then await the mapped promises together:

```ts
const board = await Promise.all(
  shipmentIds.map((id) =>
    getShipment(id).then((shipment) =>
      getTrackingSnapshot(shipment.trackingNumber),
    ),
  ),
);
```

Each item's second leg starts when that item lands. The common two-stage shape — `await` all items, then `await` all follow-ups — turns the slowest first-stage item into a barrier for every follow-up, even for items that resolved long ago.

Handle failure per item when the page tolerates partial data. Under a bare `Promise.all`, one rejected item discards the whole board; catching inside the per-item chain and returning a typed failure entry keeps the other rows. Reserve the uncaught form for callers that genuinely want all-or-nothing.

When the second stage needs all first-stage results — an aggregate, a cross-item deduplication, a sort over the complete set — it is a real barrier: sequence the stages. Chaining applies only when the follow-up depends on its own item alone. For very large lists, combine chaining with a concurrency cap rather than starting every chain at once.

## Anti-pattern

A logistics board awaits all shipments, then awaits all tracking snapshots. One slow carrier endpoint holds every row's snapshot even though the other shipments' carriers answered in milliseconds, because no follow-up may start until the entire first batch has settled.

## Why

Per-item chaining removes the cross-item barrier: each row's latency becomes its own two legs instead of the slowest peer's first leg plus its own second. The data dependencies are unchanged — only the point at which each dependent request may start moves earlier.

## Exceptions and boundaries

- Aggregate second stages are genuine barriers and stay sequential; this Practice does not split a computation that genuinely needs the complete first batch.
- If follow-ups do not depend on item results, there is no nesting; the work is plain parallel fetching covered by `react.async.parallel-independent-work`.
- For small lists where both stages are fast, the restructure is not worth the ceremony; this is a cost boundary, not a style rule.
- A client-side data layer with per-query caching and deduplication may remove the per-item follow-ups entirely; this Practice covers the direct server-side fan-out.

## Example

A shipments board enriches each row with its carrier's tracking snapshot. A per-item helper chains the two legs and tolerates per-row failure, and the board awaits one row promise per item. The declared helpers stand in for the data layer.

```ts
type Shipment = { id: string; carrier: string; trackingNumber: string };
type TrackingSnapshot = { status: string; updatedAt: string };

type BoardRow =
  | { shipment: Shipment; snapshot: TrackingSnapshot }
  | { shipment: Shipment; snapshot: null; error: string };

declare function getShipment(id: string): Promise<Shipment>;
declare function getTrackingSnapshot(
  carrier: string,
  trackingNumber: string,
): Promise<TrackingSnapshot>;

async function loadRow(id: string): Promise<BoardRow> {
  const shipment = await getShipment(id);
  try {
    return {
      shipment,
      snapshot: await getTrackingSnapshot(
        shipment.carrier,
        shipment.trackingNumber,
      ),
    };
  } catch (error) {
    return { shipment, snapshot: null, error: String(error) };
  }
}

export function loadShipmentBoard(shipmentIds: string[]): Promise<BoardRow[]> {
  return Promise.all(shipmentIds.map(loadRow));
}
```

Each row's snapshot starts when its own shipment resolves, so one slow or failed carrier costs only its row. If the board instead awaited all shipments and then all snapshots, the slowest shipment would delay every row's enrichment.
