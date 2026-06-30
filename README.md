# OrthoNow — Developer Assignment
**Namoza · Developer Position 1 (Client Web + Martech)**

---

## Repo Structure

```
/
├── task1/
│   └── GTM_EVENT_SCHEMA.md       # Full event schema + dataLayer JSON + GA4 setup
├── task2/
│   └── index.html                # Single-file landing page (HTML/CSS/vanilla JS)
├── task3/
│   └── INTEGRATION_DESIGN.md    # HubSpot + WhatsApp + Google Ads integration writeup
└── README.md                     # This file
```

---

## Task 01 — GTM Event Schema
See [`task1/GTM_EVENT_SCHEMA.md`](./task1/GTM_EVENT_SCHEMA.md)

Covers:
- Full 8-event schema with triggers, key parameters, and GA4 report mapping
- 3-step booking form funnel: dataLayer push JSON for each step, GTM trigger config, and GA4 Funnel Exploration setup
- Rationale for `consultation_form_submitted` as the Google Ads import conversion

**Key design decision:** The schema explicitly documents that multi-step form tracking requires front-end `window.dataLayer.push()` calls — GTM cannot detect step transitions natively. The briefing for the front-end dev is included in the markdown.

---

## Task 02 — Landing Page
See [`task2/index.html`](./task2/index.html) · [Live demo](#) *(deploy to Netlify/Vercel to get the live URL)*

Open `task2/index.html` directly in a browser — no build step, no server required.

**To verify the GTM dataLayer push:**
1. Open the page in Chrome
2. Open DevTools → Console
3. Fill in name and a 10-digit phone number
4. Click "Book my free consultation →"
5. You'll see the `[OrthoNow] dataLayer push fired:` log with the event object

**Performance targets:**
- Zero external image requests (logo and avatar are CSS/SVG)
- Single Google Fonts request with `display=swap` (non-render-blocking)
- Font size on inputs is 16px — prevents iOS Safari zoom
- No JavaScript frameworks — total page weight < 12KB

*PageSpeed Insights screenshot: see `task2/pagespeed.png` (add after running test on the deployed URL)*

**Conversion architecture decisions:**
- Headline addresses the user's actual problem, not the brand ("Stop managing the pain")
- 2-field form only (Name + Phone) — every additional field costs conversion rate
- Trust strip uses specific numbers (9 clinics, 18+ specialists, 48h wait) — specific > generic
- Doctor quote card with credential > star ratings for healthcare trust
- CTA copy is outcome-oriented ("Book my free consultation") not action-oriented ("Submit")
- Thank-you state fires without reload — no lost analytics session, no jarring transition

---

## Task 03 — Integration Design
See [`task3/INTEGRATION_DESIGN.md`](./task3/INTEGRATION_DESIGN.md)

Covers:
- Full architecture: landing page → serverless function → HubSpot Contacts API + Karix WhatsApp API
- Why direct API calls over Zapier/Make/native embed (phone-first dedup, no email required)
- **HubSpot deduplication trap:** HubSpot deduplicates on email by default. Since this form collects no email, the serverless function must search by phone number before creating a contact to prevent duplicates.
- WhatsApp 2-minute SLA: failure modes, monitoring approach, and retry fallback

---

## Submission
- GitHub: *[https://github.com/PiyushThakur01/namoza-orthonow]*
- Loom walkthrough: *[https://www.loom.com/share/7825a301db9b4e5a9dcc5c27fc9a86a3]*
- Live Landing Page: https://6a42671821903b6e2963463e--profound-liger-50a681.netlify.app/
- Sent to: naman@namoza.com
