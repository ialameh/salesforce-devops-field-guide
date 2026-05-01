# 26. Troubleshooting

A symptom-to-cause map for common Salesforce DevOps failures. Each section covers a pattern, diagnostic steps, and the fix.

## Deploy fails

### "Cannot deploy: missing reference"

A component being deployed references something that does not exist in the target.

Diagnose:

- Read the error. It names what's missing.
- Check whether the missing component is in your source.

Fix:

- If the reference is in source, include it in the deploy.
- If the reference is from a managed package, ensure the package is installed.
- If the reference is leftover from a removed component, delete the reference.

### "Test failures"

The deploy ran tests; some failed.

Diagnose:

- Read the failure message. Identify the test class and method.
- Reproduce locally in a scratch org.

Fix: fix the test (preferably) or fix the code that the test exposes.

Don't disable failing tests to make a deploy pass.

### "Mixed DML"

Some metadata types cannot be deployed in the same transaction. Common culprit: User records with Permission Set Assignments.

Fix: split the deploy. First deploy users, then permission set assignments.

### "Deploy too large"

Single deploys have size limits. ~10000 components or ~39 MB.

Fix:

- Split into multiple smaller deploys.
- Use manifest-based incremental deploys.
- Move to package-based DevOps.

### "Component is referenced by an installed package"

You're deleting something a managed package depends on.

Fix: don't delete. Either keep it, or upgrade the managed package to one that doesn't reference it.

### "Cannot deploy: object/field is missing"

You're deploying Apex that references a field, but the field is missing in the target.

Fix: include the field in the deploy, or deploy the field first as a separate step.

### "Failure due to organization edition"

The metadata uses features unavailable in the target's edition.

Fix: target a higher edition, or remove the feature dependency.

## Validate fails

### "Validation succeeded but quick deploy fails"

Validation cache expired (4 days) or the org state has drifted between validate and quick deploy.

Fix: re-validate.

### "Validation timed out"

Big deploys with full test runs hit the timeout.

Fix:

- Increase `--wait` value.
- Run async; report later via `sf project deploy report`.

## Push fails (scratch org)

### "Conflicts detected"

Source tracking sees conflicts between local source and the org.

Fix: see [Chapter 9](./09-source-tracking.md).

### "Source tracking out of sync"

The local manifest disagrees with the org.

Fix:

```bash
sf project reset tracking --target-org dev
```

Or delete and recreate the scratch org.

## Authentication fails

### "Refresh token expired"

The CLI's stored auth no longer works.

Fix:

```bash
sf org login web --alias <alias>
```

Or for CI: regenerate the SFDX auth URL.

### "JWT signature invalid"

CI is using JWT but the cert doesn't match the connected app.

Fix:

- Verify the cert in the connected app matches the key CI is using.
- Check that the connected app is enabled for the username CI is using.

### "User not pre-authorized for connected app"

Connected app's policy requires pre-authorization, but the user isn't authorized.

Fix: in the connected app's policies, set "Permitted Users" appropriately, or assign the user to a profile/permission set that's allowed.

### "Authentication blocked due to IP restrictions"

Connected app or profile has IP restrictions; CI's IP is not allowed.

Fix: add CI provider's IP ranges to the allowlist, or remove the restriction.

## Tests fail in CI but pass locally

### Test depends on org state

Local scratch org has data the CI scratch doesn't, or vice versa.

Fix: hermetic tests. Each test creates its own data.

### Test depends on user permissions

Different scratch orgs assign permissions differently.

Fix: explicitly assign needed permission sets in test setup.

### Time-dependent test

Test passes during business hours, fails at 2am UTC.

Fix: use `Date.newInstance` and `DateTime.newInstance` instead of `today()` or `now()` where possible. Or use `Test.setCreatedDate`.

### Async timing

Test asserts state after an async operation; the operation didn't finish in CI's faster scratch.

Fix: use `Test.startTest`/`Test.stopTest` correctly. Async work runs synchronously between them.

## Package version build fails

### "Coverage less than 75%"

Building with `--code-coverage` requires 75% coverage.

Fix: write more tests for the under-covered Apex.

### "Source contains invalid metadata"

The package directory has files Salesforce doesn't recognize.

Fix:

