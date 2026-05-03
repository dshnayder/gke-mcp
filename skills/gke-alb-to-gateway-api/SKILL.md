name: gke-alb-to-gateway-api
description: Guides the migration from AWS ALB Ingress Controller to GKE's modern Gateway API or standard GKE Ingress. This skill focuses on translating AWS-specific Ingress annotations (for SSL, health checks, WAF, and redirects) into GKE-native CRDs like FrontendConfig, BackendConfig, and Gateway API resources (HTTPRoute). Use this skill when migrating traffic routing and load balancing from EKS to GKE.

# AWS ALB to GKE Gateway API Migration

## Instructions

### 1. Architectural Shift: Annotations to Resources

*   **AWS ALB Controller:** Relies heavily on **hundreds of annotations** on the `Ingress` object to configure Load Balancer behavior (e.g., `alb.ingress.kubernetes.io/*`).
*   **GKE Gateway API (Recommended):** Replaces complex annotations with **dedicated Kubernetes resources** (`Gateway`, `HTTPRoute`). It offers better separation of concerns between infrastructure and application routing.
*   **GKE Ingress (Classic):** Uses a mix of standard Ingress specs and GKE-specific CRDs (`FrontendConfig`, `BackendConfig`) to handle advanced features.

### 2. Feature Mapping Table

| Feature | AWS ALB Annotation | GKE Equivalent (Gateway API / Ingress) |
| :--- | :--- | :--- |
| **SSL Certificate** | `certificate-arn` | **Gateway:** `listeners.tls` (Cert Manager) <br> **Ingress:** `ManagedCertificate` CRD |
| **Health Checks** | `healthcheck-path`, `port` | **BackendConfig** CRD (referenced by Service) |
| **WAF / Security** | `wafv2-acl-arn` | **BackendConfig** with `securityPolicy` (Cloud Armor) |
| **SSL Policy** | `ssl-policy` | **FrontendConfig** (Ingress) or `GCPGatewayPolicy` (Gateway) |
| **HTTPS Redirect** | `actions.ssl-redirect` | **FrontendConfig** (`redirectToHttps`) or `HTTPRoute` filter |
| **Static IP** | `load-balancer-address` | `Gateway.spec.addresses` or Ingress static IP annotation |
| **Target Type** | `target-type: ip` | **Network Endpoint Groups (NEGs)** (Enabled by default) |

### 3. Migrating to Gateway API (Modern)

The preferred path for GKE migration is using the **Gateway API**.

#### Step A: Define the Gateway (Infrastructure)
```yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: external-http
spec:
  gatewayClassName: gke-l7-global-external-managed
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: my-cert # Referenced from Certificate Manager
```

#### Step B: Define the HTTPRoute (Application)
```yaml
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: app-route
spec:
  parentRefs:
  - name: external-http
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path: { type: PathPrefix, value: /v1 }
    backendRefs:
    - name: app-service
      port: 80
```

### 4. Migrating to GKE Ingress (Classic)

If using classic Ingress, you must translate annotations into `BackendConfig` and `FrontendConfig`.

#### Step A: Translate Health Checks and WAF
```yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: app-backend-config
spec:
  healthCheck:
    requestPath: /healthz
    port: 8080
  securityPolicy:
    name: "my-cloud-armor-policy"
```
*Annotate the Service:* `cloud.google.com/backend-config: '{"default": "app-backend-config"}'`

#### Step B: Translate HTTPS Redirects
```yaml
apiVersion: networking.gke.io/v1beta1
kind: FrontendConfig
metadata:
  name: app-frontend-config
spec:
  redirectToHttps:
    enabled: true
```
*Annotate the Ingress:* `networking.gke.io/v1beta1.FrontendConfig: "app-frontend-config"`

### 5. Migration Strategy

1.  **Extract Intent:** Audit the `alb.ingress.kubernetes.io` annotations in EKS.
2.  **Map Backends:** For each service, create a `BackendConfig` if custom health checks or Cloud Armor are needed.
3.  **Choose Frontend:** Use `Gateway` for new projects or `Ingress` + `FrontendConfig` for direct translation of legacy logic.
4.  **Verification:**
    *   Check Load Balancer status: `kubectl get gateway` or `kubectl get ingress`.
    *   Verify health probes in Google Cloud Console under Network Services > Load Balancing.
    *   Test path-based routing using `curl -H "Host: ..."` against the new IP.
