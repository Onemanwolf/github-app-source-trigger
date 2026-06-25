# Cross-Repo Trigger via GitHub App — Implementation Guide

Trigger GitHub Actions workflows in a **target** repository from a **source**
repository using a **GitHub App**, with three working patterns:

1. **Async (fire-and-forget)** — source dispatches and exits immediately.
2. **Async + callback** — target reports success/failure back to the source in a separate run.
3. **Synchronous** — source dispatches, **waits** for the target to finish, passes/fails its own run, and can read **data the receiver produced** (e.g. a container name) back into the same run.

Authentication uses the maintained [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token)
action — no hand-rolled JWTs, no installation ID, no long-lived user tokens.

---

## ✅ Setup Checklist (do this on BOTH repos)

```
[ ] 1. Create one GitHub App (owner = the account that owns both repos)
        - Webhook "Active": OFF      - Subscribe to events: none
        - Permissions: Contents = Read & write, Actions = Read-only, Metadata = Read-only
[ ] 2. Note the App's CLIENT ID (Iv23...) and Generate a private key (.pem)
[ ] 3. Install App → All repositories (or select BOTH source + target)
[ ] 4. Add secrets to BOTH repos:  APP_CLIENT_ID, APP_PRIVATE_KEY
[ ] 5. Add workflow files (source/* to source repo, target/* to target repo)
[ ] 6. Run "Source - Cross-Repo Trigger (Sync)" → confirm it waits + returns data
```

Full details for each step are below.

---

## 📋 Prerequisites

- A GitHub account (or organization) that owns **both** repositories
- Two repositories — here: `Onemanwolf/github-app-source-trigger` and `Onemanwolf/github-app-target-receiver`
- `gh` CLI authenticated as the repos' owner (for the secret-setting commands)

---

## 🔑 Setup

### Step 1: Create the GitHub App

**Settings → Developer settings → GitHub Apps → New GitHub App**

| Setting | Value |
|---------|-------|
| App name | `cross-repo-trigger` (any unique name) |
| Homepage URL | any valid URL (e.g. the source repo URL) — required field, not otherwise used |
| Webhook → **Active** | **Uncheck** — this app only *mints tokens to call the API*; it does not receive webhooks |
| Subscribe to events | **None** — not used |

**Repository permissions:**

| Permission | Access | Why |
|-----------|--------|-----|
| **Contents** | Read and write | Required by the `POST /repos/{owner}/{repo}/dispatches` endpoint |
| **Actions** | Read-only | Required only by the **synchronous** workflow to poll the target's run status |
| **Metadata** | Read-only | Added automatically; required for all app operations |

> If you only use the async patterns, **Actions** is not needed. The sync workflow needs it.

**After creating the app:**
- Note the **Client ID** (the `Iv23...` string on the General page) — this is what the workflows use, *not* the numeric App ID.
- Click **Generate a private key** → downloads a `.pem` file.
- **Install App** (left sidebar) → install on your account → **All repositories** (or select both repos). The app must cover **both** repos so it can dispatch in either direction.

> No installation ID needed — `create-github-app-token` auto-discovers it.

### Step 2: Configure Secrets

Because the **callback** and **sync** paths authenticate from *both* sides, both repos get the same two secrets.

**Source repo** (`github-app-source-trigger`) and **Target repo** (`github-app-target-receiver`):

| Secret Name | Value |
|------------|-------|
| `APP_CLIENT_ID` | The app's **Client ID** (`Iv23...`) |
| `APP_PRIVATE_KEY` | Full contents of the downloaded `.pem` file |

Set them with the CLI:

```bash
gh secret set APP_CLIENT_ID   --repo OWNER/REPO --body "Iv23li..."
gh secret set APP_PRIVATE_KEY --repo OWNER/REPO < path/to/key.pem
```

---

## 📁 Deployed Workflows

| Repo | File | Role |
|------|------|------|
| source | `source-trigger.yml` | Async fire-and-forget trigger |
| source | `source-trigger-sync.yml` | Synchronous trigger — waits for target **and reads back its result artifact** |
| source | `source-result-receiver.yml` | Receives the async callback from the target |
| target | `target-receiver.yml` | Receives the dispatch, does the work, **uploads a `result.json` artifact** |
| target | `target-evaluator.yml` | Evaluates the receiver run and sends the callback |

### Source: `source-trigger.yml` (async)

