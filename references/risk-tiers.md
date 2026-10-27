# Risk tiers and review depth

The change sets the tier, not who wrote it. Start strict and loosen only with evidence (revert and incident rates staying flat).

## Human-approval tiers

| Tier | Typical changes | Human approvals |
|---|---|---|
| **Low** | Docs, copy/strings, internal-only tooling, purely additive code behind a feature flag that defaults off, test-only changes | **0** once automated review + security review are clean and CI is green |
| **Medium** | Ordinary product changes; sensitive-but-common areas: caching, auto-updaters, system prompts, feature-flag logic, shared utilities | **1** |
| **High** | The riskiest code; paths owners mark as always-two-person | **2** |

### Always need a person (any tier)
- Paths owners listed as sensitive (CODEOWNERS, `REVIEW.md` "always human" list)
- CI / release workflow files (e.g. `.github/workflows/**`) — they can leak secrets or ship anything
- Auth, permissions, billing/payments
- Installers, updaters, release and packaging scripts
- Data migrations that drop/rewrite data; anything hard to reverse

### Escalation signals (bump a tier or require a person)
- Change is large (rough guide: >1,000 lines usually has findings; small ones usually come back clean)
- Not behind a flag and not trivially revertible
- Touches a public API/contract, persistence schema, or cross-service interface
- Sets an architectural pattern others will copy
- Author not on the repo's trusted list
- Review couldn't run tests / evidence is thin
- Bug fix where root cause is unclear

### Rules that don't bend
- A bot/agent approval never counts as a required human approval.
- The person who prompted an agent-written change is one of its reviewers and owns it.
- Approvals are per-commit; new unvouched commits need re-approval.

## Review depth

| Depth | When | What changes |
|---|---|---|
| **Quick** | Low tier, small diffs | Correctness + conventions lenses, single verification pass, skip Deep tracing |
| **Standard** | Default | All lenses, every finding verified, separate security pass |
| **Deep** | High tier, sensitive paths, or a missed bug costs more than the review time | All lenses with whole-repo tracing, multiple independent verifiers per finding, attempt repros, stronger root-cause check on fixes |
