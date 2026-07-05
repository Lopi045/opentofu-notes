---
tags:
  - reference
  - resource
---

# load_balancer.tf

## What it is

A **load balancer** sits in front of your Cloud Run service and provides:

- A **static public IP** that DNS can point at permanently
- **HTTPS termination** — the certificate lives here, Cloud Run receives plain HTTP internally
- **HTTP → HTTPS redirect** — any request on port 80 gets redirected to port 443 automatically
- A single entry point even if the Cloud Run URL changes

```
User (HTTP port 80)  → Load Balancer → 301 redirect → HTTPS
User (HTTPS port 443) → Load Balancer → Cloud Run service
```

---

## Components

A GCP Global HTTPS Load Balancer for Cloud Run has these parts:

| Resource | What it does |
|----------|-------------|
| `google_compute_global_address` | Reserves a static public IP |
| `google_compute_region_network_endpoint_group` | Links the load balancer to a Cloud Run service |
| `google_compute_backend_service` | Configures how traffic reaches the NEG |
| `google_compute_url_map` | HTTPS routing rules (forward to backend) |
| `google_compute_url_map` (redirect) | HTTP routing rules (redirect to HTTPS) |
| `google_compute_target_https_proxy` | HTTPS proxy — attaches the certificate |
| `google_compute_target_http_proxy` | HTTP proxy — used only for the redirect |
| `google_compute_global_forwarding_rule` ×2 | Port 443 (HTTPS) and port 80 (HTTP redirect) |

---

## Full example

### Static IP

```hcl
resource "google_compute_global_address" "main" {
  project = var.project_id
  name    = "my-app-ip"
}
```

### Serverless NEG — connect the load balancer to Cloud Run

```hcl
resource "google_compute_region_network_endpoint_group" "main" {
  project               = var.project_id
  name                  = "my-app-neg"
  network_endpoint_type = "SERVERLESS"
  region                = var.region

  cloud_run {
    service = google_cloud_run_v2_service.main.name
  }
}
```

### Backend service

```hcl
resource "google_compute_backend_service" "main" {
  project     = var.project_id
  name        = "my-app-backend"
  protocol    = "HTTPS"
  port_name   = "https"

  backend {
    group = google_compute_region_network_endpoint_group.main.id
  }
}
```

### HTTPS URL map — forward traffic to backend

```hcl
resource "google_compute_url_map" "https" {
  project         = var.project_id
  name            = "my-app-https"
  default_service = google_compute_backend_service.main.id
}
```

### HTTP URL map — redirect to HTTPS

```hcl
resource "google_compute_url_map" "http_redirect" {
  project = var.project_id
  name    = "my-app-http-redirect"

  default_url_redirect {
    https_redirect         = true
    redirect_response_code = "MOVED_PERMANENTLY_DEFAULT"  # 301
    strip_query            = false
  }
}
```

This is the redirect. Any request on port 80 hits this URL map and gets a 301 back pointing to `https://` — the path and query string are preserved.

### HTTPS proxy — attaches the certificate

```hcl
resource "google_compute_target_https_proxy" "main" {
  project          = var.project_id
  name             = "my-app-https-proxy"
  url_map          = google_compute_url_map.https.id
  certificate_map  = "//certificatemanager.googleapis.com/${google_certificate_manager_certificate_map.main.id}"
}
```

### HTTP proxy — used only for the redirect

```hcl
resource "google_compute_target_http_proxy" "redirect" {
  project = var.project_id
  name    = "my-app-http-proxy"
  url_map = google_compute_url_map.http_redirect.id
}
```

### Forwarding rules — the actual listeners

```hcl
# Port 443 — real HTTPS traffic
resource "google_compute_global_forwarding_rule" "https" {
  project               = var.project_id
  name                  = "my-app-https"
  target                = google_compute_target_https_proxy.main.id
  port_range            = "443"
  ip_address            = google_compute_global_address.main.address
  load_balancing_scheme = "EXTERNAL_MANAGED"
}

# Port 80 — redirect to HTTPS
resource "google_compute_global_forwarding_rule" "http" {
  project               = var.project_id
  name                  = "my-app-http"
  target                = google_compute_target_http_proxy.redirect.id
  port_range            = "80"
  ip_address            = google_compute_global_address.main.address
  load_balancing_scheme = "EXTERNAL_MANAGED"
}
```

Both forwarding rules share the same static IP — port 80 redirects, port 443 serves.

---

## The full picture

```
User → http://example.com (port 80)
         ↓
  google_compute_global_forwarding_rule (port 80)
         ↓
  google_compute_target_http_proxy
         ↓
  google_compute_url_map (http_redirect) → 301 to https://
         ↓
User browser follows redirect

User → https://example.com (port 443)
         ↓
  google_compute_global_forwarding_rule (port 443)
         ↓
  google_compute_target_https_proxy (certificate attached)
         ↓
  google_compute_url_map (https)
         ↓
  google_compute_backend_service
         ↓
  google_compute_region_network_endpoint_group (Serverless NEG)
         ↓
  Cloud Run service
```

---

## Required API

Compute Engine API must be enabled — see [[tofu files/api.tf|api.tf]]:

```hcl
resource "google_project_service" "compute" {
  project            = var.project_id
  service            = "compute.googleapis.com"
  disable_on_destroy = false
}
```

---

## Related

- [[tofu files/certificate.tf|certificate.tf]] — the certificate attached to the HTTPS proxy
- [[tofu files/dns.tf|dns.tf]] — A record points at `google_compute_global_address.main.address`
- [[tofu files/cloud_run.tf|cloud_run.tf]] — the backend the load balancer routes to
- [[tofu files/api.tf|api.tf]] — Compute API must be enabled
- [[providers/Google Cloud|Google Cloud]] — where to find the load balancer in the GCP Console
