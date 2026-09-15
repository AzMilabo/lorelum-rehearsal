---
anti_patterns:
  - description: Running `gh issue create`, `gh pr edit`, `gh repo edit` and similar against whatever repository and account the CLI happens to resolve from the current directory, an issue number, or ambient credentials, without confirming host, account, and owner/repository first.
    id: azmilabo-engineering.tooling.write-github-without-binding
    name: GitHub write commands issued without a verified identity and target
    severity: critical
applies_when: before any GitHub CLI command that writes to a remote, such as creating or editing an issue, pull request, release, comment, label, or repository setting
id: azmilabo-engineering.tooling.bind-github-target-before-write
severity: critical
stage: tooling
tech_stack:
  - git
  - github
  - gh-cli
title: Verify gh Identity and Target Repository Before Any Remote Write
---

## When to apply

Apply to every GitHub CLI invocation that changes remote state, and re-apply when the shell,
worktree, or repository changes between invocations — environment setup verified in an earlier
command is not proof for a later one. Reading (listing issues, fetching files) tolerates ambiguity;
writing does not, because a wrong target publishes content to an unintended public place and may be
impossible to fully retract.

## Guidance

Before the first write in a session, run a preflight and state the result:

1. `gh` is installed and its version is known.
2. `gh auth status --hostname <host>` succeeds, and `gh api user` returns the account you intend to
   act as. Stop and switch identities if it is not.
3. Resolve the target as an explicit `host/owner/repository` — from the user's words when given,
   otherwise from the repository's configured remote. Never infer the target from the current
   directory name, a PR number, or an issue number alone.
4. Bind every subsequent command to that target: prefer `--repo <host>/<owner/repo>` where the
   subcommand supports it, and set `GH_HOST` (plus `GH_REPO` as fallback) so an agent cannot hang on
   an interactive prompt.
5. Before the write executes, report the verified host, account, and repository; for destructive or
   hard-to-reverse operations (visibility change, deletion of labels/milestones/branches,
   publishing) require explicit confirmation even when the target is verified.

Decision boundary: the verification cost is one or two cheap API calls; skip it only for pure reads
where a wrong target wastes nothing but time.

## Evidence

- AzMilabo/lorelum issue #17 and lorelum/lorelum#160 (2026-09-15): preflight confirmed account
  `AzMilabo` and each target repository before `gh issue create` / `gh pr create`; roughly twenty
  `gh` invocations across two repositories produced no misdirected write.
- Same session: `gh repo edit --jq` failed on flag parsing before executing anything, confirming
  that even "obvious" commands need their effect verified against the documented interface, not
  assumed.
