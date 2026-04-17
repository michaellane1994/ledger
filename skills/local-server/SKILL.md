---
name: local-server
description: Run Claude orchestrator locally with Cloudflare tunneling. Use when user asks to run locally, start local server, or test webhooks locally.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Local Server Execution

## Goal
Run the Claude orchestrator locally with Cloudflare tunnel for external webhook access. Use this during development to test changes without deploying to Modal, or when you need access to local files and credentials.

## Start Server
```bash
python3 execution/local_server.py
```

Starts FastAPI server on port 8000. You should see output like:
```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

## Cloudflare Tunnel
For external access (webhooks from Instantly, Zapier, etc.), expose the local port with Cloudflare:
```bash
cloudflared tunnel --url http://localhost:8000
```

This prints a public URL like `https://random-name.trycloudflare.com`. Use that URL wherever you'd normally use the Modal URL.

The tunnel is temporary — it closes when you stop the `cloudflared` process and the URL changes each time.

## Endpoints
Same as Modal deployment:
- `/directive?slug={slug}` - Execute directive
- `/list-webhooks` - List webhooks
- `/general-agent` - General agent tasks

## Installation and Dependencies

### Python dependencies
Install from the project root:
```bash
pip install -r requirements.txt
```

Key packages needed: `fastapi`, `uvicorn`, `anthropic`, `requests`, `python-dotenv`, `google-auth`, `google-auth-oauthlib`, `google-api-python-client`

### Cloudflare tunnel
Install `cloudflared`:
```bash
# macOS
brew install cloudflared

# Or download directly from:
# https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/
```

Verify it works:
```bash
cloudflared --version
```

No Cloudflare account needed for temporary tunnels (`--url` flag). Permanent named tunnels require an account.

## Environment

Uses local `.env` file directly (unlike Modal which uses dashboard secrets).

Create `.env` in the project root:
```
ANTHROPIC_API_KEY=sk-ant-...
INSTANTLY_API_KEY=...
OPENAI_API_KEY=...          # if needed
SLACK_WEBHOOK_URL=...        # if needed
UNPAYWALL_EMAIL=...          # for literature research
```

The server reads this file on startup via `python-dotenv`. Changes to `.env` require a server restart.

Google OAuth credentials (`token.json`) are read from the project root. Make sure `token.json` exists and has not expired before starting the server.

## Development Use
- Faster iteration than Modal deploys — change code and restart, no deploy step
- Full access to local files and credentials
- Real-time debugging with print statements visible directly in the terminal
- Can use a debugger (e.g. `python3 -m pdb execution/local_server.py` or VS Code's Python debugger)

## Debugging Guidance

**Add print statements freely:** Everything printed inside a handler shows up in the terminal immediately. There's no log aggregation to wait for.

**Test endpoints directly:**
```bash
# Test a directive
curl "http://localhost:8000/directive?slug=your-slug"

# Test with a POST body
curl -X POST "http://localhost:8000/general-agent" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "summarize this"}'

# List available webhooks
curl "http://localhost:8000/list-webhooks"
```

**Watch the terminal output** - unhandled exceptions print a full Python traceback, making it easy to pinpoint errors.

## When to Use Local vs Modal

| Situation | Use |
|-----------|-----|
| Developing and testing a new directive | Local |
| Debugging a production issue | Local (reproduce first) |
| Running scheduled cron jobs | Modal |
| Receiving webhooks from production services | Modal (or local + tunnel) |
| Sharing an endpoint with a teammate | Modal |
| Need access to local files | Local |
| Secrets are in Modal dashboard only | Modal |

The main trade-off: local is faster to iterate but the Cloudflare tunnel URL is ephemeral and changes every session. If you're setting up a long-running webhook (e.g., Instantly autoreply), Modal is the better choice for the production URL.

## Port Conflict Troubleshooting

**"Address already in use" error on port 8000:**

Find what's using it:
```bash
lsof -i :8000
```

Kill the process:
```bash
kill -9 <PID>
```

Or start the server on a different port by editing `local_server.py` to change the uvicorn port, then update the cloudflared command accordingly:
```bash
cloudflared tunnel --url http://localhost:8001
```

**Server starts but requests hang:** Check that the `.env` file exists and `ANTHROPIC_API_KEY` is set. A missing key usually causes the first Claude call to block or fail silently.
