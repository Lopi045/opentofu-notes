---
tags:
  - concept
  - reference
---

# artifact_registry.tf

## What it does

Artifact Registry is Google Cloud's managed container registry — it stores your Docker images inside GCP so that other GCP services (Cloud Run, GKE, Compute Engine) can pull them directly.

`artifact_registry.tf` declares the registry repository where your images will be pushed and managed.

---

## Why store images in Artifact Registry instead of Docker Hub

| | Artifact Registry | Docker Hub |
|---|---|---|
| **Location** | Same GCP region as your services | External, crosses the internet |
| **Pull speed** | Fast — internal network, no egress | Slower — external download |
| **Egress cost** | Free within GCP | Charged egress on every pull |
| **Auth** | GCP IAM — same system you already use | Separate credentials to manage |
| **Rate limits** | None | Pull limits on free tier |
| **Vulnerability scanning** | Built-in | Requires separate tooling |

The biggest practical reason: **iteration speed**. When Cloud Run pulls a new image on every deploy, pulling from the same region takes seconds. Pulling from Docker Hub can take 30–60 seconds or more, and counts against rate limits in CI pipelines.

---

## The resource: `google_artifact_registry_repository`

```hcl
resource "google_artifact_registry_repository" "main" {
  project       = var.project_id
  location      = var.region
  repository_id = "my-app"
  description   = "Docker repository for my-app images"
  format        = "DOCKER"

  depends_on = [google_project_service.artifactregistry]
}
```

- `location` — GCP region (e.g. `europe-west1`). Should match where your services run to avoid cross-region latency.
- `repository_id` — the name of the repo. Your image URL will be `REGION-docker.pkg.dev/PROJECT/REPO_ID/IMAGE_NAME`.
- `description` — human-readable label visible in the GCP Console. Optional but useful when you have multiple repositories.
- `format` — always `"DOCKER"` for container images.
- `depends_on` — the Artifact Registry API must be enabled first (see [[tofu files/api.tf|api.tf]]).

---

## Full image URL format

Once the repository exists, images are pushed and pulled using this URL pattern:

```
REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY_ID/IMAGE_NAME:TAG
```

Example:
```
europe-west1-docker.pkg.dev/my-project/my-app/backend:latest
```

This URL is what you pass to Cloud Run or any other GCP service when specifying which image to run.

---

## Typical artifact_registry.tf

```hcl
resource "google_artifact_registry_repository" "main" {
  project       = var.project_id
  location      = var.region
  repository_id = "my-app"
  description   = "Docker repository for my-app images"
  format        = "DOCKER"

  depends_on = [google_project_service.artifactregistry]
}
```

---

## Required API

The Artifact Registry API must be enabled before this resource can be created — see [[tofu files/api.tf|api.tf]] for how to declare it:

```hcl
resource "google_project_service" "artifactregistry" {
  project            = var.project_id
  service            = "artifactregistry.googleapis.com"
  disable_on_destroy = false
}
```

---

## Related

- [[tofu files/api.tf|api.tf]] — enabling the Artifact Registry API
