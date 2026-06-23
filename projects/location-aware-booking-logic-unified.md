---
title: Location-Aware Booking Logic (Unified Specification)
created_at: '2026-06-23T00:00:00Z'
tags:
- scheduling
- location-aware
- booking
- algorithm
- zip-code
- client-flow

## Location-Aware Booking Logic (Unified Specification)

### Purpose of this document

This is the **single, complete description** of how a client's location determines
which appointment slots they can see and book.

Single source of truth for how client location is captured, validated, and
  used to calculate appointment slot visibility, travel-time blocking, and conflict
  detection. 

Read top to bottom and you have the whole system: how an agent becomes bookable in a
city, how a client's location is captured and progressively refined, and how that
location drives every slot-visibility and conflict decision.

---

## 1. Glossary

Use these exact terms everywhere — in code, in API payloads, and in conversation —
so a person and an LLM reading this doc mean the same thing every time.

| Term | Meaning |
|---|---|
| `client_location` | A single `{latitude, longitude}` pair representing where the agent will physically go. This is the **only** location value the travel-time algorithm (Section 4) ever consumes. It has exactly one value at a time, but that value gets progressively more precise as the client moves through the booking flow (Section 3). |
| `zip_centroid` | The approximate `{latitude, longitude}` of a zip/postal code's geographic center. Used as a **temporary stand-in** for `client_location` before a street address exists. |
| `street_address_location` | The precise `{latitude, longitude}` geocoded from the client's full street address. Once this exists, it **replaces** the zip centroid as `client_location` and is never overwritten by anything less precise (see Section 3.5). |
| `serviceable_cities` | The list of city names an agent has chosen to work in. Configured by the agent, not the client. |
| `serviceable_zip_codes` | The full set of zip/postal codes that fall inside an agent's `serviceable_cities`. Derived automatically by the system — the agent never enters zip codes directly. |
| `blocking_window` | The time range after an existing appointment during which new slots are hidden from the client, because the agent can't physically arrive in time. Calculated by the travel-time algorithm. |
| `buffer` | A fixed 5-minute tolerance added to every blocking-window calculation in the agent's favor (Section 4.2). |

---

## 2. System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│  AGENT SETUP (once, or whenever agent updates their profile)         │
│  Agent picks serviceable_cities → system derives serviceable_zip_codes│
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│  CLIENT BOOKING FLOW (every time a client books)                     │
│                                                                        │
│  Step 1: Zip code entered → validated → geocoded to zip_centroid     │
│             client_location := zip_centroid   (coarse, temporary)    │
│                                                                        │
│  Step 2: Calendar + slot visibility, calculated using client_location│
│             (this is the travel-time algorithm, Section 4)           │
│                                                                        │
│  Step 3: Slot selected                                                │
│                                                                        │
│  Step 4: Full street address entered → geocoded to                   │
│          street_address_location                                     │
│             client_location := street_address_location (precise,    │
│             permanent for this appointment)                          │
│                                                                        │
│  Step 5: Phone number entered → appointment booked with the final    │
│          client_location → status "booked" (awaiting agent approval) │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│  AGENT APPROVAL                                                       │
│  Agent reviews; if a future conflict was flagged (Section 5), agent  │
│  acknowledges or cancels; on approval, client gets an SMS             │
└──────────────────────────────────────────────────────────────────────┘
```

**The one rule that ties everything together:** `client_location` is a single
value that starts coarse (a zip centroid) and gets replaced — never merged,
never averaged — by a precise value (a street address) as soon as one exists.
Every calculation always uses whatever `client_location` currently is; it never
needs to know whether that value came from a zip code or an address.

---

## 3. Client Booking Flow (Detailed)

### 3.1 Agent setup — serviceable cities (prerequisite, happens before any client books)

- Agent selects **city names** they're willing to serve (e.g. "Toronto",
  "Mississauga"). Agent does **not** enter zip codes directly.
- The system looks up every zip code belonging to each selected city from a
  central reference table (Section 7.1) and stores the result as
  `serviceable_zip_codes` for that agent.
- This list is recalculated any time the agent changes their serviceable
  cities.

### 3.2 Step 1 — Client enters zip code

- Client opens the agent's shareable link and is prompted for their zip/postal
  code (no login required).
- System normalizes the input (strip spaces, uppercase) and looks up its city
  in the reference table.

| Lookup result | System response |
|---|---|
| Zip code not found in reference table | Error: "Invalid zip code. Please try again." Client re-enters. |
| Zip code's city is **not** in agent's `serviceable_cities` | Error: "We don't serve that area yet." Client re-enters or leaves. |
| Zip code's city **is** in agent's `serviceable_cities` | Proceed. Geocode the zip code to its `zip_centroid`. Set `client_location := zip_centroid`. |

### 3.3 Step 2 — Calendar and slot selection

- Client picks a date.
- System calls the travel-time algorithm (Section 4) using the current
  `client_location` (the zip centroid at this point) to compute which slots
  are visible.
- Client sees only available slots — no blocked-reason detail is shown to the
  client (that detail exists only for agent-facing views).
- Client selects a slot.

### 3.4 Step 3 — Details form

Client provides, in one form:
- Name (required)
- Street address (required)
- City (pre-filled from the zip code, editable)
- Breed of dog (required)
- Disclaimer acknowledgment (required checkbox)

Two actions:
- **Cancel** → returns to Step 2 (the calendar), keeping the zip code already
  entered.
- **Request Appointment** → proceeds to Step 3.5.

### 3.5 Step 3.5 — Address geocoding and the location handoff

This is the moment `client_location` is finalized.

```
GEOCODE the full street address (street_address + city + postal_code)
  → street_address_location = {latitude, longitude}

