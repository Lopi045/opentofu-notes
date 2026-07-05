---
tags:
  - guide
---

# Setup — from nothing to a working OpenTofu project

A complete walkthrough: install the tools, authenticate with GCP, configure the state backend, and run your first plan.

---

## 1. Install OpenTofu

```sh
brew install opentofu
```

Verify:

```sh
tofu --version
```

---

## 2. Install the Google Cloud CLI

```sh
brew install --cask google-cloud-sdk
```

Verify:

```sh
gcloud --version
```

---

## 3. Authenticate with Google Cloud

### 3a. Log in to your Google account

```sh
gcloud init
```

This opens a browser, asks you to pick an account, and lets you select a default GCP project and region.

### 3b. Set up Application Default Credentials (ADC)

OpenTofu doesn't use `gcloud` directly — it uses **Application Default Credentials** to call GCP APIs. Run:

```sh
gcloud auth application-default login
```

This stores a credential file locally that tofu picks up automatically. You only need to do this once (or when your credentials expire).

---

## 4. Set up a GCP project

If you don't have a project yet:

```sh
gcloud projects create my-project-id --name="My Project"
gcloud config set project my-project-id
```

If the project already exists, just set it as the active one:

```sh
gcloud config set project my-project-id
```

Enable billing on the project in the [GCP Console](https://console.cloud.google.com/billing) — most APIs require it.

---

## 5. Set up Cloudflare R2 for state storage

OpenTofu needs a remote backend to store [[concepts/State|state]]. This project uses Cloudflare R2 (S3-compatible, no egress fees). See [[tofu files/tofu.tf|tofu.tf]] for how the backend is declared.

### 5a. Create an R2 bucket

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com) → R2
2. Create a new bucket (e.g. `my-tofu-state`)
3. Note your **Account ID** from the R2 overview page

### 5b. Create an R2 API token

1. In R2 → Manage R2 API tokens → Create API token
2. Give it **Object Read & Write** permissions on the bucket you just created
3. Copy the **Access Key ID** and **Secret Access Key** — you won't see the secret again

### 5c. Store the credentials as environment variables

Add to your shell profile (`~/.zshrc` or `~/.bashrc`):

```sh
export AWS_ACCESS_KEY_ID="your-r2-access-key-id"
export AWS_SECRET_ACCESS_KEY="your-r2-secret-access-key"
```

Then reload:

```sh
source ~/.zshrc
```

---

## 6. Configure tofu.tf

Make sure your [[tofu files/tofu.tf|tofu.tf]] has the backend block pointing to your R2 bucket:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket                      = "my-tofu-state"
    key                         = "my-project/terraform.tfstate"
    region                      = "auto"
    endpoint                    = "https://<account_id>.r2.cloudflarestorage.com"
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    force_path_style            = true
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

Replace `<account_id>` with your Cloudflare Account ID.

---

## 7. Initialize the project

```sh
tofu init
```

This downloads the Google Cloud provider and connects to the R2 backend. You should see:

```
Initializing the backend...
Successfully configured the backend "s3"!

Initializing provider plugins...
- Installing hashicorp/google v5.x.x...

Tofu has been successfully initialized!
```

---

## 8. Create your first workspace

```sh
tofu workspace new dev
```

See [[commands/tofu workspace|tofu workspace]] for the full workspace workflow and [[tofu files/workspace.tf|workspace.tf]] for how to map workspace names to environment config.

---

## 9. Run your first plan

```sh
tofu plan
```

If your config references GCP APIs that aren't enabled yet, the plan will show them as resources to create (via [[tofu files/api.tf|api.tf]]). That's expected — [[commands/tofu apply|tofu apply]] will enable them.

---

## 10. Set up GitHub Actions CI/CD

This wires your repository so GitHub Actions can push images and deploy to GCP automatically. The GCP side is declared in [[tofu files/github_actions.tf|github_actions.tf]] — this step is about taking the outputs from that and loading them into GitHub.

### 10a. Install the GitHub CLI

```sh
brew install gh
gh auth login
```

### 10b. Apply the GitHub Actions config

Make sure `github_actions.tf` is in your project, then apply:

```sh
tofu workspace select prod
tofu apply
```

This creates the Workload Identity Pool, provider, and service account in GCP.

