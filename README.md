# namoza-orthonow-assignment

Developer Assignment — Position 1 (Client Web + Martech)
Candidate: Krishna Kumar Singh

---

## Structure

### task1-gtm-schema/
Full GTM event schema for OrthoNow covering all key interactions (booking form, call buttons, WhatsApp widget, PDF download, clinic pages, blog scroll depth). Includes dataLayer JSON for each step of the 3-step booking funnel and justification for the Google Ads conversion import choice.

### task2-landing-page/
Single-file HTML/CSS/JS landing page for the "Book a Consultation" campaign. Zero external dependencies — no frameworks, no web fonts, no external images. Built for 90+ mobile PageSpeed score. dataLayer push fires on form submit only, not on page load.

### task3-integration/
Written architecture (300-400 words) for the end-to-end integration: landing page form → HubSpot CRM → WhatsApp via Karix → Google Ads offline conversion. Covers middleware approach, failure fallback with durable queue, WhatsApp SLA monitoring, and HubSpot phone-based deduplication (HubSpot dedupes on email by default — this form has no email field).

---

## Live page
https://namoza-orthonow-assignment.vercel.app/

## PageSpeed screenshot
See task2-landing-page/ folder for PageSpeed screenshots (100 Performance, 89 Accessibility, 100 Best Practices, 100 SEO on Mobile).
