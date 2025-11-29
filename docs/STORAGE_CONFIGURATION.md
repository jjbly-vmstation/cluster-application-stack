# Storage Configuration Guide

This guide covers storage configuration for applications in the VMStation Kubernetes cluster.

## Storage Architecture

### Overview

The VMStation cluster uses a hybrid storage approach:

```
┌────────────────────────────────────────────────────────────────┐
│                     Storage Architecture                        │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────┐    ┌────────────────────────────────┐   │
│  │   Control Plane   │    │        Storage Node            │   │
│  │    (masternode)   │    │    (storagenodet3500)          │   │
│  │                   │    │                                │   │
│  │ /srv/monitoring_  │    │ /srv/media/        (500GB+)    │   │
│  │   data/           │    │   ├── movies/                  │   │
│  │   ├── prometheus  │    │   ├── tv/                      │   │
│  │   ├── loki        │    │   ├── music/                   │   │
│  │   └── grafana     │    │   └── photos/                  │   │
│  │                   │    │                                │   │
│  │                   │    │ /var/lib/jellyfin/ (Config)    │   │
│  └───────────────────┘    └────────────────────────────────┘   │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### Storage Types

| Type | Use Case | Persistence | Performance |
|------|----------|-------------|-------------|
| HostPath | Large media files | Node-level | Native disk speed |
| Local PV | Application configs | Node-level | Native disk speed |
| EmptyDir | Caching, temp files | Pod lifecycle | Memory or disk |
| PVC (Dynamic) | Generic persistent data | Cluster-level | Varies |

## Storage Classes

### Local Path Provisioner

The cluster uses the local-path-provisioner for dynamic PV provisioning:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
```

### Usage in PVCs

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi
```

## Application Storage Patterns

### Jellyfin Storage

Jellyfin uses three storage types:

1. **Configuration (PVC)**: Persistent configuration and metadata
2. **Cache (EmptyDir)**: Temporary transcoding files
3. **Media (HostPath)**: Read-only media library

```yaml
volumes:
  - name: jellyfin-config
    persistentVolumeClaim:
      claimName: jellyfin-config-pvc
  - name: jellyfin-cache
    emptyDir:
      sizeLimit: 10Gi
  - name: jellyfin-media
    hostPath:
      path: /srv/media
      type: DirectoryOrCreate
```

### Monitoring Storage

Monitoring applications use dedicated paths on the control plane:

| Application | Path | Size |
|-------------|------|------|
| Prometheus | `/srv/monitoring_data/prometheus` | 10Gi |
| Loki | `/srv/monitoring_data/loki` | 20Gi |
| Grafana | `/srv/monitoring_data/grafana` | 2Gi |

## Preparing Storage

### On Storage Node (storagenodet3500)

```bash
# Create media directories
sudo mkdir -p /srv/media/{movies,tv,music,photos}
sudo chown -R 1000:1000 /srv/media
sudo chmod -R 755 /srv/media

# Create application config directories
sudo mkdir -p /var/lib/jellyfin
sudo chown 1000:1000 /var/lib/jellyfin

# Verify disk space
df -h /srv
```

### On Control Plane (masternode)

```bash
# Create monitoring data directory
sudo mkdir -p /srv/monitoring_data
sudo chmod 755 /srv/monitoring_data

# Create application-specific subdirectories
for app in prometheus loki grafana; do
  sudo mkdir -p /srv/monitoring_data/$app
done

# Set ownership (varies by application)
sudo chown 65534:65534 /srv/monitoring_data/prometheus  # nobody user
sudo chown 10001:10001 /srv/monitoring_data/loki        # loki user
sudo chown 472:472 /srv/monitoring_data/grafana         # grafana user
```

## Backup Strategies

### Configuration Backup

```bash
# Backup Jellyfin config
tar -czvf jellyfin-backup-$(date +%Y%m%d).tar.gz /var/lib/jellyfin

