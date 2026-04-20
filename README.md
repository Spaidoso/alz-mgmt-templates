# ALZ Management Templates

Shared GitHub Actions workflow templates and composite actions for the [alz-mgmt](https://github.com/Spaidoso/alz-mgmt) Azure Landing Zones repository.

## Overview

This repository provides reusable CI/CD components for deploying Azure Landing Zones infrastructure using Bicep. The `alz-mgmt` repo calls these templates via `workflow_call`.

## Repository Structure

```
alz-mgmt-templates/
├── .github/
│   ├── actions/
│   │   ├── bicep-deploy/           # Deploy Bicep via Azure Deployment Stacks
│   │   ├── bicep-first-deployment-check/  # Detect first-time deployments
│   │   ├── bicep-installer/        # Install Bicep CLI + Az PowerShell
│   │   └── bicep-variables/        # Extract parameters.json to env vars
│   └── workflows/
│       ├── ci-template.yaml        # CI: Bicep validation + What-If
│       └── cd-template.yaml        # CD: What-If + Deploy
└── README.md
```

---

## Workflow Templates

### CI Template (`ci-template.yaml`)

Triggered by: `workflow_call` from consumer repos (PRs to main)

| Job | Purpose |
|-----|---------|
| **validate** | Runs `bicep build` on all `.bicep` files to check syntax/lint |
| **whatif** | Authenticates via OIDC and runs What-If against all management groups |

### CD Template (`cd-template.yaml`)

Triggered by: `workflow_call` from consumer repos (push to main)

| Job | Purpose |
|-----|---------|
| **whatif** | Runs What-If (skippable via `skip_what_if` input) |
| **deploy** | Deploys all Bicep templates (requires `alz-mgmt-apply` environment approval) |

#### CD Workflow Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `skip_what_if` | boolean | `false` | Skip the What-If job entirely |
| `governance-int-root` | boolean | `true` | Deploy intermediate root MG |
| `governance-landingzones` | boolean | `true` | Deploy landing zones MG |
| `governance-landingzones-corp` | boolean | `true` | Deploy corp landing zone |
| `governance-landingzones-online` | boolean | `true` | Deploy online landing zone |
| `governance-platform` | boolean | `true` | Deploy platform MG |
| `governance-platform-connectivity` | boolean | `true` | Deploy connectivity MG |
| `governance-platform-identity` | boolean | `true` | Deploy identity MG |
| `governance-platform-management` | boolean | `true` | Deploy management MG |
| `governance-platform-security` | boolean | `true` | Deploy security MG |
| `governance-sandbox` | boolean | `true` | Deploy sandbox MG |
| `governance-decommissioned` | boolean | `true` | Deploy decommissioned MG |
| `governance-platform-rbac` | boolean | `true` | Deploy platform RBAC |
| `governance-platform-connectivity-rbac` | boolean | `true` | Deploy connectivity RBAC |
| `governance-landingzones-rbac` | boolean | `true` | Deploy landing zones RBAC |
| `core-logging` | boolean | `true` | Deploy Log Analytics |
| `networking-hubnetworking` | boolean | `true` | Deploy hub networking |

---

## Composite Actions

### `bicep-deploy`

Deploys Bicep templates using Azure Deployment Stacks.

| Input | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Deployment name |
| `displayName` | Yes | Display name for logging |
| `templateFilePath` | Yes | Path to `.bicep` file |
| `templateParametersFilePath` | Yes | Path to `.bicepparam` file |
| `subscriptionId` | Yes | Target subscription ID |
| `resourceGroupName` | Yes | Target resource group (empty for MG/sub deployments) |
| `location` | Yes | Deployment location |
| `deploymentType` | Yes | `managementGroup`, `subscription`, or `resourceGroup` |
| `whatIfEnabled` | No | Run What-If only (default: `false`) |
| `firstRunWhatIf` | No | Run What-If on first deployment (default: `false`) |

### `bicep-variables`

Extracts variables from `parameters.json` and sets them as GitHub environment variables.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `parameters_file_name` | Yes | `parameters.json` | Path to parameters file |

**Expected parameters.json structure:**
```json
{
  "LOCATION": "westus2",
  "MANAGEMENT_GROUP_ID": "alz",
  "SUBSCRIPTION_ID_MANAGEMENT": "e4fdb784-...",
  "SUBSCRIPTION_ID_CONNECTIVITY": "82ce8884-..."
}
```

### `bicep-installer`

Installs the Bicep CLI and upgrades Az PowerShell to the latest version.

No inputs required.

### `bicep-first-deployment-check`

Checks if this is the first deployment by looking for child management groups under the intermediate root MG.

| Input | Required | Description |
|-------|----------|-------------|
| `managementGroupId` | Yes | Intermediate root management group ID |

Sets `firstDeployment` environment variable to `true` or `false`.

---

## Usage from alz-mgmt

**CI Workflow (`.github/workflows/ci.yaml`):**
```yaml
name: 01 Azure Landing Zones Continuous Integration
on:
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: Spaidoso/alz-mgmt-templates/.github/workflows/ci-template.yaml@main
    secrets: inherit
```

**CD Workflow (`.github/workflows/cd.yaml`):**
```yaml
name: 02 Azure Landing Zones Continuous Delivery
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      skip_what_if:
        type: boolean
        default: false

jobs:
  cd:
    uses: Spaidoso/alz-mgmt-templates/.github/workflows/cd-template.yaml@main
    with:
      skip_what_if: ${{ inputs.skip_what_if || false }}
    secrets: inherit
```

---

## Making Changes

This repo has **branch protection on `main`**. To modify templates:

1. Create a feature branch
2. Make changes and push
3. Open a PR to `main`
4. Merge the PR
5. Re-run `alz-mgmt` workflows to pick up the changes

> **Note:** All references from `alz-mgmt` use `@main`, so changes take effect immediately after merge.

---

## Related Repositories

- [alz-mgmt](https://github.com/Spaidoso/alz-mgmt) - Main ALZ infrastructure repository (consumer of these templates)
