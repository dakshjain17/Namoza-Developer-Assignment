# Namoza Developer Assignment — OrthoNow

Candidate: [Daksh Jain]
Role: Developer – Position 1 (Client Web + Martech)

This repo contains the three deliverables for the Namoza Developer assignment.

## Contents

| File | Task |
|------|------|
| [`Task-1-gtm-schema.md`](./Task-1-gtm-schema.md) | GTM event schema + booking funnel JSON + Google Ads conversion pick |
| [`Task-2-index.html`](./Task-2-index.html) | Single-file landing page (HTML + CSS + vanilla JS) |
| [`Task-2-pagespeed.png`](./Task-2-pagespeed.png) | PageSpeed Insights Mobile screenshot (90+) |
| [`Task-3-integration.md`](./Task-3-integration.md) | HubSpot + Karix WhatsApp + Google Ads integration writeup |

## How to run Task 2 locally

Open `task-2-landing-page.html` directly in any browser — no build step, no server.
To verify the dataLayer push on submit:

1. Open Chrome DevTools → Console
2. Run `window.dataLayer = window.dataLayer || []` (already initialised on the page)
3. Fill Name + Phone, click Book my consultation
4. Console logs the pushed event; the form swaps to a thank-you state without reload.
