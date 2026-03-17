---
id: deploy-first-app
title: "Deploy Your First Application"
sidebar_label: First App
description: Walk through deploying a basic application on an openCenter cluster using Kustomize and FluxCD.
doc_type: tutorial
audience: "app developers, platform engineers"
tags: [tutorial, deployment, kustomize, fluxcd, getting-started]
---

**Purpose:** For app developers, shows how to deploy a basic web application on an openCenter-managed cluster, covering repository setup through live traffic routing.

## Outcome

By the end of this tutorial you will have:

- A running Deployment with two nginx replicas behind a ClusterIP Service.
- An HTTPRoute that exposes the app through the cluster's Gateway API gateway.
- A working local validation workflow using `kustomize build` and `kubeconform`.

## Prerequisites

| Requirement | Why |
|---|---|
| Access to an openCenter-managed Kubernetes cluster with FluxCD running | Flux reconciles your manifests automatically |
| `kubectl` configured with a valid kubeconfig for the target cluster | Needed for verification steps |
| `kustomize` CLI (v5+) | Local manifest rendering |
| `kubeconform` | Schema validation before push |
| `pre-commit` (`pip install pre-commit` or `brew install pre-commit`) | YAML linting |
| A Git repository the platform team has configured Flux to watch | Flux pulls from your repo, not the other way around |

If you don't have a Flux-watched repository yet, ask your platform team to create the `GitRepository` and `Kustomization` resources. See the [FluxCD integration reference](../reference/manifest-structure.md#platform-team-manifests) for the exact manifests they need.

## Step 1 — Clone the example repository

```bash
git clone https://github.com/<org>/openCenter-customer-app-example
cd openCenter-customer-app-example
```

The repository ships with two sample apps. This tutorial focuses on `app1/`, which uses raw Kubernetes manifests.

## Step 2 — Review the directory layout

```
.
├── kustomization.yaml          # Root: assembles gateway + apps
├── gateway-resources/
│   ├── gateway.yaml            # Shared Gateway with TLS listeners
│   └── kustomization.yaml
├── app1/
│   ├── namespace.yaml          # Dedicated namespace
│   ├── deployment.yaml         # nginx Deployment (2 replicas)
│   ├── service.yaml            # ClusterIP Service
│   ├── httproute.yaml          # Gateway API routing
│   ├── sample-selfsigned-issuer.yaml
│   ├── sample-selfsigned-ca.yaml
│   ├── sample-ca-issuer.yaml   # cert-manager ClusterIssuer
│   └── kustomization.yaml
└── app2/                       # Helm-based app (ignore for now)
```

The root `kustomization.yaml` composes `gateway-resources/` and each app directory. Flux renders this the same way `kustomize build .` does locally.

## Step 3 — Customize the hostname

Open `gateway-resources/gateway.yaml` and replace the demo hostname with your cluster's domain:

```yaml
listeners:
  - name: web-https
    port: 443
    protocol: HTTPS
    hostname: app1.<your-cluster-domain>   # ← change this
```

Then update `app1/httproute.yaml` to match:

```yaml
spec:
  hostnames:
    - "app1.<your-cluster-domain>"         # ← same hostname
```

Both values must agree — the HTTPRoute attaches to the Gateway listener by `sectionName` and hostname.

## Step 4 — Validate locally

```bash
kustomize build .
```

This should print the full set of rendered manifests without errors. Pipe the output through `kubeconform` to catch schema issues:

```bash
kubeconform <(kustomize build .)
```

A clean run produces no output. Any validation errors print the resource name and field path.

## Step 5 — Install and run pre-commit hooks

```bash
pre-commit install
pre-commit run --all-files
```

The repository includes a `.yamllint` config that enforces 2-space indentation, 120-character line length, and strict truthy values. Fix any reported issues before committing.

## Step 6 — Commit and push

```bash
git add .
git commit -m "Configure app1 for <your-cluster-domain>"
git push origin main
```

Flux polls the repository on the interval configured in the `GitRepository` resource (typically 1 minute). After the next reconciliation cycle, your resources appear in the cluster.

## Check your work

Verify the Deployment rolled out:

```bash
kubectl get deployment web-app -n app1
```

Expected output (two ready replicas):

```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
web-app   2/2     2            2           45s
```

Verify the HTTPRoute is attached to the Gateway:

```bash
kubectl get httproute web-app -n app1
```

Check the `PARENTS` column shows `customer-managed-gateway`.

If the Deployment is not appearing, force a reconciliation:

```bash
flux reconcile kustomization customer-app --with-source
```

Then check the Flux kustomize-controller logs for errors:

```bash
kubectl logs -n flux-system deploy/kustomize-controller -f
```

## Next steps

- [Add a Helm-based application](../how-to/add-helm-app.md) using the `app2/` pattern.
- [Configure Gateway API with TLS](../how-to/configure-gateway-tls.md) for production certificate management.
- Read the [multi-team GitOps model explanation](../explanation/multi-team-gitops.md) to understand how platform and app teams collaborate.
