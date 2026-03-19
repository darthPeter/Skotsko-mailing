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
| 5 | **N8N test workflow** | id: `3ASCdHgg08fPP5rS` — simple test sender to `chotebor.p@gmail.com` |
| 6 | **Test sends** | Multiple successful sends, template verified on mobile |
| 7 | **Template reviewed** | Pricing, merch names, copy, links — all iterated and approved |
| 8 | **SPF/DKIM/DMARC** | celtic.cz verified in SendGrid |
| 9 | **ASM suppression group** | "Celtic.cz - Unsubscribed", group ID: **28696**, unsubscribers uploaded |
| 10 | **Open + click tracking** | Enabled in SendGrid |
| 11 | **Supabase contacts table** | 434 contacts imported, batches assigned |
| 12 | **Production N8N workflow** | id: `lcE8F0gUUyuLodGZ` — full loop with ASM, Supabase read/update |
| 13 | **Supabase credentials connected** | Both Read Contacts and Mark Sent nodes |

### TODO (Sendout)

| # | Task | Who | Status |
|---|------|-----|--------|
| 1 | **Test: Batch 1** (5 test contacts) — full production flow verified | Petr | DONE |
| 2 | **Batch 2** (50 contacts) — sent, execution #8373, 13s | Petr | DONE |
| 3 | **Batch 3** (100 contacts) — sent, execution #8399, 25s | Petr | DONE |
| 4 | **Batch 4** (150 contacts) — sent, execution #8500, 37s | Petr | DONE |
| 5 | **Batch 5** (133 contacts) — sent, execution #8551 | Petr | DONE |

**ALL 434 CONTACTS SENT.** Campaign complete.

---

## Architecture

```
┌─────────────┐     ┌─────────┐     ┌───────────┐
│  Supabase   │────▶│   N8N   │────▶│ SendGrid  │────▶ Inbox
│  (contacts) │     │  (flow) │     │  (API)    │
└─────────────┘     └─────────┘     └───────────┘
                         │
                  ┌──────┴──────┐
                  │  Template   │
                  │ (GitHub     │
                  │  Pages)     │
                  └─────────────┘
```

### Supabase table: `contacts`

| Column | Type | Notes |
|--------|------|-------|
| `id` | uuid, PK | auto-generated |
| `email` | text, unique | contact email |
| `osloveni` | text | personalized greeting ("Petře", "Honzo", etc.) — can be empty |
| `subscribed` | boolean | default true |
| `batch` | integer 1-5 | 1=test(1), 2=first(50), 3=second(100), 4=third(150), 5=rest(133) |
| `sent` | boolean | default false, flipped to true after send (prevents double-sending) |
| `created_at` | timestamptz | auto-generated |

### N8N workflows

**Test flow** (id: `3ASCdHgg08fPP5rS`, name: "Skotsko e-mail (TEST)"):
- Simple webhook sender, hardcoded to `chotebor.p@gmail.com`
- Use for quick template iteration via Claude or n8n

**Production flow** (id: `lcE8F0gUUyuLodGZ`, name: "Skotsko e-mail (PRODUCTION)"):

```
Run (Manual Trigger)
  → Set Batch Number (value 1-5, Petr changes before each run)
    → Fetch Template (HTTP GET from GitHub Pages)
      → Read Contacts (Supabase: batch=N AND sent=false AND subscribed=true)
        → Loop Over Contacts (Split In Batches, 1 at a time)
            ↓ output 1 (loop item)
            Build Email (Code: inject osloveni, build SendGrid API body with ASM)
              → Send via SendGrid (HTTP POST to SendGrid v3 API)
                → Prepare Update (Set: contactId → id, sent=true)
                  → Mark Sent (Supabase: UPDATE contacts SET sent=true WHERE id=X)
                    → [back to Loop Over Contacts]
            ↓ output 0 (done)
            [end]
```

**Node details:**

| Node | Type | Key config |
|------|------|------------|
| Run | Manual Trigger | Petr clicks Execute Workflow in n8n editor |
| Set Batch Number | Set | `batch` = 1 (default). Change to 2-5 for real batches |
| Fetch Template | HTTP Request | GET `darthpeter.github.io/Skotsko-mailing/templates/email-2026-predprodej.html` |
| Read Contacts | Supabase (getAll) | Filter: `batch=eq.{N}&sent=is.false&subscribed=is.true`. Credential: `KmCWPRRYPk1x2Rxd` |
| Loop Over Contacts | Split In Batches v3 | batchSize=1. Output 0=done, Output 1=loop items |
| Build Email | Code | Replaces `{{osloveni}}` (empty→blank). Skips if sent=true. Builds full SendGrid API body with ASM group 28696 |
| Send via SendGrid | HTTP Request | POST `api.sendgrid.com/v3/mail/send`. Uses predefined SendGrid credential `pSDSmCCOIPiKXOo2`. Body = `JSON.stringify($json.sendgridBody)` |
| Prepare Update | Set | Maps `contactId` → `id`, sets `sent` = true |
| Mark Sent | Supabase (update) | Filter: `id=eq.{uuid}`. Data: `sent=true` (autoMap, ignores id). Credential: `KmCWPRRYPk1x2Rxd` |

**Why HTTP Request instead of SendGrid node:**
The n8n SendGrid node doesn't support ASM groups. We call the SendGrid v3 API directly to include `asm.group_id: 28696` in the request body. This makes the `<%asm_group_unsubscribe_raw_url%>` and `<%asm_preferences_raw_url%>` tags in the template work.

**Safeguards (4 layers):**
1. **Manual Trigger** — nothing fires without Petr opening n8n and clicking Execute
2. **Supabase query** filters `sent=false` — already-sent contacts never returned from DB
3. **Code node** double-checks `sent !== true` — extra safety before sending
4. **ASM group 28696** — SendGrid blocks unsubscribed recipients server-side

