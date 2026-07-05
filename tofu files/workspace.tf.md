---
tags:
  - concept
  - reference
---

# workspace

## What it is

A **workspace** in OpenTofu lets you manage multiple separate environments (dev, staging, production) using the **same config files** but with **separate state files**.

Without workspaces, you'd need a completely separate folder with duplicate `.tf` files for each environment. With workspaces, you switch context with one command and tofu keeps track of what exists in each environment independently.

---

## The problem it solves

Imagine you have `cloud_run.tf` that deploys your app. You want:
- A dev instance (cheap, scales to zero)
- A prod instance (always-on, higher limits)

If you run [[commands/tofu apply|tofu apply]] twice from the same folder, tofu would just update the same service — it wouldn't create two separate ones. Workspaces solve this by giving each environment its own state file.

---

## Basic usage

```sh
# See which workspace you're in (default: "default")
tofu workspace list

# Create a new workspace
tofu workspace new dev
tofu workspace new prod

# Switch to a workspace
tofu workspace select dev

# Apply — only affects the dev environment
tofu apply
```

Switch to prod and apply — a completely separate Cloud Run service gets created, tracked separately.

```sh
tofu workspace select prod
tofu apply
```

---

## Using the workspace name inside your config

`terraform.workspace` gives you the active workspace name. Use it in [[tofu files/locals.tf|locals.tf]] to derive environment-specific values:

```hcl
locals {
  env     = terraform.workspace
  is_prod = terraform.workspace == "prod"

  min_instances = local.is_prod ? 1 : 0
  max_instances = local.is_prod ? 20 : 3
}
```

Then consume the locals in your resources:

```hcl
resource "google_cloud_run_v2_service" "main" {
  project  = var.project_id
  name     = "my-app-${local.env}"
  location = var.region

  template {
    scaling {
      min_instance_count = local.min_instances
      max_instance_count = local.max_instances
    }
  }
}
```

- In `dev` workspace → `my-app-dev`, scales to zero
- In `prod` workspace → `my-app-prod`, always-on

Same file, different result depending on the active workspace.

---

## Full workspace config pattern

Instead of scattering `terraform.workspace == "prod" ? ... : ...` checks across your files, define all environment config in one place and look it up by workspace name:

```hcl
locals {
  workspace = terraform.workspace

  workspaces = {
    dev = {
      gcp_project       = "my-project-dev"
      gcp_region        = "europe-west1"
      image_tag         = "latest"
      min_instance      = 0
      max_instance      = 3
      greeting          = "Hello from dev!"
      cf_zone_name      = "my-zone-dev"
      hostname          = "dev.example.com"
      github_repository = "myorg/my-app"
    }
    prod = {
      gcp_project       = "my-project-prod"
      gcp_region        = "europe-west1"
      image_tag         = "stable"
      min_instance      = 1
      max_instance      = 20
      greeting          = "Hello!"
      cf_zone_name      = "my-zone-prod"
      hostname          = "example.com"
      github_repository = "myorg/my-app"
    }
  }

  config = local.workspaces[local.workspace]
}
```

Then everywhere in your config you just use `local.config.<parameter>`:

```hcl
resource "google_cloud_run_v2_service" "main" {
  project  = local.config.gcp_project
  location = local.config.gcp_region
  name     = "my-app-${local.workspace}"

  template {
    containers {
      image = "${local.config.gcp_region}-docker.pkg.dev/${local.config.gcp_project}/my-app/backend:${local.config.image_tag}"

      env {
        name  = "GREETING"
        value = local.config.greeting
      }
    }

    scaling {
      min_instance_count = local.config.min_instance
      max_instance_count = local.config.max_instance
    }
  }
}
```

If someone runs [[commands/tofu apply|tofu apply]] in the wrong workspace, they get the wrong project, wrong image tag, wrong hostname — the entire environment config is self-contained and explicit.

---

## State files per workspace

Workspaces store state in separate files under `terraform.tfstate.d/`:

```
terraform.tfstate.d/
  dev/
    terraform.tfstate
  prod/
    terraform.tfstate
```

Destroying dev doesn't touch prod, and vice versa.

---

## When NOT to use workspaces

Workspaces work well when environments are structurally identical but differ in scale. Avoid them when environments are fundamentally different — different GCP projects, different regions, different sets of resources. In those cases, separate folders or modules are cleaner.

---

## Related

- [[tofu files/locals.tf|locals.tf]] — the right place to turn `terraform.workspace` into usable values
- [[tofu files/cloud_run.tf|cloud_run.tf]] — example resource that varies by workspace
- [[Open Tofu]] — main index
