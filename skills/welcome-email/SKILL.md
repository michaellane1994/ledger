---
name: welcome-email
description: Send welcome email sequence to new clients. Use when user asks to send welcome emails, onboard new client with emails, or trigger welcome sequence.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
---

# Welcome Client Emails

## Goal
Send a 3-email welcome sequence (Nick, Peter, Sam) when a new client signs. Emails are sent from a single Gmail account but display different sender names to simulate a team introduction.

## Scripts
- `./scripts/welcome_client_emails.py` - Send welcome sequence

## Process
1. Receive client info (name, email, company)
2. Send Email 1 from Nick (welcome, sets expectations for what's coming in the next 30 min)
3. Wait 15 seconds
4. Send Email 2 from Peter (brief warm intro)
5. Wait 15 seconds
6. Send Email 3 from Sam (kickoff call booking link)

## Usage

```bash
python3 ./scripts/welcome_client_emails.py \
  --client_name "John Doe" \
  --client_email "john@company.com" \
  --company "Acme Corp"
```

The script is designed to be called as a module from a webhook handler (`run(payload, token_data, slack_notify)`), not just from the command line. The command-line entry point at the bottom uses a hardcoded test payload and reads `token.json` from the parent directory.

## Timing Between Emails

Emails 1-2 and 2-3 are separated by a 15-second sleep. This is intentional to make the sequence feel like separate people are sending messages as they see a Slack notification, rather than a bulk send. The total sequence takes about 30 seconds to complete.

This delay is noted as "demo mode" in the code. For production you could increase the delays (e.g. 5-10 minutes) to feel more realistic.

## Email Templates

All three email bodies are defined directly in `welcome_client_emails.py`. To customize them, edit the `nick_body`, `peter_body`, and `sam_body` strings in the `run()` function.

**Email 1 - Nick:** Welcomes the client by first name, mentions the team, previews what they'll receive in the next 30 minutes (more welcome emails, onboarding PDF, kickoff calendar link). Establishes that work is already starting.

**Email 2 - Peter:** Short (4-line) informal hello, references knowing the client's company from a previous call.

**Email 3 - Sam:** Brief note from "bookings", provides the cal.com kickoff call link. Update `https://cal.com/YourCompany/Client-Onboarding` to your actual booking URL.

### Personalization
- `{client_first_name}` - extracted from the first word of `client_name`
- `{company_name}` - defaults to `client_first_name` if not provided
- The "From" display name changes per email (Nick / Peter / Sam) but the sending Gmail address is always the same account

## Authentication

The script uses Gmail API via Google OAuth. It expects a `token.json` file with OAuth credentials in the parent directory of the scripts folder (i.e., one level up from `scripts/`).

The `token.json` structure:
```json
{
  "token": "...",
  "refresh_token": "...",
  "token_uri": "https://oauth2.googleapis.com/token",
  "client_id": "...",
  "client_secret": "...",
  "scopes": ["https://www.googleapis.com/auth/gmail.send"]
}
```

When run inside a webhook handler (Modal or local server), `token_data` is passed in as a dict rather than read from file.

## Error Handling

- If `client_email` is missing: returns immediately with `{"status": "error", "error": "Missing client_email"}`
- If `client_name` is missing: returns immediately with `{"status": "error", "error": "Missing client_name"}`
- If Nick's email fails: the sequence stops and returns an error (Email 1 is critical)
- If Peter's or Sam's email fails: the error is logged but the sequence continues - those emails are non-critical
- Return value on success: `{"status": "success", "emails_sent": 3, "client_name": ..., "results": [...]}`

## Customization Checklist

Before using in production, update these hardcoded values in the script:
- `you@yourdomain.com` - the actual Gmail account address in `send_email()`
- `https://cal.com/YourCompany/Client-Onboarding` - your real booking URL in Sam's email
- `YourCompany` - your company name in Nick's email body
- Team names (Nick, Peter, Sam) if your team is different
- The 15-second delays if you want more realistic timing

## Troubleshooting

**Gmail API error "insufficient authentication scopes":** The `token.json` must include `https://www.googleapis.com/auth/gmail.send` in its scopes list. Re-authorize if needed.

**Token expired:** The script auto-refreshes expired tokens using `google.auth.transport.requests.Request()`. If refresh fails, re-run the OAuth flow to generate a new `token.json`.

**Only 1-2 emails sent instead of 3:** Peter's and Sam's failures are non-critical and logged but not surfaced as errors. Check the logs for `Email 2 failed` or `Email 3 failed` messages.

**Emails arriving from wrong display name:** All emails actually come from the same Gmail address. The `From` header just shows a display name. Some email clients may show the true address on hover.