```yaml
name: Source - Cross-Repo Trigger

on:
  workflow_dispatch:
    inputs:
      target-repo:
        description: 'Target repository (owner/repo)'
        required: true
        default: 'Onemanwolf/github-app-target-receiver'
      message:
        required: false
        default: 'Triggered from source repo via GitHub App'

permissions:
  contents: read   # only actions/checkout uses the workflow token; the dispatch uses the App token

jobs:
  trigger-target:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - name: Generate GitHub App Installation Token
        id: get-token
        uses: actions/create-github-app-token@v3
        with:
          client-id: ${{ secrets.APP_CLIENT_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          # Without owner/repositories the token is scoped to ONLY this repo,
          # so the cross-repo dispatch fails with 403 "Resource not accessible by integration".
          owner: ${{ github.repository_owner }}
          repositories: github-app-target-receiver

      - name: Trigger Target Repository Workflow
        uses: actions/github-script@v9
        with:
          github-token: ${{ steps.get-token.outputs.token }}
          script: |
            const targetRepo = '${{ github.event.inputs.target-repo }}';
            if (!targetRepo || !targetRepo.includes('/')) {
              core.setFailed(`Invalid target repository: "${targetRepo}". Expected owner/repo.`);
              return;
            }
            const [owner, repo] = targetRepo.split('/');
            await github.rest.repos.createDispatchEvent({
              owner, repo,
              event_type: 'cross-repo-trigger',
              client_payload: {
                message: '${{ github.event.inputs.message }}',
                source_repo: context.repo.owner + '/' + context.repo.repo,
                source_ref: context.ref,
                triggered_by: context.actor,
                run_id: context.runId,
                timestamp: new Date().toISOString(),
              },
            });
            core.info(`✅ Dispatched to ${owner}/${repo}`);
```

> Key correctness notes baked in above: **`client-id`** (not `app-id`), **`owner`/`repositories`** scoping (or the dispatch 403s), **`context.repo.repo`** (not `context.repo.name`, which is `undefined`), and the workflow token only needs **`contents: read`** (the *App* token does the dispatch).

### Target: `target-receiver.yml`

```yaml
name: Target - Cross-Repo Receiver

# Stamp the run name with the correlation id (when present) so the synchronous
# source workflow can find exactly this run while polling.
run-name: >-
  ${{ (github.event_name == 'repository_dispatch' && github.event.client_payload.correlation_id)
      && format('Receiver · cid={0}', github.event.client_payload.correlation_id)
      || 'Target - Cross-Repo Receiver' }}

on:
  repository_dispatch:
    types: [cross-repo-trigger]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  process-trigger:
    runs-on: ubuntu-latest
    steps:
      - name: Print client payload (safe object rendering)
        if: github.event_name == 'repository_dispatch'
        run: echo "${{ toJSON(github.event.client_payload) }}"   # toJSON — an object renders blank otherwise

      - name: Process the trigger
        run: |
          echo "Message: ${{ github.event.client_payload.message }}"
          echo "Source:  ${{ github.event.client_payload.source_repo }}"
          # Replace with real CI/CD logic.
```

---

## 🔁 Pattern 2 — Async + Callback

The source learns the result via a **separate run**, not by waiting.

```
Source: Trigger ──dispatch──▶ Target: Receiver ──workflow_run──▶ Target: Evaluator
                                                                      │
                                          mints App token (scoped to source)
                                                                      │
                              ──repository_dispatch (cross-repo-result)──▶ Source: Result Receiver
```

### Target: `target-evaluator.yml` (sends the callback)

Triggers on the receiver's `workflow_run` (for **every** conclusion), then dispatches the result back:

```yaml
on:
  workflow_run:
    workflows: ["Target - Cross-Repo Receiver"]
    types: [completed]

# ... after printing the conclusion ...
      - uses: actions/create-github-app-token@v3
        id: app-token
        with:
          client-id: ${{ secrets.APP_CLIENT_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: github-app-source-trigger   # token scoped to the SOURCE repo
      - uses: actions/github-script@v9
        with:
          github-token: ${{ steps.app-token.outputs.token }}
          script: |
            const run = context.payload.workflow_run;
            await github.rest.repos.createDispatchEvent({
              owner: context.repo.owner,
              repo: 'github-app-source-trigger',
              event_type: 'cross-repo-result',
              client_payload: {
                conclusion: run.conclusion,
                workflow_name: run.name,
                run_number: run.run_number,
                run_url: run.html_url,
                target_repo: context.repo.owner + '/' + context.repo.repo,
                reported_at: new Date().toISOString(),
              },
            });
```

### Source: `source-result-receiver.yml` (handles the callback)

Listens for `cross-repo-result`, prints the outcome, and **exits 1 on failure** so a target failure shows up red in the source repo. (It only prints — it never dispatches back — so there is no infinite loop.)

---

## ⏱ Pattern 3 — Synchronous (wait for the result)

`source-trigger-sync.yml` runs everything in **one job** that blocks until the target finishes:

1. Parse `owner/repo` from the input.
2. Mint an App token scoped to the target.
3. Dispatch with a **unique `correlation_id`** in the payload.
4. **Poll** `list workflow runs` on the target every 5s, matching the receiver run whose `run-name` carries the `correlation_id`, until it is `completed`.
5. Read its `conclusion`:
   - `success` → the source run finishes **green**.
   - anything else → `core.setFailed` → the source run goes **red**.
   - no match before `timeout-seconds` (default 300) → fail with a timeout error.

```
Source - Cross-Repo Trigger (Sync)   ← ONE run, blocks the whole time
  dispatch (cid) ──▶ poll target runs ──▶ match cid ──▶ wait for completed ──▶ pass/fail
```

> **Why Actions: Read is required here:** polling calls `GET /repos/{owner}/{repo}/actions/runs`, which needs the **Actions** read permission on the App. The async patterns don't.

---

## 🔄 Passing Data Both Directions

The sync workflow also carries **data down and back up** in the same run.

### Down: trigger → receiver (trivial)

Anything added to `client_payload` is readable in the receiver as
`${{ github.event.client_payload.<key> }}`. The sync workflow sends a
`container-name` and `image-tag` input down this way.

> **GitHub limit:** `client_payload` allows at most **10 top-level keys** (≈64 KB total). Nest under one object if you need more.

### Up: receiver → source (via an artifact)

A value the receiver *computes during its run* (e.g. a resolved container name)
is **not** exposed by the run-status API — only status/conclusion/metadata are.
So the receiver writes it to an artifact and the source downloads it:

**Receiver** (`target-receiver.yml`) — derive the value, write `result.json`, upload it:

```yaml
      - name: Resolve container name and write result
        id: deploy
        run: |
          CONTAINER_NAME="${{ github.event.client_payload.container_name }}"
          IMAGE_TAG="${{ github.event.client_payload.image_tag }}"
          DEPLOYED_CONTAINER="${CONTAINER_NAME}-${IMAGE_TAG}-run${{ github.run_number }}"
          cat > "$RUNNER_TEMP/result.json" <<EOF
          { "status": "deployed", "container_name": "$DEPLOYED_CONTAINER", "image_tag": "$IMAGE_TAG" }
          EOF
      - uses: actions/upload-artifact@v4
        with:
          name: cross-repo-result
          path: ${{ runner.temp }}/result.json
          retention-days: 1
```

**Source** (`source-trigger-sync.yml`) — after the target completes, download and parse it
(`gh run download` auto-unzips; the App token's **Actions: Read** covers artifacts):

```yaml
      - name: Download result artifact from target run
        if: success()
        id: result
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
          TARGET: ${{ steps.parse.outputs.owner }}/${{ steps.parse.outputs.repo }}
          RUN_ID: ${{ steps.wait.outputs.run_id }}   # run id surfaced by the poll step
        run: |
          gh run download "$RUN_ID" -R "$TARGET" -n cross-repo-result -D ./_result
          echo "container_name=$(jq -r '.container_name' ./_result/result.json)" >> "$GITHUB_OUTPUT"
```

Downstream steps then use `${{ steps.result.outputs.container_name }}`. Add any
fields you like to `result.json` and read them back with `jq`.

```
Sync run:  send container-name DOWN ──▶ receiver "deploys" ──▶ uploads result.json
           ◀── download artifact ◀── reads container_name back ── all in ONE run
```

---

## 🔒 Security Notes

- The private key is stored as an encrypted GitHub Actions secret, never in code.
- `create-github-app-token` mints a short-lived installation token (≈1 hour) per run and handles all signing.
- Tokens are scoped to specific repos via `owner` + `repositories` — least privilege.
- The app holds only the permissions it needs (Contents: write, Actions: read for sync, Metadata: read).

---

## 🐛 Troubleshooting

| Problem | Cause / Fix |
|---------|-------------|
| `403 Resource not accessible by integration` on dispatch | Token not scoped to the target — add `owner` + `repositories` to `create-github-app-token` |
| `403` when the **sync** workflow polls | App is missing **Actions: Read** — add it and accept the permission on the installation |
| `404 Not Found` | Target repo path wrong, or the app isn't installed on the target |
| `Source: owner/undefined` in payload | Use `context.repo.repo`, not `context.repo.name` |
| Receiver prints a blank payload | Render objects with `toJSON(github.event.client_payload)` |
| `Input 'app-id' has been deprecated` | Use `client-id: ${{ secrets.APP_CLIENT_ID }}` instead of `app-id` |
| Node.js 20 deprecation warning | Bump actions: `checkout@v7`, `create-github-app-token@v3`, `github-script@v9` |
| Sync run times out | Increase `timeout-seconds`, or check the receiver actually ran (correlation id mismatch) |
| `gh run download` finds no artifact | Receiver must `actions/upload-artifact@v4` with the same name (`cross-repo-result`); artifact exists only after the run completes |