IF geocoding fails (address not found):
  Error: "We couldn't find that address. Please check the spelling."
  Client corrects and retries.

IF the geocoded city differs from the city implied by the Step 1 zip code:
  Warn: "Your address is in {geocoded_city}, which is different from the
         area you searched ({zip_code_city}). Availability may change —
         we'll recalculate your available times."
  → Recalculate Step 2's available slots using street_address_location
    (see Section 3.6 — this is the resolved version of the old "Case 5" gap)
  Client reviews the (possibly updated) slot and confirms, or picks a new one.

client_location := street_address_location
  (This permanently replaces the zip centroid for this appointment. Nothing
  downstream — booking, conflict detection, agent dashboards — ever sees the
  zip centroid again.)
```

### 3.6 Resolved decision: what happens when the address is in a different city than the zip code

Two earlier drafts of this logic left this open as "Option A vs Option B."
**This document adopts Option A — recalculate — as the standard behavior:**

> If the street address geocodes to a different city than the zip code the
> client originally searched with, the system **must** recalculate slot
> availability using the new, precise location before letting the client
> confirm. The client cannot book a slot that was only validated against the
> wrong location.

Rationale: the zip centroid was always a stand-in for the real address. If the
real address turns out to be materially different, the slot visibility shown
in Step 2 is stale and may be wrong (different travel time, possibly even a
different blocking window). Recalculating is the only way to guarantee the
booked slot is actually achievable for the agent. If this is rejected in favor
of the simpler MVP option (allow booking, flag for agent review instead), that
must be a deliberate product decision recorded here — not a silent default.

### 3.7 Step 4 — Phone number and submission

- Client enters phone number.
- Client clicks **Request Appointment**.
- System creates the appointment via `POST /appointments` using the final
  `client_location` (the street address location), with `status: "booked"`.
- Client sees: "Processing — we'll text you once [Agent] confirms."
- **No confirmation SMS is sent at this point.** SMS only goes out after
  agent approval.

### 3.8 Device GPS — explicitly out of scope for location calculations

If the client's device GPS is available at any point, it is **never** used
for `client_location` or any travel-time calculation. Its only permitted use
is map display or routing convenience after an appointment is already
approved. This avoids a client's GPS (which may not match where the dog
actually is — e.g. booking from work) silently overriding a deliberately
entered address.

---

## 4. The Travel-Time Algorithm (Slot Visibility)

This section is location-source-agnostic: it doesn't care whether
`client_location` came from a zip centroid or a street address. It only cares
about the value as it currently stands.

### 4.1 First appointment of the day

- Earliest visible slot = agent's configured start time.
- No location-based blocking applies — there's no previous appointment to
  travel from.
- Example: agent starts at 8:00 AM → the 8:00–8:30 slot is immediately
  available.

### 4.2 Subsequent appointments — blocking window calculation

For any slot after an existing appointment:

```
1. Find the most recent existing appointment before the slot being requested.

2. Calculate:
     Last_Appointment_End = end time of that appointment
     Travel_Time = GoogleMaps(
                     origin: last_appointment.client_location,
                     destination: requested_slot.client_location,
                     departure_time: Last_Appointment_End)
     Calculated_Arrival = Last_Appointment_End + Travel_Time
     Blocking_Until = Calculated_Arrival - 5_minute_buffer

