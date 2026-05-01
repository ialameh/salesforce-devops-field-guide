# 05. .forceignore and Manifests

The two files that decide what the CLI sees and what it ignores. Get them right early; they shape every deploy and retrieve afterwards.

## .forceignore

A `.gitignore`-style file that tells the Salesforce CLI which files to skip. Applies to:

- `sf project deploy start`
- `sf project deploy validate`
- `sf project retrieve start`
- `sf project deploy preview`
- `sf project deploy report`
- `push` and `pull` for source-tracked orgs

It does not apply to direct metadata API calls. Tools that bypass the CLI ignore it.

### Syntax

Same as `.gitignore`. One pattern per line. Comments start with `#`.

```
# Ignore by directory
**/profiles/**

# Ignore by file extension
**/*.bak

# Ignore a specific file
force-app/main/default/permissionsets/Old_Set.permissionset-meta.xml

# Re-include something that was excluded by an earlier pattern
!force-app/main/default/permissionsets/Important_Set.permissionset-meta.xml
```

### Common .forceignore patterns

A baseline that works for most projects:

```
# IDE and OS noise
**/.vscode/**
**/.idea/**
**/*.swp
**/.DS_Store

# Local builds and tests
**/__tests__/**
**/jsconfig.json
**/coverage/**

# Things that shouldn't be in source control
**/*.log
**/*.bak
```

### The profile question

Profiles are the most contentious entry. Three positions:

**Position A: Ignore profiles entirely.**

```
**/profiles/**
```

Pros: profiles are huge and merge-conflict-prone. They include references to every metadata component, so trivial changes elsewhere generate diffs in profile files. Permission sets are a better mechanism anyway.

Cons: you need to manage permissions separately in each org via permission sets. One-time setup is tedious.

This is the position most experienced Salesforce DevOps engineers take.

**Position B: Track profiles, but only the parts you control.**

Track profiles in source, but use other mechanisms (permission sets) for actual permission grants. Profiles in source then function as a baseline.

Pros: keeps profiles versioned without making them load-bearing.

Cons: still produces noisy diffs.

**Position C: Track profiles as the source of truth.**

Pros: profiles are how Salesforce ships permissions. Aligns with platform expectations.

Cons: pain. Constant merge conflicts. Slow deploys.

### Recommended

Position A. Ignore profiles. Manage permissions through permission sets and permission set groups. Profiles in target orgs are managed as standing config (set up once, rarely touched).

The exception: if you have one profile that legitimately needs to track in source (e.g., a custom standard profile your org depends on), `!`-include it specifically.

### Other things worth ignoring

```
# Setup-related metadata that you do not want to deploy
**/sites/**
**/communities/**

# Old change-set artifacts
**/unpackaged/**

# OmniStudio cached metadata
**/omnistudio/**

# Generated translations
**/translations/**
```

Adjust based on what your project does and does not own.

### When .forceignore matters most

- During `pull` from a source-tracked org. Without ignores, you pull every junk file the org generated.
- During `push` to a scratch org. Without ignores, you push noise.
- During retrieves with `*` wildcards.
- During deploys based on a directory rather than a manifest.

## Manifests (package.xml)

A list of metadata components. Used to:

- Tell the CLI exactly what to retrieve.
- Tell the CLI exactly what to deploy.
- Document a release.

### Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>OrderStatusController</members>
        <members>CustomerLookupService</members>
        <name>ApexClass</name>
    </types>
    <types>
        <members>Account.Status__c</members>
        <name>CustomField</name>
    </types>
    <version>62.0</version>
</Package>
```

Each `<types>` block has many `<members>` and one `<name>`. Member names follow per-type conventions (e.g., `<Object>.<Field>` for custom fields).

### Wildcards

`*` is shorthand for "all of this type":

```xml
<types>
    <members>*</members>
    <name>ApexClass</name>
</types>
```

For a few types, `*` does not include managed package or namespaced members. Read the manifest reference if you suspect this.

### Common manifest patterns

**Whole-org retrieve** (used for migrations):

```xml
<types><members>*</members><name>ApexClass</name></types>
<types><members>*</members><name>ApexTrigger</name></types>
<types><members>*</members><name>CustomObject</name></types>
<!-- ... -->
```

**Release manifest** (explicit list of what's in a release):

```xml
<types>
    <members>OrderStatusController</members>
    <members>OrderStatusControllerTest</members>
    <name>ApexClass</name>
