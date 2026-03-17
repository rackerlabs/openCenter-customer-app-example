---
id: manifest-structure
title: "Repository Manifest Structure"
sidebar_label: Manifest Structure
description: Complete reference for every file and directory in the openCenter customer app repository.
doc_type: reference
audience: "app developers, platform engineers"
tags: [reference, manifests, kustomize, gateway-api, helm, fluxcd]
---

**Purpose:** For app developers and platform engineers, documents every manifest, field convention, and directory role in the repository.

## Overview

The repository uses Kustomize composition to assemble shared Gateway API resources and per-application manifests. FluxCD reconciles the root `kustomization.yaml` against the target cluster.

## Directory layout

```
.
├── kustomization.yaml              # Root composition
├── gateway-resources/              # Shared Gateway API definitions
│   ├── gateway.yaml
│   └── kustomization.yaml
├── app1/                           # Raw-manifest application
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── httproute.yaml
│   ├── sample-selfsigned-issuer.yaml
│   ├── sample-selfsigned-ca.yaml
│   ├── sample-ca-issuer.yaml
│   └── kustomization.yaml
├── app2/                           # Helm-based application
│   ├── namespace.yaml
│   ├── source.yaml
│   ├── helmrelease.yaml
│   ├── httproute.yaml
│   ├── helm-values/
│   │   └── hardened-values-0.35.0.yaml
│   └── kustomization.yaml
├── .pre-commit-config.yaml
└── .yamllint
```

## Root kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - "gateway-resources/"
  - "app1/"
  - "app2/"
```

Each entry is a directory containing its own `kustomization.yaml`. Flux and `kustomize build` resolve them recursively.

To add a new application, append its directory to the `resources` list.

## gateway-resources/

Shared Gateway API resources consumed by all applications.

### gateway.yaml

| Field | Value | Notes |
|---|---|---|
| `apiVersion` | `gateway.networking.k8s.io/v1beta1` | Gateway API version |
| `kind` | `Gateway` | |
| `metadata.name` | `customer-managed-gateway` | Referenced by HTTPRoutes via `parentRefs` |
| `metadata.namespace` | `app1` | Gateway lives in a single namespace; routes attach cross-namespace |
| `metadata.annotations.cert-manager.io/cluster-issuer` | `sample-ca-issuer` | Triggers cert-manager certificate generation |
| `spec.gatewayClassName` | `eg` | Must match the installed gateway controller (Envoy Gateway) |

Each listener defines:

| Listener field | Description |
|---|---|
| `name` | Unique identifier; HTTPRoutes reference this via `sectionName` |
| `port` | Listener port (443 for HTTPS) |
| `protocol` | `HTTPS` for TLS termination |
| `hostname` | FQDN this listener serves |
| `tls.mode` | `Terminate` — Gateway handles TLS; backends receive plain HTTP |
| `tls.certificateRefs` | Secret containing the TLS certificate (created by cert-manager) |
| `allowedRoutes.namespaces.from` | `All` permits cross-namespace HTTPRoutes; `Same` restricts to Gateway namespace |

## app1/ — Raw manifest application

### namespace.yaml

Creates the `app1` namespace. Every application should define its own namespace for resource isolation.

### deployment.yaml

| Field | Value |
|---|---|
| `metadata.name` | `web-app` |
| `metadata.namespace` | `app1` |
| `spec.replicas` | `2` |
| `spec.template.spec.containers[0].image` | `nginx:latest` |
| `spec.template.spec.containers[0].resources.requests` | `cpu: 100m, memory: 128Mi` |
| `spec.template.spec.containers[0].resources.limits` | `cpu: 250m, memory: 256Mi` |

Resource requests and limits are required. Pods without limits may be evicted under memory pressure.

### service.yaml

| Field | Value |
|---|---|
| `metadata.name` | `web-app` |
| `spec.type` | `ClusterIP` |
| `spec.ports[0]` | `port: 80, targetPort: 80` |

The Service name must match the `backendRefs.name` in the HTTPRoute.

### httproute.yaml

| Field | Description |
|---|---|
| `spec.hostnames` | Must match the Gateway listener hostname |
| `spec.parentRefs[0].sectionName` | Must match the Gateway listener `name` |
| `spec.parentRefs[0].namespace` | Namespace where the Gateway lives |
| `spec.rules[0].backendRefs` | Points to the Service by name and port |
| `spec.rules[0].matches[0].path` | `PathPrefix: /` catches all traffic |

### cert-manager resources

Three resources form the certificate issuer chain:

1. `sample-selfsigned-issuer.yaml` — `Issuer` (self-signed, in `cert-manager` namespace)
2. `sample-selfsigned-ca.yaml` — `Certificate` (CA cert, `isCA: true`, signed by the self-signed issuer)
3. `sample-ca-issuer.yaml` — `ClusterIssuer` (uses the CA cert to sign application certificates)

For production, replace this chain with a real CA issuer (ACME, Vault PKI, or internal CA).

## app2/ — Helm-based application

### namespace.yaml

Creates the `headlamp` namespace.

### source.yaml

| Field | Value |
|---|---|
| `kind` | `HelmRepository` |
| `metadata.namespace` | `headlamp` |
| `spec.url` | `https://kubernetes-sigs.github.io/headlamp/` |
| `spec.interval` | `1h` — how often Flux checks for new chart versions |

### helmrelease.yaml

| Field | Value | Notes |
|---|---|---|
| `spec.chart.spec.chart` | `headlamp` | Chart name in the HelmRepository |
| `spec.chart.spec.version` | `0.35.0` | Pinned version; change to upgrade |
| `spec.interval` | `5m` | Reconciliation interval |
| `spec.timeout` | `10m` | Max time for install/upgrade |
| `spec.driftDetection.mode` | `enabled` | Reverts manual cluster changes |
| `spec.install.remediation.retries` | `3` | Retries on first install |
| `spec.valuesFrom` | Secret refs | Loads values from Kubernetes Secrets |

### helm-values/

Contains `hardened-values-<version>.yaml` files. The `kustomization.yaml` uses `secretGenerator` to create a Secret from these files with `disableNameSuffixHash: true` so the HelmRelease reference stays stable.

### httproute.yaml

Same structure as `app1/httproute.yaml`. References the `headlamp-https` listener on the shared Gateway.

## Platform team manifests

These manifests are not in this repository. The platform team creates them in the cluster configuration repository to connect Flux to the application repo.

### GitRepository

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: customer-app
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/<org>/openCenter-customer-app-example
  ref:
    branch: main
```

`interval: 1m` — Flux polls the repository every minute for new commits.

### Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: customer-app
  namespace: flux-system
spec:
  interval: 10m
  prune: true
  wait: true
  path: ./
  sourceRef:
    kind: GitRepository
    name: customer-app
```

| Field | Description |
|---|---|
| `prune: true` | Deletes cluster resources when removed from Git |
| `wait: true` | Waits for resources to become ready before marking reconciliation complete |
| `path: ./` | Renders from the repository root |

## Linting configuration

### .yamllint

| Rule | Setting |
|---|---|
| `indentation.spaces` | `2` |
| `indentation.indent-sequences` | `consistent` |
| `line-length.max` | `120` |
| `truthy.allowed-values` | `true`, `false`, `on`, `off`, `yes`, `no` |

### .pre-commit-config.yaml

Runs `yamllint --strict` on all files before commit using the `mirrors-yamllint` hook (v1.35.1).
