# Skotsko v Úněticích 2026 – Email Sender Pipeline

## Context

Moving away from Mailchimp (database too large/expensive) to a self-hosted email sending pipeline: **Supabase** (contact DB) → **N8N** (automation) → **SendGrid/Twilio** (delivery). Goal: send the pre-sale announcement email for the 2026 celtic punk festival (25.4.2026).

## CRITICAL RULES

- **`skotskovuneticich-repo/`** = PRODUCTION website running on **Lovable**. **READ-ONLY. NEVER modify.**
- **`Skotsko-mailing`** = NEW GitHub repo for all email/mailing work (templates, image hosting via GitHub Pages, project docs)

---

## Current Status

### DONE

| Component | Status | File/Location |
|-----------|--------|---------------|
| Festival website | Production-ready | `skotskovuneticich-repo/` (React/TS/Vite) |
| Email HTML template | 95% ready | `email-2026-predprodej.html` (24KB) |
| Template: SendGrid variables | Done | `{{osloveni}}`, `<%asm_group_unsubscribe_raw_url%>`, `<%asm_preferences_raw_url%>` |
| Template: Responsive design | Done | Mobile breakpoint 600px, columns stack |
| Template: Dark mode | Done | `prefers-color-scheme` media query |
| Template: Outlook compat | Done | VML bulletproof button, ghost tables |
| Template: Content sections | Done | Banner, body, pricing, CTA, merch (3 tees), partners (4), social, footer |
| Template: CAN-SPAM footer | Done | Company address, unsubscribe links |
| All image assets on GitHub | Done | `raw.githubusercontent.com/darthPeter/skotskovuneticich/main/src/assets/` |
| Meta Pixel tracking (web) | Done | In website repo |

### TODO

| # | Task | Depends on | Status |
|---|------|-----------|--------|
| 0 | **Create `Skotsko-mailing` GitHub repo** + GitHub Pages | — | DONE |
| 1 | **Upload banner image** + all assets to repo | #0 | DONE |
| 2 | **Update image URLs** to GitHub Pages | #0, #1 | DONE |
| 3 | **N8N workflow** for test send | — | DONE (id: `3ASCdHgg08fPP5rS`) |
| 4 | **Test send** to chotebor.p@gmail.com | #1, #2, #3 | DONE (execution #8256) |
| 5 | **Unsubscribe link** — implement working unsub (SendGrid ASM or custom via Supabase) | — | TODO |
| 6 | **Analytics** — open/click tracking (SendGrid or custom) | — | TODO |
| 7 | **Set up Supabase table** for contacts | Supabase account | TODO |
| 8 | **Import contacts** from Mailchimp export → Supabase | #7 | TODO |
| 9 | **N8N workflow** for bulk sendout (loop over Supabase contacts) | #5, #6, #7, #8 | TODO |
| 10 | **Batch sending** — send in small batches (avoid SendGrid rate limits, monitor deliverability) | #9 | TODO |
| 11 | **Full sendout** to entire contact list | #10 | TODO |

---

## Architecture

```
┌─────────────┐     ┌─────────┐     ┌───────────┐
│  Supabase   │────▶│   N8N   │────▶│ SendGrid  │────▶ Inbox
│  (contacts) │     │  (flow) │     │  (SMTP)   │
└─────────────┘     └─────────┘     └───────────┘
                         │
                    ┌────┴────┐
                    │ Template │
                    │  (HTML)  │
                    └─────────┘
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
Webhook → Fetch Template (HTTP GET GitHub Pages) → Inject Osloveni (Code) → Send via SendGrid
```
- **Fetch Template:** GETs `https://darthpeter.github.io/Skotsko-mailing/templates/email-2026-predprodej.html`
- **Inject Osloveni:** replaces `{{osloveni}}` with recipient's greeting, sets recipient + subject
- **Send via SendGrid:** from `petr@celtic.cz` as "Skotsko v Úněticích"
- **Credentials:** SendGrid account `pSDSmCCOIPiKXOo2`

**Production flow** (TODO): will add Supabase loop, unsub handling, analytics

### SendGrid setup needed

- Verified sender domain (`celtic.cz` or `celtic.monster`)
- ASM suppression group (e.g. "Festival Newsletter")
- Dynamic template or inline HTML via API

---

## Email Template Details

**File:** `email-2026-predprodej.html` (24KB – well under Gmail's 102KB clipping limit)

**Sections:**
1. Full-width header banner (new FB cover image with lineup)
2. Personalized greeting (`{{osloveni}}`)
3. Festival announcement + headliner highlight (Mr. Irish Bastard)
4. Pricing callout box (300 Kč presale / 400 Kč at door)
5. CTA button → `kape.ly/skotsko-v-uneticich`
6. Info links (celtic.cz, Facebook, FB event)
7. Signature (Petr, organizátor)
8. Merch grid (3 t-shirts × 499 Kč)
9. Partner logos (Hankey Bannister, Bláha, Únětický pivovar, Alkohol.cz)
10. Social icons (FB, IG, email)
11. Footer (copyright, address, unsubscribe)

**Design system:**
- Dark green `#122b1a` (hero, footer)
- Burnt orange `#C45C26` (CTA, accents)
- Gold `#D4A84B` (dividers, badges)
- Parchment `#f2f0eb` (outer background)
- White `#ffffff` (content area)
- Font: Oswald (headings, progressive enhancement) → Helvetica Neue/Arial fallback

**Compatibility:**
- Gmail, Outlook (desktop + new), Apple Mail, iOS Mail, Android
- Responsive: stacks to single column at 600px
- Dark mode: automatic color adaptation
- Outlook: VML bulletproof button + ghost tables

---

## Next Step

Waiting for Petr's **N8N flow** – then adapt the template and workflow together. Tasks #1–4 can be done in parallel.

---

## Key Reference Data

| | |
|---|---|
| **Festival date** | 25.4.2026, 12:00 |
| **Location** | Únětický pivovar, Rýznerova 18, Únětice |
| **Lineup** | Mr. Irish Bastard (DE), Vintage Wine, Foggy Dude, Five Leaf Clover |
| **Tickets** | 300 Kč presale (until 17.4.), 400 Kč at door |
| **Free entry** | Děti do 12 let, ZTP, Úněťáci na občanku |
| **Ticket link** | https://kape.ly/skotsko-v-uneticich |
| **Website** | https://celtic.cz/skotsko-v-uneticich/ |
| **Company** | Bannockburn Entertainments s.r.o. |
| **GitHub repo** | github.com/darthPeter/skotskovuneticich |
| **FB** | facebook.com/skotskovuneticich |
| **IG** | instagram.com/skotsko_unetice/ |
| **Contact** | petr@chotebor.org |
