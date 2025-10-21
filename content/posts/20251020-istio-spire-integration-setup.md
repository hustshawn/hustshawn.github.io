---
title: "Implement mTLS using Istio Spire Integration"
date: 2025-10-20T11:50:49+08:00
draft: false
description: ""
tags: []
categories: []
author: "Shawn Zhang"
showToc: true
TocOpen: false
hidemeta: false
comments: false
disableHLJS: false
disableShare: false
searchHidden: false
cover:
    image: "/images/20251020-title.png"
    alt: ""
    caption: ""
    relative: false
    hidden: true
---
# Istio + SPIRE Integration - Complete Setup Guide

This guide provides step-by-step instructions to integrate Istio with SPIRE for workload identity management.

## Prerequisites

- Kubernetes cluster (tested on EKS)
- `kubectl` configured with cluster access
- `helm` 3.x installed
- `istioctl` installed
- Cluster context name (e.g., `foo-eks-cluster`)

## Step 1: Install SPIRE

### 1.1 Add SPIRE Helm Repository

```bash
helm repo add spiffe https://spiffe.github.io/helm-charts-hardened/
helm repo update
```

### 1.2 Install SPIRE CRDs

```bash
helm install spire-crds spiffe/spire-crds \
  -n spire-server \
  --create-namespace
```

### 1.3 Install SPIRE Server and Agent

Create a values file for your cluster. For example, `spire-values-foo-cluster.yaml`:

```yaml
global:
  spire:
    clusterName: foo-eks-cluster
    trustDomain: foo.com

spire-server:
  controllerManager:
    watchClassless: true
  caTTL: 720h

spiffe-oidc-discovery-provider:
  enabled: false
```

Install SPIRE using the values file:

```bash
helm install spire spiffe/spire \
  -n spire-server \
  -f spire-values-foo-cluster.yaml
```

**Important:** 
- `trustDomain` must match Istio's trust domain exactly
- `clusterName` must be your actual cluster name (not "example-cluster")
- `watchClassless: true` allows ClusterSPIFFEID resources without explicit `className` field
- OIDC discovery provider is disabled as it's not needed for basic Istio integration

### 1.4 Verify SPIRE Installation

```bash
kubectl get pods -n spire-server
```

Expected output:
```text
NAME                                READY   STATUS    RESTARTS   AGE
spire-agent-xxxxx                   1/1     Running   0          1m
spire-server-0                      2/2     Running   0          1m
spire-spiffe-csi-driver-xxxxx       2/2     Running   0          1m
```

Wait for all pods to be ready:

```bash
kubectl rollout status statefulset spire-server -n spire-server --timeout=120s
kubectl rollout status daemonset spire-agent -n spire-server --timeout=120s
```

## Step 2: Create SPIRE Registration Entries

### 2.1 Create ClusterSPIFFEID for Istio Ingress Gateway

```bash
kubectl apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: istio-ingressgateway-reg
spec:
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  workloadSelectorTemplates:
    - "k8s:ns:istio-system"
    - "k8s:sa:istio-ingressgateway-service-account"
EOF
```

**Note:** The `className` field is optional when `watchClassless: true` is set in the controller manager configuration.

### 2.2 Create ClusterSPIFFEID for Application Workloads

```bash
kubectl apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: default
spec:
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels:
      spiffe.io/spire-managed-identity: "true"
  workloadSelectorTemplates:
  - "k8s:ns:{{ .PodMeta.Namespace }}"
  - "k8s:sa:{{ .PodSpec.ServiceAccountName }}"
EOF

```

### 2.3 Verify ClusterSPIFFEID Resources

```bash
kubectl get clusterspiffeid
```

Expected output:
```text
NAME                        AGE
default                     10s
istio-ingressgateway-reg    20s
```

## Step 3: Install Istio with SPIRE Integration

### 3.1 Create Istio Configuration File

