# 06. Scratch Orgs From Zero

If you have never used a scratch org, this chapter is for you. By the end you will know what they are, why they exist, when they save you time, and when they will hurt.

## The one-paragraph version

A scratch org is a temporary, throwaway Salesforce org you spin up from a definition file, work in for a few hours or days, and delete. It is created from scratch (hence the name), wiped clean, with no production data, no customer records, no other developers' work. Source tracking is on by default. Versioned source code in your repo is the truth; the scratch org is a sandbox to test changes.

## Why scratch orgs exist

Salesforce historically gave developers Developer Edition orgs and developer sandboxes. Both have long lifetimes. Both accumulate state. Both diverge over time from production. After six months, what works in your sandbox may not work in production because the sandbox has drifted.

Scratch orgs solve the drift problem by being disposable. You create one to do work. You delete it when done. The next one starts fresh. Source is the only durable thing.

This pattern enables:

- **Reproducibility.** A new team member spins up a scratch org from the same definition; they see the same starting state.
- **CI.** Every CI run gets a fresh org. No interference between runs.
- **Feature isolation.** Two developers can each have a scratch org; their work does not collide.
- **Snapshot testing.** Spin up an org, run tests, snapshot the result.

## When scratch orgs are great

- Apex development, especially when iterating quickly.
- LWC development.
- Trying a new feature that may or may not work.
- CI pipelines.
- Demoing something without polluting a long-lived sandbox.
- Reproducing a bug starting from a clean baseline.

## When scratch orgs are not the right tool

- **Testing against real customer data.** Scratch orgs are blank. If your work depends on production data shapes you cannot synthesise, use a partial-copy or full sandbox.
- **Long-running integration testing.** Scratch orgs cap at 30 days. After that they vanish.
- **UAT.** Stakeholders want a stable URL. Scratch orgs come and go.
- **Performance testing at scale.** Scratch orgs do not have the same data volume or compute as production.
- **Anything where the org's history matters.** No history. Just-in-time creation.

For those cases, sandboxes still apply.

## Prerequisites

- Salesforce CLI version 2.0+.
- A Dev Hub. Either an existing Dev Hub-enabled production org, or a free Developer Edition with Dev Hub turned on (*Setup, Dev Hub, Enable*).
- Authenticated CLI. Run `sf org login web --set-default-dev-hub --alias devhub` once.

## Your first scratch org

```bash
# Create a scratch org from the project's default definition
sf org create scratch \
    --definition-file config/project-scratch-def.json \
    --alias dev \
    --duration-days 7 \
    --set-default

# Open it
sf org open --target-org dev

# Push your local source to it
sf project deploy start --target-org dev

# When done
sf org delete scratch --target-org dev --no-prompt
```

That is the entire lifecycle. Most days you will repeat these four commands many times.

## What a scratch org actually is

When you run `sf org create scratch`, the CLI:

1. Reads your scratch definition file.
2. Calls the Dev Hub.
3. The Dev Hub provisions a new org with the requested edition, features, and settings.
4. The CLI authenticates you to the new org.
5. The org appears in `sf org list` with the alias you provided.

Behind the scenes, the new org is a separate Salesforce instance. It has its own URL, its own user, its own database. From your perspective, it is identical in shape to any other org, just with no data or history.

## Lifetime and limits

- **Default lifetime: 7 days.** Configurable up to 30.
- **Hard cap: 30 days.** Cannot be extended. Salesforce deletes it.
- **Daily creation cap.** Each Dev Hub has a daily limit on how many scratch orgs it can create. Free tier is small (3 per day in some editions). Add-on packs increase it.
- **Concurrent active cap.** Each Dev Hub limits how many scratch orgs can exist simultaneously.

If you hit caps, either delete unused orgs or buy capacity. Most teams never hit them.

## What lives in a scratch org

- Your source, after `sf project deploy start` or `sf project deploy push`.
- Salesforce's standard config (objects, profiles, etc.).
- Any features you enabled in the scratch definition.
- Any data you create during your session.

What does not live in a scratch org:

- Production data.
- Production users.
- Long-term state. Anything you set up will vanish when the org is deleted.

## The retrieve discipline

If you change anything in the Setup UI of a scratch org, retrieve it back to source:

```bash
sf project retrieve start --target-org dev
```

Without this, the change exists only in that scratch org. The next time you create a scratch org, the change is gone.

This is the most important habit with scratch orgs. Source is the truth. The scratch org is a workspace.

## Common gotchas for newcomers

### "I can't find my data, where did it go?"

Scratch orgs are blank. There is no data to start with. You have to create test data each time.

Common pattern: a Data Loader script or an Apex test data factory that runs after `deploy` to populate the org. Add it to your team's onboarding script.

### "Why isn't my permission set assigned?"

Scratch orgs don't auto-assign permission sets to your user. After deploy, run:

```bash
sf org assign permset --name My_Permission_Set --target-org dev
```

### "My scratch org disappeared!"

It expired. Check `sf org list` for the expiration date when you create one. Set a reminder if you have ongoing work.

### "I can't push to it, source is stale"

Salesforce thinks your source and the org are out of sync. Either pull what you don't have, or force the push:

```bash
# Safer: pull first, see what's there
sf project retrieve start --target-org dev

# More dangerous: force push, may overwrite
sf project deploy start --target-org dev --ignore-conflicts
```

`--ignore-conflicts` is fine in scratch orgs, where there's no shared work to lose. Never use it in shared sandboxes.

## A typical day with scratch orgs

A common workflow:

1. **Morning.** `sf org create scratch --alias today`, deploy source, run any test data setup.
2. **Working.** Edit code locally. `sf project deploy start --target-org today` to push. Test in the UI.
3. **UI changes.** When you change a layout or a permission set in the UI, `sf project retrieve start --target-org today`.
4. **End of day.** Commit and push to Git. The scratch org keeps tomorrow's state, but Git is the canonical record.
5. **Next morning.** Continue with `today`, or create a new scratch org if you want a fresh state.

Some developers keep one scratch org alive for several days. Some recreate every morning. Both work. The discipline is committing source frequently so neither approach loses work.

## A team's day with scratch orgs

For a team, scratch orgs are usually:

- One per developer per current task.
- Plus one shared CI scratch org per pipeline run, freshly created and torn down each time.
- Plus an occasional shared scratch for ad-hoc collaboration, but most collaboration happens through PR review, not by sharing a scratch.

If your team is constantly trying to share scratch orgs, you are using the wrong tool. Spin one up per developer.

## The transition from "I never use them" to "they're my default"

Most Salesforce developers go through this transition. The triggers:

- They get burned by a sandbox-vs-production drift bug.
- They join a team that uses scratch orgs and learn the workflow.
- They start writing Apex tests and discover scratch orgs make test isolation easier.
- They set up CI and need disposable orgs for it.

Once you cross over, going back to long-lived sandboxes for development feels constraining. The discipline is a small price for the reproducibility.

## What this chapter does not cover

- The detailed format of the scratch definition file. See [Chapter 7](./07-scratch-org-definitions.md).
- Why source tracking works the way it does. See [Chapter 9](./09-source-tracking.md).
- Using scratch orgs in CI. See [Chapter 11](./11-scratch-orgs-in-ci.md).
- When scratch orgs go wrong. See [Chapter 12](./12-troubleshooting-scratch-orgs.md).

## References

- [Scratch Orgs overview](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs.htm)
- [Enable Dev Hub](https://help.salesforce.com/s/articleView?id=sf.sfdx_setup_enable_devhub.htm)
- [Scratch org creation CLI reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_org_commands_unified.htm)
- [Scratch org limits](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_create.htm)
