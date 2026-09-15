---
anti_patterns:
  - description: Running the auto-formatter over the whole tree and trusting the resulting diff, which mixes intended edits with mass reformatting of files the pinned formatter version renders differently than what was committed.
    id: azmilabo-engineering.tooling.format-without-baseline
    name: Repo-wide format without checking the committed baseline
    severity: warn
applies_when: before running a repo-wide auto-formatter on a repository whose committed formatting may not match the pinned formatter version
id: azmilabo-engineering.tooling.verify-formatter-baseline
severity: warn
stage: tooling
tech_stack:
  - git
  - ci
title: Verify Formatter Baseline Before a Repo-Wide Format
---

## When to apply

Apply before executing any `format` command that rewrites the whole tree (for example
`oxfmt --write .`, `prettier --write .`), especially right after switching branches, syncing a fork,
or refreshing dependencies so that the formatter version changed under you. The risk is not the
files you meant to edit; it is the hundreds of unrelated files the tool rewrites because the
committed formatting was produced by a different formatter version.

## Guidance

Check the baseline first: run the formatter in check mode on a clean tree (or format a scratch copy)
and count how many files it would touch. If the count is far beyond the files you intend to edit,
the committed baseline disagrees with the current tool version — a repo-wide format will bury your
change.

In that case do not commit the mass reformat inside a feature change. Revert the formatter output
(`git checkout -- .`), keep the hand-written edits only, and rely on the project's CI format signal:
many repos run the format check as a non-blocking step (`continue-on-error`) or exclude formats such
as Markdown whose canonical layout is version-dependent. Verify the pre-existing failure exists at
the base commit so the review can see your change did not introduce it.

Decision boundary: a dedicated formatting-migration commit is fine when the team actually wants to
re-baseline; mixing it into a functional or docs change is what destroys reviewability.

## Evidence

- AzMilabo/lorelum#18 (2026-09-15): running `bun run fmt` after a fork sync reformatted 700 files;
  reverted to HEAD, re-applied the 9 hand edits, PR diff stayed at 5 files / +9-3. Upstream
  `fmt:check` failure confirmed identical at base `24cb0df`, and CI marks it `continue-on-error`.
