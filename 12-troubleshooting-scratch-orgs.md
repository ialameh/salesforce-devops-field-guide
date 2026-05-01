# 12. Troubleshooting Scratch Orgs

Things go wrong. Here is the diagnostic path for the most common scratch org failures, in roughly the order you will hit them.

## Creation fails

### "DAILY_LIMIT_EXCEEDED"

The Dev Hub has hit its daily scratch org creation cap.

Diagnosis:

```bash
sf org list limits --target-dev-hub devhub
```

Look for `ActiveScratchOrgs` and `DailyScratchOrgs`.

Fix: wait until tomorrow, or increase capacity (Salesforce sells expansion packs), or use snapshots so each scratch is reused.

### "feature X not supported"

A feature in your scratch definition is not allowed by your Dev Hub's licensing.

Diagnosis: read the error. It names the feature.

Fix: remove that feature, or use a Dev Hub that has the license. For ISVs, partner Dev Hubs allow more features.

### "INSUFFICIENT_ACCESS"

The user authenticated to the Dev Hub does not have permission to create scratch orgs.

Diagnosis: check the user's profile. They need `Create and Update Scratch Orgs` and access to ScratchOrgInfo records.

Fix: assign permission set or update profile.

### Creation hangs

Sometimes scratch creation seems to start and never finish. Common causes:

- Slow features (Communities, Knowledge, OmniChannel) are still provisioning.
- Salesforce backend is slow at the moment.

Diagnosis: query `ScratchOrgInfo` in the Dev Hub:

```bash
sf data query --target-org devhub \
    --query "SELECT Id, Status, ErrorCode, OrgName, CreatedDate FROM ScratchOrgInfo WHERE Status = 'Active' OR Status = 'New' ORDER BY CreatedDate DESC LIMIT 10"
```

Status values: `New`, `Active`, `Error`, `Deleted`.

Fix: if `Error`, the `ErrorCode` field has more detail. Otherwise, wait a bit longer; some features take 10+ minutes.

## Push or pull fails

### "Conflicts detected"

Source tracking sees changes in both your local source and the org since the last sync.

Diagnosis:

```bash
sf project deploy preview --target-org dev
```

Lists what's in conflict.

Fix options:

```bash
# Take org's version
sf project retrieve start --target-org dev --ignore-conflicts

# Take local version
sf project deploy start --target-org dev --ignore-conflicts

# Manually merge: pull, resolve in editor, push
```

In a personal scratch org, `--ignore-conflicts` is fine. In a shared org, never.

### "Source tracking is out of sync"

The CLI's local manifest disagrees with reality.

Symptoms: `pull` returns nothing despite UI changes; `push` says "no changes" despite local edits.

Fix:

```bash
sf project reset tracking --target-org dev
```

After reset, the next push or pull starts fresh.

If that doesn't fix it, delete the scratch org and create a new one. Faster than debugging.

### "Cannot deploy: validation failed"

A specific component fails validation. Apex compile error, missing field reference, malformed XML.

Diagnosis: read the error. The CLI is good at pointing at the component.

Common failure causes:

- Apex references a field that doesn't exist in the org.
- LWC references a component that's not deployed.
- A flow references an Apex class that's not deployed.
- A custom metadata type definition references fields that aren't there yet.

Fix: deploy the dependency first, or add the missing piece, or fix the reference.

### "Deploy too large"

Single deploy operations have size limits. Large monolithic deploys fail.

Diagnosis: how big is the source?

```bash
du -sh force-app
```

Fix: split the deploy. Use a manifest that targets only what changed:

```bash
sfdx sgd:source:delta --to HEAD --from origin/main --output manifest --generate-delta
sf project deploy start --manifest manifest/package.xml --target-org dev
```

Or split into multiple package directories.

## Tests fail in scratch org

### Tests pass locally, fail in CI's scratch

Different scratch definition. CI scratch is missing a feature or setting your local scratch has.

Fix: standardise the scratch definitions. Use the same one locally that CI uses.

### Tests pass in scratch, fail in sandbox

Scratch org is missing org-level state that the sandbox has. Common culprits: custom metadata records, named credentials, OAuth providers, license-gated features.

Fix: deploy the missing pieces, or configure them post-deploy in the scratch org.

### "INSUFFICIENT_ACCESS" inside a test

