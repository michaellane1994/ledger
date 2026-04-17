---
name: instantly-autoreply
description: Auto-generate intelligent replies to incoming Instantly email threads using knowledge bases. Use when user asks about email auto-replies, Instantly responses, or automated email handling.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Instantly Auto-Reply

## Goal
Auto-generate intelligent replies to incoming emails from Instantly campaigns using campaign-specific knowledge bases. The flow is: Instantly webhook fires on new reply -> script looks up the campaign's knowledge base -> Claude writes a reply -> reply is sent back via Instantly API.

## Scripts
- `./scripts/instantly_autoreply.py` - Main auto-reply script

## How It Works
1. Receives an incoming email thread payload from Instantly webhook
2. Extracts the `campaign_id` from the payload (or parses it from `campaign_name` if the ID field is absent)
3. Looks up the campaign in the knowledge base spreadsheet to find context and tone examples
4. Retrieves the last 10 emails in the thread from Instantly (optional, for conversation context)
5. Generates a reply using Claude (claude-opus-4-5 with extended thinking enabled)
6. Checks for dry-run mode (email_id starts with `test-`) and skips sending if so
7. Sends reply through Instantly API using `reply_to_uuid`

## Webhook Setup

Instantly webhooks are configured in the Instantly dashboard under Settings > Webhooks. Set the trigger event to "New Reply Received" and point the URL to your Modal or local server endpoint.

The payload Instantly sends on a new reply contains:
```json
{
  "campaign_id": "your-campaign-id",
  "campaign_name": "YourCompany | Campaign Name",
  "lead_email": "prospect@company.com",
  "email_account": "outreach@yourdomain.com",
  "email_id": "uuid-of-the-email",
  "reply_subject": "Re: Your subject",
  "reply_text": "Plain text of their reply",
  "reply_html": "HTML version of their reply"
}
```

If `campaign_id` is not present in the payload, the script falls back to parsing the portion before `|` in `campaign_name`.

## Knowledge Base

Spreadsheet ID: `1QS7MYDm6RUTzzTWoMfX-0G9NzT5EoE2KiCE7iR1DBLM` (update `KB_SPREADSHEET_ID` in the script)

The spreadsheet must have a header row with these columns (case-insensitive):

| Column | Description |
|--------|-------------|
| `id` | Campaign ID - must match what Instantly sends in the payload |
| Campaign Name | Human-readable label (not used in lookup, just for reference) |
| `knowledge base` | Free-text description of the offering, pricing, process, credentials |
| `reply examples` | 1-3 example replies demonstrating the desired tone and style |

### Writing Effective Knowledge Bases

The `knowledge base` column is fed directly into Claude's prompt as context. Include:
- What the service/product does in 1-2 sentences
- Specific pricing tiers or ranges (Claude will quote these when asked)
- Process overview (how onboarding works, timeline, deliverables)
- Key differentiators or credentials ("won X award", "worked with Y")
- Common objection responses ("we're not the cheapest but...")

The `reply examples` column sets the tone. Include 2-3 short examples that show the voice you want - typically confident, friendly, concise. These are shown to Claude as style guidance, not templates.

### Adding a New Campaign
1. Add a row to the spreadsheet with a unique `id` matching the Instantly campaign ID
2. Write the knowledge base content
3. Add 2-3 reply examples in the reply examples column
4. No deploy needed - the sheet is read at runtime

## Response Quality Guidance

Claude is instructed to:
- Be concise (3-8 sentences)
- Use no em dashes, no hype words, no filler phrases
- Answer pricing/process/ROI questions with specifics from the knowledge base
- Sign off with the first name from the `email_account` address

To improve response quality:
- Make the knowledge base specific and concrete - vague context produces vague replies
- Add reply examples that match the actual email voice you use
- If Claude is being too salesy, add a note to the knowledge base like "never oversell, be direct"

## Escalation and Skip Handling

Claude returns the single word `SKIP` (case-insensitive) in these situations:
- The reply contains "unsubscribe" or "remove me"
- The conversation is clearly closed (call booked, deal closed)

When skipped, the script returns `{"status": "success", "skipped": true, "reason": "claude_skip"}` without sending anything.

If no knowledge base exists for the campaign, the script also skips with `reason: "no_knowledge_base"`. This is intentional - campaigns without a knowledge base should not receive auto-replies.

## Dry Run / Testing

To test without sending an actual reply, set `email_id` to a value starting with `test-`:

```python
test_payload = {
    "campaign_id": "your-campaign-id",
    "lead_email": "test@example.com",
    "email_account": "outreach@yourdomain.com",
    "email_id": "test-uuid",       # starts with "test-" = dry run
    "reply_subject": "Re: Test",
    "reply_text": "This sounds interesting, tell me more about pricing."
}
```

In dry-run mode, the script generates the reply with Claude but does not call the Instantly send API. It returns `{"dry_run": true, "reply_preview": "..."}` so you can inspect the generated text.

You can also set `"dry_run": true` in the payload directly.

## Environment
```
INSTANTLY_API_KEY=your_key
ANTHROPIC_API_KEY=your_key
```

The Google token for spreadsheet access is passed in as `token_data` dict by the webhook handler (not read from `.env`).

## Troubleshooting

**Script skips with `reason: "no_campaign_id"`:** The campaign_id is not in the Instantly payload and couldn't be parsed from campaign_name. Check that the campaign name contains a `|` separator if you're using the name-parsing fallback.

**Script skips with `reason: "no_knowledge_base"`:** The campaign ID from the payload doesn't match any row in column A of the spreadsheet. Verify the ID matches exactly (case-sensitive comparison).

**Claude generates an error message instead of a reply:** The `generate_reply` function catches exceptions and returns `"ERROR: {message}"` as a string rather than None. This prevents silent failures but will send the error text as a reply if not caught. Check `ANTHROPIC_API_KEY` is set correctly.

**Reply not appearing in Instantly:** Confirm `reply_to_uuid` matches the `email_id` from the webhook payload. The Instantly reply API requires this UUID to thread the reply correctly.

**"INSTANTLY_API_KEY not configured" warning in logs:** The API key env var is missing. The `get_conversation_history` call will return an empty list (non-fatal), but `send_reply` will also fail. Set the key in `.env` (local) or Modal secrets (cloud).
