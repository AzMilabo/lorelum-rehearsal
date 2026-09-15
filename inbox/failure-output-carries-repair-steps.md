---
id: azmilabo-engineering.error-handling.failure-output-carries-repair-steps
title: Make Failure Output Carry the Repair Steps
applies_when: when writing error messages, validation failures, test reports, or CLI output that another agent or a human under time pressure will have to act on
stage: error-handling
severity: warn
tech_stack:
  - agentic-coding
  - cli
  - ci
anti_patterns:
  - id: azmilabo-engineering.error-handling.bare-failure-message
    name: Bare failure message
    description: A gate fails with output like "Test failed" or a bare exit code, forcing the next iteration to re-derive the entire context before it can even start fixing.
    severity: warn
---

## When to apply

Apply when designing or writing any output that reports a failure to be acted on: CLI errors,
schema-validation diagnostics, test-report summaries, CI step failures. The consumer is often an
agent running an unattended loop — a message it cannot act on costs a whole iteration just to
re-gather context, and a human's version of that cost is worse.

## Guidance

The standard is the reviewer's red-pen margin note: it points at the exact line, names the
violation, and says what would satisfy it. Failure output should do the same with three parts —
(1) which check failed and at which verification layer, (2) the concrete cause as observed, not
inferred, (3) the next repair step or the command that reveals it.

A typed error taxonomy earns its keep here: a stable code plus a one-line remedy
(`registry.unavailable` → "the descriptor is fetched unauthenticated from raw.githubusercontent;
is the repository public?") lets the consumer fix the problem in one step, which is exactly what
happened when a private pack repo failed to install.

The test for the message: would a competent agent with no prior context be able to start repairing
from this output alone, without re-running anything to understand what broke?

Decision boundary: detailed remediation prose belongs in linked recovery docs, not in the error
line itself — the output carries the pointer and the first step, the doc carries the rest.

## Evidence

- walkinglabs lecture-09, "Why agents declare victory too early"
  (https://walkinglabs.github.io/learn-harness-engineering/zh/lectures/lecture-09-why-agents-declare-victory-too-early/):
  the red-pen annotation standard ("a review comment tells you what to fix") proposed for
  harness termination checks.
- lore CLI typed errors (lorelum/lorelum `packages/cli/src/install/load-registry.ts`): the
  `registry.unavailable` error on a private repository led directly to the public-visibility fix
  in one step during the rehearsal-repo install-loop verification (2026-09-15).
- Same session: `gh repo edit --jq` failed naming the unknown flag before executing anything —
  a precise failure message made the repair immediate.
