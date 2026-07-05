---
tags:
  - concept
  - reference
---

# .tf files — how OpenTofu files work

## What is a `.tf` file?

Every file in an OpenTofu project ends in `.tf`. These are plain text files written in **HCL (HashiCorp Configuration Language)** — a format designed to be readable by both humans and machines.

OpenTofu reads **all** `.tf` files in a folder together, so it doesn't matter how you split things across files. You could write everything in one giant `main.tf` or split it into 20 files — tofu treats them all as one big config.

---

## HCL basics

HCL looks like this:

```hcl
block_type "label" "name" {
  argument = "value"
  number   = 42
  flag     = true

  nested_block {
    key = "value"
  }
}
```

- **Blocks** are the main unit — they have a type, optional labels, and a body `{ }`
- **Arguments** are `key = value` pairs inside a block
- **References** let you use a value from another resource: `aws_lb.main.dns_name`
- **Strings** use double quotes `"like this"`
- **Comments** use `#` or `//`

---

## The standard files in a tofu project

You can name files whatever you want, but most projects follow this convention:

| File | What goes in it |
|------|----------------|
| `main.tf` | The core resources — servers, databases, etc. |
| `providers.tf` | Which cloud provider(s) to use and how to authenticate |
| `variables.tf` | Input variables — the "parameters" of your config |
| `outputs.tf` | Output values — things to print after [[commands/tofu apply\|tofu apply]] (like an IP address) |
| `data.tf` | Data sources — lookups of existing resources |
| `versions.tf` | Which version of OpenTofu and providers this config requires |

Everything else (like `certificate.tf`, `dns.tf`, `load_balancer.tf`) is just a name you choose to keep related resources grouped.

---

## The four main block types

### 1. `terraform` — project settings
```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### 2. `provider` — cloud connection
```hcl
provider "aws" {
  region = "us-east-1"
}
```

### 3. `resource` — something to create
```hcl
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name"
}
```
Format: `resource "TYPE" "LOCAL_NAME"`. The local name is how you reference this resource elsewhere in your config.

### 4. `data` — something to look up
```hcl
data "aws_vpc" "default" {
  default = true
}
```
Format: `data "TYPE" "LOCAL_NAME"`. Read-only — tofu never creates or destroys these.

---

## Variables and outputs

### Input variable
```hcl
variable "environment" {
  type    = string
  default = "dev"
}
```
Use it with `var.environment`.

### Output value
```hcl
output "load_balancer_url" {
  value = aws_lb.main.dns_name
}
```
Printed to the terminal after [[commands/tofu apply|tofu apply]]. Useful for URLs, IPs, resource IDs.

---

## Referencing other resources

You can reference any resource or data source by its full address:

```
<block_type>.<type>.<name>.<attribute>
```

Examples:
- `aws_lb.main.dns_name` → the DNS name of the load balancer named `main`
- `data.aws_route53_zone.main.zone_id` → the zone ID from a data lookup
- `var.environment` → the value of an input variable
- `aws_acm_certificate.main.arn` → the ARN of a certificate

OpenTofu uses these references to figure out the **order** things need to be created in — if resource B references resource A, tofu creates A first automatically.

---

## Typical project layout

```
my-project/
├── main.tf          # core resources
├── providers.tf     # provider config
├── variables.tf     # input variables
├── outputs.tf       # output values
├── data.tf          # data source lookups
├── certificate.tf   # SSL certificate
├── dns.tf           # DNS records
├── load_balancer.tf # load balancer
└── terraform.tfstate  # state file (auto-generated, don't edit)
```

---

## Related

- [[data.tf]] — data source blocks in depth
- [[certificate.tf]] — example of a dedicated resource file
- [[dns.tf]] — another dedicated resource file
- [[load_balancer.tf]] — another dedicated resource file
- [[workspace.tf]] — managing multiple environments
- [[concepts/Infrastructure as Code|Infrastructure as Code]] — why any of this exists
