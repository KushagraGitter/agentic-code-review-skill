# agentic-code-review

An agent skill for layered, risk-scaled code review of agent-written (or any) code. It's based on [*How the Claude Code team does code review*](https://claude.dev/blog/solving-code-review-at-anthropic/) by Addy Osmani, Morgan Westlee Lunt and Qing Wang (Oct 2026).

> "Did a human read the code?" is an imprecise proxy for "what could go wrong with this change, and which check — automated or human — is the right one to catch it?"

## What it does

1. **Gathers context**: the diff, the PR's stated intent, `CLAUDE.md`, and any `REVIEW.md` (the nearest one overrides the defaults for its folder).
2. **Runs checks before judging**: tests, build, types and lint, with the real output shown. For bug fixes, it checks the root cause.
3. **Hunts for bugs in parallel**: correctness, integration, concurrency/state, tests and conventions.
4. **Verifies every finding**: it tries to refute each one (or has verifiers vote on it), drops style-only noise, and reports a repeated bug once with every location.
5. **Tags findings** as **Important**, **Nit** (max 3) or **Pre-existing** (so old bugs aren't blamed on the author).
6. **Runs a separate security pass**; its findings block the merge.
7. **Assigns a risk tier** from the change, not the author: Low → 0 human approvals, Medium → 1, High → 2. CI, auth, billing and installer changes always need a person.
8. **Converges**: it caps review at 3 rounds, re-reviews only the new diff after round 2, and doesn't repeat findings.
9. **Report**: the summary starts with the count of findings, or says "Nothing to fix" plainly.

## Layout

```
SKILL.md                         # the workflow
references/risk-tiers.md         # approval tiers, always-human paths, Quick/Standard/Deep
references/adoption-checklist.md # rolling the whole process out to a team, measuring it, improving the reviewer
assets/REVIEW.md.template        # starter REVIEW.md for code owners
assets/PULL_REQUEST_TEMPLATE.md  # Testing / Risk / How to revert
```

## Install

**Claude Code:** copy this folder to `~/.claude/skills/agentic-code-review/` (just you) or `.claude/skills/agentic-code-review/` in a repo (the whole team).

**Claude apps:** zip the folder and upload it under Settings → Capabilities → Skills.

Then ask Claude something like "review this branch before I merge" or "what risk tier is this PR?".
