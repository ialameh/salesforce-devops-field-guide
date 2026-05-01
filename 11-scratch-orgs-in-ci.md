# 11. Scratch Orgs in CI

Scratch orgs are at their best in CI. Disposable, isolated, predictable. A pipeline that uses scratch orgs well runs fast, fails fast, and never confuses one PR's tests with another's.

This chapter is the operational pattern. The CI provider syntax is in [Chapter 21](./21-cicd-architecture.md); here we focus on the orchestration.

## The basic CI scratch lifecycle

```bash
# 1. Auth to Dev Hub via stored token
sf org login sfdx-url --sfdx-url-file /tmp/devhub-auth.txt --alias devhub --set-default-dev-hub

# 2. Create a fresh scratch org
sf org create scratch \
    --definition-file config/project-scratch-def.json \
    --alias ci-org \
    --duration-days 1 \
    --set-default

# 3. Deploy the source under test
sf project deploy start --target-org ci-org

# 4. Run Apex tests
sf apex run test --target-org ci-org --code-coverage --result-format junit -d test-results

# 5. Run any other tests (LWC jest, integration scripts)
npm test

# 6. Always delete the scratch org
sf org delete scratch --target-org ci-org --no-prompt
```

The cleanup must run regardless of whether tests pass. In CI tooling, that's usually an `always()` or `if: always()` condition on the cleanup step.

## Why a fresh org per run

- **Isolation.** Two PRs can run simultaneously without interfering.
- **Reproducibility.** Same state every time. No accumulated cruft.
- **Cleanliness.** Tests can do destructive things; nothing leaks to the next run.
- **Honesty.** A test that depends on prior runs' state is a broken test.

## Auth strategies

### SFDX auth URL (simple)

A long-form URL that contains a refresh token. Store as a CI secret. CLI reads it.

```bash
echo "$SFDX_AUTH_URL_DEVHUB" > /tmp/devhub-auth.txt
sf org login sfdx-url --sfdx-url-file /tmp/devhub-auth.txt --alias devhub
```

To generate the URL:

```bash
sf org display --target-org your-devhub --verbose --json | jq -r '.result.sfdxAuthUrl'
```

Pros: simple. Pros: works in any CI tool.
Cons: refresh tokens can be revoked; secrets management still required.

### JWT bearer flow (recommended for production)

Use a connected app with a certificate. CI signs a JWT to get an access token. No refresh token to leak.

```bash
sf org login jwt \
    --client-id "$SF_CLIENT_ID" \
    --jwt-key-file /tmp/server.key \
    --username "$SF_USERNAME" \
    --alias devhub \
    --set-default-dev-hub
```

Pros: no long-lived refresh tokens. More secure.
Cons: setup is more involved (connected app, cert, etc.).

For production-targeted CI, prefer JWT.

## Speeding up CI scratch orgs

Scratch creation is the slowest step. Three optimisations:

### 1. Use snapshots

See [Chapter 10](./10-scratch-org-snapshots.md). A weekly-refreshed snapshot drops scratch creation from 5-10 minutes to 30-60 seconds.

### 2. Minimise features in the CI scratch definition

The CI scratch definition does not need every feature your developer scratches have. Strip it down to what tests actually need.

```json
// config/project-scratch-def-ci.json
{
  "orgName": "CI",
  "edition": "Developer",
  "features": [
    "EnableSetPasswordInApi"
  ],
  "settings": {
    "lightningExperienceSettings": { "enableS1DesktopEnabled": true }
  }
}
```

Reference this in CI instead of the full developer definition.

### 3. Parallelise where possible

If your CI provider supports matrix builds, run tests in parallel scratch orgs. Each gets a fraction of the test suite.

```yaml
strategy:
  matrix:
    test-shard: [1, 2, 3, 4]
steps:
  - run: |
      sf org create scratch --alias ci-${{ matrix.test-shard }}
      sf apex run test --target-org ci-${{ matrix.test-shard }} \
          --tests $(scripts/get-shard.sh ${{ matrix.test-shard }} 4)
```

Trades scratch quota for wall-clock time.

## Test data in CI

Scratch orgs are blank. Tests that depend on data must create it.

Three options:

### Apex test data factories

`@TestSetup` methods or factory classes that create records inline. Lives with the tests.

```apex
@IsTest
public class OrderProcessorTest {
    @TestSetup
    static void setup() {
        Account a = TestDataFactory.createAccount('Acme');
        insert a;
    }

    @IsTest
    static void processesOrders() {
        // ...
    }
}
```

Pros: lives with the test, version-controlled, deterministic. Cons: slow to write for complex object graphs.

### Apex script seeded data

A non-test script that runs after scratch creation:

```bash
sf apex run --file scripts/seed-data.apex --target-org ci-org
```

Pros: shared across many tests. Cons: tests then assume that data exists, coupling them to the script.

### Loaded test data

JSON or CSV files loaded via `sf data import`:

