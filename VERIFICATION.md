# Verification Notes

This guide was cross-checked against published Salesforce documentation in **May 2026**, against the **Spring '26** release. Salesforce ships changes every release; some claims in this guide may have evolved by the time you read it.

If you find a claim that no longer matches the official Salesforce documentation, open a PR or an issue.

## Verification scope

| Topic | Verified against |
|-------|-----------------|
| `sfdx-project.json` schema | DX Developer Guide, Spring '26 |
| `sf` CLI commands and flags | CLI command reference, May 2026 |
| Source format and metadata format | Salesforce DX Developer Guide |
| Apex governor limits | Apex Developer Guide, current |
| `@InvocableMethod` and `callout=true` | Apex Developer Guide |
| `AiAuthoringBundle` (referenced as agent metadata in [25-ai-assisted-development]) | Metadata API, Spring '26 |
| 2GP package structure (Unlocked, Managed) | 2GP Developer Guide |
| Ancestry rules | 2GP Developer Guide |
| Scratch org features and definition fields | DX Developer Guide |
| Org shapes | DX Developer Guide |
| Snapshots | DX Developer Guide |
| Source tracking behavior | DX Developer Guide |
| Sandbox types and refresh cadence | Salesforce Help |
| JWT bearer flow | Salesforce Help |
| Connected app configuration | Salesforce Help |

## Items most likely to change

These parts are most exposed to platform changes. Re-verify when you upgrade to a new release.

| Item | Why it can change |
|------|-------------------|
| `sf` CLI command flags and behavior | Salesforce iterates the CLI continuously |
| Scratch org feature names | New features added each release |
| Sandbox limits | Salesforce occasionally adjusts |
| 2GP package limits and capabilities | Active area of Salesforce development |
| Source tracking behavior in sandboxes | Recent feature, still evolving |
| Pricing model | Restructured in 2025; further changes possible |

## Acknowledged uncertainty

Some items are based on community practice or author experience rather than official documentation.

- **CI auth patterns.** Multiple valid approaches; the guide picks the recommended ones rather than enumerating all.
- **Cookbook examples.** Inspired by real projects, but the specific code is illustrative. Adapt to your project.
- **Migration paths.** Salesforce documents 1GP-to-2GP migration; the guide summarises but does not replace the official migration steps.
- **AI tool integration patterns.** Active area; specific tool behaviours change often.

When in doubt, prefer the canonical Salesforce documentation linked in [REFERENCES.md](./REFERENCES.md). This guide aims to be useful, but the platform is the source of truth.

## How to re-verify

When you suspect a claim is out of date:

1. Check the official Salesforce documentation linked in REFERENCES.md.
2. Check the most recent Release Notes.
3. Test against a current scratch org if it might be runtime behaviour.
4. Open an issue or PR with the finding.

## Verification process for v0.1.0

The initial release was verified by:

1. Cross-checking each chapter's specific claims against the linked Salesforce documentation.
2. Running selected commands against a current scratch org to confirm CLI behavior.
3. Testing 2GP package commands end-to-end.
4. Reviewing the Spring '26 release notes for changes affecting the guide's claims.

The verification covered approximately 200 specific claims across the 28 chapters. The corrections that resulted are reflected in the published version.
