# Skotsko v Úněticích 2026 – Email Sender Pipeline

## Context

Moving away from Mailchimp (database too large/expensive) to a self-hosted email sending pipeline: **Supabase** (contact DB) → **N8N** (automation) → **SendGrid/Twilio** (delivery). Goal: send the pre-sale announcement email for the 2026 celtic punk festival (25.4.2026).

## CRITICAL RULES

- **`skotskovuneticich-repo/`** = PRODUCTION website running on **Lovable**. **READ-ONLY. NEVER modify.**
- **`Skotsko-mailing`** = GitHub repo for all email/mailing work (templates, image hosting via GitHub Pages, project docs)

---

## Current Status

### DONE

| # | Task | Details |
|---|------|---------|
| 1 | **GitHub repo + Pages** | `github.com/darthPeter/Skotsko-mailing` — live at `darthpeter.github.io/Skotsko-mailing/` |
| 2 | **Email template** | `templates/email-2026-predprodej.html` (~24KB, under Gmail 102KB limit) |
| 3 | **All image assets** | Banner, merch (3), partners (4) — hosted on GitHub Pages |
| 4 | **Image URLs updated** | All point to `darthpeter.github.io/Skotsko-mailing/assets/...` |
| 5 | **N8N test workflow** | id: `3ASCdHgg08fPP5rS` — Webhook → Fetch Template → Inject Osloveni → SendGrid |
| 6 | **Test sends** | Multiple successful sends to `chotebor.p@gmail.com`, template verified on mobile |
| 7 | **Template reviewed** | Pricing box, merch names, copy, links — all iterated and approved |

### TODO (Next Steps)

| # | Task | Who | Status |
|---|------|-----|--------|
| | **--- SendGrid setup ---** | | |
| 1 | **Verify SPF/DKIM/DMARC** for celtic.cz — SendGrid → Settings → Sender Authentication → Authenticate Your Domain → add DNS records where celtic.cz is hosted | Petr | TODO |
| 2 | **Create ASM suppression group** — SendGrid → Settings → Suppression Management → Create New Group (name: "Festival Newsletter") | Petr | TODO |
| 3 | **Upload existing unsubscribers** from Mailchimp export to the ASM group — Suppression Management → select group → Upload CSV | Petr | TODO |
| 4 | **Enable open tracking** — SendGrid → Settings → Tracking → Open Tracking → ON | Petr | TODO |
| 5 | **Enable click tracking** — SendGrid → Settings → Tracking → Click Tracking → ON (HTML only) | Petr | TODO |
| | **--- Database ---** | | |
| 6 | **Set up Supabase table** for contacts (columns: email, osloveni, subscribed, batch, sent) | Petr/Claude | TODO |
| 7 | **Export contacts from Mailchimp** — Audience → All contacts → Export as CSV | Petr | TODO |
| 8 | **Import contacts** CSV → Supabase, assign batch numbers (1=first 100, 2=next 200, 3=remaining 200) | Petr/Claude | TODO |
| | **--- N8N workflow ---** | | |
| 9 | **Add ASM group ID to N8N workflow** (from step 2) | Claude | TODO |
| 10 | **Build production workflow** — read Supabase (where sent=false AND batch=N), loop, send, mark sent=true | Claude | TODO |
| | **--- Sendout ---** | | |
| 11 | **Batch 1** (100 contacts) — send, wait 2-3h, check stats | Petr triggers | TODO |
| 12 | **Batch 2** (200 contacts) — if #11 clean (bounces <2%, no spam) | Petr triggers | TODO |
| 13 | **Batch 3** (200 contacts) — next day | Petr triggers | TODO |

### Batching strategy (500 contacts, existing domain reputation)

Domain celtic.cz was used with Mailchimp last year (50% open rate). Audience = ticket buyers. Moving to SendGrid = new sending infrastructure, so mild warmup needed.

| Day | Batch | Size | Action |
|-----|-------|------|--------|
| Day 1 | #1 | 100 | Send, wait 2-3 hours, check stats |
| Day 1 | #2 | 200 | If #1 clean (bounces <2%, no spam reports) |
| Day 2 | #3 | 200 | Remaining |

**Kill switch:** If batch #1 has >5% bounces or any spam reports → stop, investigate.

**Supabase columns for batching:**
- `batch` (integer 1-3) — assigned during import
- `sent` (boolean, default false) — flipped to true after successful send, prevents double-sending

---

## Architecture

```
┌─────────────┐     ┌─────────┐     ┌───────────┐
│  Supabase   │────▶│   N8N   │────▶│ SendGrid  │────▶ Inbox
│  (contacts) │     │  (flow) │     │  (SMTP)   │
└─────────────┘     └─────────┘     └───────────┘
                         │
                  ┌──────┴──────┐
                  │  Template   │
                  │ (GitHub     │
                  │  Pages)     │
                  └─────────────┘
```

