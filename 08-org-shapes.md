# 08. Org Shapes

A scratch definition specifies a few features and settings. Real production orgs have hundreds of fine-grained configurations that took years to accumulate. An org shape is a way to capture all of that and apply it to scratch orgs, so they look like production from the start.

## The problem org shapes solve

Without org shapes, your scratch definition lists what you remember to list. The scratch org has whatever the Dev Hub provisions plus those things. Production has all of that plus everything that has been turned on, configured, or licensed over the years.

Result: code that works in scratch fails in production because production has features, settings, or org-level configs that the scratch did not.

Org shapes capture the source-of-truth configuration of a "shape org" (typically your production org or a representative sandbox) and let scratch orgs mirror it.

## When org shapes earn their keep

- Your org has many features enabled that scratch defaults do not include.
- You have specific settings tuned over years that affect Apex or LWC behaviour.
- You're tired of "but it works in scratch" bugs.

## When you don't need org shapes

- Vanilla developer org with default features. Scratch definition handles it.
- A new project with no production org yet.
- Personal experiments.

For most internal teams, shapes are overkill until they're needed. For ISVs targeting many customer orgs, shapes are essential.

## How org shapes work

1. **Designate a shape org.** Usually production or a full sandbox.
2. **Capture the shape.** `sf org create shape` creates an `OrgShape` record in the Dev Hub representing that org's configuration.
3. **Reference the shape in your scratch definition.** Add a `sourceOrg` field pointing to the shape org's id.
4. **Create scratch orgs from the shape.** New scratch orgs are provisioned to match.

```bash
# In the shape org (e.g., production)
sf org create shape --target-org production
```

```json
// In your scratch definition
{
  "orgName": "Dev",
  "edition": "Enterprise",
  "sourceOrg": "00D...."  // the shape org's 18-character id
}
```

```bash
# Create from shape
sf org create scratch --definition-file config/project-scratch-def.json --alias dev
```

The new scratch org has the shape org's features, settings, and licenses, minus data.

## What a shape captures

- Enabled features and licenses.
- Org-level settings.
- Edition.
- Most setup-level configuration.

What it does not capture:

- Data (records, files).
- Users (just the new scratch user).
- Custom metadata records.
- Custom code (you deploy from source).
- Active sessions or transient state.

## Shape vs feature-based scratch definitions

| Aspect | Feature-based definition | Shape-based |
|--------|--------------------------|-------------|
| Configuration | Explicit feature list | Mirrors a shape org |
| Reproducibility | Same definition, same org | Shape can change as the source org changes |
| Maintenance | Update the JSON when features change | Recapture shape periodically |
| Drift risk | Explicit, reviewable | Implicit, hidden |
| First-time setup | Quick | Requires a shape org |
| Best for | Greenfield projects, ISV with controlled features | Teams with existing complex orgs |

A useful pattern: use feature-based for new projects, switch to shape-based when the org's complexity outgrows what you want to manually maintain.

## Shape lifecycle

A shape is a snapshot. It does not auto-update.

When the shape org gains a new feature (Salesforce releases a new licensed module, or you turn one on), existing shapes do not pick it up. You have to:

```bash
# Recreate the shape
sf org create shape --target-org production
```

This creates a new shape record. The Dev Hub keeps multiple shape records; the scratch CLI uses the most recent one matching `sourceOrg`.

Plan to recreate shapes:

- Quarterly, as a hygiene task.
- After major Salesforce releases.
- After turning on a new feature in the shape org.

## Limits and gotchas

- **Shape orgs must be Enterprise or higher.** Developer Edition is not supported.
- **Shape creation requires Dev Hub permissions.** The user running the command needs them.
- **Shape orgs and Dev Hubs can be different orgs.** The shape org can be a sandbox; the Dev Hub can be production.
- **Some features are not capturable.** Newer or rarely used features may not appear in shapes.
- **Slow scratch creation.** Shape-based scratches typically take longer to provision than feature-based ones.

## Snapshots vs shapes

Two related but distinct concepts:

- **Shape.** Captures *configuration* (features, settings, edition). Scratch orgs from a shape have the same setup but no metadata or data.
- **Snapshot.** Captures *configuration plus metadata plus optional data*. Scratch orgs from a snapshot are clones.

Snapshots are covered in [Chapter 10](./10-scratch-org-snapshots.md). They are heavier, slower to create, but reproduce your project's full state.

A typical setup:

- Shape captures the org configuration.
- Snapshot adds your project's metadata on top.
- Scratch orgs from the snapshot are ready to run, with both org config and project metadata in place.

## A practical pattern

For a mature project:

1. **Capture a shape from production once.** Recreate quarterly.
2. **Build a snapshot from a fresh scratch + your repo's source.** Recreate weekly via CI.
3. **Scratch orgs in CI use the snapshot.** Fast creation, accurate org shape.
4. **Developers use scratch orgs from the snapshot too.** Or from the shape directly if they want a clean baseline.

This combination gets you reproducibility (everyone starts from the same place) plus speed (snapshots are fast to instantiate).

## When shapes are not enough

For very complex orgs, shapes still miss things. Specifically:

- **Connected apps with secrets.** The connected app definition can deploy; the secret cannot.
- **Named credentials with OAuth tokens.** Same.
- **Email deliverability state.**
- **Org-wide email addresses.**
- **Some platform-level settings that require user interaction to fully apply.**

For these, document the manual steps needed after scratch creation. Add them to your setup script. Accept that scratch orgs cannot be pixel-perfect mirrors of production.

## A useful test

Once you have shapes set up, run this experiment:

1. Note ten things in production that took years to configure.
2. Spin up a scratch org from the shape.
3. Check whether each thing is present.

The first time, you will find some are missing. Note them, document them, automate them where possible. Repeat quarterly.

## References

- [Scratch Org Shape](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_shape.htm)
- [Create a Scratch Org from a Shape](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_create.htm)
- [`sf org create shape`](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_org_commands_unified.htm)
- [Scratch Org Snapshots](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_snapshots.htm)
