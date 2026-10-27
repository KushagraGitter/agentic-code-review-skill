# Adopting the layered review process

Think of review as layers, not one gate. Each layer misses some things; few problems get through all of them, and fast rollback handles the rest.

```
verification → required CI → AI review → security review → risk-based human approval → merge queue + flags → fast rollback
```

## Before the pull request
- [ ] Give the agent checks it can run (tests, build, lint, screenshot diff) and list them in a short `CLAUDE.md`. Ask it to show output, not claim success.
- [ ] Add hooks that auto-run formatters/linters.
- [ ] Run a local review (`/code-review`, `/security-review`, or a fresh-context subagent over the diff) before opening the PR.
- [ ] Add a `REVIEW.md`; whenever the same review comment repeats, move it there (or into a lint rule).

## On every pull request
- [ ] Require CI: lint, types, tests on every supported platform, build/packaging. Red blocks merge.
- [ ] PR template with **Testing**, **Risk**, **How to revert** (see `assets/PULL_REQUEST_TEMPLATE.md`). An agent may fill it; the requesting human signs off.
- [ ] AI reviewer on one repo first; tune `REVIEW.md` for a few weeks; then widen.
- [ ] Automated security review; require every review thread resolved (fixed or closed with a reason) before merge.
- [ ] Match depth to risk (Quick / Standard / Deep).
- [ ] Cap review rounds (~3); after the first couple, re-review only the new diff.

## Approval
- [ ] Write down risk tiers set from the change, not the author (`references/risk-tiers.md`).
- [ ] Lighter paths only for trusted authors (usually the owning team).
- [ ] CODEOWNERS + branch protection so auth, billing, CI/release workflows and installers always need a person.
- [ ] Bot approvals never replace a required human; dismiss approvals on new commits.

## Merge and after
- [ ] Merge queue that re-runs checks against latest main with the change applied.
- [ ] Feature flags; internal dogfooding first; staged release channels (e.g. latest → stable).
- [ ] One-step rollback; flags that turn off without a deploy; on-call that watches revert/incident dashboards.
- [ ] Incident reviews ask: *which layer should have caught this?* Then fix that layer.

## Team norms that make lighter review safe
- Small changes (easy to review, debug, revert).
- Everything reversible within minutes.
- Blast radius sets the bar — expensive code gets more people and checks.
- "You land it, you own it" — the author usually fixes what breaks; it's context, not blame.
- Raise "should we build this?" early in the thread, not as a blocking comment on a finished PR. Closing a cheap-to-write PR isn't a loss.

## Measuring
- Human time: approvals per PR (median, p95), human comments, time a PR waits on a person.
- Guardrail: revert rate and incident rate. If these rise, you didn't save time — you moved it.
- Convergence: rounds-to-pass, share of PRs passing after one round, share needing 5+ rounds.
- Noise: classify every reviewer comment by the author's response — fixed / wrong / intentional / didn't matter. Also run the reviewer on known-clean PRs and count findings that would have led to a harmful change.
- Written priority: catch real bugs first, reduce review rounds second.

## Improving the reviewer
1. Collect past PRs that caused incidents or reverts, with their fix-forwards.
2. Have engineers who know that code confirm each bug and where it lived.
3. Run the reviewer against this set on a schedule (e.g. daily).
4. Any change to the reviewer's prompt, tools, `REVIEW.md` or model is scored against the whole set — keep it if the score rises, drop it if not.
5. Each new incident becomes a new case.
6. When review misses something, ask what the model didn't have in front of it, and put that into `CLAUDE.md`, a skill, or a new check.
7. Optionally run review agents nightly over existing code to surface old bugs.
