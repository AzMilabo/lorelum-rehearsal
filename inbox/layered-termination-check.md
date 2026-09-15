---
id: azmilabo-engineering.verification.layered-termination-check
title: Gate Completion with Layered Verification Owned by the Harness
applies_when: when an agentic coding task approaches done and someone must decide which checks gate completion, or when designing the harness or CI gate that terminates the agent loop
stage: verification
severity: warn
tech_stack:
  - agentic-coding
  - ci
anti_patterns:
  - id: azmilabo-engineering.verification.self-report-completion
    name: Self-reported completion
    description: Declaring a task done because the agent's own assessment or a single green unit-test run says so, without the layered external gate passing in order.
    severity: warn
---

## When to apply

Apply when defining what "done" means for an agent-driven task — in the harness, the CI pipeline,
or the task's acceptance criteria — and at the moment of declaring completion. This practice is
about the design of the termination gate, not the wording of the final report (that is covered by
`agentic-coding.delivery.claim-only-supported-outcome`). It does not apply to exploratory work with
no acceptance criteria; there, define the criteria first or report the work as exploratory.

## Guidance

Completion is defined by a gate outside the agent's self-assessment — the harness or CI owns the
definition of done, and the agent reports against it, never replaces it. Self-confidence and a
green unit-test run are not the gate: unit tests passing while the feature is broken is the
signature failure, because the three defect classes live at different layers.

Run the three verification layers in order and never skip ahead:

1. Static (types, lint, schema validation) — catches interface mismatch.
2. Runtime (tests asserting observable behavior) — catches logic and state-propagation defects.
3. System-level (real install/run/deploy path in a clean environment) — catches wiring,
   environment, and dependency gaps.

No refactor or bonus work before the core functionality passes all layers — the lecture's "don't
gild the lily before the exam passes". Report each layer's actual result at completion time rather
than an aggregate "it works".

Decision boundary: the layers are ordered by cost, so a failure at a cheap layer means stop and
fix before paying for the expensive one; but passing a cheap layer never licenses skipping the
next.

## Evidence

- walkinglabs lecture-09, "Why agents declare victory too early"
  (https://walkinglabs.github.io/learn-harness-engineering/zh/lectures/lecture-09-why-agents-declare-victory-too-early/):
  calibration-bias research and the Anthropic planner/generator/evaluator experiment as the source
  of the ladder.
- AzMilabo/lorelum-rehearsal release of azmilabo-engineering v0.1.1 (2026-09-15): the release was
  declared working after tag + registry push without exercising a client install; a follow-up
  `lore pack update` failed with `registry.pack-not-found` while the raw descriptor was correct
  (cache lag), and the release was reverted. The system-level layer had been skipped.
- lorelum/lorelum CI (`.github/workflows/ci.yml`) and PR template checklist: typecheck →
  design:lint → lint → test → build:site, an in-the-wild layered gate where "done" is owned by the
  harness, not the author.
- Relates to `agentic-coding.delivery.claim-only-supported-outcome` and
  `agentic-coding.verification.close-or-declare-evidence-gaps`, which govern the reporting moment;
  this practice governs the gate design itself.
