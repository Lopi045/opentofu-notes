---
tags:
  - concept
---

# Resources

## What they are

A **resource** is the basic building block of an OpenTofu config — it represents one piece of infrastructure that tofu will create, manage, and eventually destroy. A Cloud Run service is a resource. A DNS record is a resource. A certificate is a resource.

Every `resource` block in your `.tf` files maps one-to-one to something real in the cloud.

---

## The resource block

```hcl
resource "google_cloud_run_v2_service" "main" {
  project  = var.project_id
  name     = "my-app"
  location = var.region
}
```

The block has three parts:

| Part | Example | What it is |
|------|---------|-----------|
| Block type | `resource` | Always `resource` |
| Resource type | `"google_cloud_run_v2_service"` | What kind of thing to create — comes from the [[concepts/Providers|provider]] |
| Local name | `"main"` | Your name for it — used to reference it elsewhere in config |

The resource type tells tofu (and its provider) exactly which API to call. The local name is only meaningful inside your config.

---

## Referencing a resource

Once a resource is declared, other resources can reference its attributes:

```hcl
resource "google_artifact_registry_repository" "main" {
  repository_id = "my-app"
  location      = var.region
}

resource "google_cloud_run_v2_service" "app" {
  template {
    containers {
      # Reference the registry's location and ID directly
      image = "${google_artifact_registry_repository.main.location}-docker.pkg.dev/..."
    }
  }
}
```

Format: `<resource_type>.<local_name>.<attribute>`

OpenTofu uses these references to automatically determine the **creation order** — if Cloud Run references Artifact Registry, tofu creates Artifact Registry first, without you having to say so explicitly.

---

## What tofu does with resources

| Operation | When it happens |
|-----------|----------------|
| **Create** (`+`) | Resource is in config but not in [[concepts/State|state]] |
| **Update** (`~`) | Resource exists but config changed — can be done in-place |
| **Replace** (`-/+`) | Config changed in a way that requires destroy + recreate |
| **Destroy** (`-`) | Resource is in state but removed from config |

The `-/+` replace case is the most dangerous — a change that looks minor (like renaming a resource) can cause downtime if the cloud provider doesn't support in-place updates for that attribute. [[commands/tofu plan|tofu plan]] always shows you which case applies before you commit.

---

## `depends_on` — explicit dependencies

tofu infers most dependencies from references. But sometimes a resource depends on another without directly referencing it — for example, a Cloud Run service needs the Cloud Run API enabled before it can be created:

```hcl
resource "google_cloud_run_v2_service" "main" {
  ...
  depends_on = [google_project_service.run]
}
```

This forces tofu to create `google_project_service.run` first, even though the service block doesn't reference it directly. See [[tofu files/api.tf|api.tf]] for the full pattern.

---

## Related

- [[concepts/Providers|Providers]] — resource types come from providers
- [[concepts/State|State]] — tofu tracks every resource in the state file
- [[tofu files/tofu files|tofu files]] — how resources are organized across `.tf` files
- [[commands/tofu plan|tofu plan]] — shows what will happen to each resource before apply
- [[commands/tofu apply|tofu apply]] — creates/updates/destroys resources for real
