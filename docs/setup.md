# Setup Guide

This guide explains how to set up the **BrightSmile AI Receptionist** project locally.

The project connects:

```text
Vapi Voice Assistant
→ ngrok public webhook
→ n8n workflow
→ Supabase scheduling backend
→ Slack staff alerts
→ Google Calendar events
```

The setup below is written for a prototype/demo environment. Do not use real patient data or production healthcare data without a proper privacy, security, and compliance review.

---

## 1. Prerequisites

You need accounts or local access for:

- Vapi
- n8n
- Supabase
- Slack
- Google Calendar
- ngrok
- Postman

Recommended local tools:

- Docker Desktop
- A code editor such as VS Code
- A browser
- Postman desktop app

---

## 2. Project Folder Structure

Recommended repository structure:

```text
brightsmile-ai-receptionist/
├── README.md
├── .env.example
├── .gitignore
├── docs/
│   ├── architecture.md
│   ├── setup.md
│   ├── testing.md
│   ├── demo-script.md
│   └── vapi-prompt.md
├── workflows/
│   └── n8n/
│       └── Dental_AI_Receptionist_Booking_Core_SANITIZED.json
└── assets/
    ├── diagrams/
    │   └── brightsmile-system-flow.png
    └── screenshots/
        ├── n8n-workflow.png
        ├── vapi-booking-transcript.png
        ├── slack-alert.png
        ├── supabase-booking-row.png
        └── google-calendar-event.png
```

---

## 3. Environment and Secret Handling

Create a local `.env` file from `.env.example` if needed.

Example:

```env
SUPABASE_URL=https://your-project-ref.supabase.co
SUPABASE_SERVICE_ROLE_KEY=replace-with-your-service-role-key

N8N_WEBHOOK_URL=https://your-ngrok-domain/webhook/dental-booking

SLACK_CHANNEL_ID=replace-with-your-channel-id
GOOGLE_CALENDAR_ID=primary
```

Do **not** commit real secrets.

Do not commit:

- Supabase service role key
- Vapi private keys
- Slack tokens
- Google OAuth credentials
- ngrok auth tokens
- Real patient information
- Screenshots showing secrets

---

## 4. Supabase Setup

Supabase is the source of truth for scheduling.

It stores:

- Services
- Doctors
- Doctor availability
- Bookings
- Waitlist entries
- Workflow logs

### Required Tables

The database should include these tables:

```text
clinic_services
doctors
doctor_availability
bookings
waitlist
workflow_logs
```

### Required RPC Functions

The workflow depends on these Supabase RPC functions:

```text
try_book_slot
cancel_booking_and_suggest_waitlist
```

---

## 5. Booking RPC Responsibility

The `try_book_slot` function should handle:

- Required input validation
- Service validation
- Doctor availability checks
- Clinic schedule checks
- Booking window checks
- Overlap protection
- Daily capacity checks
- Confirmed booking creation
- Waitlist offer logic
- Waitlist insertion after consent
- Safe response statuses

Expected booking statuses:

```text
confirmed
unavailable
waitlist_offer
waitlisted
human_required
invalid_input
```

---

## 6. Cancellation RPC Responsibility

The `cancel_booking_and_suggest_waitlist` function should handle:

- Cancellation by `booking_id` when available
- Cancellation by patient details when booking ID is missing
- Matching only confirmed bookings
- Already-cancelled detection
- Updating booking status to cancelled
- Looking for matching waitlisted patients
- Returning a staff recommendation when a waitlisted patient matches
- Safe response statuses

Expected cancellation statuses:

```text
cancelled_no_waitlist_match
cancelled_waitlist_recommended
cancel_not_found
already_cancelled
verification_required
```

---

## 7. n8n Local Setup

You can run n8n locally using Docker.

Example:

```powershell
docker ps
```

If your n8n container is running, you should see something like:

```text
n8n-local   docker.n8n.io/n8nio/n8n:stable   Up ...
```

To restart the container:

```powershell
docker restart n8n-local
```

Your container name may be different. Check it with:

