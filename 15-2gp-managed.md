# 15. 2GP Managed Packages

For ISVs publishing on the AppExchange and for projects that need true IP protection. More overhead than Unlocked. Different rules. Same general shape.

## What you get

- A namespace prefix on every component (`mypkg__OrderStatus__c`).
- Hidden Apex source in target orgs (customers can't read your code).
- Atomic, versioned upgrades.
- Customer-managed install/upgrade workflows.
- AppExchange listing support (with security review).
- Salesforce-managed updates (Salesforce can patch managed packages).

In exchange:

- A namespace forever (cannot be removed once registered).
- Stricter rules: no editing in target, ancestry locks, deprecation only.
- A security review for the AppExchange (slow, formal, painful first time).
- License management complexity.

## Prerequisites

- A **namespace** registered in your Dev Hub.
- A **packaging org** for legacy 1GP-style flows (less needed for pure 2GP managed).
- Salesforce Partner Program membership for AppExchange.

The namespace is the point of no return. Once registered to a Dev Hub, it's tied there permanently. Pick carefully.

## Registering a namespace

1. In the Dev Hub: Setup, Packages, Namespace Settings.
2. Pick a unique 3-15 character namespace.
3. Confirm.

The namespace must:

- Start with a letter.
- Be alphanumeric.
- Not match an existing namespace anywhere on Salesforce.

Once registered, the namespace shows up in `sfdx-project.json`:

```json
"namespace": "mypkg"
```

All metadata in this Dev Hub gets the prefix.

## Creating the package

```bash
sf package create \
    --name "MyApp Managed" \
    --package-type Managed \
    --path force-app \
    --target-dev-hub devhub
```

The package type is `Managed`. The `Package2` record is created with that type.

## Building versions

Same command as Unlocked:

```bash
sf package version create \
    --package "MyApp Managed" \
    --installation-key-bypass \
    --code-coverage \
    --wait 60 \
    --target-dev-hub devhub
```

What's different in the build:

- Components get the namespace prefix.
- Apex `public` modifiers get hidden in the install. Only `global` is callable from outside.
- `WebService` Apex methods become globally exposed.
- Some metadata types are restricted (e.g., certain Setup configurations).

## Beta vs Released

Same as Unlocked: betas can install in non-production orgs, released versions install anywhere.

```bash
sf package version promote \
    --package "MyApp Managed@1.0.0-1" \
    --target-dev-hub devhub
```

Once promoted, the version is immutable and entered into Salesforce's records. AppExchange listings track promoted versions.

## Ancestry

The most distinctive rule of Managed packages.

When you create a new version, you specify which version is the *ancestor*. The new version must build on top of the ancestor without breaking changes.

```json
"ancestorVersion": "1.0.0.1"
```

Or via id:

```json
"ancestorId": "04t..."
```

Ancestry rules:

- A `public` method cannot become `private` between ancestor and descendant.
- A `global` method cannot be removed.
- Custom fields cannot be deleted (only deprecated).
- Custom objects cannot be deleted.

This is a feature: customers depend on your package's API, and ancestry guarantees backward compatibility.

It's also a constraint: you cannot just rip out old code. Plan deprecations carefully.

## Deprecation

The way to remove things from a managed package:

1. Mark the component as deprecated in code: `@Deprecated` or by removing it from `global` API exposure.
2. Ship a new version with the deprecation.
3. Wait at least one release cycle (often more) for customers to update their code.
4. In a future major version, remove or rework.

Salesforce documents the specific rules per metadata type. They are stricter than most platforms.

## Code visibility

In a target org:

- `public` Apex classes and methods are not visible to other namespaces. Only the namespace owner can call them.
- `global` Apex is visible.
- Apex source is hidden from the customer's developer console.
- Customers cannot see, edit, or override your managed code.

Fields, objects, and metadata records may be visible (read), but not editable by default. The package author can selectively allow extensibility through:

- `protected` custom metadata (can be referenced but not edited by customers).
- Subscriber overrides (formula fields, validation rules) where the package allows.

## License management

Managed packages can be licensed. Each customer org gets a license that controls who in the org can use which features.

For paid AppExchange listings:

- Set up the License Management App (LMA) in your packaging org.
- Define license types (Free, Free Trial, Paid).
- Customer subscriptions show up in your LMA.
- Salesforce handles the billing for paid listings.

For free internal-only managed packages, license management is optional.

## Push upgrades

Managed package authors can push upgrades to subscribed orgs:

```bash
sf package install \
    --package "MyApp Managed@2.0.0-1" \
    --target-org customerOrg \
    --upgrade-type DeprecateOnly \
    --apex-compile package
```

Customers can also pull upgrades themselves via Setup, Installed Packages.

The `Push Upgrades` feature (different from manual install) lets you forcibly upgrade subscribers. Use carefully; customers expect to be informed.

## Patch versions

Patches are semver-style minor updates that don't require ancestry validation in the same way. Useful for bug fixes:

```json
"versionNumber": "1.0.1.NEXT",
"isReleased": true
```

Salesforce limits patch versions per major version.

## AppExchange security review

To list a managed package on the AppExchange, Salesforce requires a security review.

The review covers:

- Apex security (CRUD/FLS enforcement, SOQL injection, XSS in Visualforce/LWC).
- Permissions and sharing.
- Authentication and session management.
- Customer-data handling.

The review is detailed. Companies have failed it on first submission and spent quarters fixing.

Tools that help:

- **PMD** with Apex rulesets.
- **Salesforce Code Analyzer** (the modern wrapper).
- **Checkmarx** for static security analysis.

Run these continuously, not just before submission. Habits beat heroics.

## Common managed package gotchas

### "I forgot to mark this `global`"

Every public class, method, or field that needs to be callable across namespaces must be `global`. Missing this means customers cannot use what you intended.

The lesson: design the API surface deliberately. Mark `global` what you intend as public, `public` only for namespace-internal helpers.

### Ancestry forces forever-compatibility

Once you ship a `global void doThing(String x)`, you cannot change its signature. You can deprecate and add `global void doThing(String x, Integer y)` alongside, but the old one stays.

Plan public APIs with this constraint in mind. Conservative APIs age better.

### Tests must be `@isTest`

In a managed package, test classes have to be marked `@isTest`. Salesforce runs them during install and upgrade to verify the package state. Failed tests can block customer upgrades.

### Customer overrides

Customers can override certain metadata in your package: validation rules, layouts, picklist values. Test that your code handles overridden state gracefully.

## When 2GP Managed is the wrong choice

- Internal projects that don't need IP protection. Use Unlocked.
- Single-org deployments. Use org-based deploys.
- Projects you intend to keep agile (managed package's ancestry constraint is real).

## When 2GP Managed is the right choice

- You sell or distribute Salesforce code as a product.
- IP protection is a real requirement.
- You commit to multi-year ancestry compatibility.
- You have the capacity to handle security review.

## References

- [Managed 2GP overview](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_create_managed_pkg.htm)
- [Namespace registration](https://help.salesforce.com/s/articleView?id=sf.register_a_namespace.htm)
- [License Management App](https://help.salesforce.com/s/articleView?id=sf.lma_lma_overview.htm)
- [AppExchange security review](https://partners.salesforce.com/s/education/general/Security_Review)
- [Push upgrades](https://developer.salesforce.com/docs/atlas.en-us.pkg2_dev.meta/pkg2_dev/sfdx_dev2gp_push_upgrade.htm)
