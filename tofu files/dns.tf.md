---
tags:
  - reference
  - resource
---

# dns.tf

## What it is

`dns.tf` manages **DNS records** — the entries that translate a domain name like `example.com` into an actual IP address or cloud resource.

Without DNS records, your server exists but nobody can find it by name.

---

## How DNS works (very short version)

When someone types `example.com` in their browser:
1. Their computer asks a DNS server: "what's the IP for example.com?"
2. The DNS server looks up the record and returns an answer
3. The browser connects to that IP

Your DNS records live in a **DNS zone** managed by a DNS provider. This vault uses **Cloudflare** as the DNS provider and **GCP** for the load balancer — a common setup where your domain sits on Cloudflare but your infrastructure runs on Google Cloud.

---

## Common record types

| Type | What it does |
|------|-------------|
| `A` | Points a domain to an IPv4 address |
| `AAAA` | Points a domain to an IPv6 address |
| `CNAME` | Points a domain to another domain name |
| `MX` | Points to a mail server |
| `TXT` | Stores arbitrary text (often used for validation) |

---

## Setup — Cloudflare DNS + GCP load balancer

In this pattern:
- The **load balancer IP** is a reserved static IP in GCP (`google_compute_global_address`)
- The **DNS records** live in Cloudflare and point at that IP
- tofu manages both sides in one apply

### 1. Add the Cloudflare provider

In [[tofu files/tofu.tf|tofu.tf]]:

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}
```

The Cloudflare API token should be stored as a secret — never hardcoded.

### 2. Look up the existing Cloudflare zone

Your domain is already registered on Cloudflare. Look it up as a data source instead of recreating it:

```hcl
data "cloudflare_zone" "main" {
  name = "example.com"
}
```

### 3. Reserve a static IP on GCP

The load balancer needs a fixed IP that DNS can point at permanently:

```hcl
resource "google_compute_global_address" "main" {
  project = var.project_id
  name    = "my-app-ip"
}
```

### 4. Create the DNS records on Cloudflare

Point the root domain and `www` at the GCP load balancer IP:

```hcl
resource "cloudflare_record" "root" {
  zone_id = data.cloudflare_zone.main.id
  name    = "example.com"
  type    = "A"
  value   = google_compute_global_address.main.address
  ttl     = 1        # 1 = automatic TTL (managed by Cloudflare)
  proxied = false    # true = traffic goes through Cloudflare proxy
}

resource "cloudflare_record" "www" {
  zone_id = data.cloudflare_zone.main.id
  name    = "www"
  type    = "CNAME"
  value   = "example.com"
  ttl     = 1
  proxied = false
}
```

`proxied = true` routes traffic through Cloudflare's CDN and DDoS protection. `proxied = false` is a plain DNS record — traffic goes directly to GCP. For a GCP load balancer handling SSL itself, use `false` (the load balancer terminates TLS, not Cloudflare).

---

## Certificate validation record via Cloudflare

When [[tofu files/certificate.tf|certificate.tf]] creates a Certificate Manager DNS authorization, GCP gives you a record to add to prove domain ownership. With Cloudflare as the DNS provider, that record goes into Cloudflare instead of Cloud DNS:

```hcl
resource "cloudflare_record" "cert_validation" {
  zone_id = data.cloudflare_zone.main.id
  name    = trimsuffix(
    google_certificate_manager_dns_authorization.main.dns_resource_record[0].name,
    ".example.com."
  )
  type    = google_certificate_manager_dns_authorization.main.dns_resource_record[0].type
  value   = trimsuffix(
    google_certificate_manager_dns_authorization.main.dns_resource_record[0].data,
    "."
  )
  ttl     = 300
  proxied = false
}
```

`trimsuffix` strips the trailing dot and domain suffix that GCP adds — Cloudflare expects relative names, not fully qualified ones.

---

## HTTP → HTTPS redirect via Cloudflare

If your DNS records use `proxied = true` (traffic flows through Cloudflare), you can manage the redirect as a Cloudflare zone setting instead of handling it in the GCP load balancer:

```hcl
resource "cloudflare_zone_settings_override" "main" {
  zone_id = data.cloudflare_zone.main.id

  settings {
    always_use_https = "on"
    min_tls_version  = "1.2"
    ssl              = "full"
  }
}
```

- `always_use_https = "on"` — Cloudflare issues a 301 redirect for any HTTP request before it ever reaches GCP
- `ssl = "full"` — Cloudflare connects to GCP over HTTPS (requires a valid certificate on the origin)
- `ssl = "flexible"` — Cloudflare connects to GCP over HTTP internally (only use if you have no certificate on the origin)

> **Only works with `proxied = true`.** If records are `proxied = false` (grey cloud), traffic goes directly to GCP and Cloudflare never sees port 80 — the redirect must be in the GCP load balancer instead. See [[tofu files/load_balancer.tf|load_balancer.tf]].

Where to verify this in the dashboard: [[providers/Cloudflare|Cloudflare]] → `SSL/TLS` → `Edge Certificates` → `Always Use HTTPS`.

---

## The full picture

```
Browser → example.com
              ↓ (Cloudflare DNS, A record)
    GCP static IP (google_compute_global_address)
              ↓
    GCP Load Balancer (HTTPS, certificate attached)
              ↓
    Cloud Run service
```

---

## Related

- [[tofu files/certificate.tf|certificate.tf]] — the cert validation record is added here
- [[tofu files/load_balancer.tf|load_balancer.tf]] — the GCP load balancer the A record points at
- [[tofu files/tofu.tf|tofu.tf]] — where the Cloudflare provider is declared
- [[tofu files/data.tf|data.tf]] — the `cloudflare_zone` data source pattern
- [[providers/Google Cloud|Google Cloud]] — where to find DNS records in the GCP Console (if using Cloud DNS instead)
