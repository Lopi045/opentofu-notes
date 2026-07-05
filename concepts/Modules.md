---
tags:
  - concept
---

# Modules

## What they are

A **module** is a reusable package of OpenTofu config — a folder of `.tf` files that can be called from other configs, just like a function you can invoke with different arguments.

Every OpenTofu project is already a module (the **root module**). When you call another folder or registry package from within it, that's a **child module**.

---

## Why use modules

Without modules, if you want to deploy the same stack (Cloud Run + Artifact Registry + DNS) for two different services, you either duplicate all the `.tf` files or put everything in one config that becomes hard to read.

Modules let you write the stack once and call it twice with different inputs:

```hcl
module "api" {
  source       = "./modules/cloud-run-service"
  service_name = "api"
  image_tag    = var.api_image_tag
  region       = var.region
}

module "worker" {
  source       = "./modules/cloud-run-service"
  service_name = "worker"
  image_tag    = var.worker_image_tag
  region       = var.region
}
```

Same infra pattern, two independent deployments.

---

## Calling a module

```hcl
module "my_service" {
  source = "./modules/cloud-run-service"   # path to the module folder

  # These become the module's input variables
  project_id   = var.project_id
  region       = var.region
  service_name = "my-app"
  image_tag    = "latest"
}
```

- `source` — where the module lives (local path, Git URL, or Terraform Registry)
- Everything else — input variables the module declares

After adding a new module call, run [[commands/tofu init|tofu init]] to install it.

---

## Module sources

| Source type | Example |
|-------------|---------|
| Local folder | `"./modules/cloud-run-service"` |
| Terraform Registry | `"hashicorp/consul/aws"` |
| GitHub | `"github.com/org/repo//modules/service"` |
| Git tag | `"git::https://github.com/org/repo.git?ref=v1.2.0"` |

Local modules are the most common for in-project reuse. Registry and Git sources are used for shared modules across projects or teams.

---

## Inside a module — inputs and outputs

A module is just a folder with `.tf` files. It exposes:

- **Inputs** via `variable` blocks — the caller must supply these
- **Outputs** via `output` blocks — the caller can read these back

```hcl
# modules/cloud-run-service/variables.tf
variable "service_name" { type = string }
variable "image_tag"    { type = string }
variable "region"       { type = string }

# modules/cloud-run-service/outputs.tf
output "url" {
  value = google_cloud_run_v2_service.main.uri
}
```

The caller reads the output with `module.<name>.<output>`:

```hcl
output "api_url" {
  value = module.api.url
}
```

---

## When to use modules

Modules add indirection — use them when the benefit is clear:

| Use a module | Don't bother |
|--------------|-------------|
| Same pattern deployed 2+ times | Used only once |
| Shared across multiple projects | Internal to one project |
| Complex enough to warrant a boundary | A few resources that fit in one file |

For a single-service project, splitting into modules is usually over-engineering. Separate `.tf` files (like [[tofu files/cloud_run.tf|cloud_run.tf]], [[tofu files/dns.tf|dns.tf]]) within one root module is enough — tofu reads them all as one config.

---

## Module versioning

When using a module from Git or the registry, pin to a specific version:

```hcl
module "service" {
  source  = "org/service/google"
  version = "~> 2.0"
}
```

Same `~>` constraint as [[concepts/Providers|providers]] — allows patch updates, blocks breaking major changes. Never leave a remote module source unpinned.

---

## Related

- [[concepts/Variables and Outputs|Variables and Outputs]] — how modules receive inputs and return outputs
- [[concepts/Providers|Providers]] — providers are inherited by child modules from the root
- [[commands/tofu init|tofu init]] — downloads module sources before first use
- [[tofu files/tofu files|tofu files]] — for simple projects, separate files beat premature modules
