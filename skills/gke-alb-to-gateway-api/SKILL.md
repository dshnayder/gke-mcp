name: gke-alb-to-gateway-api
description: Guides the migration from AWS ALB Ingress Controller to GKE's modern Gateway API. This skill focuses on translating AWS-specific Ingress annotations (for SSL, health checks, WAF, and redirects) into GKE Gateway API resources (Gateway, HTTPRoute, GCPGatewayPolicy, HealthCheckPolicy, GCPBackendPolicy). Use this skill when migrating traffic routing and load balancing from EKS to GKE.

# AWS ALB to GKE Gateway API Migration

## Instructions

### 1. Architectural Shift: Annotations to Gateway API Resources

*   **AWS ALB Controller:** Relies heavily on **annotations** on the `Ingress` object to configure Load Balancer behavior (e.g., `alb.ingress.kubernetes.io/*`). This mixes infrastructure and application concerns within a single, often complex, resource.
*   **GKE Gateway API (Recommended):** An implementation of the open-source Kubernetes Gateway API. It replaces complex annotations with **dedicated Kubernetes resources** (`GatewayClass`, `Gateway`, `HTTPRoute`, and various Policy CRDs). This provides:
    *   **Role-Oriented:** Clear separation of concerns between Platform Admins (managing `GatewayClass`, `Gateway`) and Application Developers (managing `HTTPRoute`).
    *   **Expressive:** Native support for advanced routing, traffic splitting, header manipulation, etc., without custom annotations.
    *   **Extensible:** Uses Policy CRDs (`GCPGatewayPolicy`, `GCPBackendPolicy`, `HealthCheckPolicy`) to configure cloud provider specific features.

### 2. Feature Mapping: AWS Annotations to GKE Gateway API

| Feature              | AWS ALB Annotation                                     | GKE Gateway API Equivalent                                                                                                                              |
| :------------------- | :----------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Load Balancer Type** | `alb.ingress.kubernetes.io/scheme`                     | `Gateway.spec.gatewayClassName` (e.g., `gke-l7-global-external-managed`, `gke-l7-rilb`)                                                                   |
| **SSL Certificate**    | `alb.ingress.kubernetes.io/certificate-arn`            | `Gateway.spec.listeners.tls.certificateRefs` (to k8s Secrets or GCP Certificate Manager certs)                                                            |
| **Health Checks**      | `alb.ingress.kubernetes.io/healthcheck-path`, etc.     | `HealthCheckPolicy` CRD, `spec.targetRef` to Service. *Explicit policy often required.*                                                                     |
| **WAF / Security**     | `alb.ingress.kubernetes.io/wafv2-acl-arn`              | `GCPBackendPolicy` CRD with `spec.default.securityPolicy` (Cloud Armor), `spec.targetRef` to Service.                                                       |
| **SSL Policy**         | `alb.ingress.kubernetes.io/ssl-policy`                 | `GCPGatewayPolicy` CRD with `spec.default.sslPolicy`, `spec.targetRef` to Gateway.                                                                        |
| **HTTPS Redirect**     | `alb.ingress.kubernetes.io/actions.ssl-redirect`       | `HTTPRoute.spec.rules.filters` with `type: RequestRedirect` and `scheme: https`.                                                                        |
| **Timeouts**           | `alb.ingress.kubernetes.io/load-balancer-attributes` | `GCPBackendPolicy` CRD with `spec.default.timeoutSec`.                                                                                                  |
| **Session Affinity**   | (Various attributes)                                     | `GCPBackendPolicy` CRD with `spec.default.sessionAffinity`.                                                                                             |
| **Static IP**          | `alb.ingress.kubernetes.io/ip-address-type`          | `Gateway.spec.addresses` (referencing a static IP address resource).                                                                                    |
| **Target Type**        | `alb.ingress.kubernetes.io/target-type` (ip/instance)  | NEGs are used by default with GKE Gateway. Controlled by Service annotations `cloud.google.com/neg`.                                                  |
| **Subnets**            | `alb.ingress.kubernetes.io/subnets`                    | Determined by Cluster/VPC networking. Multi-network configs can use Network resources.                                                                  |
| **Routing Rules**      | `Ingress.spec.rules`                                   | `HTTPRoute.spec.rules` (with matches for host, path, headers, methods).                                                                                 |
| **Traffic Splitting**  | `alb.ingress.kubernetes.io/actions` (weighted forward) | `HTTPRoute.spec.rules.backendRefs` with `weight` field.                                                                                                 |
| **Header Manipulation**| `alb.ingress.kubernetes.io/actions` (fixed-response)   | `HTTPRoute.spec.rules.filters` with `type: RequestHeaderModifier` or `ResponseHeaderModifier`.                                                          |

### 3. Migration Example

#### Step A: Define the Gateway (Infrastructure Layer)
Created by Platform Admin.
```yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: external-http
  namespace: infra-ns
spec:
  gatewayClassName: gke-l7-global-external-managed # External Global Load Balancer
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - group: ""
        kind: Secret
        name: my-tls-secret
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: "true"
---
apiVersion: networking.gke.io/v1
kind: GCPGatewayPolicy
metadata:
  name: my-gateway-policy
  namespace: infra-ns
spec:
  default:
    sslPolicy: gcp-managed-ssl-policy # Reference to a GCP SSL Policy
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: external-http
