# HashiCorp Vault Enterprise - GCP Marketplace

⚠️ **TESTING ONLY - NOT FOR PRODUCTION USE** ⚠️

This is the GCP Marketplace Click-to-Deploy package for **Vault Enterprise**
using a **file storage backend** on GKE. This architecture is designed for testing
and evaluation purposes only.

## Architecture Overview

- **Type**: Kubernetes App (Click-to-Deploy via `mpdev`)
- **Storage**: File storage backend with PersistentVolume (100Gi)
- **Deployment**: Single replica Deployment (no high availability)
- **License**: Vault Enterprise license injected via `VAULT_LICENSE` env var
- **Images**: Vault Enterprise base image from Docker Hub
- **Billing**: UBB agent sidecar for GCP Marketplace usage reporting

**⚠️ Testing-Only Limitations:**
- **No High Availability**: Single pod only - pod failure means service downtime
- **No Replication**: File backend does not support HA or data replication
- **No Clustering**: Single-node deployment with no Raft or cluster configuration
- **Limited Performance**: File backend has lower throughput than integrated storage
- **Data Persistence**: All data stored on a single PersistentVolume - if PV is deleted, all data is lost

### Components

| Component | Purpose |
|-----------|---------|
| `vault` Deployment | Vault Enterprise server (fixed at 1 replica) |
| `vault-init` | Init container for pre-flight checks |
| `ubbagent` | Usage-based billing sidecar |
| `deployer` | Marketplace deployer image |
| `tester` | Marketplace verification tests |

### Services

| Service | Type | Purpose |
|---------|------|---------|
| `$name-vault` | ClusterIP | Vault API + UI access |

## Prerequisites

- GKE cluster v1.21+
- `gcloud`, `kubectl`, `docker`
- `mpdev` (auto-wrapped by validation script if missing)
- Vault Enterprise license file (`.hclic`)
- Artifact Registry repository for images

## Environment Variables

```bash
export PROJECT_ID=<your-gcp-project>
export REGISTRY=us-docker.pkg.dev/$PROJECT_ID/vault-marketplace
export TAG=1.21.0
```

## Step-by-Step Deployment

### 1) Configure GCP and Docker

```bash
gcloud auth login
gcloud config set project $PROJECT_ID
gcloud auth configure-docker us-docker.pkg.dev
```

### 2) Add Your Vault Enterprise License

Place your `.hclic` file in this directory:

```bash
cp /path/to/vault.hclic products/vault/
```

The validation script automatically detects the license file and injects it for tests.

### 3) Build Images

```bash
cd products/vault
REGISTRY=$REGISTRY TAG=$TAG make app/build
```

### 4) Run Full Marketplace Validation (Recommended)

```bash
REGISTRY=$REGISTRY TAG=$TAG \
  ../../shared/scripts/validate-marketplace.sh vault
```

Optional:

```bash
REGISTRY=$REGISTRY TAG=$TAG \
  ../../shared/scripts/validate-marketplace.sh vault --keep-deployment
```

### 5) (Optional) Direct mpdev Install/Verify

```bash
REGISTRY=$REGISTRY TAG=$TAG make app/install
REGISTRY=$REGISTRY TAG=$TAG make app/verify
```

## Post-Deploy Steps (Init + Unseal)

Vault deploys **sealed**. Initialize and unseal before use.

```bash
# Get the pod name
POD_NAME=$(kubectl get pods -n $namespace -l app.kubernetes.io/name=vault -o jsonpath='{.items[0].metadata.name}')

# Initialize
kubectl exec -it $POD_NAME -n $namespace -- vault operator init

# Unseal (run 3 times with different keys)
kubectl exec -it $POD_NAME -n $namespace -- vault operator unseal
```

### Access the UI

```bash
kubectl port-forward svc/$name-vault -n $namespace 8200:8200
```

Open: `https://localhost:8200`

## Resource Defaults

Vault server pods default to:

- **Requests**: `cpu: 2000m`, `memory: 8Gi`
- **Limits**: `cpu: 2000m`, `memory: 16Gi`

These can be overridden via schema inputs:

- `vaultResourcesRequestsCpu`
- `vaultResourcesRequestsMemory`
- `vaultResourcesLimitsCpu`
- `vaultResourcesLimitsMemory`

## Schema Properties (Key)

| Property | Default | Description |
|---------|---------|-------------|
| `storageSize` | 100Gi | PersistentVolume size for Vault data |
| `vaultLicense` | required | Enterprise license (masked) |
| `reportingSecret` | required | Marketplace billing secret |
| `vaultResourcesRequestsCpu` | 2000m | CPU request per pod |
| `vaultResourcesRequestsMemory` | 8Gi | Memory request per pod |
| `vaultResourcesLimitsCpu` | 2000m | CPU limit per pod |
| `vaultResourcesLimitsMemory` | 16Gi | Memory limit per pod |

## Troubleshooting

### Common Issues

| Error | Cause | Fix |
|-------|-------|-----|
| `ImagePullBackOff` | Wrong tag or registry config | Verify `TAG`, run `gcloud auth configure-docker us-docker.pkg.dev` |
| `license is not valid` | Missing/expired license | Verify `vaultLicense` secret |
| `vault status` exit 2 | Vault is sealed | Unseal with `vault operator unseal` |
| PVC not binding | Storage class issues | Check PVC status: `kubectl get pvc -n $namespace` |

### Verify Enterprise

```bash
# Get the pod name
POD_NAME=$(kubectl get pods -n $namespace -l app.kubernetes.io/name=vault -o jsonpath='{.items[0].metadata.name}')

# Check logs for Enterprise
kubectl logs -n $namespace $POD_NAME -c vault | grep "Enterprise"
```

## Cleanup

```bash
REGISTRY=$REGISTRY TAG=$TAG \
  ../../shared/scripts/validate-marketplace.sh vault --cleanup
```

## Version Synchronization Checklist

Keep these in sync when bumping versions:

- `schema.yaml` (`publishedVersion`)
- `apptest/deployer/schema.yaml` (`publishedVersion`)
- `manifest/application.yaml.template` (`version`)
- `product.yaml` (`version`)
- `Makefile` (`VERSION`, `VAULT_VERSION`)
