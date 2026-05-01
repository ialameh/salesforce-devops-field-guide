# 14. 2GP Unlocked Packages

The most useful packaging type for internal Salesforce teams. Versioned, atomic, multi-target, no namespace required. This chapter is the practical recipe.

## What you need

- A Dev Hub.
- Salesforce CLI authenticated to the Dev Hub.
- A `packageDirectory` in `sfdx-project.json` containing the source you want to package.

## Create the package

```bash
sf package create \
    --name "MyApp" \
    --package-type Unlocked \
    --path force-app \
    --description "My internal app" \
    --target-dev-hub devhub
```

This creates a `Package2` record in the Dev Hub and updates `sfdx-project.json` with a package alias.

After:

```json
{
  "packageDirectories": [
    {
      "path": "force-app",
      "default": true,
      "package": "MyApp",
      "versionNumber": "1.0.0.NEXT",
      "versionName": "Initial"
    }
  ],
  "packageAliases": {
    "MyApp": "0Ho..."
  }
}
```

## Create a package version

A package version is an immutable build:

```bash
sf package version create \
    --package "MyApp" \
    --installation-key-bypass \
    --wait 30 \
    --target-dev-hub devhub
```

Behind the scenes:

1. The CLI creates a temporary scratch org.
2. Deploys your source into it.
3. Records the resulting metadata as a `Package2Version`.
4. Returns a version id (`04t...`).
5. Updates `sfdx-project.json` with the new alias.

After success:

```json
"packageAliases": {
    "MyApp": "0Ho...",
    "MyApp@1.0.0-1": "04t..."
}
```

The `1.0.0-1` is the version number plus build number. `1.0.0` came from `versionNumber` in `sfdx-project.json`. `-1` is the build counter.

## Install a package version

In a target org:

```bash
sf package install \
    --package "MyApp@1.0.0-1" \
    --target-org sandbox \
    --wait 10
```

Or by version id:

```bash
sf package install --package 04t... --target-org sandbox --wait 10
```

The version id is portable; the alias is your repo's name for it.

## Promote a version to released

Beta versions install only in non-production orgs. Released versions install anywhere, including production:

```bash
sf package version promote \
    --package "MyApp@1.0.0-1" \
    --target-dev-hub devhub
```

Once promoted, the version is immutable. Cannot be deleted. Cannot be re-promoted. Plan accordingly.

## Installation keys

For sensitive packages, set an installation key:

```bash
sf package version create \
    --package "MyApp" \
    --installation-key "secret-key" \
    --wait 30 \
    --target-dev-hub devhub
```

Installation requires the key:

```bash
sf package install --package "MyApp@1.0.0-1" \
    --installation-key "secret-key" \
    --target-org target
```

Use this for packages you do not want random parties installing.

## Version building options

A few flags worth knowing:

```bash
sf package version create \
    --package "MyApp" \
    --version-name "Spring 26" \
    --version-description "Includes order automation" \
    --version-number "1.1.0.NEXT" \
    --code-coverage \
    --release-notes-url "https://example.com/release-notes/1.1.0" \
    --post-install-url "https://example.com/post-install" \
    --wait 60 \
    --target-dev-hub devhub
```

- `--version-name`: human-readable version name.
- `--version-description`: longer description.
- `--code-coverage`: requires Apex tests to pass with 75%+ coverage. Strongly recommended.
- `--release-notes-url`: shown in target org during install.
- `--post-install-url`: page to redirect to after install.

## Package directory configuration

`sfdx-project.json` controls everything:

```json
{
  "packageDirectories": [
    {
      "path": "force-app",
      "default": true,
      "package": "MyApp",
      "versionName": "v1.0",
      "versionNumber": "1.0.0.NEXT",
      "versionDescription": "Initial release",
      "definitionFile": "config/project-scratch-def.json",
      "scopeProfiles": false,
      "ancestorVersion": null,
      "dependencies": []
    }
  ]
}
```

Most fields are optional. The required ones are `path`, `package`, `versionNumber`. Everything else has defaults.

The `versionNumber` uses `MAJOR.MINOR.PATCH.BUILD`, where `BUILD` is usually `NEXT` (auto-increment) or a specific number. See [Chapter 17](./17-versioning.md).

## Multiple packages

Larger projects split into multiple package directories, each becoming its own package:

```json
"packageDirectories": [
    {
      "path": "core",
      "default": true,
      "package": "MyApp Core",
      "versionNumber": "1.0.0.NEXT"
    },
    {
      "path": "extension",
      "package": "MyApp Extension",
      "versionNumber": "1.0.0.NEXT",
      "dependencies": [
        { "package": "MyApp Core@1.0.0-1" }
      ]
    }
]
```

Build them in dependency order:

```bash
sf package version create --package "MyApp Core" --target-dev-hub devhub --wait 30
# Update dependencies in sfdx-project.json to reference the new core version
sf package version create --package "MyApp Extension" --target-dev-hub devhub --wait 30
```

## Listing what you have

```bash
# All packages in the Dev Hub
sf package list --target-dev-hub devhub

# Versions of a specific package
sf package version list --packages "MyApp" --target-dev-hub devhub

# What's installed in a target org
sf package installed list --target-org sandbox
```

## Updating an installed package

Each new version is installable as an upgrade:

```bash
sf package install \
    --package "MyApp@1.1.0-1" \
    --target-org sandbox \
    --wait 10 \
    --upgrade-type DeprecateOnly
```

`--upgrade-type` options:

- `DeprecateOnly`: existing components stay, new ones add. Default.
- `Mixed`: components can be deleted as part of the upgrade.
- `Delete`: removes obsolete components. Use carefully.

## Uninstalling a package

```bash
sf package uninstall --package "MyApp@1.0.0-1" --target-org sandbox --wait 10
```

Salesforce blocks uninstall if uninstall would break references (e.g., custom fields used in flows from another package). Resolve the references first.

## Common pitfalls

### Forgetting `--code-coverage`

Without it, you can create a package version even if Apex tests fail. The version is technically valid but cannot be promoted to released.

Always use `--code-coverage`. Catch coverage problems at build time, not at promotion time.

### Conflicting metadata between packages

If two packages declare the same custom field, install fails in any org that has both.

Mitigations:

- Plan field ownership. One package owns each field.
- Use namespaces for managed packages. Unlocked packages don't, which is part of why this is a risk.
- Test installs in a sandbox before promoting to production.

### Soft references

Apex in Package A references Apex in Package B. If Package B is not present in the target, install fails. The dependency must be explicit:

```json
"dependencies": [{ "package": "Package B@1.0.0-1" }]
```

### Version explosion

Without discipline, you create dozens of beta versions. The Dev Hub keeps them all. Quota fills.

Mitigations: only build versions on merge to main. Promote to released sparingly. Delete unpromoted betas you no longer need.

## Org-Dependent variant

Add `orgDependent` to the package directory:

```json
{
  "path": "force-app",
  "default": true,
  "package": "MyApp",
  "versionNumber": "1.0.0.NEXT",
  "orgDependent": true
}
```

This relaxes the validation that all references resolve at build time. The package builds without checking that referenced metadata exists; references are validated at install time, against the target org.

Useful when your package depends on org-specific custom objects that vary per customer. Most projects do not need this; turn it on deliberately when you do.

## A simple build pipeline

```bash
#!/bin/bash
set -euo pipefail

# Create version
VERSION_ID=$(sf package version create \
    --package "MyApp" \
    --installation-key-bypass \
    --code-coverage \
    --wait 60 \
    --target-dev-hub devhub \
    --json | jq -r '.result.SubscriberPackageVersionId')

echo "Created version: $VERSION_ID"

# Install in integration sandbox
sf package install \
    --package "$VERSION_ID" \
    --target-org integration \
    --wait 10

# Run smoke tests
sf apex run test --target-org integration --test-level RunLocalTests --wait 30

# If smoke passes, promote
sf package version promote \
    --package "$VERSION_ID" \
    --target-dev-hub devhub
```

This is the spine of a 2GP unlocked release pipeline.

## When 2GP Unlocked is the wrong choice

- Single-org projects with simple deploys. Org-based deploys are simpler.
- AppExchange listings (use Managed).
- Projects where IP visibility matters (use Managed).

For everyone else, Unlocked is the recommended packaging path.

## References

- [Unlocked Packages](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_create_unlocked_pkg.htm)
- [`sf package` commands](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_package_commands_unified.htm)
- [Org-Dependent Unlocked Packages](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_org_dependent_pkg.htm)
- [Package versioning](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_define_pkg_versions.htm)
