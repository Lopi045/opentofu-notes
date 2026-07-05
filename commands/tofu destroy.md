---
tags:
  - reference
  - command
---

# `tofu destroy`

## What it does

`tofu destroy` deletes every resource tracked in the current [[concepts/State|state file]]. It is the reverse of [[commands/tofu apply|tofu apply]] — instead of creating and updating, it tears everything down.

It only destroys what tofu knows about. Resources created outside of tofu are unaffected.

---

## Basic usage

```sh
tofu destroy
```

Like [[commands/tofu apply|tofu apply]], it shows a plan first and asks for confirmation before doing anything:

```
OpenTofu will perform the following actions:

  # google_cloud_run_v2_service.main will be destroyed
  - resource "google_cloud_run_v2_service" "main" {
      - name     = "my-app"
      - location = "europe-west1"
    }

  # google_artifact_registry_repository.main will be destroyed
  - resource "google_artifact_registry_repository" "main" {
      - repository_id = "my-app"
    }

Plan: 0 to add, 0 to change, 5 to destroy.

Do you really want to destroy all resources?
  Only 'yes' will be accepted to confirm.

  Enter a value:
```

Type `yes` to proceed. Anything else cancels.

---

## Common flags

| Flag | What it does |
|------|-------------|
| `-auto-approve` | Skip the confirmation prompt — destroys immediately |
| `-target="resource.name"` | Destroy only a specific resource, not everything |
| `-var="key=value"` | Pass an input variable inline |
| `-var-file="file.tfvars"` | Load variables from a file |

---

## Preview before destroying

Run [[commands/tofu plan|tofu plan]] with the `-destroy` flag to see what would be deleted without actually doing it:

```sh
tofu plan -destroy
```

Same output as `tofu destroy` but no confirmation prompt, no changes made. Useful for double-checking before a real destroy.

---

## Destroy a single resource

Use `-target` to destroy one resource without touching everything else:

```sh
tofu destroy -target="google_cloud_run_v2_service.main"
```

Useful in development when you want to recreate one resource without tearing down the whole environment. Use with care — partial destroys can leave state inconsistent if other resources depend on the one you're deleting.

---

## Destroy is workspace-scoped

`tofu destroy` only destroys resources in the **currently active [[concepts/Workspaces|workspace]]**. Switching to `dev` and destroying won't touch `prod`:

```sh
tofu workspace select dev
tofu destroy   # only dev resources are affected
```

---

## When to use it

- Tearing down a dev or staging environment to save cost
- Cleaning up after a failed deploy
- Removing a project entirely
- Deleting a workspace (destroy resources first, then `tofu workspace delete`)

---

## Related

- [[commands/tofu plan|tofu plan]] — use `tofu plan -destroy` to preview before destroying
- [[commands/tofu apply|tofu apply]] — the opposite operation
- [[commands/tofu workspace|tofu workspace]] — destroy is scoped to the active workspace
- [[concepts/State|State]] — tofu only destroys what's in state
