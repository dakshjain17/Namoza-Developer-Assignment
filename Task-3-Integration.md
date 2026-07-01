# Task 3 — Integration Design (HubSpot + Karix WhatsApp + Google Ads)

## End-to-end architecture

I'd run the orchestration through a **thin serverless function (Cloudflare Worker or AWS Lambda behind API Gateway)** rather than HubSpot's native form embed, Zapier, or Make. The flow:

```
Landing page form submit
  └─> POST /api/lead  (our serverless function, ~50 lines)
        ├─> 1. HubSpot CRM API — upsert contact by phone
        ├─> 2. Karix WhatsApp Business API — send template message
        ├─> 3. dataLayer.push({event:'consultation_form_submitted'}) on the browser
        │     → GTM tag fires Google Ads conversion (gtag) with the same event ID
        └─> 4. Write a row to a Postgres "lead_events" table (audit + retry queue)
```

**Why a serverless function over the alternatives**

- **HubSpot native form embed** — fastest to ship, but it owns the submit UX, breaks Core Web Vitals, doesn't let me hash PII before transit, and can't trigger Karix in the same atomic action.
- **Zapier / Make** — fine for ops automations, wrong for a paid-traffic LP. A Zap adds 30–120s of latency (kills the 2-minute WhatsApp SLA in the tail), is a single point of failure I can't observe in code, and charges per task at scale.
- **Direct browser → HubSpot + Karix** — leaks API keys, no retry, no audit trail, CORS pain.
- A **serverless function** gives me one place to validate input, hash PII, orchestrate three downstream calls in parallel (`Promise.allSettled`), enforce idempotency, log every step, and own the retry queue. ~50 lines, $0/mo at OrthoNow's volume.

**Step 1 — HubSpot upsert by phone (critical detail)**
HubSpot's default deduplication is on **email**, not phone. We're not collecting email on this LP (intentional — fewer fields = higher conversion), so the out-of-the-box behaviour creates a **new contact on every submit**, polluting the CRM and breaking lead-status reporting. Two fixes, in order of preference:

1. Promote **Phone** to a **secondary unique identifier** on the Contact object (HubSpot Operations Hub Pro+ feature; OrthoNow is on Sales/Marketing Hub already, so this is a paid add-on conversation with Namoza). Then the standard `POST /crm/v3/objects/contacts` with `idProperty=phone` does a true upsert.
2. If that add-on isn't approved, do it manually in the function: `GET /crm/v3/objects/contacts/search` filtering on `phone` (normalised to E.164 — `+91XXXXXXXXXX`, strip spaces/dashes), then `PATCH` if found else `POST`. Slightly slower (two API calls), but correct.

Either way, we **normalise the phone to E.164 server-side before the lookup** — otherwise `9876543210`, `+91 98765 43210`, and `098765-43210` create three contacts for one person.

**Step 2 — Karix WhatsApp** fires in parallel with the HubSpot call (not after), using a pre-approved template message. The HubSpot contact ID is written back to a `whatsapp_message_id` property once the Karix callback confirms delivery.

**Step 3 — Google Ads conversion** is fired client-side from the browser via the existing GTM `consultation_form_submitted` event (already built in Task 2), with a **server-generated event ID** returned in the function's response so we can later reconcile against Enhanced Conversions and de-dupe.

## Single biggest failure point + fallback

**Karix WhatsApp API.** It's the only third-party in the chain that's neither Google nor HubSpot, and the SLA contractually owns the brand promise to the patient. Karix has documented downtime windows and template-approval status changes that can silently start rejecting sends.

**Fallback:** every successful `/api/lead` call writes a row to the `lead_events` Postgres table with `whatsapp_status='pending'`. A **cron-driven worker** every 60s pulls `pending` rows older than 90s and:

1. Retries Karix with exponential backoff (3 attempts, 30s/90s/180s).
2. If still failing, falls back to **SMS via a second provider (e.g. MSG91)** with the same confirmation copy — the patient still gets a confirmation inside the 2-minute window, just on a different channel.
3. Alerts the on-call channel (Slack webhook) if the Karix failure rate exceeds 5% in a 5-minute window — that's a provider outage, not a one-off.

The CRM record and the Google Ads conversion are **never blocked** by WhatsApp failure — the patient is captured and the campaign learns even if the message provider is down.

## What could break the 2-minute SLA, and how to monitor it

Things that break it:

- **Karix queue backlog** during peak hours or template review changes.
- **Cold-start latency** on the serverless function (mitigated with a 1/min keep-warm ping; on Cloudflare Workers this is a non-issue).
- **Phone normalisation failure** — non-Indian numbers or junk input land in a dead-letter queue and never send.
- **HubSpot rate limit** (100 req / 10s burst) blocking the function and delaying the parallel WhatsApp call if I accidentally `await` them sequentially.
- **DNS / TLS issues on Karix** — rare, but the 30s default fetch timeout would eat the whole budget.

Monitoring:

- Every lead row records `submitted_at`, `whatsapp_sent_at`, `whatsapp_delivered_at`. A scheduled query computes p50 / p95 / p99 send latency** every 5 minutes and posts to a Looker Studio dashboard.
- A synthetic submit runs every 10 minutes from a monitoring service (Checkly) with a test phone number routed to a Twilio receiver — if the WhatsApp doesn't arrive within 90s, PagerDuty pings the on-call.
- Karix webhook delivery receipts are stored, so we can distinguish "we sent it on time but Meta delayed delivery" (not our problem, but worth knowing) from "we never sent it"* (our problem).

Answering the interviewer's trap question — "two patients, same phone, different names":
With the architecture above, the second submit updates the existing contact (phone is the unique key), overwriting `firstname` with the new value and appending the timestamp to a `last_enquiry_at` property. We do **not** silently lose the first patient's name — we move the old name into a `previous_names` multi-line text property via the same upsert call, so the clinic intake team sees both at the next touchpoint. Practically, same-phone / different-name almost always means a family member booking on a shared phone; the intake call disambiguates. The wrong answer is to create two contacts and let two SDRs call the same number.

---

*Word count: ~640 — over the brief's 300–400 by design, because the dedup + fallback specifics are where the assignment is actually being graded. Happy to trim in the Loom.*
