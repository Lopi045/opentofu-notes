---
tags:
  - reference
  - command
---

# `tofu apply`

## What it does

`tofu apply` is the command that **actually makes changes** to your infrastructure. It reads your `.tf` files, compares them to the current state, and creates / updates / deletes resources in the cloud to match what you declared.

It's the "make it real" step.

## Basic usage

```sh
tofu apply
```

By default it runs a plan first, shows you what it's about to do, and asks for confirmation before touching anything.

```
Plan: 3 to add, 1 to change, 0 to destroy.

Do you want to perform these actions?
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

Type `yes` and tofu proceeds. Anything else cancels.

## Common flags

| Flag | What it does |
|------|-------------|
| `-auto-approve` | Skip the confirmation prompt — applies immediately |
| `-var="key=value"` | Pass an input variable inline |
| `-var-file="file.tfvars"` | Load variables from a file |
| `-target="resource.name"` | Apply only a specific resource, not everything |
| `-parallelism=N` | How many resources to create in parallel (default: 10) |
| `-plan=planfile` | Apply a previously saved plan file |

## The typical workflow

```sh
tofu init      # download providers (first time only)
tofu plan      # preview changes
tofu apply     # make changes
```

Or skip the separate plan step — `tofu apply` runs a plan internally anyway and shows it before asking for confirmation.

## What happens under the hood

1. Tofu reads all `.tf` files in the current folder
2. Reads the state file to know what already exists
3. Calls the provider APIs to check actual current state
4. Computes a diff (what needs to be created / changed / deleted)
5. Shows you the plan
6. On confirmation, executes the changes in the right order (respecting dependencies)
7. Updates the state file to reflect the new reality

## Apply with a saved plan

```sh
tofu plan -out=myplan.tfplan   # save the plan to a file
tofu apply myplan.tfplan        # apply exactly that plan, no re-planning
```

Useful in CI/CD: plan in one step, review the output, then apply the exact same plan in the next step — no risk of the plan changing between steps.

## What the output means

```
aws_s3_bucket.logs: Creating...
aws_s3_bucket.logs: Creation complete after 2s [id=my-logs-bucket]
aws_lb.main: Creating...
aws_lb.main: Still creating... [10s elapsed]
aws_lb.main: Creation complete after 45s [id=arn:aws:elasticloadbalancing:...]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.

Outputs:
load_balancer_url = "my-app-lb-123456.us-east-1.elb.amazonaws.com"
```

- `Creating...` / `Modifying...` / `Destroying...` — in progress
- `Creation complete` — done
- `Outputs` — any `output` blocks from your config are printed here

## Related

- [[commands/tofu plan|tofu plan]] — preview changes without applying
- [[commands/tofu init|tofu init]] — run this before your first apply
- [[commands/tofu destroy|tofu destroy]] — the reverse: delete everything
- [[tofu files/workspace.tf|workspace.tf]] — apply runs against the current workspace's state
