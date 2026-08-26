# 🚀 Bitbucket Server to GitHub Repository Migration Pipeline

> A GitHub Actions–based solution for migrating **Bitbucket Server** repositories to **GitHub** at scale. Supports parallel migrations, pre-migration checks, post-migration validation, multiple storage backends, and GitHub Data Residency.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Migration Tool](https://img.shields.io/badge/Tool-gh--bbs2gh-181717.svg)](https://github.com/github/gh-bbs2gh)
[![Platform](https://img.shields.io/badge/Platform-Bash%20%7C%20PowerShell-blue.svg)](https://github.com/github/gh-bbs2gh)

---

## 📋 Table of Contents

- [Introduction](#-introduction)
- [Limitations](#️-limitations)
- [Prerequisites](#️-prerequisites)
- [Initial Setup](#-initial-setup)
- [Quick Start](#-quick-start)
- [Repository names vs slugs](#-repository-names-vs-slugs)
- [Large files / Git LFS](#-large-files--git-lfs)

---

## 📖 Introduction

Migrating repositories from Bitbucket Server to GitHub is a multi-stage process that includes readiness validation, parallel repository migration, and post-migration verification. When applied across hundreds or thousands of repositories, this process becomes difficult to coordinate, error-prone, and hard to scale using ad-hoc commands.

This toolkit addresses those challenges through a staged, CSV-driven execution model. Each stage runs independently, produces machine-readable output, and can be executed from the command line or embedded inside a CI/CD pipeline. Failures in individual repositories are isolated - they do not block the remaining batch.

---

## ⚠️ Limitations

- **Repository Migration Size Limits**
The [GitHub Enterprise Importer](https://github.com/github/gh-gei) has the following size limits:

| Item | Maximum Size |
|------|--------------|
| Repository archive | ~40 GiB |
| Single file (during migration) | 400 MiB |
| Single file (after migration) | 100 MiB (larger files must use Git LFS) |
| Single commit | 2 GiB |

- **What Gets Migrated:**
  - Git repository content (all files)
  - Complete commit history
  - All branches and tags
  - Commit metadata (authors, dates, messages, SHAs)

- **Maximum Concurrency:**
  - The default concurrency is **3**. Increase with `--max-concurrent` up to 10.
  - The actual repository migration runs on **GitHub's backend services**, not on the local machine. The script only polls migration status at regular intervals.

 - **Github Hosted runners timeout:**
   - It is recommended to run GitHub Actions on self-hosted runners, where the job timeout can be configured to 0, allowing long-running migrations to complete without interruption. By contrast, GitHub-hosted runners are limited to a maximum job runtime of 360 minutes.

- **Track Long-Running Migrations:**
  - If a migration is taking longer than expected, monitor progress directly using the GitHub CLI: [GitHub Migration Monitor](https://github.com/mona-actions/gh-migration-monitor)
    ```bash
    gh extension install mona-actions/gh-migration-monitor
    gh migration monitor
    ```

---

## ⚙️ Prerequisites

- organization owner role for the destination organization in GitHub, or an organization owner must grant the migrator role.
- You must also have required permissions and access to your Bitbucket Server instance:
  -  Admin or super admin permissions.
  -  If your Bitbucket Server instance runs Linux, SFTP access to the instance, using a supported SSH private key.
  -  If your Bitbucket Server instance runs Windows, file sharing (SMB) access to the instance.
-  **GitHub Data Residency**, set `TARGET_API_URL` to the regional GitHub API endpoint (for example, https://api.tenant.ghe.com). 
- **GitHub PAT** (`GH_PAT`) with scopes `admin:org`, and `workflow`
- **Bitbucket Server:**
  - **Bitbucket Server URL:** `BBS_BASE_URL` — e.g., http://bitbucket.example.com:7990
  - **Basic auth:**
    - `BBS_AUTH_TYPE` : `Basic`
    - `BBS_USERNAME`
    - `BBS_PASSWORD`
  - **SSH:**
    - `SSH_USER` - SSH username for the Bitbucket Server host.
    - `SSH_PRIVATE_KEY` — an unencrypted (passphrase-free) private key.
- The `repos.csv` file must exist with the required columns: `project-key`, `project-name`, `repo`, `github_org`, `github_repo`, `gh_repo_visibility`.
- **Optional Storage backend:**
  - **AWS S3:** `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_BUCKET_NAME`, `AWS_REGION`
  - **Azure Blob:** `AZURE_STORAGE_CONNECTION_STRING`
  - **Github-owned:** 	*(nothing needed — automatic fallback)*

---

## 🔧 Initial Setup

Two workflows are present: `bbs2gh-migration.yml` (pre-check → migration → validation → monitoring) and `bbs2gh-gitlfs.yml` (large-file scan → large-file migration). You only need to configure the secrets, variables and environments on GitHub.

**All secrets and variables live in one place.** Put them either on the `bbs-migration` environment or at repository level - you never need to duplicate them. The approval gate is a separate job that holds no secrets, so it cannot cause "secret is empty" failures.

- **Add Secrets:** Go to your GitHub repo → `Settings` → `Security` → `Secrets and variables` → `Actions` → `Secrets` → `New repository secret`, and add the following:
  - `GH_PAT`: GitHub PAT with `repo`, `admin:org`, and `workflow` scopes.
  - `BBS_PAT`: Required for Pre and Post Migration validation stages.
  - `BBS_PASSWORD`: Bitbucket password (only if using Basic auth)
  - `SSH_PRIVATE_KEY`: Contents of a passphrase-free private key (e.g. `~/.ssh/id_rsa`)
  - **Storage backend secrets** (add only one set, or none for GitHub-owned storage):
    - `AZURE_STORAGE_CONNECTION_STRING`: Azure Blob Storage
    - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_BUCKET_NAME`, `AWS_REGION`: AWS S3
  - **Add Repository Variables:** Go to `Settings` → `Security` → `Secrets and variables` → `Actions` → `Variables` → `New Repository variable`, and add:
    - `BBS_AUTH_TYPE`: `Basic` (only if using Basic auth; omit when using `BBS_PAT`)
    - `BBS_USERNAME`: Bitbucket username (only if using Basic auth)
    - `SSH_USER`: SSH username for the Bitbucket Server host
    - `BBS_BASE_URL`: Bitbucket Server URL e.g. http://bitbucket.example.com:7990
    - `TARGET_API_URL`: Only for GitHub Data Residency e.g. https://api.tenant.ghe.com
    - `BBS_SHARED_HOME` *(optional)*: Bitbucket **shared home** path *on the Bitbucket Server host* (usually `<bitbucket-home>/shared`). Needed when the archive is not under the gh bbs2gh default `/var/atlassian/application-data/bitbucket/shared`.
    - `BBS_SSH_PORT` *(optional)*: SSH port for downloading the export archive (default `22`)
    - `BBS_ARCHIVE_DOWNLOAD_HOST` *(optional)*: Host to SSH to for the archive, if different from the Bitbucket URL host
    - `BBS_DISABLE_SSL_VERIFY` *(optional)*: `true` for self-signed / internal CA certificates
    - `RUN_LARGE_FILE_SCAN` / `LARGE_FILE_THRESHOLD_MB` *(optional)*: control the large-file scan (default threshold `400`)
    - `COMMIT_CHECK` *(optional)*: `false` to skip per-branch commit/SHA checks (default `true`)
    - `FAIL_ON_VALIDATION_FAILURES` *(optional)*: `true` to fail the Validation job on any discrepancy (default `false`)
  - **Create two Environments:** Go to `Settings` → `Environments` → `New environment` and create both:
    - `bbs-migration` — **holds all secrets and variables**. Every working job (pre-check, migration, validation, monitoring) runs here. Do **not** add required reviewers.
    - `bbs-migration-approval` — **holds nothing at all**. Add one or more **Required reviewers** and leave it otherwise empty.
  - **How the approval works:** a tiny `Approve migration` job runs in `bbs-migration-approval` and does nothing but wait for a reviewer. The real Migration job depends on it and runs in `bbs-migration`, so it reads the same single set of secrets as every other stage. `bbs2gh-gitlfs.yml` gates its large-file migration the same way.
  - **Environment names** must match the `environment` and `approval_environment` workflow inputs (defaults above). Change either input at run time if you use different names.
  -  **Prepare `repos.csv`:** Edit repos.csv in the repository root with the repos you want to migrate
   
  
      | Column | Description |
      |--------|-------------|
      | `project-key` | Bitbucket project key (e.g., `MYPROJ`) |
      | `project-name` | Bitbucket project display name |
      | `repo` | Bitbucket repository **slug or display name** - a name such as `Maven Hello World` is resolved to its slug automatically |
      | `github_org` | Target GitHub organization |
      | `github_repo` | Target GitHub repository name |
      | `gh_repo_visibility` | `private`, `internal`, or `public` |


---

## 🚀 Quick Start

**Before you begin**, ensure you've completed the [Initial Setup](#-initial-setup):
- ✅ Secrets and variables configured in **one** place (`bbs-migration` environment, or repo level)
- ✅ `bbs-migration` environment created **without** reviewers; `bbs-migration-approval` created **with** reviewers and nothing else
- ✅ repos.csv updated and pushed to the default branch

1. **Trigger the workflow:** Go to `Actions` → `bbs2gh-migration` → Run workflow and set your inputs, then click `Run workflow`.
2. **Review Stage 0 - Pre-checks:** The Pre-check job runs automatically.
      - Go to the job's Summary tab and review the pre-check table
      - Check the uploaded artifact `bbs-prechecks-<run-id>` → `bbs_pr_validation_output-<timestamp>.csv`
      - Ensure no repos show open PR warnings. If they do, merge/close those PRs in Bitbucket before proceeding.
3. **Approve Stage 1 - Migration:** The `Approve migration` job pauses waiting for approval. Go to the workflow run and click `Review deployments` → `Approve to release it`. The Migration job starts once approved.
      - Monitor the live status in the job logs (QUEUED / IN PROGRESS / MIGRATED / FAILED).
      - Once complete, download artifact migration-output-csv-<run-id> → repo_migration_output-<timestamp>.csv
      - Confirm all repos show MIGRATED.
4. **Review Stage 2 — Validation:**  The Validation job runs automatically after migration. Download artifact `validation-output-<run-id>` and check `validation-summary.md`:
      - ✅ All entries show Matching for branches, commit counts, and latest SHAs.
      - **Branch count and the "missing branches" comparison always cover every branch.** The per-branch commit/SHA deep-check is bounded for large repos: it checks the **default branch only** when a repo has ≤10 branches, or the **default branch plus up to 10 branches** when it has more than 10.

5. **Mannequins generation and reclaim:**  After you run a migration, all user activity in the migrated repository (except Git commits) is attributed to placeholder identities called mannequins.
    -  To generate a CSV file with a list of mannequins for an organization, Optionally, to include mannequins that have already been reclaimed, add the --include-reclaimed flag.: 
    `gh bbs2gh generate-mannequin-csv --github-org TARGET_ORG --output mannequins-bbs.csv`
    -  To reclaim generaate mannequins: 
    `gh bbs2gh reclaim-mannequin --github-org TARGET_ORG --csv mannequins-bbs.csv`

---

 
**Next Steps:**
- **More repositories?** Update `repos.csv` and rerun the workflow
- **Partial failures?** Fix the root cause, remove successfully migrated repos from `repos.csv`, and rerun workflow for remaining repos

---

## 🔤 Repository names vs slugs

`gh bbs2gh` identifies a repository by its **slug**, not its display name. Bitbucket
derives the slug from the name (`Maven Hello World` → `maven-hello-world`,
`solar system` → `solar-system`, `Test21` → `test21`).

You may put **either** form in the `repo` column - the scripts look the repository up
and substitute the real slug, logging what they did:

```
[WARNING] 'Maven Hello World' is a repository NAME, not a slug. Resolved to slug 'maven-hello-world' for BBSMIG.
```

If a value matches no repository at all, the row fails immediately - before any export
is created - and is listed with the valid options:

```
[ERROR] No repository in BBSMIG matches the name or slug 'Hello World Repo'.
[ERROR] Available in BBSMIG: calotracker maven-hello-world solar-system solid-api
```

That row is recorded in the output CSV as `Failure - Repo Not Found` and the rest of the
batch continues.

The **Pre-check** job also publishes a name → slug table for every project in `repos.csv`,
and flags any `repos.csv` entry that matches nothing - use it to fix the CSV before migrating.

---

## 🧬 Large files / Git LFS

Repositories containing very large files can fail to migrate. `bbs2gh-gitlfs.yml` handles
them separately from the main pipeline:

| Stage | What it does |
|-------|--------------|
| `Large-file Scan` | Clones each repo mirror-only and reports every file at or above `large_file_threshold_mb` (default 400MB), by filename |
| `Large-file Migration` | Migrates **only** the flagged repositories, gated by its own approval |

Outputs: `large_files_report-*.csv` (per file) and `large_file_repos-*.csv` (the skip-list),
published as the `bbs-lfs-scan-<run-id>` artifact.

To keep flagged repositories out of the main run, pass that scan's run ID to
`bbs2gh-migration.yml` as the **`lfs_scan_run_id`** input - those repos are then skipped
and recorded as `Skipped - Large Files`. Leave it empty to migrate everything.

Nothing is converted to Git LFS automatically: `git lfs migrate import` rewrites history,
so that step stays manual and deliberate.

---

## 📚 References

1. GitHub CLI | [cli.github.com](https://cli.github.com)
2. gh-bbs2gh extension | [github/gh-gei](https://github.com/github/gh-gei)
3. gh-migration-monitor | [mona-actions/gh-migration-monitor](https://github.com/mona-actions/gh-migration-monitor)
4. Migrate from Bitbucket Server (GitHub Docs) | [docs.github.com – BBS migrations](https://docs.github.com/en/migrations/using-github-enterprise-importer/migrating-from-bitbucket-server-to-github-enterprise-cloud/migrating-repositories-from-bitbucket-server-to-github-enterprise-cloud)



