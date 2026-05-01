# 17. Versioning Strategy

What your version numbers mean. Whose problem it is when you bump them. The conventions that hold up over years.

## The Salesforce version format

`MAJOR.MINOR.PATCH.BUILD`

- `MAJOR`: incompatible API changes.
- `MINOR`: backward-compatible additions.
- `PATCH`: backward-compatible bug fixes.
- `BUILD`: auto-incrementing per build (use `NEXT` in `versionNumber` for auto).

For a 2GP package version `1.2.3-7`:

- 1 = MAJOR
- 2 = MINOR
- 3 = PATCH
- 7 = BUILD

## Semver, with a Salesforce twist

Roughly semver. The twist:

- BUILD has no semver equivalent. It's a Salesforce-only counter.
- "Backward-compatible" is enforced by Salesforce in Managed packages (ancestry). In Unlocked, it's a convention.

## What each bump means

### MAJOR bump

You broke something a customer (or another package) depended on. Examples:

- Removed a `global` Apex method (Managed: not allowed without long deprecation).
- Removed a custom field that other packages or code referenced.
- Changed the type of a field in an incompatible way.
- Changed an action signature.
- Removed an Apex class entirely.

For Managed packages, MAJOR is the way to introduce breaking changes that ancestry rules wouldn't allow within a MINOR. Even then, the ancestor must be a previous MAJOR's last MINOR.

### MINOR bump

You added something. Examples:

- New `global` method.
- New custom field.
- New custom object.
- New flow.
- New permission set.

Customers can upgrade without breaking. Their existing code keeps working.

### PATCH bump

You fixed something without adding or removing. Examples:

- Bug fix in existing Apex logic.
- Fixed a typo in a label.
- Fixed an incorrect formula.
- Fixed a flow.

PATCH is the only bump that should be invisible to customers (except that the bug they hit no longer happens).

### BUILD bump

Each new build of the same `MAJOR.MINOR.PATCH`. Used by the CI to keep iterating versions while you test before promoting. The customer never sees BUILD; only the promoted version's `MAJOR.MINOR.PATCH` matters.

## When to bump what

Practical guidance:

- **Every CI build** -> BUILD increments automatically (use `NEXT`).
- **Every PR ships** -> typically PATCH (small change) or MINOR (new feature).
- **Every release with new features** -> MINOR.
- **Major API change or breaking removal** -> MAJOR.

A typical project might bump MAJOR once a year, MINOR with each release (every 1-3 months), PATCH with each hotfix.

## Setting the version number

In `sfdx-project.json`:

```json
{
  "packageDirectories": [
    {
      "path": "force-app",
      "package": "MyApp",
      "versionNumber": "1.2.0.NEXT",
      "versionName": "Spring 2026"
    }
  ]
}
```

`NEXT` lets the CLI auto-increment BUILD. After build:

```json
"versionNumber": "1.2.0.NEXT"  // unchanged in source
```

The actual version is recorded in `packageAliases`:

```json
"packageAliases": {
    "MyApp@1.2.0-1": "04t...",
    "MyApp@1.2.0-2": "04t...",
    "MyApp@1.2.0-3": "04t..."
}
```

When you decide to ship MINOR 1.3 next, update `versionNumber`:

```json
"versionNumber": "1.3.0.NEXT"
```

## Version names

The human-readable label:

```json
"versionName": "Spring 2026 - Order Management Improvements"
```

Customers see this in their installed packages list. Make it useful.

Common patterns:

- Calendar version: "Spring 2026", "Q3 2026".
- Theme name: "Order Management", "Reporting Update".
- Just the number: "1.2.0".

Pick a convention. Stick to it.

## Pre-release identifiers

For betas and release candidates:

```json
"versionNumber": "2.0.0.NEXT"
```

The Salesforce 2GP system distinguishes betas (not yet promoted) from releases (promoted). Promotion is the formal step. There's no separate "rc" or "beta" suffix in the version number itself; the package version's status reflects it.

## Managing major version transitions

When 2.0 ships, 1.x customers can:

- Upgrade to 2.0 directly (if your ancestry allows).
- Stay on 1.x and receive patches you backport.

For an extended period, you may maintain both:

- Active development on `main`, building 2.0+.
- Bug fixes on a `release/1.x` branch, building 1.x patches.

This is GitFlow territory. See [Chapter 4](./04-branching.md).

## Versioning Apex API metadata

Apex has its own API version, separate from the package version. Each `.cls-meta.xml` has:

```xml
<apiVersion>62.0</apiVersion>
```

This determines which Salesforce platform features the class can use. Bumping this is a platform-level change, not a package version change.

Best practice: keep Apex API versions roughly in sync with your project's `sourceApiVersion`. Bump them deliberately on each Salesforce release.

## When customers report bugs

The reported version drives your fix:

- Customer is on 1.2.0-3, bug is in 1.2.0-3, fixed in 1.2.0-4. Tell them to upgrade.
- Bug is in current main (about to ship as 1.3.0). Backport to 1.2.0-5 if you support 1.x.

Track customer versions via your LMA (for Managed) or the package install records. Knowing who's on what helps prioritise patches.

## Semantic version drift

Common ways teams lose track:

- Bumping MAJOR for marketing reasons (1.0 to 2.0 because "version 2 sounds bigger").
- Skipping MINOR (1.0.0 -> 1.0.5 with no MINOR in between).
- Inconsistent between packages.
- BUILD numbers that don't reset between PATCH bumps.

Decide a versioning policy and document it.

## A versioning policy template

Adapt for your team:

> - Major: bumped only with a known breaking change. Approved by tech lead. Documented in changelog.
> - Minor: bumped with each new feature merged to main. Auto-bumped at release time.
> - Patch: bumped for hotfixes against current released version.
> - Build: auto-incremented by CI on every build. Customers never see this.
>
> Version names follow "Calendar Quarter - Theme" format.
>
> Promotion to released happens after QA sign-off in UAT. Beta versions stay beta until then.

## Versioning for Org-based deploys

If you're not using packages, versioning is informal. Common patterns:

- Git tags (`v1.0.0`) at each production release.
- Commit hashes for traceability.
- A `version.txt` file in the repo, updated on release.

Without packages, the version is metadata about your release process, not an artifact Salesforce tracks.

## Changelog discipline

Whatever versioning scheme, keep a `CHANGELOG.md`:

```markdown
## v1.2.0 - 2026-04-15

### Added
- Order automation flow.
- New permission set: Sales_Manager.

### Changed
- Quote calculation respects discount tier.

### Fixed
- Account lookup performance issue.
```

The changelog is your release notes. Easier to write incrementally than to reconstruct.

## References

- [Semantic Versioning](https://semver.org/)
- [Package versioning](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_define_pkg_versions.htm)
- [Keep a Changelog](https://keepachangelog.com/)
- [Apex API versions](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_versions.htm)
