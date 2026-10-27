---
name: agentic-code-review
description: Layered, risk-scaled code review for agent-written (or any) code, modeled on the Claude Code team's process. Use whenever the user asks to review a diff, branch, PR or commit, says "review this before I merge", "is this safe to ship", "re-review", "what tier is this change", or wants a REVIEW.md, PR template or review policy set up. Verifies before judging, hunts bugs in parallel, keeps only findings that survive verification, tags them Important / Nit / Pre-existing, runs a separate security pass, assigns a human-approval risk tier, and caps review rounds so author and reviewer converge.
---

# Agentic Code Review

Code review has two jobs: catch issues before they ship, and spread understanding of the codebase. When agents write most of the code, line-by-line human reading can't keep up, so the question changes from *"did a human read this?"* to **"what could go wrong with this change, and which check — automated or human — is the right one to catch it?"**

This skill is the automated part of that answer. Its goal is to save human attention for the calls only humans can make, without letting reverts or incidents rise. Noise is a real cost: a person waves off a bad nit in seconds, but an agent author will "fix" it, adding a commit and another review round. So precision matters as much as recall.

Based on *How the Claude Code team does code review* (Addy Osmani, Morgan Westlee Lunt, Qing Wang — claude.dev, Oct 2026).

---

## Step 0 — Gather context

Before reading the diff, collect:

1. **The change.** `git diff <base>...HEAD` (default base: `main`, else `master`), or the PR / patch the user gave. Note files touched and line count.
2. **Stated intent.** PR title/description, linked issue, or ask the user in one line if there's none. You review against intent: does it solve the problem it claims to?
3. **Project rules.** Read `CLAUDE.md` (stack, conventions, goals, non-goals) and any `REVIEW.md` files at the repo root and in directories the diff touches. `REVIEW.md` overrides the defaults in this skill for its subtree (what failure looks like there, paths to skip, nit cap).
4. **Round number.** Is this a first review or a re-review? If prior findings exist (PR comments, an earlier report in this conversation), load them — they drive the convergence rules in Step 7.
5. **Depth.** Pick **Quick / Standard / Deep** (see `references/risk-tiers.md`). Default Standard. Go Deep when a missed bug costs more than the extra time (auth, billing, data migrations, updaters, CI/release, concurrency), or when the user asks.

If there is no `REVIEW.md` and the user would benefit, offer to generate one from `assets/REVIEW.md.template` at the end — don't block on it.

## Step 1 — Verify before judging

Evidence beats opinion. Where the environment allows, run what the repo provides and **show the output** rather than claiming it passed:

- Tests (prefer the ones touching changed files first), build, type-check, linter/formatter.
- For UI changes, a screenshot or render compared against the design/guidelines if available.
- For a bug fix, try to reproduce the original bug and confirm the fix addresses the **root cause**, not the symptom.

If you can't run anything (no shell, no deps), say so in the report's Evidence section — a reviewer who can't say what the checks would miss should ask for a real human review, not wave it through.

## Step 2 — Parallel bug hunt

Look at the diff against the **whole repo**, not just the hunks: callers, callees, shared types, config, migrations. Hunt along independent lenses. If subagents are available, give each lens to its own fresh-context agent and run them in parallel; otherwise do separate, deliberate passes.

| Lens | Asks |
|---|---|
| Correctness | Does it do what the intent says? Off-by-one, null/undefined, wrong branch, broken invariants, unhandled errors, bad edge cases. |
| Integration | Does it fit the system? Broken callers, changed contracts/APIs, schema/migration mismatches, config drift, platform differences. |
| Concurrency & state | Races, ordering, retries, idempotency, caching/invalidation, resource leaks. |
| Tests | Do tests actually exercise the new behavior? Missing negative cases? Tests that pass for the wrong reason? |
| Conventions | Breaks a rule in `CLAUDE.md`/`REVIEW.md`? Makes a line in `CLAUDE.md` out of date? |

Security is **not** a lens here — it gets its own pass (Step 5).

Each candidate finding must carry: `file:line`, what goes wrong, a concrete trigger (input/sequence/state), and a suggested fix. A guess from a function name is not evidence — open the code.

## Step 3 — Verify every finding

Candidates are hypotheses. Before anything is reported, try to **refute** each one:

- Re-read the surrounding code and any guards upstream/downstream.
- Trace a concrete path to the failure; run it or write a tiny repro if cheap.
- If subagents are available, have 2–3 independent verifiers vote; keep a finding only if a majority confirm it's real. Otherwise, refute it yourself with a fresh read.

Then **drop**:
- Pure style, formatting, naming taste, "consider adding a comment", diagnostics/logging and cleanup suggestions — unless `REVIEW.md` explicitly asks for them.
- Anything you couldn't tie to a specific line with a plausible trigger.
- Duplicates (keep the clearest instance; see "bug class" below).

