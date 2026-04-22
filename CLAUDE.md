# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Helm chart for deploying [BRouter](https://github.com/abrensch/brouter) (a bicycle routing engine) to Kubernetes. The chart uses the `joeakeem/brouter` container image and is published via GitHub Pages using [chart-releaser](https://github.com/helm/chart-releaser-action).

## Related projects

This chart is used as a subchart in two sibling projects:
- [singletrailmap-helm-charts](https://github.com/joe-akeem/singletrailmap-helm-charts) — umbrella chart that composes brouter with tileserver-gl, brouter-web, and planetiler; overrides initContainers and volumes for production use
- [singletrailmap-gitops](https://github.com/joe-akeem/singletrailmap-gitops) — ArgoCD GitOps repo that deploys the umbrella chart to production with environment-specific values (NFS storage, hostnames, secrets)

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

The chart ships two init containers that download routing data at pod startup:
- One downloads an `.rd5` routing segment (default: `E5_N45.rd5`) into the `segments4` volume
- One downloads routing profiles from the BRouter GitHub repo into the `profiles2` volume

Both volumes are `emptyDir`, so data is re-downloaded on every pod restart. The list of profiles and segment URL are configurable in `values.yaml` under `initContainers`.

### Networking

- The service always exposes port **17777**
- `ingress.enabled` and `httpRoute.enabled` are mutually exclusive in practice (both can be set but that's unusual)
- `httpRoute` targets the Gateway API (`gateway.networking.k8s.io/v1`) and requires a compatible controller