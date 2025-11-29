# Application Deployment Guide

This guide covers deploying applications to the VMStation Kubernetes cluster using the manifests and tooling in this repository.

## Overview

The cluster-application-stack repository provides:
- Kubernetes manifests for application workloads
- Kustomize configurations for environment-specific deployments
- Ansible playbooks for automated deployment
- Documentation for setup and operations

## Architecture

### Cluster Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                    VMStation Kubernetes Cluster                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────┐   ┌─────────────────────┐               │
│  │     masternode      │   │   storagenodet3500  │               │
│  │  (Control Plane)    │   │   (Storage Node)    │               │
│  │  192.168.4.63       │   │   192.168.4.61      │               │
│  │                     │   │                     │               │
│  │  - kube-apiserver   │   │  - Jellyfin         │               │
│  │  - kube-scheduler   │   │  - Media Storage    │               │
│  │  - kube-controller  │   │  - Future Apps      │               │
│  │  - etcd             │   │                     │               │
│  └─────────────────────┘   └─────────────────────┘               │
│                                                                   │
│  ┌─────────────────────┐                                         │
│  │      homelab        │                                         │
│  │   (Compute Node)    │                                         │
│  │   192.168.4.62      │                                         │
│  │                     │                                         │
│  │  - General Workloads│                                         │
│  │  - Future Apps      │                                         │
│  └─────────────────────┘                                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Node Roles

| Node | Role | Primary Use |
|------|------|-------------|
| masternode | Control Plane | Cluster management, monitoring |
| storagenodet3500 | Worker | Storage workloads, media apps |
| homelab | Worker | General compute workloads |

## Deployment Methods

### Method 1: Ansible (Recommended)

The most automated and consistent deployment method.

```bash
cd ansible

# Deploy all applications to production
ansible-playbook playbooks/deploy-applications.yml -e environment=production

# Deploy only Jellyfin
ansible-playbook playbooks/deploy-jellyfin.yml -e environment=production

# Deploy to staging
ansible-playbook playbooks/deploy-applications.yml -e environment=staging
```

### Method 2: Kustomize

Direct Kubernetes deployment using Kustomize overlays.

```bash
# Preview production deployment
kubectl kustomize kustomize/overlays/production

# Deploy to production
kubectl apply -k kustomize/overlays/production

# Deploy to staging
kubectl apply -k kustomize/overlays/staging
```

### Method 3: Direct Manifests

Apply individual manifests directly (useful for debugging).

```bash
# Apply Jellyfin manifests
kubectl apply -f manifests/jellyfin/

# Apply with dry-run
kubectl apply -f manifests/jellyfin/ --dry-run=client
```

## Environment Management

### Production vs Staging

| Aspect | Production | Staging |
|--------|------------|---------|
| Image Tags | Specific versions | `latest` |
| Resources | Full allocation | Reduced |
| Logging | Warning level | Debug level |
| Service Type | NodePort | ClusterIP |
| Node Selector | Storage node | Any |

### Switching Environments

```bash
# Production deployment
kubectl apply -k kustomize/overlays/production

# Staging deployment
kubectl apply -k kustomize/overlays/staging

# Verify current environment
kubectl get configmap -n jellyfin jellyfin-config -o yaml
```

## Adding New Applications

### Step 1: Create Manifests

```bash
mkdir -p manifests/<app-name>
```

Create the following files:
- `namespace.yaml` - Namespace definition
- `configmap.yaml` - Configuration data
- `deployment.yaml` - Workload definition
- `service.yaml` - Service exposure
- `kustomization.yaml` - Kustomize resource list

### Step 2: Add to Kustomize Base

Edit `kustomize/base/kustomization.yaml`:

```yaml
resources:
  - ../../manifests/common/namespace.yaml
  - ../../manifests/common/resource-quotas.yaml
  - ../../manifests/jellyfin
  - ../../manifests/<app-name>  # Add new app
```

### Step 3: Add Environment Overlays

Add patches to each overlay as needed:

```yaml
# kustomize/overlays/production/kustomization.yaml
patches:
  - target:
      kind: Deployment
      name: <app-name>
    patch: |-
      # Production-specific patches
```

### Step 4: Create Ansible Playbook (Optional)

Create `ansible/playbooks/deploy-<app-name>.yml` following the Jellyfin playbook pattern.

## Deployment Verification

### Check Application Status

```bash
# List all pods in all namespaces
kubectl get pods -A

# Check specific namespace
kubectl get all -n jellyfin

# Describe pod for details
kubectl describe pod -n jellyfin <pod-name>

# View logs
kubectl logs -n jellyfin -l app=jellyfin
```

### Verify Services

```bash
# List services
kubectl get svc -A

# Test service connectivity (from cluster)
kubectl run test --rm -i --tty --image=busybox -- wget -qO- http://jellyfin-service.jellyfin:8096/health
```

### Verify Storage

```bash
# Check PVCs
kubectl get pvc -A

# Check PV binding
kubectl get pv

# Verify storage on node
ssh storagenodet3500 'df -h /srv/media'
```

## Rollback Procedures

### Kubernetes Rollback

```bash
# View rollout history
kubectl rollout history deployment/<app-name> -n <namespace>

# Rollback to previous version
kubectl rollout undo deployment/<app-name> -n <namespace>

# Rollback to specific revision
kubectl rollout undo deployment/<app-name> -n <namespace> --to-revision=2
```

### Kustomize Rollback

```bash
# Revert to previous git commit
git log --oneline -5
git checkout <previous-commit> -- kustomize/overlays/production/

# Re-apply
kubectl apply -k kustomize/overlays/production
```

## Monitoring Deployments

### Watch Rollout Progress

```bash
# Watch deployment status
kubectl rollout status deployment/<app-name> -n <namespace>

# Watch pods
kubectl get pods -n <namespace> -w
```

### Check Events

```bash
# Namespace events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Cluster-wide events
kubectl get events -A --sort-by='.lastTimestamp' | head -20
```

## Best Practices

### Resource Management

1. Always define resource requests and limits
2. Use LimitRanges for default limits
3. Implement ResourceQuotas per namespace
4. Monitor resource usage

### Security

1. Run containers as non-root
2. Drop all capabilities by default
3. Use read-only filesystems where possible
4. Implement NetworkPolicies

### High Availability

1. Use Deployments with multiple replicas (when stateless)
2. Configure PodDisruptionBudgets
3. Use anti-affinity rules
4. Implement proper health checks

### Configuration

1. Externalize configuration with ConfigMaps
2. Use Secrets for sensitive data
3. Version configuration in git
4. Use environment-specific overlays

## Troubleshooting

### Common Issues

#### Pod ImagePullBackOff
- Verify image name and tag
- Check image registry accessibility
- Verify image pull secrets if needed

#### Pod CrashLoopBackOff
- Check logs: `kubectl logs -n <ns> <pod> --previous`
- Verify resource limits aren't too restrictive
- Check application configuration

#### PVC Pending
- Verify StorageClass exists
- Check node affinity matches available nodes
- Verify storage provisioner is running

### Debug Commands

```bash
# Run debug pod
kubectl run debug --rm -i --tty --image=busybox -- /bin/sh

# Execute into running container
kubectl exec -it -n <namespace> <pod> -- /bin/sh

# Port forward for local access
kubectl port-forward -n <namespace> svc/<service> 8080:8096
```

## References

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)
- [Ansible Kubernetes Collection](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/)
