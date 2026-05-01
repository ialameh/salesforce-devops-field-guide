# 07. Scratch Org Definitions

The JSON file that defines what kind of scratch org you get. Tiny, dense, opinionated. Most fields you set once. A few you change often.

## A starter definition

```json
{
  "orgName": "My Project Dev",
  "edition": "Developer",
  "features": [],
  "settings": {
    "lightningExperienceSettings": {
      "enableS1DesktopEnabled": true
    },
    "mobileSettings": {
      "enableS1EncryptedStoragePref2": false
    }
  }
}
```

That gives you a Developer Edition scratch org with Lightning Experience enabled. Enough to write Apex and LWC. Not enough for most real projects.

## The fields

### orgName

Display name. Cosmetic. Shows up in `sf org list` and the org's UI.

### edition

The Salesforce edition. Determines what features and limits the org has out of the box.

| Value | When |
|-------|------|
| `Developer` | Default. Free, low limits, no licenses included. |
| `Enterprise` | Most production-like. Slightly higher limits. |
| `Professional` | Mid-tier features. |
| `Group` | Small business edition. Rare in DevOps. |

Pick the edition closest to what your customers run on. For ISVs, Enterprise is most common.

### features

A list of features to enable. Strings. Each one corresponds to a feature flag the Dev Hub knows about.

```json
"features": [
    "EnableSetPasswordInApi",
    "Communities",
    "EinsteinGPT",
    "AgentforceForServiceCloud"
]
```

The list of valid features is long and grows with each Salesforce release. The reference is the [Scratch Org Features documentation](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file_config_values.htm).

Pick exactly the features your project needs. More features means slower org creation and more places to debug.

### settings

A nested object that mirrors Salesforce's setup-level settings. The CLI applies them to the new org during creation.

Common settings:

```json
"settings": {
    "lightningExperienceSettings": {
        "enableS1DesktopEnabled": true
    },
    "communitiesSettings": {
        "enableNetworksEnabled": true
    },
    "omniChannelSettings": {
        "enableOmniChannel": true
    },
    "chatterSettings": {
        "enableChatter": true
    },
    "searchSettings": {
        "documentContentSearchEnabled": true
    }
}
```

The full list is in Salesforce's reference. Each setting maps to a metadata API entry.

### sourceApiVersion

Optional. Sets the API version of the scratch org's metadata. Useful when you want a specific version regardless of what the Dev Hub defaults to.

```json
"sourceApiVersion": "62.0"
```

### release

Optional. Pre-release scratch orgs let you test against the next Salesforce release.

```json
"release": "preview"
```

Three values:

| Value | What you get |
|-------|--------------|
| `preview` | Next major release, available before GA. |
| `previous` | The release before the current one. |
| (omitted) | The current release. |

For most work, omit. For testing release upgrades, set to `preview` and run your CI.

### snapshot

Optional. Use a previously created scratch org snapshot as the base. Faster than creating from scratch.

```json
"snapshot": "MyBaseSnapshot"
```

See [Chapter 10](./10-scratch-org-snapshots.md).

### namespace

Not in the scratch definition itself. The namespace is set in `sfdx-project.json` for the project. The scratch org inherits the project's namespace at creation if one is set.

## Definitions for different needs

### Vanilla developer

```json
{
  "orgName": "Dev",
  "edition": "Developer",
  "features": []
}
```

Fast to create. Limited. Good for plain Apex and LWC work.

### Service Cloud features

```json
{
  "orgName": "Service Dev",
  "edition": "Enterprise",
  "features": [
    "ServiceCloud",
    "OmniChannelForCommunity",
    "Communities"
  ],
  "settings": {
    "omniChannelSettings": { "enableOmniChannel": true },
    "caseSettings": { "feedItemSettings": { "feedItemActions": ["LogACall"] } }
  }
}
```

For Service Cloud-flavored projects.

### Sales Cloud features

