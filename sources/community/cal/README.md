# Cal.com

**Version:** 0.1.0
**Backend:** HTTP
**Tables:** 4
**Base URL:** `https://api.cal.com` (override with `CAL_BASE_URL`)

Query bookings, event types, schedules, and profile data from Cal.com
(Cloud or self-hosted).

## Authentication

Requires a `CAL_API_KEY`. Find it in **Settings → Security → API Keys**.

- Test mode keys have the prefix `cal_`
- Live mode keys have the prefix `cal_live_`

```bash
CAL_API_KEY=cal_live_... coral source add --file sources/community/cal/manifest.yaml
```

Or interactively:

```bash
CAL_API_KEY=cal_live_... coral source add --file sources/community/cal/manifest.yaml --interactive
```

### Self-hosted

Set `CAL_BASE_URL` to your instance URL:

```bash
CAL_API_KEY=cal_... CAL_BASE_URL=https://cal.example.com \
  coral source add --file sources/community/cal/manifest.yaml
```

## Tables

| Table | Description | Optional filters |
|---|---|---|
| `me` | Profile of the authenticated user | — |
| `event_types` | Bookable meeting configurations | — |
| `bookings` | Scheduled meetings and their status | `status`, `attendee_email`, `attendee_name`, `event_type_id` |
| `schedules` | Availability schedules | — |

## Quick start

```bash
# Confirm connectivity and see your profile
coral sql "SELECT id, username, email, name, time_zone FROM cal.me"

# List all event types
coral sql "
  SELECT id, title, slug, length_in_minutes, hidden, booking_url
  FROM cal.event_types
  ORDER BY length_in_minutes
"

# All bookings
coral sql "
  SELECT id, uid, title, status, start, duration, event_type_id
  FROM cal.bookings
  ORDER BY start DESC
  LIMIT 20
"

# Filter bookings by status
coral sql "
  SELECT uid, title, start, host__name, host__email
  FROM cal.bookings
  WHERE status = 'cancelled'
  ORDER BY start DESC
  LIMIT 20
"

# Bookings for a specific event type
coral sql "
  SELECT uid, title, start, status
  FROM cal.bookings
  WHERE event_type_id = <your-event-type-id>
  ORDER BY start DESC
"

# Booking volume by event type
coral sql "
  SELECT event_type__slug, status, COUNT(*) as count
  FROM cal.bookings
  GROUP BY event_type__slug, status
  ORDER BY count DESC
"

# Availability schedules
coral sql "
  SELECT id, name, time_zone, is_default
  FROM cal.schedules
"
```

## Discovery order

```text
me
  → default_schedule_id → schedules.id

event_types
  → id (event_type_id)
    → bookings (WHERE event_type_id = '...')
  → schedule_id → schedules.id

bookings
  → event_type_id → event_types.id
```
