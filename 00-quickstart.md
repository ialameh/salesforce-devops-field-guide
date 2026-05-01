# 00. Quickstart

A minimal Salesforce DX project, a working scratch org, a clean deploy, and a CI-ready repo. Twenty minutes if you have the CLI installed. Forty if you don't.

## What you need

- The Salesforce CLI: `sf --version` should print 2.0 or later.
- A Dev Hub. Either an existing one or a free Developer Edition org you enable Dev Hub in: *Setup, Dev Hub, Enable*.
- A code editor.
- Git.

## Step 1. Create the project

```bash
sf project generate --name my-salesforce-project --output-dir .
cd my-salesforce-project
```

That gives you the standard layout:

```
my-salesforce-project/
    config/
        project-scratch-def.json
    force-app/main/default/
    manifest/
    sfdx-project.json
    package.json
    .forceignore
    README.md
```

## Step 2. Authorise the Dev Hub

```bash
sf org login web --set-default-dev-hub --alias devhub
```

A browser opens. Log in. The CLI saves the auth.

## Step 3. Create a scratch org

```bash
sf org create scratch \
    --definition-file config/project-scratch-def.json \
    --alias dev \
    --duration-days 7 \
    --set-default
```

Wait a couple of minutes. The CLI prints a username and the org is ready.

## Step 4. Push initial source

The default project is empty, so add a tiny Apex class first:

```bash
mkdir -p force-app/main/default/classes
cat > force-app/main/default/classes/Hello.cls <<'APEX'
public with sharing class Hello {
    public static String world() {
        return 'Hello, world';
    }
}
APEX
cat > force-app/main/default/classes/Hello.cls-meta.xml <<'META'
<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <status>Active</status>
</ApexClass>
META
```

Push it:

```bash
sf project deploy start --target-org dev
```

You should see one component deployed.

## Step 5. Run a test

Add a test class:

```bash
cat > force-app/main/default/classes/HelloTest.cls <<'APEX'
@IsTest
public with sharing class HelloTest {
    @IsTest
    static void worldReturnsExpected() {
        System.assertEquals('Hello, world', Hello.world());
    }
}
APEX
cat > force-app/main/default/classes/HelloTest.cls-meta.xml <<'META'
<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <status>Active</status>
</ApexClass>
META
sf project deploy start --target-org dev
sf apex run test --target-org dev --code-coverage --result-format human
```

Test should pass. Coverage should be 100% for `Hello`.

## Step 6. Pull any UI changes back

If you changed anything in the Setup UI of the scratch org (you didn't here, but you will eventually):

```bash
sf project retrieve start --target-org dev
```

This is the habit. After every UI change, retrieve. See [Chapter 7](./07-source-tracking.md) for why.

## Step 7. Initialise Git

```bash
git init
cat > .gitignore <<'GIT'
.sf/
.sfdx/
.localdevserver/
.vscode/
.idea/
*.log
node_modules/
GIT
git add .
git commit -m "Initial commit: empty Salesforce DX project"
```

## Step 8. Validate against a target org

Once you have a sandbox or production org authenticated, you can dry-run a deploy:

```bash
sf org login web --alias staging
sf project deploy validate --target-org staging --test-level RunLocalTests
```

This runs the deploy in validation-only mode. No changes to the target org. Tests run. Result is cached for 4 days. If you decide to deploy:

```bash
sf project deploy quick --target-org staging --use-most-recent
```

That promotes the validated job. Seconds, not minutes. See [Chapter 13](./13-deploy-semantics.md).

## Step 9. Delete the scratch org when you're done

```bash
sf org delete scratch --target-org dev --no-prompt
```

Scratch orgs are disposable. Don't treat them like long-lived sandboxes. Recreate fresh whenever the definition changes.

## What you have at this point

- A Salesforce DX project in source control.
- A working scratch org you can recreate from definition at any time.
- A green deploy and a green test run.
- The validate-and-quick-deploy pattern available for production deploys.
- A clean retrieve workflow for syncing UI changes back to source.

That's the foundation. Everything else in this guide is variations on this pattern, plus the operational discipline that keeps it working as the codebase and the team grow.

## Next steps

- Read [Chapter 1: Mental Model](./01-mental-model.md) to understand the four stores of truth.
- Pick a packaging strategy: see [Chapter 8: Packaging Overview](./08-packaging-overview.md).
- Set up CI: see [Chapter 16: CI/CD Architecture](./16-cicd-architecture.md).
- Read [Chapter 7: Source Tracking](./07-source-tracking.md) before you go too far. It catches everyone out at least once.

## References

- [Salesforce CLI command reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_top.htm)
- [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm)
- [Apex testing](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_qs_test.htm)
- [Scratch org overview](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs.htm)
