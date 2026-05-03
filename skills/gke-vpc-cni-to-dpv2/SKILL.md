name: gke-vpc-cni-to-dpv2
description: Assists in migrating Kubernetes pod networking architectures from AWS EKS (VPC CNI) to GKE Dataplane V2. This skill guides the mapping of EKS IP allocation strategies (VPC-based) to GKE's VPC-native secondary ranges, ensuring routability, security, and integration with peered networks. Use this skill when a user asks about EKS-to-GKE networking migration, IP address planning, or configuring Dataplane V2.

# EKS to GKE Pod Networking Migration

## Instructions

### 1. Understand Architectural Differences

*   **AWS VPC CNI (EKS):** By default, Pods share the same VPC subnets as nodes. Each Pod gets an IP from the subnet's primary range. This often leads to IP exhaustion in the VPC. "Custom Networking" in EKS, using the `ENIConfig` custom resource, allows nodes to source Pod IPs from different subnets/CIDRs to mitigate this.
*   **GKE Dataplane V2 (GKE):** Uses **VPC-native networking** with **Alias IP ranges**. Nodes use the primary range of a subnet, while Pods and Services use **Secondary IP ranges**. Dataplane V2 is based on Cilium and eBPF, providing high-performance routing, built-in NetworkPolicy enforcement, and replacing legacy `iptables` and `kube-proxy`. This generally leads to better performance, scalability, and observability.

### 2. Map EKS Networking to GKE Architecture

Analyze the existing EKS networking setup to determine the target GKE configuration.

| EKS Networking Feature                                | GKE Dataplane V2 Equivalent                      | Migration Note                                                                                                                                                                                             |
| :---------------------------------------------------- | :----------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Default VPC CNI** (Pods in node subnet)             | **VPC-native Subnet** with Secondary Ranges    | Move Pods to a dedicated secondary CIDR range to conserve primary VPC IPs.                                                                                                                                 |
| **Custom Networking** (ENIConfig, Pods in separate subnets) | **User-managed Secondary Ranges**                | Map the EKS pod subnets to GKE secondary ranges within the same VPC subnet.                                                                                                                                |
| **Security Groups for Pods / Nodes**                  | **Kubernetes NetworkPolicies** + VPC Firewall Rules | Dataplane V2 enforces NetworkPolicies natively via eBPF for Pod-to-Pod traffic. Use GCP VPC firewalls for node-level ingress/egress and to control traffic between subnets and external resources. |
| **VPC Peering / Direct Connect**                      | **VPC Peering / Cloud Interconnect**             | GKE secondary ranges must be included in exported/imported routes. Cloud Router typically handles this for Interconnect. Ensure peering configurations exchange all subnet routes.                         |
| **IPAMD (IP Address Management)**                     | **GKE IPAM** (Automatic)                         | GKE automatically manages the assignment of CIDR slices to nodes from the Pod secondary range.                                                                                                           |

### 3. Plan the GKE Network (CIDR Sizing)

Before creating the GKE cluster, calculate the required CIDR sizes based on the EKS workload.

*   **IP Address Uniqueness:** Critical: GKE VPC-native clusters require Pod and Service secondary ranges to be unique within the *entire* VPC network and any VPCs directly peered with it. Overlapping CIDRs between clusters or other subnets in the peered topology are not supported.
*   **Node Range (Primary):** `Number of nodes + 4 reserved IPs`. (e.g., /24 provides ~250 nodes).
*   **Pod Range (Secondary):** `Max Nodes * IP per Node`. By default, GKE allocates a /24 (256 IPs) per node for up to 110 pods.
    *   *Optimization:* Use **Flexible Pod CIDR** to reduce the allocation per node (e.g., a /26 for 64 pods/node) if EKS used small instance types or low pod densities. This is highly recommended to conserve IP space. See [Configure maximum Pods per node](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr).
*   **Service Range (Secondary):** Typically a /20 or /16, depending on the number of Services.

### 4. Implementation Steps (GCloud Example)

#### A. Create Subnet with Secondary Ranges
```bash
gcloud compute networks subnets create gke-subnet \
    --network=my-vpc \
    --range=10.0.0.0/22 \
    --secondary-range=gke-pods=10.128.0.0/14,gke-services=10.0.4.0/22 \
    --region=us-central1
