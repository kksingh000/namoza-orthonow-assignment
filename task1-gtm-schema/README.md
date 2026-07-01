# Task 01 - GTM Event Schema: OrthoNow

## Context

OrthoNow has GA4 configured for pageviews only - no GTM container, no event tracking, no conversion data feeding paid campaigns. This is the full event schema to implement before any paid traffic goes live.

**Naming note:** This schema covers OrthoNow's existing 3-step site-wide booking form (location/specialty → contact details → confirm). The Task 2 landing page fires its own distinct event `consultation_form_submitted` - it is a separate 2-field campaign form and should not share an event name with the 3-step funnel or GA4 Funnel Exploration would conflate two different conversion paths.

**Key principle:** GTM only fires off what already exists in the DOM or the dataLayer. For static single-action interactions (clicks, page loads) a GTM trigger alone is enough. But for the 3-step booking form, GTM has no concept of "step 2 has been reached." The front-end must explicitly push a structured object to `window.dataLayer` at each step transition. This requires a front-end dev to instrument the form - it does not happen automatically by installing GTM.

---

## 1. Complete Event Schema

| Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
|---|---|---|---|
| `booking_step_complete` | Custom Event - GTM listens for `dataLayer.push` with `event: 'booking_step_complete'` - **requires front-end dataLayer push, not a native GTM trigger** | `step_number`, `step_name`, `clinic_location`, `specialty` | GA4 Funnel Exploration; feeds "started booking, did not complete" remarketing audience |
| `booking_form_submitted` | Custom Event - `dataLayer.push` on final step confirmation | `clinic_location`, `specialty`, `preferred_date`, `booking_id` | GA4 Conversion event; primary signal imported into Google Ads |
| `call_now_click` | GTM Click Trigger - Just Links, scoped to `tel:` href pattern | `call_location` (homepage / clinic_page / landing_page), `clinic_name`, `page_path` | GA4 Engagement → Events; secondary conversion event |
| `whatsapp_widget_click` | GTM Click Trigger - Click URL contains `wa.me` | `page_path`, `device_category`, `widget_position` | GA4 Engagement → Events; lower-funnel-weight conversion signal |
| `patient_guide_download` | Custom Event - fires after gated form submits successfully, not on PDF click | `guide_name`, `lead_source`, `page_path` | GA4 Conversion event; feeds "guide downloaders" nurture audience |
| `clinic_page_view` | GTM Page View trigger - URL path matches `/clinics/*` | `clinic_name`, `clinic_city`, `page_path` | GA4 Pages and Screens, segmented by `clinic_name` |
| `blog_scroll_depth` | GTM Scroll Depth trigger - 25/50/75/90% thresholds, scoped to blog pages | `scroll_percentage`, `article_title`, `page_path` | GA4 Engagement; feeds "highly engaged readers" remarketing audience |

---

## 2. Booking Form Funnel: Step-Level Drop-Off Tracking

### The core issue

GTM's native triggers have no visibility into multi-step form state. The only way GTM knows "step 2 has been reached" is if the front-end JavaScript pushes a dataLayer event at the moment each step completes. **This must be hand-coded into the form's front-end logic - it does not happen automatically.**

### Who writes the push

I write the dataLayer push syntax, but it must be inserted at the exact point in the form's step-transition logic. Brief to the dev: *fire on completion of step N, not on render of step N+1* - so drop-off math captures intent even if the user abandons before the next step finishes rendering.

### Step 1 - Location + specialty selected

GTM Trigger: Custom Event, Event name = `booking_step_complete`, condition `step_number equals 1`

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

### Step 2 - Contact details entered

GTM Trigger: Custom Event, Event name = `booking_step_complete`, condition `step_number equals 2`

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "preferred_date": "{{preferred date}}"
}
```

Note: `clinic_location` and `specialty` must be carried forward from step 1 in every subsequent push. The front-end must persist this in its own state object - not re-derived from GTM.

### Step 3 - Booking confirmed

GTM Trigger: Custom Event, Event name = `booking_step_complete`, condition `step_number equals 3`. Also separately fires `booking_form_submitted` as the clean conversion signal for Google Ads.

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "preferred_date": "{{preferred date}}"
}
```

### Surfacing drop-off in GA4 Funnel Exploration

1. Create a new Funnel Exploration in GA4
2. Define 3 open funnel steps each matching `event_name = booking_step_complete` with condition on custom parameter `step_number` (1, 2, 3)
3. Mark funnel as **open** (not closed) - paid campaigns could land users at any step
4. Enable **Show elapsed time** - distinguishes friction (high time on step) vs bounce (near-zero time)
5. Break down by `clinic_location` as secondary dimension - drop-off concentrated in one clinic = clinic-specific issue, not form UX
6. Register `step_number`, `clinic_location`, `specialty`, `preferred_date` as GA4 custom dimensions (event-scoped) in GA4 Admin before they are queryable in Funnel Exploration

---

## 3. Google Ads Conversion Import

**Conversion action to import: `booking_form_submitted`**

This is the only event representing a completed, qualified action - a patient who gave real contact details and confirmed a specific clinic and specialty.

`call_now_click` only proves someone tapped a phone number - not that a call connected or was a genuine lead. `patient_guide_download` is early-funnel research, not appointment-ready intent.

Google Ads Smart Bidding optimizes toward whatever conversion you import. Importing a noisy signal like `call_now_click` trains the algorithm to chase phone taps, not actual patients booked.

`booking_form_submitted` also carries `clinic_location` and `specialty` - so once imported, conversions can be segmented by which clinic or specialty converts best, directly useful for shifting budget toward highest-performing campaigns.

`call_now_click` and `patient_guide_download` are configured as **secondary/observed conversions** in Google Ads - visible for reporting but not driving bid optimization.
