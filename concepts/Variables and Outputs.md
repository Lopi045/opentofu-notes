---
tags:
  - concept
---

# Variables and Outputs

## What they are

**Variables** are the inputs to your config — values you pass in from outside so the same `.tf` files can be reused across environments. **Outputs** are the values your config exposes after [[commands/tofu apply|tofu apply]] — useful for reading results or passing values between configs.

Together they are the public interface of a tofu config: variables in, outputs out.

---

## Input variables

Declared with a `variable` block, usually in `variables.tf`:

```hcl
variable "project_id" {
  type        = string
  description = "GCP project ID"
}

variable "region" {
  type        = string
  description = "GCP region"
  default     = "europe-west1"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod"
  }
}
```

- `type` — enforces the value type (`string`, `number`, `bool`, `list(...)`, `map(...)`, `object(...)`)
- `default` — makes the variable optional; without it the variable is required
- `description` — shown in `tofu plan` output and documentation
- `validation` — rejects bad values before any resource is touched

Reference a variable anywhere in your config with `var.<name>`:

```hcl
resource "google_cloud_run_v2_service" "main" {
  project  = var.project_id
  location = var.region
}
```

---

## How to set variable values

In order of precedence (last wins):

| Method | How |
|--------|-----|
| Default value | Defined in the `variable` block |
| `.tfvars` file | `tofu apply -var-file="prod.tfvars"` |
| `-var` flag | `tofu apply -var="project_id=my-project"` |
| Environment variable | `export TF_VAR_project_id="my-project"` |

A `terraform.tfvars` file in the project folder is loaded automatically without any flag:

```hcl
# terraform.tfvars
project_id  = "my-project-dev"
region      = "europe-west1"
environment = "dev"
```

For multi-environment projects, the [[tofu files/workspace.tf|workspace.tf]] pattern is cleaner than separate `.tfvars` files — all environment values live in one `locals` map keyed by workspace name.

---

## Output values

Declared with an `output` block, usually in `outputs.tf`:

```hcl
output "cloud_run_url" {
  value       = google_cloud_run_v2_service.main.uri
  description = "The URL of the deployed Cloud Run service"
}

output "service_account_email" {
  value       = google_service_account.github_actions.email
  description = "GitHub Actions service account email"
}
```

After [[commands/tofu apply|tofu apply]], outputs are:
- Printed to the terminal automatically
- Stored in [[concepts/State|state]] so they can be read later with [[commands/tofu output|tofu output]]
- Available to parent modules if this config is used as a module

### Sensitive outputs

Mark an output `sensitive = true` to prevent tofu from printing the value in logs:

```hcl
output "db_password" {
  value     = random_password.db.result
  sensitive = true
}
```

The value is still stored in state — `sensitive` only hides it from terminal output.

---

## The difference between variables and locals

| | `var.x` | `local.x` |
|---|---|---|
| Set by | Caller (outside the config) | You, inside the config |
| Contains logic | No | Yes |
| Overridable | Yes | No |

Use variables for values that change between environments or callers. Use locals for values derived from variables. See [[tofu files/locals.tf|locals.tf]] for the pattern.

---

## Related

- [[tofu files/locals.tf|locals.tf]] — computed values derived from variables
- [[tofu files/workspace.tf|workspace.tf]] — alternative to per-environment tfvars files
- [[commands/tofu output|tofu output]] — read output values from state after apply
- [[concepts/Modules|Modules]] — variables and outputs are how modules communicate with their callers
