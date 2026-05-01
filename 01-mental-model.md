# 01. Mental Model

Before any tactical chapter makes sense, you need the picture of where Salesforce code and configuration actually live. Not the official one. The honest one.

## The four stores of truth

![Salesforce DevOps: Four Stores of Truth](diagrams/devops-mental-model.svg)

Every Salesforce DevOps decision is a question about which of these the canonical answer lives in.

### 1. The source repository

Where developers commit. Apex, LWC, flows, layout XML, custom object metadata, all in source format. Plus the things that govern the project itself: `sfdx-project.json`, `.forceignore`, scratch org definitions, manifests, CI configs.

This is the one most teams treat as authoritative. They are mostly right.

### 2. Scratch and dev orgs

Where iteration happens. Scratch orgs are disposable, source-tracked, created fresh from definition. Developer sandboxes are longer-lived but still personal. Both expect the developer to push from source and pull what changes through the UI.

This is not a store of truth in the long term. It is a workspace.

### 3. The package registry

For teams that use 2GP unlocked or managed packages, the Dev Hub holds the truth about what versions exist, their ancestry, their dependencies. Versions are immutable once promoted. Ancestry chains are permanent.

This is canonical for what shipped, when, and what depended on what. Often underused.

### 4. The target orgs

Sandboxes, UAT, staging, production. Long-lived. Source tracking off. Years of state. Last touched by who-knows-what tool.

These are not stores of truth for code. They are the consequence of code shipping. Treat what is in production as the result of deployments, not the source of deployments.

## The mental shift

Most newcomers from non-Salesforce backgrounds think source is the truth. Most longtime Salesforce admins think production is the truth. Both are wrong in isolation. The reality:

- **Source is the truth for what should be in production.**
- **Production is the truth for what is currently running.**
- **The deployment pipeline is the bridge between them.**

A drift between source and production is not an oddity. It is a leak in your process. Investigating drift is a regular DevOps task.

## Two valid workflows

Beyond the four stores, every Salesforce DevOps engagement picks one of two workflows.

### Workflow A. Org-based deploys

Source → sandbox → sandbox → production via metadata API. No package management. Each environment receives a deploy.

When this fits:

- Small or medium internal projects.
- Single deployment chain.
- No managed package, no AppExchange.
- Codebase is small enough that a single deploy operation finishes in under 30 minutes.

When it stops fitting:

- Multiple production orgs need the same code.
- Deploy times balloon past an hour.
- You need to track exactly what version is running where.
- You're publishing on the AppExchange.

### Workflow B. Package-based deploys

Source → 2GP unlocked or managed package version → install in sandboxes → install in production. Each version is built once and installed many times.

When this fits:

- Multi-org installations.
- Versioning matters (compliance, audit, rollback).
- Multiple teams contributing to a shared codebase.
- AppExchange.

When it doesn't:

- Tiny project.
- Single team, single org chain.
- The team has not yet built the muscle for it (packaging adds a learning curve).

The transition from A to B is painful and slow. Most teams that wish they had picked B started with A and stuck with it for too long. Most teams that picked B too early regret the upfront complexity.

The honest rule: default to A unless you know you need B. Revisit when A starts hurting.

## The DevOps maturity curve

Where most Salesforce teams sit, in roughly increasing order of maturity:

1. **Change Sets.** Manual UI-driven deploys. Source not in version control. Painful for any team larger than two.
2. **Source-controlled but manual.** Source is in Git, but deploys are still done by hand from a local machine. Common stage. Better than 1, not yet a real pipeline.
3. **Org-based CI/CD.** Pipelines deploy from source on merge. Sandboxes managed deliberately. Validate-then-quick-deploy used for production. This is the realistic target for most internal teams.
4. **Package-based CI/CD.** Versions are built once, installed many. Dependency graphs explicit. Releases are atomic. The standard for ISVs and large internal projects.
5. **Multi-package, multi-team.** Several packages with cross-dependencies, owned by different teams, integrated through a shared release calendar. The frontier; only large organisations.

The mistake teams make is trying to jump levels. Going from 1 to 4 in a quarter is a recipe for an abandoned migration. Each level requires the discipline of the previous one.

## Things that are easy to confuse

A short list of distinctions that catch people. Every chapter that follows assumes you know these.

### Source format vs metadata format

Source format is what `sf project retrieve` produces. One file per LWC component, one folder per custom object, files split for diff-friendly review. Metadata format is what the metadata API consumes: a flat ZIP of XML files. The CLI converts between the two.

You author in source format. You deploy in either, but the CLI handles the conversion.

### Push vs deploy

`push` works against source-tracked orgs (scratch orgs, source-tracked sandboxes). It diffs your source against the org and pushes only what changed. `deploy` works against any org. It deploys what you tell it to deploy, whether or not the org has it already.

Use `push` for scratch org iteration. Use `deploy` for everything else.

### Pull vs retrieve

`pull` is the opposite of `push`. Source-tracked. Returns what changed in the org since the last sync. `retrieve` is the opposite of `deploy`. Pulls whatever you ask for, regardless of source tracking.

### Dev Hub vs target org

The Dev Hub is the org you authenticate `--set-default-dev-hub` against. It is where scratch org templates live and where 2GP packages are administered. It is not where your code runs. Target orgs are where the code runs (sandboxes, production).

A common mistake: treating the Dev Hub as a deployment target. It is not.

### Sandbox vs scratch org

Both are non-production Salesforce orgs you develop against. Sandboxes are long-lived copies of production, refreshed periodically. Scratch orgs are short-lived, defined by config, created on demand.

Different lifetimes, different uses, different DevOps treatment. Don't mix the patterns.

## The first decision

In any new Salesforce DevOps engagement, the first decision is which workflow (A or B above) you are going to use. The second is what your branching strategy looks like. The third is what your sandbox layout is.

Get those three right and most of the rest of the discipline is mechanical. Get them wrong and you spend a year repaying technical debt.

## References

- [Salesforce DX Developer Guide: Project structure](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_ws_intro.htm)
- [Source vs metadata format](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_source_file_format.htm)
- [Scratch org overview](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs.htm)
- [Sandbox types](https://help.salesforce.com/s/articleView?id=sf.data_sandbox_environments.htm)
- [2GP overview](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp.htm)