</types>
<types>
    <members>Account.Status__c</members>
    <name>CustomField</name>
</types>
<types>
    <members>OrderStatusFlow</members>
    <name>Flow</name>
</types>
```

This pattern is reviewable. A reviewer can read the manifest and know exactly what's shipping.

### Generating manifests

Several ways:

**Manually.** Write the XML by hand. Practical for small manifests; tedious for large.

**From source.** The CLI can scan a directory and generate a manifest:

```bash
sf project generate manifest --source-dir force-app --output-dir manifest
```

Produces `manifest/package.xml` containing every component in `force-app`.

**From an org.** List what's in an org and write it as a manifest:

```bash
sf project generate manifest --from-org dev --output-dir manifest
```

Useful for retrieving everything from a sandbox.

**From a diff.** With Git or other tooling, identify what changed since a tag and produce a manifest of just those components. Useful for incremental deploys. Several community tools (e.g., `sfdx-git-delta`) do this.

### Manifest folders

A common project structure:

```
manifest/
    package.xml              <-- the canonical manifest
    package-test.xml         <-- a test-only manifest for CI
    destructiveChanges.xml   <-- pre-deploy deletions
    destructiveChangesPost.xml  <-- post-deploy deletions
    package-feature-X.xml    <-- a feature-specific manifest
```

Don't be afraid to have multiple. Each one should have a clear purpose.

### Manifests and CI

For incremental deploys in CI, generating a manifest from the Git diff is the standard pattern. Example using `sfdx-git-delta`:

```bash
sfdx sgd:source:delta \
    --to HEAD \
    --from origin/main \
    --output manifest \
    --generate-delta
sf project deploy start --manifest manifest/package.xml --target-org integration
```

Only the components that changed are deployed. Faster than full deploys.

## Profiles, again, in manifests

If you do track profiles, you need to be aware that referencing a profile in a manifest will retrieve only the *fragments* of the profile that relate to other components in the manifest. This is rarely what you want.

To retrieve full profiles, use:

```xml
<types>
    <members>Sales</members>
    <members>Service</members>
    <name>Profile</name>
</types>
<types>
    <members>*</members>
    <name>ApexClass</name>
</types>
<!-- and explicit lists for every type the profile references -->
```

This is one of several reasons profiles are painful and why ignoring them in `.forceignore` is the recommended pattern.

## destructiveChanges.xml

A manifest of metadata to delete from the target org. See [Chapter 15](./15-destructive-changes.md) (or the renumbered destructive changes chapter).

Structurally identical to `package.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>Old_Field__c</members>
        <name>CustomField</name>
    </types>
    <version>62.0</version>
</Package>
```

Two timing options:

- `destructiveChangesPre.xml`: deletions before the deploy.
- `destructiveChangesPost.xml`: deletions after the deploy.

Use `Pre` for breaking changes (you need the old thing gone before the new thing deploys). Use `Post` for cleanup (the new thing replaces the old, then the old is deleted).

## A common mistake

Mixing source format and manifest-driven deploys without understanding how they interact. A manifest deploy with `sf project deploy start --manifest manifest/package.xml --source-dir force-app` ships the components listed in the manifest, sourced from the directory. The directory must contain those components.

A frequent confusion: someone updates the source, forgets to update the manifest, and the deploy ships the old version because the manifest is the index. Conversely, someone adds a component to the manifest without adding the source file, and the deploy fails.

Generate manifests from source, not by hand, except for very small ones. Diff them in CI to catch drift.

## Practical guidance

- Adopt a baseline `.forceignore` on day one. Tweak as you go.
- Prefer permission sets to profiles. Ignore profiles in `.forceignore`.
- Use manifests for releases. Generate them from Git diffs in CI.
- Keep `manifest/` in source control. Treat manifests as artifacts.
- Document each manifest's purpose in a comment at the top.

## References

- [`.forceignore` reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_exclude_source.htm)
- [Manifest file syntax](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/file_based_manifest.htm)
- [`sf project generate manifest`](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_project_commands_unified.htm)
- [`sfdx-git-delta` (community)](https://github.com/scolladon/sfdx-git-delta)
- [Destructive changes](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/daugmenteddeletes_section.htm)
