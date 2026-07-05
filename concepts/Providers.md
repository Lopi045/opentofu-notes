---
tags:
  - concept
---

# Providers

## What they are

A **provider** is a plugin that lets OpenTofu talk to a specific platform's API. Every resource you declare (`google_cloud_run_v2_service`, `google_dns_record_set`, etc.) belongs to a provider — the provider is what actually knows how to create, update, and delete that resource by calling the right API.

Without a provider, tofu has no idea what `google_cloud_run_v2_service` means. The provider translates your HCL config into real API calls.

---

## How providers work

```
Your .tf files
     ↓
OpenTofu core — reads config, computes plan
     ↓
Provider plugin — translates resources into API calls
     ↓
Cloud API (GCP, Cloudflare, GitHub, etc.)
     ↓
Real infrastructure
```

Providers are downloaded automatically by [[commands/tofu init|tofu init]] based on what you declare in [[tofu files/tofu.tf|tofu.tf]].

---

## Declaring a provider

In [[tofu files/tofu.tf|tofu.tf]]:

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}
```

- `required_providers` — tells tofu which plugin to download and from where
- `provider` block — configures it (default project, region, credentials)
- `version = "~> 5.0"` — allows any 5.x, blocks 6.0+ to prevent breaking changes

---

## The Google Cloud provider

The `hashicorp/google` provider covers all GCP services. Every `google_*` resource in this vault comes from it:

| Resource | Provider resource type |
|----------|----------------------|
| Cloud Run service | `google_cloud_run_v2_service` |
| Artifact Registry repo | `google_artifact_registry_repository` |
| DNS zone | `google_dns_managed_zone` |
| DNS record | `google_dns_record_set` |
| GCP API toggle | `google_project_service` |
| Certificate | `google_certificate_manager_certificate` |
| IAM binding | `google_project_iam_member` |

---

## Multiple providers

A single project can use multiple providers at once — for example, Google Cloud for infrastructure and Cloudflare for DNS:

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}
```

Each provider manages its own set of resource types — they don't interfere with each other.

---

## Related

- [[tofu files/tofu.tf|tofu.tf]] — where providers are declared and configured
- [[commands/tofu init|tofu init]] — downloads the provider plugins
- [[concepts/Resources|Resources]] — everything you create belongs to a provider