Create a file named `istio-spire-config.yaml` with the following content (replace `foo.com` and `foo-eks-cluster` with your values):

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  profile: default
  meshConfig:
    trustDomain: foo.com
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: foo-eks-cluster
      network: foo-network
    sidecarInjectorWebhook:
      templates:
        spire: |
          labels:
            spiffe.io/spire-managed-identity: "true"
          spec:
            containers:
            - name: istio-proxy
              volumeMounts:
              - name: workload-socket
                mountPath: /run/secrets/workload-spiffe-uds
                readOnly: true
            volumes:
              - name: workload-socket
                csi:
                  driver: "csi.spiffe.io"
                  readOnly: true
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      label:
        istio: ingressgateway
      k8s:
        overlays:
          # This is used to customize the ingress gateway template.
          # It adds the CSI driver mounts, as well as an init container
          # to stall gateway startup until the CSI driver mounts the socket.
          - apiVersion: apps/v1
            kind: Deployment
            name: istio-ingressgateway
            patches:
              - path: spec.template.spec.volumes.[name:workload-socket]
                value:
                  name: workload-socket
                  csi:
                    driver: "csi.spiffe.io"
                    readOnly: true
              - path: spec.template.spec.containers.[name:istio-proxy].volumeMounts.[name:workload-socket]
                value:
                  name: workload-socket
                  mountPath: "/run/secrets/workload-spiffe-uds"
                  readOnly: true
```

### 3.2 Create Istio Namespace

```bash
kubectl create namespace istio-system
```

### 3.3 Install Istio

```bash
istioctl install -y -f istio-spire-config.yaml
```

### 3.4 Verify Istio Installation

```bash
kubectl get pods -n istio-system
```

Expected output:
```text
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-xxxxxxxxxx-xxxxx   1/1     Running   0          1m
istiod-xxxxxxxxxx-xxxxx                 1/1     Running   0          1m
```

Wait for deployments to be ready:

```bash
kubectl rollout status deployment istiod -n istio-system --timeout=120s
kubectl rollout status deployment istio-ingressgateway -n istio-system --timeout=120s
```

### 3.5 Enable Istio Injection for Default Namespace

```bash
kubectl label namespace default istio-injection=enabled
```

### 3.6 Configure Strict mTLS Policy (Optional)

Apply a PeerAuthentication policy to enforce strict mTLS for all workloads in the default namespace:

```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: "default"
  namespace: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

**What this does:**
- Enforces mutual TLS for all service-to-service communication in the default namespace
- Rejects any plaintext traffic between services
- Ensures all workloads must present valid SPIRE-issued certificates

**Note:** You can also apply this policy at the mesh level by creating it in the `istio-system` namespace without a namespace-specific selector.

## Step 4: Deploy Test Workloads

### 4.1 Deploy Sleep and Httpbin Applications

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sleep
  namespace: default
---
apiVersion: v1
kind: Service
metadata:
  name: sleep
  namespace: default
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: sleep
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
        spiffe.io/spire-managed-identity: "true"
    spec:
      serviceAccountName: sleep
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "infinity"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: httpbin
  namespace: default
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: default
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 8080
  selector:
    app: httpbin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
        spiffe.io/spire-managed-identity: "true"
    spec:
      serviceAccountName: httpbin
      containers:
      - image: mccutchen/go-httpbin
        name: httpbin
        ports:
        - containerPort: 8080
