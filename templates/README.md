# Templates

Copy-paste skeletons for Salesforce DevOps artefacts. Replace placeholders, then deploy.

## What's here

| File | Purpose |
|------|---------|
| `sfdx-project.json` | Starter SFDX project config |
| `project-scratch-def.json` | Default scratch definition |
| `project-scratch-def-ci.json` | Minimal scratch for CI |
| `package.xml` | Whole-org retrieve manifest |
| `destructiveChanges.xml` | Delete metadata template |
| `gh-actions-pr.yml` | GitHub Actions PR validation workflow |
| `gh-actions-deploy.yml` | GitHub Actions deploy workflow |
| `permissionset-skeleton.xml` | Permission set for CI users |

## How to use

1. Copy the relevant files into your project's correct locations.
2. Replace `<ANGLE_BRACKET>` placeholders with real values.
3. Commit. Deploy. Iterate.
