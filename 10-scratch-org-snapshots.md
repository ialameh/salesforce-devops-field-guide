# 10. Scratch Org Snapshots

Snapshots are scratch orgs frozen at a point in time. New scratch orgs can be cloned from a snapshot in seconds instead of minutes. The right tool when scratch creation is slow enough to hurt.

## Why snapshots

A vanilla scratch org takes 60-90 seconds to create. A scratch org with Service Cloud, Communities, OmniChannel, plus your project's metadata can take 10-15 minutes. Multiply by 50 CI runs a day and the cost is real.

Snapshots solve this by capturing a fully-set-up scratch org's state. Subsequent scratch orgs cloning from the snapshot start in 30-60 seconds with everything already in place.

## How snapshots work

1. Create a scratch org normally.
2. Deploy your project's metadata, install dependencies, set up test data.
3. Run `sf org create snapshot` to freeze it.
4. Future scratch definitions reference the snapshot by name.
5. Scratch creation from the snapshot skips the slow setup phase.

```bash
# Step 1: Create the source scratch org
sf org create scratch --definition-file config/project-scratch-def.json --alias source-org

# Step 2: Set it up
sf project deploy start --target-org source-org
sf package install --package SomeDependency --target-org source-org
sf apex run --file scripts/seed-data.apex --target-org source-org

# Step 3: Snapshot it
sf org create snapshot --target-org source-org --name MyProjectSnapshot --description "Project metadata + seed data, refreshed weekly"

# Step 4: Reference it in a scratch definition
```

```json
// config/project-scratch-def-snapshot.json
{
  "orgName": "Dev from Snapshot",
  "edition": "Enterprise",
  "snapshot": "MyProjectSnapshot"
}
```

```bash
# Step 5: Create scratch orgs from the snapshot
sf org create scratch --definition-file config/project-scratch-def-snapshot.json --alias dev
```

The new scratch org has everything the snapshot had, ready to go.

## What snapshots capture

- Edition and features.
- All metadata in the source scratch org.
- Settings.
- Custom metadata records.
- Most data records (with caveats).
- Installed managed packages.

What they do not capture:

- The user (each scratch from snapshot gets its own user).
- Session state.
- Some encrypted secrets (e.g., named credential tokens).
- Audit trail history.

## Snapshot lifecycle

Snapshots are tied to a Dev Hub. They have a lifetime:

- **Active.** Available for new scratch orgs.
- **Inactive.** Hidden but recoverable.
- **Deleted.** Gone.

Each Dev Hub has a cap on active snapshots (typically 40 in current limits). Older snapshots get aged out if you exceed.

Plan to maintain a small set of active snapshots, recreated periodically.

## A typical snapshot strategy

For a mature project:

- **Weekly snapshot of the current main branch state.** Refreshed by a CI job every Monday.
- **Per-release snapshot when cutting a release.** Tied to a specific commit. Useful for reproducing release-time bugs.
- **Per-developer optional snapshots.** Some developers keep a personal snapshot for their work-in-progress.

The CI job is the highest-leverage:

```bash
# Weekly refresh job
sf org create scratch --definition-file config/project-scratch-def.json --alias snap-source
sf project deploy start --target-org snap-source --source-dir force-app
sf apex run --file scripts/seed-data.apex --target-org snap-source

# Mark old snapshots inactive
sf org list snapshot --target-dev-hub devhub | grep "MainSnapshot" | older-than 14d | xargs deactivate

# Create new
sf org create snapshot --target-org snap-source --name "MainSnapshot-$(date +%Y%m%d)" --description "Main branch as of $(date)"

sf org delete scratch --target-org snap-source --no-prompt
```

Subsequent scratch creations reference whichever snapshot is current.

## Snapshots vs shapes

A reminder of the distinction:

- **Shape** captures org-level configuration (features, settings). Useful when you need scratch orgs that look like production but are still empty.
- **Snapshot** captures everything: configuration, metadata, data. Useful when you want a ready-to-use environment.

You can combine them. The source scratch for a snapshot can itself be created from a shape. The resulting snapshot then has the shape's config plus your project's content.

## When snapshots are not the right tool

- **Vanilla setup, fast scratch creation.** If creating from scratch takes 90 seconds, snapshots add complexity for little benefit.
- **Highly variable test data.** If each test needs different data, snapshots constrain you. Use Apex test setup instead.
- **Source of truth concerns.** Snapshots are derived. They go stale. Don't treat them as the canonical state.

## Pitfalls

### Stale snapshots

The most common issue. Code in the snapshot diverges from your repo's `main` branch over a week or two. Developers create scratch orgs from the snapshot, find they have outdated metadata, get confused.

Mitigation: refresh snapshots automatically (CI job). Document the cadence.

### Snapshot caps

If you create snapshots without aging old ones, you hit the cap and new snapshot creation fails. Set up a scheduled cleanup.

### Auth state

Snapshots clone the source org. The source org's authenticated state does not transfer cleanly. Each scratch from snapshot gets a new user, and you need to set up auth (named credentials, connected apps) per scratch.

For tokens that survive: use Salesforce-native auth flows (OAuth, JWT bearer) that re-establish on first use. For secrets: have a setup script that injects them after scratch creation.

### Managed package versions

If your snapshot has a managed package version installed and you upgrade the package, scratch orgs from the old snapshot still have the old version. Refresh the snapshot when upgrading dependencies.

## Snapshots in CI

A common CI optimisation:

```yaml
- name: Create scratch org from snapshot
  run: |
    sf org create scratch \
        --definition-file config/project-scratch-def-snapshot.json \
        --alias ci-org \
        --duration-days 1
- name: Run tests
  run: sf apex run test --target-org ci-org --code-coverage --result-format junit
- name: Cleanup
  if: always()
  run: sf org delete scratch --target-org ci-org --no-prompt
```

The PR pipeline starts in 30-60 seconds instead of 5-10 minutes. Worth it for high-frequency CI.

## Building a snapshot pipeline

A simple, robust pattern:

```bash
#!/bin/bash
# refresh-snapshot.sh
set -euo pipefail

SNAPSHOT_NAME="MainSnapshot-$(date +%Y%m%d-%H%M)"
DEV_HUB="${DEV_HUB:-devhub}"

# Create source scratch
sf org create scratch \
    --definition-file config/project-scratch-def.json \
    --alias snap-source \
    --duration-days 1 \
    --target-dev-hub "$DEV_HUB"

# Set it up
sf project deploy start --target-org snap-source --source-dir force-app
# ... install packages, deploy data, etc.

# Snapshot it
sf org create snapshot \
    --target-org snap-source \
    --name "$SNAPSHOT_NAME" \
    --target-dev-hub "$DEV_HUB" \
    --description "Auto-generated $(date)"

# Cleanup
sf org delete scratch --target-org snap-source --no-prompt

# Update scratch definition to reference latest
jq ".snapshot = \"$SNAPSHOT_NAME\"" config/project-scratch-def-snapshot.json > tmp.json
mv tmp.json config/project-scratch-def-snapshot.json
```

Run weekly via CI. Keeps the snapshot fresh, the definition file referencing the latest.

## When to introduce snapshots in your project

Two signals:

- Scratch creation has gotten slow enough that developers complain.
- CI runs are dominated by scratch setup time.

Until those signals appear, don't bother. Snapshots add maintenance overhead. Adopt when the maintenance is cheaper than the setup cost.

## References

- [Scratch Org Snapshots](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_snapshots.htm)
- [`sf org create snapshot`](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_org_commands_unified.htm)
- [`sf org list snapshot`](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_org_commands_unified.htm)
- [Dev Hub limits](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_setup_devhub.htm)
