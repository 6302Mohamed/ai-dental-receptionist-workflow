# Testing Guide

## Valid Booking Test

Voice phrase:

```text
I want to book a dental cleaning for Safiya Abdullahi on Tuesday July 14 at 11 AM. My phone number is +252611990088.
```

Expected tool payload:

```json
{
  "action": "book",
  "patient_name": "Safiya Abdullahi",
  "patient_contact": "+252611990088",
  "service_code": "dental_cleaning",
  "requested_start_time": "2026-07-14T08:00:00+00:00",
  "urgency": "normal",
  "waitlist_consent": false
}
```

Expected result:

```text
status=confirmed
assigned_doctor=Dr. Yusuf Ali
```

## Cancellation Test

Same conversation phrase:

```text
Actually, cancel that appointment.
```

Expected behavior:

- Vapi uses the previous confirmed booking details.
- Vapi includes `booking_id` if available.
- n8n calls cancellation RPC.
- Supabase marks the booking cancelled.
- Slack receives cancellation alert.

## Waitlist Test

Step 1: Book a slot.

```text
I want to book a dental cleaning for Aisha Roble on Tuesday July 14 at 11 AM. My phone number is +252611223344.
```

Step 2: Try the same slot with another patient.

```text
I want to book a dental cleaning for Huda Ali on Tuesday July 14 at 11 AM. My phone number is +252611554433.
```

Step 3: Agree to waitlist.

```text
Yes, put me on the cancellation waitlist. My email is huda@example.com.
```

Expected waitlist payload:

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

## Cancellation Waitlist Recommendation Test

Cancel the original booked appointment.

Expected result:

```text
status=cancelled_waitlist_recommended
```

The workflow should recommend the waitlisted patient to staff. It should not automatically rebook the waitlisted patient.

## Common Issues

### Wrong phone number from Vapi

If cancellation returns:

```text
cancel_not_found
```

check whether Vapi transcribed the phone number differently from the original booking.

Same-conversation cancellation should use `booking_id` from the previous confirmed booking.

### Wrong timezone

If the caller says 2:00 PM clinic time, the tool must send 11:00 AM UTC:

```text
2026-07-16T11:00:00+00:00
```

### Outside booking window

The system only accepts appointments inside the configured booking window.

### Supabase connection refused

If Docker/n8n returns `ECONNREFUSED`, retry logic should make the call again before returning fallback status.
