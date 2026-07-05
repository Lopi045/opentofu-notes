---
tags:
  - reference
  - command
---

# `tofu workspace`

## What it does

`tofu workspace` manages **workspaces** — separate state environments that let you run the same config against dev, staging, and prod independently. Each workspace has its own state file, so changes in one don't affect the others.

See [[tofu files/workspace.tf|workspace.tf]] for how to use `terraform.workspace` inside your config to vary resources per environment.

---

## Subcommands

### `list` — show all workspaces

```sh
tofu workspace list
```

Output:
```
  default
* dev
  prod
```

The `*` marks the currently active workspace.

---

### `new` — create a workspace

```sh
tofu workspace new dev
tofu workspace new prod
```

Creates the workspace and switches to it immediately. The first [[commands/tofu apply|tofu apply]] in a new workspace creates a fresh set of resources — it doesn't touch other workspaces.

---

### `select` — switch to a workspace

```sh
tofu workspace select prod
```

All subsequent commands ([[commands/tofu plan|tofu plan]], [[commands/tofu apply|tofu apply]], [[commands/tofu destroy|tofu destroy]]) now operate against `prod`'s state. Always check which workspace you're on before applying.

---

### `show` — print the current workspace name

```sh
tofu workspace show
```

Output:
```
prod
```

Useful in scripts to confirm the active workspace before running destructive commands.

---

### `delete` — remove a workspace

```sh
tofu workspace delete dev
```

Deletes the workspace and its state file. **This is destructive** — any resources tracked in that state are orphaned (tofu forgets about them but they still exist in GCP). Destroy the resources first with [[commands/tofu destroy|tofu destroy]], then delete the workspace.

---

## Typical workflow

```sh
# First time setup
tofu workspace new dev
tofu workspace new prod

# Deploy to dev
tofu workspace select dev
tofu plan
tofu apply

# Deploy to prod
tofu workspace select prod
tofu plan
tofu apply
```

---

## Related

- [[tofu files/workspace.tf|workspace.tf]] — how to read `terraform.workspace` in config and map it to per-env settings
- [[commands/tofu apply|tofu apply]] — runs against the currently selected workspace
- [[commands/tofu destroy|tofu destroy]] — destroys resources in the currently selected workspace only
