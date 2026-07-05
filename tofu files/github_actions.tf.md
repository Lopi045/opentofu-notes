---
tags:
  - concept
  - reference
---

# github_actions.tf

## What it does

`github_actions.tf` sets up the GCP side of your CI/CD pipeline — it gives GitHub Actions the permissions it needs to push Docker images to Artifact Registry and deploy to Cloud Run (or any other GCP resource) without storing long-lived credentials.

---

## The problem it solves

GitHub Actions needs to authenticate to GCP to do things like:
- Push a Docker image to Artifact Registry
- Run [[commands/tofu apply|tofu apply]] to update infrastructure
- Deploy a new revision to Cloud Run

The old way was to create a service account key (a JSON file), store it as a GitHub secret, and use it in workflows. This is risky — keys don't expire, can be leaked, and are hard to rotate.

The modern way is **Workload Identity Federation**: GCP trusts GitHub's OIDC tokens directly. No keys, no secrets to manage. GitHub proves its identity via a short-lived token that GCP verifies automatically.

---

## How it works

```
GitHub Actions workflow runs
        ↓
GitHub issues a short-lived OIDC token (proves: "I am repo X, branch Y")
        ↓
GCP verifies the token against the Workload Identity Pool
        ↓
GCP issues a temporary credential for the linked service account
        ↓
The workflow uses that credential to push images / deploy / etc.
```

No static secrets. The token is valid only for that workflow run.

---

## The resources

### 1. Workload Identity Pool

```hcl
resource "google_iam_workload_identity_pool" "github" {
  project                   = var.project_id
  workload_identity_pool_id = "github-pool"
  display_name              = "GitHub Actions pool"
  description               = "Identity pool for GitHub Actions OIDC authentication"
}
```

### 2. GitHub OIDC Provider

```hcl
resource "google_iam_workload_identity_pool_provider" "github" {
  project                            = var.project_id
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-provider"
  display_name                       = "GitHub Actions OIDC provider"
  description                        = "Allows GitHub Actions workflows to authenticate via OIDC"

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
    "attribute.ref"        = "assertion.ref"
  }
}
```

### 3. Service account for GitHub to impersonate

```hcl
resource "google_service_account" "github_actions" {
  project      = var.project_id
  account_id   = "github-actions"
  display_name = "GitHub Actions deployer"
  description  = "Impersonated by GitHub Actions workflows to deploy to GCP"
}
```

### 4. Allow the specific repo to impersonate the service account

```hcl
resource "google_service_account_iam_member" "github_wif" {
  service_account_id = google_service_account.github_actions.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/${var.github_repo}"
}
```

`var.github_repo` should be `"org/repo-name"` — this restricts access to that specific repository only.

### 5. Grant the service account what it actually needs

```hcl
# Push images to Artifact Registry
resource "google_project_iam_member" "github_artifact_registry" {
  project = var.project_id
  role    = "roles/artifactregistry.writer"
  member  = "serviceAccount:${google_service_account.github_actions.email}"
}

# Deploy to Cloud Run
resource "google_project_iam_member" "github_cloud_run" {
  project = var.project_id
  role    = "roles/run.developer"
  member  = "serviceAccount:${google_service_account.github_actions.email}"
}
```

---

## Outputs — values to push to GitHub

After [[commands/tofu apply|tofu apply]] creates these resources, declare outputs so [[commands/tofu output|tofu output]] can surface the values that need to be loaded into GitHub Variables:

```hcl
output "workload_identity_provider" {
  description = "WIF provider name — set as GH variable WIF_PROVIDER"
  value       = google_iam_workload_identity_pool_provider.github.name
}

output "service_account_email" {
  description = "Service account email — set as GH variable WIF_SERVICE_ACCOUNT"
  value       = google_service_account.github_actions.email
}

output "gcp_project_id" {
  description = "GCP project ID — set as GH variable GCP_PROJECT_ID"
  value       = var.project_id
}

output "gcp_region" {
  description = "GCP region — set as GH variable GCP_REGION"
  value       = var.region
}

output "artifact_registry_url" {
  description = "Base image URL — set as GH variable ARTIFACT_REGISTRY_URL"
  value       = "${var.region}-docker.pkg.dev/${var.project_id}/${var.repository_id}"
}

output "cloud_run_service" {
  description = "Cloud Run service name — set as GH variable CLOUD_RUN_SERVICE"
  value       = google_cloud_run_v2_service.main.name
}
```

For how to push these into GitHub with `gh variable set`, see [[Setup]] step 10.

---

## Required APIs

See [[tofu files/api.tf|api.tf]] — these must be enabled:

```hcl
resource "google_project_service" "iam_credentials" {
  project            = var.project_id
  service            = "iamcredentials.googleapis.com"
  disable_on_destroy = false
}
```

---

## Related

- [[Setup]] — step 10 covers wiring tofu outputs into GitHub Variables
- [[commands/tofu output|tofu output]] — reads output values from state after apply
- [[tofu files/artifact_registry.tf|artifact_registry.tf]] — the registry GitHub pushes images to
- [[tofu files/cloud_run.tf|cloud_run.tf]] — the service GitHub deploys to
- [[tofu files/api.tf|api.tf]] — IAM Credentials API must be enabled
