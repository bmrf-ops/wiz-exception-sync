# sync-hub

Centralized management of [Wiz](https://www.wiz.io/) security exception files (`.wiz`) across an organization's repositories.

Teams request exceptions by opening pull requests in this repo. Once approved and merged, the changes are automatically pushed to the target repository via a bot-generated PR.

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│  sync-hub (this repo)                                           │
│                                                                 │
│  1. sync-repos workflow discovers org repos daily               │
│  2. Creates repositories/<name>/.wiz from template              │
│  3. Engineers edit .wiz files via pull request                   │
│  4. Security team reviews and merges                            │
│  5. push-exceptions workflow opens PR in target repo            │
└─────────────────────────────────────────────────────────────────┘
```

### Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| **sync-repos** | Daily cron + manual | Discovers new repos via the GitHub App, scaffolds folders with `.wiz` template |
| **push-exceptions** | Push to `main` (when `repositories/**/.wiz` changes) | Pushes updated `.wiz` to target repos via PR |
| **validate** | Pull request (when `repositories/**`, `templates/**`, or `schemas/**` changes) | Lints YAML and validates `.wiz` files against JSON Schema |

## Requesting an Exception

1. Find your repository's folder under `repositories/<repo-name>/`
2. Edit the `.wiz` file with your exception
3. Open a pull request
4. The **validate** workflow checks YAML syntax and structure
5. A member of `@org/security-team` reviews the change (enforced by `CODEOWNERS`)
6. Once merged, the **push-exceptions** workflow opens a PR in your repository with the updated `.wiz` file

### `.wiz` File Format

The `.wiz` file supports exception categories under a top-level `ignore` key:

```yaml
ignore:
  vulnerabilities:
    CVE-2024-1234:
      /path/to/file:
        reason: BY_DESIGN        # BY_DESIGN | EXCEPTION | FALSE_POSITIVE
        expires: 2025-08-01T00:00:00Z

  iac:
    <rule-id>:
      /path/to/file:
        reason: BY_DESIGN

  sensitive_data:
    <rule-id>:
      /path/to/file:
        reason: FALSE_POSITIVE

  sast:
    <rule-id>:
      /path/to/file:
        reason: BY_DESIGN
        expires: 2025-12-01T00:00:00Z

  software_management:
    <rule-id>:
      /path/to/file:
        reason: BY_DESIGN
        expires: 2025-08-01T00:00:00Z
```

**Supported categories:** `vulnerabilities`, `iac`, `sensitive_data`, `sast`, `software_management`

**Supported reasons:** `BY_DESIGN`, `EXCEPTION`, `FALSE_POSITIVE`

Use `ALL` as a rule ID to apply to all findings in a category. Use glob patterns in paths (e.g., `/test/**/*`).

The full schema is defined in [`schemas/wiz.schema.json`](schemas/wiz.schema.json) and enforced by the **validate** workflow on every pull request.

## Repository Structure

```
sync-hub/
├── .github/
│   └── workflows/
│       ├── sync-repos.yml        # Discover and scaffold repos
│       ├── push-exceptions.yml   # Fan out changes to target repos
│       └── validate.yml          # PR validation checks
├── repositories/
│   └── <repo-name>/
│       ├── .wiz                  # Exception file (edit this)
│       └── META.json             # Repo identity metadata (auto-generated)
├── schemas/
│   └── wiz.schema.json           # JSON Schema for .wiz file validation
├── templates/
│   └── .wiz                      # Bootstrap template for new repos
├── CODEOWNERS                    # Enforces security team review
└── README.md
```

### META.json

Each repository folder contains an auto-generated `META.json` with the repository's numeric ID. This ensures the push workflow can track repos through renames:

```json
{
  "id": 123456789,
  "full_name": "org/repo-name",
  "scaffolded_at": "2026-03-31T06:00:00Z"
}
```

Do not edit `META.json` manually.

## Setup

### 1. Create a GitHub App

Create a GitHub App in your organization settings with these permissions:

| Permission | Access | Purpose |
|---|---|---|
| Repository: Contents | Read & Write | Push `.wiz` files to branches in target repos |
| Repository: Pull requests | Read & Write | Open/close PRs in target repos |
| Repository: Metadata | Read | Required baseline |

Set the App's webhook to inactive (not needed).

### 2. Install the App

Install the App on the repositories you want to manage. The installation scope acts as the allowlist -- only repos with the App installed will be discovered and managed.

To add a new repo to sync-hub, install the App on that repo. The next `sync-repos` run will scaffold its folder.

### 3. Configure Secrets

Add these secrets to the sync-hub repository:

| Secret | Value |
|---|---|
| `APP_ID` | The GitHub App's numeric ID |
| `APP_PRIVATE_KEY` | The App's private key (PEM format) |

### 4. Configure Branch Protection

On the sync-hub repository:

- Require pull request reviews before merging
- Require the **validate** status check to pass
- Require `CODEOWNERS` review

On target repositories (recommended):

- Branch protection on the default branch ensures the bot's PR still requires review before merge

### 5. Update CODEOWNERS

Edit `CODEOWNERS` to reference your actual org team:

```
/repositories/ @your-org/your-security-team
/templates/    @your-org/your-security-team
```

## Operational Notes

- **Template changes** do not retroactively update existing repos. The template is only used when scaffolding new repos. A separate rollout workflow can be added if needed.
- **Archived repos** are skipped during both discovery and push.
- **Forked repos** are excluded from discovery.
- **Renamed repos** are handled via the numeric `repository.id` in `META.json`.
- **Deleted repos** are not recreated. Orphan folders are harmless -- the push workflow fails gracefully and reports.
- **Concurrent merges** are handled by per-repo concurrency groups that serialize pushes to the same target.
- **Duplicate PRs** are avoided by comparing file content before opening a new PR. Stale PRs are closed automatically.
