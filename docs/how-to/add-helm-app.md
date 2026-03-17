---
id: add-helm-app
title: "Add a Helm-Based Application"
sidebar_label: Add Helm App
description: Add a Helm-managed application to the repository using FluxCD HelmRelease and HelmRepository resources.
doc_type: how-to
audience: "app developers, platform engineers"
tags: [helm, fluxcd, helmrelease, deployment]
---

**Purpose:** For app developers, shows how to add a new Helm-based application alongside existing Kustomize-managed workloads.

## Task summary

Create a new application directory with a `HelmRepository` source, a `HelmRelease`, an `HTTPRoute` for ingress, and wire it into the root `kustomization.yaml`. The `app2/` directory in this repository demonstrates this pattern with a Headlamp deployment.

## Prerequisites

- A working openCenter cluster with FluxCD Helm Controller running (`helm-controller` pod in `flux-system`).
- The target Helm chart's repository URL and chart version.
- Familiarity with the repository structure (see [manifest structure reference](../reference/manifest-structure.md)).

## Steps

### 1. Create the application directory

```bash
mkdir -p app3/helm-values
```

### 2. Define the namespace

Create `app3/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app3
```

### 3. Add the HelmRepository source

Create `app3/source.yaml` pointing to the chart's repository:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: my-chart-repo
  namespace: app3
spec:
  url: https://charts.example.com
  interval: 1h
```

`interval: 1h` means Flux checks for new chart versions hourly. Shorten this during active development if needed.

### 4. Create the HelmRelease

Create `app3/helmrelease.yaml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: my-app
  namespace: app3
spec:
  releaseName: my-app
  interval: 5m
  timeout: 10m
  driftDetection:
    mode: enabled
  install:
    remediation:
      retries: 3
      remediateLastFailure: true
  upgrade:
    remediation:
      retries: 0
      remediateLastFailure: false
  targetNamespace: app3
  chart:
    spec:
      chart: my-chart
      version: 1.2.3          # Pin to an exact version
      sourceRef:
        kind: HelmRepository
        name: my-chart-repo
        namespace: app3
  valuesFrom:
    - kind: Secret
      name: my-app-values-base
      valuesKey: hardened.yaml
    - kind: Secret
      name: my-app-values-override
      valuesKey: override.yaml
      optional: true
```

Key fields:

- `driftDetection.mode: enabled` — Flux reverts manual changes in the cluster.
- `install.remediation.retries: 3` — retries on first install; avoids transient failures.
- `valuesFrom` — loads values from Kubernetes Secrets. Never commit raw values containing credentials.

### 5. Add Helm values

Place your hardened base values in `app3/helm-values/hardened-values-1.2.3.yaml`. Include the chart version in the filename so upgrades are traceable.

### 6. Wire up the Kustomization

Create `app3/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - source.yaml
  - helmrelease.yaml
secretGenerator:
  - name: my-app-values-base
    type: Opaque
    files:
      - hardened.yaml=helm-values/hardened-values-1.2.3.yaml
    options:
      disableNameSuffixHash: true
    namespace: app3
```

`disableNameSuffixHash: true` keeps the Secret name stable so the HelmRelease `valuesFrom` reference doesn't break.

### 7. Add to the root Kustomization

Edit the root `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - "gateway-resources/"
  - "app1/"
  - "app2/"
  - "app3/"                    # ← add this line
```

### 8. (Optional) Add an HTTPRoute

If the app needs external traffic, create `app3/httproute.yaml` following the pattern in `app1/httproute.yaml`. Reference the appropriate Gateway listener `sectionName` and add a matching listener to `gateway-resources/gateway.yaml`.

### 9. Validate and push

```bash
kustomize build . --enable-helm
pre-commit run --all-files
git add .
git commit -m "Add app3 Helm-based application"
git push origin main
```

Use `--enable-helm` so Kustomize renders the HelmRelease locally.

## Verification

After Flux reconciles (typically within 1–2 minutes):

```bash
# Check HelmRelease status
kubectl get helmrelease my-app -n app3

# Check pods
kubectl get pods -n app3
```

The HelmRelease `READY` column should show `True`. If it shows `False`, inspect the Helm controller logs:

```bash
kubectl logs -n flux-system deploy/helm-controller -f
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `HelmChart not found` | HelmRepository URL is wrong or unreachable | Verify `source.yaml` URL; check `kubectl get helmrepository -n app3` |
| `values secret not found` | Secret name mismatch or `disableNameSuffixHash` missing | Compare `valuesFrom` name with `secretGenerator` output |
| `timeout waiting for install` | Chart needs more time or has failing hooks | Increase `spec.timeout`; check pod events with `kubectl describe pod -n app3` |
