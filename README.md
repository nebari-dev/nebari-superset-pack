# nebari-superset

A [Nebari](https://nebari.dev) Software Pack for [Apache Superset](https://superset.apache.org/).

Wraps the upstream Apache Superset Helm chart with:
- **NebariApp CRD** for routing, TLS, and gateway authentication on Nebari clusters
- **Keycloak OAuth** integration with role-based access control
- **Standalone mode** for non-Nebari Kubernetes clusters

## Quick Start

### On a Nebari Cluster

```bash
helm repo add nebari-superset https://nebari-dev.github.io/helm-repository
helm repo update

# Label the namespace
kubectl label namespace superset nebari.dev/managed=true --overwrite

# Deploy
helm upgrade --install superset nebari-superset/nebari-superset \
  -f examples/nebari-values.yaml \
  -n superset --create-namespace
```

See [examples/nebari-values.yaml](examples/nebari-values.yaml) for configuration details.

### Standalone (non-Nebari)

```bash
helm repo add nebari-superset https://nebari-dev.github.io/helm-repository
helm repo update

helm upgrade --install superset nebari-superset/nebari-superset \
  -f examples/standalone-values.yaml \
  -n superset --create-namespace
```

See [examples/standalone-values.yaml](examples/standalone-values.yaml) for configuration details.

### ArgoCD

See [examples/argocd-app.yaml](examples/argocd-app.yaml) for an ArgoCD Application example.

## Configuration

### NebariApp CRD

When `nebariapp.enabled: true`, the chart creates a NebariApp custom resource that the nebari-operator reconciles into:
- HTTPRoute for traffic routing
- TLS certificate
- (Optional) Keycloak OIDC client and Envoy SecurityPolicy

| Value | Description | Default |
|-------|-------------|---------|
| `nebariapp.enabled` | Create NebariApp resource | `false` |
| `nebariapp.hostname` | FQDN for the application | `""` (required when enabled) |
| `nebariapp.routing.tls.enabled` | Provision TLS certificate and HTTPS listener | `true` (when routing is set) |
| `nebariapp.routing.routes` | Path-based routing rules | `[]` |
| `nebariapp.auth.enabled` | Enable gateway auth | `false` |
| `nebariapp.auth.provisionClient` | Auto-provision Keycloak client | `true` |
| `nebariapp.auth.scopes` | OAuth scopes | `[openid, profile, email]` |
| `nebariapp.auth.groups` | Restrict to groups | `[]` |
| `nebariapp.gateway` | Gateway type | `public` |

**Important:** The `routing` section must be included for the operator to create HTTPRoutes and TLS certificates. Without it, no route or certificate will be provisioned.

### OAuth / Keycloak

OAuth is configured via `superset.configOverrides.oauth_config` in your values file. See [examples/nebari-values.yaml](examples/nebari-values.yaml) for a complete Keycloak configuration including:
- Custom SecurityManager for role mapping
- OAuth provider configuration
- Role mapping (Keycloak roles to Superset roles)

On Nebari clusters, the nebari-operator creates an OIDC client secret (`<release>-nebari-superset-oidc-client`) containing:
- `client-id` - the OIDC client ID
- `client-secret` - the client secret
- `issuer-url` - the Keycloak issuer URL

Reference this secret via `superset.extraEnvRaw` to inject credentials into the OAuth config.

### Upstream Superset

All values under `superset.*` pass through to the [upstream Apache Superset Helm chart](https://github.com/apache/superset/tree/master/helm/superset). See the upstream chart's [values.yaml](https://github.com/apache/superset/blob/master/helm/superset/values.yaml) for all available options.

## Local Development

```bash
cd dev

# Full Nebari stack
make up

# Standalone mode
make up-standalone

# Tear down
make down
```

See [dev/Makefile](dev/Makefile) for all available targets.

## License

Apache License 2.0 - see [LICENSE](LICENSE).
