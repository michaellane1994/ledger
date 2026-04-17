---
name: modal-deploy
description: Deploy execution scripts to Modal cloud. Use when user asks to deploy to Modal, push code to cloud, or update Modal functions.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Modal Cloud Deployment

## Goal
Deploy execution scripts to Modal for serverless cloud execution.

## Deploy Command
```bash
modal deploy execution/modal_webhook.py
```

## Key Endpoints

| Endpoint | Purpose |
|----------|---------|
| `directive` | Execute a directive by slug |
| `list_webhooks` | List available webhooks |
| `general_agent` | Run general agent tasks |
| `scrape_leads` | Lead scraping endpoint |
| `generate_proposal` | Proposal generation |
| `youtube_outliers` | YouTube outlier scraping |

## Adding New Functions

1. Add function to `execution/modal_webhook.py`
2. Decorate with `@app.function()` or `@app.function(schedule=modal.Cron(...))`
3. Deploy: `modal deploy execution/modal_webhook.py`

## Environment and Secrets Management

Modal secrets are configured in the Modal dashboard under Settings > Secrets, not in a local `.env` file. The app reads them at runtime via `modal.Secret`.

To add or update a secret:
1. Go to modal.com > your workspace > Secrets
2. Create or edit the secret group (e.g., `my-app-secrets`)
3. Add key/value pairs (e.g., `ANTHROPIC_API_KEY`, `INSTANTLY_API_KEY`, `GOOGLE_TOKEN`)
4. Reference in code: `@app.function(secrets=[modal.Secret.from_name("my-app-secrets")])`
5. Redeploy for the new secret to take effect

Local development uses a `.env` file directly. Modal ignores `.env` — only dashboard secrets are available in the cloud.

## Cron Jobs
```python
@app.function(schedule=modal.Cron("0 * * * *"))  # Every hour
def my_scheduled_function():
    pass
```

Common cron expressions:
- `"0 * * * *"` — top of every hour
- `"0 9 * * 1-5"` — 9am weekdays
- `"*/15 * * * *"` — every 15 minutes

## Debugging Deployed Functions

Modal streams logs in real time. To tail logs during a run:
```bash
modal app logs claude-orchestrator
```

Or view past logs in the Modal dashboard under the app's "Logs" tab.

To test a specific function locally before deploying:
```bash
modal run execution/modal_webhook.py::my_function
```

Print statements inside Modal functions appear in the logs stream.

## Testing After Deploy

After deploying, verify the endpoint is live:
```bash
# List available webhooks
curl "https://your-modal-username--claude-orchestrator-list-webhooks.modal.run"

# Execute a directive
curl "https://your-modal-username--claude-orchestrator-directive.modal.run?slug=your-slug"
```

Modal URLs follow the pattern: `https://{workspace}--{app-name}-{function-name}.modal.run`

## Rollback Procedure

Modal does not have a one-click rollback. To revert to a previous version:
1. Check out the previous commit in git: `git checkout <commit-hash> execution/modal_webhook.py`
2. Redeploy: `modal deploy execution/modal_webhook.py`
3. Restore the file: `git checkout HEAD execution/modal_webhook.py`

Alternatively, keep the previous deploy URL — Modal keeps old deployments accessible until they are explicitly stopped.

## Common Issues

**Deploy fails with import error:** Run `node --check` or `python3 -c "import execution.modal_webhook"` locally first to catch syntax errors before pushing.

**Secrets not found at runtime:** Confirm the secret group name in the dashboard matches the name used in `modal.Secret.from_name(...)`.

**Function not appearing after deploy:** Check that the function is decorated with `@app.function()` (not just a plain Python function) and that the file was saved before deploying.

**Timeout errors:** Modal functions default to a 30s timeout. Increase with `@app.function(timeout=300)` for long-running tasks.
