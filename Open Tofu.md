---
tags:
  - index
---

# Open Tofu

Infrastructure-as-code: define cloud resources in HCL config files, tofu applies them. Provider-agnostic — switch clouds without reconfiguring everything.

> **Provider scope:** all examples and resource references in this vault use **Google Cloud (GCP)** as the reference provider.

> **One folder per project.** Each project in a repo gets its own folder with its own `.tf` files and its own state file. If a repo has two projects, it has two separate tofu folders — they never share state.

- [[Setup]] — install tools and configure everything from scratch
- Docs: https://opentofu.org/
- Lightning repo (examples): https://github.com/paulcdejean/lightning

---

## Provider reference

- [[providers/Google Cloud|Google Cloud]] — where to find every tofu-managed resource in the GCP Console
- [[providers/Cloudflare|Cloudflare]] — where to find DNS records, R2 state bucket, and API tokens in the Cloudflare Dashboard

## Concepts

- [[concepts/Infrastructure as Code|Infrastructure as Code]] — what IaC is and why it exists
- [[concepts/State|State]] — how tofu tracks what's deployed
- [[concepts/Providers|Providers]] — plugins that talk to cloud APIs
- [[concepts/Resources|Resources]] — the actual infra objects you declare
- [[concepts/Variables and Outputs|Variables and Outputs]] — parameterize configs
- [[concepts/Modules|Modules]] — reusable config packages
- [[concepts/Workspaces|Workspaces]] — manage multiple envs from one config

## Tofu files

- [[tofu files/tofu files|tofu files]] — how .tf files work, HCL syntax, standard project layout
- [[tofu files/tofu.tf|tofu.tf]] — required version, providers, and backend (Cloudflare R2 state storage)
- [[tofu files/api.tf|api.tf]] — enable GCP APIs/services so other resources can work
- [[tofu files/artifact_registry.tf|artifact_registry.tf]] — Docker image registry on GCP, faster pulls and no rate limits vs Docker Hub
- [[tofu files/cloud_run.tf|cloud_run.tf]] — run the project as a serverless container on GCP
- [[tofu files/certificate.tf|certificate.tf]] — SSL/TLS certificates
- [[tofu files/data.tf|data.tf]] — read-only lookups of existing resources
- [[tofu files/dns.tf|dns.tf]] — DNS records
- [[tofu files/github_actions.tf|github_actions.tf]] — Workload Identity Federation so GitHub Actions can deploy to GCP without stored keys
- [[tofu files/load_balancer.tf|load_balancer.tf]] — load balancer, target groups, listeners
- [[tofu files/locals.tf|locals.tf]] — computed values and logic shared across the config
- [[tofu files/workspace.tf|workspace.tf]] — managing dev/staging/prod with separate state

## Commands

- [[commands/tofu init|tofu init]] — initialize providers and backend
- [[commands/tofu plan|tofu plan]] — preview changes before applying
- [[commands/tofu apply|tofu apply]] — apply changes to infrastructure
- [[commands/tofu destroy|tofu destroy]] — destroy all resources in the current workspace
- [[commands/tofu workspace|tofu workspace]] — manage dev/staging/prod workspaces
- [[commands/tofu output|tofu output]] — read output values from state after apply

## 12-factor app

A checklist of 12 rules for building apps that are easy to deploy, scale, and maintain in the cloud. Created by the Heroku team. The idea: if your app follows these rules, it will work cleanly on any cloud platform without surprises.

| # | Factor | In plain words |
|---|--------|---------------|
| 1 | **Codebase** | One repo, many deploys (dev, staging, prod all run the same code) |
| 2 | **Dependencies** | Declare all dependencies explicitly — never assume something is pre-installed |
| 3 | **Config** | Store config (passwords, URLs, ports) in environment variables, not in code |
| 4 | **Backing services** | Treat databases, queues, email as external attachments you can swap out |
| 5 | **Build / Release / Run** | Keep these three stages separate — don't patch code while it's running |
| 6 | **Processes** | The app is one or more stateless processes — don't store anything locally between requests |
| 7 | **Port binding** | The app serves itself on a port — it doesn't need a web server installed around it |
| 8 | **Concurrency** | Scale by running more processes, not by making one process bigger |
| 9 | **Disposability** | Processes can start fast and stop cleanly — makes deploys and crashes less painful |
| 10 | **Dev / prod parity** | Keep dev, staging, and production as identical as possible |
| 11 | **Logs** | Write logs to stdout — let the platform collect and store them |
| 12 | **Admin processes** | Run one-off tasks (migrations, scripts) the same way as the app itself |

**Why it matters for OpenTofu:** factors 3, 4, and 10 are directly relevant — tofu manages the infrastructure that makes those rules possible (env vars via secrets managers, swappable backing services, identical environments).

