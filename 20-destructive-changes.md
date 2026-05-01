# 20. Destructive Changes

Deletions in Salesforce are not undo-able and not always safe. This chapter covers how to delete metadata via deploy, when to use Pre versus Post timing, and the failure modes that come up most.

## What "destructive change" means

A destructive change is a deletion of metadata from the target org. Examples:

- Deleting a custom field.
- Deleting a custom object.
- Deleting an Apex class.
- Removing a layout assignment.
- Deleting a flow.

Salesforce executes deletions through a `destructiveChanges.xml` file, which is structurally identical to a `package.xml` but interpreted as "delete these instead of deploy these".

## destructiveChanges.xml syntax

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>Account.Old_Field__c</members>
        <name>CustomField</name>
    </types>
    <types>
        <members>OldFlow</members>
        <name>Flow</name>
    </types>
    <version>62.0</version>
</Package>
```

Place this file in your `manifest/` directory.

## Pre vs Post timing

Two file names matter:

- `destructiveChangesPre.xml`: delete *before* the deploy.
- `destructiveChangesPost.xml`: delete *after* the deploy.

Same shape. Different timing.

### When to use Pre

The new components in your deploy conflict with the old ones. You need the old gone before the new can land.

Example: replacing a custom field with a different type:

- Pre: delete `Account.Old_Type__c`.
- Deploy: create `Account.Old_Type__c` with the new type.

If you tried to deploy without the Pre delete, Salesforce would refuse because the field already exists with a different definition.

### When to use Post

The new components replace the old, and you want the old cleaned up after.

Example: adding a new flow and retiring the old one:

- Deploy: create `NewFlow`.
- Post: delete `OldFlow`.

The old flow is removed only after the new one is in place. Less risk of a brief window with no flow at all.

If in doubt: Post. Most deletions are Post.

## Including destructive changes in a deploy

```bash
sf project deploy start \
    --manifest manifest/package.xml \
    --pre-destructive-changes manifest/destructiveChangesPre.xml \
    --post-destructive-changes manifest/destructiveChangesPost.xml \
    --target-org sandbox
```

Either or both can be included.

## What can be deleted

Most metadata types support destructive changes. The exceptions:

- Things tied to data (custom field with non-null records still has the data).
- Things tied to other things (a custom object referenced in a flow can't be deleted until the flow is updated).
- Things in managed packages (you can't delete the publisher's metadata).

## What can NOT be deleted

A few specific cases:

- Standard fields and objects (Account.Name, etc.).
- Required system metadata.
- Components from installed managed packages.
- Components actively in use (Salesforce blocks if usage is detected).

## Soft delete vs hard delete

Some metadata types have soft delete. Custom fields, for example: when you "delete" a field through the UI, it goes to the recycle bin. You have 15 days to restore.

Through the metadata API and `destructiveChanges.xml`, the delete is the same: it goes to the recycle bin, then is hard-deleted later.

To hard-delete immediately, use the UI's "Erase" button or empty the recycle bin programmatically.

## Common failures

### "References still exist"

The deploy fails because something still references the metadata you're trying to delete.

Diagnosis: read the error. It names the referencing component.

Fix: update the referencing component to not reference the deletion target. Deploy together.

Example: deleting `Account.Old_Field__c` while a flow still references it. Either delete the flow first, or update the flow to use a different field.

### "Cannot delete this object: it has data"

For custom objects with records, deletion is not allowed via metadata API. The records have to be removed first.

Fix: bulk delete the records:

```bash
sf data query --query "SELECT Id FROM My_Object__c" --target-org sandbox --result-format csv > records.csv
# Then bulk delete via Data Loader or sf data delete
```

Then delete the object.

### "Cannot delete this picklist value: it's in use"

Picklist values cannot be deleted while records use the value. Either:

1. Update records to use a different value.
2. Mark the value inactive instead of deleting.

The metadata API has `deactivate` semantics for picklist values that are gentler than `delete`.

### "Mixed deletion and creation in same deploy"

Sometimes you cannot both create and delete a component named the same thing in one deploy. Use destructiveChangesPre to delete first, then the deploy creates.

### "Profile or permission set references missing field"

After deleting a field, profiles and permission sets may still reference it. Salesforce usually cleans these up automatically; if not, retrieve the profile fresh.

## Generating destructive changes from Git diff

```bash
sfdx sgd:source:delta \
    --to HEAD \
    --from origin/main \
    --output manifest \
    --generate-delta