### Supabase table (proposed schema)

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid, PK | auto-generated |
| `email` | text, unique | contact email |
| `osloveni` | text | personalized greeting ("Petře", "Honzo", etc.) |
| `subscribed` | boolean | default true |
| `batch` | integer | 1-3, assigned during import |
| `sent` | boolean | default false, flipped to true after send (prevents double-sending) |
| `created_at` | timestamptz | auto-generated |

### N8N workflow

**Test flow** (id: `3ASCdHgg08fPP5rS`, name: "Skotsko e-mail (TEST)"):
```
Webhook (POST, no auth) → Fetch Template → Inject Osloveni → Send via SendGrid
```
- **Fetch Template:** HTTP GET `https://darthpeter.github.io/Skotsko-mailing/templates/email-2026-predprodej.html`
- **Inject Osloveni:** Code node — replaces `{{osloveni}}` with recipient greeting, sets recipient + subject
- **Send via SendGrid:** from `petr@celtic.cz` as "Skotsko v Úněticích"
- **Credentials:** SendGrid account `pSDSmCCOIPiKXOo2`

**Production flow** (TODO #5): Supabase read → loop contacts → inject osloveni per recipient → SendGrid batch send

### SendGrid setup needed

- Verified sender domain (`celtic.cz`) — check if already done
- **ASM suppression group** — Petr creates in SendGrid: Settings → Suppression Management → Add Group. Name: "Festival Newsletter". Then upload existing Mailchimp unsubscribers to this group. Give Claude the **group ID** (number) to wire into N8N.
- **Tracking** — enable open tracking + click tracking: Settings → Tracking
- Template ASM tags already in place: `<%asm_group_unsubscribe_raw_url%>`, `<%asm_preferences_raw_url%>`

---

## Email Template Details

**File:** `templates/email-2026-predprodej.html`
**Preview:** `https://darthpeter.github.io/Skotsko-mailing/templates/email-2026-predprodej.html`

**Sections:**
1. Full-width header banner (FB cover image with lineup)
2. Personalized greeting (`{{osloveni}}`)
3. Festival announcement with Mr. Irish Bastard YT link
4. Pricing callout box (Předprodej 300 Kč / Na místě 400 Kč / ends 17.4.)
5. CTA button "Chci lístek!" → `celtic.cz/`
6. Info links (celtic.cz, Facebook, FB události)
7. Signature (Petr, organizátor)
8. Merch grid: Únětický Trooper 2026 | The Beer for All | Festivalové OG (all 499 Kč, link → `kape.ly/skotsko-v-uneticich#merch`)
9. Partner logos (Hankey Bannister, Bláha, Únětický pivovar, Alkohol.cz)
10. Social icons (FB, IG, email)
11. Footer (copyright, Bannockburn Entertainments s.r.o., address, SendGrid unsub tags)

**Design system:**
- Dark green `#122b1a` (hero, footer)
- Burnt orange `#C45C26` (CTA, accents, MIB link)
- Gold `#D4A84B` (dividers, pricing badge border)
- Parchment `#f2f0eb` (outer background)
- White `#ffffff` (content area)
- Font: Oswald 700 (headings) → Helvetica Neue/Arial fallback

**Compatibility:**
- Gmail, Outlook (desktop + new), Apple Mail, iOS Mail, Android
- Responsive: stacks to single column at 600px, merch images full-width on mobile
- Dark mode: `prefers-color-scheme` media query with adapted colors
- Outlook: VML bulletproof CTA button + ghost tables for columns

**Links in template:**
- Banner → `celtic.cz/skotsko-v-uneticich/`
- CTA "Chci lístek!" → `celtic.cz/`
- Mr. Irish Bastard → `youtube.com/@MrIrishBastardOfficial`
- Merch (images + Koupit buttons) → `kape.ly/skotsko-v-uneticich#merch`
- Info links → `celtic.cz/skotsko-v-uneticich/`, `facebook.com/skotskovuneticich`
- Social → FB, IG, email (petr@chotebor.org)
- Footer unsub → `<%asm_group_unsubscribe_raw_url%>`, `<%asm_preferences_raw_url%>`

---

## Key Reference Data

| | |
|---|---|
| **Festival date** | 25.4.2026, 12:00 |
| **Location** | Únětický pivovar, Rýznerova 18, Únětice |
| **Lineup** | Mr. Irish Bastard (DE), Vintage Wine, Foggy Dude, Five Leaf Clover |
| **Tickets** | 300 Kč presale (until 17.4.), 400 Kč at door |
| **Free entry** | Děti do 12 let, ZTP, Únětičtí na občanku |
| **Ticket link** | https://kape.ly/skotsko-v-uneticich |
| **Website** | https://celtic.cz/ |
| **Company** | Bannockburn Entertainments s.r.o., Rýzmberská 539, 252 62 Horoměřice |
| **Mailing repo** | github.com/darthPeter/Skotsko-mailing |
| **Production repo** | github.com/darthPeter/skotskovuneticich (READ-ONLY) |
| **FB** | facebook.com/skotskovuneticich |
| **IG** | instagram.com/skotsko_unetice/ |
| **Contact** | petr@chotebor.org |