When one bug pattern repeats, report it **once as a class** ("unchecked `fetch` response — 4 call sites: a.ts:12, b.ts:40, …") so the author fixes every instance in one round.

## Step 4 — Tag and rank

Every surviving finding gets exactly one tag:

- **Important** — a real bug or regression this change introduces or exposes. Must be fixed or explicitly resolved before merge.
- **Nit** — real but minor. Cap at **3** per review (or the cap in `REVIEW.md`); keep the most valuable.
- **Pre-existing** — a bug that was already there and this PR didn't introduce. Reported so it isn't lost, but framed so it doesn't read as the author's fault and doesn't block merge. Use `git blame` when unsure.

Sort Important by severity (data loss / security / crash > wrong output > degraded behavior), then Nits, then Pre-existing.

## Step 5 — Separate security pass

Run a distinct pass focused only on security — injection (SQL, shell, template, prompt), authn/authz gaps, secrets in code or logs, unsafe deserialization, SSRF/path traversal, missing input validation at trust boundaries, dependency changes, permission/scope widening, and CI workflow edits that could leak tokens. Verify findings the same way as Step 3. Security findings block merge exactly like Important ones.

## Step 6 — Assign a risk tier

Classify the **change**, not the author. Use `references/risk-tiers.md`. In short:

- **Low** — docs, copy, internal-only, additive behind a flag → no human approval needed once automated + security review are clean.
- **Medium** — ordinary product changes; sensitive areas like caching, updaters, system prompts → one human.
- **High** — highest-risk code and always-two-person paths → two humans.

Then ask a second question: *does this specific change need a person even though its tier says it doesn't?* Escalate if yes, and say why. Some paths always need a person regardless of tier: anything owners listed as sensitive (CODEOWNERS / `REVIEW.md`), CI workflow files, auth, billing, release/installer code.

Hard rules to state in the report when relevant:
- An agent's approval (including this review) never fills a required human slot.
- On an agent-written change, the person who directed the agent counts as one of the required reviewers and owns the outcome.
- Approval covers one commit. New unvouched commits (an agent push, a hand-resolved merge) need re-approval.
- Untrusted authors (not on the repo's trusted list) always need a second person.

## Step 7 — Convergence (re-reviews)

Agent author + agent reviewer can loop forever, mostly re-finding old code a little at a time. On re-review:

- **Round cap.** After round 3, stop listing findings and tell the author to step back and reconsider the approach instead.
- **Diff-only after round 2.** Review only what changed since the last reviewed commit, plus anything it directly affects.
- **No repeats.** Don't re-post a finding already raised; only note whether it's now fixed, still open, or resolved-with-reason.
- **Fixed means cleared.** If a later commit fixes an issue, mark it resolved.
- Prefer flagging bug classes (Step 3) so one round fixes all instances.

## Step 8 — Report

Use this exact structure. The summary line starts with the count and says plainly when there's nothing to fix.

```markdown
## Review — <short title> (round N, <Quick|Standard|Deep>)

**<X> important · <Y> nits · <Z> pre-existing · <S> security**  — or —  **Nothing to fix.** <one line why you're confident>

**Risk tier:** <Low|Medium|High> → <0|1|2> human approval(s). <one-line reason; note any escalation or always-human path>

### Evidence
- `<command>` → <result, key lines>
- <what couldn't be run, and what that means for confidence>

### Important
1. **<title>** — `path/file.ext:123`
   <what breaks, concrete trigger>
   **Fix:** <specific change>

### Security
...

### Nits (max 3)
...

### Pre-existing (not introduced by this PR)
...

### Before merge
- Every finding above must be answered: fixed, or resolved with a short reason (that's how a false positive is cleared).
- <PR template gaps: missing Testing / Risk / How to revert>
- <human approvals required and from whom (owners), if any>
```

Keep it tight. No praise padding, no restating the diff. If the change shouldn't exist at all (duplicates existing functionality, contradicts an earlier decision, a much simpler route exists), say that **first and early** as a direction question — not buried as a blocking comment at the end.

## Beyond a single review

Offer these when they fit — don't do them unasked:

- **Set up the process** → `references/adoption-checklist.md` (before-PR habits, PR template, approval rules, merge/rollout, measuring).
- **Write a `REVIEW.md`** → fill `assets/REVIEW.md.template` from the repo's actual failure modes.
- **Add a PR template** → `assets/PULL_REQUEST_TEMPLATE.md` (Testing / Risk / How to revert).
- **Same finding keeps recurring** → fix it at the source: a lint rule, a `CLAUDE.md`/`REVIEW.md` line, or a skill — not another review comment.
- **Learn a part of the codebase** → instead of reading trivial diffs, write a focused explainer of the module the user is about to change.
- **Improve the reviewer itself** → turn past incidents/reverts into an eval set; see `references/adoption-checklist.md` § "Improving the reviewer".