3. Slots from Blocking_Until onward are shown; everything before is hidden.
```

**Worked example:**
- Last appointment ends 8:30. Travel time to new location = 35 min.
- Calculated arrival = 8:30 + 35 = 9:05.
- Blocking_Until = 9:05 − 5 min buffer = 9:00.
- Result: slots from 9:00 onward are shown; 8:00–8:30 and 8:30–9:00 stay
  hidden.

**The buffer is a tolerance, not a deduction.** It is accepted lateness baked
into what's shown to the client — it never makes slots appear *later* than the
real calculated arrival would allow; it only ever pulls the cutoff slightly
*earlier* so the agent has 5 minutes of slack.

Each time a new appointment is added, this calculation re-runs from that new
appointment as the "most recent" one for any later slot request.

### 4.3 Same-location rule

If the new appointment's `client_location` is the same as the previous
appointment's:

- Travel time is still calculated via the Google Maps API (not assumed to be
  zero).
- If travel time < 5 minutes: no buffer needed — appointments can be
  scheduled back-to-back.
- If travel time ≥ 5 minutes: normal blocking logic (Section 4.2) applies as
  usual.

---

## 5. Future Conflict Detection (Agent-Facing Warning)

Runs **after** a new appointment is booked, to check whether it creates a
problem for an appointment that comes *after* it.

```
1. New appointment booked: ends at New_Appointment_End, location L2.
2. Find the next existing appointment after it (location L3, starts at
   Required_Arrival_Time).
3. Calculate:
     Travel_Time = GoogleMaps(origin: L2, destination: L3,
                               departure_time: New_Appointment_End)
     Actual_Arrival_Time = New_Appointment_End + Travel_Time
     Delay = Actual_Arrival_Time − Required_Arrival_Time
4. If Delay > 0: show the agent (never the client) a warning:
     "You may be delayed by ~{Delay} minutes for your next appointment at
      {location, time}. Do you want to proceed?"
5. Agent chooses to acknowledge and keep both appointments, or cancel the new
   one. This is a soft warning — never a hard block.
```

Only the **immediately next** appointment is checked; conflicts are not
cascaded further down the day automatically (see Section 6 for what happens
when the schedule actually changes).

---

## 6. Rescheduling and Cancellations

- **Agent cancels an appointment** → all slots that were blocked because of it
  become available again; recalculate visibility for everything after it.
- **Agent reschedules an appointment** (time or location change) → recalculate
  blocking for every appointment after the rescheduled one. If this creates a
  conflict, show the agent an error and require manual resolution — there is
  no automatic shifting of other appointments.
- Only agents can cancel or reschedule. Per the MVP scope, clients cannot
  cancel or reschedule their own bookings.

---

## 7. Supporting Data & API Changes

### 7.1 Reference table: city → zip codes

A centrally maintained lookup, independent of any single agent:

```sql
CREATE TABLE zip_code_reference (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  country VARCHAR(100) NOT NULL,
  city_name VARCHAR(100) NOT NULL,
  zip_code VARCHAR(20) NOT NULL,
  latitude DECIMAL(10, 8),
  longitude DECIMAL(11, 8),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  UNIQUE(country, city_name, zip_code)
);

CREATE INDEX idx_zip_code_reference_zip ON zip_code_reference(country, zip_code);
CREATE INDEX idx_zip_code_reference_city ON zip_code_reference(country, city_name);
```

Sourced from a postal-code provider (e.g. Canada Post data) and refreshed
periodically (suggest quarterly, or whenever an agent's serviceable cities
change and a lookup miss occurs).

### 7.2 Agent serviceable cities

```sql
CREATE TABLE agent_serviceable_cities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID NOT NULL REFERENCES agents(id),
  city_name VARCHAR(100) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  UNIQUE(agent_id, city_name)
);

CREATE INDEX idx_agent_serviceable_cities_agent ON agent_serviceable_cities(agent_id);
```

`serviceable_zip_codes` for an agent is computed as a join: every
`zip_code_reference` row whose `city_name` matches one of the agent's rows in
this table. This is a derived view, not stored redundantly.

### 7.3 Appointments table

No schema change needed beyond what already exists in
`location-aware-scheduling-database-structure.md` — `location_latitude`,
`location_longitude`, `location_address`, and `location_postal_code` on the
`appointments`/`clients` tables already hold exactly the final
`client_location` value described in this doc. `location_source` should be
set to `"manual"` (street address) for every appointment created through this
flow; `"device_gps"` is reserved for cases explicitly outside this flow (see
Section 3.8).

### 7.4 New endpoint — validate zip code

Optional, for instant feedback in the UI before the client reaches the
calendar:

```
GET /agents/{agent_id}/validate-zip-code?zip_code=M5V3A8

