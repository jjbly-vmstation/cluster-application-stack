# VMStation Cluster Application Stack

This repository manages application layer workloads for the VMStation Kubernetes cluster.

## Overview

The cluster-application-stack contains:
- **Media servers** (Jellyfin)
- **Lab applications** and custom workloads
- **Kubernetes manifests** with Kustomize overlays
- **Ansible playbooks** for automated deployments


## Repository Structure

```
cluster-application-stack/
├── README.md                        # This file
├── IMPROVEMENTS_AND_STANDARDS.md    # Best practices documentation
├── manifests/                       # Kubernetes manifests
├── kustomize/                       # Kustomize configurations
├── helm-charts/                     # Helm charts (future)
├── ansible/                         # Ansible automation
```

## Documentation

All detailed application and deployment documentation has been centralized in the [cluster-docs/components/](../cluster-docs/components/) directory. Please refer to that location for:
- Application deployment guides
- Jellyfin setup
- Storage configuration

This repository only contains the README and improvements/standards documentation.

## Quick Start

### Prerequisites

- Kubernetes cluster running (v1.29+)
- kubectl configured with cluster access
- Ansible installed (for playbook deployments)

### Deploy with Ansible (Recommended)

```bash
cd ansible

# Deploy all applications to production
ansible-playbook playbooks/deploy-applications.yml -e deploy_environment=production

# Deploy only Jellyfin
ansible-playbook playbooks/deploy-jellyfin.yml -e deploy_environment=production
```

### Deploy with Kustomize

Run these `kubectl` commands on the Linux control-plane host (masternode) or any Linux host with a working kubeconfig (e.g. `/etc/kubernetes/admin.conf`).

```bash
# Deploy to production
kubectl apply -k kustomize/overlays/production

# Deploy to staging
kubectl apply -k kustomize/overlays/staging

# Preview configuration
kubectl kustomize kustomize/overlays/production
```

### Deploy Individual Manifests

```bash
# Apply Jellyfin directly
kubectl apply -f manifests/jellyfin/

# Validate without applying
kubectl apply -f manifests/jellyfin/ --dry-run=client
```

## Applications

### Jellyfin Media Server

A self-hosted media streaming solution.

| Attribute | Value |
|-----------|-------|
| Namespace | `jellyfin` |
| Port | 8096 (HTTP), 8920 (HTTPS) |
| NodePort | 30096 (HTTP), 30920 (HTTPS) |
| Node | storagenodet3500 |

**Access (direct, no SSO)**: http://192.168.4.61:30096

**Access (web SSO via oauth2-proxy)**: http://192.168.4.61:30097

Note: the Jellyfin SSO flow is browser-based and redirects your browser to Keycloak. For LAN clients, the Keycloak authorization endpoint must be externally reachable (e.g., `http://192.168.4.63:30180/auth`). The production overlay configures oauth2-proxy to use the external Keycloak URL for browser redirects while using the in-cluster Keycloak service for token + JWKS.

Note: Jellyfin native clients (TV/phone apps) generally do not work well behind oauth2-proxy because it requires a browser-based OIDC login flow and cookie handling. Use the direct endpoint for apps, and the SSO endpoint for browser access.
Tip: for TVs/phones, Jellyfin supports "Quick Connect" pairing so family members can authorize a device from the web UI without typing a password on the device.

For setup details, see [docs/JELLYFIN_SETUP.md](docs/JELLYFIN_SETUP.md).

## Cluster Nodes

| Node | IP | Role | Applications |
|------|-----|------|--------------|
| masternode | 192.168.4.63 | Control Plane | Monitoring stack |
| storagenodet3500 | 192.168.4.61 | Worker (Storage) | Jellyfin |
| homelab | 192.168.4.62 | Worker (Compute) | General workloads |

## Environments

### Production
- Specific image versions (not `:latest`)
- NodePort services for external access
- Node selectors for proper placement
- Reduced logging verbosity

### Staging
- Uses `:latest` tags
- Reduced resource limits
- Debug-level logging
- ClusterIP services

## Documentation

| Document | Description |
|----------|-------------|
| [IMPROVEMENTS_AND_STANDARDS.md](IMPROVEMENTS_AND_STANDARDS.md) | Best practices and standards |
| [docs/JELLYFIN_SETUP.md](docs/JELLYFIN_SETUP.md) | Jellyfin setup guide |
| [docs/APPLICATION_DEPLOYMENT.md](docs/APPLICATION_DEPLOYMENT.md) | Deployment procedures |
| [docs/STORAGE_CONFIGURATION.md](docs/STORAGE_CONFIGURATION.md) | Storage setup and management |
| [kustomize/README.md](kustomize/README.md) | Kustomize usage guide |
| [helm-charts/README.md](helm-charts/README.md) | Helm charts (future) |

## Contributing

1. Create a feature branch
2. Add or modify manifests
3. Update Kustomize overlays if needed
4. Test with `kubectl apply --dry-run`
5. Update documentation
6. Submit a pull request

## Related Repositories

- [vmstation](https://github.com/jjbly-vmstation/vmstation) - Original monorepo
- [cluster-infrastructure](https://github.com/jjbly-vmstation/cluster-infrastructure) - Cluster infrastructure

## License

Apache License 2.0 - See [LICENSE](LICENSE) for details.
