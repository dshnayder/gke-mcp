name: gke-irsa-to-workload-identity
description: Guides the migration of workload authentication from AWS EKS IRSA (IAM Roles for Service Accounts) to GKE Workload Identity. This skill assists in auditing EKS trust policies, creating GKE IAM bindings, and updating application SDKs to use Google Application Default Credentials (ADC). Use this skill when migrating identity perimeters and service-to-service authentication from EKS to GKE.

# EKS IRSA to GKE Workload Identity Migration

## Instructions

### 1. Understanding the Authentication Shift

*   **AWS EKS IRSA:** Relies on an OIDC provider associated with the EKS cluster. Kubernetes Service Accounts (KSA) are annotated with an AWS Role ARN (`eks.amazonaws.com/role-arn`). A "Trust Policy" in AWS IAM allows the OIDC provider to assume the role for a specific KSA.
*   **GKE Workload Identity:** Relies on Google Cloud's Workload Identity Federation. A KSA is bound to a Google Service Account (GSA) via an IAM policy binding on the GSA. The KSA is annotated with the GSA email (`iam.gke.io/gcp-service-account`).

### 2. Migration Workflow

#### Step A: Audit EKS IRSA Setup
Identify all KSAs in EKS that use IRSA and their associated AWS IAM Roles.
*   **EKS Command:** `kubectl get sa -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.metadata.annotations.eks\.amazonaws\.com/role-arn}{"\n"}{end}'`
*   **Action:** Inspect the AWS IAM Role's "Trust Relationships" to confirm which KSAs are authorized.

#### Step B: Create and Configure GKE Identities
For each AWS IAM Role, determine the equivalent Google Cloud permissions needed and create a corresponding GSA.

1.  **Create GSA:**
    ```bash
    gcloud iam service-accounts create [GSA_NAME] --project=[PROJECT_ID]
    ```
2.  **Grant Permissions:** Assign GCP IAM roles to the GSA based on the AWS Role's policy (e.g., `roles/storage.objectViewer` for S3 read access equivalent).

#### Step C: Bind KSA to GSA
Allow the GKE KSA to impersonate the GSA.
```bash
gcloud iam service-accounts add-iam-policy-binding [GSA_NAME]@[PROJECT_ID].iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:[PROJECT_ID].svc.id.goog[[NAMESPACE]/[KSA_NAME]]"
```

#### Step D: Annotate the KSA in GKE
```bash
kubectl annotate serviceaccount [KSA_NAME] \
    --namespace [NAMESPACE] \
    iam.gke.io/gcp-service-account=[GSA_NAME]@[PROJECT_ID].iam.gserviceaccount.com
```

### 3. Application SDK Updates

Migration is not complete until the application code is updated to look for Google credentials.

*   **AWS SDKs:** Often configured to look for the `AWS_WEB_IDENTITY_TOKEN_FILE` and `AWS_ROLE_ARN` environment variables (injected by IRSA).
*   **Google Cloud SDKs:** Use **Application Default Credentials (ADC)**. When running on GKE with Workload Identity, the SDK automatically fetches tokens from the GKE Metadata Server (`169.254.169.254`).
*   **Action:** Remove AWS-specific credential logic or environment variable overrides. Ensure the latest Google Cloud client libraries are used.

### 4. Comparison Table

| Feature | AWS EKS (IRSA) | GKE Workload Identity |
| :--- | :--- | :--- |
| **Trust Anchor** | OIDC Provider | Workload Identity Pool (`[PROJECT].svc.id.goog`) |
| **KSA Annotation** | `eks.amazonaws.com/role-arn` | `iam.gke.io/gcp-service-account` |
| **IAM Policy Member** | `Principal: Federated [OIDC_ARN]` | `serviceAccount:[PROJECT].svc.id.goog[[NS]/[KSA]]` |
| **Token Injection** | Projected volume mount (token file) | GKE Metadata Server (HTTP request) |
| **Credential Discovery** | `WebIdentityTokenCredentials` | `Application Default Credentials (ADC)` |

### 5. Troubleshooting & Validation

*   **Verify Binding:** Use `gcloud iam service-accounts get-iam-policy` to ensure the `workloadIdentityUser` binding is correct.
*   **Test Metadata Access:** From inside a pod, test if you can reach the metadata server:
    ```bash
    curl -H "Metadata-Flavor: Google" http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/identity
    ```
*   **Check Logs:** Look for "permission denied" errors in the application logs, which may indicate a missing IAM role on the GSA or a misconfigured binding.