```bash
sf data import tree --plan data/test-plan.json --target-org ci-org
```

Pros: representative of real data shapes. Cons: maintaining the data files.

For most CI: Apex test setup is the lowest-friction option.

## Test results and coverage

Run tests with structured output:

```bash
sf apex run test \
    --target-org ci-org \
    --code-coverage \
    --result-format junit \
    --output-dir test-results \
    --wait 10
```

`--result-format junit` produces JUnit XML, which most CI tools render natively. Coverage is in `test-results/test-result-codecoverage.json`.

Block the build on coverage drops below a threshold:

```bash
COVERAGE=$(jq -r '.summary.testRunCoverage' test-results/test-result.json)
if (( $(echo "$COVERAGE < 75" | bc -l) )); then
    echo "Coverage $COVERAGE% below threshold"
    exit 1
fi
```

## Cleanup discipline

Scratch orgs that survive past their tests cost money and cap quota. Always clean up:

```yaml
- name: Delete scratch org
  if: always()
  run: |
    sf org delete scratch --target-org ci-org --no-prompt || true
```

The `|| true` swallows errors. If the scratch org was already deleted (e.g., tests crashed before getting that far), don't fail the cleanup step.

## Quota management

Each Dev Hub has limits on:

- Concurrent active scratch orgs.
- Daily scratch org creations.

If your CI burns through these:

- Buy more capacity (Salesforce sells expansion packs).
- Add concurrency caps in your CI provider.
- Cache more aggressively (snapshots).
- Run fewer scratch-org-dependent tests on every PR (e.g., only on merge to main).

The numbers matter when you have a busy team. Track usage:

```bash
sf org list --json | jq '.result.scratchOrgs | length'
```

Alert when nearing limits.

## A reference CI scratch script

Bring it all together:

```bash
#!/bin/bash
set -euo pipefail

ALIAS="ci-${CI_RUN_ID:-$RANDOM}"

cleanup() {
    sf org delete scratch --target-org "$ALIAS" --no-prompt || true
}
trap cleanup EXIT

# Auth
echo "$SFDX_AUTH_URL_DEVHUB" > /tmp/devhub.txt
sf org login sfdx-url --sfdx-url-file /tmp/devhub.txt --alias devhub --set-default-dev-hub

# Create
sf org create scratch \
    --definition-file config/project-scratch-def-ci.json \
    --alias "$ALIAS" \
    --duration-days 1 \
    --set-default

# Deploy
sf project deploy start --target-org "$ALIAS"

# Test
sf apex run test \
    --target-org "$ALIAS" \
    --code-coverage \
    --result-format junit \
    --output-dir test-results \
    --wait 30

# (cleanup happens via trap)
```

Run this in CI. Wraps the lifecycle, handles cleanup on success and failure, leaves clean test results.

## Local CI simulation

Most CI providers let you run the same script locally. Setting `CI_RUN_ID` and the auth URL secret allows developers to debug CI locally:

```bash
export SFDX_AUTH_URL_DEVHUB="$(sf org display --target-org devhub --verbose --json | jq -r '.result.sfdxAuthUrl')"
export CI_RUN_ID="local-$(date +%s)"
./ci/run-tests.sh
```

Trains developers in the same patterns CI uses. Catches CI-only bugs locally.

## Common CI failure modes

**Scratch creation fails: "DAILY_LIMIT_EXCEEDED".**
You hit the Dev Hub daily creation limit. Wait until tomorrow, increase capacity, or reduce CI frequency.

**Scratch creation fails with feature errors.**
The Dev Hub does not have the licenses required to enable a feature in your scratch definition. Either remove the feature or get the license.

**Tests pass locally, fail in CI.**
Most often: test data setup differs. Debug by re-running locally with the same scratch definition.

**Auth fails.**
Refresh tokens expire. Rotate the SFDX auth URL or switch to JWT.

**Cleanup didn't run, scratch orgs accumulating.**
Add a Dev Hub-level cleanup job that deletes orgs older than N hours:

```bash
sf org list --json | jq -r '.result.scratchOrgs[] | select(.createdDate < "1 day ago") | .alias' | \
    xargs -I {} sf org delete scratch --target-org {} --no-prompt
```

## When CI doesn't need a scratch org

Not every CI step needs an org:

- Static analysis. PMD, ESLint, jest tests, schema validation.
- Manifest generation. `sfdx-git-delta`-based diff manifests.
- Security scanning.

Run those in fast jobs. Save the scratch-org-dependent jobs for what actually requires an org.

## References

- [`sf org create scratch`](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_org_commands_unified.htm)
- [SFDX auth URL format](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth.htm)
- [JWT bearer flow](https://help.salesforce.com/s/articleView?id=sf.remoteaccess_oauth_jwt_flow.htm)
- [Apex testing in CI](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_qs_test.htm)
- [Salesforce CLI for CI](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_continuous_integration.htm)
