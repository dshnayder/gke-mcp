name: gke-vpc-cni-to-dpv2
description: Assists in migrating Kubernetes pod networking architectures from AWS EKS (VPC CNI) to GKE Dataplane V2. This skill guides the mapping of EKS IP allocation strategies (VPC-based) to GKE's VPC-native secondary ranges, ensuring routability, security, and integration with peered networks. Use this skill when a user asks about EKS-to-GKE networking migration, IP address planning, or configuring Dataplane V2.

# EKS to GKE Pod Networking Migration

## Instructions

### 1. Understand Architectural Differences

*   **AWS VPC CNI (EKS):** By default, Pods share the same VPC subnets as nodes. Each Pod gets an IP from the subnet's primary range. This often leads to IP exhaustion in the VPC. "Custom Networking" in EKS allows placing Pods in different subnets/CIDRs to mitigate this.
*   **GKE Dataplane V2 (GKE):** Uses **VPC-native networking** with **Alias IP ranges**. Nodes use the primary range of a subnet, while Pods and Services use **Secondary IP ranges**. Dataplane V2 uses eBPF (Cilium) for high-performance routing and built-in NetworkPolicy enforcement, bypassing legacy `iptables` and `kube-proxy`.

### 2. Map EKS Networking to GKE Architecture

Analyze the existing EKS networking setup to determine the target GKE configuration.

| EKS Networking Feature | GKE Dataplane V2 Equivalent | Migration Note |
| :--- | :--- | :--- |
| **Default VPC CNI** (Pods in node subnet) | **VPC-native Subnet** with Secondary Ranges | Move Pods to a dedicated secondary CIDR range to conserve primary VPC IPs. |
| **Custom Networking** (ENIConfig, Pods in separate subnets) | **User-managed Secondary Ranges** | Map the EKS pod subnets to GKE secondary ranges within the same VPC subnet. |
| **Security Groups for Pods** | **Kubernetes NetworkPolicies** + VPC Firewall Rules | Dataplane V2 enforces NetworkPolicies natively via eBPF. Use VPC firewalls for edge protection. |
| **VPC Peering / Direct Connect** | **VPC Peering / Cloud Interconnect** | GKE secondary ranges must be included in the exported/imported routes of the peering connection. |
| **IPAMD (IP Address Management)** | **GKE IPAM** (Automatic) | GKE automatically manages the assignment of /24 (or custom) slices to nodes from the secondary range. |

### 3. Plan the GKE Network (CIDR Sizing)

Before creating the GKE cluster, calculate the required CIDR sizes based on the EKS workload.

*   **Node Range (Primary):** `Number of nodes + 4 reserved IPs`. (e.g., /24 provides ~250 nodes).
*   **Pod Range (Secondary):** `Max Nodes * IP per Node`. By default, GKE allocates a /24 (256 IPs) per node for up to 110 pods.
    *   *Optimization:* Use **Flexible Pod CIDR** to reduce the allocation per node (e.g., a /26 for 32 pods/node) if EKS used small instance types or low pod densities.
*   **Service Range (Secondary):** Typically a /20 or /16, depending on the number of Services.

### 4. Implementation Steps (GCloud Example)

#### A. Create Subnet with Secondary Ranges
```bash
gcloud compute networks subnets create gke-subnet \
    --network=my-vpc \
    --range=10.0.0.0/22 \
    --secondary-range=gke-pods=10.128.0.0/14,gke-services=10.0.4.0/22 \
    --region=us-central1
```

#### B. Create GKE Cluster with Dataplane V2
```bash
gcloud container clusters create-auto my-migrated-cluster \
    --location=us-central1 \
    --network=my-vpc \
    --subnetwork=gke-subnet \
    --cluster-secondary-range-name=gke-pods \
    --services-secondary-range-name=gke-services \
    --enable-dataplane-v2
```

### 5. Advanced Migration Scenarios

*   **External VPC Peering:** Ensure the GKE pod secondary range is explicitly added to the peering configuration if "export/import subnets with public IPs" or similar flags are required for secondary range visibility.
*   **Hybrid Connectivity:** If migrating from EKS connected to on-premises via VPN/Direct Connect, ensure the GKE secondary ranges are advertised via BGP on the Cloud Router.
*   **Network Policy Migration:** Translate AWS Security Group rules (applied to pods) into `Networking.k8s.io/v1/NetworkPolicy` objects. Dataplane V2 provides built-in logging for these policies.

### 6. Validation & Verification
*   **Connectivity:** Test Pod-to-Pod and Pod-to-External-Service (e.g., Cloud SQL) connectivity.
*   **IP Utilization:** Use `gcloud container clusters describe` to check CIDR usage.
*   **Network Policy:** Verify enforcement using `kubectl get networkpolicy` and checking Dataplane V2 logs in Cloud Logging.
