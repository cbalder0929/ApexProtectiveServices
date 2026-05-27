# Planned Changes — Apex Protective Services

This doc describes converting the existing "Request a Quote" form in `index.html` into a **call-booking widget** that:

1. Lets visitors pick a date + time for a phone consultation.
2. Creates an event on the company Google Calendar.
3. Sends a notification (email + optional SMS/push) to the owner whenever a new booking comes in.

---

## 1. Current state

The form lives in `index.html` around lines **1397–1433**. It collects:

- Name, phone, email
- Venue / business type (dropdown)
- Free-text "How can we help?" textarea
- Submits via a client-side `onsubmit` that only shows an `alert()` — no data is sent anywhere.

The site is fully static (no backend, no build step). It is served as plain HTML + CSS + a small inline `<script>`.

---

## 2. Target user flow

> Visitor clicks "Book a Call" → picks a 30-min slot from a calendar → enters name/phone/email/venue type → confirms → gets an email confirmation → company Google Calendar gets a new event → owner gets a phone notification.

---

## 3. Recommended approach — embed a booking service (fastest, no backend)

Because the site is static, the lowest-friction path is to use a hosted booking tool that already handles Google Calendar sync and owner notifications. Two solid options:

| Tool | Why pick it | Cost |
| --- | --- | --- |
| **Cal.com** | Open-source, white-label-friendly, free tier, native Google Calendar 2-way sync, webhooks for custom notifications | Free tier works |
| **Calendly** | Most polished UX, easiest setup, Google Calendar sync out of the box, mobile push notifications via the app | Free tier works, paid for branding removal |

**Recommendation: Cal.com** — better look-and-feel control to match the gold/black brand, and webhooks let you wire in custom notifications later.

### Steps to integrate Cal.com

1. **Create the account**
   - Sign up at https://cal.com with `info@apexpcom.com` (or whichever address should own the calendar).
   - In Cal.com → **Settings → Apps → Calendar** → connect the company Google account. This is the calendar bookings will land on.
   - Create an event type: **"Free Security Consultation — 30 min"**. Set business hours, buffer time, and the questions to ask the booker.

2. **Add the booker questions** (replace the current form fields)
   - Name (built-in)
   - Email (built-in)
   - Phone — add as a required custom field, type `phone`
   - Venue / Business Type — add as a required custom field, type `select` with options:
     `Bar / Nightclub`, `Restaurant`, `Private Event`, `Corporate / Office`, `Residential`, `Other`
   - Notes — optional textarea, "Anything specific we should know before the call?"

3. **Embed it on the site**
   - In Cal.com → **Event type → Embed → Inline embed**. Copy the snippet (looks like a `<script>` + `<div id="my-cal-inline">`).
   - In `index.html`, **replace lines 1397–1433** (the entire `<div class="contact-form reveal">…</div>`) with the Cal.com embed wrapped in the same `.contact-form` styling so it visually matches the rest of the section.
   - Update the heading from `Tell us about your needs.` to **`Book a consultation call.`** and the eyebrow from `REQUEST A QUOTE` to **`SCHEDULE A CALL`**.
   - Update the surrounding copy:
     - Section title (line 1363): `Let's talk <em>protection.</em>` → keep, still works.
     - Subhead (line 1364): `Tell us about your venue or event. We'll respond within one business day…` → change to **`Pick a time that works. We'll call you then — no forms, no waiting.`**
   - Update the CTA buttons that point to the form:
     - Nav CTA (line 1020): `Get a Quote` → **`Book a Call`**
     - Hero secondary CTA (line 1097): `Request Quote` → **`Book a Call`**
     - CTA banner button (line 1352): `Get Your Free Quote` → **`Book Your Free Call`**
     - CTA banner subhead (line 1350): `Talk to us about a custom security plan — quotes in 24 hours.` → **`Book a free 30-minute consultation. We'll call you at the time you pick.`**

4. **Style overrides** (so the Cal embed matches the gold/black theme)
   - In Cal.com → **Event type → Appearance**, set:
     - Brand color: `#d4af37`
     - Dark mode: on
   - If the embed still looks off, target it with custom CSS in `index.html`:
     ```css
     #my-cal-inline { background: var(--bg-2); border: 1px solid var(--rule); padding: 24px; }
     ```

5. **Connect Google Calendar (the key piece)**
   - In Cal.com → **Apps → Google Calendar → Install → Authorize** with the company Google account.
   - In the event type's **Advanced** tab, set "Add to calendar" → the company calendar.
   - From now on, every booking auto-creates an event on that Google Calendar with the booker's name, phone, email, and answers in the event description.

6. **Set up owner notifications**

   Cal.com sends email confirmations by default. To get a real-time notification on a phone:

   - **Email → push:** Install the **Gmail app** on the owner's phone with notifications enabled for the address that receives Cal.com booking emails. Easiest path, zero extra setup.
   - **SMS:** In Cal.com → **Workflows → New workflow → Trigger: "New booking" → Action: "Send SMS to host"**. Add the owner's phone number. Free tier includes a small monthly SMS quota; paid tier removes the limit.
   - **Phone push via Cal.com mobile app:** Install the Cal.com app on the owner's phone and sign in. New bookings push instantly.
   - **(Optional, advanced) Custom webhook:** Cal.com → **Webhooks → Add → URL** pointing to a small handler (e.g., Zapier, Make.com, or a Cloud Function) that fans out notifications anywhere — Slack, Discord, Twilio SMS, custom push, etc.

---

## 4. Alternative — build it ourselves (more control, more work)

Only do this if Cal.com / Calendly don't fit. Sketch:

- Add a date/time picker UI to the form (e.g., `flatpickr` for the calendar, a `<select>` for time slots).
- Stand up a tiny backend endpoint (Cloudflare Worker, Vercel function, or Google Apps Script) that:
  1. Receives the booking POST.
  2. Calls the **Google Calendar API** (`events.insert`) using a service account with domain-wide delegation, or an OAuth2 refresh token tied to the company calendar.
  3. Sends an email via SendGrid/Resend and/or an SMS via Twilio to the owner.
  4. Returns success → site shows a confirmation message.
- Pros: full visual control, no third-party branding.
- Cons: have to handle availability conflicts, timezone math, double-booking, spam, and ongoing maintenance. Not worth it for v1.

---

## 5. Cleanup after the swap

- Delete the inline `onsubmit="event.preventDefault(); alert(...)"` handler (line 1400) — no longer needed.
- Remove the `.form-row`, `.form-field`, `.form-submit`, `.form-note` CSS rules **only if** nothing else uses them (lines ~765–829). Safe to leave them; they're small.
- Keep the **Phone / Email / Service Area** contact-method block (lines 1366–1394) as-is — it's still useful for visitors who'd rather call directly than book.
- Sanity-check the page on mobile — the Cal.com embed should reflow at the 1000px and 480px breakpoints already in the stylesheet.

---

## 6. Acceptance checklist

- [ ] Booking widget visible in the "Get in touch" section, styled to match the brand.
- [ ] A test booking creates an event on the company Google Calendar with all the booker's details in the description.
- [ ] Owner receives a notification (email + SMS or push) within ~1 minute of a test booking.
- [ ] All CTA buttons across the page say "Book a Call" / "Book Your Free Call" instead of "Quote" wording.
- [ ] No console errors; page still passes basic mobile responsiveness check.
