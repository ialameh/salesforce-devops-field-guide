# 27. Anti-Patterns

A list of common mistakes in Salesforce DevOps. Each one was a real practice somewhere. Each one was followed by months of pain. Avoid.

## Source-of-truth confusion

### Treating production as the source of truth

A team makes changes in production via the Setup UI. Sometimes someone retrieves them. Sometimes they don't. The repo lags reality.

Symptom: deploys to other environments don't include changes that are already in production. Drift accumulates.

Fix: source is the truth. UI changes get retrieved into source as part of the change. If retrieving every change is too painful, the discipline isn't working; consider locking down production UI access.

### Treating one sandbox as the source of truth

A long-lived sandbox where everyone authors. Refresh is feared because it would delete the canonical state.

Symptom: refresh paralysis. Sandbox diverges from production. The team grows afraid to refresh.

Fix: source is the truth. Sandboxes are deployment targets, not authoring environments.

### "We'll commit it later"

UI changes that will, theoretically, be committed at some unspecified point.

Symptom: undocumented changes accumulate. Different orgs disagree about reality.

Fix: retrieve and commit immediately, or don't make the change.

## Pipeline mistakes

### Rebuilding the artifact at each stage

The package version (or source bundle) built for QA is not the one shipped to production. Production has an artifact that nobody validated.

Symptom: "It worked in QA" bugs.

Fix: build once, deploy many. The same package version flows through all stages.

### Auto-deploy to production

CI deploys to production on every merge to main. No human in the loop.

Symptom: someone merges a bug Friday at 5pm. Production is broken until Monday.

Fix: human gate before production. Even one approval is enough friction.

### No staging environment

Direct from QA to production with no intermediate. Production is the first place "production-like" testing happens.

Symptom: production-only bugs.

Fix: have a UAT or staging org. Full sandbox refreshed near production.

### Skipping validate

Production deploys go straight without `validate` first. Tests run during the deploy. Half an hour later, you find out it failed.

Symptom: long deploy windows. Production stuck mid-deploy.

Fix: validate first, then quick-deploy. Five minutes versus ninety.

### CI without notifications

Builds fail silently. Nobody notices for days.

Symptom: regressions ship because nobody saw the broken build.

Fix: wire notifications. Slack, email, anything. Failed builds scream.

## Branching mistakes

### Long-lived feature branches

A branch for a "big feature" that runs for two months. Diverges from main. Rebases get harder. The merge becomes radioactive.

Symptom: nobody wants to merge it. It dies.

Fix: small, frequent merges with feature flags for incomplete work.

### Environment branches

`main` for production, `qa` for QA, `integration` for integration. Promotion requires merging from one to the next.

Symptom: branches diverge over time. Promoting becomes painful rebase work. Confusion about which branch is canonical for what.

Fix: trunk-based with tags. `main` is authoritative; environments deploy specific tags or commits.

### Force-pushing to main

Someone rewrites history on `main`. Other developers' work is lost.

Symptom: panic. Investigations. Sometimes data loss.

Fix: branch protection rules. No force-push to main. Ever.

## Packaging mistakes

### Skipping packaging when you should use it

Project has multiple production orgs that should run identical code. Team uses org-based deploys, ships slightly different metadata to each one. Drift.

Symptom: "It works in customer A's org but not customer B's".

Fix: 2GP unlocked packages. Build once, install many.

### Using packaging when you shouldn't

Small project, single org, no real need for atomic deploys. Team adopts 2GP because it sounds modern.

Symptom: complexity for no benefit. Slow iteration.

Fix: org-based deploys. Move to packaging when there's a clear reason.

### Ignoring ancestry in Managed packages

Managed package with constant breaking changes. Customers can't upgrade.

Symptom: upgrade failures, customer churn, support burden.

Fix: design APIs for backward compatibility. Use `@Deprecated` and patches. Plan major version bumps.

### Mixing 1GP and 2GP without a migration plan

Someone built a 2GP package that depends on a 1GP managed package. The two packaging systems don't fully cooperate.

Symptom: install failures, dependency hell, ancestry conflicts.

Fix: pick one. If migrating from 1GP to 2GP, follow Salesforce's documented path.

## Test mistakes

### Tests that don't actually test

`System.assert(true);` or no assertions at all. Tests pass but verify nothing.

Symptom: production bugs that "should have been caught".

Fix: every test should have meaningful assertions about behaviour.

### Tests that depend on org state

A test that assumes a specific record exists or a specific user is set up.

Symptom: tests pass in dev, fail in CI, pass in QA, fail in production.

Fix: hermetic tests. Each test creates its own data.

### Tests that hit external systems

A test that calls a real REST API. Flaky, slow, depends on network.

Symptom: random failures.

Fix: `Test.setMock` with `HttpCalloutMock`.

### "It's only 75%, that's fine"

Coverage is at the minimum. New code doesn't add tests. Coverage drifts down to 76, 75, 74. The deploy fails.

Symptom: under-tested code, blocked deploys.

Fix: aim for 90%. Treat 75% as a hard floor, not a target.

### Disabling tests to make the deploy pass

A test fails. Someone marks it `@isTest(SeeAllData=true)` or comments it out.

Symptom: real bugs slip through. The test was right.

Fix: investigate the failure. Fix the code or the test. Don't disable.

