# React Web Craft

> **Unreleased local Pack candidate.** The canonical Pack root is `packs/react-web-craft/`. Version `0.1.0` is a mutable working marker, not a release. No Registry entry, immutable release ref, or tag exists. The Pack license is undecided, so this candidate is not claimed to pass complete publication validation, be redistributable, or be installable.

Canonical English. A simplified-Chinese companion is added after content freezes, at release preparation; the workspace entry is `drafts/react/README.md`.

## Intended reader

React DOM web application developers making day-to-day design and performance decisions: what component state represents and who owns it, how async work is sequenced, what code loads when, how rendering cost is controlled, and how component APIs are composed. Next.js is in scope only where the sources pin its React-facing semantics.

## Coverage

Work proceeds by category; each category groups Practices around one decision area. Selection follows the pinned-source research in `drafts/react/`, not a rule-count quota.

| Category | Decision area | Status |
|---|---|---|
| [state](./practices/state/) | component-state ownership, representation, initialization, and callback update semantics | 6 Practices |
| [async](./practices/async/) | data-fetch dependency ordering, parallelization, and Suspense boundaries | 4 Practices |
| [bundle](./practices/bundle/) | deferring non-critical code and loading on user intent | 2 Practices |
| [server](./practices/server/) | request-scoped data boundaries, module-state isolation, and Server Action authorization | 3 Practices |
| [rendering](./practices/rendering/) | render conditions, hydration consistency, resource priority, and update priority | 5 Practices |
| [composition](./practices/composition/) | component API shape and composition patterns | 4 Practices |

### state

| Decision moment | Practice |
|---|---|
| A callback uses a functional updater but still reads other reactive values | [A Functional Updater Does Not Refresh Other Captures](./practices/state/functional-updater-scope.md) |
| Multiple components must coordinate one changing UI value | [Keep Coordinated State at One Shared Owner](./practices/state/share-one-owner.md) |
| A value must persist between renders but is not rendered UI state | [Use Refs for Non-Rendered Per-Instance Bookkeeping](./practices/state/ref-for-nonrendered-values.md) |
| An editable value follows a changing default until local user intent exists | [Preserve a Reactive Default Until the User Overrides It](./practices/state/fallback-until-overridden.md) |
| Multiple flags allow impossible combinations of one exclusive workflow | [Represent Mutually Exclusive UI Modes with One State](./practices/state/model-exclusive-modes.md) |
| Expensive pure work constructs a mount-time editable snapshot | [Lazy-Initialize Expensive Mount-Time State](./practices/state/lazy-initializer.md) |

The six entries are distilled from pinned Vercel Skills and official React documentation; per-decision selection records and rejected-candidate tombstones are in `drafts/react/CANDIDATE-CARDS.md`. Provenance, exact Vercel source digests, and include/exclude rationale are in [SOURCES.md](./SOURCES.md); source attribution does not determine license permission.

### async

