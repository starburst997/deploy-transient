# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a GitHub composite action that deploys transient Helm-based environments by inheriting values from existing HelmReleases. It's designed for creating temporary environments like PR previews or feature branch deployments that inherit configuration from existing dev/staging deployments.

## Architecture

### Composite Action Structure

This is a **composite action** (not a JavaScript or Docker action), defined entirely in `action.yml`. The action executes bash scripts directly within GitHub Actions workflow steps.

### Key Components

**action.yml**: The entire action implementation, containing:
- Input definitions (kube-config, helm-name, namespace, chart-version, etc.)
- Three composite steps:
  1. Set up Helm (using azure/setup-helm@v4)
  2. Setup kubectl (using azure/setup-kubectl@v4)
  3. Deploy transient environment (bash script that does the actual work)

### Deployment Flow

1. **Namespace Resolution**: Target namespace is `{namespace}{suffix-pr}` (default: `-pr`). Source namespace is auto-detected based on environment:
   - `staging` environment → uses `{namespace}` as source
   - Other environments → uses `{namespace}{suffix-dev}` as source (default: `-dev`)
   - Can be overridden with `source-namespace` input

2. **Value Inheritance**: Fetches existing HelmRelease values using:
   ```bash
   kubectl get helmrelease {helm-name} -n {source-namespace} -o jsonpath='{.spec.values}'
   ```

3. **Helm Deployment**: Runs `helm upgrade --install` with:
   - Base values from source HelmRelease
   - Chart from OCI registry: `oci://{registry}/{repository-owner}/{chart-prefix}{chart-name}`
   - Overrides for transient-specific config (ingress-host, future-version, etc.)
   - Atomic rollback on failure

### OCI Registry Chart Path

The chart URL is constructed as:
```
oci://{registry}/{repository-owner}/{chart-prefix}{chart-name}:{chart-version}
```

Defaults:
- registry: `ghcr.io`
- repository-owner: `github.repository_owner`
- chart-prefix: `charts/`
- chart-name: `github.event.repository.name`

## Development

### Testing Locally

This action cannot be easily tested locally since it's a composite action. To test:

1. Push changes to a branch
2. Reference the branch in a test workflow:
   ```yaml
   uses: starburst997/deploy-transient@your-branch-name
   ```

### Testing in a Workflow

Create a test workflow in `.github/workflows/` that uses the action with all required inputs:
- `kube-config`: Kubernetes config (base64 or plain)
- `helm-name`: Name of the Helm release
- `namespace`: Base namespace without suffixes
- `chart-version`: Version to deploy

### Releasing

This action uses GitHub releases with tags (e.g., `v1`). Users reference it via:
```yaml
uses: starburst997/deploy-transient@v1
```

### Key Assumptions

- The cluster already has a HelmRelease resource deployed (likely via Flux CD)
- The source namespace contains a HelmRelease with the same name as `helm-name` input
- kubectl has access to both source and target namespaces
- The OCI registry is accessible (default: ghcr.io)

### Important Implementation Details

- The action uses `--atomic` flag for helm, which rolls back on failure
- Namespace creation is optional via `create-namespace` input (default: false)
- The action builds the helm command dynamically, appending `--set` flags for optional values
- Additional helm values can be passed via `additional-set-values` (newline-separated)

## Documentation Synchronization Rule

**CRITICAL**: When adding, removing, or modifying inputs or outputs in `action.yml`, you MUST update all three documentation sources:

1. **action.yml** - The source of truth for input/output definitions
2. **README.md** - The inputs table and usage examples (lines 30-48 for the inputs table)
3. **docs/index.html** - The reference section with input documentation (lines 467-603 contain the reference cards)

### Update Checklist

When modifying inputs/outputs:
- [ ] Update `action.yml` with the new/modified input or output
- [ ] Update the inputs table in `README.md` (ensure required/optional, defaults, and descriptions match)
- [ ] Update `docs/index.html` reference cards with the same information
  - Required inputs go in the "Required Inputs" card (starting line 469)
  - Namespace-related inputs go in "Namespace Configuration" card (starting line 505)
  - Chart-related inputs go in "Chart Configuration" card (starting line 537)
  - Other deployment options go in "Deployment Options" card (starting line 569)
- [ ] Verify all defaults, descriptions, and requirements are consistent across all three files
