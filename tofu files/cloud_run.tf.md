---
tags:
  - concept
  - reference
---

# cloud_run.tf

## What it does

Cloud Run is GCP's serverless container platform — you give it a Docker image and it runs it for you. No servers to manage, no Kubernetes to configure. GCP handles scaling, availability, and infrastructure.

`cloud_run.tf` is where you declare the Cloud Run service: which image to run, how much CPU/memory to give it, what environment variables to inject, and who can access it.

---

## The resource: `google_cloud_run_v2_service`

```hcl
resource "google_cloud_run_v2_service" "main" {
  project  = var.project_id
  name     = "my-app"
  location = var.region
  description = "Main application service"

  template {
    containers {
      image = "${var.region}-docker.pkg.dev/${var.project_id}/${var.repository_id}/my-app:latest"

      resources {
        limits = {
          cpu    = "1"
          memory = "512Mi"
        }
      }

      env {
        name  = "ENV"
        value = var.environment
      }
    }

    scaling {
      min_instance_count = 0
      max_instance_count = 10
    }
  }

  depends_on = [google_project_service.run]
}
```

---

## Key arguments

| Argument | Description |
|----------|-------------|
| `name` | The service name — also part of the auto-generated URL |
| `location` | GCP region where the service runs. Should match your Artifact Registry region |
| `description` | Human-readable label visible in the GCP Console |
| `template.containers.image` | The Docker image to run — pulled from Artifact Registry |
| `resources.limits.cpu` | CPU allocated per container instance (e.g. `"1"`, `"2"`) |
| `resources.limits.memory` | Memory per instance (e.g. `"512Mi"`, `"1Gi"`) |
| `scaling.min_instance_count` | Minimum running instances. `0` means scale to zero when idle (saves cost) |
| `scaling.max_instance_count` | Maximum instances Cloud Run can scale up to under load |

---

## Injecting secrets as environment variables

Hardcoding secrets in `env { value = "..." }` is wrong — use Secret Manager instead:

```hcl
env {
  name = "DATABASE_URL"
  value_source {
    secret_key_ref {
      secret  = google_secret_manager_secret.db_url.secret_id
      version = "latest"
    }
  }
}
```

---

## Making the service publicly accessible

By default Cloud Run services are private. To allow unauthenticated access (public web app):

```hcl
resource "google_cloud_run_v2_service_iam_member" "public" {
  project  = var.project_id
  location = var.region
  name     = google_cloud_run_v2_service.main.name
  role     = "roles/run.invoker"
  member   = "allUsers"
}
```

---

## Scale to zero

`min_instance_count = 0` means when no traffic is coming in, Cloud Run shuts down all instances — you pay nothing. When a request arrives, it cold-starts a new instance (usually under a second for small images).

Set `min_instance_count = 1` if you can't tolerate any cold start latency (always-on, always billing).

---

## Required API

The Cloud Run API must be enabled first — see [[tofu files/api.tf|api.tf]]:

```hcl
resource "google_project_service" "run" {
  project            = var.project_id
  service            = "run.googleapis.com"
  disable_on_destroy = false
}
```

---

## Related

- [[tofu files/artifact_registry.tf|artifact_registry.tf]] — where the Docker image comes from
- [[tofu files/api.tf|api.tf]] — Cloud Run API must be enabled
- [[tofu files/load_balancer.tf|load_balancer.tf]] — put a load balancer in front for custom domains and HTTPS
