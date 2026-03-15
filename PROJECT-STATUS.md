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

| # | Task | Depends on | Priority |
|---|------|-----------|----------|
| 0 | **Create `Skotsko-mailing` GitHub repo** – host templates, images (GitHub Pages), project docs | — | HIGH |
| 1 | **Upload banner image** to `Skotsko-mailing` repo + enable GitHub Pages for image hosting | #0 | HIGH |
| 2 | **Update image URLs** in email template to point to `Skotsko-mailing` GitHub Pages (not production repo) | #0, #1 | HIGH |
| 3 | **Set up Supabase table** for contacts | Supabase account | HIGH |
| 4 | **Import contacts** from Mailchimp export → Supabase | #3 | HIGH |
| 5 | **Set up SendGrid ASM** (suppression group) for unsubscribe handling | SendGrid account | HIGH |
| 6 | **Build N8N workflow** for email sendout | #3, #4, #5 | HIGH |
| 7 | **Test send** to a small batch | #1, #2, #6 | HIGH |
| 8 | **Full sendout** to entire contact list | #7 | HIGH |

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

### N8N workflow (to be built)

Petr will provide his existing flow to adapt. Expected shape:
- **Trigger:** manual or scheduled
- **Read contacts** from Supabase (where `subscribed = true`)
- **Loop/batch:** for each contact, send via SendGrid API
- **Template:** inject `{{osloveni}}` into the HTML template
- **SendGrid ASM** group ID for unsubscribe handling

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
