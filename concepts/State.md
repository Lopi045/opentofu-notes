---
tags:
  - concept
---

# State

## What it is

State is a record of everything OpenTofu has created. It lives in a file called `terraform.tfstate` and maps every resource in your `.tf` files to its real counterpart in GCP — its ID, attributes, and current values.

Without state, tofu has no memory. It wouldn't know whether a resource already exists, what its current configuration is, or whether it needs to be updated or left alone.

---

## Why it exists

When you run [[commands/tofu plan|tofu plan]], tofu needs to answer: "what's the difference between what I declared and what already exists?" To do that it needs three things:

1. Your `.tf` files — what you *want*
2. The real cloud — what *actually exists* (fetched live from GCP APIs)
3. **State** — what tofu *thinks* exists, including values only known after creation (like auto-generated IDs and IPs)

State is the bridge. It stores values that can't be derived from your config alone — like the ID GCP assigned to a Cloud Run service after it was created, or the IP address of a load balancer.

---

## What's inside the state file

The state file is JSON. You should never edit it manually, but it's useful to know what's in it:

```json
{
  "resources": [
    {
      "type": "google_cloud_run_v2_service",
      "name": "main",
      "instances": [
        {
          "attributes": {
            "id": "projects/my-project/locations/europe-west1/services/my-app",
            "name": "my-app",
            "uri": "https://my-app-abc123-ew.a.run.app"
          }
        }
      ]
    }
  ]
}
```

Every resource in your config has a corresponding entry here after [[commands/tofu apply|tofu apply]] runs.

---

## Where state lives

### Local (default)

By default, state is stored in `terraform.tfstate` in the same folder as your `.tf` files. Fine for solo projects, bad for teams — only one person can have the file, and it's easy to lose.

### Remote backend (recommended)

For any real project, state should live in a remote backend so the whole team and CI/CD share the same file. This vault uses **Cloudflare R2** as the backend — configured in [[tofu files/tofu.tf|tofu.tf]].

```
Your machine   →  tofu plan/apply  →  reads/writes  →  R2 bucket (terraform.tfstate)
Team member    →  tofu plan/apply  →  reads/writes  →  same R2 bucket
GitHub Actions →  tofu plan/apply  →  reads/writes  →  same R2 bucket
```

Everyone always works from the same source of truth.

---

## State drift

Drift happens when the real world diverges from what's in state — for example, someone manually changes a Cloud Run environment variable in the GCP Console without going through tofu.

On the next [[commands/tofu plan|tofu plan]], tofu detects the difference between state and reality and proposes changes to bring things back in line. This is why you should **always manage infrastructure through tofu**, not the console.

---

## State locking

When one person (or CI job) is running [[commands/tofu apply|tofu apply]], the state file is **locked** — nobody else can apply at the same time. This prevents two simultaneous applies from corrupting the state by writing conflicting changes.

Remote backends like R2 support locking automatically.

---

## One state per workspace

Each [[tofu files/workspace.tf|workspace]] has its own independent state file:

```
terraform.tfstate.d/
  dev/
    terraform.tfstate   ← dev environment
  prod/
    terraform.tfstate   ← prod environment
```

Destroying resources in `dev` never touches `prod`'s state.

---

## Related

- [[tofu files/tofu.tf|tofu.tf]] — configures where state is stored (the backend)
- [[tofu files/workspace.tf|workspace.tf]] — each workspace has its own state file
- [[commands/tofu plan|tofu plan]] — reads state to compute what needs to change
- [[commands/tofu apply|tofu apply]] — updates state after making changes
