---
name: add-webhook
description: Add new Modal webhooks for event-driven execution. Use when user asks to create a webhook, add an endpoint, or set up event triggers.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Add Webhook

## Goal
Create new Modal webhooks for event-driven Claude orchestration. Each webhook maps a URL slug to a directive file, which Claude reads as instructions when the endpoint is called.

## Process

1. **Create Directive File**
   Create `directives/your_directive.md`

2. **Register in webhooks.json**
   Add entry to `execution/webhooks.json`

3. **Deploy**
   ```bash
   modal deploy execution/modal_webhook.py
   ```

4. **Test**
   ```bash
   curl "https://your-modal-username--claude-orchestrator-directive.modal.run?slug=your-webhook-slug"
   ```

---

## Step 1: Directive File

Directives live in `directives/`. Create `directives/your_directive.md`.

### Template
```markdown
# [Directive Name]

## Goal
One sentence describing what this webhook accomplishes.

## Inputs
Describe what data comes in via the webhook payload. List expected fields.

Example payload:
```json
{
  "client_name": "John Smith",
  "client_email": "john@company.com"
}
```

## Process
Step-by-step instructions for Claude to follow. Be specific about which tools to call and in what order.

1. Read the spreadsheet at ID `your-sheet-id` to get current data
2. Check if [condition]
3. If yes, send email to `{{client_email}}` with subject "..."
4. Update row N in the sheet to record completion

## Outputs
What Claude should return or produce. Describe the success state.

## Edge Cases
- What to do if the input is missing a required field
- What to do if an API call fails
- What to do if no matching record is found
```

### Example Directives

**Simple notification:**
```markdown
# Slack Deal Alert

## Goal
Send a Slack message when a new deal is logged in the CRM sheet.

## Inputs
Webhook payload with `deal_name`, `deal_value`, `client_email`.

## Process
1. Format message: "New deal: {deal_name} - ${deal_value} from {client_email}"
2. POST to Slack webhook URL from environment

## Outputs
Slack message sent with deal details.

## Edge Cases
- If deal_value is missing, use "unknown value"
```

**Sheet-based workflow:**
```markdown
# Process New Lead

## Goal
Enrich a new lead row in the leads sheet and send a prospecting email.

## Inputs
Payload with `lead_email` and `row_number` indicating which sheet row to update.

## Process
1. Read row {row_number} from sheet `1abc...xyz`
2. Look up the company domain from the email
3. Draft a 3-sentence outreach email using the company name
4. Send email to `lead_email`
5. Update column E in row {row_number} to "Contacted"

## Outputs
Email sent, sheet updated.

## Edge Cases
- If row is already marked "Contacted", skip and return early
```

---

## Step 2: webhooks.json

Add an entry to `execution/webhooks.json`:
```json
{
  "your-webhook-slug": {
    "directive": "your_directive",
    "description": "What this webhook does",
    "tools": ["send_email", "read_sheet", "update_sheet"]
  }
}
```

The `directive` value must match the filename without `.md` (e.g. `"your_directive"` maps to `directives/your_directive.md`).

The `tools` array tells the orchestrator which tool functions to make available to Claude for this webhook. Only list tools the directive actually needs.

### Available Tools
- `send_email` - Send email via Gmail API
- `read_sheet` - Read rows from a Google Sheet
- `update_sheet` - Write/update rows in a Google Sheet
- `slack_notify` - Post a message to Slack
- `http_request` - Make an arbitrary HTTP request

---

## Step 3: Deploy

```bash
modal deploy execution/modal_webhook.py
```

After deploying, verify the webhook appears in the list:
```bash
curl "https://your-modal-username--claude-orchestrator-list-webhooks.modal.run"
```

---

## Step 4: Testing

### Basic curl test
```bash
curl "https://your-modal-username--claude-orchestrator-directive.modal.run?slug=your-webhook-slug"
```

### Test with a payload (POST)
```bash
curl -X POST \
  "https://your-modal-username--claude-orchestrator-directive.modal.run?slug=your-webhook-slug" \
  -H "Content-Type: application/json" \
  -d '{"client_name": "Test User", "client_email": "test@example.com"}'
```

### Test locally before deploying
With the local server running (`python3 execution/local_server.py`):
```bash
curl "http://localhost:8000/directive?slug=your-webhook-slug"

# With payload
curl -X POST "http://localhost:8000/directive?slug=your-webhook-slug" \
  -H "Content-Type: application/json" \
  -d '{"client_name": "Test", "client_email": "test@test.com"}'
```

Local testing is faster than deploying to Modal for iteration. Only deploy when you're confident the directive works.

### Reading the response
A successful run returns something like:
```json
{"status": "success", "result": "...", "steps_completed": 3}
```

If Claude hits an error or the directive is malformed, check Modal logs:
```bash
modal app logs claude-orchestrator
```

---

## Argument Handling

Webhook payloads are passed to the directive as template variables. In the directive markdown, reference payload fields with `{field_name}` or describe them as "from the payload."

Claude reads the full payload automatically. If a required field is missing, add explicit handling in the "Edge Cases" section of your directive:

```markdown
## Edge Cases
- If `client_email` is not present in the payload, log a warning and return early without sending
```

For webhooks called from Zapier or Make, test what the actual payload looks like before writing the directive - the field names vary by integration.

## Common Patterns

**Trigger on sheet row addition (via Zapier):**
Zapier -> "New Row in Google Sheet" -> POST to webhook URL with row data as JSON

**Trigger on form submission:**
Typeform/Tally webhook -> POST to endpoint with form field values

**Trigger on Instantly email reply:**
Instantly -> "New Reply" webhook -> POST with `campaign_id`, `lead_email`, `reply_text`

**Scheduled (no external trigger):**
Add a cron function in `modal_webhook.py` that calls the directive function on a schedule instead of using the HTTP endpoint.

## Endpoints
- List: `https://your-modal-username--claude-orchestrator-list-webhooks.modal.run`
- Execute: `https://your-modal-username--claude-orchestrator-directive.modal.run?slug={slug}`
