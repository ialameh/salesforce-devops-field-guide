# 09. Source Tracking

The mechanism that makes scratch orgs feel different from sandboxes. Source tracking is what lets you `pull` what changed in the org without naming it explicitly. It is also what bites people when they assume sandboxes work the same way.

## What source tracking does

When source tracking is on, Salesforce records every metadata change in an org. Whose user changed it, what they changed, and when. The CLI uses this to:

- Tell you what's different between your local source and the org.
- Pull only what changed since you last synced.
- Push only what changed since the org last synced.
- Warn you about conflicts.

Without source tracking, the CLI cannot tell what changed without asking the metadata API for everything and diffing it (slow, error-prone).

## Where source tracking is on

- **Scratch orgs:** on by default.
- **Sandboxes:** off by default. Can be turned on with the `Source Tracking in Sandboxes` setting (Spring '23 and later).
- **Production orgs:** off. Cannot be turned on.

Most teams enable source tracking on at least one sandbox (the integration sandbox) and leave others off.

## How `push` and `pull` work

In source-tracked orgs, the CLI maintains a local manifest of what version of each component the org has. When you run:

```bash
sf project deploy start --target-org dev
```

In a scratch org, this is implemented as `push`. The CLI:

1. Walks your local source.
2. Diffs each file against the last-known org state.
3. Sends only the changes.
4. Updates its local manifest after success.

Pull works the other way:

```bash
sf project retrieve start --target-org dev
```

1. CLI asks the org what changed since last sync.
2. Org returns a list.
3. CLI downloads those components.
4. Updates the local manifest.

Fast. Incremental. Magical when it works. Frustrating when the local manifest gets out of sync.

## How `deploy` works (no source tracking)

Against non-tracked orgs, the CLI cannot ask "what changed". It deploys what you tell it:

```bash
sf project deploy start --source-dir force-app --target-org production
sf project deploy start --manifest manifest/package.xml --target-org production
```

You must specify what to deploy. Either by directory (everything under that path) or by manifest (the listed components).

This is more verbose but explicit. For shared sandboxes and production, this is what you want.

## Conflicts

Source tracking detects conflicts. If both you and the org changed the same component since your last sync, the CLI refuses to push or pull without you choosing how to resolve.

Example:

```
ERROR: There are conflicts between local and remote changes.
```

Resolution options:

```bash
# Pull, accepting org's version
sf project retrieve start --target-org dev --ignore-conflicts

# Push, accepting your version
sf project deploy start --target-org dev --ignore-conflicts

# See what's in conflict
sf project deploy preview --target-org dev
```

`--ignore-conflicts` is dangerous. In a personal scratch org, it's fine. In a shared sandbox, you may overwrite a colleague's work.

## When source tracking goes wrong

The local manifest gets out of sync with reality. Symptoms:

- "I made a change in the UI but pull doesn't see it."
- "Pull shows changes I already pulled yesterday."
- "Push says nothing to deploy but the org doesn't have my latest code."

Two reset options:

```bash
# Force a full re-sync from the org's perspective. CLI re-queries everything.
sf project reset tracking --target-org dev

# Reset, then pull everything fresh
sf project reset tracking --target-org dev
sf project retrieve start --target-org dev
```

After reset, the next push or pull starts from a clean slate. Use sparingly.

## Source tracking in sandboxes

To enable source tracking in a sandbox:

1. *Setup, Dev Hub, Source Tracking in Sandboxes, Enable*.
2. Refresh the sandbox or wait for Salesforce to backfill.
3. The sandbox now behaves like a scratch org for source tracking purposes.

When this is useful:

- **Integration sandboxes** where one team works.
- **Personal Developer sandboxes** for individual developers.

When it's not:

- **UAT sandboxes** with stakeholders. They don't push or pull; they use the UI.
- **Full sandboxes refreshed from production.** Source tracking won't reflect the production state without extra work.
- **Production-equivalent sandboxes.** Treat these like production: deploy explicit lists.

## A common pattern: hybrid orgs

A team typically has:

- Many scratch orgs (source-tracked, disposable).
- One source-tracked integration sandbox (push/pull workflow).
- Several non-tracked sandboxes for QA, UAT, performance.
- Production (non-tracked, explicit deploys only).

Each org type uses the right command. Scratch orgs use `push` and `pull`. Sandboxes use a mix depending on tracking. Production uses `deploy` and `validate` and `quick`.

The mental shift: source-tracked orgs are workspaces, not source-of-truth. Non-tracked orgs are state-of-the-org, where deploys add or change things.

## Inspecting tracking state

```bash
# What does the CLI think has changed?
sf project deploy preview --target-org dev

# What's in the org that I don't have locally?
sf project retrieve preview --target-org dev

# Status overview
sf project info --target-org dev
```

These commands answer "what would push / pull do?" without doing it.

## Tracking and CI

In CI, a scratch org is created fresh every run. Source tracking starts clean. The CLI deploys your source, runs tests, deletes the org. No accumulated tracking issues.

This is one of the reasons CI loves scratch orgs.

## Tracking and source format

Source tracking works with source format. It does not really work with metadata format. If you push a metadata-format ZIP to a scratch org, the tracking state may not understand what was deployed.

Always work in source format with source-tracked orgs. Convert at the edges if you need metadata format for some other reason.

## Tracking and destructive changes

Destructive changes (deletions) are tracked too. If you delete a class in source and push, the org loses the class. If you delete a class in the org's UI and pull, your source loses the class.

The CLI is good about flagging deletions. It will not silently delete things you didn't ask it to.

## Tracking and external changes

If a managed package is installed into your scratch org, its metadata appears in tracking. Usually you do not want to pull it (it's not your code). The CLI knows to skip managed metadata in most cases, but not always.

If you keep pulling managed package metadata you didn't intend, add the namespace prefix to your `.forceignore`:

```
**/lumin__**
```

## Common newcomer questions

**"Why does push work in scratch but not in my full sandbox?"**
Source tracking. The full sandbox doesn't have it. Use `deploy` instead.

**"Why does the CLI keep asking about conflicts in my scratch org? I'm the only one using it."**
You probably switched between branches or rebased. The local source moved relative to what was in the org. Resolve once.

**"Can I use source tracking with two scratch orgs sharing the same source?"**
Each scratch org has its own tracking state in your local CLI cache. They don't conflict with each other, but you can confuse yourself by pushing different code to each.

**"Why is push slow on my project?"**
Either you have a lot of changes (pushing tens of MB), or your source has many small files (LWC tests, granular metadata). The CLI optimises but cannot parallelise everything.

## When to give up and reset

If you've spent more than 15 minutes fighting source tracking on a scratch org, delete the scratch org and create a new one. The new one starts with clean tracking. Faster than debugging.

Scratch orgs are disposable for a reason.

## References

- [Source Tracking](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_source_tracking.htm)
- [Source Tracking in Sandboxes](https://help.salesforce.com/s/articleView?id=sf.deploy_sandboxes_st.htm)
- [`sf project reset tracking`](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_project_commands_unified.htm)
- [Deploy and retrieve commands](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_project_commands_unified.htm)