```powershell
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
```

---

## 8. Import the n8n Workflow

Import the sanitized workflow from:

```text
workflows/n8n/Dental_AI_Receptionist_Booking_Core_SANITIZED.json
```

After importing, reconnect or update:

- Supabase project URL
- Supabase service role key
- Slack credential
- Slack channel
- Google Calendar credential
- Google Calendar ID
- Webhook URL

Do not publish your real working n8n export if it contains keys or credentials.

---

## 9. n8n Workflow Structure

Main flow:

```text
Webhook
→ Normalize Vapi or Postman Input
→ Is Cancellation Action?
```

Booking branch:

```text
Call Supabase try_book_slot
→ Normalize Booking Result
→ Slack Booking Alert
→ Is Confirmed Booking?
   → If confirmed: Create Google Calendar Event
   → If not confirmed: Respond to Webhook
→ Respond to Webhook
```

Cancellation branch:

```text
Cancel booking and suggest waitlist
→ Normalize Cancellation Result
→ Slack Cancellation Alert
→ Respond to Webhook
```

---

## 10. Supabase Code Nodes in n8n

The Supabase RPC calls are handled through n8n Code nodes.

This was done because local Docker-to-Supabase HTTPS calls occasionally returned errors such as:

```text
ECONNREFUSED
```

The Code nodes include retry logic.

The retry logic prevents one temporary network refusal from breaking the demo.

If all retries fail:

- Booking returns a safe `human_required` result.
- Cancellation returns a safe `verification_required` result.

This prevents the assistant from falsely saying an appointment was booked or cancelled.

---

## 11. ngrok Setup

The local n8n webhook must be reachable by Vapi.

Start ngrok and point it to local n8n:

```powershell
ngrok http 5678
```

Use the HTTPS forwarding URL in Vapi.

Example Vapi server URL:

```text
https://your-ngrok-domain.ngrok-free.dev/webhook/dental-booking
```

Use the **production webhook** path:

```text
/webhook/dental-booking
```

not the test path:

```text
/webhook-test/dental-booking
```

Make sure the n8n workflow is active.

---

## 12. Vapi Setup

Create a Vapi assistant for the dental receptionist.

The assistant should have:

1. A knowledge base tool for clinic information.
2. A custom tool named:

```text
bookDentalAppointment
```

The custom tool should point to the n8n production webhook.

Recommended tool settings:

```text
Timeout: 60 seconds
Async: Off
Strict mode: Off
Authorization: empty unless webhook auth is added
```

---

## 13. Vapi Tool Schema

The most important rule is that `requested_start_time` must be sent in UTC `+00:00`.

Example:

```json
{
  "requested_start_time": {
    "description": "Requested appointment start time converted to UTC in ISO 8601 format with +00:00 timezone. Example: if the caller says July 16, 2026 at 2:00 PM clinic time, send 2026-07-16T11:00:00+00:00.",
    "type": "string"
  }
}
```

The schema should also include optional `booking_id`.

This makes same-conversation cancellation safer.

Example:

```json
{
  "booking_id": {
    "description": "Optional booking ID for cancellation when the appointment was just confirmed earlier in the same conversation or is otherwise known.",
    "type": "string"
  }
}
```

---

## 14. Timezone Setup

The clinic speaks in local time:

```text
Africa/Nairobi / UTC+03:00
```

The backend receives UTC:

```text
+00:00
```

Example:

```text
Caller says: Tuesday July 14 at 11:00 AM
Tool sends: 2026-07-14T08:00:00+00:00
```

Do not send local `+03:00` timestamps to Supabase.

---

## 15. Slack Setup

Create or choose a Slack channel for staff alerts.

Example:

```text
#clinic-alerts
```

Connect Slack credentials in n8n.

Slack alerts should include:

- Booking status
- Patient demo name
- Contact
- Service
- Requested time
- Assigned doctor if available
- Waitlist reason if available
- Staff action if needed

For public screenshots, blur or replace anything sensitive.

---

## 16. Google Calendar Setup

