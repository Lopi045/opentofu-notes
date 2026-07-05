---
tags:
  - concept
  - reference
---

# locals.tf

## What it is

`locals.tf` defines **local values** — named expressions you compute once and reuse throughout your config. They are never exposed as inputs or outputs; they exist purely to avoid repetition and centralize logic.

Think of them as constants or computed variables inside your tofu config.

---

## Basic syntax

```hcl
locals {
  environment  = var.environment
  app_name     = "my-app"
  full_name    = "${local.app_name}-${local.environment}"
  is_prod      = var.environment == "prod"
}
```

Reference a local anywhere in your config with `local.<name>`.

---

## Why use locals instead of just using variables directly

| | `var.x` | `local.x` |
|---|---|---|
| Set by | The caller (tfvars, CLI) | You, inside the config |
| Can contain logic | No — just a value | Yes — any expression |
| Can reference other resources | No | Yes |
| Reusable across files | Yes | Yes |

Locals are for **derived values** — things you compute from variables, resource attributes, or other locals. If the same expression appears in three places, it belongs in a local.

---

## Common patterns

### Naming convention

Centralise the naming logic so every resource follows the same pattern:

```hcl
locals {
  prefix = "${var.project_name}-${var.environment}"
}

resource "google_cloud_run_v2_service" "main" {
  name = "${local.prefix}-api"
  ...
}

resource "google_artifact_registry_repository" "main" {
  repository_id = "${local.prefix}-images"
  ...
}
```

Change the naming convention in one place, every resource updates.

### Environment-based logic

```hcl
locals {
  is_prod         = var.environment == "prod"
  min_instances   = local.is_prod ? 1 : 0
  max_instances   = local.is_prod ? 20 : 3
}
```

In prod, keep at least one instance running (no cold starts). In dev/staging, scale to zero to save cost.

### Building the Artifact Registry image URL

Rather than repeating the full image URL in every resource that needs it:

```hcl
locals {
  image_url = "${var.region}-docker.pkg.dev/${var.project_id}/${var.repository_id}/${var.app_name}:${var.image_tag}"
}

resource "google_cloud_run_v2_service" "main" {
  template {
    containers {
      image = local.image_url
    }
  }
}
```

---

## Key rules

- Locals can reference other locals — but not circularly.
- Locals can reference `var.*`, `resource.*`, and `data.*` — any value available in the config.
- Locals are evaluated lazily — tofu only computes them when something references them.
- They cannot be overridden from outside (unlike variables).

---

## Related

- [[tofu files/tofu files|tofu files]] — where locals fit in the standard project layout
- [[tofu files/cloud_run.tf|cloud_run.tf]] — consumes locals for image URL and scaling values
- [[tofu files/artifact_registry.tf|artifact_registry.tf]] — repository ID often built from a local prefix