The test runs as a user without rights to a class or record.

Fix: assign the permission set in the test setup:

```apex
@TestSetup
static void setup() {
    User runningUser = [SELECT Id FROM User WHERE Username = :UserInfo.getUserName()];
    PermissionSetAssignment psa = new PermissionSetAssignment(
        AssigneeId = runningUser.Id,
        PermissionSetId = [SELECT Id FROM PermissionSet WHERE Name = 'My_Permission_Set'].Id
    );
    insert psa;
}
```

Or use `System.runAs` to elevate.

## UI behaves strangely

### "I made a change in setup, my scratch org doesn't see it"

The scratch org might be cached. Hard refresh (Cmd-Shift-R or Ctrl-Shift-R).

Or the change was applied to the wrong org. Check which org you're in via the user menu.

### Scratch org URL changed

When a scratch org reaches its expiration warning window, its URL may rotate. Re-open via:

```bash
sf org open --target-org dev
```

Don't bookmark scratch URLs.

### "My scratch org is gone"

Three possible reasons:

1. **Expired.** Scratch orgs have a maximum 30-day lifetime.
2. **Manually deleted.** Check `sf org list --all`.
3. **Dev Hub deleted it for limit reasons.** Older scratches sometimes get cleaned up.

`sf org list --all` shows expired and deleted orgs too.

Fix: create a new one. The data is gone; the source isn't, so deploy and continue.

## Dev Hub issues

### Dev Hub auth expired

Refresh tokens can expire if unused for a long time.

Fix:

```bash
sf org login web --set-default-dev-hub --alias devhub
```

Re-authenticate via browser.

For CI, regenerate the SFDX auth URL or rotate the JWT cert.

### Dev Hub user removed

If the user that created the scratch org is removed from the Dev Hub, the scratch org may be unreachable.

Fix: have multiple users with Dev Hub access, or use a service account.

### Dev Hub limits exceeded

```bash
sf org list limits --target-dev-hub devhub
```

Look for `ActiveScratchOrgs`. If at the cap, delete unused orgs:

```bash
# List candidates
sf org list --json | jq -r '.result.scratchOrgs[] | "\(.alias) \(.expirationDate)"'

# Delete a specific one
sf org delete scratch --target-org old-alias --no-prompt
```

## When to give up

A general rule: if you've spent more than 30 minutes troubleshooting a scratch org, delete it and create a new one. Scratch orgs are disposable for a reason.

The exception: if the failure mode reproduces on every new scratch org, the issue is in your source or scratch definition, not the org. Debug those.

## Diagnostics that often help

```bash
# CLI version
sf --version

# Authenticated orgs
sf org list

# Detail on a specific org
sf org display --target-org dev --verbose

# Scratch org info from Dev Hub
sf data query --target-org devhub \
    --query "SELECT Id, Status, ErrorCode, OrgName, ExpirationDate FROM ScratchOrgInfo WHERE Status != 'Deleted'"

# Local source tracking state
sf project deploy preview --target-org dev
sf project retrieve preview --target-org dev

# Limits
sf org list limits --target-org dev
sf org list limits --target-dev-hub devhub
```

When opening a Salesforce support case for scratch org issues, include all of these.

## A short anti-pattern to avoid

**Treating a scratch org as a sandbox.** Some teams resist deleting and recreating, hoping to avoid the setup cost. They leave a scratch org running for a week, treat it as their personal dev environment, accumulate state. Then it expires unexpectedly and they lose work.

The fix is twofold:

- Commit source frequently. The scratch org's state is not the source of truth.
- If scratch creation is too slow, fix that with snapshots, not by avoiding the lifecycle.

## When scratch orgs are not the answer

A few situations where the right answer is a sandbox, not a scratch org:

- Stakeholder UAT.
- Reproducing a bug that depends on production data shapes.
- Running scheduled jobs that take days.
- Long-running integration testing with external systems that need stable URLs.

Recognising these saves you from debugging "why won't this work in scratch" when the answer is "it shouldn't be in scratch".

## References

- [Scratch Org Troubleshooting](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_create_troubleshoot.htm)
- [`ScratchOrgInfo` object](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_scratch_orgs.htm)
- [Source tracking reset](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_project_commands_unified.htm)
- [Dev Hub limits](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_setup_devhub.htm)
