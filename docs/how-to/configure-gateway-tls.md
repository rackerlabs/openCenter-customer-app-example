---
id: configure-gateway-tls
title: "Configure Gateway API with TLS"
sidebar_label: Gateway TLS
description: Set up HTTPS listeners on the shared Gateway using cert-manager for automatic certificate provisioning.
doc_type: how-to
audience: "app developers, platform engineers"
tags: [gateway-api, tls, cert-manager, https, ingress]
---

**Purpose:** For app developers and platform engineers, shows how to configure TLS termination on the shared Gateway resource using cert-manager and Gateway API.

## Task summary

Add an HTTPS listener to the Gateway, create the cert-manager issuer chain, and attach an HTTPRoute so traffic reaches your application over TLS.

## Prerequisites

- cert-manager deployed on the cluster (standard in openCenter platform services).
- Gateway API CRDs installed (provided by the platform team).
- A Gateway controller running (e.g., Envoy Gateway with `gatewayClassName: eg`).

## Steps

### 1. Create the certificate issuer chain

For development and testing, use a self-signed CA. For production, replace with a real CA issuer (ACME/Let's Encrypt or an internal PKI).

Create a self-signed issuer in the `cert-manager` namespace:

```yaml
# app1/sample-selfsigned-issuer.yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: sample-selfsigned-issuer
  namespace: cert-manager
spec:
  selfSigned: {}
```

Create a CA certificate signed by the self-signed issuer:

```yaml
# app1/sample-selfsigned-ca.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: sample-selfsigned-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: opencenter.dev
  secretName: sample-root-secret
  duration: 87600h0m0s       # 10 years
  renewBefore: 360h0m0s      # 15 days
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: sample-selfsigned-issuer
    kind: Issuer
    group: cert-manager.io
```

Create a ClusterIssuer that uses the CA:

```yaml
# app1/sample-ca-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: sample-ca-issuer
spec:
  ca:
    secretName: sample-root-secret
```

The chain is: self-signed issuer → CA certificate → ClusterIssuer. The ClusterIssuer can then sign certificates for any namespace.

### 2. Configure the Gateway listener

Add or update a listener in `gateway-resources/gateway.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: customer-managed-gateway
  namespace: app1
  annotations:
    cert-manager.io/cluster-issuer: sample-ca-issuer
spec:
  gatewayClassName: eg
  listeners:
    - name: my-app-https
      port: 443
      protocol: HTTPS
      hostname: "myapp.<your-cluster-domain>"
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - group: ""
            kind: Secret
            name: myapp-tls
```

Key points:

- The `cert-manager.io/cluster-issuer` annotation tells cert-manager which issuer to use for certificate generation.
- `certificateRefs` points to the Secret where cert-manager stores the generated certificate.
- `hostname` must match the hostname in your HTTPRoute.
- `allowedRoutes.namespaces.from: All` permits HTTPRoutes from any namespace to attach. Restrict to `Same` or use a selector if you need namespace isolation.

### 3. Create the HTTPRoute

In your application directory, create an HTTPRoute that references the new listener:

```yaml
# app1/httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: app1
spec:
  hostnames:
    - "myapp.<your-cluster-domain>"
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: customer-managed-gateway
      namespace: app1
      sectionName: my-app-https    # Must match listener name
  rules:
    - backendRefs:
        - group: ""
          kind: Service
          name: my-app
          port: 80
          weight: 1
      matches:
        - path:
            type: PathPrefix
            value: /
```

The `sectionName` field binds this route to the specific listener. If omitted, the route attaches to all listeners matching the hostname.

### 4. Validate and push

```bash
kustomize build .
pre-commit run --all-files
git add .
git commit -m "Add TLS listener for myapp"
git push origin main
```

## Verification

After Flux reconciles:

```bash
# Check the certificate was issued
kubectl get certificate -n app1

# Verify the TLS secret exists
kubectl get secret myapp-tls -n app1

# Check the Gateway status
kubectl get gateway customer-managed-gateway -n app1 -o yaml
```

The certificate `READY` column should show `True`. If it stays `False`, check cert-manager logs:

```bash
kubectl logs -n cert-manager deploy/cert-manager -f
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Certificate stays `False` | ClusterIssuer not found or CA secret missing | Verify `sample-root-secret` exists in `cert-manager` namespace |
| Gateway listener shows `Invalid` | Hostname conflict with another listener | Each listener needs a unique hostname; check for duplicates |
| HTTPRoute not attaching | `sectionName` mismatch | Confirm the HTTPRoute `sectionName` matches the Gateway listener `name` exactly |
| `connection refused` on port 443 | Gateway controller not processing the listener | Check the gateway controller pod logs; verify `gatewayClassName` matches the installed controller |
