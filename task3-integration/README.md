# Task 03 - Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

## End-to-end architecture

I would use a direct server-side API call from a lightweight middleware function - not a native HubSpot embed and not Zapier or Make. Concretely: the landing page form posts to a small serverless endpoint (Vercel or Cloudflare Function) I control, which then makes the HubSpot Contacts API call, triggers the WhatsApp send via Karix API, and confirms the Ads conversion - all from one place.

Why not native HubSpot embed: an embedded HubSpot form hands control of submission to HubSpot's own JS, which makes it harder to guarantee the dataLayer push fires at the right moment (the brief requires this fires on submit not on page load) and harder to chain the WhatsApp send in sequence.

Why not Zapier or Make: both add 30 seconds to several minutes of trigger latency depending on plan tier, which directly threatens the WhatsApp 2-minute SLA. They are also a black box for debugging - when the SLA is missed I want server logs I can grep, not a third-party run history UI.

**Sequence:**
1. User submits form → client-side validation passes → dataLayer push fires immediately (not blocked by any backend call, so never delayed by network issues downstream)
2. Form payload POSTs to middleware endpoint
3. Middleware calls **HubSpot Contacts API** (`PATCH /crm/v3/objects/contacts` using phone as idempotency key - detailed below) to create or update the contact with Name, Phone, Clinic Preference, Source = 'Google Ads - Consultation Landing Page', Lead Status = 'New Enquiry'
4. On successful HubSpot write, middleware calls **Karix WhatsApp Business API** to send confirmation message - sequential not parallel, since the WhatsApp message should only fire after the lead is confirmed saved
5. Middleware also fires an **offline conversion import via Google Ads API** once HubSpot confirms the write - conversion tied to "lead actually saved" not just "form submitted"

## The single biggest failure point and the fallback

**Biggest failure point:** the HubSpot API call failing or timing out after the dataLayer/Ads conversion has already fired client-side. The conversion event is decoupled from the CRM write by design (so the user sees a fast thank-you state), but this means it is possible to count a conversion for a lead that never actually landed in HubSpot.

**Fallback:** the middleware writes every incoming submission to a durable queue (a Postgres/Supabase table) *before* attempting the HubSpot call, with a status column (`pending` / `synced` / `failed`). If the HubSpot call fails the row stays `failed` and a retry job (cron every 5 minutes) reattempts the sync. No lead is ever silently lost even if HubSpot is down or rate-limiting - the source of truth is the queue table and HubSpot becomes an eventually-consistent mirror of it.

## Protecting the WhatsApp 2-minute SLA

**What could break it:** Karix API latency or downtime, slow HubSpot call (since WhatsApp send happens after it in the sequence), or middleware function cold-starting on a serverless platform under low traffic.

**Monitoring:** log a timestamp at form-submit-received and at WhatsApp-send-confirmed for every row in the queue table and alert via Slack webhook any time the gap exceeds 90 seconds - giving a buffer before the hard 2-minute SLA is actually breached. If Karix proves more reliable than HubSpot in practice, the WhatsApp send can be decoupled to fire as soon as the submission is queued rather than waiting for the full HubSpot write to confirm.

## The HubSpot phone deduplication issue

HubSpot's native contact deduplication is **email-based by default**. This form collects no email. So two patients submitting with the same 10-digit phone number but different names would on a default setup create two separate contact records - not update one.

Fix: before calling the HubSpot create endpoint, the middleware first does a search/lookup call (`POST /crm/v3/objects/contacts/search`, filtered on a custom `phone_number` property - not HubSpot's default `phone` field which is not a unique/searchable key by default). If a contact with that phone number exists, I `PATCH` that existing contact ID instead of creating a new one.

Edge case: two patients genuinely sharing a phone number (common in Indian households - shared family phone). The second submission updates Clinic Preference and Lead Status but appends the second name as a note/activity on the contact rather than silently overwriting it - so the clinic does not miss a patient because the system assumed one phone number means one person.
