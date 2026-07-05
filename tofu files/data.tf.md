---
tags:
  - reference
  - concept
---

# data.tf

## What it is

`data.tf` contains **data sources** — read-only lookups of things that already exist, rather than things you're creating.

Think of it as the difference between:
- `resource` block → "create this thing"
- `data` block → "look up this existing thing and give me its details"

---

## Why you need it

Sometimes infrastructure already exists and you don't want to recreate it — you just need to reference it. Common examples:
- A DNS zone that was created manually or outside this tofu project
- An existing VPC or network
- Secrets stored in Secret Manager
- Project metadata like project number

---

## Basic example — look up an existing DNS zone

```hcl
data "google_dns_managed_zone" "main" {
  project = var.project_id
  name    = "my-app-zone"
}

# Now use it elsewhere
resource "google_dns_record_set" "app" {
  project      = var.project_id
  managed_zone = data.google_dns_managed_zone.main.name
  name         = "app.example.com."
  type         = "A"
  ttl          = 300
  rrdatas      = [google_compute_global_address.main.address]
}
```

You didn't create the DNS zone — it already existed. You just fetched its name so you could use it.

---

## Example — look up the current GCP project

Useful when you need the project number (not just the ID) for IAM bindings or API calls:

```hcl
data "google_project" "main" {
  project_id = var.project_id
}

# Use the project number
resource "google_project_iam_member" "example" {
  project = var.project_id
  role    = "roles/run.invoker"
  member  = "serviceAccount:${data.google_project.main.number}-compute@developer.gserviceaccount.com"
}
```

---

## Example — look up a secret from Secret Manager

```hcl
data "google_secret_manager_secret_version" "db_url" {
  project = var.project_id
  secret  = "database-url"
  version = "latest"
}

# Use the secret value
resource "google_cloud_run_v2_service" "main" {
  ...
  template {
    containers {
      env {
        name  = "DATABASE_URL"
        value = data.google_secret_manager_secret_version.db_url.secret_data
      }
    }
  }
}
```

---

## Key idea

Data sources don't create or modify anything. They just read. If you run [[commands/tofu destroy|tofu destroy]], data sources are untouched — only `resource` blocks get destroyed.

---

## Related

- [[tofu files/certificate.tf|certificate.tf]] — data sources used to look up DNS zones for cert validation
- [[tofu files/dns.tf|dns.tf]] — data sources provide the zone name needed to create DNS records
