---
title: "How Istio mTLS With Spire Works"
date: 2025-10-17T11:45:22+08:00
draft: false
description: ""
tags: ["Istio", "SPIFFE", "SPIRE"]
categories: ["Security"]
author: "Shawn Zhang"
showToc: true
TocOpen: false
hidemeta: false
comments: false
disableHLJS: false
disableShare: false
searchHidden: false
mermaid: true
cover:
    image: ""
    alt: ""
    caption: ""
    relative: false
    hidden: true
---

# Istio mTLS with SPIRE - How It Works

## Overview

When using Istio with SPIRE, applications communicate using plain HTTP, but the Istio sidecars automatically upgrade connections to mTLS using SPIRE-issued certificates. This provides transparent security without requiring application code changes.

## Communication Flow

{{< mermaid >}}
sequenceDiagram
    participant Curl as curl container<br/>(Plain HTTP)
    participant CurlProxy as curl's istio-proxy<br/>SPIFFE: spiffe://foo.com/ns/default/sa/curl
    participant HttpbinProxy as httpbin's istio-proxy<br/>SPIFFE: spiffe://foo.com/ns/default/sa/httpbin
    participant Httpbin as httpbin container<br/>(Plain HTTP)

    Curl->>CurlProxy: 1. HTTP Request<br/>http://httpbin:8000/headers
    CurlProxy->>HttpbinProxy: 2. mTLS Handshake<br/>(mutual authentication)
    CurlProxy->>HttpbinProxy: 3. Encrypted mTLS Connection<br/>(SPIRE certificates)
    HttpbinProxy->>Httpbin: 4. HTTP Request<br/>(decrypted, localhost)
    Httpbin->>HttpbinProxy: HTTP Response
    HttpbinProxy->>CurlProxy: 5. Encrypted Response<br/>(adds X-Forwarded-Client-Cert)
    CurlProxy->>Curl: HTTP Response<br/>(decrypted)
{{< /mermaid >}}

## Step-by-Step Process

### 1. Application Makes HTTP Request
```bash
curl http://httpbin:8000/headers
```
- The curl container sends a plain HTTP request
- No TLS, no certificates, no encryption at application level

### 2. Sidecar Intercepts Request
- curl's istio-proxy sidecar intercepts the outbound HTTP request
- Determines the destination is httpbin service

### 3. mTLS Handshake
- curl's sidecar initiates mTLS connection to httpbin's sidecar
- Both sidecars present their SPIRE-issued certificates:
  - **curl sidecar**: `spiffe://foo.com/ns/default/sa/curl`
  - **httpbin sidecar**: `spiffe://foo.com/ns/default/sa/httpbin`
- Mutual authentication succeeds using SPIRE trust domain

### 4. Encrypted Communication
- HTTP request is encrypted and sent over mTLS connection
- Only the sidecars handle encryption/decryption
- Application containers remain unaware of TLS

### 5. Sidecar Forwards to Application
- httpbin's sidecar decrypts the request
- Forwards plain HTTP to httpbin container on localhost
- Adds `X-Forwarded-Client-Cert` header with client identity

### 6. Response Returns
- httpbin container sends HTTP response
- httpbin's sidecar encrypts it with mTLS
- curl's sidecar decrypts and forwards to curl container

## Evidence of mTLS

### X-Forwarded-Client-Cert Header
```json
{
  "X-Forwarded-Client-Cert": [
    "By=spiffe://foo.com/ns/default/sa/httpbin;Hash=...;Subject=\"O=SPIRE,C=US\";URI=spiffe://foo.com/ns/default/sa/curl"
  ]
}
```

