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
- 🛡️ **Rate limiting** - Built-in DDoS protection (enabled by default)
- 🔒 **Security headers** - HSTS, XSS protection, clickjacking prevention (enabled by default)
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

## Rate Limiting

Rate limiting is **enabled by default** to protect your applications from abuse and DDoS attacks. It limits requests per IP address using nginx ingress annotations.

**Default configuration (permissive to avoid false positives):**

- 30 requests per second per IP (allows page loads with many assets)
- 600 requests per minute per IP (10/sec average)
- 50 concurrent connections per IP (handles browser parallel requests)
- Burst multiplier of 10x (allows spiky traffic patterns)
- Internal networks whitelisted by default (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`)
- Returns HTTP 503 when limit is exceeded

**Configuration options:**

```yaml
ingress:
  rateLimit:
    enabled: true # Toggle rate limiting (default: true)
    rpm: 600 # Requests per minute per IP
    rps: 30 # Requests per second per IP
    connections: 50 # Concurrent connections per IP
    burstMultiplier: 10 # Burst size multiplier (default: 10)
    whitelist: "10.0.0.0/8,172.16.0.0/12,192.168.0.0/16" # CIDRs excluded
```

**To disable rate limiting:**

```yaml
ingress:
  rateLimit:
    enabled: false
```

**Important notes:**

- Rate limits are applied **per nginx ingress controller replica**. If you run 3 replicas, effective limit is 3x the configured value.
- Internal Kubernetes networks are whitelisted by default to allow pod-to-pod communication.
- Monitor your traffic patterns to tune these values appropriately.

## Security Headers

Security headers are **enabled by default** to protect against common web vulnerabilities including XSS, clickjacking, and MIME sniffing attacks.

**Default headers applied:**

| Header | Default Value | Protection |
|--------|---------------|------------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` | Forces HTTPS for 1 year |
| `X-Frame-Options` | `SAMEORIGIN` | Prevents clickjacking |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME sniffing |
| `X-XSS-Protection` | `1; mode=block` | Enables browser XSS filter |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Controls referrer information |
| `Permissions-Policy` | `geolocation=(), microphone=(), camera=()` | Disables sensitive browser features |

**Configuration options:**

```yaml
ingress:
  securityHeaders:
    enabled: true # Toggle security headers (default: true)
    hsts: "max-age=31536000; includeSubDomains; preload"
    frameOptions: "SAMEORIGIN" # or "DENY" for stricter
    contentTypeOptions: "nosniff"
    xssProtection: "1; mode=block"
    referrerPolicy: "strict-origin-when-cross-origin"
    permissionsPolicy: "geolocation=(), microphone=(), camera=()"
  forceSSLRedirect: true # Forces HTTPS redirect (default: true)
  sslRedirect: true # SSL redirect (default: true)
```

**To disable security headers:**

```yaml
ingress:
  securityHeaders:
    enabled: false
```

**Note:** Security headers use nginx `configuration-snippet` annotation. Ensure your nginx ingress controller has `allow-snippet-annotations: "true"` in its ConfigMap (enabled by default in most installations).

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
  rateLimit:
    enabled: true # Rate limiting enabled by default
    rpm: 600 # Requests per minute per IP
    rps: 30 # Requests per second per IP
    connections: 50 # Concurrent connections per IP
    burstMultiplier: 10 # Burst multiplier (default 10)
    whitelist: "10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
  securityHeaders:
    enabled: true # Security headers enabled by default
    hsts: "max-age=31536000; includeSubDomains; preload"
    frameOptions: "SAMEORIGIN"
    contentTypeOptions: "nosniff"
    xssProtection: "1; mode=block"
    referrerPolicy: "strict-origin-when-cross-origin"
    permissionsPolicy: "geolocation=(), microphone=(), camera=()"
  forceSSLRedirect: true
  sslRedirect: true

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

## Complete Rendered Example

Here's what `{{ include "common.web" . }}` generates for a real application:

**Input values.yaml:**

```yaml
namespace: s3-mirror-sample
replicaCount: 1

image:
  repository: ghcr.io/starburst997/s3-mirror-sample-app

service:
  targetPort: 3000

healthCheck:
  port: 3000

resources:
  requests:
    cpu: "10m"
    memory: "32Mi"
  limits:
    cpu: "250m"
    memory: "128Mi"

ingress:
  host: s3-mirror-sample.jd.boiv.in
  proxyBodySize: "250m"

env:
  ENV: "production"
  S3_ENDPOINT: "https://xxxxxxxxx.r2.cloudflarestorage.com"
  S3_BUCKET: "s3-mirror"

secrets:
  AWS_ACCESS_KEY_ID: s3-mirror-sample-app/AWS_ACCESS_KEY_ID
  AWS_SECRET_ACCESS_KEY: s3-mirror-sample-app/AWS_SECRET_ACCESS_KEY
```

**Rendered output (helm template):**

<details>
<summary>Click to expand the full rendered YAML (200+ lines)</summary>

```yaml
---
# Source: s3-mirror-sample-app/templates/common.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: s3-mirror-sample
---
# Source: s3-mirror-sample-app/templates/common.yaml
apiVersion: v1
kind: Service
metadata:
  name: s3-mirror-sample-app
  namespace: s3-mirror-sample
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 3000
      protocol: TCP
  selector:
    app: s3-mirror-sample-app
---
# Source: s3-mirror-sample-app/templates/common.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: s3-mirror-sample-app
  namespace: s3-mirror-sample
spec:
  replicas: 1
  selector:
    matchLabels:
      app: s3-mirror-sample-app
  template:
    metadata:
      annotations:
        reloader.stakater.com/search: "true" # Auto-reload on ConfigMap/Secret changes
      labels:
        app: s3-mirror-sample-app
    spec:
      imagePullSecrets:
        - name: ghcr-auth
      containers:
        - name: app
          image: "ghcr.io/starburst997/s3-mirror-sample-app:1.0.0"
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
          env:
            - name: ENV
              value: "production"
            - name: S3_BUCKET
              value: "s3-mirror"
            - name: S3_ENDPOINT
              value: "https://xxxxxxxxxx.r2.cloudflarestorage.com"
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: s3-mirror-sample-app
                  key: AWS_ACCESS_KEY_ID
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: s3-mirror-sample-app
                  key: AWS_SECRET_ACCESS_KEY
          resources:
            limits:
              cpu: 250m
              memory: 128Mi
            requests:
              cpu: 10m
              memory: 32Mi
          readinessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            successThreshold: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
---
# Source: s3-mirror-sample-app/templates/common.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: s3-mirror-sample-app
  namespace: s3-mirror-sample
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-http"
    nginx.ingress.kubernetes.io/proxy-body-size: "250m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/keep-alive: "3600"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - s3-mirror-sample.jd.boiv.in
      secretName: s3-mirror-sample-app-tls
  rules:
    - host: s3-mirror-sample.jd.boiv.in
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: s3-mirror-sample-app
                port:
                  number: 80
---
# Source: s3-mirror-sample-app/templates/common.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  annotations:
    "helm.sh/hook": pre-install, pre-upgrade
    "helm.sh/hook-weight": "-5"
  name: s3-mirror-sample-app
  namespace: s3-mirror-sample
spec:
  refreshInterval: "6h"
  secretStoreRef:
    name: onepassword
    kind: ClusterSecretStore
  target:
    name: s3-mirror-sample-app
    creationPolicy: Owner
  data:
    - secretKey: AWS_ACCESS_KEY_ID
      remoteRef:
        key: s3-mirror-sample-app
        property: AWS_ACCESS_KEY_ID
    - secretKey: AWS_SECRET_ACCESS_KEY
      remoteRef:
        key: s3-mirror-sample-app
        property: AWS_SECRET_ACCESS_KEY
---
# Source: s3-mirror-sample-app/templates/common.yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  annotations:
    "helm.sh/hook": pre-install, pre-upgrade
    "helm.sh/hook-weight": "-10"
  name: ghcr-auth
  namespace: s3-mirror-sample
spec:
  refreshInterval: "6h"
  secretStoreRef:
    name: onepassword
    kind: ClusterSecretStore
  target:
    name: ghcr-auth
    creationPolicy: Owner
    template:
      type: kubernetes.io/dockerconfigjson
      data:
        .dockerconfigjson: |
          {
            "auths": {
              "ghcr.io": {
                "username": "{{ .username }}",
                "password": "{{ .token }}"
              }
            }
          }
  data:
    - secretKey: username
      remoteRef:
        key: registry
        property: GH_USERNAME
    - secretKey: token
      remoteRef:
        key: registry
        property: GH_REGISTRY_PAT
```

</details>

**Key things to note:**

- Single line `{{ include "common.web" . }}` generates 6 complete Kubernetes resources
- Deep merging in action: `className: nginx`, `clusterIssuer: letsencrypt-http`, and default timeouts from library are merged with the custom `host` and `proxyBodySize`
- Health checks automatically configured with library defaults
- Secrets are split into two resources: application secrets and image pull secrets with different hook weights
- Secret path notation `s3-mirror-sample-app/AWS_ACCESS_KEY_ID` parsed into `key` and `property` for External Secrets

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
