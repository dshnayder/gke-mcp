name: gke-security-groups-to-ebpf-network-policy
description: Guides the translation of AWS EKS Security Groups (infrastructure-level) to GKE Dataplane V2 eBPF Network Policies (Kubernetes-native). This skill focuses on the "reverse-engineering" of AWS SG rules and their conversion into efficient, label-based K8s NetworkPolicy manifests. Use this skill when migrating security perimeters from EKS to GKE.

# EKS Security Groups to GKE eBPF Network Policies

## Instructions

### 1. The Security Paradigm Shift

*   **AWS Security Groups (EKS):** Operate at the instance (EC2) or Elastic Network Interface (ENI) level. They are stateful firewalls. When used with EKS:
    *   **Node-Level SG:** Applied to the worker nodes, affecting all pods on those nodes for traffic leaving/entering the node.
    *   **Security Groups for Pods:** Allows associating specific SGs directly to Pods or Deployments, often using the `SecurityGroupPolicy` Custom Resource Definition (CRD). This provides more granular, Pod-specific rules.
    *   Rules are often based on IP CIDRs, Port/Protocol, or references to other SG IDs.

*   **GKE Dataplane V2 (GKE):** Security is enforced within the Kubernetes cluster using Network Policies, implemented via eBPF (Cilium) directly in the Linux kernel on each node.
    *   **Kubernetes-Native:** Policies are defined using the `networking.k8s.io/v1/NetworkPolicy` API.
    *   **Label-Based:** Rules primarily use Kubernetes labels and selectors to identify source and destination Pods and Namespaces.
    *   **eBPF Advantages:** High performance, no `iptables` overhead, native support for features like FQDN-based egress, and rich Network Policy Logging.

### 2. Reverse-Engineering AWS Security Groups

To translate rules, you need to understand the *intent* behind the AWS SG configuration:

1.  **Identify Scope:** Determine if SGs are applied at the Node level or to specific Pods via "Security Groups for Pods" and the `SecurityGroupPolicy` CRD.
2.  **Analyze Rules:** For each relevant SG:
    *   **Direction:** Inbound or Outbound.
    *   **Protocol/Port:** e.g., TCP 8080.
    *   **Source/Destination:** This is the crucial part.
        *   **CIDR Blocks:** Identify what these represent (e.g., External IPs, other VPC Subnets, On-premises, Node IPs).
        *   **Referenced SG IDs:** Determine which *Kubernetes workloads* (Pods, Deployments) are effectively part of the referenced SG. This requires mapping the SG ID back to Pods, likely through instance tags or `SecurityGroupPolicy` selectors. **The key is to find the corresponding Kubernetes labels.**
3.  **SecurityGroupPolicy CRD:** If used, inspect the `podSelector` and `namespaceSelector` within the `SecurityGroupPolicy` manifest to see which specific Pods the SG rules were intended to target.

| AWS SG Component          | Information Needed for GKE NetworkPolicy                                 |
| :------------------------ | :----------------------------------------------------------------------- |
| Inbound/Outbound Rules    | Port, Protocol, Source/Destination intent.                               |
| Referenced SG-ID (Source/Dest) | The Kubernetes labels/namespaces of Pods associated with that SG-ID.     |
| CIDR Blocks (Source/Dest) | The nature of the network range (External, Internal VPC, On-prem, etc.). |
| `SecurityGroupPolicy` CRD | The `podSelector` and `namespaceSelector` defining the target Pods.      |

### 3. Mapping Rules to Kubernetes NetworkPolicy

Translate the intent into `networking.k8s.io/v1/NetworkPolicy` YAML:

| AWS SG Rule Intent              | GKE NetworkPolicy Implementation                                                                                                                                                                                                                                                                                         |
| :------------------------------ | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Allow from/to another SG        | Use `podSelector` (and potentially `namespaceSelector` if cross-namespace) in the `from` (ingress) or `to` (egress) clauses, matching the labels of the Pods formerly grouped by the SG ID.                                                                                                                              |
| Allow from/to a CIDR          | Use `ipBlock` in the `from` or `to` clauses. **CRITICAL:** In GKE Dataplane V2, `ipBlock` rules do *not* apply to Pod-to-Pod traffic within the cluster. They only apply to traffic external to the cluster (e.g., Internet, VMs outside GKE, On-prem). To allow/deny Pod traffic, you MUST use `podSelector`/`namespaceSelector`. |
| Default Egress (Allow All)    | By default, Kubernetes Network Policies are allow-all until a policy selects a pod. If no Egress policy is defined for a selected pod, all egress is denied. To replicate "Allow All" egress, you need an explicit egress rule.                                                                                              |
| Stateful Nature                 | Kubernetes NetworkPolicies are also stateful. If an ingress connection is allowed, the return traffic for that connection is automatically permitted, no separate egress rule for the response is needed.                                                                                                              |

**Explicit Egress Example (Allow All):**
```yaml
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
      # Note: This allows all NON-POD traffic.
      # To also allow all pod traffic, add selectors:
    - podSelector: {} # All pods in the same namespace
    - namespaceSelector: {} # All pods in all namespaces
