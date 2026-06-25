# github-app-source-trigger (SOURCE)

The **source** side of a cross-repo GitHub Actions trigger built on a **GitHub App**.
This repo starts the chain: it authenticates as the App and dispatches a
`repository_dispatch` event to the **target** repo
([`github-app-target-receiver`](https://github.com/Onemanwolf/github-app-target-receiver)).

📖 **Full guide:** [`IMPLEMENTATION.md`](./IMPLEMENTATION.md) — setup checklist, App config, all patterns, and troubleshooting.

## Workflows in this repo

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| **Source - Cross-Repo Trigger** | manual | Async fire-and-forget: dispatch and exit |
| **Source - Cross-Repo Trigger (Sync)** | manual | Dispatch, **wait** for the target, pass/fail with it, and read back its result (e.g. container name) |
| **Source - Result Receiver** | `repository_dispatch: cross-repo-result` | Receives the async callback and reports the target's result |

## Secrets required (this repo)

| Secret | Value |
|--------|-------|
| `APP_CLIENT_ID` | The GitHub App's Client ID (`Iv23...`) |
| `APP_PRIVATE_KEY` | The App's private key (`.pem` contents) |

```bash
gh secret set APP_CLIENT_ID   --repo Onemanwolf/github-app-source-trigger --body "Iv23li..."
gh secret set APP_PRIVATE_KEY --repo Onemanwolf/github-app-source-trigger < path/to/key.pem
```

## Run it

Actions tab → **Source - Cross-Repo Trigger (Sync)** → **Run workflow**
(inputs: target repo, container name, image tag). The run waits for the target
and prints the container name it returned.
