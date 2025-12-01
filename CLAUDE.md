# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Helm library chart repository that provides reusable Kubernetes templates for web applications with GitOps best practices. The main chart is `common`, which is designed to be used as a dependency in other Helm charts.

## Key Architecture Concepts

### Template Structure and Deep Merging

The library uses a "deep merging" pattern implemented in `_helpers.yaml` via the `common.withMergedValues` helper. This is critical to understand:

- When this library is used as a dependency, values from the parent chart need to be merged with the library's default values
- The `common.withMergedValues` template checks if the chart has `Subcharts.common` and merges parent values with library defaults
- All standalone templates (those ending in `.standalone`) use this helper to ensure proper value inheritance
- **Important**: When modifying templates, always use the `.standalone` wrapper for any template that should support deep merging

### Template Naming Convention

Templates follow a dual-naming pattern:

- `common.<resource>.standalone` - Entry point that handles value merging (e.g., `common.deployment.standalone`)
- `common.<resource>` - Core template implementation (e.g., `common.deployment`)

Users can call either the standalone version (recommended) or the core template directly.

### The Web Bundle

The `common.web` template in `_web.yaml` is a convenience wrapper that includes all resources needed for a typical web application:

1. Deployment
2. Service
3. Ingress
4. External Secrets (application secrets)
5. Image Pull Secret (registry authentication)
6. Namespace

All resources are separated by `---` YAML document separators.

## Common Development Commands

### Testing Chart Locally

```bash
# Package the chart
helm package charts/common

# Test template rendering with a values file
helm template test-release charts/common -f charts/common/values.yaml

# Lint the chart
helm lint charts/common
```

### Release Process

The repository uses automated versioning and releasing via GitHub Actions:

- Every commit to `main` triggers an automatic release
- Version is auto-incremented (minor version bump)
- Charts are published to `oci://ghcr.io/starburst997/charts`
- All charts in the repo share the same version number (stored in `Chart.yaml`)
- The workflow uses custom GitHub Actions: `auto-version`, `docker-helm-push`, and `commits-logs`

**Important**: Do not manually modify the `version` field in `Chart.yaml` as it's managed by CI/CD.

## Key Value Fields

### Health Checks

The deployment template includes sophisticated health check handling:

- Health checks are enabled by default unless explicitly disabled with `healthCheck.enabled: false`
- Supports both custom probes (`readinessProbe`, `livenessProbe`) and auto-generated HTTP probes
- Auto-generated probes use configurable defaults with separate thresholds for readiness vs liveness
- Note: The liveness probe uses `mul $periodSeconds 2` for its period (double the readiness period)

### External Secrets Integration

The `_secret.yaml` template integrates with External Secrets Operator:

- Secrets can reference remote keys in two formats:
  - Simple key: `API_KEY: entry-name` → references entire entry
  - Path notation: `API_KEY: entry/property` → references specific property within entry
- The template parses the `/` separator to determine which format to use

### Image Pull Secrets

Separate from application secrets, uses External Secrets to dynamically fetch registry credentials from the secret store with a configurable refresh interval.

## Template Modification Guidelines

When adding or modifying templates:

1. Create both the core template and `.standalone` wrapper
2. Use `common.withMergedValues` in the standalone version
3. Follow existing naming conventions: `apiVersion`, `kind`, `metadata`, `spec` structure
4. Test that values from parent charts properly override library defaults

### Embedded Defaults Pattern (Critical)

**Important**: This is a Helm **library chart**. When used as a dependency, the `values.yaml` file from this library is NOT automatically merged with parent chart values. Helm only uses the parent chart's values.

Therefore, **all defaults MUST be embedded directly in the templates** using one of these patterns:

**Pattern 1: Simple defaults with `| default`**
```yaml
{{- $healthPath := (.Values.healthCheck).path | default "/" }}
{{- $healthPort := (.Values.healthCheck).port | default .Values.service.targetPort }}
```

**Pattern 2: Object defaults with `merge` and `dict`**
```yaml
{{- $rateLimitDefaults := dict "enabled" true "rpm" 600 "rps" 30 "connections" 50 -}}
{{- $rateLimit := merge (.Values.ingress.rateLimit | default dict) $rateLimitDefaults -}}
{{- if $rateLimit.enabled }}
nginx.ingress.kubernetes.io/limit-rpm: {{ $rateLimit.rpm | quote }}
{{- end }}
```

**Pattern 3: Boolean defaults with `| default true/false`**
```yaml
{{- $forceSSLRedirect := .Values.ingress.forceSSLRedirect | default true -}}
{{- if $forceSSLRedirect }}
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
{{- end }}
```

### values.yaml Purpose

The `charts/common/values.yaml` file serves as **documentation only**. It shows:
- All available configuration options
- The exact default values that are embedded in templates
- Brief descriptions of each field's purpose

**When updating defaults**: Always update BOTH the template (where the actual default is used) AND `values.yaml` (for documentation). They must match exactly.

## Dependencies and Requirements

This library assumes the following are installed in the Kubernetes cluster:

- External Secrets Operator (for secret management)
- cert-manager (for TLS certificate generation)
- nginx-ingress-controller (ingress uses nginx-specific annotations)
- Reloader (deployment annotations trigger auto-reload on secret changes)
