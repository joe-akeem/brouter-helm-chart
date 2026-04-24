# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Helm chart for deploying [BRouter](https://github.com/abrensch/brouter) (a bicycle routing engine) to Kubernetes. The chart uses the `joeakeem/brouter` container image and is published via GitHub Pages using [chart-releaser](https://github.com/helm/chart-releaser-action).

## Common commands

```bash
# Lint the chart
helm lint ./brouter

# Dry-run install to inspect rendered manifests
helm install brouter ./brouter --dry-run --debug

# Install into a cluster
helm install brouter ./brouter -n brouter --create-namespace

# Run Helm tests against a deployed release
helm test brouter -n brouter

# Template with custom values file
helm template brouter ./brouter -f my-values.yaml
```

## Releasing a new version

1. Update `version` in `brouter/Chart.yaml` (chart version) and `appVersion` (BRouter upstream version)
2. Push to `main` — the GitHub Actions workflow (`.github/workflows/release-helm-chart.yaml`) packages and publishes the chart to the `gh-pages` branch automatically

## Architecture

The chart lives entirely under `brouter/`. Templates follow standard Helm conventions:

- `_helpers.tpl` — shared name/label helpers used by all other templates
- `deployment.yaml` — the main workload; conditionally omits `replicas` when HPA is enabled
- `ingress.yaml` / `httproute.yaml` — mutually optional networking layers (classic Ingress vs Gateway API)
- `hpa.yaml`, `serviceaccount.yaml` — conditionally rendered based on `enabled` flags

### BRouter-specific design decisions

BRouter has no HTTP health endpoint, so probes use TCP socket checks on port 17777.

The chart ships one init container and one ConfigMap for routing data:
- The `download-segments` init container downloads an `.rd5` routing segment (default: `E5_N45.rd5`) into the `segments4` emptyDir volume at pod startup
- Routing profiles (`.brf` files + `lookups.dat`) are bundled directly in the chart under `brouter/profiles2/` and shipped as a ConfigMap mounted read-only at `/profiles2`; no network access is needed for profiles at pod startup

The segment URL is configurable in `values.yaml` under `initContainers`. Profile files are versioned with the chart in `brouter/profiles2/`.

### Networking

- The service always exposes port **17777**
- `ingress.enabled` and `httpRoute.enabled` are mutually exclusive in practice (both can be set but that's unusual)
- `httpRoute` targets the Gateway API (`gateway.networking.k8s.io/v1`) and requires a compatible controller