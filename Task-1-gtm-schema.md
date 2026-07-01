# Task 1 — GTM Event Schema for OrthoNow

## 1. Full event schema

All events are pushed to `window.dataLayer` and picked up in GTM via **Custom Event** triggers (unless noted). Built-in triggers (Click, Form, History) are used only where the interaction is reliably observable from the DOM without dev involvement.

| # | Event Name | Trigger Type (GTM) | Key Parameters (min. 3) | Feeds into (GA4) |
|---|---|---|---|---|
| 1 | `page_view` (enhanced) | History Change + Initialization | `page_location`, `page_title`, `clinic_city` (from URL), `content_group` (home / clinic / blog / landing) | GA4 Pages & Screens report; **Content Group** dimension; baseline for all comparisons |
| 2 | `clinic_page_view` | Custom Event (fired by site on the 9 clinic pages) | `clinic_id`, `clinic_name`, `clinic_city`, `specialties_listed` | Custom report *Clinic performance*; **Audience**: "Visited a clinic page in last 30 days" for remarketing |
| 3 | `call_now_click` | Click - Just Links, URL matches `^tel:` | `call_source` (homepage / clinic_page / landing), `clinic_name`, `phone_number_masked`, `page_location` | GA4 Key Event; Google Ads call conversion; Audience: "Called but didn't book" |
| 4 | `whatsapp_click` | Click - Just Links, URL contains `wa.me` | `whatsapp_source`, `clinic_name` (if on clinic page), `message_template_id`, `page_location` | GA4 Key Event; Channel attribution report |
| 5 | `patient_guide_form_submit` | Custom Event | `guide_name`, `lead_name_hashed`, `lead_phone_hashed`, `page_location` | GA4 Key Event = conversion; Audience: "Downloaded guide, didn't book" → nurture campaign |
| 6 | `patient_guide_downloaded` | Custom Event (fires after PDF link click resolves) | `guide_name`, `file_size_kb`, `lead_phone_hashed` | Used in Funnel Exploration: form_submit → download |
| 7 | `booking_form_start` | Custom Event (fires when Step 1 renders / first field focused) | `clinic_location_preselected`, `entry_source`, `device_category`, `page_location` | Top of booking funnel in Funnel Exploration |
| 8 | `booking_step_complete` | Custom Event | `step_number`, `step_name`, plus step-specific fields (see §2) | **Funnel Exploration** step-by-step drop-off |
| 9 | `booking_form_submitted` | Custom Event (Step 3 confirm) | `clinic_location`, `specialty`, `appointment_date`, `lead_phone_hashed`, `booking_id` | **Primary conversion** — imported to Google Ads |
| 10 | `booking_form_error` | Custom Event | `step_number`, `field_name`, `error_type` (validation / api), `error_message` | Diagnostic — why people drop |
| 11 | `scroll_depth` | Built-in Scroll Depth Trigger (25 / 50 / 75 / 90) | `percent_scrolled`, `article_id`, `article_category`, `read_time_seconds` | GA4 *Article engagement* report; Audience: "Read ≥75% of a back-pain article" |
| 12 | `article_read_complete` | Custom Event (fires at 90% + 30s dwell) | `article_id`, `article_category`, `author`, `word_count` | Engaged audiences for remarketing |
| 13 | `outbound_click` | Click - All Links, hostname ≠ orthonow.in | `outbound_url`, `link_text`, `page_location` | Hygiene / referral monitoring |

>   Notes
> - Phone and name are **hashed (SHA-256) client-side** before any dataLayer push. PII never enters GA4 in raw form.
> - `clinic_*` parameters are registered as **custom dimensions** in GA4 (event-scoped) so they're usable in Explorations and Audiences.

---

## 2. Booking form — funnel tracking (3 steps)

### Who writes the dataLayer push?

The front-end developer writes it, not the GTM/marketing engineer. GTM has no native way to observe state changes inside a multi-step React/JS form — there is no page load, no URL change, and the "Next" button is a generic `<button>` that GTM cannot semantically distinguish per step. The only reliable signal is an explicit `window.dataLayer.push()` emitted by the app's step-transition handler.

My brief to the front-end dev for Step 2 would be:

> In the handler that runs **after** Step 2's client-side validation passes and **before** you advance UI state to Step 3, call:
>
> ```js
> window.dataLayer = window.dataLayer || [];
> window.dataLayer.push({ event: 'booking_step_complete', step_number: 2, step_name: 'patient_details_entered', /* …params below… */ });
> ```
>
> Fire it **once per successful advance** (debounce on rapid double-clicks). Do **not** fire it on Back navigation. If validation fails, fire `booking_form_error` instead with `step_number: 2` and the field that failed. PII (`name`, `phone`) must be SHA-256 hashed before push; raw values stay in component state only.

### GTM triggers

- One Custom Event trigger per step: `Event equals booking_step_complete` AND `step_number equals 1 | 2 | 3`. Three triggers, three tags → three GA4 events (`booking_step_1`, `booking_step_2`, `booking_step_3`) — *or* a single GA4 event with `step_number` as a parameter and use GA4 Funnel Exploration's step conditions. I prefer the single-event approach: cleaner schema, easier to add steps later.

### dataLayer pushes (actual JSON)

Step 1 — Location & specialty selected
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar, Bengaluru",
  "clinic_id": "ortho-blr-04",
  "specialty": "Knee Pain",
  "entry_source": "google_ads_consultation_lp"
}
```

Step 2 — Patient details entered
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee Pain",
  "lead_name_hashed": "e3b0c44298fc1c149afbf4c8996fb924...",
  "lead_phone_hashed": "a665a45920422f9d417e4867efdc4fb8...",
  "preferred_date": "2026-07-04",
  "preferred_slot": "evening"
}
```

**Step 3 — Booking confirmed (also fires `booking_form_submitted`)**
```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee Pain",
  "appointment_date": "2026-07-04",
  "appointment_slot": "evening",
  "booking_id": "BK-20260630-7421",
  "lead_phone_hashed": "a665a45920422f9d417e4867efdc4fb8..."
}
```

### Surfacing drop-off in GA4 Funnel Exploration

In **Explore → Funnel Exploration**, build an **open funnel** with 4 steps:

1. `booking_form_start`
2. `booking_step_complete` where `step_number = 1`
3. `booking_step_complete` where `step_number = 2`
4. `booking_step_complete` where `step_number = 3`

Enable **"Show elapsed time"** between steps and break the funnel down by `clinic_location`, `specialty`, and `device_category`. This shows both the **% drop** between each step and **where time-to-completion balloons** — usually the smoking gun for a friction point (e.g. Step 2 phone field on iOS Safari).

---

## 3. The one conversion I'd import into Google Ads — and why

**`booking_form_submitted`** (Step 3, the confirmed booking).

Why this one and not, say, `call_now_click` or `patient_guide_form_submit`:

- It's the **only event tied to revenue intent**. A booked consultation is the act OrthoNow charges for; everything else is a proxy.
- Google's Smart Bidding is only as smart as the signal it optimises toward. Optimising to `call_now_click` rewards Google for sending people who tap a phone number and never connect; optimising to `patient_guide_form_submit` rewards top-of-funnel lead magnets that don't convert to revenue. Either of those will train the algorithm toward cheap, low-quality conversions.
- It happens with high enough volume on a healthcare LP (target 6–8% of LP traffic) to give Smart Bidding usable conversion density within 2–3 weeks.
- `call_now_click` should still be tracked as a **secondary** conversion (observation only, not bidding) so we can attribute call-driven revenue post-hoc via CRM match-back. `patient_guide_form_submit` is a remarketing audience, not a bid signal.
