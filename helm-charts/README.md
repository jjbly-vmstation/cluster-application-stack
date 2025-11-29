# Helm Charts

This directory is reserved for future Helm chart deployments.

## Planned Charts

### Application Charts
- **jellyfin**: Helm chart for Jellyfin media server (future migration from Kustomize)
- **plex**: Alternative media server option
- **nextcloud**: File sync and share platform

### Supporting Charts
- **ingress-nginx**: Ingress controller for HTTP(S) routing
- **cert-manager**: Automatic TLS certificate management
- **external-dns**: Automatic DNS management

## Creating a New Chart

```bash
# Create a new chart
helm create charts/<chart-name>

# Validate chart
helm lint charts/<chart-name>

# Template chart (dry-run)
helm template <release-name> charts/<chart-name>

# Install chart
helm install <release-name> charts/<chart-name> -n <namespace>
```

## Chart Standards

When creating charts, follow these standards:

1. **Values**: Use sensible defaults in `values.yaml`
2. **Templates**: Include deployment, service, ingress, and configmap templates
3. **NOTES.txt**: Provide post-installation instructions
4. **README.md**: Document all configurable values
5. **Tests**: Include Helm tests for validation

## Migration from Kustomize

When migrating Kustomize resources to Helm:

1. Create chart structure
2. Extract values to `values.yaml`
3. Templatize Kubernetes manifests
4. Test with `helm template` and `helm install --dry-run`
5. Update documentation

## Resources

- [Helm Documentation](https://helm.sh/docs/)
- [Artifact Hub](https://artifacthub.io/) - Find existing charts
- [Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