**Credentials:**
| Credential | ID | Used by |
|-----------|-----|---------|
| Supabase (SkotskoMailing) | `KmCWPRRYPk1x2Rxd` | Read Contacts, Mark Sent |
| SendGrid (SendGrid account) | `pSDSmCCOIPiKXOo2` | Send via SendGrid |

### Batching strategy

434 contacts total. Domain celtic.cz has history (used with Mailchimp last year, 50% open rate). Moving to SendGrid = new sending infrastructure, so mild warmup.

| Day | Batch | Size | Action |
|-----|-------|------|--------|
| Test | #1 | 5 | Test emails (chotebor.p@gmail.com + 4 others) — DONE ✅ |
| Day 1 | #2 | 50 | First real batch, monitor 2-3 hours |
| Day 1 | #3 | 100 | If #2 clean (bounces <2%, no spam reports) |
| Day 2 | #4 | 150 | Next batch |
| Day 2 | #5 | 133 | Remaining |
| | **Total** | **434** | |

**Kill switch:** If any batch has >5% bounces or spam reports → stop, investigate.

---

## Email Template Details

**File:** `templates/email-2026-predprodej.html`
**Preview:** `https://darthpeter.github.io/Skotsko-mailing/templates/email-2026-predprodej.html`

**Sections:**
1. Full-width header banner (FB cover image with lineup)
2. Personalized greeting (`{{osloveni}}`) — empty = just "Ahoj ,"
3. Festival announcement with Mr. Irish Bastard YT link
4. Pricing callout: Předprodej 300 Kč / Na místě 400 Kč / ends 17.4.
5. CTA button "Chci lístek!" → `celtic.cz/`
6. Info links (celtic.cz, Facebook, FB události)
7. Signature (Petr, organizátor)
8. Merch grid: Únětický Trooper 2026 | The Beer for All | Festivalové OG (all 499 Kč → `kape.ly/skotsko-v-uneticich#merch`)
9. Partner logos (Hankey Bannister, Bláha, Únětický pivovar, Alkohol.cz)
10. Social icons (FB, IG, email)
11. Footer (copyright, address, SendGrid ASM unsubscribe/preferences links)

**Design system:**
- Dark green `#122b1a` (hero, footer)
- Burnt orange `#C45C26` (CTA, accents, MIB link)
- Gold `#D4A84B` (dividers, pricing badge border)
- Parchment `#f2f0eb` (outer background)
- White `#ffffff` (content area)
- Font: Oswald 700 (headings) → Helvetica Neue/Arial fallback

**Compatibility:**
- Gmail, Outlook (desktop + new), Apple Mail, iOS Mail, Android
- Responsive: single column at 600px, merch images full-width on mobile
- Dark mode: `prefers-color-scheme` media query
- Outlook: VML bulletproof CTA button + ghost tables

**Links in template:**
| Link | Target |
|------|--------|
| Banner | `celtic.cz/skotsko-v-uneticich/` |
| CTA "Chci lístek!" | `celtic.cz/` |
| Mr. Irish Bastard | `youtube.com/@MrIrishBastardOfficial` |
| Merch images + Koupit | `kape.ly/skotsko-v-uneticich#merch` |
| "celtic.cz" info link | `celtic.cz/skotsko-v-uneticich/` |
| "facebooku festivalu" | `facebook.com/skotskovuneticich` |
| "FB události" | `facebook.com/skotskovuneticich` |
| Social: FB | `facebook.com/skotskovuneticich` |
| Social: IG | `instagram.com/skotsko_unetice/` |
| Social: Email | `petr@chotebor.org` |
| Unsubscribe | `<%asm_group_unsubscribe_raw_url%>` (SendGrid ASM) |
| Preferences | `<%asm_preferences_raw_url%>` (SendGrid ASM) |

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

### One-time setup (already done)

1. ~~Supabase credentials connected in Read Contacts + Mark Sent~~ ✅
2. ~~SendGrid credentials connected in Send via SendGrid~~ ✅
3. Workflow stays **inactive** — you run it manually each time

### Sending a batch

1. Open n8n → workflow **"Skotsko e-mail (PRODUCTION)"** (`lcE8F0gUUyuLodGZ`)
2. Click **"Set Batch Number"** node → change the `batch` value (1-5)
3. Click **"Execute Workflow"** (play button top-right)
4. Watch the nodes light up green as they loop through contacts
5. Check **SendGrid → Activity** for delivery stats

### Batch schedule

| Batch | Size | When |
|-------|------|------|
| 1 | 1 | **TEST FIRST** — just your email, verify everything works + unsubscribe link |
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

- **Workflow failed mid-batch?** Safe to rerun — contacts already sent have `sent=true` and won't be sent again.
- **Need to resend to someone?** In Supabase: `UPDATE contacts SET sent = false WHERE email = 'their@email.com'`
- **Want to skip someone?** In Supabase: `UPDATE contacts SET subscribed = false WHERE email = 'their@email.com'`
- **Template change needed?** Edit in GitHub repo (`templates/`), push, wait ~1 min for Pages deploy. Workflow fetches latest template every run.
- **Supabase filter issue?** Boolean columns use `is` operator (not `eq`). If `sent` is `null`, run: `UPDATE contacts SET sent = false WHERE sent IS NULL`
- **Gmail "show trimmed content"?** Gmail conversation threading collapses repeated content when multiple test emails have the same subject. Won't happen for real recipients. View in new window to see full email.

### Quick test (without production workflow)

Use test workflow **"Skotsko e-mail (TEST)"** (`3ASCdHgg08fPP5rS`) — always sends to `chotebor.p@gmail.com` only. Trigger via Claude or n8n.
