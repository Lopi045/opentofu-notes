---
tags:
  - reference
  - provider
---

# Cloudflare — where to find everything in the Dashboard

A map of every resource OpenTofu manages on Cloudflare and where to find it in the [Cloudflare Dashboard](https://dash.cloudflare.com) for manual inspection. Use tofu to make changes — this is for verification and debugging only.

---

## DNS Records

**Dashboard path:** `dash.cloudflare.com` → select your account → select your domain → `DNS` → `Records`

| What tofu manages | Where to look |
|-------------------|--------------|
| A record (domain → GCP IP) | Records list — filter by type `A` |
| CNAME record (www → root) | Records list — filter by type `CNAME` |
| Certificate validation TXT | Records list — look for `_dnsauth` or `_acme-challenge` prefix |
| TTL | Shown in the `TTL` column (Auto = managed by Cloudflare) |
| Proxied status | Orange cloud icon = proxied, grey = DNS only |

After [[commands/tofu apply|tofu apply]], new records appear here within seconds. If a record is missing, check that the `cloudflare_zone` data source in [[tofu files/dns.tf|dns.tf]] resolved to the right zone ID.

Managed by: [[tofu files/dns.tf|dns.tf]]

---

## R2 Object Storage (state backend)

**Dashboard path:** `dash.cloudflare.com` → select your account → `R2 Object Storage`

| What to look for | Where to look |
|------------------|--------------|
| State bucket | Buckets list — e.g. `my-tofu-state` |
| State file | Bucket → `Objects` tab → browse to `my-project/terraform.tfstate` |
| Bucket settings | Bucket → `Settings` tab |
| Usage & billing | R2 overview page — storage and operations used this month |

The state file is JSON — you can download and inspect it from here if something looks wrong, but **never edit it manually**. See [[concepts/State|State]] for what's inside it.

R2 is not managed by tofu itself — the bucket is created manually (see [[Setup]] step 5). tofu only reads and writes to it as a backend.

---

## HTTP → HTTPS Redirect

**Dashboard path:** select your domain → `SSL/TLS` → `Edge Certificates`

Cloudflare can enforce HTTPS at the edge — before traffic even reaches GCP — with the **Always Use HTTPS** toggle.

| Setting | Where |
|---------|-------|
| Always Use HTTPS | `SSL/TLS` → `Edge Certificates` → toggle `Always Use HTTPS` |
| Minimum TLS Version | Same page — set to TLS 1.2 minimum |
| HTTP Strict Transport Security (HSTS) | Same page → `HTTP Strict Transport Security (HSTS)` |

> **When to use this vs the GCP redirect:**
> - DNS record `proxied = false` (grey cloud) → traffic goes directly to GCP. Cloudflare's "Always Use HTTPS" has no effect — the redirect **must** be handled by the GCP load balancer. See [[tofu files/load_balancer.tf|load_balancer.tf]].
> - DNS record `proxied = true` (orange cloud) → traffic goes through Cloudflare first. "Always Use HTTPS" handles the redirect at the Cloudflare edge before it ever reaches GCP. The GCP redirect is redundant but harmless.
>
> This vault uses `proxied = false`, so the redirect lives in the GCP load balancer, not here.

If you switch to `proxied = true`, the redirect is managed in tofu via `cloudflare_zone_settings_override` in [[tofu files/dns.tf|dns.tf]].

---

## API Tokens

**Dashboard path:** `dash.cloudflare.com` → top-right profile icon → `My Profile` → `API Tokens`

| Token | Used for |
|-------|---------|
| R2 token (Access Key + Secret) | tofu backend — authenticates state reads/writes to R2 |
| Cloudflare API token | tofu provider — authenticates `cloudflare_record` resource changes |

If tofu can't connect to the R2 backend or can't create DNS records, the token is the first thing to check:
- R2 token → verify `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` env vars are set
- Cloudflare provider token → verify `var.cloudflare_api_token` is set and has DNS edit permissions on the right zone

Tokens are created here too — see [[Setup]] step 5 for R2 token setup.

---

## Zone Overview

**Dashboard path:** select your domain → `Overview`

| What to check | Where |
|---------------|-------|
| Zone ID | Bottom-right of the Overview page — needed for the `data "cloudflare_zone"` lookup |
| Account ID | Bottom-right of the Overview page — needed for R2 endpoint URL |
| Nameservers | Overview → `Cloudflare nameservers` — must be set at your registrar for Cloudflare DNS to work |
| Zone status | `Active` = nameservers pointing to Cloudflare, DNS is live |

If DNS records exist in tofu but aren't resolving, check that the zone status is `Active` and the registrar is using Cloudflare's nameservers.

---

## Related

- [[tofu files/dns.tf|dns.tf]] — creates DNS records via the Cloudflare provider
- [[tofu files/tofu.tf|tofu.tf]] — declares the Cloudflare provider and the R2 backend
- [[Setup]] — how to create the R2 bucket and API tokens
- [[concepts/State|State]] — the state file that lives in the R2 bucket