```json
{
  "orgName": "Sales Dev",
  "edition": "Enterprise",
  "features": [
    "SalesCloud",
    "Forecasting",
    "PathAssistant"
  ],
  "settings": {
    "forecastingSettings": { "enableForecasts": true },
    "opportunitySettings": { "enableUpdateReminders": true }
  }
}
```

### Agentforce features

```json
{
  "orgName": "Agentforce Dev",
  "edition": "Developer",
  "features": [
    "EinsteinGPT",
    "EinsteinCopilot",
    "AgentforceForServiceCloud"
  ],
  "release": "preview"
}
```

Adjust based on which Agentforce capabilities your project uses.

## Multiple definitions

A common pattern: keep several scratch definitions for different purposes:

```
config/
    project-scratch-def.json          <-- default for daily dev
    project-scratch-perf.json         <-- larger edition for perf testing
    project-scratch-preview.json      <-- preview release
    project-scratch-minimal.json      <-- minimal features, fastest creation
```

Use `--definition-file` to pick:

```bash
sf org create scratch --definition-file config/project-scratch-perf.json --alias perf
```

## How long does scratch org creation take

Three factors:

- **Edition.** Developer is fastest. Enterprise takes a few extra minutes.
- **Features.** Each enabled feature adds time. Some features (Communities, Service Cloud) are notably slow.
- **Settings.** Most settings are fast. A few (Knowledge, OmniChannel) are slow.
- **Snapshot.** A scratch from snapshot is faster than from scratch.

A typical Developer Edition org with no features takes 60-90 seconds. A full Enterprise with Service Cloud, Communities, Knowledge, and OmniChannel can take 8-12 minutes. Plan accordingly.

## Where settings come from

The settings tree mirrors Salesforce's metadata-level settings types. Each top-level key (`lightningExperienceSettings`, `mobileSettings`, etc.) corresponds to a metadata API settings type.

To find the right shape for a setting:

1. Check the Salesforce Setup UI for the option you want.
2. Look up the corresponding Metadata API entry (e.g., the `LightningExperienceSettings` metadata type).
3. Copy the field name into your scratch definition.

The naming is consistent: setup option → camelCase field → metadata type.

## Validating your definition

Before relying on a definition, create a scratch from it and verify the org has what you expected.

```bash
sf org create scratch --definition-file config/project-scratch-def.json --alias verify
sf org open --target-org verify
```

Click around in Setup. Confirm features are enabled. Confirm settings are correct.

If something is missing, the scratch creation logs may tell you why. Common reason: a feature requires a license that the Dev Hub does not have.

## Things that cannot go in a scratch definition

- **Custom metadata records.** Scratch orgs are blank. Deploy these from source after creation.
- **User data.** Same.
- **Complex permission set assignments.** Scratch orgs have one user (you). Assign permission sets via CLI after creation.
- **External system credentials.** Set up Named Credentials via deployment, then configure the secret manually.

## A common pattern: a setup script

Most teams wrap scratch org creation in a script that handles the post-creation steps:

```bash
#!/bin/bash
set -euo pipefail

ALIAS="${1:-dev}"

sf org create scratch \
    --definition-file config/project-scratch-def.json \
    --alias "$ALIAS" \
    --duration-days 7 \
    --set-default

sf project deploy start --target-org "$ALIAS"

sf org assign permset --name My_Permission_Set --target-org "$ALIAS"

sf apex run --file scripts/create-test-data.apex --target-org "$ALIAS"

echo "Scratch org $ALIAS ready. URL:"
sf org open --target-org "$ALIAS" --url-only
```

Adapt to your project. The principle: one command, one ready-to-go scratch org.

## When the definition file changes

When you change the scratch definition (add a feature, change a setting), existing scratch orgs do not retroactively pick up the change. Delete and recreate.

This is one reason scratch orgs should be short-lived. If you treat them as long-lived, definition changes diverge them from your team's setup.

## References

- [Scratch Org Definition Configuration Values](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file_config_values.htm)
- [Scratch Org Features](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_def_file.htm)
- [Metadata API: settings types](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_settings.htm)
- [Pre-release scratch orgs](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_scratch_orgs_release.htm)