Create a demo calendar such as:

```text
BrightSmile Demo Calendar
```

Connect Google Calendar credentials in n8n.

The workflow should create calendar events only when:

```text
status = confirmed
```

Do not create calendar events for:

- unavailable
- waitlist_offer
- waitlisted
- human_required
- invalid_input

---

## 17. Postman Testing

Before testing Vapi, test the n8n webhook with Postman.

Production webhook format:

```text
POST https://your-ngrok-domain.ngrok-free.dev/webhook/dental-booking
```

Example booking body:

```json
{
  "action": "book",
  "patient_name": "Demo Patient",
  "patient_contact": "+252600000000",
  "service_code": "dental_cleaning",
  "requested_start_time": "2026-07-14T08:00:00+00:00",
  "urgency": "normal",
  "waitlist_consent": false,
  "notification_email": null
}
```

Example cancellation body:

```json
{
  "action": "cancel",
  "booking_id": "example-booking-id",
  "patient_name": "Demo Patient",
  "patient_contact": "+252600000000",
  "service_code": "dental_cleaning",
  "requested_start_time": "2026-07-14T08:00:00+00:00",
  "cancelled_by": "postman_test",
  "cancellation_reason": "Patient requested cancellation"
}
```

---

## 18. Vapi Testing

After Postman works, test Vapi by voice.

Valid booking phrase:

```text
I want to book a dental cleaning for Safiya Abdullahi on Tuesday July 14 at 11 AM. My phone number is +252611990088.
```

Expected tool time:

```text
2026-07-14T08:00:00+00:00
```

Same-conversation cancellation phrase:

```text
Actually, cancel that appointment.
```

The assistant should use the previous `booking_id` if available.

---

## 19. Recommended Testing Order

Use this order to avoid debugging too many layers at once:

```text
1. Supabase RPC directly
2. Postman → n8n webhook
3. Booking branch
4. Cancellation branch
5. Slack alert
6. Google Calendar event
7. Vapi booking
8. Vapi cancellation
9. Waitlist flow
10. Cancellation waitlist recommendation
```

---

## 20. Common Setup Issues

### Vapi sends the wrong timezone

Fix the tool schema and prompt. The tool must send UTC `+00:00`.

### Cancellation returns `cancel_not_found`

Check:

- Phone number transcription
- Patient name spelling
- Service code
- Requested start time
- Whether `booking_id` was included

Same-conversation cancellation should use `booking_id`.

### Postman works but Vapi fails

Check:

- Vapi tool schema
- Server URL
- Timeout
- Prompt instructions
- Required fields
- Whether the workflow is active

### n8n execution succeeds but Postman gets no response

Check that each branch reaches a `Respond to Webhook` node.

### Supabase connection fails randomly

Use retry logic in the Supabase Code nodes.

---

## 21. Public GitHub Safety Checklist

Before publishing:

- [ ] Remove real Supabase service role key.
- [ ] Remove real OAuth credentials.
- [ ] Remove Slack credential IDs if needed.
- [ ] Remove real calendar IDs if needed.
- [ ] Replace real ngrok URL with placeholder if preferred.
- [ ] Use sanitized n8n workflow JSON.
- [ ] Use demo patient data only.
- [ ] Blur sensitive screenshots.
- [ ] Do not publish real patient information.
- [ ] Do not publish production-ready healthcare claims.

---

## 22. Production Notes

This prototype is not production-ready for a real healthcare environment without further work.

A production version should add:

- Stable hosted n8n or backend service
- Proper webhook authentication
- Environment-based secrets management
- Audit logs
- Role-based staff access
- Data retention rules
- Privacy review
- Vendor compliance review
- HIPAA or local healthcare compliance review where applicable
- Monitoring and alerting
- Backup and recovery process

---

## Final Setup Result

When setup is complete, the system should support:

- Voice booking
- Voice cancellation
- Waitlist consent
- Cancellation waitlist recommendation
- Slack alerts
- Google Calendar creation for confirmed appointments
- Safe fallback handling
- Staff review for uncertain cases
