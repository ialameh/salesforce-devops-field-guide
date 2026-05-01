# 16. Dependencies and Ancestry

Two related but distinct concepts. Dependencies are about which packages reference which. Ancestry is about which version succeeded which. Both need clean handling or your packaging story collapses.

## Dependencies

A dependency is a declaration that one package needs another to be installed first.

```json
"packageDirectories": [
    {
      "path": "extension",
      "package": "ExtensionApp",
      "versionNumber": "1.0.0.NEXT",
      "dependencies": [
        { "package": "CoreApp@1.0.0-1" }
      ]
    }
]
```

When you install ExtensionApp into a target org, the install fails unless CoreApp version 1.0.0-1 is already there.

### Why explicit dependencies matter

- **Predictable installs.** Customers know what to install in what order.
- **Build-time validation.** Salesforce checks that referenced metadata exists in the dependency at build time.
- **Documentation.** The dependency graph is itself the architecture diagram.

### Implicit dependencies are bad

If Package A's Apex references a class in Package B, but Package A doesn't declare a dependency on Package B, the relationship is invisible. The build may succeed (especially with Org-Dependent Unlocked). The install may then fail in unexpected orgs.

Make dependencies explicit. Every cross-package reference should have a corresponding `dependencies` entry.

### Dependency versions

You can pin to:

- A specific version: `CoreApp@1.0.0-1`. The exact build.
- A floating version: `CoreApp@1.0.0`. Any build of 1.0.0.
- A loose version: `CoreApp@1.0`. Any 1.0.x build.

For most projects, pin to a specific version. Loose dependencies invite surprises.

### Dependency graph

For a project with many packages:

```
CommonUtilities (v1.2.0-3)
       |
       +---- CoreApp (v2.0.0-5)
       |        |
       |        +---- ExtensionA (v1.0.0-1)
       |        |
       |        +---- ExtensionB (v1.0.0-1)
       |
       +---- StandaloneApp (v1.0.0-1)
```

Build order: CommonUtilities first, then CoreApp, then ExtensionA and ExtensionB in parallel, then StandaloneApp at any point.

A cycle (A depends on B, B depends on A) is impossible in 2GP. The build will reject it.

### When dependencies can't be expressed cleanly

Sometimes Package A genuinely needs both Package B and a runtime configuration that lives in the target org but not in Package B. Three options:

1. **Move the configuration into Package B.** Make it a dependency too.
2. **Use Org-Dependent Unlocked.** The build skips reference validation; install validates against the target.
3. **Document the manual step.** Customers run a setup process after install.

Each has costs. The right choice depends on how stable the configuration is.

## Ancestry (Managed only)

Ancestry is a chain of versions where each builds on the previous in a backward-compatible way.

```
1.0.0-1 -> 1.0.0-2 -> 1.1.0-1 -> 1.1.0-2 -> 2.0.0-1
```

Each link in the chain promises that nothing public was removed or broken between predecessor and successor.

### Why ancestry matters

Customers running version 1.0.0-2 can upgrade to 1.1.0-1, then 2.0.0-1, without their downstream code breaking. Ancestry is the backward compatibility contract.

For Unlocked packages, ancestry is technically supported but rarely enforced. For Managed packages, it is mandatory.

### Setting an ancestor

In `sfdx-project.json`:

```json
"ancestorVersion": "1.1.0.2",
"versionNumber": "2.0.0.NEXT"
```

Or by id:

```json
"ancestorId": "04t..."
```

When building 2.0.0, Salesforce verifies that 1.1.0-2's API is preserved.

### Ancestry rules

The constraints from one ancestor to the next:

- `global` Apex methods cannot be removed.
- `global` Apex method signatures cannot change.
- Custom field types cannot change in incompatible ways.
- Custom objects cannot be deleted.
- Picklist values cannot be removed (only deprecated).
- Permissions cannot become more restrictive (in some cases).

The full list is in Salesforce's docs. Plan API design with this in mind.

### Skipping ancestors

You can skip ancestors in some cases. For example, if you have versions 1.0.0-1, 1.0.0-2, 1.1.0-1, you can declare 1.0.0-2 as the ancestor of 2.0.0-1 (skipping 1.1.0-1 if you never released it).

Use sparingly. Ancestry chains are easier to reason about when contiguous.

### Ancestry and patches

Patches are exempt from some ancestry rules. They can include bug fixes that would otherwise be considered breaking. Patches are limited per release.

### Mistakes that lock you in

- **Promoting a version with bad public API.** Once promoted, the API is part of the ancestry chain forever. Take time to design `global` carefully.
- **Forgetting `@Deprecated`.** Removing a `global` method without first deprecating it across multiple releases breaks ancestry.
- **Picklist value cleanup.** Once a value is in production, it stays.

## Combined dependencies and ancestry

For a Managed package with dependencies, both apply:

```json
{
  "path": "extension",
  "package": "ExtensionApp",
  "versionNumber": "2.0.0.NEXT",
  "ancestorVersion": "1.1.0.3",
  "dependencies": [
    { "package": "CoreApp@2.0.0-1" }
  ]
}
```

The new version of ExtensionApp ancestors from its own 1.1.0-3, and depends on CoreApp 2.0.0-1.

Both packages need to be ancestor-compatible internally, AND the dependency relationship has to make sense across versions.

## Practical recommendations

- **Start simple.** A single package, no dependencies, no ancestry. Most projects can ship 80% of their value with this.
- **Add dependencies when you split.** Splitting one package into core + extensions adds dependencies. Be deliberate.
- **Set ancestors from the start in Managed.** Don't promote anything without an ancestor (or no ancestor for v1 only). Trying to add ancestry retroactively is painful.
- **Document the dependency graph.** A README diagram saves the next person a week of investigation.
- **Test installs.** In a fresh sandbox or scratch org, install your packages in dependency order. Fail loudly if anything breaks.

## When dependencies break

Common failure modes:

### Circular dependency

Package A depends on Package B, Package B depends on Package A. Salesforce rejects.

Fix: extract the shared part into Package C. Both A and B depend on C.

### Missing dependency

Package A's Apex references something in Package B, but A doesn't declare a dependency. Build may succeed, install fails in target orgs that don't have B.

Fix: add the dependency.

### Wrong version pinned

Package A depends on `CoreApp@1.0.0-1`, but the customer org has `CoreApp@2.0.0-1`. Install of A fails with "minimum version not met" or similar.

Fix: update Package A's dependency to a compatible CoreApp version.

### Customer skipped a version

Customer is on `MyApp 1.0`, you released `MyApp 2.0`. They want to upgrade. If 2.0's ancestor is 1.0.0-2 and the customer is on 1.0.0-1, they can upgrade. If 2.0's ancestor is 1.1.0-1 and the customer is on 1.0.0-1, the upgrade fails because they skipped 1.1.

Fix: always set ancestors based on what you expect customers to have. Or release intermediate versions to bridge gaps.

## Ancestry as architectural principle

The ancestry constraint forces good design. You cannot ship a half-baked API in v1 and clean it up in v2. You have to think about API surface from the start.

Teams who treat this as a feature ship more polished APIs. Teams who fight it ship fragile ones.

## References

- [Package dependencies](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_package_dependencies.htm)
- [Package ancestry](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_package_ancestors.htm)
- [Patch versions](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_patch_packages.htm)
- [Deprecation guidelines](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_deprecation.htm)
