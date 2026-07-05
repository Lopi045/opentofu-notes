---
tags:
  - reference
  - command
---

# `tofu init`

## What it does

`tofu init` initializes the working directory. It must be run **before any other command** — it sets up everything tofu needs to work:

1. Downloads the providers declared in your `tofu.tf` (e.g. the Google Cloud provider)
2. Connects to the backend where state will be stored (e.g. Cloudflare R2)
3. Creates the `.terraform/` folder with the downloaded plugins

You only need to run it once per project, and again whenever you add a new provider or change the backend config.

---

## Basic usage

```sh
tofu init
```

---

## Common flags

| Flag | What it does |
|------|-------------|
| `-upgrade` | Re-download providers to their latest allowed version |
| `-reconfigure` | Reinitialize the backend even if already initialized (useful when switching backends) |
| `-migrate-state` | Move existing state to a new backend instead of starting fresh |
| `-backend-config="key=value"` | Pass backend config values at init time instead of hardcoding them in `.tf` files |

---

## What the output looks like

```
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/google versions matching "~> 5.0"...
- Installing hashicorp/google v5.21.0...
- Installed hashicorp/google v5.21.0 (signed by HashiCorp)

Tofu has been successfully initialized!
```

If the backend is configured (e.g. R2), tofu also confirms the connection to the state bucket.

---

## When to re-run init

- First time cloning a project
- After adding a new `required_providers` entry in [[tofu files/tofu.tf|tofu.tf]]
- After changing the `backend` block
- After upgrading provider versions (`-upgrade` flag)

---

## Related

- [[tofu files/tofu.tf|tofu.tf]] — declares the providers and backend that init sets up
- [[commands/tofu plan|tofu plan]] — run after init to preview changes
- [[commands/tofu apply|tofu apply]] — run after plan to apply changes
