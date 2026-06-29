# Task 01 — GTM Event Schema: OrthoNow

## Full Event Schema

| Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
|---|---|---|---|
| `booking_step_complete` | Custom Event (dataLayer push) | `step_number`, `step_name`, `clinic_location`, `specialty` (step 1); `preferred_date` (step 2); `booking_id` (step 3) | Funnel Exploration (3-step funnel); Remarketing audience: users who abandoned at step 1 or 2 |
| `call_now_click` | Click Trigger (CSS selector or Click Text) | `click_location` (homepage / clinic_page / landing_page), `clinic_name`, `phone_number` | Conversions report; GA4 Event count by page; Audience: clicked but didn't book |
| `whatsapp_widget_click` | Click Trigger (outbound link or element ID) | `click_location`, `wa_number`, `page_url` | Engagement report; Audience: WhatsApp-engaged users (retargeting) |
| `patient_guide_form_submit` | Custom Event (dataLayer push on form submit) | `form_name` ('patient_guide_gate'), `clinic_interest`, `page_url` | Lead gen report; Audience: high-intent top-of-funnel (downloaded guide but not booked) |
| `patient_guide_download` | Custom Event (dataLayer push after gate cleared) | `guide_name`, `file_url`, `user_phone_captured` (boolean) | Content engagement; combined with `patient_guide_form_submit` for gate-to-download rate |
| `clinic_page_view` | Page View Trigger (URL path filter `/clinic/`) | `clinic_name`, `clinic_city`, `page_path` | Location-level traffic report; Audience: users interested in a specific clinic |
| `blog_scroll_depth` | Scroll Depth Trigger (25%, 50%, 75%, 90%) | `scroll_threshold`, `article_title`, `article_category`, `page_url` | Engagement / User Lifetime report; Audience: high-scroll readers (re-engage with booking CTA) |
| `consultation_form_submitted` | Custom Event (dataLayer push on landing page form submit) | `form_location` ('landing_page'), `clinic_preference`, `lead_source` | Primary conversion action imported to Google Ads (see section below) |

---

## 3-Step Booking Form: Funnel Drop-Off Tracking

### Architecture Note

GTM **cannot** natively detect multi-step form interactions. There are no built-in triggers for "user completed step 2 of a multi-step form." The front-end developer must instrument each step transition with a `window.dataLayer.push()` call. GTM then listens for these custom events using a **Custom Event trigger** matching the event name `booking_step_complete`.

---

### Briefing the Front-End Developer

At each step transition (i.e., when the user successfully passes validation and the UI advances to the next step), the dev must fire:

```javascript
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({ <your push object here> });
```

This must fire **after validation passes**, not on button click (a user clicking "Next" and getting a validation error should not count as a step completion).

---

### Step 1 — Location & Specialty Selected

Fires when the user selects a clinic location and specialty and clicks "Next" (post-validation).

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement"
}
```

**GTM Trigger:** Custom Event — Event Name equals `booking_step_complete`  
**GTM Variable:** Data Layer Variable `step_number` (used to differentiate steps in GA4)

---

### Step 2 — Patient Details Entered

Fires when the user submits their name, phone, and preferred date and the UI advances to the confirmation screen.

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement",
  "preferred_date": "2025-08-14",
  "has_phone": true
}
```

**Note:** Do NOT push raw phone/name values into the dataLayer — this is PII. Use boolean flags like `has_phone: true` to confirm capture without logging personal data to GA4/GTM.

---

### Step 3 — Booking Confirmed

Fires when the backend confirms the booking and the success screen renders.

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Koramangala, Bengaluru",
  "specialty": "Knee Replacement",
  "booking_id": "ON-20250814-4471",
  "preferred_date": "2025-08-14"
}
```

---

### Surfacing Drop-Off in GA4 Funnel Exploration

1. Go to **GA4 → Explore → Funnel Exploration**
2. Set **Funnel type:** Standard (closed or open depending on whether you want to count re-entries)
3. Define 3 steps using **Event** conditions:
   - Step 1: `booking_step_complete` AND `step_number = 1`
   - Step 2: `booking_step_complete` AND `step_number = 2`
   - Step 3: `booking_step_complete` AND `step_number = 3`
4. Add **Breakdown dimension:** `clinic_location` — this surfaces which clinics have the worst drop-off
5. Enable **"Show elapsed time"** to see how long users spend between steps (useful for diagnosing a slow/confusing step 2)

Drop-off percentage between steps 1→2 and 2→3 are the key metrics. Step 1→2 drop-off typically signals friction in the form fields; step 2→3 drop-off often indicates trust issues at confirmation.

---

## Google Ads Conversion Action: Why `consultation_form_submitted`

**Chosen conversion:** `consultation_form_submitted` (from the landing page)

**Reasoning:**

1. **Closest to revenue.** A completed consultation form is a declared intent to visit the clinic — it is the action the paid campaign is directly purchasing. Importing `booking_step_complete` (step 3) from the main site would be ideal if the campaign drives to the booking flow, but for a campaign landing page, form submission is the bottom-of-funnel signal.

2. **Clean attribution.** The form is on a campaign-specific landing page with UTM parameters, so Google Ads can attribute this conversion directly to the ad click without multi-touch ambiguity.

3. **Volume vs. quality balance.** `call_now_click` has higher volume but lower intent signal (someone can click by accident). `booking_confirmed` (step 3) is higher quality but lower volume — too sparse to give Smart Bidding enough signal at typical healthcare CPCs. `consultation_form_submitted` sits in the right position on the intent-volume curve.

4. **Smart Bidding compatibility.** With Target CPA or Maximize Conversions bidding, Google needs ≥30 conversions/month in the conversion window to optimise. At 2.1% conversion rate on a modest budget, form submissions will hit this threshold faster than confirmed bookings.

**Import method:** Via GA4 → Google Ads linked property → mark `consultation_form_submitted` as a conversion in GA4, then import into Google Ads under Tools → Conversions → Import from GA4.
