name: gke-security-groups-to-ebpf-network-policy
description: Guides the translation of AWS EKS Security Groups (infrastructure-level) to GKE Dataplane V2 eBPF Network Policies (Kubernetes-native). This skill focuses on the "reverse-engineering" of AWS SG rules and their conversion into efficient, label-based K8s NetworkPolicy manifests. Use this skill when migrating security perimeters from EKS to GKE.

# EKS Security Groups to GKE eBPF Network Policies

## Instructions

### 1. The Security Paradigm Shift

*   **AWS Security Groups (EKS):** Often manage security at the ENI (Network Interface) level. Even when using "Security Groups for Pods," you are still dealing with AWS-native firewall rules that are stateful and often IP-based or SG-ID-based.
*   **GKE Dataplane V2 (GKE):** Security is decoupled from the VPC infrastructure. Rules are enforced directly in the Linux kernel using **eBPF (Cilium)**. This allows for high-performance, identity-aware (label-based) filtering that is highly scalable and provides deep observability.

### 2. Reverse-Engineering AWS Security Groups

Before writing GKE policies, you must extract the intent from AWS rules.

| AWS SG Component | Information Needed for GKE |
| :--- | :--- |
| **Inbound/Outbound Rules** | Port, Protocol, and Source/Destination. |
| **Referenced SG-ID** | What workloads (pods) does that SG-ID represent? (Find their K8s labels). |
| **CIDR Blocks** | Are these external IPs, other VPC subnets, or on-premises ranges? |
| **SecurityGroupPolicy CRD** | Which pods are these rules actually applied to? (Check the `podSelector`). |

### 3. Mapping Rules to Kubernetes NetworkPolicy

Translate the extracted intent into a `networking.k8s.io/v1` `NetworkPolicy`.

| AWS SG Rule | GKE NetworkPolicy Implementation |
| :--- | :--- |
| **Allow from another SG** | Use `podSelector` (and `namespaceSelector` if cross-namespace) matching the pods in that SG. |
| **Allow from CIDR** | Use `ipBlock`. *Warning: Dataplane V2 `ipBlock` excludes pod traffic by default.* |
| **Default Egress (Allow All)** | Do not define `Egress` rules, or explicitly allow all if the default is changed to deny. |
| **Stateful Nature** | NetworkPolicies are also stateful (return traffic is automatically allowed). |

### 4. Implementation Example: The "Three-Tier" Migration

**Scenario:** An EKS "App" pod only allows traffic from "Web" pods on port 8080 and allows all egress to a "DB" SG.

#### AWS EKS Configuration (Intent)
*   **Target:** Pods with `role: app`
*   **Ingress:** Allow TCP 8080 from pods with `role: web`
*   **Egress:** Allow all to pods with `role: db`

#### GKE Dataplane V2 Configuration (Implementation)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-isolation-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: web
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: db
```

### 5. Advanced GKE Security Features (eBPF Advantages)

*   **FQDN-based Egress:** Unlike AWS SGs which usually require IP ranges for external services, GKE Dataplane V2 allows you to restrict egress by domain name (e.g., `*.google.com`).
*   **Network Policy Logging:** Dataplane V2 provides a built-in `NetworkLogging` object to audit every "Allow" or "Deny" action in Cloud Logging.
*   **Metadata Server Protection:** Always include a rule to allow access to `169.254.169.254` on ports `80`/`8080` if using Workload Identity.

### 6. Validation & Migration Workflow

1.  **Extract:** Export AWS SG rules and `SecurityGroupPolicy` manifests.
2.  **Label:** Ensure GKE pods have labels that represent their former EKS Security Group membership.
3.  **Translate:** Use the logic above to create `NetworkPolicy` YAMLs.
4.  **Dry-Run:** Apply policies and monitor `anetd` logs in Cloud Logging.
5.  **Enforce:** Verify that unauthorized traffic is dropped by testing with `kubectl run --labels=unauthorized-label`.
