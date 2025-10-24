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
- 🔄 **Deep value merging** - Override nested values without losing defaults

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

## Deep Value Merging

This library solves Helm's shallow value inheritance limitation. When you override a nested object in your app's `values.yaml`, standard Helm dependency behavior replaces the entire object, losing all default values from the library chart.

**The Problem:**

Library chart defaults (`charts/common/values.yaml`):

```yaml
ingress:
  enabled: true
  className: nginx
  clusterIssuer: letsencrypt-http
  host: example.com
```

Your app values (`values.yaml`) - you only want to change the host:

```yaml
ingress:
  host: myapp.com
```

**Standard Helm behavior:** The `ingress` object is completely replaced. You lose `enabled`, `className`, and `clusterIssuer` from defaults, breaking your deployment.

**With deep merging (this library):** All fields are preserved. You get `enabled: true`, `className: nginx`, `clusterIssuer: letsencrypt-http` from defaults, plus your override `host: myapp.com`.

**How it works:**

Each template has a `.standalone` wrapper that uses the `common.withMergedValues` helper to merge parent values with library defaults before rendering:

```yaml
{{- define "common.ingress.standalone" -}}
{{- include "common.withMergedValues" (dict "templateName" "common.ingress" "context" .) -}}
{{- end -}}
```

When you call `{{ include "common.web" . }}` from your app's template, it automatically detects the parent chart context, merges your values with the library's defaults, and passes the complete merged values to all templates. No configuration needed.

Implementation: [`charts/common/templates/_helpers.yaml`](charts/common/templates/_helpers.yaml)

## Usage

### Web Application Bundle

Use the complete web bundle (deployment, service, ingress, secrets):

```yaml
# templates/common.yaml
{ { include "common.web" . } }
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

## Values

Define only what you need to override. All fields have defaults in the library chart ([`charts/common/values.yaml`](charts/common/values.yaml)).

**Minimal example:**

```yaml
namespace: my-app
image:
  repository: ghcr.io/myorg/myapp
ingress:
  host: myapp.example.com
env:
  ENV: production
secrets:
  API_KEY: entry/api-key
```

**Complete example with all available fields:**

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
