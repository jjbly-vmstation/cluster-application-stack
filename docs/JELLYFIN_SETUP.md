# Jellyfin Setup Guide

This guide covers the complete setup and configuration of Jellyfin media server on the VMStation Kubernetes cluster.

## Overview

Jellyfin is an open-source media server that manages and streams your personal media collection. This deployment is optimized for the VMStation cluster with proper storage configuration and Kubernetes best practices.

## Prerequisites

- Kubernetes cluster running (v1.29+)
- kubectl configured with cluster access
- Storage node (`storagenodet3500`) available
- Media files accessible at `/srv/media` on the storage node

## Quick Start

### Deploy with Ansible

```bash
# Production deployment
cd ansible
ansible-playbook playbooks/deploy-jellyfin.yml -e environment=production

# Staging deployment
ansible-playbook playbooks/deploy-jellyfin.yml -e environment=staging
```

### Deploy with kubectl

```bash
# Production
kubectl apply -k kustomize/overlays/production

# Staging
kubectl apply -k kustomize/overlays/staging
```

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `JELLYFIN_PublishedServerUrl` | External URL for server | `http://localhost:8096` |
| `JELLYFIN_HTTP_BIND_ADDRESS` | Bind address for HTTP server | `0.0.0.0` |
| `JELLYFIN_HTTP_PORT` | HTTP port | `8096` |
| `JELLYFIN_LOG_LEVEL` | Log verbosity | `Information` |

### Storage Paths

| Path | Description | Type |
|------|-------------|------|
| `/config` | Jellyfin configuration and metadata | PVC |
| `/cache` | Transcoding and cache files | EmptyDir |
| `/media` | Media library (read-only) | HostPath |

### Network Ports

| Port | Protocol | Description |
|------|----------|-------------|
| 8096 | TCP | HTTP web interface |
| 8920 | TCP | HTTPS web interface |
| 1900 | UDP | DLNA service discovery |
| 7359 | UDP | Client discovery |

## Storage Configuration

### Production Setup

In production, Jellyfin uses:
- **Config PVC**: 5Gi persistent volume for configuration
- **Cache EmptyDir**: 10Gi ephemeral storage for transcoding
- **Media HostPath**: `/srv/media` on storage node (read-only)

### Preparing Media Directory

On the storage node (`storagenodet3500`):

```bash
# Create media directory structure
sudo mkdir -p /srv/media/{movies,tv,music,photos}
sudo chown -R 1000:1000 /srv/media
sudo chmod -R 755 /srv/media

# Create config directory
sudo mkdir -p /var/lib/jellyfin
sudo chown 1000:1000 /var/lib/jellyfin
```

### Organizing Media

Recommended folder structure:
```
/srv/media/
├── movies/
│   └── Movie Name (Year)/
│       └── Movie Name (Year).mkv
├── tv/
│   └── Series Name/
│       └── Season 01/
│           └── Series Name - S01E01 - Episode Name.mkv
├── music/
│   └── Artist/
│       └── Album/
│           └── 01 - Track Name.flac
└── photos/
    └── Year/
        └── Event/
            └── photo.jpg
```

## Accessing Jellyfin

### Production

- **URL**: `http://192.168.4.61:30096`
- **NodePort**: 30096 (HTTP), 30920 (HTTPS)

### Initial Setup

1. Navigate to the Jellyfin URL
2. Select your preferred language
3. Create an admin account
4. Add media libraries (use `/media/*` paths)
5. Configure remote access settings

## Health Checks

The deployment includes comprehensive health checks:

### Startup Probe
- **Path**: `/health`
- **Initial Delay**: 180 seconds
- **Period**: 15 seconds
- **Failure Threshold**: 40

### Readiness Probe
- **Path**: `/health`
- **Initial Delay**: 240 seconds
- **Period**: 30 seconds
- **Failure Threshold**: 10

### Liveness Probe
- **Path**: `/health`
- **Initial Delay**: 300 seconds
- **Period**: 60 seconds
- **Failure Threshold**: 5

## Troubleshooting

### Pod Not Starting

```bash
# Check pod status
kubectl get pods -n jellyfin

# View pod events
kubectl describe pod jellyfin -n jellyfin

# Check logs
kubectl logs -n jellyfin deployment/jellyfin
```

### Common Issues

#### Pod stuck in ContainerCreating
- Verify storage node is available
- Check PVC status: `kubectl get pvc -n jellyfin`

#### Health check failures
- Jellyfin takes 3-5 minutes to fully start
- Check if port 8096 is accessible from the pod
- Verify network policies allow traffic

#### Permission denied on media
- Ensure media directory is owned by UID 1000
- Check SELinux/AppArmor policies on the host

### Logs

```bash
# Follow logs
kubectl logs -f -n jellyfin deployment/jellyfin

# Previous container logs (after crash)
kubectl logs -n jellyfin deployment/jellyfin --previous
```

## Backup and Restore

### Backup

```bash
# On storage node
tar -czvf jellyfin-config-backup.tar.gz /var/lib/jellyfin

# Or use PVC snapshot if supported
kubectl get volumesnapshot -n jellyfin
```

### Restore

```bash
# Stop Jellyfin
kubectl scale deployment jellyfin -n jellyfin --replicas=0

# Restore on storage node
tar -xzvf jellyfin-config-backup.tar.gz -C /

# Restart Jellyfin
kubectl scale deployment jellyfin -n jellyfin --replicas=1
```

## Performance Tuning

### Hardware Acceleration

If the node has a GPU, add the following to the deployment:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1  # For NVIDIA GPUs
    # or
    gpu.intel.com/i915: 1  # For Intel QuickSync
```

### Transcoding

For optimal transcoding performance:
1. Use Intel QuickSync if available
2. Increase cache EmptyDir size for heavy transcoding
3. Consider dedicated transcoding storage

### Memory

Adjust resources based on library size:
- Small library (<1000 items): 512Mi-1Gi
- Medium library (1000-10000 items): 1Gi-2Gi
- Large library (>10000 items): 2Gi-4Gi

## Upgrading

### Changing Image Version

Edit the Kustomize overlay:

```yaml
# kustomize/overlays/production/kustomization.yaml
images:
  - name: jellyfin/jellyfin
    newTag: "10.9.12"  # New version
```

Apply the update:

```bash
kubectl apply -k kustomize/overlays/production
```

### Rolling Back

```bash
# Check deployment history
kubectl rollout history deployment/jellyfin -n jellyfin

# Rollback to previous version
kubectl rollout undo deployment/jellyfin -n jellyfin
```

## Security Considerations

1. **Non-root container**: Jellyfin runs as UID 1000
2. **Dropped capabilities**: All Linux capabilities are dropped
3. **Read-only media**: Media mount is read-only
4. **Network policies**: Consider implementing NetworkPolicies
5. **API authentication**: Enable API key authentication in Jellyfin settings

## References

- [Jellyfin Documentation](https://jellyfin.org/docs/)
- [Jellyfin Docker Guide](https://jellyfin.org/docs/general/administration/installing#docker)
- [Media Naming Conventions](https://jellyfin.org/docs/general/server/media/movies/)
