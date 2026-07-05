---
tags:
  - concept
---

# Infrastructure as Code

## What is infrastructure?

Before talking about IaC, it helps to know what "infrastructure" means in this context.

Infrastructure is everything your app needs to run that isn't the app itself:
- Servers (virtual machines in the cloud)
- Databases
- Networks and firewalls
- Storage buckets
- Load balancers
- DNS records

Basically: all the "plumbing" behind your application.

---

## The old way — clicking around

Traditionally, a developer or sysadmin would:
1. Log into AWS / Azure / Google Cloud
2. Click through a web interface to create a server
3. Manually configure it — install software, open ports, set permissions
4. Repeat for every environment (dev, staging, production)

This works, but it has serious problems:
- **Hard to reproduce** — you can't remember exactly what you clicked 6 months ago
- **Easy to make mistakes** — one wrong setting and things break
- **Not version-controlled** — no history of what changed, when, and why
- **Slow** — spinning up a new environment takes hours of manual work
- **"Snowflake" servers** — every server ends up slightly different, which causes bugs that only appear in prod

---

## The IaC way — writing config files

Infrastructure as Code means you describe your infrastructure in text files, exactly like writing code.

Instead of clicking "create a server with 2 CPUs and 4GB RAM", you write:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

Then a tool (like OpenTofu) reads that file and creates the actual server for you.

---

## Why this is much better

### It's reproducible
Run the same files → get the exact same infrastructure, every time. Dev, staging, and production can be identical.

### It's version-controlled
The config files live in Git. You can see who changed what, when, and why. You can roll back a bad change.

### It's fast
Creating 10 servers takes the same effort as creating 1 — just run the tool again.

### It's reviewable
Before applying changes, you can see a preview of exactly what will be created, changed, or deleted. No surprises.

### It documents itself
The files are the documentation. You don't need a separate wiki explaining your infrastructure — the config files show it directly.

---

## How OpenTofu works specifically

1. **You write `.tf` files** — HCL (HashiCorp Configuration Language) files that describe what you want
2. **[[commands/tofu plan|tofu plan]]** — OpenTofu compares your files against what actually exists in the cloud and shows you what it would change
3. **[[commands/tofu apply|tofu apply]]** — OpenTofu makes those changes happen for real
4. **State file** — OpenTofu keeps a record (`terraform.tfstate`) of what it created, so it knows what to update or delete next time

```
Your .tf files  →  tofu plan  →  review changes  →  tofu apply  →  real cloud infra
```

---

## Why provider-agnostic matters

AWS, Google Cloud, and Azure all have different APIs and different web consoles. If you set up everything manually on AWS and then want to move to Google Cloud, you have to start from scratch.

With OpenTofu, you change the **provider** at the top of your config and most of your infrastructure description stays the same. You're not locked in.

---

## Quick analogy

Think of IaC like a recipe vs. cooking from memory:

| Without IaC | With IaC |
|-------------|----------|
| Cook from memory every time | Follow a written recipe |
| Results vary | Same result every time |
| Hard to share with others | Anyone can follow the recipe |
| No history of changes | Recipe can be versioned in Git |
| Takes hours to replicate | Run the tool, done in minutes |

---

## One folder per project

Each project gets its own folder with its own `.tf` files and its own state file. If a repo has two projects, they live in two separate folders — they never share state.

```
my-repo/
├── project-a/
│   ├── main.tf
│   ├── providers.tf
│   └── terraform.tfstate   ← project A's state, independent
└── project-b/
    ├── main.tf
    ├── providers.tf
    └── terraform.tfstate   ← project B's state, independent
```

Running [[commands/tofu apply|tofu apply]] inside `project-a/` only touches project A's infrastructure. Project B is completely unaffected. No special config needed — just separate folders.

---

## Related

- [[Open Tofu]] — the tool this vault is about
- [[concepts/State|State]] — how OpenTofu tracks what it created
- [[concepts/Providers|Providers]] — how OpenTofu talks to different clouds
