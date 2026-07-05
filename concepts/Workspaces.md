---
tags:
  - concept
---

# Workspaces

## What they are

A **workspace** is an isolated environment with its own independent [[concepts/State|state file]]. Every workspace runs against the same `.tf` config files, but tracks completely separate infrastructure.

The practical result: one codebase, many environments.

---

## The problem they solve

Without workspaces, deploying the same config to dev and prod requires either:
- **Duplicate folders** — copy all your `.tf` files, maintain them separately, drift over time
- **Manual variable swapping** — change `project_id` by hand before each apply, easy to forget

Workspaces solve this cleanly: switch context with one command, tofu handles the rest.

---

## How state isolation works

Each workspace stores its state in a separate file:

```
terraform.tfstate.d/
  dev/
    terraform.tfstate   ← dev resources tracked here
  prod/
    terraform.tfstate   ← prod resources tracked here
```

A [[commands/tofu apply|tofu apply]] in `dev` reads and writes only `dev`'s state file. `prod` is completely untouched — even if both environments have a Cloud Run service named `my-app`, tofu sees them as independent resources.

---

## Reading the active workspace in config

Inside any `.tf` file, `terraform.workspace` returns the name of the currently active workspace. The standard pattern is to map workspace names to full environment configs in [[tofu files/locals.tf|locals.tf]]:

```hcl
locals {
  workspace = terraform.workspace

  workspaces = {
    dev  = { gcp_project = "my-project-dev",  min_instance = 0, ... }
    prod = { gcp_project = "my-project-prod", min_instance = 1, ... }
  }

  config = local.workspaces[local.workspace]
}
```

Then every resource just uses `local.config.gcp_project`, `local.config.min_instance`, etc. See [[tofu files/workspace.tf|workspace.tf]] for the full example.

---

## The commands

| Command | What it does |
|---------|-------------|
| `tofu workspace list` | Show all workspaces, `*` marks the active one |
| `tofu workspace new dev` | Create a workspace and switch to it |
| `tofu workspace select prod` | Switch to an existing workspace |
| `tofu workspace show` | Print the current workspace name |
| `tofu workspace delete dev` | Delete a workspace (destroy resources first) |

Full reference: [[commands/tofu workspace|tofu workspace]].

---

## When NOT to use workspaces

Workspaces are for environments that are **structurally identical** but differ in scale or config values. Avoid them when environments are fundamentally different — different sets of resources, different regions, or different GCP organizations. In those cases, separate folders with separate backends are cleaner.

---

## Related

- [[tofu files/workspace.tf|workspace.tf]] — how to use `terraform.workspace` in config, with a full per-environment config example
- [[commands/tofu workspace|tofu workspace]] — all workspace subcommands
- [[tofu files/locals.tf|locals.tf]] — where workspace config maps live
- [[concepts/State|State]] — each workspace has its own state file
