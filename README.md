# gitops-dev

A GitOps configuration for the gitops-dev cluster using Helm, Helmfile, and ArgoCD for orchestrating cluster services with vault-based secret management.

## Overview

This repository orchestrates the deployment of cluster services from the [gitops-cluster-services](../gitops-cluster-services/) repository. It manages:
- **Initial bootstrap** with Vault integration for secrets management
- **Kubernetes cluster configuration** via Helm and Helmfile
- **ArgoCD deployment** for continuous GitOps syncing
- **External Secrets Operator** for managing secrets from HashiCorp Vault

## Prerequisites

### Kubernetes Setup

Ensure your active Kubernetes context is set to `gitops-dev`:

```sh
kubectx gitops-dev
kubectl config current-context
```

### Environment Variables

Before deployment, set the required environment variables. See [Environment Variables](#environment-variables) section below for details.

## Environment Variables

All required and optional environment variables are documented in `.env.example`. Copy and customize for your environment:

```sh
cp .env.example .env
# Edit .env with your vault and GitHub credentials
source .env
```

### Required Variables

No required variables - all repos are public. Vault credentials below are only needed for bootstrap.

### Vault Configuration (Bootstrap Only)

These variables are **only required for first-time deployment**. After initial bootstrap, ArgoCD manages vault credentials internally via External Secrets Operator.

| Variable | Description | Default |
|----------|-------------|---------|
| `EXTERNAL_VAULT_SERVER` | Vault server URL | None (required for bootstrap) |
| `VAULT_ADDR` | Vault server address (for helmfile plugin) | Same as EXTERNAL_VAULT_SERVER |
| `VAULT_TOKEN` | Vault authentication token | None (required for bootstrap) |
| `EXTERNAL_VAULT_PATH` | Vault KV secret path | `secret` |
| `EXTERNAL_VAULT_AUTH_PATH` | Kubernetes auth path in Vault | `kubernetes` |
| `EXTERNAL_VAULT_AUTH_ROLE` | Kubernetes auth role name | `external-secrets` |

#### Vault Secret Structure

Before deploying, store the ArgoCD admin password hash in Vault. Generate the hash:

```bash
htpasswd -nbBC 10 "" "YourStrongPassword" | sed 's/^://'
```

Then store it in Vault at the appropriate path for your environment (the gitops-cluster-services will reference this via External Secrets Operator).

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `INGRESS_REPLICAS` | Number of ingress controller replicas | `1` |
| `CLOUDFLARED_REPLICAS` | Number of Cloudflare tunnel replicas | `1` |
| `CLOUDFLARED_PROTOCOL` | Cloudflare tunnel protocol | `auto` |

## Deployment

### Phase 1: Bootstrap (Initial Deployment)

Bootstrap deploys cluster services and sets up vault integration. **Only the initial deployer needs vault credentials.**

```sh
# Set all required environment variables
source .env

# Deploy cluster services
helmfile template . | kubectl create -f - 2>/dev/null
helmfile template . | kubectl apply -f - --server-side=true 2>/dev/null
```

**What happens during bootstrap:**
1. Vault credentials (VAULT_ADDR, VAULT_TOKEN) fetch secrets from Vault:
   - ArgoCD admin password hash
   - Grafana Cloud credentials (username, password, url, type)
   - Cloudflare API credentials (apiToken, accountId, zoneId)
2. External Secrets Operator pulls these secrets and creates Kubernetes secrets
3. ArgoCD deployment reads and uses these K8s secrets
4. **ArgoCD server stores all secrets internally** for future use

### Phase 2: Team Member Deployments

After initial bootstrap, team members can deploy without any vault credentials:

```sh
# No vault credentials needed - deploy via ArgoCD
kubectl apply -f application.yaml
```

Or sync via ArgoCD UI - no setup required.

**Why team members don't need credentials:**
- ArgoCD server already has all secrets stored internally
- External Secrets Operator maintains secret synchronization
- Kubernetes secrets are managed automatically
- No one needs direct vault access after bootstrap

## Vault Secret Management

The External Secrets Operator automatically retrieves secrets from Vault and creates corresponding Kubernetes secrets in appropriate namespaces during deployment.

**Before bootstrap deployment**, ensure all required secrets are stored in your Vault instance. The gitops-cluster-services helmfile expects secrets at specific paths relative to your environment (dev/staging/prod).

Secrets are synced automatically with configurable refresh intervals (default: 1 hour). Team members never need direct Vault access after initial setup.

## Testing Cluster Services Locally

To test cluster services using local path instead of GitHub, ensure the gitops-cluster-services helmfile has the `.gotmpl` extension (Helmfile requires this for template processing):

```sh
# In gitops-cluster-services directory:
mv helmfile.yaml helmfile.yaml.gotmpl
```

Then in `helmfile.yaml.gotmpl` use the local path:

```yaml
helmfiles:
  - path: ../gitops-cluster-services/helmfile.yaml.gotmpl
```

For production or specific versions, use the remote GitHub reference:

```yaml
helmfiles:
  - path: https://github.com/draju1980/gitops-cluster-services.git/helmfile.yaml.gotmpl?ref=v1.0.0
```

## Helmfile Commands

### Template (Dry Run)

Preview what will be deployed without applying:

```sh
helmfile template .
```

### Apply

Deploy cluster services:

```sh
helmfile apply .
```

### Diff

See what would change:

```sh
helmfile diff .
```

## ArgoCD Integration

The `application.yaml` file defines the ArgoCD Application CRD for continuous syncing:

```sh
kubectl apply -f application.yaml
```

This enables:
- Automatic deployment when this repository is updated
- ArgoCD helmfile plugin for templating
- Multi-namespace application deployment
- ArgoCD-managed secret storage for vault credentials

## Important Notes

- **Vault Credentials**: Store VAULT_TOKEN securely. Never commit `.env` files or token values to git.
- **GitHub Token**: Use fine-grained personal access tokens with minimal required permissions.
- **Kubernetes Auth**: After bootstrap, the cluster uses Kubernetes authentication to Vault, not token-based auth.
- **.env Files**: Add `.env*` to `.gitignore` to prevent accidental credential commits.
