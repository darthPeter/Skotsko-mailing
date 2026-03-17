# Skotsko v Úněticích 2026 – Email Sender Pipeline

## Context

Moving away from Mailchimp (database too large/expensive) to a self-hosted email sending pipeline: **Supabase** (contact DB) → **N8N** (automation) → **SendGrid/Twilio** (delivery). Goal: send the pre-sale announcement email for the 2026 celtic punk festival (25.4.2026).

## CRITICAL RULES

- **NEVER send any email to real clients without explicit confirmation from Petr.** Test only with `chotebor.p@gmail.com`. Be extremely careful not to trigger bulk sends by mistake.
- **Production workflow uses MANUAL TRIGGER only.** Petr manually enters batch number and clicks Execute. No webhooks, no automation. This is the safeguard.
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
| 1 | **Verify SPF/DKIM/DMARC** for celtic.cz | Petr | DONE |
| 2 | **Create ASM suppression group** — "Celtic.cz - Unsubscribed", group ID: **28696** | Petr | DONE |
| 3 | **Wire ASM group ID into N8N workflow** | Claude | DONE |
| 4 | **Upload existing unsubscribers** to ASM group | Petr | DONE |
| 5 | **Enable open tracking** | Petr | DONE |
| 6 | **Enable click tracking** | Petr | DONE |
| | **--- Database ---** | | |
| 6 | **Set up Supabase table** `contacts` (email, osloveni, subscribed, batch, sent) | Petr/Claude | DONE |
| 7 | **Export + import contacts** to Supabase (434 contacts) | Petr | DONE |
| 8 | **Assign batch numbers** (1=test, 2=50, 3=100, 4=150, 5=133) | Petr/Claude | DONE |
| | **--- N8N workflow ---** | | |
| 9 | **Add ASM group ID to N8N workflow** (28696) | Claude | DONE |
| 10 | **Build production workflow** `lcE8F0gUUyuLodGZ` | Claude | DONE |
| 11 | **Connect Supabase credentials** in "Read Contacts" + "Mark Sent" nodes | Petr | TODO |
| 12 | **Activate production workflow** (toggle on in n8n) | Petr | TODO |
| | **--- Sendout ---** | | |
| 13 | **Test: Batch 1** (1 contact = chotebor.p@gmail.com) — verify full flow | Petr | TODO |
| 14 | **Batch 2** (50 contacts) — send, wait 2-3h, check stats | Petr | TODO |
| 15 | **Batch 3** (100 contacts) — if #14 clean (bounces <2%, no spam) | Petr | TODO |
| 16 | **Batch 4** (150 contacts) — next day | Petr | TODO |
| 17 | **Batch 5** (133 contacts) — remaining | Petr | TODO |

### Batching strategy (500 contacts, existing domain reputation)

Domain celtic.cz was used with Mailchimp last year (50% open rate). Audience = ticket buyers. Moving to SendGrid = new sending infrastructure, so mild warmup needed.

| Day | Batch | Size | Contents |
|-----|-------|------|----------|
| Test | #1 | 1 | Just `chotebor.p@gmail.com` — verify full flow works |
| Day 1 | #2 | 50 | First real batch, monitor 2-3 hours |
| Day 1 | #3 | 100 | If #2 clean (bounces <2%, no spam reports) |
| Day 2 | #4 | 150 | Next batch |
| Day 2 | #5 | 133 | Remaining |
| | **Total** | **434** | |

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
- Simple test sender, hardcoded to `chotebor.p@gmail.com`
- Use for quick template iteration

**Production flow** (id: `lcE8F0gUUyuLodGZ`, name: "Skotsko e-mail (PRODUCTION)"):
```
Form Trigger (Enter Batch Number)
  → Fetch Template (GitHub Pages)
    → Read Contacts (Supabase: batch=N, sent=false, subscribed=true)
      → Build Emails (Code: inject osloveni per contact, handle empty)
        → Send via SendGrid (ASM group 28696)
          → Prepare Update (Code: get contactId)
            → Mark Sent (Supabase: set sent=true)
```

**Nodes detail:**
| Node | Type | Purpose |
|------|------|---------|
| Enter Batch Number | Form Trigger | Web form with one field — Petr types batch number and submits |
| Fetch Template | HTTP Request | GETs template from GitHub Pages |
| Read Contacts | Supabase | Gets contacts WHERE batch=N AND sent=false AND subscribed=true |
| Build Emails | Code | Replaces `{{osloveni}}` (handles empty), sets recipient/subject/html per contact |
| Send via SendGrid | SendGrid | Sends from `petr@celtic.cz` as "Skotsko v Úněticích", ASM group 28696 |
| Prepare Update | Code | Maps contactId for database update |
| Mark Sent | Supabase | Updates `sent=true` for each successfully sent contact |

**Safeguards:**
- Form Trigger = manual only, no automation, no webhook
- Petr must type batch number and click Submit
- `sent=true` after each send prevents double-sending
- `subscribed=true` filter respects unsubscribes
- ASM group 28696 = SendGrid also blocks server-side
- Batch 1 = only `chotebor.p@gmail.com` for testing

**Credentials needed:**
- SendGrid: `pSDSmCCOIPiKXOo2` (already connected)
- Supabase: must be connected manually in "Read Contacts" and "Mark Sent" nodes

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

---

## User Manual: How to Send Emails

### One-time setup (do once)

1. Open n8n → workflow **"Skotsko e-mail (PRODUCTION)"** (`lcE8F0gUUyuLodGZ`)
2. Click **"Read Contacts"** node → connect your Supabase credentials
3. Click **"Mark Sent"** node → connect your Supabase credentials (same ones)
4. **Activate** the workflow (toggle top-right)
5. The form trigger URL will appear — bookmark it

### Sending a batch

1. Open the form trigger URL in your browser
2. Enter the **batch number** (1-5)
3. Click **Submit**
4. Wait for the workflow to finish — the form will show a completion message
5. Check your SendGrid Activity Feed for delivery stats

### Batch schedule

| Batch | Size | When |
|-------|------|------|
| 1 | 1 | **TEST FIRST** — just your email, verify everything works |
| 2 | 50 | Day 1 — first real send, wait 2-3 hours |
| 3 | 100 | Day 1 — if batch 2 looks clean |
| 4 | 150 | Day 2 |
| 5 | 133 | Day 2 |

### What to check between batches

Go to **SendGrid → Activity**:
- **Bounce rate** should be <2%
- **Spam reports** should be 0
- **Opens** should start appearing within 1 hour

If bounce rate >5% or any spam reports → **STOP** and investigate before next batch.

### If something goes wrong

- **Workflow failed mid-batch?** Safe to rerun the same batch number — contacts already sent have `sent=true` and won't be sent again.
- **Need to resend to someone?** In Supabase, set their `sent` back to `false`.
- **Want to skip someone?** In Supabase, set their `subscribed` to `false`.
- **Template change needed?** Update in GitHub repo (`Skotsko-mailing/templates/`), push, wait 1 min for Pages deploy, then send. The workflow fetches the latest template every time.

### Quick test (without production workflow)

Use the test workflow **"Skotsko e-mail (TEST)"** (`3ASCdHgg08fPP5rS`) — always sends to `chotebor.p@gmail.com` only. Trigger via Claude or n8n webhook.
