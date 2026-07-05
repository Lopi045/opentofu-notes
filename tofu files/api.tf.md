---
tags:
  - concept
  - reference
---

# api.tf

## What it does

In Google Cloud, most services are off by default — you have to explicitly **enable the API** for a service before you can use it (e.g., you can't create a Cloud DNS zone if the DNS API isn't enabled).

`api.tf` is where you declare which GCP APIs your project needs. OpenTofu enables them automatically during [[commands/tofu apply|tofu apply]], so you never have to touch the Google Cloud Console manually.

---

## The resource: `google_project_service`

```hcl
resource "google_project_service" "dns" {
  project = var.project_id
  service = "dns.googleapis.com"

  disable_on_destroy = false
}
```

- `project` — the GCP project ID where the API should be enabled
- `service` — the API endpoint string (always ends in `.googleapis.com`)
- `disable_on_destroy` — whether tofu should disable the API when you run [[commands/tofu destroy|tofu destroy]]. Usually set to `false` to avoid accidentally breaking other things that depend on the same API.

---

## Common APIs to enable

| Service | API string |
|---------|-----------|
| Cloud DNS | `dns.googleapis.com` |
| Compute Engine | `compute.googleapis.com` |
| Cloud Run | `run.googleapis.com` |
| Cloud SQL | `sqladmin.googleapis.com` |
| Secret Manager | `secretmanager.googleapis.com` |
| Cloud Storage | `storage.googleapis.com` |
| Container Registry | `containerregistry.googleapis.com` |
| Artifact Registry | `artifactregistry.googleapis.com` |
| Certificate Manager | `certificatemanager.googleapis.com` |
| Cloud Load Balancing | `compute.googleapis.com` (same as Compute) |

---

## Typical api.tf

```hcl
resource "google_project_service" "dns" {
  project            = var.project_id
  service            = "dns.googleapis.com"
  disable_on_destroy = false
}

resource "google_project_service" "compute" {
  project            = var.project_id
  service            = "compute.googleapis.com"
  disable_on_destroy = false
}

resource "google_project_service" "secretmanager" {
  project            = var.project_id
  service            = "secretmanager.googleapis.com"
  disable_on_destroy = false
}
```

---

## Why this matters

Other resources in your config will **implicitly depend** on the API being enabled. If you reference `google_dns_managed_zone` but haven't enabled `dns.googleapis.com`, the apply will fail with a permission error.

To make the dependency explicit and ensure the API is enabled before the resource is created, use `depends_on`:

```hcl
resource "google_dns_managed_zone" "main" {
  name     = "my-zone"
  dns_name = "example.com."

  depends_on = [google_project_service.dns]
}
```

---

## Related

- [[tofu files/dns.tf|dns.tf]] — needs DNS API enabled
- [[tofu files/certificate.tf|certificate.tf]] — needs Certificate Manager API
- [[tofu files/load_balancer.tf|load_balancer.tf]] — needs Compute API