This header proves:
- **By**: Server identity (httpbin's sidecar)
- **URI**: Client identity (curl's sidecar)
- **Subject**: Certificate issued by SPIRE
- **Hash**: Certificate fingerprint

### Certificate Verification
```bash
# Check curl's certificate
istioctl proxy-config secret curl-pod -o json | \
  jq -r '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | \
  base64 --decode | openssl x509 -text -noout

# Shows:
# Issuer: O=SPIRE
# URI: spiffe://foo.com/ns/default/sa/curl
```

## Why HTTP Instead of HTTPS?

### Benefits of Transparent mTLS

1. **Zero Application Changes**
   - No TLS libraries needed in application code
   - No certificate management in applications
   - Developers write simple HTTP code

2. **Centralized Security**
   - Security policy managed by platform team
   - Consistent mTLS across all services
   - Certificate rotation handled automatically

3. **Simplified Development**
   - Local development uses plain HTTP
   - Production gets automatic mTLS
   - No environment-specific code

4. **Performance**
   - Sidecars handle TLS termination
   - Applications focus on business logic
   - Optimized TLS implementation in Envoy

## Key Components

### SPIRE
- Issues X.509 certificates (SVIDs) to workloads
- Provides SPIFFE IDs based on Kubernetes identity
- Manages certificate lifecycle and rotation

### SPIFFE CSI Driver
- Mounts SPIRE socket into pods
- Path: `/run/secrets/workload-spiffe-uds`
- Enables Envoy to fetch certificates from SPIRE

### Istio Sidecar (Envoy)
- Intercepts all inbound/outbound traffic
- Performs mTLS handshake with peer sidecars
- Fetches certificates from SPIRE via CSI socket
- Adds identity headers for authorization

### Application Container
- Sends/receives plain HTTP
- Unaware of mTLS or certificates
- Focuses on business logic only

## Configuration Requirements

### 1. SPIRE Installation
```yaml
global:
  spire:
    clusterName: foo-eks-cluster
    trustDomain: foo.com
```

### 2. Istio Configuration
```yaml
meshConfig:
  trustDomain: foo.com  # Must match SPIRE

values:
  sidecarInjectorWebhook:
    templates:
      spire: |
        labels:
          spiffe.io/spire-managed-identity: "true"
        spec:
          volumes:
            - name: workload-socket
              csi:
                driver: "csi.spiffe.io"
```

### 3. Workload Registration
```yaml
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: istio-sidecar-reg
spec:
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe.io/spire-managed-identity: "true"
```

### 4. Pod Deployment
```yaml
metadata:
  labels:
    spiffe.io/spire-managed-identity: "true"
  annotations:
    inject.istio.io/templates: "sidecar,spire"
```

## Verification Commands

### Test mTLS Communication
```bash
kubectl exec curl-pod -c curl -- curl http://httpbin:8000/headers
```

### Check Certificate
```bash
istioctl proxy-config secret curl-pod -o json | \
  jq -r '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | \
  base64 --decode | openssl x509 -text -noout | grep "URI:spiffe"
```

### Verify SPIRE Entries
```bash
kubectl exec -n spire-server spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server entry show -socketPath /tmp/spire-server/private/api.sock
```

## Common Misconceptions

### ❌ "Applications must use HTTPS"
**Reality**: Applications use HTTP. Sidecars handle mTLS automatically.

### ❌ "Need to manage certificates in application"
**Reality**: SPIRE and Istio manage all certificates. Applications are unaware.

### ❌ "mTLS requires code changes"
**Reality**: Zero code changes. Just add labels and annotations to pods.

### ❌ "HTTP is insecure in service mesh"
**Reality**: HTTP between container and sidecar is on localhost. mTLS protects network traffic.

## Security Guarantees

1. **Mutual Authentication**: Both client and server verify each other's identity
2. **Encryption**: All network traffic encrypted with TLS 1.3
3. **Identity-based Authorization**: Policies based on SPIFFE IDs
4. **Automatic Rotation**: Certificates rotated without application restart
5. **Zero Trust**: Every connection authenticated, even within cluster

## Troubleshooting

### Pod Stuck at 1/2 Ready
- Check if pod has label: `spiffe.io/spire-managed-identity: "true"`
- Verify ClusterSPIFFEID matches the pod
- Check istio-proxy logs: `kubectl logs pod-name -c istio-proxy`

### "workload is not authorized" Error
- ClusterSPIFFEID selectors don't match the pod
- SPIRE entry not created for the workload
- Check: `kubectl get clusterspiffeid -o yaml`

### No X-Forwarded-Client-Cert Header
- mTLS not enabled or working
- Check both pods have sidecars injected
- Verify both pods have SPIRE certificates

## References

- [Istio SPIRE Integration](https://istio.io/latest/docs/ops/integrations/spire/)
- [SPIFFE Specification](https://spiffe.io/docs/latest/spiffe-about/overview/)
- [SPIRE Documentation](https://spiffe.io/docs/latest/spire-about/spire-concepts/)
- [Envoy SDS API](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret)
