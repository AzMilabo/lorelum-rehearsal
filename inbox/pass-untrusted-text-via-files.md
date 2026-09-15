---
anti_patterns:
  - description: Building commands like `gh issue create --title "$value"` where the value is produced by string interpolation of user or model text, so shell substitutions such as `$()` and backticks inside the text still execute, and quotes or newlines corrupt the payload.
    id: azmilabo-engineering.tooling.interpolate-untrusted-text
    name: Untrusted text interpolated directly into shell command templates
    severity: critical
applies_when: when a CLI invocation must embed a title, body, comment, release note, or any other user- or model-generated text
id: azmilabo-engineering.tooling.pass-untrusted-text-via-files
severity: critical
stage: tooling
tech_stack:
  - git
  - github
  - gh-cli
title: Pass Untrusted Text to CLIs Through Files, Never Shell Interpolation
---

## When to apply

Apply whenever free-form text (titles, descriptions, comments, release notes, branch names, labels
invented by a user or a model) has to reach a CLI argument or file content. Treat every such string
as untrusted input, exactly like user input in a web form: it may contain shell metacharacters,
substitution syntax, control characters, or content that only looks harmless.

## Guidance

Never paste an untrusted value directly into a shell command template, even inside double quotes —
`$()` and backticks still execute there. Two safe routes:

1. Whole payloads: write the text to a temporary file with a proper file-writing tool (not `echo`,
   not an interpolated heredoc), then hand the CLI the file path — `gh issue create --body-file`,
   `gh release create --notes-file`, `git commit --file`. Delete the temporary files afterwards.
2. Scalar arguments the CLI cannot take from a file: write the value to a temp file, read it into a
   variable with a quoted expansion (`value=$(<"$file")`), and pass `"$value"`. The file round-trip
   keeps the shell from ever interpreting the content.

Validate constrained identifiers before use against their documented formats — PR and issue numbers
must be decimal integers; hosts, repository names, branch names, dates, and colors must match their
patterns — so a malformed value fails loudly instead of silently hitting an unintended target. Never
paste access tokens, OAuth tokens, or credential-store contents into chat or commands; if an
environment token overrides stored credentials, say so without revealing its value.

## Evidence

- AzMilabo/lorelum issue #17, lorelum/lorelum#160 and AzMilabo/lorelum#18 (2026-09-15): all issue
  and PR bodies went through `--body-file` with Write-tool-created temporary files; one intended
  filename already existed and the collision failed safely at write time instead of overwriting
  content — the file route also made the cleanup (`rm`) explicit.
- lorelum/lorelum-packs#13 body (2026-09-15): markdown tables, CJK text, and code spans passed
  through `--body-file` without any shell-escaping incident.
