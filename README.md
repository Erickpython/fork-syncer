# Fork Syncer

Automated daily synchronization for all active forked GitHub repositories using **GitHub Actions** and the **GitHub CLI (`gh`)**.

Instead of adding workflow files to individual forks—which modifies their commit history and can trigger merge conflicts—this repository acts as a single central controller that keeps every non-archived fork up to date with its upstream repository.

## Features

- **Centralized Control:** Runs from one repository without touching your individual forks.
- **Auto-Discovery:** Automatically detects and syncs any new repositories you fork in the future.
- **Fault-Tolerant:** Skips repositories with local divergence or merge conflicts gracefully without interrupting the rest of the batch.
- **Daily & On-Demand Execution:** Runs automatically every day at `00:00 UTC` and supports manual triggers via `workflow_dispatch`.

## Setup Guide

### 1. Generate a Personal Access Token (PAT)
GitHub's default `GITHUB_TOKEN` is scoped only to the repository running the workflow. To update other repositories in your account, create a classic Personal Access Token:

1. Go to **GitHub Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**.
2. Click **Generate new token** → **Generate new token (classic)**.
3. Add a descriptive note (e.g., `Fork Sync Automation`) and choose an expiration date.
4. Select the following scopes:
   - `repo` (Full control of private and public repositories)
   - `workflow` (Required when upstream repositories update `.github/workflows/` files)
5. Click **Generate token** and copy the `ghp_...` token immediately.

### 2. Create the Control Repository
Create a new public or private repository (such as `fork-syncer`) to store this workflow.

### 3. Add the Repository Secret
1. In your control repository, navigate to **Settings** → **Secrets and variables** → **Actions**.
2. Click **New repository secret**.
3. Set **Name** to `SYNC_TOKEN`.
4. Paste your Personal Access Token into the **Secret** field and click **Add secret**.

### 4. Create the Workflow File
Create `.github/workflows/sync-forks.yml` in this repository with the following configuration:

```yaml
name: Sync All Forked Repositories

on:
  schedule:
    # Runs every day at 00:00 UTC
    - cron: '0 0 * * *'
  # Allows manual runs from the Actions tab
  workflow_dispatch:

jobs:
  sync-forks:
    runs-on: ubuntu-latest
    steps:
      - name: Sync all active forks with upstream
        env:
          GH_TOKEN: ${{ secrets.SYNC_TOKEN }}
        run: |
          repos=$(gh repo list --fork --no-archived --limit 1000 --json nameWithOwner --jq '.[].nameWithOwner')
          
          for repo in $repos; do
            echo "Syncing $repo..."
            if gh repo sync "$repo"; then
              echo "✓ Successfully synced $repo"
            else
              echo "⚠️ Skipped or merge conflict on $repo"
            fi
          done
```

### 5. Run a Manual Verification
1. Open the **Actions** tab in your repository.
2. Select **Sync All Forked Repositories** from the left sidebar.
3. Click **Run workflow** → **Run workflow** and confirm each fork logs `✓ Successfully synced`.

## Maintenance Notes

- **Schedule Customization:** Edit the `cron` expression in `.github/workflows/sync-forks.yml` to change the sync frequency.
- **60-Day Inactivity Rule:** GitHub automatically disables scheduled workflows in repositories with no commit activity for 60 days. GitHub sends an email notification beforehand with a one-click link to re-enable the schedule, or you can re-enable it anytime in the **Actions** tab.
- **Token Expiration:** When your PAT expires, generate a new token with the same scopes and update the `SYNC_TOKEN` secret.
