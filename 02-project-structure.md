# 02. Project Structure

The shape of your repository determines how easy your DevOps work will be for the next two years. Get this right early. Changing it later is painful.

## The minimum viable project

```
my-project/
    config/
        project-scratch-def.json
    force-app/
        main/
            default/
                classes/
                lwc/
                objects/
                ...
    manifest/
        package.xml
    .forceignore
    .gitignore
    sfdx-project.json
    package.json
```

Generate it with `sf project generate`. Don't reinvent.

## sfdx-project.json: the file that controls everything

The most under-read file in the Salesforce ecosystem. Read it. Understand every field.

```json
{
  "packageDirectories": [
    {
      "path": "force-app",
      "default": true
    }
  ],
  "name": "my-project",
  "namespace": "",
  "sfdcLoginUrl": "https://login.salesforce.com",
  "sourceApiVersion": "62.0"
}
```

### packageDirectories

A list of folders that contain deployable source. Order matters: dependencies before dependents. The CLI walks them in order.

For a single-package project, one entry is fine. For multi-package, list them with explicit dependencies:

```json
"packageDirectories": [
    { "path": "core", "default": true,
      "package": "core",
      "versionNumber": "1.0.0.NEXT" },
    { "path": "extension", "package": "extension",
      "versionNumber": "1.0.0.NEXT",
      "dependencies": [ { "package": "core@1.0.0-1" } ] }
]
```

### namespace

The org's namespace prefix. Empty string for default-namespace projects. Set this when:

- Your scratch orgs use a namespaced package.
- You are publishing a managed package.
- You are working in a packaging-developer org.

The most common gotcha: the project namespace must match the scratch org's namespace, or the scratch org will treat references to namespaced metadata as foreign.

### sourceApiVersion

The default API version for new metadata files. Should match the API version your team is targeting. Bump it deliberately on each Salesforce release; do not let it drift behind by years.

### sfdcLoginUrl

The default login URL. `login.salesforce.com` for production-flavored orgs, `test.salesforce.com` for sandbox-flavored. Some teams override this per environment with a separate config.

### packageAliases

Once you start packaging, this section grows. Each alias maps a package name to its package id (`0Ho...`) and each version to its version id (`04t...`):

```json
"packageAliases": {
    "core": "0Ho...",
    "core@1.0.0-1": "04t...",
    "extension": "0Ho...",
    "extension@1.0.0-1": "04t..."
}
```

The CLI manages this for you when you create package versions. Don't edit by hand.

## force-app vs main vs default

`force-app/main/default/` is the canonical path. Why three levels?

- `force-app` is the package directory (configurable in `sfdx-project.json`).
- `main` is the source set. Salesforce reserves this name for the primary set.
- `default` is the metadata namespace within the package. For default-namespace projects, this is "default". For namespaced packages, you might see other folders here.

For most projects you will only ever touch `force-app/main/default/`. The other levels are scaffolding for future complexity.

## Directory layout inside force-app/main/default

Salesforce defines the structure. You don't get to invent it.

| Directory | What goes there |
|-----------|-----------------|
| `classes/` | Apex classes and their `cls-meta.xml` siblings |
| `triggers/` | Apex triggers |
| `lwc/` | Lightning Web Components, one folder per component |
| `aura/` | Aura components, one folder per bundle |
| `objects/` | Custom object metadata, one folder per object |
| `flows/` | Flow definitions |
| `permissionsets/` | Permission sets |
| `profiles/` | Profiles (use with caution; they are heavy) |
| `layouts/` | Page layouts |
| `flexipages/` | Lightning record pages |
| `tabs/` | Custom tabs |
| `staticresources/` | Static resources |
| `applications/` | Lightning apps |
| `customMetadata/` | Custom metadata records |
| `customLabels/` | Custom labels |
| `globalValueSets/` | Global picklist value sets |

A few less common ones you may need:

| Directory | What goes there |
|-----------|-----------------|
| `connectedApps/` | Connected apps (treat with care; they include secrets) |
| `namedCredentials/` | Named credentials |
| `externalCredentials/` | External credentials (newer pattern) |
| `remoteSiteSettings/` | Remote site settings (legacy) |
| `cspTrustedSites/` | CSP trusted sites |

## .forceignore

The companion to `sfdx-project.json`. Tells the CLI which files to ignore on push, deploy, retrieve, and pull. See [Chapter 5](./05-forceignore-and-manifests.md).

A starter `.forceignore`:

```
# LWC build artifacts
**/lwc/**/__tests__/**

# IDE noise
**/.vscode/**
**/.idea/**
**/*.swp

# Logs
**/*.log

# Profiles (heavy, cause merge conflicts)
**/profiles/**
```

The profiles entry is the contentious one. See Chapter 5 for the long discussion.

## Multiple package directories

For larger projects, splitting into multiple `packageDirectories` lets you build each part independently.

```json
"packageDirectories": [
    { "path": "core" },
    { "path": "feature-a", "dependencies": [{ "package": "core@1.0.0" }] },
    { "path": "feature-b", "dependencies": [{ "package": "core@1.0.0" }] }
]
```

This is the foundation of package-based DevOps. Each directory becomes a 2GP package. You can deploy any subset independently.

The trap: cross-directory references that aren't reflected in `dependencies`. The CLI may not catch them; you find out at install time when an Apex class can't see a class from another package.

## Naming conventions

A few that hold up:

- **Apex classes:** `<Domain><Action><Suffix>` like `OrderStatusController`, `CustomerLookupService`, `InvoiceTrigger`.
- **Test classes:** `<ClassUnderTest>Test`. So `OrderStatusController` → `OrderStatusControllerTest`.
- **Triggers:** `<sObject>Trigger`, e.g. `AccountTrigger`. One per object. Real logic in a handler class.
- **LWCs:** `kebab-case` folder names. Component names auto-generated as `c-kebab-case`.
- **Custom fields:** `Snake_Case_Words__c`. Avoid abbreviations except universally known ones.
- **Custom metadata types:** `Type_Name__mdt` with descriptive names like `Pricing_Rule__mdt`.
- **Permission sets:** `<Domain>_<Role>` like `Sales_Manager`, `Support_Agent_Read_Only`.

Don't be cute. Names are read more than written.

## Things that should not be in your repo

- Personal IDE settings (`.vscode/settings.json` is occasionally fine; entire workspace state is not).
- Authentication tokens, scratch org session state (`.sfdx/`, `.sf/`).
- Build artifacts (LWC `__tests__` outputs, sourcemaps).
- Customer data dumps (a frequent leak in test data setups).
- Personal browser auth (`oauth-app/`).

A solid `.gitignore` for Salesforce projects:

```
# Salesforce CLI state
.sf/
.sfdx/
.localdevserver/

# Build outputs
node_modules/
coverage/
junit/

# IDE
.vscode/
.idea/

# OS
.DS_Store
Thumbs.db

# Logs
*.log
```

## When to refactor your project structure

A few signals that the current structure is hurting you:

- Single deploy operation takes more than 20 minutes for routine changes.
- Different teams' work conflicts in the same files repeatedly.
- You want to deploy "just this feature" and can't.
- You need to release on different cadences for different parts.

The right answer is usually to introduce package directories, then graduate to 2GP packages. See [Chapter 8](./08-packaging-overview.md).

## References

- [Salesforce DX Project Configuration](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_ws_config.htm)
- [`sfdx-project.json` reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_ws_config.htm)
- [Source format](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_source_file_format.htm)
- [Multi-package project layout](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_modular_dev_model.htm)
