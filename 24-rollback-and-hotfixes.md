# 24. Rollback and Hotfixes

A bad release is going to happen. The plan you make before it does decides whether the recovery is minutes or days.

## Rollback options, by cause

### Apex compile or test failure during deploy

The deploy was rejected. Production unchanged. No rollback needed.

Fix: address the root cause, redeploy.

### Deploy succeeded but introduces a runtime bug

Production is running the new code. Bug is happening. Need to revert.

Options:

1. **Forward fix.** Identify the bug, fix it, deploy a patch.
2. **Reverse deploy.** Deploy the previous source state.
3. **Toggle off.** If the new code is gated by a feature flag, turn it off.

Forward fix is usually fastest if the bug is small. Reverse deploy works if the change was big and identifying the specific bug takes too long. Toggle off is the safest if you have feature flags.

### Bad data created by a deploy

The deploy ran, executed Apex, created or modified records. Now those records are wrong.

Recovery is data-specific:

1. **Identify the affected records.** SOQL.
2. **Either delete or revert.** If the data should not exist, delete. If the data was a corruption, restore from backup.
3. **Patch the code that caused it.** Otherwise the next deploy makes it worse.

Have a SOQL query in the rollback runbook that identifies affected records.

### Wrong data deployed

Custom metadata records, picklist values, configuration that should not have changed.

Forward fix: deploy the correct values back over.

### Bad managed package version installed

For Managed packages, you cannot uninstall mid-conversation if customers depend on it. Options:

1. Push a patch (PATCH bump fixing the bug).
2. Promote a new MINOR with a deprecation.
3. In the worst case, push a forced upgrade with `--upgrade-type Mixed` to remove the offending parts.

Test patches in a staging org before pushing to subscribers.

## The reverse deploy

The most common rollback for org-based deploys.

```bash
# 1. Identify the previous good state
git log --oneline -- force-app/main/default

# 2. Check out the previous version
git checkout <previous-good-sha> -- force-app

# 3. Validate against production
sf project deploy validate \
    --source-dir force-app \
    --target-org production \
    --test-level RunLocalTests

# 4. Quick deploy
sf project deploy quick --use-most-recent --target-org production
```

This is slower than a forward fix but more deterministic. You're returning to a state that worked.

The catch: anything between the old state and the current state has to also be reverted. If the bad deploy added a custom field that other code now uses, removing the field breaks the other code. You'd need to either keep the field (just revert the new code) or revert all the changes.

For this reason, frequent small deploys are easier to reverse than infrequent big ones.

## The hotfix branch

When production is broken and the fix needs to ship now, you cannot wait for the normal release cycle.

```bash
# Branch from the production-equivalent commit
git checkout main
git checkout -b hotfix/bug-description

# Make the fix
# Test
git commit -am "Fix: account lookup performance"

# Tag for release
git tag hotfix-1.2.1
git push origin hotfix-1.2.1
```

Ship the hotfix branch through:

1. Validate against production (or production-shaped sandbox).
2. Smoke test.
3. Deploy.
4. Merge the hotfix back to main so next release includes it.

## Hotfix discipline

Hotfixes skip approval gates. They have to. They also bypass much of your normal release verification. So:

- **Limit hotfixes to genuine emergencies.** Production down. Customer-impacting bug. Compliance issue.
- **Document every hotfix.** Why was it deployed outside the normal process? Who approved it?
- **Always merge back.** A fix that exists only in production but not in main means main will lose the fix on the next release.
- **Post-mortem.** What slipped through? How do we prevent it?

## Feature flags

The best rollback tool is one you build before you ship.

A feature flag is a custom metadata record that gates whether a new feature is active:

```apex
public class FeatureFlags {
    public static Boolean isEnabled(String featureName) {
        Feature_Flag__mdt flag = Feature_Flag__mdt.getInstance(featureName);
        return flag != null && flag.Is_Active__c == true;
    }
}

// Usage
if (FeatureFlags.isEnabled('NewOrderProcessor')) {
    NewOrderProcessor.process(orders);
} else {
    OldOrderProcessor.process(orders);
}
```

Ship the new code with the flag set to false. Flip the flag in production via Setup (changes a custom metadata record, no deploy). If the new path misbehaves, flip the flag back.

This converts a code rollback into a configuration change. Configuration changes are seconds. Code rollbacks are hours.

Use feature flags for:

- Risky logic changes.
- Performance-sensitive changes.
- Anything where you want a fast revert option.

Don't use feature flags for:

- Trivial changes (overhead not worth it).
- Permanently dual-pathed code (clean up the flag once the new path is proven).

## Backup and restore

Before any risky operation:

### Metadata backup

```bash
# Specific component
sf project retrieve start \
    --metadata "ApexClass:OrderProcessor" \
    --target-org production \
    --target-metadata-dir backups/$(date +%Y%m%d)

# Whole-org snapshot (slow)
sf project retrieve start \
    --manifest manifest/full-org.xml \
    --target-org production \
    --target-metadata-dir backups/$(date +%Y%m%d)
```

### Data backup

For records:

```bash
# Specific records
sf data export tree \
    --query "SELECT Id, Name, Status__c FROM Order__c" \
    --target-org production \
    --output-dir backups/$(date +%Y%m%d)

# Full data export (use Setup, Data Export for the official path)
```

Salesforce also offers a weekly data export (Setup, Data Export). Subscribe to it. It's free and gives you a baseline.

For mission-critical data, supplement with a backup tool: OwnBackup, Spanning, Gearset Backup. They version data over time and provide point-in-time recovery.

## Post-incident review

After every rollback or hotfix:

1. **What broke.** Specific behavior, customer impact, time to detection.
2. **Why it broke.** Root cause. Was it a bug, a process gap, a tooling failure?
3. **What helped.** What recovery action worked?
4. **What didn't help.** What was tried that wasted time?
5. **Action items.** Concrete changes to prevent recurrence.

The action items are the point. Write them down. Track them. Verify they're done.

A useful rule: every incident must produce at least one action item that, if done before the incident, would have prevented or accelerated recovery.

## A rollback runbook template

```markdown
# Production rollback runbook

## Decide
- Confirm the issue is production-impacting.
- Identify the root cause if possible.
- Decide: forward fix or reverse deploy?

## Forward fix path
- [ ] Reproduce locally.
- [ ] Write the fix.
- [ ] Deploy through hotfix process.

## Reverse deploy path
- [ ] Identify the last good commit (Git tag).
- [ ] Check out that state in a working branch.
- [ ] Validate against production.
- [ ] Quick deploy to production.
- [ ] Verify production is restored.
- [ ] Merge the hotfix branch to main.

## Communicate
- Stakeholders informed.
- Status page updated (if customer-facing).
- Internal channel notified.

## Post-incident
- Schedule review.
- Capture timeline, root cause, action items.
- Track action items to completion.
```

Print it. Tape it next to your monitor.

## Practice

The runbook is only useful if the team can execute it under pressure.

Practice:

- Run a "fire drill" once a quarter. Pretend production is broken. Walk through the runbook.
- Have multiple people who can run a production deploy. Single point of failure on a hotfix is worse than the original bug.
- Document any deviation from the runbook. Update the runbook.

The first real incident is much better when you've practiced.

## References

- [Apex feature flags pattern (community)](https://developer.salesforce.com/blogs/2018/02/feature-management.html)
- [Salesforce data backup options](https://help.salesforce.com/s/articleView?id=sf.admin_exportdata.htm)
- [Quick deploy](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_deploy_quick.htm)
- [Salesforce Support Cases](https://help.salesforce.com/s/) for production-impacting platform issues.
