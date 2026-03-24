---
name: codex-review
description: >
  Run OpenAI Codex CLI as a second-opinion reviewer. Automatically triggered
  by hooks after writing plans or implementing changes. Manual: /codex-review.
user-invocable: true
version: 2.0.0
---

# Codex Review — Second Opinion

Two modes. The hook tells you which one to use.

## Prerequisites

- Codex CLI installed: `npm install -g @openai/codex`
- `OPENAI_API_KEY` set in your environment

## Mode 1: Plan Review

When you've written or updated a plan file:

```bash
rm -f /tmp/codex-plan-review.txt && codex exec \
  --full-auto --ephemeral \
  -o /tmp/codex-plan-review.txt \
  "Read the plan file at <plan-file-path>.

  Review standard:
  - Flag only material issues: wrong assumptions, missing edge cases, integration gaps, operational risks.
  - Do NOT praise, restate the plan, or nitpick style.
  - If a concern is plausible but unproven, put it under 'Uncertain concerns'.

  Output format:
  Plan goal: <one sentence>
  Material issues:
  - [high|medium] <issue>
    Evidence: <plan section or specific missing assumption>
    Impact: <what could go wrong>
    Suggested improvement: <smallest useful fix>
  Uncertain concerns:
  - <optional>
  If no material issues: No material issues found in plan."
```

After: read output, evaluate findings, update plan if warranted, tell user what changed.

## Mode 2: Implementation Review

When code files changed. Fill in the context fields before running.

**5 or fewer files → Targeted review (read current files, no diff):**

```bash
rm -f /tmp/codex-targeted-review.txt && codex exec \
  --full-auto --ephemeral \
  -o /tmp/codex-targeted-review.txt \
  "Targeted feature review. Read CURRENT file content directly, do NOT run git diff.

  Intent:
  - Feature: <name>
  - Goal: <one sentence>
  - Expected behavior / invariants:
    - <invariant 1>
    - <invariant 2>
  - Non-goals:
    - <what to ignore>

  Scope (hard boundary):
  - Files: <file1> <file2>
  - Symbols: <function1> <function2>

  Instructions:
  1. Locate each symbol with fixed-string search.
  2. Read enclosing implementation + immediate callers.
  3. Review ONLY for: correctness, integration mismatches, race conditions, error handling, security.
  4. Ignore style, naming, unrelated code.

  Output format:
  Reviewed scope: <feature> in <files>
  Material findings:
  - [high|medium] <file>:<line> - <issue>
    Evidence: <what proves it>
    Impact: <failure mode>
    Fix: <smallest fix>
  Uncertain concerns:
  - <optional>
  If no issues: No material issues found in scoped feature."
```

**More than 5 files → Scoped diff review:**

```bash
rm -f /tmp/codex-code-review.txt && codex exec \
  --full-auto --ephemeral \
  -o /tmp/codex-code-review.txt \
  "Review uncommitted changes in: git diff -- <file1> <file2> ...
  Read surrounding CURRENT file context before making claims.

  Context:
  - Goal: <one sentence>
  - Expected behavior / invariants:
    - <invariant 1>
    - <invariant 2>
  - Non-goals:
    - <what to ignore>

  Review standard:
  - Only flag issues backed by concrete code evidence.
  - Ignore style, naming, unrelated code.

  Output format:
  Reviewed files: <files>
  Material findings:
  - [high|medium] <file>:<line> - <issue>
    Evidence: <what proves it>
    Impact: <failure mode>
    Fix: <smallest fix>
  Uncertain concerns:
  - <optional>
  If no issues: No material issues found in reviewed changes."
```

After: read output, validate scope header matches, fix legitimate issues, discard out-of-scope findings, tell user what Codex found.

## Mode 3: PR Review (manual only)

For reviewing pull requests. Not auto-triggered — use when asked to review a PR.

First verify `gh` CLI is available: `which gh`

```bash
rm -f /tmp/codex-pr-review.txt && codex exec \
  --full-auto --ephemeral \
  -o /tmp/codex-pr-review.txt \
  "Review PR #<number>. Run: gh pr diff <number>.
  Also read the key changed files for full context.
  Focus on: security (IDOR, auth bypass), data integrity
  (race conditions, orphaned data), correctness, and regressions.

  Output format:
  PR: #<number> (<title>)
  Files reviewed: <count>
  Material findings:
  - [high|medium] <file>:<line> - <issue>
    Evidence: <what proves it>
    Impact: <failure mode>
    Fix: <smallest fix>
  If no issues: No material issues found in PR."
```

## Multi-Round Reviews

When re-reviewing after fixes (round 2+), the hook provides the path to previous findings.
Include them in the prompt:

```
Previous review (round N-1) found these issues:
<read contents of previous findings file>

Check: were these issues addressed? Flag any that remain + any new issues introduced by the fixes.
```

This ensures Codex verifies its previous findings were actually addressed, not just that new code was written.

## Rules

1. **Always `--full-auto --ephemeral`** — Codex runs headless, no interaction
2. **Always `-o`** — capture to file, then read it
3. **Always `rm -f` before** — prevents stale output
4. **Timeout: 120s** — kill and report "Codex timed out" if hung
5. **Validate output** — check scope/goal header matches expected review
6. **Don't blindly apply** — Codex is a second opinion, evaluate critically
7. **Never share secrets** — no .env values or API keys in prompts
8. **Model + reasoning** — Default: `gpt-5.4` with `xhigh` reasoning. Override in `.codex-review.json` with `"model"` and `"reasoningEffort"`. Pass to codex: `codex exec --model <model> --reasoning-effort <effort> ...`. Available efforts: none, minimal, low, medium, high, xhigh
