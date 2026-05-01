# 28. Glossary

Terms that appear throughout this guide, defined.

## Salesforce concepts

**Apex.** Salesforce's Java-like language for server-side logic. Runs on the platform.

**Apex test class.** An `@IsTest`-marked class containing test methods. Runs in isolated transactions; data is rolled back after.

**API version.** The Salesforce platform release a metadata file targets. Bumps each release. Affects which features the metadata can use.

**AppExchange.** Salesforce's marketplace for managed packages and listings.

**Bulk Apex.** Apex written to handle 200 records per call (the platform's default batch size). Most performance issues come from non-bulk Apex.

**Connected app.** Salesforce's representation of an external app that can authenticate via OAuth. Used for CLI auth, JWT bearer flow, integrations.

**Custom metadata type.** Configuration stored as records, deployable, queryable. Used for feature flags, environment-specific settings.

**Custom object.** A user-defined sObject. Has fields, layouts, validation rules, sharing rules, etc.

**Dev Hub.** A Salesforce org that creates and manages scratch orgs and 2GP packages. Usually a production org with the Dev Hub feature enabled.

**Field-Level Security (FLS).** Per-field, per-profile access control. Determines whether a user can read or edit a field.

**Flow.** Declarative automation. Replaces process builder and many trigger use cases.

**Governor limits.** Salesforce's transaction-level limits (SOQL queries, DML, CPU time, heap, callouts).

**LWC.** Lightning Web Component. Modern Salesforce frontend component framework.

**Managed package.** A package distributed through AppExchange. IP-protected, namespaced, ancestry-controlled.

**Metadata API.** Salesforce's API for deploying and retrieving metadata (objects, classes, layouts, etc.).

**Namespace.** A 3-15 character prefix that scopes metadata. Required for managed packages. Optional for unlocked. Once registered, permanent.

**Org.** A Salesforce instance. Production, sandbox, scratch, all are "orgs" in Salesforce vocabulary.

**Permission set.** A bundle of permissions assigned to users. Lighter weight than a profile.

**Permission set group.** A bundle of permission sets assigned together. Useful for role-based access.

**Profile.** A bundle of permissions tied to a user license. Heavy, merge-conflict-prone.

**Sandbox.** A copy of production used for development or testing. Several types (Developer, Developer Pro, Partial Copy, Full).

**Salesforce CLI (`sf`).** Command-line tool for Salesforce DevOps. Replaces the older `sfdx` CLI for new projects.

**Scratch org.** A disposable, source-tracked org created from a definition file. Up to 30 days lifetime.

**SOQL.** Salesforce's query language. Like SQL with restrictions.

**sObject.** A Salesforce object. Standard (Account, Contact) or custom.

**Source format.** The CLI-friendly format for metadata, broken into per-component files. The format you author in.

**Standard Invocable Action.** A Salesforce-shipped action callable from Flow, Agentforce, or Apex.

**Test level.** A flag on a deploy that controls which Apex tests run (`NoTestRun`, `RunSpecifiedTests`, `RunLocalTests`, `RunAllTestsInOrg`).

**Trigger.** Apex that runs on database events (insert, update, delete, etc.).

## DevOps concepts

**Ancestry (in Managed packages).** The chain of versions where each builds on the previous in a backward-compatible way.

**Branch protection.** Repository rules that prevent force-push, require reviews, require passing CI, etc.

**CI/CD.** Continuous Integration / Continuous Delivery. Automated build, test, deploy.

**CLI auth URL (SFDX auth URL).** A long-form URL containing a refresh token, used for non-interactive CLI auth.

**Code coverage.** Percentage of Apex code exercised by tests. Required to be ≥75% for production deploys.

**Deploy.** Sending metadata to a target org via the metadata API.

**Deployment job.** A specific run of a deploy. Has a job id (`0Af...`).

**Destructive change.** A deletion of metadata via deploy.

**Force ignore (`.forceignore`).** A file telling the CLI which files to ignore.

**JWT bearer flow.** OAuth flow using a signed JWT instead of a refresh token. Recommended for production CI.

**Manifest (`package.xml`).** A list of metadata components. Used to drive deploys and retrieves.

**Metadata format.** The XML-zip format the metadata API consumes. Contrast with source format.

**Org-based deploy.** Deploying source directly to an org without packaging.

**Org-Dependent Unlocked Package.** A 2GP unlocked variant that allows references to org-specific metadata at install time.

**Package.** A 2GP unit (`Package2`). Has a developer name, type, package id (`0Ho...`).

**Package alias.** A name in `sfdx-project.json` that maps to a package id.

**Package directory.** A folder under `packageDirectories` in `sfdx-project.json`.

**Package version.** An immutable build of a package (`Package2Version`, id `04t...`).

**Permission set group (PSG).** A grouping of permission sets that can be assigned together.

**Pull / push.** Source-tracked operations against scratch orgs and source-tracked sandboxes. `pull` retrieves changes; `push` deploys local changes.

**Pull request (PR).** A proposed change to a Git branch, reviewed before merging.

**Quick deploy.** Promoting a previously validated deploy job to a real deploy. Skips re-running tests.

**Refresh token.** A long-lived OAuth token used to acquire short-lived access tokens. Stored in CI secrets for SFDX auth URL flow.

**Retrieve.** Pulling metadata from an org to local source.

**Runbook.** A documented procedure for an operation. Used for releases, rollbacks, sandbox refreshes.

**Sandbox template.** A definition of which records to copy into a Partial Copy sandbox.

**Scratch org definition.** The JSON file describing what kind of scratch org to create.

**Scratch org snapshot.** A frozen state of a scratch org used as a base for future scratch orgs.

**Source tracking.** A mechanism that records metadata changes in an org so the CLI knows what changed since the last sync.

**Trunk-based development.** A branching strategy with one long-lived branch (`main`) and short-lived feature branches.

**Validate (deploy validate).** A dry-run deploy that catches errors without modifying the target. Result cached for 4 days.

## Tools and acronyms

**1GP.** First-Generation Package (legacy managed packages).

**2GP.** Second-Generation Package (modern unlocked or managed packages).

**Apex `@isTest`.** Annotation marking a class or method as a test.

**`@InvocableMethod`.** Annotation exposing an Apex method to Flow, Agentforce, etc.

**`@InvocableVariable`.** Annotation marking a class field as an input or output to an invocable method.

**FLS.** Field-Level Security.

**LMA.** License Management App. Used for managed package licensing.

**LWC.** Lightning Web Component.

**OWD.** Organization-Wide Default. Per-object sharing default.

**PMD.** A static code analyzer with Apex support.

**SF CLI.** Salesforce CLI (`sf` command).

**SFDX.** Salesforce DX. The previous CLI (`sfdx` command). Now mostly subsumed by `sf`.

**SOQL.** Salesforce Object Query Language.

**SOSL.** Salesforce Object Search Language (full-text search).

**Spring '26, Summer '26, Winter '27, etc.** Salesforce's three-times-yearly major releases.

**UAT.** User Acceptance Testing.

## Common file paths

**`force-app/main/default/`.** The default package directory. Where most source lives.

**`config/project-scratch-def.json`.** The default scratch org definition.

**`manifest/package.xml`.** The default manifest for deploys and retrieves.

**`.forceignore`.** Files the CLI should ignore.

**`sfdx-project.json`.** Project configuration. Most important file in the repo.

**`.gitignore`.** Files Git should ignore. Includes `.sf/`, `.sfdx/`.

## Common CLI commands

**`sf project deploy start`.** Deploy source to an org.

**`sf project deploy validate`.** Validate a deploy without applying.

**`sf project deploy quick`.** Promote a validated deploy.

**`sf project deploy report`.** Get details on a recent deploy.

**`sf project retrieve start`.** Pull metadata from an org.

**`sf project deploy preview`.** Show what a deploy would do without doing it.

**`sf org login web`.** Authenticate via browser.

**`sf org login sfdx-url`.** Authenticate using a stored SFDX auth URL.

**`sf org login jwt`.** Authenticate using JWT bearer flow.

**`sf org create scratch`.** Create a scratch org.

**`sf org delete scratch`.** Delete a scratch org.

**`sf package create`.** Create a 2GP package.

**`sf package version create`.** Build a package version.

**`sf package version promote`.** Promote a beta version to released.

**`sf package install`.** Install a package version in a target.

**`sf apex run test`.** Run Apex tests.

**`sf data query`.** Run a SOQL query.

For the comprehensive list, see [the CLI reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_top.htm).

## References

- [Salesforce Glossary](https://help.salesforce.com/s/articleView?id=sf.glossary.htm)
- [Apex Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/)
- [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm)
- [Salesforce CLI Command Reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_top.htm)
- [Metadata API Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_intro.htm)
- [2GP Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp.htm)
