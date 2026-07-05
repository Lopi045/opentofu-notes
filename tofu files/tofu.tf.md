---
tags:
  - concept
  - reference
---

# tofu.tf

## What it is

`tofu.tf` is the project's root configuration file. It declares three things:

1. **Which version of OpenTofu** this config requires
2. **Which providers** to download and use (e.g. Google Cloud)
3. **Where to store state** — the backend (e.g. Cloudflare R2)

Every project has one. It's the first file tofu reads during [[commands/tofu init|tofu init]].

---

## The three blocks

### 1. `required_version` — lock the tofu version

```hcl
terraform {
  required_version = ">= 1.6.0"
}
```

Prevents the config from being applied with an incompatible version of tofu.

### 2. `required_providers` — declare providers

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}
```

`~> 5.0` means "any 5.x version, but not 6.0" — allows patch updates, blocks breaking major changes.

### 3. `backend` — where state lives

```hcl
terraform {
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
```

---

## Why Cloudflare R2 as the backend

State needs to live somewhere remote so that everyone on the team (and CI/CD) reads and writes the same file. Options include GCS, S3, and R2.

| | Cloudflare R2 | GCS |
|---|---|---|
| **Egress cost** | Free — no egress fees ever | Charged per GB downloaded |
| **API** | S3-compatible — works with tofu's `s3` backend | Native GCS backend |
| **Setup** | Needs `skip_*` flags for S3 compat | Cleaner config |
| **Auth** | R2 API token (Access Key + Secret) | GCP service account |

R2 is the cheaper option for state — state files are small but are read on every [[commands/tofu plan|tofu plan]] and [[commands/tofu apply|tofu apply]], so egress adds up in CI/CD pipelines.

---

## Full tofu.tf example

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

The `provider` block lives here too — it configures the default project and region used by all `google_*` resources in the config.

---

## Backend credentials

R2 credentials (Access Key ID and Secret) should **never** be hardcoded in the file. Pass them at init time via environment variables:

```sh
export AWS_ACCESS_KEY_ID="your-r2-access-key"
export AWS_SECRET_ACCESS_KEY="your-r2-secret-key"
tofu init
```

Or pass them with `-backend-config` flags:

```sh
tofu init \
  -backend-config="access_key=your-r2-access-key" \
  -backend-config="secret_key=your-r2-secret-key"
```

---

## Related

- [[commands/tofu init|tofu init]] — reads this file to download providers and connect the backend
- [[tofu files/tofu files|tofu files]] — where tofu.tf fits in the standard project layout