200 OK
{
  is_valid: true,
  city: "Toronto",
  is_serviceable_by_agent: true,
  message: "We serve Toronto! Select a date below."
}
```

### 7.5 Existing endpoint — available slots (no change)

`GET /agents/{agent_id}/available-slots` already accepts
`client_latitude`/`client_longitude` and is location-source-agnostic exactly
as designed in Section 4 — it's called once with the zip centroid (Step 2)
and, if Section 3.6's mismatch case fires, called again with the street
address location.

---

## 8. Key Assumptions & Constraints

✅ All appointment times are in the agent's local timezone
✅ Appointment duration is fixed per agent; client cannot change it
✅ No prep/cleanup buffer beyond the 5-minute travel buffer in MVP
✅ Google Maps Distance Matrix API provides departure-time-aware travel
   estimates (accounts for traffic)
✅ Each agent's schedule and blocking logic is independent of other agents
✅ Travel times are cached once per day per agent (Section 9) rather than
   queried live for every slot check
✅ The 5-minute buffer is hard-coded for MVP
✅ `client_location` has exactly one current value at any time — it is
   replaced, never merged, as better data becomes available
✅ Device GPS never feeds into travel-time calculations
✅ Only agents (not clients) can cancel or reschedule appointments

---

## 9. Travel Time Caching

- Departure time used: the end time of the previous appointment.
- Source: Google Maps Distance Matrix API.
- All travel times needed for a given agent's day are loaded once, in a
  single batch call, at the start of the day — not recalculated per slot
  request — because most of a day's appointments remain stable once set.
- Cache entries are keyed by agent + origin/destination client pair + date,
  and expire at end of day (see `travel_time_cache` table in the database
  spec).

---

## 10. Error Handling & Edge Cases — Consolidated Reference

| Case | Trigger | System response |
|---|---|---|
| Invalid zip code | Zip code not found in reference table | "Invalid zip code. Please try again." |
| Out-of-service zip code | Zip code's city not in agent's serviceable cities | "We don't serve that area yet." |
| Address geocoding fails | Street address can't be resolved to coordinates | "We couldn't find that address. Please check the spelling." |
| Address city ≠ zip code city | Geocoded street address resolves to a different city than the Step 1 zip code | Warn client; recalculate slots using the street address (Section 3.6); client re-confirms |
| Future conflict on booking | New appointment's travel time would delay the next existing appointment | Warn the **agent only**; agent acknowledges or cancels — never a hard block |
| Appointment cancelled | Agent cancels | Recalculate visibility for all later slots that day |
| Appointment rescheduled | Agent changes time or location | Recalculate visibility for everything after it; manual resolution required if a new conflict appears |

---

## 11. Testing Checklist

- [ ] Zip code normalization (whitespace, casing) is consistent
- [ ] Reference table has full coverage for every city an agent might select
- [ ] Zip-to-city lookup returns the correct city
- [ ] Agent serviceable-city changes correctly update derived
      `serviceable_zip_codes`
- [ ] Invalid / out-of-area zip codes show the correct error and block
      progress
- [ ] Zip centroid geocoding is reasonably accurate for slot calculation
- [ ] Available slots in Step 2 use the zip centroid, not a street address
- [ ] Street address geocoding succeeds for well-formed addresses and fails
      gracefully for malformed ones
- [ ] Address/zip city mismatch triggers a recalculation, not a silent
      override
- [ ] Final booked appointment stores the street-address location, never the
      zip centroid
- [ ] All travel-time calculations after Step 3.5 use the street address
      location exclusively
- [ ] Device GPS, even if granted, never appears in any travel-time call
- [ ] Future-conflict warning appears only to the agent, never the client
- [ ] Cancelling/rescheduling correctly recalculates only the affected later
      slots

---

## 12. One-Paragraph Summary (for quick recall)

An agent declares which cities they serve; the system derives every zip code
inside those cities automatically. A client enters a zip code, which is
validated against that list and geocoded into a rough centroid — that
centroid temporarily stands in as `client_location` so the system can show
real slot availability using the travel-time algorithm (first appointment of
the day is unblocked; every later slot is blocked until the agent could
plausibly arrive, with a 5-minute buffer). Once the client provides a full
street address, that address is geocoded and permanently replaces the
centroid as `client_location` — if it turns out to be in a meaningfully
different place, slot availability is recalculated against the real address
before the client can confirm. The booked appointment always carries this
final, precise location, which is what every later conflict check, agent
dashboard view, and cancellation/reschedule recalculation uses going forward.
