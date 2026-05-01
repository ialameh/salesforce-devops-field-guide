# 19. Test Levels

What `--test-level` actually does, when each makes sense, and how to handle the common production deploy slowdowns.

## The four levels

| Level | What runs | Typical use |
|-------|-----------|-------------|
| `NoTestRun` | Nothing | Sandboxes, quick iterations |
| `RunSpecifiedTests` | Only named test classes | Incremental production deploys |
| `RunLocalTests` | All tests in your namespace | Standard production deploys |
| `RunAllTestsInOrg` | Every test, including managed packages | Rarely; new managed package install |

## NoTestRun

Default in many sandbox deploys. Runs no tests.

Production rejects this. So do some sandbox configurations.

Use only in scratch orgs for fast iteration.

## RunSpecifiedTests

Run only the test classes you name:

```bash
sf project deploy start \
    --manifest manifest/package.xml \
    --target-org production \
    --test-level RunSpecifiedTests \
    --tests OrderProcessorTest CustomerLookupServiceTest
```

For this to satisfy production deploy requirements, the named tests must cover at least 75% of the Apex being deployed.

When this fits:

- Incremental deploys where you know which tests cover the change.
- Hot-fix deploys where running all tests is overkill.
- CI pipelines with parallel test execution.

When it doesn't:

- Deploys involving many components.
- When you don't have confidence the named tests cover the change.

## RunLocalTests

Run every test class in your org's namespace. Skips tests in installed managed packages.

```bash
sf project deploy start \
    --manifest manifest/package.xml \
    --target-org production \
    --test-level RunLocalTests
```

The default for production. Most teams stick with this.

When this fits:

- Standard production deploys.
- Releases where you want full coverage validation.

When it's painful:

- Slow test suites. A 90-minute test run gates every production deploy.

## RunAllTestsInOrg

Runs every test, including managed packages. Slowest option.

When this fits:

- New managed package installations where you want to verify package compatibility.
- Comprehensive validation before a major release.

When it doesn't:

- Routine deploys. Way too slow.

## Coverage requirements

Production deploys (and managed package version creation) require 75% Apex coverage. The rules:

- Each Apex class needs ≥75% coverage.
- Triggers count towards coverage.
- Managed package code is not counted.

If your deploy includes a class with <75% coverage, the deploy fails.

## Speeding up production tests

The most common operational pain.

### 1. Run tests asynchronously

`sf apex run test --synchronous` uses one transaction. Slower for many tests. Async runs in parallel and finishes faster.

Salesforce manages parallelism automatically. You don't usually configure it. But know that async is the default and is faster than sync.

### 2. Reduce fixture size

Tests that create thousands of records take seconds each, multiplied by hundreds of tests. Mock or reduce fixture sizes:

```apex
// Before: creates 200 records every test
@TestSetup
static void setup() {
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 200; i++) {
        accounts.add(new Account(Name = 'Test ' + i));
    }
    insert accounts;
}

// After: create only what you need
@TestSetup
static void setup() {
    insert new Account(Name = 'Test');
}
```

### 3. Mock callouts

Don't let tests hit external systems. `Test.setMock` makes them deterministic and fast:

```apex
Test.setMock(HttpCalloutMock.class, new MyMock());
```

### 4. Stub heavy dependencies

For tests that don't actually exercise a complex dependency, stub it:

```apex
ServiceLocator.set(MyService.class, new MyServiceStub());
```

### 5. Eliminate sleeps and waits

Tests with `Test.startTest()`/`Test.stopTest()` boundaries that wrap async work can run quickly. Tests with explicit sleeps or timeouts cannot.

### 6. Parallelise

For specified-tests deploys, you can split tests across multiple deploy jobs running in parallel CI. Each runs a subset, fails fast.

This is more CI complexity than savings warrant in most cases.

## When tests fail in production deploy

The deploy rolls back. The deploy failed.

Diagnostic steps:

1. **Read the test output.** The CLI shows failed tests and assertions.
2. **Reproduce locally.** Pull the relevant source and run the test in a scratch org.
3. **Determine the cause.** Is the test wrong? Is the new code wrong? Is the test depending on data that exists in production but not in scratch?
4. **Fix the test or the code.** Whichever is wrong.
5. **Re-validate and re-deploy.**

Don't disable the test. Don't change `--test-level`. Don't ship.

## RunSpecifiedTests trap

A common mistake: pass `--test-level RunSpecifiedTests` with `--tests` that don't include sufficient coverage. The deploy fails:

```
Test coverage of selected tests does not cover any subscriber package class with at least 75%
```

Fix: either include all tests that cover the changed Apex, or use `RunLocalTests`.

## Tests that pass in scratch but fail in production

Different state. Common causes:

- Test relies on data that exists in production but not in scratch.
- Test relies on data that exists in scratch but not in production.
- Test is timezone-sensitive and runs at a problematic hour.
- Test doesn't enforce CRUD/FLS but production does.

Fix: write hermetic tests. Each test sets up its own data. Don't depend on what's in the org.

## Test method best practices

A few that pay off:

### Use @TestSetup

```apex
@TestSetup
static void setupData() {
    Account a = new Account(Name = 'Test');
    insert a;
}
```

Runs once per test class. Faster than per-method setup.

### Use Test.startTest() and Test.stopTest()

Tests have their own governor limits between `startTest` and `stopTest`. Useful when verifying bulk behaviour.

### Assert behaviour, not implementation

```apex
// Bad: brittle, breaks on refactor
System.assertEquals('SuccessResult', controller.getResultJson());

// Good: tests the behaviour
System.assertEquals(true, controller.isSuccess());
```

### Assert with messages

```apex
System.assertEquals(expected, actual, 'Status should be Closed after archive');
```

Failed asserts with messages save debugging time.

### Test edge cases explicitly

Empty lists. Null values. Bulk (200 records). Permissions denied. Each one its own method.

## When tests are not enough

Apex tests verify Apex. They don't verify:

- Layout assignments.
- Permission set effective state.
- Flow execution paths (Flow tests are limited).
- LWC behaviour (use jest separately).
- Integration with external systems (use UAT).

Don't conflate "deploy succeeded with tests passing" with "the feature works". The deploy is a step. Verification is multi-layered.

## A working test discipline

Adopt as a team:

- Every Apex class has a corresponding `<ClassName>Test`.
- Tests run on every PR.
- Coverage threshold enforced (75% as a hard floor, 90% as a target).
- New code requires new tests.
- Bug fixes require regression tests.
- LWC components have jest tests.

These don't directly affect deploy speed but they make production deploys reliable.

## References

- [Apex testing](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_qs_test.htm)
- [Apex test levels](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_testing_test_specified_class.htm)
- [Code coverage](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_code_coverage_intro.htm)
- [Test data factories](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_testing_utility_classes.htm)
