---
tags:
  - reference
  - provider
---

# Google Cloud — where to find everything in the Console

A map of every resource OpenTofu manages and where to find it in the [GCP Console](https://console.cloud.google.com) for manual inspection. Nothing here should need to be changed by hand — use tofu for that — but this is useful for verifying what was created, reading logs, or debugging.

---

## Cloud Run

**Console path:** `Cloud Run` → `Services`

| What tofu manages | Where to look |
|-------------------|--------------|
| Service (the deployed app) | Services list — click a service for details |
| Revisions (deploy history) | Service → `Revisions` tab |
| Logs | Service → `Logs` tab |
| Environment variables | Service → `Edit & Deploy New Revision` → `Variables & Secrets` |
| Scaling config | Service → `Edit & Deploy New Revision` → `Capacity` |
| IAM (who can invoke) | Service → `Security` tab → `Authentication` |
| URL | Service overview — shown at the top |

Managed by: [[tofu files/cloud_run.tf|cloud_run.tf]]

---

## Artifact Registry

**Console path:** `Artifact Registry` → `Repositories`

| What tofu manages | Where to look |
|-------------------|--------------|
| Repository | Repositories list |
| Docker images | Repository → click to browse images and tags |
| Vulnerabilities | Repository → `Vulnerabilities` column |
| Permissions | Repository → `Permissions` tab |

Managed by: [[tofu files/artifact_registry.tf|artifact_registry.tf]]

---

## Cloud DNS

**Console path:** `Network Services` → `Cloud DNS`

| What tofu manages | Where to look |
|-------------------|--------------|
| DNS zone | Zones list |
| DNS records (A, CNAME, TXT…) | Zone → `Records` tab |
| Certificate validation record | Look for a `_acme-challenge` or `_dnsauth` TXT record |

Managed by: [[tofu files/dns.tf|dns.tf]]

---

## Certificate Manager

**Console path:** `Security` → `Certificate Manager`

| What tofu manages | Where to look |
|-------------------|--------------|
| Certificates | `Certificates` tab |
| DNS authorizations | `DNS Authorizations` tab |
| Certificate status | Certificate details — should show `ACTIVE` once validated |

A certificate stays in `PROVISIONING` until the DNS validation record exists and GCP verifies it. If it's stuck, check that the DNS record in [[tofu files/dns.tf|dns.tf]] was created correctly.

Managed by: [[tofu files/certificate.tf|certificate.tf]]

---

## Load Balancing

**Console path:** `Network Services` → `Load Balancing`

| What tofu manages | Where to look |
|-------------------|--------------|
| Load balancer | Load balancers list |
| Frontend (IP + port + certificate) | Load balancer → `Frontend` tab |
| Backend (target: Cloud Run service) | Load balancer → `Backend` tab |
| URL map / routing rules | Load balancer → `Routing rules` tab |
| HTTP → HTTPS redirect | Load balancer → `Frontend` tab — look for the port 80 entry pointing to the redirect URL map |

The redirect shows up as a **second frontend** on the same load balancer — one entry for port 443 (HTTPS, certificate attached) and one for port 80 (HTTP, no certificate, redirect action). If the redirect isn't working, check that the port 80 frontend exists and its URL map has `https_redirect = true` under routing rules.

Managed by: [[tofu files/load_balancer.tf|load_balancer.tf]]

---

## APIs & Services

**Console path:** `APIs & Services` → `Enabled APIs & Services`

The full list of APIs that are currently enabled on the project. Every API declared in [[tofu files/api.tf|api.tf]] should appear here after [[commands/tofu apply|tofu apply]].

Use the search bar to find a specific API and confirm it's enabled. If a resource is failing, this is the first place to check.

---

## IAM & Admin

**Console path:** `IAM & Admin`

### IAM roles → `IAM`

Who has what permissions on the project. Service accounts created by tofu appear here with their assigned roles (e.g. `roles/run.developer`, `roles/artifactregistry.writer`).

### Service Accounts → `Service Accounts`

| What tofu manages | Where to look |
|-------------------|--------------|
| GitHub Actions service account | Listed by email: `github-actions@PROJECT.iam.gserviceaccount.com` |
| Service account roles | Click account → `Permissions` tab |

### Workload Identity Federation → `Workload Identity Federation`

| What tofu manages | Where to look |
|-------------------|--------------|
| Identity pool | Pools list |
| GitHub OIDC provider | Pool → `Providers` tab |
| Allowed principals | Pool → provider → `Connected service accounts` |

Managed by: [[tofu files/github_actions.tf|github_actions.tf]]

---

## Secret Manager

**Console path:** `Security` → `Secret Manager`

| What tofu manages | Where to look |
|-------------------|--------------|
| Secrets | Secrets list |
| Secret versions | Secret → `Versions` tab |
| Access log | Secret → `Audit Logs` |

Secrets referenced by Cloud Run via `value_source.secret_key_ref` in [[tofu files/cloud_run.tf|cloud_run.tf]] appear here.

---

## Billing & Quotas

**Console path:** `Billing` → linked billing account

Not managed by tofu, but useful to monitor costs after an apply. Key areas:
- `Reports` → cost by service and resource
- `Quotas` → if Cloud Run or API calls are being throttled

---

## Related

- [[tofu files/api.tf|api.tf]] — enables the GCP APIs that make all of the above work
- [[Setup]] — how to authenticate and configure tofu to manage all of this
- [[concepts/State|State]] — tofu's record of what it created; if something looks wrong in the Console, compare against state
