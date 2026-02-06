# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with the Vault Enterprise GCP Marketplace product.

## Build Commands

**Prerequisites:**
```bash
# Authenticate to GCP and configure Artifact Registry
gcloud auth login && gcloud auth configure-docker us-docker.pkg.dev

# Place your Vault Enterprise license file in this directory
cp /path/to/your/vault.hclic .

# No registry login needed - Vault Enterprise images are publicly available on Docker Hub
```

**Standard Validation Workflow (USE THIS):**
```bash
# Full validation pipeline - builds, schema check, install, verify, vuln scan
# The shared script auto-detects the *.hclic file and includes license in parameters
REGISTRY=us-docker.pkg.dev/ibm-software-mp-project-test/vault-marketplace TAG=1.21.0 \
  ../../shared/scripts/validate-marketplace.sh vault

# With --keep-deployment to inspect after validation
REGISTRY=us-docker.pkg.dev/ibm-software-mp-project-test/vault-marketplace TAG=1.21.0 \
  ../../shared/scripts/validate-marketplace.sh vault --keep-deployment
```

**Individual targets (for quick iteration):**
```bash
# Build only
REGISTRY=us-docker.pkg.dev/ibm-software-mp-project-test/vault-marketplace TAG=1.21.0 make app/build

# Direct mpdev verify
REGISTRY=us-docker.pkg.dev/ibm-software-mp-project-test/vault-marketplace TAG=1.21.0 make app/verify

# Direct mpdev install
REGISTRY=us-docker.pkg.dev/ibm-software-mp-project-test/vault-marketplace TAG=1.21.0 make app/install
```

**Other targets:**
- `make clean` - Clean local build artifacts

## Architecture

⚠️ **TESTING ONLY - NOT FOR PRODUCTION USE** ⚠️

This is a GCP Marketplace Click-to-Deploy product for **Vault Enterprise** using a **file storage backend** (single replica, no HA). This is a simplified testing architecture.

**Key Limitations:**
- Single replica Deployment (no high availability)
- File storage backend (no clustering or replication)
- All data on single PersistentVolume (100Gi)
- No Raft consensus or leader election

### Image Source
- Base image: `hashicorp/vault-enterprise:1.21-ent` (Docker Hub, publicly available)
- No registry login required (unlike Consul/Terraform Enterprise which use `images.releases.hashicorp.com`)

### Image Build Pipeline
```
images/vault/Dockerfile          -> us-docker.pkg.dev/.../vault:TAG
images/vault-init/Dockerfile     -> us-docker.pkg.dev/.../vault/vault-init:TAG
(pulled from Google)             -> us-docker.pkg.dev/.../vault/ubbagent:TAG
deployer/Dockerfile              -> us-docker.pkg.dev/.../vault/deployer:TAG
apptest/deployer/Dockerfile      -> us-docker.pkg.dev/.../vault/tester:TAG
```

### Key Files
- `schema.yaml` - GCP Marketplace schema defining user inputs
- `apptest/deployer/schema.yaml` - Test schema with default values for mpdev verify
- `manifest/manifests.yaml.template` - Kubernetes resources (Secret, ConfigMap, Service, PVC, Deployment)
- `manifest/application.yaml.template` - GCP Marketplace Application CRD
- `product.yaml` - Product metadata

### Version Synchronization
All files must have matching versions:
- `schema.yaml` -> `publishedVersion: '1.21.0'`
- `apptest/deployer/schema.yaml` -> `publishedVersion: '1.21.0'`
- `manifest/application.yaml.template` -> `version: '1.21.0'`
- `product.yaml` -> `version: '1.21.0'`

## Enterprise License

**Build-time:** No registry auth needed — Vault Enterprise images are on Docker Hub.

**Runtime (deployment):**
1. `schema.yaml` -> `vaultLicense` property (MASKED_FIELD in GCP Marketplace UI)
2. Secret: `$name-vault-license` (created from schema property)
3. Environment variable: `VAULT_LICENSE` in Deployment vault container
4. License file (`*.hclic`) auto-detected by `validate-marketplace.sh` for mpdev verify/install

## Debugging

```bash
# Check pod status
kubectl get pods -n <namespace>

# Get pod name (single replica)
POD_NAME=$(kubectl get pods -n <namespace> -l app.kubernetes.io/name=vault -o jsonpath='{.items[0].metadata.name}')

# Check Vault logs
kubectl logs -n <namespace> $POD_NAME -c vault

# Check if running Enterprise
kubectl logs -n <namespace> $POD_NAME -c vault | grep "Enterprise"

# Check Vault status
kubectl exec -n <namespace> $POD_NAME -- vault status

# Check PVC status
kubectl get pvc -n <namespace>

# Access UI
kubectl port-forward svc/$name-vault -n <namespace> 8200:8200
```

## Common Issues

| Error | Cause | Fix |
|-------|-------|-----|
| `ImagePullBackOff` | Wrong tag or registry misconfiguration | Verify TAG matches publishedVersion and `gcloud auth configure-docker us-docker.pkg.dev` was run |
| `license is not valid` | Missing or expired Enterprise license | Check `vaultLicense` secret and license expiry |
| `No .hclic file found` | License file missing | Place your Vault Enterprise license (*.hclic) in the product directory |
| `vault status` exit 2 | Vault is sealed (normal after deploy) | Initialize and unseal: `vault operator init` then `vault operator unseal` |
| PVC not binding | Storage issues | Check `kubectl get pvc -n <namespace>` and PV provisioning |
