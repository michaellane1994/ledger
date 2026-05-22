# Trove SaaS — Roadmap

Everything needed to go from demo template to a paying product.
Budget: **~20 hrs of focused work, ~$10 to start.**

**Brand:** Trove. **Domain:** householdtrove.com (Cloudflare Registrar, $10/yr).

---

## Current status (2026-05-21)

Shipped:
- Brand + domain locked in: Trove on `householdtrove.com`
- Cloudflare Worker live, custom domain wired, SSL provisioned
- Supabase auth with both magic link AND email/password sign-in; forgot-password + inline change-password flows
- Stripe Checkout + Customer Portal + webhooks operating in test mode (32 invocations / 0 errors verified end-to-end on new domain)
- Google Drive sync working (user-owned `ledger-data.json`)
- Privacy + Terms pages at `/privacy.html` and `/terms.html`, linked from sidebar foot and demo hero
- Mobile UX polish (topbar overflow fix, sidebar close-on-tap, larger logo + topbar title)
- "Manage My Trove" Settings page with Account section (subscription status badge + billing controls + inline set/change password)
- Sample-data warning banner on Upload page (prevents mixing demo data with real imports)
- Welcome hero consolidated (one banner, not two) — adapts CTAs based on signed-in vs demo state

Active near-term:
- Phase 5 (Resend transactional email) — not started
- Cloudflare Email Routing for `support@householdtrove.com` — pending user action
- Supabase config verification (Email provider enabled, reset-password URL whitelisted) — pending user action
- Stripe Public Business Info page (legal URLs + statement descriptor) — partially started, pending user action