## Sandbox mistakes

### Shared development sandbox

One sandbox for all developers. They step on each other's work.

Symptom: someone's deploy overwrites someone else's. Trust collapses.

Fix: per-developer scratch orgs. Or per-developer Developer sandboxes. Isolation.

### Sandbox as production-equivalent

Treating a Developer sandbox as if it had production data and load. It doesn't.

Symptom: performance bugs in production.

Fix: Full sandbox for performance testing.

### Forgetting to reset email deliverability

Sandbox sends emails to real customer addresses after refresh.

Symptom: angry customers receiving test emails.

Fix: post-refresh checklist with deliverability disable. Automate.

### Refreshing during UAT

Refreshing a UAT sandbox while stakeholders are using it.

Symptom: lost UAT work, frustrated stakeholders.

Fix: schedule refreshes in coordination with the team. Communicate before.

## Auth mistakes

### Production password in CI

Hardcoding (or storing without secrets management) a production password in CI scripts.

Symptom: leaked credentials when CI logs are accessed by the wrong person.

Fix: SFDX auth URL or JWT. Stored as encrypted CI secrets.

### Personal admin user as CI user

CI authenticates as your admin user. Audit logs say you did everything.

Symptom: indistinguishable manual changes from CI changes.

Fix: dedicated CI user. Separate audit trail.

### Credentials in commit messages

Someone pasted an SFDX auth URL into a commit message for "documentation".

Symptom: credential exposure.

Fix: train the team. Use `git-secrets` or similar to scan commits. Rotate the leaked credential.

## Operational mistakes

### No rollback plan

Production breaks. Nobody knows how to revert. Investigation takes hours; recovery takes hours more.

Symptom: long incidents.

Fix: documented rollback runbook. Practiced before the real incident.

### Single point of failure

Only one person knows how to deploy to production.

Symptom: that person goes on vacation, deploys are blocked. Or they leave; the team is paralysed.

Fix: at least two people who can run any production operation. Document the process. Practice.

### No post-mortems

Production incidents get fire-fought, then forgotten. The same class of bug recurs in three months.

Symptom: repeated incidents.

Fix: every meaningful incident gets a written post-mortem with action items. Track the action items.

### Deploy on Friday at 5pm

Maximum stress, minimum staff. If something breaks, it's the weekend.

Symptom: weekend incidents.

Fix: deploy on Tuesday or Wednesday morning. Avoid Friday afternoons.

## AI tool mistakes

### Production credentials in AI agent context

A developer authenticates an AI agent to production for "convenience". The agent's context goes to the AI provider's servers.

Symptom: credential leak risk; agent could execute destructive commands by mistake.

Fix: see [Chapter 25](./25-ai-assisted-development.md). AI agents authenticate to scratch and dev sandboxes only.

### Pasting production data into AI prompts

A developer pastes a production record's data into an AI prompt for help debugging.

Symptom: PII or sensitive data in AI provider's logs.

Fix: synthesise example data. Strip PII before sharing.

### Accepting AI suggestions without review

AI generates code; developer accepts the diff without reading carefully. Bug ships.

Symptom: lower code quality, hidden bugs, AppExchange security review failures.

Fix: AI suggests, human reviews, CI verifies. Three independent checks.

## Communication mistakes

### Quiet deployments

Production gets deployed without anyone announcing it. Two days later, support is fielding bug reports related to the change.

Symptom: confusion. Stakeholders feel out of the loop.

Fix: announce production deploys in a team channel. Even routine ones.

### Hidden incident state

Production is broken; the team is investigating; nobody outside the team knows.

Symptom: customers find out from your customers.

Fix: a status page or Slack channel that broadcasts incident state. Keep it updated.

### No release notes

Customers don't know what changed. Support doesn't know what's new.

Symptom: confusion about whether a behavior change is a bug or a feature.

Fix: a CHANGELOG. One entry per release. What changed, what's new, what's deprecated.

## Documentation mistakes

### Documentation in code comments

Specifications buried in Apex comments. Out of sync within weeks.

Symptom: code comments contradict reality.

Fix: documentation in `docs/` or a wiki. Code comments cover only intent and non-obvious decisions.

### "Read the code"

The answer to every question is "look at the source".

Symptom: new team members take months to ramp. Knowledge silos.

Fix: write the architectural overview. Write the runbook. Write the pattern library.

### Wikis nobody reads

A wiki page exists but isn't linked from anywhere. Nobody finds it. They ask the same question again next quarter.

Symptom: knowledge entropy.

Fix: link the wiki from the README. Update both as the project evolves.

## When you spot an anti-pattern

A few rules:

- Don't shame anyone. Anti-patterns happen because they were the easiest path at the time.
- Document the cost. "We've spent N hours on this issue" makes the case for change.
- Propose a smaller next step. Going from "no CI" to "full pipeline" is too big. "PR validation" is achievable.
- Track the change to completion. Half-done migrations are worse than no migration.

The discipline of avoiding anti-patterns is a long game. Each one fixed compounds.

## References

- [Salesforce DX best practices](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm)
- [Apex code best practices](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_pattern.htm)
- [Salesforce Architects: well-architected framework](https://architect.salesforce.com/well-architected/overview)
- [DevOps culture and practices (broader)](https://martinfowler.com/articles/cd-pipelines-101.html)
