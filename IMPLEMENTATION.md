# Cross-Repo Trigger via GitHub App - Implementation Guide

This repository contains the complete implementation for triggering GitHub Actions workflows in a target repository from a source repository using a GitHub App.

## 📋 Prerequisites

- GitHub account (or organization)
- Two repositories: one for source, one for target

## 🔑 Setup Steps

### Step 1: Create the GitHub App

1. Go to **Settings** → **Developer settings** → **GitHub Apps** → **New GitHub App**
2. Configure as follows:

| Setting | Value |
|---------|-------|
| App name | `cross-repo-trigger` |
| Homepage URL | (optional) |
| Webhook URL | (leave empty) |
| Setup URL | (leave empty) |
| Redirect URL | (leave empty) |

3. Set permissions:
   - **Repository permissions:**
     - Contents: Write (required by the `/dispatches` endpoint)
     - Metadata: Read (required for all app operations)
   - **Subscribe to events:**
     - `repository_dispatch`
     - `workflow_dispatch`
   - **Repository access:** Select both source and target repositories

4. Install the app and note:
   - **App ID** (found on the app settings page)
   - **Private Key** (click "Generate a private key" to download)

> **Note:** Installation ID is no longer needed as a secret. The `actions/create-github-app-token@v1` action auto-discovers the installation ID from the app configuration.

### Step 2: Configure Secrets

#### Source Repository

Go to **Settings** → **Secrets and variables** → **Actions** → **New repository secret**:

| Secret Name | Value |
|------------|-------|
| `APP_ID` | Your GitHub App ID |
| `APP_PRIVATE_KEY` | The full content of the downloaded `.pem` file |

#### Target Repository

No secrets needed.

### Step 3: Deploy Workflow Files

#### Source Repository: `.github/workflows/source-trigger.yml`

```yaml
name: Cross-Repo Trigger

on:
  workflow_dispatch:
    inputs:
      target-repo:
        description: 'Target repository (owner/repo)'
        required: true
        default: 'your-username/target-repo'
      message:
        description: 'Message to include'
        required: false
        default: 'Triggered from source repo'

permissions:
  contents: write   # Required by the /dispatches endpoint
  metadata: read    # Required for basic operations

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: Get GitHub App Installation Token
        id: token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - name: Trigger Target Repository
        uses: actions/github-script@v7
        with:
          github-token: ${{ steps.token.outputs.token }}
          script: |
            const targetRepo = '${{ github.event.inputs.target-repo }}';
            const [owner, repo] = targetRepo.split('/');

            if (!targetRepo || !targetRepo.includes('/')) {
              core.setFailed('Invalid target repository format. Use: owner/repo-name');
              return;
            }

            try {
              const result = await github.rest.repos.createDispatchEvent({
                owner: owner,
                repo: repo,
                event_type: 'cross-repo-trigger',
                client_payload: {
                  message: '${{ github.event.inputs.message }}',
                  source_repo: context.repo.owner + '/' + context.repo.name,
                  source_ref: context.ref,
                  triggered_by: context.actor,
                  run_id: context.runId
                }
              });

              core.info(`✅ Dispatched to ${owner}/${repo} (Event ID: ${result.id})`);
            } catch (error) {
              core.error(`❌ Dispatch failed: ${error.message}`);
              core.setFailed(`Dispatch failed: ${error.message}`);
            }
```

#### Target Repository: `.github/workflows/target-receiver.yml`

```yaml
name: Cross-Repo Receiver

on:
  repository_dispatch:
    types: [cross-repo-trigger]

permissions:
  contents: read

jobs:
  receive:
    runs-on: ubuntu-latest
    steps:
      - name: Display trigger information
        run: |
          echo "📥 Cross-Repo Trigger Received!"
          echo "Event Type: ${{ github.event.type }}"
          echo "Client Payload:"
          echo "${{ toJSON(github.event.client_payload) }}"

      - name: Process the trigger
        run: |
          echo "✅ Processing cross-repo trigger..."
          echo "Message: ${{ github.event.client_payload.message }}"
          echo "Source: ${{ github.event.client_payload.source_repo }}"

          # Add your actual CI/CD steps here
          echo "🚀 Your build/test/deploy steps go here"
```

### Step 4: Test

1. Push the workflow files to their respective repositories
2. Go to the **Source Repository** → **Actions** tab
3. Click on **Cross-Repo Trigger** workflow
4. Click **Run workflow** button
5. Enter the target repository name and message
6. Click **Run workflow**
7. Check the **Target Repository** → **Actions** tab to see the triggered run

## 🔍 How It Works

```
Source Workflow                    GitHub API                     Target Workflow
─────────────────                 ────────────                 ───────────────

1. workflow_dispatch            │
2. Generate JWT (sign with      │                              │
   app private key)             │                              │
3. Exchange JWT for             │                              │
   Installation Token           │                              │
4. POST /repos/{owner}/{repo}/  │                              │
   dispatches                   │                              │
5.                                ────────────────────────────▶│
6.                                │  repository_dispatch event │
7.                                │  → runs target workflow    │
```

## 🔒 Security Notes

- The private key is stored as a GitHub Secret, never in code
- JWTs expire after 5 minutes
- Installation tokens expire after 1 hour
- The app only has permissions for the specific repositories it's installed on

## 🐛 Troubleshooting

| Problem | Solution |
|---------|----------|
| 401 Unauthorized | Verify the private key is correctly stored in secrets |
| 403 Forbidden | Check that the app has `contents: write` permission |
| 404 Not Found | Verify the target repository path is correct |
| Workflow doesn't run | Check that `repository_dispatch` event type matches |
