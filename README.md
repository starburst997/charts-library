# Charts Library

A collection of reusable Helm library charts for Kubernetes deployments with GitOps best practices.

## Charts

### Common

A Helm library chart providing reusable templates for common Kubernetes resources.

**Features:**

- 🚀 Standard deployment patterns
- 🔐 External Secrets integration
- 🐳 Private registry support (imagePullSecrets)
- 🌐 Ingress with nginx annotations
- 📦 Service and namespace management
- 🎯 Web application bundle template

## Installation

### As a Dependency

Add to your `Chart.yaml`:

```yaml
dependencies:
  - name: common
    version: ">=0.0.0"
    repository: "oci://ghcr.io/starburst997/charts"
```

Update dependencies:

```bash
helm dependency update
```

## Usage

### Web Application Bundle

Use the complete web bundle (deployment, service, ingress, secrets):

```yaml
# templates/common.yaml
{{ include "common.web" . }}
```

### Individual Templates

Or use individual components:

```yaml
# templates/deployment.yaml
{{ include "common.deployment" . }}

# templates/service.yaml
{{ include "common.service" . }}

# templates/ingress.yaml
{{ include "common.ingress" . }}

# templates/secret.yaml
{{ include "common.externalSecret" . }}

# templates/image-pull-secret.yaml
{{ include "common.imagePullSecret" . }}

# templates/namespace.yaml
{{ include "common.namespace" . }}
```

## Required Values

Your `values.yaml` must include:

```yaml
namespace: my-app
replicaCount: 1

image:
  repository: ghcr.io/myorg/myapp
  pullPolicy: IfNotPresent
  tag: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: nginx
  clusterIssuer: letsencrypt-http
  host: myapp.example.com
  proxyBodySize: "200m" # optional
  timeout: "3600" # optional
  keepAlive: "3600" # optional

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

env:
  ENV: "production"
  API_URL: "https://api.example.com"

secretStore:
  name: onepassword
  kind: ClusterSecretStore

secrets:
  API_KEY: entry/api-key
  DATABASE_PASSWORD: entry/database-password

imagePullSecret:
  create: true
  name: ghcr-auth
  registry: ghcr.io
  usernameKey: registry/GH_USERNAME
  tokenKey: registry/GH_REGISTRY_PAT
  refreshInterval: 6h
```

## Example Application Chart

```yaml
# Chart.yaml
apiVersion: v2
name: my-app
version: 0.0.0
appVersion: "0.0.0"
dependencies:
  - name: common
    version: ">=0.0.0"
    repository: "oci://ghcr.io/starburst997/charts"

# templates/common.yaml
{{ include "common.web" . }}

# values.yaml
namespace: my-app
replicaCount: 1
image:
  repository: ghcr.io/myorg/my-app
  tag: "1.0.0"
# ... rest of required values
```

## Development

### Publishing Charts

Charts are automatically published to `ghcr.io/starburst997/charts` on every commit to `main` via GitHub Actions.

### Manual Publishing

```bash
# Package chart
helm package charts/common

# Push to registry
helm push common-0.1.0.tgz oci://ghcr.io/starburst997/charts
```

### Versioning

- Charts use semantic versioning
- Version is auto-incremented (minor) on every commit to main
- All charts in this repo share the same version number

## Available Templates

| Template                 | Description                             |
| ------------------------ | --------------------------------------- |
| `common.web`             | Complete web app bundle (all resources) |
| `common.deployment`      | Kubernetes Deployment                   |
| `common.service`         | Kubernetes Service                      |
| `common.ingress`         | Kubernetes Ingress with nginx support   |
| `common.namespace`       | Kubernetes Namespace                    |
| `common.externalSecret`  | External Secrets for app secrets        |
| `common.imagePullSecret` | External Secrets for registry auth      |

## Requirements

- Helm 3.8+
- Kubernetes 1.19+
- External Secrets Operator (for secret management)
- cert-manager (for TLS certificates)
- nginx-ingress-controller (for ingress)

## License

MIT

## Maintainers

- [@starburst997](https://github.com/starburst997)
