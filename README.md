# OpenTofu Notes

An Obsidian vault for learning and reference — covers OpenTofu (the open-source Terraform fork) with a focus on GCP and Cloudflare deployments.

## Structure

| Folder | Contents |
|--------|----------|
| `concepts/` | Core IaC concepts: state, providers, variables, modules, workspaces |
| `commands/` | CLI reference for `tofu init`, `plan`, `apply`, `destroy`, `output`, `workspace` |
| `tofu files/` | Per-file guides: `tofu.tf`, `cloud_run.tf`, `github_actions.tf`, `dns.tf`, and more |
| `providers/` | Provider-specific notes for Google Cloud and Cloudflare |
| `_templates/` | Obsidian templates for Concept, Command, and Resource notes |

## Usage

Open the folder as an Obsidian vault. Start from `Open Tofu.md` (the index note).

The Templates plugin is pre-configured to use `_templates/` — enable it in Obsidian settings to insert templates with a slash command.
