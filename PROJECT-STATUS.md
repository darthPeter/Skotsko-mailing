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

| # | Task | Depends on | Priority |
|---|------|-----------|----------|
| 1 | **Unsubscribe link** — implement working unsub (SendGrid ASM or custom via Supabase) | — | HIGH |
| 2 | **Analytics** — open/click tracking (SendGrid built-in or custom) | — | HIGH |
| 3 | **Set up Supabase table** for contacts | Supabase account | HIGH |
| 4 | **Import contacts** from Mailchimp export → Supabase | #3 | HIGH |
| 5 | **N8N production workflow** — loop over Supabase contacts, inject per-recipient `osloveni` | #1, #2, #3, #4 | HIGH |
| 6 | **Batch sending** — small batches to avoid SendGrid rate limits, monitor deliverability | #5 | HIGH |
| 7 | **Full sendout** to entire contact list | #6 | HIGH |

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

- Verified sender domain (`celtic.cz`)
- ASM suppression group (e.g. "Festival Newsletter") for unsubscribe handling
- Enable open/click tracking in SendGrid settings

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
