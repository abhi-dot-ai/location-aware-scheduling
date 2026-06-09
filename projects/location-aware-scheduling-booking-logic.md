---
title: Location Aware Scheduling - Booking Logic
created_at: '2026-06-08T19:33:31Z'
updated_at: '2026-06-08T19:35:27Z'
tags:
- scheduling
- location-aware
- booking
- algorithm
- logic
summary: Core algorithm for location-aware appointment booking, including availability
  slot calculation, travel time blocking, same-location rules, and future conflict
  detection with agent warnings.
---

## Location Aware Scheduling - Booking Logic

### Core Algorithm Overview

The location-aware scheduling system determines which appointment slots are visible to clients based on the field service agent's existing appointments and travel times between locations.

---

## Phase 1: First Appointment of the Day

- **Earliest visible slot:** Agent's configured start time
- **No location-based blocking** applies
- **Example:** If agent starts at 8:00 AM, the first slot (8:00-8:30) is immediately available

---

## Phase 2: Subsequent Appointments (With Existing Appointments)

### Step 1: Identify Most Recent Appointment
- Find the most recent appointment before the slot client is trying to book
- Example: Client wants 9:00-9:30 slot, agent has 8:00-8:30 appointment

### Step 2: Calculate Blocking Window

```
Last_Appointment_End = 8:30
Travel_Time = Google Maps(origin: last_location, destination: client_location, departure_time: 8:30)
  = 35 minutes (in example)

Calculated_Arrival = Last_Appointment_End + Travel_Time
  = 8:30 + 35 min = 9:05

Blocking_Until = Calculated_Arrival - 5_min_buffer
  = 9:05 - 5 min = 9:00

SHOW SLOTS FROM: 9:00 onwards
HIDE SLOTS UNTIL: 8:59 (i.e., 8:00-8:30, 8:30-9:00 are blocked)
```

### Step 3: Apply Logic for Each Subsequent Appointment Request
- If client later tries to book 10:00 AM slot, recalculate from the 9:00-9:30 appointment (now the most recent)
- The 5-minute buffer is a TOLERANCE, not a deduction
  - Travel time = 35 min
  - Last appointment ends = 8:20
  - Available slot = 8:20 + 35 - 5 = 8:50
  - But we SHOW slots from 9:00 onwards (the buffer is just accepted lateness)

---

## Same-Location Rule

**If previous appointment location = new appointment location:**
- Travel time = distance via Google Maps API
- **If travel time < 5 minutes:** No buffer needed, appointments can be back-to-back
- **If travel time ≥ 5 minutes:** Apply normal blocking logic

---

## Future Conflict Detection & Warning

**When:** After a new appointment is booked at time T with location L2

**Check:** Is there an existing appointment after the newly booked one?
- Example: New appointment booked at 9:00-9:30 at Location B. Existing appointment at 10:00-10:30 at Location C.

### Calculate Conflict

```
New_Appointment_End = 9:30
Travel_Time = Google Maps(origin: L2, destination: Location_C, departure_time: 9:30)
  = 60 minutes (in example)

Required_Arrival_Time = 10:00 (start of next appointment)
Actual_Arrival_Time = 9:30 + 60 min = 10:30

Time_Conflict = Actual_Arrival_Time - Required_Arrival_Time
  = 10:30 - 10:00 = 30 minutes late
```

### Action

- Show **WARNING to agent (not client):** "You may be delayed by ~30 minutes for your next appointment at [Location C, 10:00 AM]. Do you want to proceed?"
- Agent can **OVERRIDE and APPROVE** or **CANCEL** the new booking
- No hard block, just a warning that agent can dismiss and continue

---

## Rescheduling & Cancellations

### If Agent Cancels an Appointment
- All subsequent blocked time slots become available
- Recalculate all future appointment visibilities

### If Agent Reschedules an Appointment (Time or Location Change)
- Recalculate blocking for all appointments after the rescheduled one
- If new schedule creates conflicts with existing appointments, show error popup to agent
- **No automatic changes;** agent must manually resolve
- Error message: "Rescheduling creates a {X} minute delay for your next appointment at {time}. Please resolve manually."

---

## Key Constraints & Assumptions

✅ All times in agent's local timezone  
✅ Appointment duration is fixed per agent (client cannot override)  
✅ No prep/cleanup buffer in MVP  
✅ Google Maps API provides departure-time-aware travel estimates  
✅ Each agent has independent schedule & logic  
✅ Time blocks are configurable (default: 30 min)  
✅ Only agents can cancel/reschedule appointments  
✅ Travel time is cached at the start of each day  
✅ 5-minute buffer is hard-coded for MVP  

---

## Travel Time Calculation

- **Departure time:** End time of previous appointment
- **Source:** Google Maps Distance Matrix API
- **Traffic consideration:** Accounts for traffic at departure time
- **Caching:** All required travel times loaded at beginning of day in single call
- **Reason:** Most appointments in a given day remain the same, so batch loading is efficient

---

## Special Cases Handled

1. **Back-to-back appointments at same location** - No buffer needed if travel time < 5 min
2. **Multiple future appointments** - Only check the NEXT appointment for conflict
3. **Rescheduling cascades** - All subsequent appointments' visibility recalculated
4. **Same location bookings** - Treated as 0 travel time if within 5 min threshold