| Decision moment | Practice |
|---|---|
| A function performs several independent I/O operations | [Start Independent Async Work Together](./practices/async/parallel-independent-work.md) |
| Cheap local guards could skip the branches that await | [Run Cheap Checks Before Starting Async Work](./practices/async/await-after-cheap-work.md) |
| A list fan-out needs a per-item follow-up request | [Chain Each Item's Follow-Up Fetch Inside Its Own Promise](./practices/async/chain-nested-item-fetches.md) |
| One slow data source delays the whole page shell | [Put Slow Subtrees Behind Their Own Suspense Boundaries](./practices/async/suspense-boundary-scope.md) |

The four entries are distilled from pinned Vercel rules and official React, MDN, and Next.js documentation; selection records are in `drafts/react/CANDIDATE-CARDS.md` and provenance in [SOURCES.md](./SOURCES.md).

### bundle

| Decision moment | Practice |
|---|---|
| A large module is needed only on a conditional or later path | [Defer Heavy Non-Critical Loads Until They Are Needed](./practices/bundle/defer-heavy-loads.md) |
| A deferred chunk sits behind a visible, predictable step | [Preload Deferred Code When User Intent Signals It](./practices/bundle/preload-on-intent.md) |

Selection records are in `drafts/react/CANDIDATE-CARDS.md` and provenance in [SOURCES.md](./SOURCES.md).

### server

| Decision moment | Practice |
|---|---|
| One server render fetches the same request-scoped value from several components | [Deduplicate Request-Scoped Lookups with React cache](./practices/server/request-dedup-cache.md) |
| Request-scoped data must reach several server components or helpers | [Keep Request Data Out of Module Scope](./practices/server/no-module-request-state.md) |
| A `"use server"` mutation's authentication and authorization need a home | [Authorize Inside Every Server Action](./practices/server/authorize-server-actions.md) |

Selection records are in `drafts/react/CANDIDATE-CARDS.md` and provenance in [SOURCES.md](./SOURCES.md).

### rendering

| Decision moment | Practice |
|---|---|
| A `&&` render guard can leak a renderable falsy value | [Make Render Conditions Actually Boolean](./practices/rendering/boolean-render-guards.md) |
| Server output and the first client render can diverge on clocks, locale, or storage | [Keep the First Client Render Equal to Server Output](./practices/rendering/hydration-consistency.md) |
| Hints and scripts each need a priority decision per resource | [Match Resource Hints and Script Loading to Real Need](./practices/rendering/resource-hints-scripts.md) |
| One interaction drives both an urgent control and expensive follow-up work | [Split Urgent Input Updates from Slow Follow-Up Renders](./practices/rendering/update-priority.md) |
| A computed value is copied into state and refreshed by an effect | [Derive Render Values; Do Not Mirror Props into State](./practices/rendering/derive-dont-mirror.md) |

Selection records are in `drafts/react/CANDIDATE-CARDS.md` and provenance in [SOURCES.md](./SOURCES.md).

### composition

| Decision moment | Practice |
|---|---|
| A component grows boolean mode flags carrying mode-specific data | [Prefer Explicit Component Variants to Boolean Mode Flags](./practices/composition/explicit-variants-over-flags.md) |
| A fillable component's slots need an API | [Compose with Children; Reserve Render Props for Data-Bound Slots](./practices/composition/children-over-render-props.md) |
| A component must expose its node to callers across React version lines | [Pass ref as a Prop on React 19; Keep forwardRef for Older Lines](./practices/composition/react19-ref-prop.md) |
| One retained public component API has mutually exclusive typed modes | [Model Mutually Exclusive React Props with a Discriminated Union](./practices/composition/model-mutually-exclusive-props.md) |

Selection records are in `drafts/react/CANDIDATE-CARDS.md` and provenance in [SOURCES.md](./SOURCES.md).

## Boundaries

This Pack targets React DOM web applications. React Native/Expo, view transitions, state-library selection, global/server data ownership, and generic JavaScript micro-optimization remain out of scope. Framework-specific server rules are admitted only where a pinned source fixes the behavior; each category's Practices state their own narrower boundaries.

## Review and release status

Working draft status as of 2026-09-14: all twenty-four Practices (state 6, async 4, bundle 2, server 3, rendering 5, composition 4) were verified against the pinned Vercel snapshot (file digests in SOURCES.md all match), and all forty of their code blocks compile under TypeScript 5.9 (strict; `noImplicitAny` relaxed where example-external data-layer names are stubbed `any`, per each example's declared-helper convention) with React 19 and Next 15 types. That gate found and fixed one defect: a missing React import in `react19-ref-prop`'s example. The maintainer content review passed on 2026-09-14. Static Pack conformance (pack.yaml contract, frontmatter shape, ID uniqueness and naming, section structure) validates CONFORMANT; it caught and fixed one defect: the `composition` category's ID namespace now matches its directory (`react.composition.*`). Still open: the real `lore install` path and retrieval testing - the Lorelum CLI is not yet publicly distributed (npm holds a name reservation only), so those sub-gates are blocked on tooling, not on this Pack - and the publication-license determination (source declares MIT but the snapshot carries no LICENSE file).

No Registry entry, immutable release, or tag has been created.