```

This generates both `package.xml` (additions and updates) and `destructiveChanges.xml` (deletions).

It uses `destructiveChangesPost.xml` by default. If your changes need Pre, you'll need to manually manage that file.

## Production destructive changes

Real talk: deletions in production are higher-risk than additions.

A few rules that hold up:

- **Validate before destroying.** A `destructiveChanges.xml` should go through `deploy validate` first, just like any other deploy.
- **Check for data first.** Before deleting a custom field, query the org for non-null values. Decide whether to lose them or migrate.
- **Soft-deprecate before hard-deleting.** For ISVs, `@Deprecated` an Apex method for one or two releases before removing.
- **Communicate.** Stakeholders may have processes that depend on what's being deleted.

## Backups

Always: export the metadata you're about to delete. If something goes wrong, you have it.

```bash
sf project retrieve start \
    --metadata "CustomField:Account.Old_Field__c" \
    --target-org production \
    --target-metadata-dir backups/$(date +%Y%m%d)
```

Or include the entire affected component set:

```bash
sf project retrieve start \
    --manifest manifest/destructiveChangesPost.xml \
    --target-org production \
    --target-metadata-dir backups/$(date +%Y%m%d)
```

If the deploy goes wrong, you can re-deploy the backup.

## Backups for data, not just metadata

Metadata backup is only half the story. If you delete a custom field, the data goes with it (or to the recycle bin temporarily).

Before any production destructive change involving fields with data:

1. Export records to a CSV via Data Loader or `sf data export tree`.
2. Store the export in a versioned location (S3 with a date prefix, etc.).
3. Confirm the export is complete and readable.
4. Then deploy.

## Rollback after a bad delete

You deleted something, you regret it.

For custom fields: the recycle bin holds them for 15 days. Restore via Setup or via metadata API.

For Apex classes: re-deploy from source control. The class is back.

For flows: re-deploy. Activation status may need re-toggling.

For data: depends on whether you deleted records. Recycle bin gives you 15 days for soft-deleted records.

After 15 days: data is gone. Hope you have backups.

## When to use destructive changes in CI

Auto-generate them from Git diff. Apply via `sfdx-git-delta` or similar:

```bash
sfdx sgd:source:delta \
    --to HEAD \
    --from origin/main \
    --output manifest \
    --generate-delta

sf project deploy start \
    --manifest manifest/package.xml \
    --post-destructive-changes manifest/destructiveChangesPost.xml \
    --target-org integration
```

For production, gate destructive changes behind explicit human approval. Don't auto-deploy deletions to prod.

## A safe destructive-change workflow

1. **Identify what to delete.** From a feature branch's diff.
2. **Review what's affected.** Search for usages. List dependencies.
3. **Backup metadata.** Retrieve to a versioned folder.
4. **Backup data** (if applicable). Export to CSV.
5. **Validate the deploy.** Catch errors before production.
6. **Deploy to integration first.** Verify nothing downstream breaks.
7. **Test.** Run the smoke tests.
8. **Sign-off.** Human approval.
9. **Production deploy with quick deploy.** Promote the validated job.
10. **Monitor.** Watch for unexpected errors.

This is more ceremony than a routine deploy. Destructive changes earn it.

## References

- [Destructive Changes](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/daugmenteddeletes_section.htm)
- [Deploying with destructive changes](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_deploy.htm)
- [`sfdx-git-delta`](https://github.com/scolladon/sfdx-git-delta)
- [Recycle bin](https://help.salesforce.com/s/articleView?id=sf.home_delete.htm)