- Check `.forceignore` excludes non-source files.
- Validate against a scratch org first.

### "Dependency not found"

The package version's dependencies reference packages that aren't accessible to the Dev Hub.

Fix:

- Verify the dependency package versions are released or accessible.
- Check the namespace prefix matches.
- Confirm `packageAliases` in `sfdx-project.json` are correct.

## Install fails

### "Package version requires installation key"

The package was built with `--installation-key` but the install doesn't provide one.

Fix:

```bash
sf package install --package "MyApp@1.0.0-1" --installation-key "secret"
```

### "Package version is beta"

Beta versions can install only in non-production orgs.

Fix: promote to released, or install in a sandbox/scratch.

### "Dependent package version not installed"

Install fails because a dependency package isn't there.

Fix: install the dependency first, then install your package.

## Sandbox issues

### "Refresh failed"

Salesforce backend issue. Often resolves on retry.

Fix: wait, retry. If it persists, open a Salesforce support case.

### "Sandbox has stale config after refresh"

Post-refresh setup wasn't applied.

Fix: run the post-refresh checklist (see [Chapter 23](./23-sandbox-strategy.md)).

### "Sandbox running production scheduled jobs"

After refresh, scheduled jobs from production are running in sandbox.

Fix: abort jobs:

```apex
List<CronTrigger> jobs = [SELECT Id FROM CronTrigger];
for (CronTrigger ct : jobs) System.abortJob(ct.Id);
```

## Source tracking issues

### "Push says no changes, but I edited a file"

Source tracking doesn't see the change. Common causes:

- File ignored by `.forceignore`.
- File saved but not flushed.
- Source format file in an unexpected location.

Fix:

```bash
sf project deploy preview --target-org dev
```

Reveals what the CLI sees as different.

### "Pull keeps pulling things I already pulled"

The org modified the metadata after your pull.

Fix: pull again. If recurring, the org has automated processes generating noise; ignore them.

## CLI issues

### "Command not found: sf"

CLI not in PATH or not installed.

Fix:

```bash
npm install -g @salesforce/cli
```

### "sf: command not found in CI"

CI environment doesn't have the CLI.

Fix: install it as part of CI setup:

```yaml
- run: npm install -g @salesforce/cli
```

### "Salesforce CLI version is too old"

Some commands require recent CLI versions.

Fix: update:

```bash
sf update
```

## Performance issues

### "Apex test suite takes 90 minutes"

Test suite has grown unmanageable.

Fix:

- Audit slow tests. Often a few tests dominate.
- Mock callouts.
- Reduce fixture sizes.
- Move integration tests out of unit test runs.

### "Deploy takes 45 minutes"

Big deploy with full test runs.

Fix:

- Use validate-then-quick-deploy.
- Use `RunSpecifiedTests` with delta tooling.
- Move to package-based DevOps.

### "Scratch org creation takes 12 minutes"

Heavy feature set in the definition.

Fix:

- Use snapshots.
- Reduce features in the CI scratch definition.

## When to open a Salesforce support case

For:

- Platform-side errors that aren't documented.
- Persistent deploy failures with internal Salesforce errors.
- Sandbox refresh failures.
- Dev Hub or scratch org provisioning issues.
- Unexpected behavior of governor limits or platform features.

Have ready:

- Org id of the affected org.
- Job id of the failing deploy.
- Exact error message.
- Steps to reproduce.
- What you've already tried.

Salesforce support is generally responsive but slow. Don't wait for them when you can resolve yourself.

## A useful diagnostic toolkit

Commands worth running when something is wrong:

```bash
# CLI version
sf --version

# All known orgs
sf org list --all

# Detail on a specific org
sf org display --target-org <alias> --verbose

# Recent deploy
sf project deploy report --use-most-recent --target-org <alias>

# Source tracking state
sf project deploy preview --target-org <alias>

# Limits
sf org list limits --target-org <alias>

# Apex log
sf apex tail log --target-org <alias>
```

Save these in a `troubleshooting.md` in your team's repo. Easier than remembering them under pressure.

## References

- [Salesforce CLI troubleshooting](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_troubleshoot.htm)
- [Deploy errors](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_deploy.htm)
- [Apex governor limits](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_gov_limits.htm)
- [Salesforce Support](https://help.salesforce.com/s/)
