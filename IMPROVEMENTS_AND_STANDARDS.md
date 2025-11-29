# Improvements and Standards

This document outlines the Kubernetes and application deployment best practices implemented in this repository, along with recommendations for future enhancements.

## Table of Contents

- [Improvements Implemented During Migration](#improvements-implemented-during-migration)
- [Kubernetes Best Practices](#kubernetes-best-practices)
- [Application Deployment Patterns](#application-deployment-patterns)
- [Kustomize Best Practices](#kustomize-best-practices)
- [Ansible Best Practices](#ansible-best-practices)
- [Security Standards](#security-standards)
- [Recommended Future Improvements](#recommended-future-improvements)

---

## Improvements Implemented During Migration

The following improvements were made during the migration from the VMStation monorepo:

### ✅ Kustomize Structure

- **Base configuration**: Environment-agnostic base in `kustomize/base/`
- **Overlay pattern**: Production and staging overlays
- **Resource organization**: Separate files for each resource type
- **Image management**: Centralized image tag management in overlays

### ✅ Resource Management

```yaml
# All deployments include resource requests and limits
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "2"
    memory: 2Gi
```

- LimitRanges for namespace-level defaults
- ResourceQuotas to prevent resource exhaustion

### ✅ Health Checks

```yaml
# Comprehensive probe configuration
startupProbe:
  httpGet:
    path: /health
    port: 8096
  initialDelaySeconds: 180
  periodSeconds: 15
  failureThreshold: 40

readinessProbe:
  httpGet:
    path: /health
    port: 8096
  initialDelaySeconds: 240
  periodSeconds: 30
  failureThreshold: 10

livenessProbe:
  httpGet:
    path: /health
    port: 8096
  initialDelaySeconds: 300
  periodSeconds: 60
  failureThreshold: 5
```

### ✅ Security Context

```yaml
# Non-root container with dropped capabilities
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  
containers:
  - securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: false
      capabilities:
        drop:
          - ALL
```

### ✅ Deployment from Pod

Changed Jellyfin from a standalone Pod to a Deployment for:
- Rollout management
- Self-healing capabilities
- Declarative update strategy

### ✅ ConfigMap Externalization

- Environment variables moved to ConfigMaps
- Easy environment-specific overrides via Kustomize patches

### ✅ DNS Configuration

```yaml
dnsPolicy: ClusterFirst
dnsConfig:
  options:
    - name: ndots
      value: "2"
    - name: edns0
```

### ✅ Tolerations for Network Issues

```yaml
tolerations:
  - key: "node.kubernetes.io/network-unavailable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300
```

### ✅ Comprehensive Labeling

```yaml
labels:
  app: jellyfin
  app.kubernetes.io/name: jellyfin
  app.kubernetes.io/component: media-server
  app.kubernetes.io/part-of: vmstation
  vmstation.io/component: media
```

---

## Kubernetes Best Practices

### Resource Management

| Practice | Status | Notes |
|----------|--------|-------|
| Define resource requests | ✅ | All containers have requests |
| Define resource limits | ✅ | All containers have limits |
| Use LimitRanges | ✅ | Default limits for namespace |
| Use ResourceQuotas | ✅ | Namespace-level quotas |
| Horizontal Pod Autoscaling | 📋 | Future: Add HPA for stateless apps |
| Vertical Pod Autoscaler | 📋 | Future: VPA recommendations |

### Pod Configuration

| Practice | Status | Notes |
|----------|--------|-------|
| Run as non-root | ✅ | runAsUser: 1000 |
| Drop all capabilities | ✅ | capabilities.drop: [ALL] |
| Read-only filesystem | ⚠️ | Not for Jellyfin (needs writes) |
| Use startup probes | ✅ | Extended startup time allowed |
| Use readiness probes | ✅ | Service traffic control |
| Use liveness probes | ✅ | Auto-restart on failure |

### Labels and Annotations

```yaml
# Recommended label schema
labels:
  # Standard Kubernetes labels
  app.kubernetes.io/name: application-name
  app.kubernetes.io/instance: instance-id
  app.kubernetes.io/version: "1.0.0"
  app.kubernetes.io/component: component-type
  app.kubernetes.io/part-of: project-name
  app.kubernetes.io/managed-by: kustomize
  
  # Custom labels for VMStation
  vmstation.io/component: category
  vmstation.io/environment: production
```

---

## Application Deployment Patterns

### Deployment Strategy

For stateful applications like Jellyfin:

```yaml
spec:
  replicas: 1
  strategy:
    type: Recreate  # Ensures data consistency
```

For stateless applications:

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

### Storage Patterns

| Pattern | Use Case | Example |
|---------|----------|---------|
| PVC | Application config | jellyfin-config-pvc |
| EmptyDir | Cache/temp data | jellyfin-cache |
| HostPath | Large media files | /srv/media |

### Service Types

| Environment | Service Type | External Access |
|-------------|--------------|-----------------|
| Production | NodePort | Direct node IP access |
| Staging | ClusterIP | Port-forward or Ingress |
| Public | LoadBalancer | External IP |

---

## Kustomize Best Practices

### Base Configuration

```yaml
# base/kustomization.yaml - Keep environment-agnostic
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../manifests/common/
  - ../../manifests/jellyfin/

# No environment-specific values in base
```

### Overlay Configuration

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

# Environment-specific modifications
commonLabels:
  environment: production

patches:
  - target:
      kind: ConfigMap
      name: app-config
    patch: |-
      - op: replace
        path: /data/LOG_LEVEL
        value: "Warning"

images:
  - name: myapp
    newTag: "1.2.3"  # Specific version, not :latest
```

### Patch Types

| Type | Use Case |
|------|----------|
| JSON Patch | Precise operations (add, replace, remove) |
| Strategic Merge | Complex nested modifications |
| ConfigMap Generator | Dynamic config generation |

---

## Ansible Best Practices

### Playbook Structure

```yaml
---
- name: "Deploy Application"
  hosts: control_plane
  become: true
  gather_facts: true
  
  vars:
    environment: production
    
  tasks:
    - name: "Validation"
      block:
        - name: "Check prerequisites"
          # Pre-flight checks
          
    - name: "Deployment"
      block:
        - name: "Apply manifests"
          kubernetes.core.k8s:
            state: present
            src: "{{ manifest_path }}"
            wait: true
            
    - name: "Verification"
      block:
        - name: "Check deployment status"
          # Post-deployment validation
```

### Idempotency

```yaml
# Use kubernetes.core.k8s module with state management
- name: "Apply manifest"
  kubernetes.core.k8s:
    state: present
    src: manifest.yaml
    wait: true
    apply: true  # Enables server-side apply
```

### Wait Conditions

```yaml
- name: "Wait for deployment"
  kubernetes.core.k8s_info:
    api_version: apps/v1
    kind: Deployment
    name: myapp
    namespace: myns
  register: deployment
  until: >
    deployment.resources | length > 0 and
    deployment.resources[0].status.readyReplicas is defined and
    deployment.resources[0].status.readyReplicas >= 1
  retries: 20
  delay: 30
```

---

## Security Standards

### Container Security

| Standard | Implementation |
|----------|----------------|
| Non-root user | runAsUser: 1000 |
| No privilege escalation | allowPrivilegeEscalation: false |
| Minimal capabilities | drop: [ALL] |
| Read-only root (where possible) | readOnlyRootFilesystem: true |

### Pod Security Standards

```yaml
# Namespace-level enforcement
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Network Security (Future)

```yaml
# NetworkPolicy example
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: jellyfin-network-policy
  namespace: jellyfin
spec:
  podSelector:
    matchLabels:
      app: jellyfin
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - port: 8096
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - port: 443
        - port: 80
```

---

## Recommended Future Improvements

### High Priority

| Improvement | Description | Benefit |
|-------------|-------------|---------|
| **NetworkPolicies** | Restrict pod-to-pod communication | Security |
| **Pod Disruption Budgets** | Ensure availability during maintenance | Reliability |
| **GitOps (ArgoCD/Flux)** | Automated sync from Git | Automation |
| **External Secrets Operator** | Secure secrets management | Security |

### Medium Priority

| Improvement | Description | Benefit |
|-------------|-------------|---------|
| **Helm Charts** | Package applications for distribution | Portability |
| **Service Mesh (Istio/Linkerd)** | mTLS, observability, traffic management | Security/Observability |
| **Prometheus Monitoring** | Application metrics collection | Observability |
| **Centralized Logging** | Log aggregation with Loki | Troubleshooting |

### Low Priority / Nice to Have

| Improvement | Description | Benefit |
|-------------|-------------|---------|
| **Canary Deployments** | Gradual rollouts with Flagger | Risk reduction |
| **Chaos Engineering** | Fault injection testing | Reliability |
| **Cost Optimization** | Resource right-sizing | Efficiency |
| **Backup Automation** | Velero for cluster backups | Disaster recovery |

### Implementation Roadmap

```
Phase 1 (Current)
  ✅ Kustomize structure
  ✅ Resource management
  ✅ Health checks
  ✅ Security contexts
  ✅ Documentation

Phase 2 (Next Quarter)
  📋 NetworkPolicies
  📋 PodDisruptionBudgets
  📋 External Secrets Operator
  📋 Prometheus metrics

Phase 3 (Future)
  📋 GitOps with ArgoCD
  📋 Helm charts
  📋 Service mesh evaluation
  📋 Automated backups
```

---

## Validation Checklist

Before deploying any application, verify:

- [ ] All manifests pass `kubectl apply --dry-run=client`
- [ ] Kustomize builds without errors: `kubectl kustomize overlays/production`
- [ ] Resource requests and limits are defined
- [ ] Health probes are configured
- [ ] Security context uses non-root user
- [ ] Labels follow standard schema
- [ ] ConfigMaps/Secrets are externalized
- [ ] Documentation is updated
- [ ] Ansible playbook is idempotent

---

## References

- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)
- [Ansible Kubernetes Collection](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/)
- [12 Factor App](https://12factor.net/)
