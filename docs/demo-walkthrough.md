# Demo Walkthrough

This document explains how the **BrightSmile AI Receptionist** demo is intended to work.

The demo is designed to show the complete workflow from a patient voice request to backend scheduling, staff notification, waitlist handling, and calendar automation.

> Demo video coming soon.

---

## Demo Goal

The goal of the demo is to show that the AI receptionist is more than a simple chatbot.

It can:

- Handle voice appointment booking.
- Route structured tool calls through n8n.
- Use Supabase as the scheduling source of truth.
- Create Google Calendar events only for confirmed bookings.
- Send Slack alerts to staff.
- Add callers to a cancellation waitlist after explicit consent.
- Recommend waitlisted patients to staff after a cancellation.
- Use safe fallback statuses when staff verification is needed.

---

## Demo Architecture

The demo follows this flow:

```text
Caller
→ Vapi AI Receptionist
→ bookDentalAppointment tool
→ n8n webhook
→ Supabase RPC scheduling logic
→ Slack staff alert
→ Google Calendar event when confirmed
→ Response back to caller
```

Clinic FAQ questions follow a separate path:

```text
Caller
→ Vapi AI Receptionist
→ DefaultQueryTool / Knowledge Base
→ Answer clinic information
```

---

## Demo Scenario 1 — Successful Booking

### Caller Request

```text
I want to book a dental cleaning for Safiya Abdullahi on Tuesday July 14 at 11 AM. My phone number is +252611990088.
```

### Expected Tool Payload

The assistant should convert the clinic local time to UTC.

```json
{
  "action": "book",
  "patient_name": "Sophia Abdullahi",
  "patient_contact": "+252611990088",
  "service_code": "dental_cleaning",
  "requested_start_time": "2026-07-14T08:00:00+00:00",
  "urgency": "normal",
  "waitlist_consent": false
}
```

### Expected Backend Result

```text
status = confirmed
assigned_doctor = Dr. Yusuf Ali
```

### Expected Output

The caller hears that the dental cleaning is confirmed.

The system should also:

- Store the confirmed booking in Supabase.
- Send a Slack alert to staff.
- Create a Google Calendar event.
- Return the booking result to Vapi.

---

## Demo Scenario 2 — Slot Already Taken

After the first booking is confirmed, another caller requests the same slot.

### Caller Request

```text
I want to book a dental cleaning for Huda Ali on Tuesday July 14 at 11 AM. My phone number is +252611554433.
```

### Expected Backend Result

The system should not double-book the appointment.

Expected status:

```text
waitlist_offer
```

or another unavailable status depending on the backend response.

### Expected Output

The assistant should explain that the requested slot is not available and ask whether the caller wants to join the cancellation waitlist or choose another time.

---

## Demo Scenario 3 — Join Cancellation Waitlist

If the caller agrees to join the waitlist:

```text
Yes, put me on the cancellation waitlist. My email is huda@example.com.
```

### Expected Tool Payload

```json
{
  "action": "book",
  "patient_name": "Huda Ali",
  "patient_contact": "+252611554433",
  "service_code": "dental_cleaning",
  "requested_start_time": "2026-07-14T08:00:00+00:00",
  "urgency": "normal",
  "waitlist_consent": true,
  "notification_email": "huda@example.com"
}
```

### Expected Backend Result

```text
status = waitlisted
```

### Expected Output

The caller hears that they have been added to the cancellation waitlist.

The system should also:

- Store the waitlist entry in Supabase.
- Notify staff through Slack.
- Avoid automatically rebooking the patient.

---

## Demo Scenario 4 — Cancellation

The original confirmed appointment is cancelled.

### Same-Conversation Cancellation

If the caller cancels an appointment that was just booked in the same Vapi conversation:

```text
Actually, cancel that appointment.
```

The assistant should use the previous confirmed booking details and `booking_id` if available.

### Existing Appointment Cancellation

If the appointment existed before the current call, the caller can provide matching details:

```text
I want to cancel a dental cleaning for Safiya Abdullahi. The phone number is +252611990088. The appointment is Tuesday July 14 at 11 AM.
```

### Expected Tool Payload

```json
{
  "action": "cancel",
  "booking_id": "previous-confirmed-booking-id-if-available",
  "patient_name": "Sophia Abdullahi",
  "patient_contact": "+252611990088",
  "service_code": "dental_cleaning",
  "requested_start_time": "2026-07-14T08:00:00+00:00",
  "cancelled_by": "patient_voice_call",
  "cancellation_reason": "Patient requested cancellation"
}
```

### Expected Backend Result

Possible successful statuses:

```text
cancelled_no_waitlist_match
```

or:

```text
cancelled_waitlist_recommended
```

---

## Demo Scenario 5 — Cancellation Waitlist Recommendation

If a matching waitlisted patient exists, the cancellation workflow recommends that patient to staff.

Expected status:

```text
cancelled_waitlist_recommended
```

The workflow should return a staff recommendation, for example:

```json
{
  "recommended_waitlist_patient": {
    "staff_note": "Cancellation opened a matching slot. Staff should review and contact this waitlisted patient.",
    "patient_name": "Huda Ali",
    "priority_score": 90,
    "waitlist_reason": "exact_slot_taken"
  }
}
```

Important:

The system does **not** automatically rebook the waitlisted patient.

Staff must review the recommendation and contact the patient manually.

---

## Screenshots to Include

Recommended screenshots are stored in:

```text
assets/screenshots/
```

Suggested screenshot names:

```text
n8n-workflow.png
vapi-booking-transcript.png
slack-alert.png
supabase-booking-row.png
google-calendar-event.png
```

Optional screenshots:

```text
vapi-cancellation-transcript.png
vapi-waitlist-transcript.png
n8n-booking-execution.png
n8n-cancellation-execution.png
supabase-waitlist-row.png
```

---

## What the Demo Proves

The demo proves that:

- Vapi can collect caller details and call a structured tool.
- n8n can route the request correctly.
- Supabase can enforce scheduling rules.
- The system prevents double-booking.
- The system handles cancellation safely.
- The waitlist flow requires caller consent.
- Staff are notified when action is needed.
- Calendar events are created only for confirmed appointments.
- The AI does not fake booking or cancellation success.

---

## Known Prototype Boundaries

This is a prototype/demo project.

For real clinic deployment, additional work would be required around:

- Webhook authentication
- Secure production hosting
- Secrets management
- Audit logs
- Access control
- Data retention rules
- Monitoring and alerting
- Healthcare privacy and compliance review
- HIPAA or local healthcare compliance requirements where applicable

---

## Privacy Note

No real patient data should be used in the public demo, screenshots, or GitHub repository.

Use only demo names, fake phone numbers, and fake emails.

Do not expose:

- Supabase service role keys
- OAuth credentials
- Slack tokens
- Vapi private keys
- ngrok auth tokens
- Real patient names
- Real phone numbers
- Real appointment information
