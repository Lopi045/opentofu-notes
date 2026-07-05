---
tags:
  - reference
  - command
---

# `tofu plan`

## What it does

`tofu plan` shows you exactly what OpenTofu **would** do if you ran [[commands/tofu apply|tofu apply]] — without actually doing anything. No resources are created, changed, or deleted.

It's a dry run. A preview. The "check before you act" step.

## Basic usage

```sh
tofu plan
```

Tofu reads your `.tf` files, compares them against the state file, and prints a list of planned changes.

## Reading the output

```
OpenTofu will perform the following actions:

  # google_artifact_registry_repository.main will be created
  + resource "google_artifact_registry_repository" "main" {
      + repository_id = "my-app"
      + format        = "DOCKER"
      + id            = (known after apply)
    }

  # google_cloud_run_v2_service.main will be updated in-place
  ~ resource "google_cloud_run_v2_service" "main" {
      ~ template[0].scaling[0].max_instance_count = 10 -> 20
        id = "projects/my-project/locations/europe-west1/services/my-app"
    }

  # google_dns_record_set.old will be destroyed
  - resource "google_dns_record_set" "old" {
      - name = "old.example.com."
    }

Plan: 1 to add, 1 to change, 1 to destroy.
```

| Symbol | Meaning |
|--------|---------|
| `+` | Will be created |
| `-` | Will be destroyed |
| `~` | Will be updated in-place |
| `-/+` | Will be destroyed and recreated (can't update in-place) |
| `(known after apply)` | Value doesn't exist yet — will be set once the resource is created |

The `-/+` case is the most important to watch for — it means a change you consider minor (like renaming a resource) will actually **delete and recreate** it, with potential downtime.

## Common flags

| Flag | What it does |
|------|-------------|
| `-out=file` | Save the plan to a file so you can apply it exactly later |
| `-var="key=value"` | Pass an input variable inline |
| `-var-file="file.tfvars"` | Load variables from a file |
| `-target="resource.name"` | Plan only a specific resource |
| `-destroy` | Show what a full [[commands/tofu destroy\|tofu destroy]] would do |
| `-detailed-exitcode` | Exit code 2 = changes exist, useful in CI scripts |

## Saving a plan for later

```sh
tofu plan -out=myplan.tfplan
tofu apply myplan.tfplan
```

When you apply a saved plan, tofu skips re-planning and applies **exactly** what was in the file. This is the standard pattern in CI/CD pipelines — plan in one step, apply in the next, with a human review in between.

## What tofu checks during a plan

1. Reads all `.tf` files in the current folder
2. Reads the state file (what tofu thinks exists)
3. Calls provider APIs to refresh actual current state
4. Computes the diff and prints it

If something was changed outside of tofu (manually in the GCP Console, for example), the plan will detect that drift and account for it.

## Why always run plan first

- Catch mistakes before they hit prod
- Spot unexpected `-/+` replacements that would cause downtime
- Confirm the right resources are being targeted
- Required in team environments so others can review changes before apply

## Related

- [[commands/tofu apply|tofu apply]] — the command that actually executes the plan
- [[commands/tofu init|tofu init]] — must be run before plan works
- [[commands/tofu destroy|tofu destroy]] — use `tofu plan -destroy` to preview a destroy
