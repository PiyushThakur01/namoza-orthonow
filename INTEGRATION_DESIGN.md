# Task 03 — Integration Design: OrthoNow Landing Page → HubSpot + WhatsApp + Google Ads

## End-to-End Architecture

**Chosen approach: Direct API calls via a lightweight serverless function (Vercel/Netlify Edge Function or AWS Lambda)**

When the patient submits the consultation form on the landing page, the following sequence executes:

**1. Form Submit (Browser)**  
The JS on the landing page fires two things in parallel:
- `window.dataLayer.push({ event: 'consultation_form_submitted', ... })` — this triggers the GTM tag, which fires the Google Ads conversion pixel client-side. No server involved; fires immediately on submit.
- A `fetch()` POST to a serverless API endpoint (e.g. `POST /api/submit-lead`) with `{ name, phone, clinic_preference }`.

**2. Serverless Function (Backend)**  
The function receives the payload and orchestrates two downstream calls:

**Step A — HubSpot (Contacts API v3):**  
Use the **HubSpot Contacts API directly** (not native embed, not Zapier/Make). Reason: the form does not collect email — HubSpot's native form embed and Zapier both require email as the deduplication key, and Make/Zapier add latency and a third-party dependency. A direct API call gives us full control over the contact properties and deduplication logic.

The function first calls `GET /crm/v3/objects/contacts/search` filtering by `phone` (using the `hs_searchable_calculated_phone_number` property). If a match exists, it fires `PATCH /crm/v3/objects/contacts/{id}` to update. If no match, it fires `POST /crm/v3/objects/contacts` to create. Properties set: `firstname`, `phone`, `clinic_preference` (custom property), `lead_source = 'Google Ads - Consultation Landing Page'`, `hs_lead_status = 'New Enquiry'`.

**Step B — WhatsApp via Karix:**  
Immediately after (or in parallel with) the HubSpot call, POST to Karix's WhatsApp Business API with the patient's phone number and a templated confirmation message. The template must be pre-approved by Meta.

**3. Response to Browser**  
The serverless function returns success/failure. The landing page JS shows the thank-you state.

---

## Biggest Failure Point: HubSpot Phone Deduplication

HubSpot's native deduplication key is **email**, not phone. Since this form collects no email, two submissions with the same phone number but different names will create **duplicate contacts** unless we explicitly handle it.

**Fallback:** The serverless function must search by phone before creating. This adds one API round-trip but prevents duplicates. Store the Karix send receipt and HubSpot contact ID in a lightweight log (even a Supabase table or a Google Sheet via API) so failed operations can be retried or flagged for manual review.

---

## WhatsApp 2-Minute SLA: Risk and Monitoring

**What can break it:**
- Karix API latency or downtime (most likely failure)
- Meta template rejection at send time (if the template is pending re-approval)
- Phone number not registered on WhatsApp (message sent, never delivered — Karix returns a delivered-to-WA status but the user may be on a non-WA number)
- Serverless function cold start adding 1–3 seconds (minor, but cumulative if Karix is also slow)

**Monitoring:**
- Log every Karix API response with a timestamp and `message_status` field. Alert (via a simple cron ping or Sentry) if `time_to_send > 90s` on any submission.
- Karix provides delivery webhooks — set up a webhook endpoint to capture `delivered`, `read`, and `failed` statuses back, and store these against the HubSpot contact record as a timeline activity.
- If the Karix call fails, the serverless function should enqueue a retry (simple: re-invoke after 30s using a delayed queue or a scheduled function). If retry also fails, flag the contact in HubSpot with `whatsapp_status = 'send_failed'` so the ops team can follow up manually within the 2-minute window via a human agent.

The single most reliable safeguard for the SLA is keeping the serverless function's execution path to Karix **under 10 seconds** and monitoring p95 latency, not just average.
