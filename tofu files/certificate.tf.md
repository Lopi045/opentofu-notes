---
tags:
  - reference
  - resource
---

# certificate.tf

## What it is

A `certificate.tf` file manages **SSL/TLS certificates** — the thing that makes your website use `https://` instead of `http://`.

Without a certificate, browsers show a "Not Secure" warning and traffic between the user and your server is unencrypted.

## Why it's a separate file

Certificates are usually created before the load balancer and DNS records, because both of them need to reference the certificate. Keeping it in its own file makes the dependency obvious.

---

## How it works on GCP

On GCP, certificates are managed by **Certificate Manager**. The process has two steps:

1. **DNS authorization** — GCP gives you a special DNS record to prove you own the domain
2. **Certificate** — once the DNS record exists, GCP issues and auto-renews the certificate

```hcl
resource "google_certificate_manager_dns_authorization" "main" {
  project     = var.project_id
  name        = "my-app-dns-auth"
  description = "DNS authorization for example.com"
  domain      = "example.com"
}

resource "google_certificate_manager_certificate" "main" {
  project     = var.project_id
  name        = "my-app-cert"
  description = "Managed certificate for example.com"

  managed {
    domains            = ["example.com", "www.example.com"]
    dns_authorizations = [google_certificate_manager_dns_authorization.main.id]
  }
}
```

---

## The validation dance

1. You request a certificate for `example.com`
2. GCP gives you a special DNS record to add (`google_certificate_manager_dns_authorization` exposes it as `.dns_resource_record`)
3. You add that record to your DNS zone (done in [[tofu files/dns.tf|dns.tf]])
4. GCP confirms you own the domain and issues the certificate
5. The certificate is ready to attach to a load balancer

In tofu, steps 2–4 are automated by combining `certificate.tf` with `dns.tf`:

```hcl
resource "google_dns_record_set" "cert_validation" {
  project      = var.project_id
  name         = google_certificate_manager_dns_authorization.main.dns_resource_record[0].name
  type         = google_certificate_manager_dns_authorization.main.dns_resource_record[0].type
  ttl          = 300
  managed_zone = google_dns_managed_zone.main.name

  rrdatas = [google_certificate_manager_dns_authorization.main.dns_resource_record[0].data]
}
```

---

## Required API

Certificate Manager must be enabled first — see [[tofu files/api.tf|api.tf]]:

```hcl
resource "google_project_service" "certificatemanager" {
  project            = var.project_id
  service            = "certificatemanager.googleapis.com"
  disable_on_destroy = false
}
```

---

## Related

- [[tofu files/dns.tf|dns.tf]] — adds the DNS record that validates the certificate
- [[tofu files/load_balancer.tf|load_balancer.tf]] — where the certificate gets attached so HTTPS works
- [[tofu files/api.tf|api.tf]] — Certificate Manager API must be enabled
