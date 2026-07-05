---
tags:
  - reference
  - command
---

# `tofu output`

## What it does

`tofu output` prints the values of all `output` blocks defined in your config. It reads directly from [[concepts/State|state]] — no API calls, no planning. Whatever was captured during the last [[commands/tofu apply|tofu apply]] is what you get.

Useful for grabbing values you need after a deploy: a Cloud Run URL, a load balancer IP, a service account email.

---

## Basic usage

```sh
# Print all outputs
tofu output

# Print a specific output
tofu output cloud_run_url
```

Example output:

```
cloud_run_url = "https://my-app-abc123-ew.a.run.app"
service_account_email = "github-actions@my-project.iam.gserviceaccount.com"
artifact_registry_url = "europe-west1-docker.pkg.dev/my-project/my-app"
```

---

## Common flags

| Flag | What it does |
|------|-------------|
| `-json` | Print all outputs as a JSON object — useful in scripts |
| `-raw` | Print a single output value with no quotes or formatting — useful in shell scripts |
| `-no-color` | Strip color codes from output |

### `-json` example

```sh
tofu output -json
```

```json
{
  "cloud_run_url": {
    "value": "https://my-app-abc123-ew.a.run.app",
    "type": "string"
  },
  "service_account_email": {
    "value": "github-actions@my-project.iam.gserviceaccount.com",
    "type": "string"
  }
}
```

### `-raw` example

```sh
tofu output -raw cloud_run_url
```

```
https://my-app-abc123-ew.a.run.app
```

No quotes, no label — just the raw value. Pipe it directly into another command:

```sh
curl $(tofu output -raw cloud_run_url)/health
```

---

## Defining outputs in your config

Outputs are declared with `output` blocks, usually in `outputs.tf`:

```hcl
output "cloud_run_url" {
  value       = google_cloud_run_v2_service.main.uri
  description = "The URL of the deployed Cloud Run service"
}

output "service_account_email" {
  value       = google_service_account.github_actions.email
  description = "Email of the GitHub Actions service account"
}

output "workload_identity_provider" {
  value       = google_iam_workload_identity_pool_provider.github.name
  description = "WIF provider name to use in GitHub Actions workflows"
}
```

After [[commands/tofu apply|tofu apply]], these are printed automatically at the end and stored in [[concepts/State|state]] for `tofu output` to read later.

---

## Common use cases in CI/CD

In GitHub Actions, capture tofu outputs and use them in subsequent steps:

```yaml
- name: Get tofu outputs
  id: tofu
  run: |
    echo "cloud_run_url=$(tofu output -raw cloud_run_url)" >> $GITHUB_OUTPUT
    echo "wif_provider=$(tofu output -raw workload_identity_provider)" >> $GITHUB_OUTPUT

- name: Use the URL
  run: curl ${{ steps.tofu.outputs.cloud_run_url }}/health
```

---

## Related

- [[concepts/State|State]] — outputs are read from state, not from the cloud
- [[commands/tofu apply|tofu apply]] — outputs are captured and printed at the end of apply
- [[tofu files/github_actions.tf|github_actions.tf]] — typical outputs for WIF provider and service account email