# Backup monitoring configs
tar -czvf monitoring-backup-$(date +%Y%m%d).tar.gz /srv/monitoring_data
```

### Automated Backups

Create a CronJob for automated backups:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: config-backup
  namespace: kube-system
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            command:
            - /bin/sh
            - -c
            - |
              tar -czvf /backup/jellyfin-$(date +%Y%m%d).tar.gz /config
            volumeMounts:
            - name: config
              mountPath: /config
              readOnly: true
            - name: backup
              mountPath: /backup
          volumes:
          - name: config
            hostPath:
              path: /var/lib/jellyfin
          - name: backup
            hostPath:
              path: /srv/backups
          restartPolicy: OnFailure
          nodeSelector:
            kubernetes.io/hostname: storagenodet3500
```

### Restore Procedure

```bash
# Stop the application
kubectl scale deployment jellyfin -n jellyfin --replicas=0

# Restore backup on node
sudo tar -xzvf jellyfin-backup-20241015.tar.gz -C /

# Fix permissions if needed
sudo chown -R 1000:1000 /var/lib/jellyfin

# Start the application
kubectl scale deployment jellyfin -n jellyfin --replicas=1
```

## Storage Best Practices

### 1. Use Appropriate Storage Types

| Data Type | Recommended Storage | Reason |
|-----------|---------------------|--------|
| Media files | HostPath | Large files, native performance |
| App config | PVC | Needs persistence, portable |
| Temp/cache | EmptyDir | Ephemeral, fast |
| Logs | EmptyDir (sizeLimit) | Limited retention |

### 2. Set Resource Limits

Always set size limits for EmptyDir:

```yaml
volumes:
  - name: cache
    emptyDir:
      sizeLimit: 10Gi  # Prevents disk exhaustion
```

### 3. Use Read-Only Where Possible

```yaml
volumeMounts:
  - name: media
    mountPath: /media
    readOnly: true  # Prevents accidental modifications
```

### 4. Node Affinity for HostPath

When using HostPath, ensure pods are scheduled to the correct node:

```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: storagenodet3500
```

### 5. Reclaim Policy

For production data, use `Retain` policy:

```yaml
apiVersion: v1
kind: PersistentVolume
spec:
  persistentVolumeReclaimPolicy: Retain
```

## Troubleshooting

### PVC Stuck in Pending

```bash
# Check events
kubectl describe pvc <pvc-name> -n <namespace>

# Verify StorageClass
kubectl get sc

# Check provisioner logs
kubectl logs -n local-path-storage -l app=local-path-provisioner
```

### Permission Denied

```bash
# Check container user
kubectl exec -n <namespace> <pod> -- id

# On node, fix permissions
sudo chown -R <uid>:<gid> /path/to/data
```

### Disk Full

```bash
# Check disk usage on node
df -h

# Find large files
sudo find /srv -type f -size +1G -exec ls -lh {} \;

# Clear old cache
sudo rm -rf /var/lib/jellyfin/cache/*
```

### Volume Not Mounting

```bash
# Check pod events
kubectl describe pod -n <namespace> <pod>

# Verify volume exists on node
ssh <node> 'ls -la /path/to/volume'

# Check mount points in container
kubectl exec -n <namespace> <pod> -- mount | grep <path>
```

## Capacity Planning

### Current Allocations

| Node | Path | Allocated | Used |
|------|------|-----------|------|
| storagenodet3500 | /srv/media | 500GB+ | Varies |
| storagenodet3500 | /var/lib/jellyfin | 5GB | ~1GB |
| masternode | /srv/monitoring_data | 32GB | ~5GB |

### Growth Projections

- **Media library**: Plan for ~10GB/month growth
- **Monitoring data**: ~500MB/month (with retention)
- **Application configs**: Minimal growth (<100MB/year)

### Expansion Options

1. **Add local storage**: Attach additional drives to nodes
2. **NFS**: Add network storage for shared access
3. **Distributed storage**: Consider Longhorn or Rook-Ceph

## References

- [Kubernetes Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Local Path Provisioner](https://github.com/rancher/local-path-provisioner)
- [Storage Best Practices](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#best-practices)
