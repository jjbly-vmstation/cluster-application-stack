# Kustomize Configuration

This directory contains Kustomize configurations for deploying VMStation applications with environment-specific overlays.

## Structure

```
kustomize/
├── base/                    # Base configuration
│   └── kustomization.yaml   # References manifests from ../manifests/
└── overlays/
    ├── production/          # Production environment overlay
    │   └── kustomization.yaml
    └── staging/             # Staging environment overlay
        └── kustomization.yaml
```

The base configuration references the canonical manifests in `manifests/` directory,
avoiding duplication and ensuring a single source of truth.

## Usage

### Preview Changes

```bash
# Preview production configuration
kubectl kustomize kustomize/overlays/production

# Preview staging configuration
kubectl kustomize kustomize/overlays/staging
```

### Deploy

```bash
# Deploy to production
kubectl apply -k kustomize/overlays/production

# Deploy to staging
kubectl apply -k kustomize/overlays/staging
```

### Using with Ansible

```bash
# Deploy via Ansible playbook
ansible-playbook ansible/playbooks/deploy-applications.yml -e deploy_environment=production
```

## Overlays

### Production

- Uses specific image versions (not `:latest`)
- NodePort service for external access
- NodeSelector for storage node placement
- Reduced log verbosity

### Staging

- Uses `:latest` image tags
- Reduced resource limits
- Debug-level logging
- ClusterIP service type

## Customization

### Adding New Applications

1. Create manifests in `manifests/<app-name>/`
2. Add `kustomization.yaml` to the manifest directory
3. Include the resource in `base/kustomization.yaml`
4. Add overlay patches in `overlays/*/kustomization.yaml`

### Modifying Resources

Use patches in overlays rather than modifying base configurations:

```yaml
# Example: Strategic merge patch
patches:
  - target:
      kind: Deployment
      name: my-app
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
```

## Best Practices

1. **Keep base environment-agnostic**: Base should work anywhere
2. **Use patches for environment changes**: Don't duplicate entire files
3. **Version images in production**: Never use `:latest` in production
4. **Use generators for ConfigMaps/Secrets**: Leverage Kustomize generators
5. **Test with dry-run**: Always preview changes before applying

## Troubleshooting

### Common Issues

**Build Error**: Check that all paths in `kustomization.yaml` are correct relative paths.

```bash
# Validate kustomization
kubectl kustomize kustomize/base --enable-helm 2>&1 | head -20
```

**Missing Resources**: Ensure all required manifests are listed in `resources:`.

**Patch Failures**: Verify target resource names match exactly.

## References

- [Kustomize Documentation](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)
- [Kustomize Examples](https://github.com/kubernetes-sigs/kustomize/tree/master/examples)