EOF
```

**Critical:** All workloads must have the label `spiffe.io/spire-managed-identity: "true"` to receive SPIRE certificates.

### 4.2 Wait for Pods to be Ready

```bash
kubectl wait --for=condition=ready pod -n default -l app=sleep --timeout=60s
kubectl wait --for=condition=ready pod -n default -l app=httpbin --timeout=60s
```

## Step 5: Verify SPIRE Integration

### 5.1 Check Sleep Pod Certificate

```bash
POD=$(kubectl get pod -n default -l app=sleep -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config secret $POD.default -o json | \
  jq -r '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | \
  base64 --decode | \
  openssl x509 -text -noout | \
  grep -E "(Issuer:|Subject Alternative Name:)" -A1
```

Expected output:
```text
        Issuer: O=foo.com
        Validity
--
            X509v3 Subject Alternative Name: critical
                URI:spiffe://foo.com/ns/default/sa/sleep
```

### 5.2 Check Httpbin Pod Certificate

```bash
POD=$(kubectl get pod -n default -l app=httpbin -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config secret $POD.default -o json | \
  jq -r '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | \
  base64 --decode | \
  openssl x509 -text -noout | \
  grep "URI:spiffe"
```

Expected output:
```text
                URI:spiffe://foo.com/ns/default/sa/httpbin
```

### 5.3 Verify Ingress Gateway Certificate

```bash
POD=$(kubectl get pod -n istio-system -l app=istio-ingressgateway -o jsonpath='{.items[0].metadata.name}')
istioctl proxy-config secret $POD.istio-system -o json | \
  jq -r '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | \
  base64 --decode | \
  openssl x509 -text -noout | \
  grep "URI:spiffe"
```

Expected output:
```text
                URI:spiffe://foo.com/ns/istio-system/sa/istio-ingressgateway-service-account
```

### 5.4 Test mTLS Communication

**Important:** In Istio service mesh, mTLS happens transparently between the Envoy proxies (sidecars). The application containers communicate using plain HTTP, but the Istio proxies automatically upgrade the connection to mTLS.

Test communication between workloads:

```bash
POD=$(kubectl get pod -n default -l app=sleep -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n default $POD -c sleep -- curl -s http://httpbin:8000/headers
```

**What's happening:**
1. Sleep container sends plain HTTP request to `http://httpbin:8000`
2. Sleep's Istio sidecar intercepts the request
3. Sleep's sidecar establishes mTLS connection to httpbin's sidecar using SPIRE certificates
4. Httpbin's sidecar receives the mTLS connection and forwards plain HTTP to httpbin container
5. Httpbin's sidecar adds the `X-Forwarded-Client-Cert` header with the client certificate info
![mTLS](/images/20251020-mtls-verification.png)

Look for the `X-Forwarded-Client-Cert` header in the response. This header proves that mTLS is working - it contains the client certificate information that was used in the mTLS connection between the sidecars.

Example output showing mTLS is working:
```json
{
  "headers": {
    "X-Forwarded-Client-Cert": "By=spiffe://foo.com/ns/default/sa/httpbin;Hash=...",
    ...
  }
}
```

### 5.5 Test Multiple Requests

```bash
POD=$(kubectl get pod -n default -l app=sleep -o jsonpath='{.items[0].metadata.name}')
for i in {1..10}; do
  kubectl exec -n default $POD -c sleep -- curl -s -o /dev/null -w "%{http_code}\n" http://httpbin:8000/get
done | sort | uniq -c
```

Expected output (all 200 OK):
```text
  10 200
```

## Step 6: Verify SPIRE Registration Entries

### 6.1 Check Registration Entries in SPIRE Server

```bash
kubectl exec -n spire-server spire-server-0 -c spire-server -- \
  /opt/spire/bin/spire-server entry show \
  -socketPath /tmp/spire-server/private/api.sock
```

You should see entries for:
- `spiffe://foo.com/ns/default/sa/sleep`
- `spiffe://foo.com/ns/default/sa/httpbin`
- `spiffe://foo.com/ns/istio-system/sa/istio-ingressgateway-service-account`

### 6.2 Check ClusterSPIFFEID Status

```bash
kubectl get clusterspiffeid default -o yaml
```

Look for the `status` section showing the number of pods selected and entries created.

## Step 7: Setup Observability (Optional)

### 7.1 Install Prometheus

Install Prometheus to collect metrics from Istio and workloads:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/prometheus.yaml
```

Wait for Prometheus to be ready:

```bash
kubectl wait --for=condition=available deployment/prometheus -n istio-system --timeout=120s
```

### 7.2 Install Kiali

Install Kiali for service mesh visualization:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/kiali.yaml
```

Wait for Kiali to be ready:

```bash
kubectl wait --for=condition=available deployment/kiali -n istio-system --timeout=120s
```

### 7.3 Access Kiali Dashboard

Port-forward to access Kiali:

```bash
kubectl port-forward svc/kiali -n istio-system 20001:20001
```

Open your browser to: **http://localhost:20001**

### 7.4 Generate Traffic for Visualization

Generate continuous traffic to see it in Kiali:

```bash
POD=$(kubectl get pod -n default -l app=sleep -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n default $POD -c sleep -- sh -c \
  "while true; do curl -s http://httpbin:8000/headers > /dev/null; sleep 2; done" &
```

### 7.5 Visualize mTLS and SPIRE Identities in Kiali
{{< figure style="text-align: center; margin: 0 auto;" src="/images/20251020-istio-spire-kiali-security.png" alt="Kiali" caption="Visualize mTLS and SPIRE Identities in Kiali" >}}

In the Kiali dashboard:

1. **Graph View**:
   - Go to **Graph** tab
   - Select namespace: **default**
   - Display: Enable "Security" badges
   - You'll see lock icons (🔒) indicating mTLS connections
   - Traffic flows: sleep → httpbin

2. **Verify SPIRE Identities**:
   - Click on the **httpbin** service node
   - Go to the **Workloads** tab
   - Click on the httpbin workload
   - In the **Logs** tab, select the `istio-proxy` container
   - Search for "SPIFFE" to see SPIRE identity logs

3. **Check mTLS Status**:
   - In the Graph view, the lock icons confirm mTLS is active
   - Click on the edge between sleep and httpbin
   - The side panel shows connection details including mTLS status

4. **View Certificates**:
   - Go to **Workloads** → select **httpbin**
   - Click **Envoy** tab
   - Navigate to **Secrets** section
   - You'll see the SPIRE-issued certificates with SPIFFE IDs

### 7.6 Stop Traffic Generation

To stop the background traffic:

```bash
pkill -f "curl.*httpbin"
```

### 7.7 Install Grafana (Optional)

For advanced metrics visualization:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/grafana.yaml
kubectl wait --for=condition=available deployment/grafana -n istio-system --timeout=120s
```

Access Grafana:

```bash
kubectl port-forward svc/grafana -n istio-system 3000:3000
```

Open: **http://localhost:3000**

Pre-configured Istio dashboards are available showing:
- Mesh metrics
- Service performance
- Workload metrics
- Control plane metrics

## Troubleshooting

### Issue: Pods Not Getting SPIRE Certificates

**Symptoms:** Workload pods show "default" identity errors in logs

**Solutions:**

1. Verify the workload has the required label:
```bash
kubectl get pod <pod-name> -o jsonpath='{.metadata.labels}' | grep spire-managed-identity
```

2. Check SPIRE agent logs:
```bash
kubectl logs -n spire-server daemonset/spire-agent --tail=50
```

3. Verify ClusterSPIFFEID exists and matches:
```bash
kubectl get clusterspiffeid
kubectl describe clusterspiffeid default
```

### Issue: Trust Domain Mismatch

**Symptoms:** Certificate verification errors, authentication failures

**Solutions:**

1. Verify SPIRE trust domain:
```bash
kubectl get configmap -n spire-server spire-server -o jsonpath='{.data.server\.conf}' | grep trust_domain
```

2. Verify Istio trust domain:
```bash
kubectl get configmap istio -n istio-system -o jsonpath='{.data.mesh}' | grep trustDomain
```

3. Both must match exactly. If they don't, reinstall with matching trust domains.

### Issue: Ingress Gateway Not Starting

**Symptoms:** Gateway pod stuck in CrashLoopBackOff or not ready

**Solutions:**

1. Check if ClusterSPIFFEID exists for ingress gateway:
```bash
kubectl get clusterspiffeid istio-ingressgateway-reg
```

2. Verify CSI volume is mounted:
```bash
kubectl get pod -n istio-system -l app=istio-ingressgateway -o yaml | grep -A10 "workload-socket"
```

3. Check gateway logs:
```bash
kubectl logs -n istio-system -l app=istio-ingressgateway -c istio-proxy --tail=100
```

### Issue: Wrong Cluster Name in SPIRE

**Symptoms:** Node attestation failures, agents not connecting

**Solutions:**

1. Check current cluster name in SPIRE config:
```bash
kubectl get configmap -n spire-server spire-agent -o jsonpath='{.data.agent\.conf}' | jq '.plugins.NodeAttestor[0].k8s_psat.plugin_data.cluster'
```

2. If it shows "example-cluster", upgrade SPIRE with correct cluster name:
```bash
helm upgrade spire spiffe/spire -n spire-server \
  --set global.spire.trustDomain=foo.com \
  --set global.spire.clusterName=foo-eks-cluster \
  --reuse-values
```

3. Restart SPIRE agents:
```bash
kubectl rollout restart daemonset spire-agent -n spire-server
```

## Key Configuration Points

### 1. Trust Domain Alignment
- SPIRE `global.spire.trustDomain` **MUST** equal Istio `meshConfig.trustDomain`
- Mismatch causes authentication failures
- Both must use the same value (e.g., `foo.com`)

### 2. Cluster Name
- SPIRE `global.spire.clusterName` must be the actual cluster name
- Default "example-cluster" causes attestation failures
- Use your real cluster name (e.g., `foo-eks-cluster`)

### 3. CSI Driver
- SPIFFE CSI driver is automatically installed with SPIRE Helm chart
- Mounts SPIRE socket at `/run/secrets/workload-spiffe-uds/socket`
- Istio ingress gateway must be configured to use CSI volume (shown in Step 3.1)

### 4. Workload Labels
- Workloads **MUST** have label: `spiffe.io/spire-managed-identity: "true"`
- This label is used by ClusterSPIFFEID selector
- Without this label, workloads won't get SPIRE certificates

### 5. SPIFFE ID Format
- Istio requires: `spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`
- ClusterSPIFFEID template must follow this exact pattern
- Template shown in Step 2.2 provides correct format

### 6. Service Account
- Each workload must have a ServiceAccount
- SPIFFE ID includes the service account name
- Different service accounts get different SPIFFE IDs

### 7. Controller Manager Class Configuration
- `watchClassless: true` allows ClusterSPIFFEID resources without `className` field
- When set to `false`, all ClusterSPIFFEID resources must explicitly specify `className: spire-server-spire`
- Recommended to use `watchClassless: true` for simpler configuration
- Multiple SPIRE installations in the same cluster should use different `className` values

## Verification Checklist

Use this checklist to verify your setup:

- [ ] SPIRE server pod is running (2/2 containers)
- [ ] SPIRE agent pods are running on all nodes (1/1 containers)
- [ ] SPIFFE CSI driver pods are running on all nodes (2/2 containers)
- [ ] ClusterSPIFFEID resources exist (istio-ingressgateway-reg and default)
- [ ] Istio control plane (istiod) is running
- [ ] Istio ingress gateway is running
- [ ] Workload pods have label `spiffe.io/spire-managed-identity: "true"`
- [ ] Workload certificates show correct SPIFFE ID format
- [ ] Workload certificates issued by correct trust domain
- [ ] mTLS communication works (X-Forwarded-Client-Cert header present)
- [ ] All HTTP requests return 200 OK

## Clean Up

To remove the installation:

```bash
# Delete workloads
kubectl delete deployment sleep httpbin -n default
kubectl delete svc sleep httpbin -n default
kubectl delete sa sleep httpbin -n default

# Delete ClusterSPIFFEID resources
kubectl delete clusterspiffeid default istio-ingressgateway-reg

# Uninstall Istio
istioctl uninstall --purge -y
kubectl delete namespace istio-system

# Uninstall SPIRE
helm uninstall spire -n spire-server
helm uninstall spire-crds -n spire-server
kubectl delete namespace spire-server
```

## Multi-Cluster Setup

For setting up Istio + SPIRE across multiple clusters:

1. Create cluster-specific values files for each cluster:

**Cluster 1 (foo-eks-cluster)** - `spire-values-foo-cluster.yaml`:
```yaml
global:
  spire:
    clusterName: foo-eks-cluster
    trustDomain: foo.com

spire-server:
  controllerManager:
    watchClassless: true

spiffe-oidc-discovery-provider:
  enabled: false
```

**Cluster 2 (bar-eks-cluster)** - `spire-values-bar-cluster.yaml`:
```yaml
global:
  spire:
    clusterName: bar-eks-cluster
    trustDomain: bar.com

spire-server:
  controllerManager:
    watchClassless: true

spiffe-oidc-discovery-provider:
  enabled: false
```

2. Install SPIRE on each cluster using its respective values file:
```bash
# On cluster 1
helm install spire spiffe/spire -n spire-server -f spire-values-foo-cluster.yaml

# On cluster 2
helm install spire spiffe/spire -n spire-server -f spire-values-bar-cluster.yaml
```

3. Each cluster operates independently with its own SPIRE server and trust domain

4. For cross-cluster communication, additional configuration is required:
   - Istio multi-cluster setup with remote secrets
   - East-west gateways for cross-cluster traffic
   - SPIRE federation for trust bundle exchange (advanced)

**Note:** Cross-cluster mTLS with federated SPIRE trust domains requires additional Envoy configuration beyond the scope of this guide.

## References

- [Istio SPIRE Integration](https://istio.io/latest/docs/ops/integrations/spire/)
- [SPIRE Documentation](https://spiffe.io/docs/latest/spire-about/)
- [SPIRE Helm Charts](https://artifacthub.io/packages/helm/spiffe/spire)
- [SPIFFE CSI Driver](https://github.com/spiffe/spiffe-csi)
- [SPIRE Controller Manager](https://github.com/spiffe/spire-controller-manager)
- [How to Integrate Istio and SPIRE for Secure Workload Identity](https://imesh.ai/blog/istio-spire-workload-identity/)
