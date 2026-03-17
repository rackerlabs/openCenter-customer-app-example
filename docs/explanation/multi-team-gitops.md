---
id: multi-team-gitops
title: "Multi-Team GitOps Model"
sidebar_label: Multi-Team GitOps
description: How platform teams and application teams collaborate through separated Git repositories and FluxCD reconciliation.
doc_type: explanation
audience: "app developers, platform engineers, operators"
tags: [gitops, fluxcd, multi-team, architecture, separation-of-concerns]
---

**Purpose:** For platform engineers and app developers, explains the multi-team GitOps collaboration model, covering repository boundaries, trust relationships, and the reasoning behind the separation.

## Concept summary

The openCenter customer app pattern splits cluster management into two distinct Git boundaries:

1. A cluster configuration repository owned by the platform team.
2. One or more application repositories owned by individual app teams.

FluxCD bridges the two. The platform team creates `GitRepository` and `Kustomization` resources that tell Flux where to find application manifests. Application teams push changes to their own repos. Flux pulls and applies them. Neither team needs write access to the other's repository.

## How it works

```
┌─────────────────────────┐       ┌─────────────────────────┐
│  Cluster Config Repo    │       │  Application Repo       │
│  (platform team)        │       │  (app team)             │
│                         │       │                         │
│  GitRepository ─────────┼──────▶│  kustomization.yaml     │
│  Kustomization          │  pull │  gateway-resources/     │
│  Flux Git Secret        │       │  app1/                  │
│                         │       │  app2/                  │
└────────────┬────────────┘       └─────────────────────────┘
             │
             │ Flux reconciles both
             ▼
┌─────────────────────────┐
│  Kubernetes Cluster     │
│                         │
│  Platform services      │
│  Application workloads  │
│  Gateway + HTTPRoutes   │
└─────────────────────────┘
```

The flow:

1. The platform team commits a `GitRepository` resource pointing to the app team's repo, plus a `Kustomization` that tells Flux which path to render.
2. The platform team creates a Flux Git Secret (SSH deploy key) and shares the public key with the app team.
3. The app team adds the public key as a read-only deploy key on their repository.
4. Flux Source Controller polls the app repo on the configured interval (typically 1 minute).
5. On new commits, Flux Kustomize Controller renders `kustomize build` against the repo and applies the diff to the cluster.
6. Resources removed from Git are pruned from the cluster (when `prune: true`).

## Why separate repositories

A single monorepo for all teams is simpler to set up but creates problems at scale:

- Access control becomes coarse. Granting an app team write access to the cluster config repo means they can modify platform services, other teams' apps, or Flux configuration.
- Blast radius grows. A bad commit from one team can break reconciliation for every application in the repo.
- Review bottlenecks form. Platform teams become gatekeepers for every app change, slowing deployment velocity.

Separate repos solve these by giving each team full ownership of their deployment lifecycle while the platform team controls what Flux watches and how it applies resources.

The tradeoff: more repositories to manage, and the platform team must create `GitRepository` + `Kustomization` + Secret for each new app. In practice this is a one-time setup per application, and the manifests are small and templatable.

## Trust boundaries

| Boundary | Who controls it | What it governs |
|---|---|---|
| Cluster config repo | Platform team | Which app repos Flux watches, reconciliation intervals, prune behavior |
| Flux Git Secret | Platform team creates, app team adds deploy key | Authentication between Flux and the app repo |
| Application repo | App team | Kubernetes manifests, Helm values, Gateway routes |
| Kyverno policies | Platform team (via openCenter-gitops-base) | What resources are allowed in the cluster |
| Pod Security Admission | Platform team (via Kubespray) | Baseline pod security enforcement |

App teams can deploy anything their manifests describe, but Kyverno policies and Pod Security Admission act as guardrails. A Deployment requesting privileged containers, for example, will be rejected by the Kyverno `disallow-privileged-containers` policy regardless of what the app team commits.

## Shared resources: the Gateway pattern

The Gateway API resource sits in a shared location (`gateway-resources/`) because multiple applications attach HTTPRoutes to the same Gateway. This is a deliberate design choice:

- One Gateway per cluster (or per tenant group) reduces the number of load balancer IPs and TLS certificates.
- Each app creates its own HTTPRoute in its own namespace, referencing the Gateway by name and `sectionName`.
- The Gateway's `allowedRoutes.namespaces.from: All` permits cross-namespace attachment.

The risk: conflicting hostnames. If two teams configure HTTPRoutes with the same hostname, the Gateway controller's conflict resolution behavior varies by implementation. Coordinate hostname assignments through the platform team or a shared registry.

## Alternatives considered

### Single monorepo with path-based Kustomizations

Flux supports multiple `Kustomization` resources pointing to different paths in the same repo. This avoids multi-repo complexity but requires fine-grained Git permissions (e.g., GitHub CODEOWNERS) to enforce team boundaries. It works for small teams but breaks down when teams need independent release cadences.

### ArgoCD ApplicationSets

ArgoCD's ApplicationSet controller can auto-discover and deploy from multiple repos using generators. The openCenter platform chose FluxCD for its pull-based model and native Kustomize/Helm support, but the multi-team pattern translates to ArgoCD if needed.

### Namespace-scoped Flux instances

Running separate Flux instances per namespace provides stronger isolation but increases operational overhead. The shared Flux control plane with per-app `GitRepository` resources is the recommended default unless regulatory requirements demand full namespace isolation.

## Common misconceptions

"App teams need cluster-admin to deploy." They don't. Flux runs with its own service account in `flux-system`. App teams never interact with `kubectl` directly in production — they push to Git and Flux applies. The platform team controls what Flux is allowed to do via RBAC on the Flux service account.

"Removing a file from Git deletes it from the cluster." Only when `prune: true` is set on the Kustomization. This is the recommended default, but the platform team can disable it for specific applications where manual cleanup is preferred.

"HelmRelease values must be in the Git repo." Values can come from Kubernetes Secrets (via `valuesFrom`), which is the recommended pattern for sensitive configuration. The `secretGenerator` in Kustomize creates these Secrets from local files, but the actual secret data can also be managed by External Secrets Operator or Sealed Secrets.

## Further reading

- [Deploy your first application](../tutorials/deploy-first-app.md) — hands-on walkthrough
- [Manifest structure reference](../reference/manifest-structure.md) — field-level documentation for every file
- [FluxCD multi-tenancy documentation](https://fluxcd.io/flux/installation/configuration/multitenancy/) — upstream guidance on tenant isolation