Deferred:
- Dashboard hero Needs/Wants discrepancy (math doesn't quite reconcile with heatmap totals)

---

## TL;DR

| | |
|---|---|
| **Product** | Privacy-first household expense tracker. Data stays in user's own Google Drive. |
| **Price** | $9/mo or $89/yr (test mode). 14-day free trial. No card to start. |
| **Moat** | "We literally never see your data." |
| **Stack** | Cloudflare Workers + Supabase + Stripe + Resend |
| **Backend code** | ~340 lines in a Cloudflare Worker (`src/worker.js`) |
| **Day-1 cost** | ~$15 USD (domain). Rest is free tier. |
| **At 100 paying users** | ~$847/mo profit after Stripe fees (at $9/mo) |

---

## Services I need

| Service | For | Cost |
|---|---|---|
| Cloudflare Pages | Host the HTML | Free |
| Cloudflare Registrar | Domain | ~$10/yr |
| Supabase | Login + store (email, trial expiry, sub status) | Free |
| Stripe | Subscriptions, trials, card updates | 2.9% + 30¢/txn |
| Resend | Welcome + trial-ending emails | Free (3k/mo) |
| Sentry *(optional)* | Catch JS errors | Free |
| Plausible *(optional)* | Analytics, no cookies | $9/mo |

**I never store transaction data.** Supabase only holds: email, `trial_ends_at`, `stripe_customer_id`, `subscription_status`. That's it.

### Services — what each does + when you start paying

**Cloudflare Pages + Workers** — Hosts the static HTML and runs ~50 lines of serverless backend (Stripe checkout, webhooks, customer portal). GitHub Pages is static-only, so Workers is where all dynamic/paid logic lives.
- Free tier: unlimited bandwidth, 100k Worker requests/day, 500 builds/month.
- You pay: $5/mo only when you exceed 100k Worker requests/day (≈3M/mo). At that volume you're at ~30k daily active users — years away.

**Google Cloud (Drive API + OAuth)** — OAuth consent + Drive API for the `ledger-data.json` each user owns in their own Drive. Zero-cost storage that reinforces the privacy pitch.
- Free tier: 1B API queries/day. Verification is free for `drive.file` scope (non-sensitive).
- You pay: **never**, as long as billing stays disabled on the Cloud project. Hard ceiling is 100 test users during unverified testing — submit for verification (free, ~1 week) before crossing that.

**Supabase** — Auth (email / Google / magic links) + tiny Postgres table for user records (email, `trial_ends_at`, `stripe_customer_id`, `subscription_status`). Never stores transactions.
- Free tier: 50,000 Monthly Active Users, 500 MB database, 5 GB bandwidth.
- You pay: $25/mo when you cross 50k MAU. At that scale you're doing ~$250k MRR, so it's noise. Gotcha: free projects with 7 days of zero activity auto-pause — one-click restore, and stops mattering after launch.

**Stripe** — Subscription billing, 14-day trials, card updates via Customer Portal. No monthly fee.
- You pay: per transaction — 2.9% + 30¢ (Canada). A $5/mo subscription nets ~$4.56; a $49/yr nets ~$47.28.

**Resend** — Transactional email (welcome, trial-ending-3d, payment-failed). Clean API, good deliverability.
- Free tier: 3,000 emails/month, 100/day.
- You pay: $20/mo when you exceed 3k/mo. At ~5 emails per user lifecycle that's ~600 signups/month → ~$3k MRR. Comfortable headroom.
- Blocker: requires a **verified custom domain** (DNS records). Cannot send until you own a domain.

**Sentry** *(optional)* — JavaScript error monitoring. Errors only, never user data in payloads (privacy rule).
- Free tier: 5k errors/month.
- You pay: $26/mo for 50k errors. Free tier covers you for a long time.

**Plausible** *(optional)* — Privacy-friendly analytics, no cookies.
- No free tier.
- You pay: $9/mo for 10k pageviews, $19/mo for 100k. Skip until post-launch.

**Running total:**
- Pre-launch: **~$10 (domain only)**
- Launch day: **still ~$10 + Stripe fees per txn**
- First 1,000 paying users: **$10/yr domain + Stripe fees only** — everything else fits in free tiers
- Post-50k MAU: **+$25/mo Supabase Pro** (you'll be doing $250k MRR by then)

---

## Build order

### ✅ Phase 1 — Demo polish (DONE)

- [x] Demo top strip + CTAs
- [x] Demo hero card with sample data button (now unified with welcome banner — single hero serves both audiences)
- [x] Sync controls greyed out in demo
- [x] Pricing modal (monthly/yearly)
- [x] Reset demo / Clear demo buttons
- [x] Import preview with statement-total check
- [x] Remove "Repair missing income" button (both files)
- [x] Banner refers users to the `?` help button
- [x] FAQ modal (Help modal with Quick Start + Features + FAQ + Contact sections)
- [x] Footer (Privacy · Terms in sidebar foot, also in demo hero legal line)
- [x] Final brand name decision — **Trove**

### Phase 1.5 — Sync architecture: migrate SaaS template to Google Drive API

**Decision:** replace the Apps Script sync with Google Drive API (`drive.file` scope). Personal site stays on Apps Script — it already works, not worth migrating.

**Why:** Apps Script onboarding is ~8 steps including an "unverified app" warning that kills ~30% of signups. Drive API is one click → standard OAuth consent screen → done.

**Privacy pitch preserved:** user's data lives in a JSON file in their own Drive. OAuth token never leaves the browser. We never see the data or the token.

**Architecture decisions locked in:**
- ✓ Visible file in user's Drive (named `ledger-data.json`) — transparency
- ✓ Single file per user, no multi-vault
- ✓ Last-write-wins on conflicts (same as current Apps Script behavior)
- ✓ `drive.file` scope only — non-sensitive, no expensive security assessment
- ✓ Testing mode → 100 users free, then free verification (~1 week turnaround)

**Costs:** $0. Don't enable billing on the Cloud project — Drive API + OAuth are free-forever products outside the $300 credit program.

#### Google Cloud Console setup (user does once)

1. console.cloud.google.com → create project "Ledger"
2. APIs & Services → Library → **Google Drive API** → Enable
3. OAuth consent screen:
   - User type: External
   - App name: Ledger
   - Support email: yours
   - Scopes: `.../auth/drive.file` ONLY
   - Test users: add your own gmail
4. Credentials → Create OAuth Client ID:
   - Type: Web application
   - Authorized JavaScript origins:
     - `https://michaellane1994.github.io` (covers both /ledger and /ledger-saas)
     - `http://localhost` (local testing)
   - Authorized redirect URIs: empty (using implicit flow)
5. Copy the Client ID → send to Claude

#### Code changes (template only)

- Strip: `<pre id="apps-script-code">`, Setup Guide card, Web app URL field, all Apps Script references
- Add: Google Identity Services + gapi Drive client (2 CDN tags)
- New functions: `gdConnect()`, `gdDisconnect()`, `gdPush(data)`, `gdPull()`, `gdFindOrCreateFile()`
- Rewire: Push/Pull buttons to call Drive API
- Replace: "Google Sheets" → "Cloud Sync" section in Settings with single "Connect Google Drive" button
- Update all copy: "Google Sheet" → "Google Drive"
- Help modal: strip the Apps Script short-version, replace with "Click Connect Google Drive, you're done"

#### Migration path to 100+ users (later)

Submit for Google OAuth verification when approaching 100 users. Requirements:
- Privacy Policy URL (deferred to Phase 7)
- App homepage URL
- YouTube demo video (30–60 sec of the consent flow)
- ~1 week turnaround, free

#### Deferred (not doing now)

- Setup video (Loom walkthrough) — no longer needed since Drive API onboarding is one click
- Apps Script template sheet (`/copy` URL approach) — abandoned in favor of Drive API

### ✅ Phase 2 — Deploy (DONE)

GitHub source of truth → Cloudflare Worker auto-deploys on `git push origin main`.

- [x] GitHub repo `ledger-saas` created
- [x] Cloudflare Worker deployed (auto-deploys from `main`)
- [x] Domain `householdtrove.com` registered on Cloudflare Registrar
- [x] Custom domain attached: apex + `www.householdtrove.com`, SSL auto-provisioned
- [x] End-to-end tested on production domain

**Live URL:** `https://householdtrove.com` (fallback `ledger-saas.michael-lane867.workers.dev` still active — disable before public launch).

### ✅ Phase 3 — Auth (DONE)

- [x] Supabase project created
- [x] Email auth enabled (magic link + email/password)
- [x] `users` table with row-level security (see schema below)
- [x] Supabase JS via CDN
- [x] Login / signup modal with method toggle (Password / Magic link)
- [x] Forgot password flow + `/reset-password.html` recovery page
- [x] Inline Set/Change Password in Manage My Trove → Account
- [x] `_isDemo` flag wired to session state
- [x] Signup → set `trial_ends_at = now + 14d`
- [x] Trial expired / cancelled → fallback to demo mode

**Pending small actions (user, in Supabase Dashboard):**
- [ ] Authentication → Providers → Email: confirm enabled, "Confirm email" OFF
- [ ] Authentication → URL Configuration → add `https://householdtrove.com/reset-password.html` to Redirect URLs

```sql
users:
  id              uuid (from auth)
  email           text
  created_at      timestamp
  trial_ends_at   timestamp
  stripe_customer_id  text
  subscription_status text  -- 'trialing' | 'active' | 'cancelled' | 'past_due'
```

### ✅ Phase 4 — Billing (DONE in test mode)

Stripe Checkout + Customer Portal + webhooks fully operational on `householdtrove.com`.

- [x] Stripe account created (Test mode)
- [x] Two products: Monthly $9 (`price_1TPzDqBo3t6OY2ICaaC1KtDK`), Yearly $89 (`price_1TPzEIBo3t6OY2ICBOBW8cYb`)
- [x] Customer Portal activated
- [x] Cloudflare Worker endpoints: `/api/create-checkout-session`, `/api/create-portal-session`, `/api/webhook`, `/api/me`, `/api/diag`
- [x] Client wiring: `startCheckout(plan)`, `openCustomerPortal()`, Subscribe/Manage buttons, `?subscribed=1` return handler
- [x] 5 secrets configured in Cloudflare (`STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SECRET_KEY`)
- [x] Stripe webhook configured at `https://householdtrove.com/api/webhook` listening to 5 events: `checkout.session.completed`, `customer.subscription.created/updated/deleted`, `invoice.payment_failed`
- [x] End-to-end tested: signup → subscribe (4242 4242 4242 4242) → webhook fires → Supabase row updates → Manage in portal → cancel → status reverts
- [x] Cloudflare Cron heartbeat (10:00 UTC daily) keeps Supabase free-tier project from auto-pausing
- [x] Cloudflare Workers Logs / Observability enabled

**Pending before live mode (deferred — user wants to stay in test mode for now):**
- Stripe Public Business Info (business name, website, privacy/terms URLs, statement descriptor)
- Stripe identity + bank account verification
- Recreate products in live mode (new price IDs)
- Live-mode webhook endpoint + signing secret swap

### Phase 5 — Email *(2 hrs)*

- [ ] Resend account, verify domain DNS
- [ ] 3 templates: welcome, trial-ending-3d, payment-failed
- [ ] Triggered from Supabase hooks or Worker

### Phase 6 — Monitoring *(1 hr, optional)*

- [ ] Sentry SDK (errors only, never payloads)
- [ ] Plausible (pageviews, no cookies)
- [ ] Set up `hello@domain.com` support inbox

### Phase 7 — Launch prep *(2–3 hrs)*

- [x] Privacy Policy at `/privacy.html` (key line: "we don't see your data" — covered)
- [x] Terms of Service at `/terms.html`
- [x] Test full loop in test mode: signup → trial → subscribe → portal → cancel → resubscribe
- [ ] Add privacy + terms URLs to Stripe Public Business Info
- [ ] Beta-test with 3–5 real users in test mode
- [ ] Canadian sales tax: register for GST/HST if I expect >$30k/yr

---

## Future state goals

Captured ideas for v2 / v3. Not committed; thinking out loud. Add to this list as ideas come up. Each entry: **what / why / tradeoffs**.

### 1. AI-powered PDF parser (Claude)

Replace the current bank-specific regex parsers with a Claude-powered extraction step.

**How it would work:** PDF upload → Worker proxies to Anthropic API with a structured-extraction prompt → Claude returns `[{date, merchant, amount, category_guess}]` → app shows preview → user confirms → import (same UX as today).

**Why:** current parser only handles ~6 specific bank formats; everything else is broken. Claude can read any layout — RBC, Scotia, Wealthsimple, US banks, international, weird credit card statements. Bonus: Claude can also guess categories, reducing manual cleanup.

**Tradeoffs:**
- Latency: ~5–15s vs instant local parse
- Cost: ~$0.01–$0.05 per PDF in API spend (a $5/mo user uploading 4 statements/mo = ~$0.20/mo COGS)
- Privacy: PDF transits Anthropic servers — softens the "we never see your data" promise. Probably keep regex parser as default and offer AI as opt-in fallback when regex returns 0 results.
- Dependency: needs internet + API key

**Implementation note:** position as a paid-tier-only feature to recoup API costs and reinforce the value of subscribing.

### 2. Annual / quarterly snapshot ("Trove Wrapped")

Spotify-Wrapped-style end-of-period recap of expense activity. Sharable.

**Content ideas:** top 3 categories by spend, biggest single transaction, most-spent vs least-spent month, year-over-year comparison (if data exists), savings streak stats, quirky stats ("your weekend spending is 2.3× your weekday spending").

**Output formats:** in-app animated view (scrollable like Stories), shareable PNG / PDF export, optional end-of-year email.

**Why:** triple win — user delight + retention + viral growth. Users share with friends → friends curious → free user acquisition. Anchors the product in users' minds at a memorable moment each quarter.

**Open questions:**
- Privacy: users probably won't share absolute numbers, but might share percentages and category rankings
- Needs 3+ months of data to feel meaningful — gate behind a minimum data threshold

### 3. YouTube onboarding video

Short walkthrough video(s) — 2–5 min total.

**Content:** 30s "why Trove" overview (positioning) + 2-min full demo (upload CSV → categorise → dashboard → savings/debt). Optional follow-ups: power-user tips, reconciling with your bank, sharing with a partner.

**Where it shows up:** linked from the welcome banner, "Watch a 2-min tour" in Help modal, embedded on a future landing page, linked from email onboarding (Phase 5).

**Why:** reduces support burden for visual learners (huge audience), sets expectations correctly before signup, SEO bonus on YouTube. One-time effort, long-term return.

### 4. Free / freemium tier — *open design question*

**Goal:** prevent users churning at the 14-day cliff. Give them a reason to stay engaged.

**Honest tradeoff context:**
- Trial → paid conversion: typically **30–60%**
- Free → paid conversion: typically **1–5%**
- Free users still cost in support + infrastructure
- Risk: cannibalise paying users who would've subscribed anyway

**Options to consider:**

| Option | What it looks like | Pros | Cons |
|---|---|---|---|
| **A. Free with transaction cap** | "50 txns/mo free" | Familiar freemium pattern; clear upgrade prompt | Casual users never hit cap → stay free forever, never convert |
| **B. Local-only after trial** *(already exists as demo mode)* | After 14 days, lose cloud sync but app still works locally | Soft cliff, not a hard lock; users already in the habit | Some users won't notice the difference, never upgrade |
| **C. Longer trial (30 days)** | Same model, more time to build habit | Smaller eng lift; better conversion mechanics | Just delays churn 2 weeks |
| **D. "Lite" plan at $2/mo** | Half-price tier with limited features (no AI parser, no sharing, etc.) | Captures price-sensitive users; cleaner economics than free | Adds plan complexity |
| **E. Trial-ending nudge email** | Day 12 email: "your trial ends in 2 days, here's what you'd lose" | Cheap to implement; targets the actual churn moment | Doesn't change the cliff — just makes users aware |

**Current lean:** B + E. Local-only-after-trial is already there; add Phase 5 to send the day-12 nudge. Re-evaluate freemium once you have 100+ subscribers and real churn data.

**Open question for later:** what % of trial users actually finish onboarding (load real data)? If conversion is low *during* trial, no amount of free tier fixes that — fix onboarding first.

---

## Customer acquisition strategy

Living plan for how to find paying users. Updated as channels are tested.

### Honest framing

- Indie SaaS at $9/mo grows slowly. Typical: 5–15 paying users month 1–3, 30–100 by year 1, scale via referrals + SEO in year 2+.
- Paid ads do not math at $9 LTV (CAC $20–50 on Google/Meta). Skip paid until annual plans + low churn give $30+ LTV proven.
- Moat is the angle, not the features. Mint/Monarch/Copilot all have more features. Trove wins on **"your data stays in YOUR Drive."** Every marketing message hammers this.

### Positioning

**The story, three lines:**
> Mint shut down. The replacements want your bank login, scrape your data, and sell you ads. Trove doesn't. Your transactions stay in your Google Drive — we literally never see them. $9/mo. Households welcome.

**Target audiences, ranked by fit:**
1. **Mint refugees** — Intuit killed Mint Mar 2024; users still drifting between alternatives. Hot lead.
2. **Privacy-conscious tech users** — r/selfhosted, r/privacy, r/degoogle, r/datahoarder
3. **Couples managing joint finances** — Trove's household angle is genuinely unique
4. **YNAB-curious but simpler** — YNAB has a cult following but is opinionated; some bounce off it

### Phase A — Pre-launch foundations (now → 4 weeks)

Before pushing anywhere publicly, the launch surface needs to exist:

- [ ] **Separate marketing landing page** — `householdtrove.com` currently *is* the app. Need a proper marketing page at the root with hero → problem → solution → pricing → FAQ → CTA. App lives at `/app` or `/dashboard`. ~half-day of work.
- [ ] **30–60s demo video on YouTube** — what is Trove, why does it exist. Drives signups + onboards.
- [ ] **5–10 beta users** in test mode — friends, family, Reddit volunteers. Real testimonials before any public push.
- [ ] **Live Stripe mode activated** — can't sell without it. Requires identity + bank verification.
- [ ] **PDF parser improved** — current parsers only handle ~6 banks; Claude fallback for the rest. #1 hidden blocker.

### Phase B — Soft launch (4–8 weeks out)

Free / near-free channels, in this order:

- [ ] **r/PersonalFinanceCanada + r/personalfinance** — write a "what I built and why" post (mods strict — must be helpful, not promotional). Single best inbound channel for personal-finance indie SaaS.
- [ ] **Indie Hackers launch post** — their community supports indie SaaS launches.
- [ ] **Hacker News "Show HN"** — Tue/Wed/Thu morning. Risk: HN can be brutal, but a front-page day = 500–2000 visits.
- [ ] **X/Twitter "build in public"** — share progress, screenshots, lessons. Slow burn but compounds.
- [ ] **Direct outreach** — message 20 friends/contacts personally with "would you try this?"

Expected outcome: **30–50 trials, 5–15 paying users.**

### Phase C — Scale (3–12 months out)

Compounding channels — start when Phase B gives real signal:

- [ ] **SEO blog** — articles like "Mint alternatives 2026", "household budget templates", "private personal finance apps". ~1 post/week.
- [ ] **YouTube channel** — same topics; even higher SEO yield. Pairs with the onboarding video.
- [ ] **Product Hunt launch** — saved for when you have testimonials + screenshots ready.
- [ ] **Affiliate / referral program** — "give a month, get a month" for existing users.
- [ ] **Partnerships** — privacy newsletters, finance newsletters, mutual cross-promo with other indie tools.

### What NOT to do

- Cold email — low conversion, often spammy
- Facebook/Instagram ads — CAC > LTV at $9 pricing
- TikTok ads — same
- Sponsored content from generic influencers — expensive, low ROI
- Affiliate networks — low margins to share
- Generic "growth hacking" funnel tools
- Posting on every subreddit — self-promo rule kills it

### Marketing assets to build before any public push

| Asset | Status | Notes |
|---|---|---|
| Marketing landing page | not started | `householdtrove.com/` (with app moved to `/app`) |
| Onboarding video (60–90s) | not started | YouTube, embedded on landing page |
| 5–10 beta testimonials | not started | From test-mode friends/family |
| Reddit / IH / Show HN launch posts | not started | Draft now, file away; launch-day is the wrong time to write |
| Twitter / X build-in-public account | not started | Slow burn, start anyway |
| Newsletter or waitlist signup | not started | Capture interest from non-converters |

### Notes / lessons (update as channels are tested)

*(Empty for now — log results as channels are tried.)*

---

## Minimum code additions

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script src="https://js.stripe.com/v3/"></script>
```

```js
async function checkAuth() {
  const { data: { session } } = await supabase.auth.getSession();
  if (!session) { _isDemo = true; return; }
  const { data: u } = await supabase.from('users').select('*').eq('id', session.user.id).single();
  const active = u.subscription_status === 'active'
    || (u.subscription_status === 'trialing' && new Date(u.trial_ends_at) > new Date());
  _isDemo = !active;
}
```

New functions: `login`, `signup`, `logout`, `startCheckout(plan)`, `openCustomerPortal`, `checkAuth` (on page load).

Everything else already works. The only paywalled action is Google Sheets sync.

---

## Support playbook

I cannot see user data. So:

- **Import broken?** Preview + totals panel already lets them diagnose. They can screenshot and email.
- **Sync error?** Show the message inline. No black boxes.
- **Support email:** `hello@domain.com` → my inbox. No live chat at launch.
- **Refunds:** one click in Stripe. Be generous with forgotten trials.
- **FAQ must answer:** where's my data, what happens if I cancel, can I export, does it work on mobile.

---

## Launch checklist (the day)

Post once, in three places:
- r/personalfinance or r/PersonalFinanceCanada ("Show HN"-style, no hype)
- Indie Hackers launch post
- Product Hunt (Tue–Thu morning)

Message: **"Your data stays in your Google Drive. $5/mo, no feature gates, no ads."**

Reach out personally to the first 20 signups. Ask what's missing.

---

## Don't do these

- iOS/Android app (web works on mobile)
- Plaid/bank APIs (CSV+PDF is fine, Plaid is too expensive for $5/mo)
- Multi-tenant households with separate logins (households share one sheet)
- Chat widget
- Session replay tools (Hotjar/FullStory) — kills the privacy pitch
- Google Ads at launch (too expensive at $5 LTV)
- Fake social proof
- Feature gates on the paid tier

---

## Red flags to watch for

| Signal | Means |
|---|---|
| <5% trial → paid | Activation broken or price too high for perceived value |
| >10%/mo churn | Not sticky — onboarding fix needed |
| "Import didn't work" dominates tickets | Invest in more bank/card presets |
| Endless feature requests | Attracting wrong audience |

---

## Economics at scale

| Users | MRR | Stripe fee | Infra | Net |
|---|---|---|---|---|
| 10 | $50 | $3 | $0 | $47 |
| 100 | $500 | $29 | $0 | $471 |
| 1,000 | $5,000 | $290 | ~$25 | ~$4,685 |
| 10,000 | $50,000 | $2,900 | ~$200 | ~$46,900 |

Supabase free tier covers 50k MAU. Cloudflare Pages is unlimited. This scales without a new architecture.

---

## The rule I can't break

**Never store transactions.** Not in Supabase, not in Sentry, not in logs, not anywhere I control. The privacy pitch is the entire moat. Protect it.

---

## Lock-in map — what's reversible vs not

A reference for thinking about which decisions to lean into vs which to defer.

### Effectively irreversible (think hard now)

- **The privacy promise.** Once "we never see your data" is the public pitch, breaking it is brand-ending. No future product idea (AI features, server-side analytics, etc.) can require seeing user data without massive trust damage.
- **Brand name + domain.** Cheap to change pre-launch. After launch, switching costs SEO, social presence, marketing materials, user memory. Decide before any landing page or domain purchase.

### Hard to change (avoid if possible)

- **Stripe pricing for existing subscribers.** Once N users subscribe at $9/mo, they're grandfathered there forever — Stripe doesn't auto-update existing subs. New customers can pay a new price; existing ones stay unless they explicitly opt in (Canadian consumer law requires consent). Means: lower-than-needed launch price is hard to fix later. Sleep on $9 vs $12 before launch.
- **Drive file schema (`ledger-data.json` structure).** Breaking changes to field names/structure invalidate every existing user's file. Mitigated by `version: 1` field added 2026-04-24 — future migrations can branch on version. Add migration logic, don't break the schema in place.
- **Google OAuth scope.** Currently `drive.file` (non-sensitive, free verification, ~1 week). If a feature ever needs `drive.full` or another sensitive scope, expect a $15k–$75k third-party security audit and weeks of review. Stay on `drive.file` indefinitely.

### Easy to change (don't agonize)

- **Hosting (Cloudflare).** Worker code uses Web standard APIs (fetch, crypto.subtle). Portable to Vercel Edge / Deno Deploy / AWS Lambda in a day.
- **Supabase.** Postgres data is exportable anytime. Auth password hashes can move via Supabase's bulk export — users don't notice the migration.
- **Stripe products / prices** (for new customers). Archive old, create new, anytime. Existing customers grandfathered (see above).
- **Trial length.** Each new signup gets the value at signup time. No retroactive enforcement.
- **Database schema.** Postgres is forgiving. Add columns and indexes anytime; drop with care.
- **Features.** Add, remove, rebuild. Nothing in the stack locks the product surface.
- **localStorage prefixes / Drive file names.** Renamable with a one-time migration script. Don't optimise for these now.

### Decisions to lock in before launch

- [x] Final brand name — **Trove**
- [x] Final domain (Cloudflare Registrar) — **householdtrove.com**
- [x] Final monthly + yearly Stripe price — **$9/mo, $89/yr** (test mode; can change for new live-mode customers before public launch)
- [x] Privacy policy text — published at `/privacy.html`, explicitly covers "we never see your transactions"
- [x] Drive file `version: 1` shipped (2026-04-24)
