name: gke-irsa-to-workload-identity
description: Guides the migration of workload authentication from AWS EKS IRSA (IAM Roles for Service Accounts) to GKE Workload Identity. This skill assists in auditing EKS trust policies, creating GKE IAM bindings, and updating application SDKs to use Google Application Default Credentials (ADC). Use this skill when migrating identity perimeters and service-to-service authentication from EKS to GKE.

# EKS IRSA to GKE Workload Identity Migration

## Instructions

### 1. Understanding the Authentication Shift

*   **AWS EKS IRSA:** Relies on an OIDC provider associated with the EKS cluster. Kubernetes Service Accounts (KSA) are annotated with an AWS Role ARN (`eks.amazonaws.com/role-arn`). A "Trust Policy" in the AWS IAM Role is configured to trust the EKS cluster's OIDC provider as a federated principal (`Federated: arn:aws:oidc-provider/...`). This policy typically includes conditions to scope the role assumption to a specific KSA name and namespace, for example: `StringEquals: "${OIDC_PROVIDER}:sub": "system:serviceaccount:NAMESPACE:KSA_NAME"`. This allows the KSA to call `sts:AssumeRoleWithWebIdentity`.
*   **GKE Workload Identity:** Utilizes Google Cloud's Workload Identity Federation. A KSA is bound to a Google Service Account (GSA) via an IAM policy binding on the GSA. The KSA in GKE is annotated with the GSA email (`iam.gke.io/gcp-service-account`). Pods running as this KSA can obtain tokens for the GSA through the GKE Metadata Server.

### 2. Migration Workflow

#### Step A: Audit EKS IRSA Setup
Identify all KSAs in EKS using IRSA and their associated AWS IAM Roles.
*   **EKS Command:** `kubectl get sa -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.metadata.annotations.eks\.amazonaws\.com/role-arn}{"\n"}{end}'`
*   **Action:** For each ARN found, inspect the AWS IAM Role's "Trust Relationships" tab in the AWS Console to understand the exact conditions and authorized KSAs. Note the permissions granted to the AWS role.

#### Step B: Create and Configure GKE Identities
For each AWS IAM Role, determine the equivalent Google Cloud permissions needed and create a corresponding GSA.

1.  **Create GSA:** Follow the principle of least privilege – create a unique GSA for each distinct set of permissions required.
    ```bash
    gcloud iam service-accounts create [GSA_NAME] --project=[PROJECT_ID]
    ```
2.  **Grant Permissions (IAM Mapping):** Assign GCP IAM roles to the GSA. This requires careful analysis:
    *   **Note:** Mapping AWS IAM policies to GCP IAM roles is not always 1:1. AWS permissions are often action-based on resources (e.g., `s3:GetObject`), while GCP roles group permissions.
    *   **Analysis:** Review the actions allowed by the AWS IAM policy and find the predefined or custom GCP IAM roles granting the equivalent capabilities (e.g., `roles/storage.objectViewer` for S3 read access). You might need to create custom GCP IAM roles.
    *   **Resources:** Consult [Compare AWS and Azure services to Google Cloud](https://cloud.google.com/free/docs/aws-azure-gcp-service-comparison) for general service equivalents.

#### Step C: Bind KSA to GSA
Allow the GKE KSA to impersonate the GSA. This binding is at the GCP IAM level.
```bash
gcloud iam service-accounts add-iam-policy-binding [GSA_NAME]@[PROJECT_ID].iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:[PROJECT_ID].svc.id.goog[[NAMESPACE]/[KSA_NAME]]"
