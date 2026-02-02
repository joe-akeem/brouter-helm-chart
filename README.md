# brouter-helm-chart

A Helm chart for deploying [BRouter](https://github.com/abrensch/brouter) to Kubernetes.

This chart packages the official `joeakeem/brouter` container image and exposes it via a `Service`, with optional `Ingress` or Gateway API `HTTPRoute`.

## Prerequisites
- Kubernetes 1.25+
- Helm 3.8+
- (Optional) Ingress controller if you enable `ingress.enabled`
- (Optional) Gateway API CRDs and a controller if you enable `httpRoute.enabled`

## Quick start

### 1) Add the Helm repo
The chart is published to GitHub Pages by the automated workflow.

Replace `<OWNER>` with your GitHub username or org name if you are using this repository as-is.

```bash
helm repo add brouter https://<OWNER>.github.io/brouter-helm-chart
helm repo update
```

If you are testing locally without a repo, you can also install from the chart directory:

```bash
helm install brouter ./brouter --namespace brouter --create-namespace
```

### 2) Install

Install with defaults (ClusterIP service, no ingress):

```bash
helm install brouter brouter/brouter \
  --namespace brouter \
  --create-namespace
```

Specify an image tag (defaults to the chart `appVersion`, currently `v1.7.8`):

```bash
helm install brouter brouter/brouter \
  --namespace brouter \
  --create-namespace \
  --set image.tag=v1.7.8
```

### 3) Upgrade
```bash
helm upgrade brouter brouter/brouter -n brouter
```

### 4) Uninstall
```bash
helm uninstall brouter -n brouter
```

## Common configurations

Create a `my-values.yaml` and pass it with `-f my-values.yaml`.

### 1) Expose via Ingress
```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: brouter.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: brouter-tls
      hosts:
        - brouter.example.com
```

Install/upgrade with:
```bash
helm upgrade --install brouter brouter/brouter -n brouter -f my-values.yaml
```

### 2) Expose via Gateway API HTTPRoute
Requires Gateway API CRDs and a `Gateway` present.

```yaml
httpRoute:
  enabled: true
  parentRefs:
    - name: gateway
      sectionName: http
  hostnames:
    - brouter.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
```

### 3) Autoscaling (HPA)
```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### 4) Resources
```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### 6) Image settings
```yaml
image:
  repository: joeakeem/brouter
  pullPolicy: IfNotPresent
  # tag defaults to the chart appVersion if empty
  tag: "v1.7.8"
```

### 7) Extra volumes and mounts
```yaml
volumes:
  - name: data
    emptyDir: {}
volumeMounts:
  - name: data
    mountPath: /data
```

### 8) Init containers / Get Routing Segments

`initContainers` can be used to download the desired segments, for example like so:

```yaml
initContainers:
  - name: download-segments
    image: busybox:latest
    command: [ 'sh', '-c', 'wget -P /segments4 https://brouter.de/brouter/segments4/E5_N45.rd5' ]
    volumeMounts:
      - name: segments4
        mountPath: /segments4
```

## All configurable values
The chart supports the following top-level values (see `brouter/values.yaml` for the full list and inline docs):

- `replicaCount` (number)
- `image.repository`, `image.tag`, `image.pullPolicy`
- `imagePullSecrets` (list)
- `nameOverride`, `fullnameOverride`
- `serviceAccount.*`
- `podAnnotations`, `podLabels`
- `podSecurityContext`, `securityContext`
- `service.type`, `service.port`
- `ingress.*`
- `httpRoute.*`
- `resources`
- `livenessProbe`, `readinessProbe`
- `autoscaling.*`
- `volumes`, `volumeMounts`
- `nodeSelector`, `tolerations`, `affinity`
- `initContainers`

## Repository and releases
This repository uses the official `helm/chart-releaser-action` to package the chart and publish it to the `gh-pages` branch. New chart versions are created from Git tags/versions and become available at:

```
https://<OWNER>.github.io/brouter-helm-chart
```

If you use a custom domain, set the `charts_repo_url` accordingly and update the `helm repo add` URL.

## Troubleshooting
- Run `kubectl get pods -n brouter` and check pod logs with `kubectl logs -n brouter deploy/brouter`.
- Ensure your Ingress controller or Gateway is installed and configured if exposing externally.
- If pulling images from a private registry, configure `imagePullSecrets` and reference it under `values.yaml:imagePullSecrets`.
