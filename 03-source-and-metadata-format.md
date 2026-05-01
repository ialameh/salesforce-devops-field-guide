# 03. Source Format and Metadata Format

Two formats. Same content. Different shape. Most Salesforce DevOps confusion starts with not knowing which one you are looking at.

## The short version

- **Metadata format**: what the Metadata API consumes and produces. A flat ZIP of XML files. Fine for machines.
- **Source format**: what `sf project retrieve` produces. Files broken up for humans, optimised for diffs and merges.

You author and commit in source format. The CLI converts to metadata format internally when it talks to the org.

## What changes between the two

### Custom objects

Metadata format puts the entire object in one file:

```
objects/Account.object
```

The file contains every field, validation rule, list view, record type, layout assignment, etc.

Source format breaks it apart:

```
objects/Account/
    Account.object-meta.xml
    fields/
        Status__c.field-meta.xml
        Region__c.field-meta.xml
    listViews/
        AllAccounts.listView-meta.xml
    validationRules/
        BillingCountryRequired.validationRule-meta.xml
    recordTypes/
        Standard.recordType-meta.xml
        Premium.recordType-meta.xml
```

This is the biggest reason for source format. A single change to one validation rule produces a one-file diff in source format. The same change in metadata format produces a 2000-line diff in `Account.object`.

### Lightning Web Components

Metadata format keeps each LWC as a folder, but the bundle metadata is named differently. Source format names the metadata file `<componentName>.js-meta.xml` and pairs it with the component files in the same folder.

### Translations

Metadata: one file per language with all customisations.

Source: split by category (custom labels, picklist values, etc.) into smaller files.

### Profiles and permission sets

Both formats keep profiles as single files. They are notoriously large and merge-conflict-prone. See the `.forceignore` discussion in [Chapter 5](./05-forceignore-and-manifests.md).

## When you see metadata format

Three places:

1. **Outputs from older tools.** Workbench, Eclipse, Ant Migration Tool, change sets. They speak metadata format.
2. **Some retrievals.** `sf project retrieve start --target-metadata-dir` produces a metadata-format zip.
3. **Manifest-based deploys.** `sf project deploy start --manifest manifest/package.xml` is metadata-format under the hood, even when the source is in source format.

Most of the time, you should not see metadata format. If you do, convert.

## Converting between formats

```bash
# Source format -> metadata format (for legacy tools or change set manipulation)
sf project convert source --root-dir force-app --output-dir metadata-output

# Metadata format -> source format (after retrieving with --target-metadata-dir)
sf project convert mdapi --root-dir metadata-output --output-dir force-app
```

Both commands are non-destructive. They write to a different directory.

## The package.xml manifest

A list of metadata components, in metadata format conventions. Used to:

- Tell the CLI what to retrieve.
- Tell the CLI what to deploy.
- Document a release's contents.

A small example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>OrderStatusController</members>
        <members>OrderStatusControllerTest</members>
        <name>ApexClass</name>
    </types>
    <types>
        <members>Account.Status__c</members>
        <members>Account.Region__c</members>
        <name>CustomField</name>
    </types>
    <types>
        <members>BillingCountryRequired</members>
        <name>ValidationRule</name>
    </types>
    <version>62.0</version>
</Package>
```

`*` is shorthand for "all of this type":

```xml
<types>
    <members>*</members>
    <name>ApexClass</name>
</types>
```

Use `*` for whole-package retrieves and migrations. For deploys, prefer explicit lists. They are diffable, reviewable, and self-documenting.

## destructiveChanges.xml

A second manifest that tells the CLI what to *delete*. Same shape as `package.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>Account.Old_Field__c</members>
        <name>CustomField</name>
    </types>
    <version>62.0</version>
</Package>
```

Two variants:

- `destructiveChangesPre.xml`: deletions that run before the deploy.
- `destructiveChangesPost.xml`: deletions that run after the deploy.

See [Chapter 15](./15-destructive-changes.md) for the long discussion.

## Practical guidance

- Author in source format. Commit source format. Review source format diffs.
- Convert to metadata format only when a tool requires it.
- Use manifests for documenting releases and for explicit deploys.
- Don't mix. Don't try to deploy a metadata-format zip as if it were source format.

## A common confusion

The `manifest/package.xml` file in your project is *not* the same as the `package.xml` you see in metadata-format outputs. The former is your manifest of source you may want to deploy. The latter is the index of a metadata-format ZIP. Same name, different role.

Keep them straight. The manifest in `manifest/` evolves with your codebase. The one in metadata outputs is a snapshot at a moment.

## When you might convert intentionally

- Migrating a legacy project from metadata format to source format. One-off conversion.
- Building a release artifact for a downstream tool that expects metadata format.
- Producing a change set-equivalent ZIP for an org that accepts only metadata API.

In each case, convert at the edge. Keep your repo in source format internally.

## References

- [Source format vs metadata format](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_source_file_format.htm)
- [`sf project convert source` and `convert mdapi`](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_project_commands_unified.htm)
- [Manifest file syntax](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/file_based_manifest.htm)
- [Destructive changes](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/daugmenteddeletes_section.htm)
