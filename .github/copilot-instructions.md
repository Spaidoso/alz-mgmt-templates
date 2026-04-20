# Copilot Instructions for ALZ Management Templates Repo

## Project Overview

This is the **shared workflow templates repository** for Azure Landing Zones (ALZ) CI/CD. The main consumer is the [alz-mgmt](https://github.com/Spaidoso/alz-mgmt) repository, which calls these reusable workflows via `workflow_call`.

## Relationship to alz-mgmt

| Repo | Purpose |
|------|---------|
| **alz-mgmt** | Contains Bicep templates, policy definitions, and parameters for ALZ infrastructure |
| **alz-mgmt-templates** (this repo) | Contains reusable GitHub Actions workflows and composite actions |

All references from `alz-mgmt` use `@main` branch:
```yaml
uses: Spaidoso/alz-mgmt-templates/.github/workflows/ci-template.yaml@main
```

## Architecture

### Workflow Templates

| File | Trigger | Purpose |
|------|---------|---------|
| `ci-template.yaml` | `workflow_call` | Bicep validation + What-If on PRs |
| `cd-template.yaml` | `workflow_call` | What-If + Deploy on merge to main |

Both workflows use `workflow_call`, making them reusable workflows (NOT standalone).

### Composite Actions

All actions are **composite actions** (PowerShell-based), not JavaScript or Docker:

| Action | Purpose |
|--------|---------|
| `bicep-deploy` | Deploys Bicep using Azure Deployment Stacks |
| `bicep-variables` | Extracts `parameters.json` values to env vars |
| `bicep-installer` | Installs Bicep CLI + upgrades Az PowerShell |
| `bicep-first-deployment-check` | Detects first deployment by checking child MGs |

## Azure Context

| Item | Value |
|------|-------|
| **Tenant ID** | `4d00acda-e258-43e1-bd90-9370a4d118e1` |
| **Plan Environment** | `alz-mgmt-plan` (What-If, no approval) |
| **Apply Environment** | `alz-mgmt-apply` (Deploy, manual approval) |
| **Region** | `westus2` only |
| **Auth** | OIDC federated credentials via UAMIs |

### User-Assigned Managed Identities

| Identity | Principal ID | Permissions |
|----------|--------------|-------------|
| `id-alz-mgmt-westus2-plan-001` | `3f1188ab-9ef0-447b-af80-24c0e7ea0c50` | Reader @ tenant |
| `id-alz-mgmt-westus2-apply-001` | `a8915b05-4eca-49f7-917a-40cac673b1c5` | Owner @ MG `alz` |

## Key Conventions

### Branch Protection
- `main` branch is protected — all changes require PRs
- After merging here, re-run `alz-mgmt` workflows to pick up changes

### Workflow Inputs
The CD workflow supports selective deployment via boolean inputs:
- `skip_what_if` — Skip the What-If job entirely (useful after rate limiting)
- `governance-*` — Toggle individual governance deployments
- `core-logging`, `networking-hubnetworking` — Toggle infrastructure deployments

### Rate Limiting
ALZ governance templates contain hundreds of policy definitions. Each What-If makes many ARM API calls, which can hit the tenant-level rate limit (150 requests/minute).

**Workarounds:**
1. Wait 1-2 minutes between workflow runs
2. Use `skip_what_if: true` on CD workflow dispatch
3. Don't run CI and CD simultaneously

## File Structure

```
.github/
├── actions/
│   ├── bicep-deploy/action.yaml           # Main deployment action
│   ├── bicep-first-deployment-check/action.yaml
│   ├── bicep-installer/action.yaml
│   └── bicep-variables/action.yaml
├── workflows/
│   ├── ci-template.yaml                   # CI reusable workflow
│   └── cd-template.yaml                   # CD reusable workflow
└── copilot-instructions.md                # This file
```

## Making Changes

1. Create a feature branch
2. Modify workflows or actions
3. Test changes (if possible, point `alz-mgmt` to your branch temporarily)
4. Open PR to `main`
5. Merge
6. Re-run `alz-mgmt` workflows — they reference `@main` so changes apply immediately

## Common Tasks

### Adding a New Deployment Step

1. Add input toggle to `cd-template.yaml` under `workflow_call.inputs`
2. Add What-If step to `whatif` job with `if: ${{ inputs.your-toggle }}`
3. Add Deploy step to `deploy` job with same condition
4. Update README.md with new input

### Modifying bicep-deploy Action

The action supports three deployment types:
- `managementGroup` — Deploys at MG scope
- `subscription` — Deploys at subscription scope
- `resourceGroup` — Deploys at RG scope

When `whatIfEnabled: true`, it runs What-If only. When `false`, it deploys.

### Debugging Workflow Failures

1. Check if it's a rate limiting issue (`Too Many Requests on TenantandUserLevel`)
2. Check OIDC authentication — UAMIs need correct permissions
3. Check Bicep syntax — run `bicep build` locally first
4. Check `firstDeployment` detection — may skip What-If on first run

## Documentation Maintenance

When making changes, update:
1. This file (`copilot-instructions.md`) with any new constraints or context
2. `README.md` with updated inputs, actions, or usage examples
3. `alz-mgmt/.github/copilot-instructions.md` if it affects the consumer repo
