# Deploy Transient Environment

A GitHub composite action for deploying transient Helm-based environments by inheriting values from existing HelmReleases.

## Overview

This action simplifies the deployment of temporary environments (PR previews, feature branches, etc.) by:

- Fetching configuration from an existing HelmRelease in your cluster
- Deploying a new Helm release with inherited values
- Overriding specific values for the transient environment

Perfect for creating isolated preview environments that match your dev/staging configuration.

## Usage

```yaml
- name: Deploy PR Preview
  uses: starburst997/deploy-transient@v1
  with:
    kube-config: ${{ secrets.KUBE_CONFIG }}
    helm-name: my-app
    namespace: my-app
    chart-version: 1.2.3
    ingress-host: pr-123.myapp.com
```

## Inputs

| Input                   | Required | Default                        | Description                                 |
| ----------------------- | -------- | ------------------------------ | ------------------------------------------- |
| `kube-config`           | ✅       | -                              | Kubernetes configuration (base64 or plain)  |
| `helm-name`             | ✅       | -                              | Base name for the Helm release              |
| `namespace`             | ✅       | -                              | Base namespace (without suffixes)           |
| `chart-version`         | ✅       | -                              | Chart version to deploy                     |
| `repository-owner`      | ❌       | `github.repository_owner`      | Repository owner/organization               |
| `chart-name`            | ❌       | `github.event.repository.name` | Chart name (repository name)                |
| `chart-prefix`          | ❌       | `charts/`                      | Chart prefix path in OCI registry           |
| `suffix-pr`             | ❌       | `-pr`                          | Suffix for target namespace                 |
| `suffix-dev`            | ❌       | `-dev`                         | Suffix for source namespace (non-staging)   |
| `source-namespace`      | ❌       | _auto_                         | Override source namespace detection         |
| `environment`           | ❌       | `dev`                          | Environment name (affects source detection) |
| `registry`              | ❌       | `ghcr.io`                      | OCI registry URL                            |
| `ingress-host`          | ❌       | -                              | Ingress host override                       |
| `future-version`        | ❌       | -                              | Future version env var                      |
| `additional-set-values` | ❌       | -                              | Extra `--set` values (one per line)         |
| `create-namespace`      | ❌       | `false`                        | Create namespace if missing                 |
| `helm-timeout`          | ❌       | `5m`                           | Helm timeout duration                       |

### Namespace Logic

**Target namespace**: `{namespace}{suffix-pr}` (default: `my-app-pr`)

**Source namespace** (auto-detected if not specified):

- **staging**: Uses `{namespace}` (e.g., `my-app`)
- **other**: Uses `{namespace}{suffix-dev}` (e.g., `my-app-dev`)

## Example: PR Preview Workflow

```yaml
jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy Preview
        uses: starburst997/deploy-transient@v1
        with:
          kube-config: ${{ secrets.KUBE_CONFIG }}
          helm-name: s3-mirror-sample
          namespace: s3-mirror-sample
          chart-version: ${{ steps.version.outputs.version }}
          ingress-host: ${{ steps.deployment.outputs.domain }}
          future-version: ${{ steps.version.outputs.future-version }}
```

This will:

- Deploy to namespace: `s3-mirror-sample-pr` (namespace + default `-pr` suffix)
- Fetch values from: `s3-mirror-sample-dev` (namespace + default `-dev` suffix)
- Use chart: `oci://ghcr.io/{repository_owner}/charts/{repository_name}` (with default `charts/` prefix)
- Use default registry: `ghcr.io`

## How It Works

1. **Setup**: Installs Helm and kubectl
2. **Configure**: Sets up Kubernetes context
3. **Fetch**: Retrieves values from existing HelmRelease (`kubectl get helmrelease`)
4. **Deploy**: Runs `helm upgrade --install` with:
   - Base values from source HelmRelease
   - Overrides for transient-specific config
   - Atomic rollback on failure

## Requirements

- Kubernetes cluster with existing HelmRelease resources
- kubectl access to source and target namespaces
- Helm 3.x compatible charts

## License

MIT