### 10c. Read the tofu outputs

```sh
tofu output
```

You should see values like:

```
artifact_registry_url         = "europe-west1-docker.pkg.dev/my-project/my-app"
cloud_run_service             = "my-app-prod"
gcp_project_id                = "my-project-prod"
gcp_region                    = "europe-west1"
service_account_email         = "github-actions@my-project-prod.iam.gserviceaccount.com"
workload_identity_provider    = "projects/123/locations/global/workloadIdentityPools/github-pool/providers/github-provider"
```

### 10d. Push values to GitHub

These are not secrets — they are configuration values. Use **GitHub Variables** (`vars.*`) for them, not Secrets:

```sh
gh variable set GCP_PROJECT_ID       --body "$(tofu output -raw gcp_project_id)"
gh variable set GCP_REGION           --body "$(tofu output -raw gcp_region)"
gh variable set WIF_PROVIDER         --body "$(tofu output -raw workload_identity_provider)"
gh variable set WIF_SERVICE_ACCOUNT  --body "$(tofu output -raw service_account_email)"
gh variable set ARTIFACT_REGISTRY_URL --body "$(tofu output -raw artifact_registry_url)"
gh variable set CLOUD_RUN_SERVICE    --body "$(tofu output -raw cloud_run_service)"
```

> **Variables vs Secrets:** GitHub Variables (`vars.NAME`) are visible in logs and can be read by anyone with repo access. GitHub Secrets (`secrets.NAME`) are encrypted and redacted from logs. Use secrets for passwords and API keys — WIF values are identifiers, not credentials, so variables are fine.

### 10e. The deploy workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write   # required for WIF OIDC token

    steps:
      - uses: actions/checkout@v4

      - uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.WIF_PROVIDER }}
          service_account: ${{ vars.WIF_SERVICE_ACCOUNT }}

      - uses: google-github-actions/setup-gcloud@v2

      - name: Configure Docker for Artifact Registry
        run: gcloud auth configure-docker ${{ vars.GCP_REGION }}-docker.pkg.dev

      - name: Build and push image
        run: |
          docker build -t ${{ vars.ARTIFACT_REGISTRY_URL }}:${{ github.sha }} .
          docker push ${{ vars.ARTIFACT_REGISTRY_URL }}:${{ github.sha }}

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy ${{ vars.CLOUD_RUN_SERVICE }} \
            --image ${{ vars.ARTIFACT_REGISTRY_URL }}:${{ github.sha }} \
            --region ${{ vars.GCP_REGION }} \
            --project ${{ vars.GCP_PROJECT_ID }}
```

---

## Quick reference — full setup sequence

```sh
# Install
brew install opentofu
brew install --cask google-cloud-sdk
brew install gh

# Auth
gcloud init
gcloud auth application-default login
gcloud config set project my-project-id
gh auth login

# R2 creds (in ~/.zshrc)
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."

# Project
tofu init
tofu workspace new dev
tofu workspace select dev
tofu plan
tofu apply

# Wire GitHub Actions (after apply)
gh variable set GCP_PROJECT_ID        --body "$(tofu output -raw gcp_project_id)"
gh variable set GCP_REGION            --body "$(tofu output -raw gcp_region)"
gh variable set WIF_PROVIDER          --body "$(tofu output -raw workload_identity_provider)"
gh variable set WIF_SERVICE_ACCOUNT   --body "$(tofu output -raw service_account_email)"
gh variable set ARTIFACT_REGISTRY_URL --body "$(tofu output -raw artifact_registry_url)"
gh variable set CLOUD_RUN_SERVICE     --body "$(tofu output -raw cloud_run_service)"
```

---

## Related

- [[tofu files/tofu.tf|tofu.tf]] — backend and provider configuration
- [[concepts/State|State]] — what the state file is and why it lives in R2
- [[commands/tofu init|tofu init]] — what init does under the hood
- [[commands/tofu workspace|tofu workspace]] — workspace commands
- [[commands/tofu output|tofu output]] — reading outputs to populate GitHub Variables
- [[tofu files/github_actions.tf|github_actions.tf]] — the GCP resources that enable WIF
- [[tofu files/api.tf|api.tf]] — enabling GCP APIs your project needs
