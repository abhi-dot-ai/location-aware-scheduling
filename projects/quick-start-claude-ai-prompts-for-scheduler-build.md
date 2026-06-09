---
title: 'Quick-Start: Claude.ai Prompts for Scheduler Build'
created_at: '2026-06-08T18:16:56Z'
updated_at: '2026-06-08T18:18:01Z'
tags:
- project
- claude
- development
- prompts
- quickstart
- reference
summary: Quick-start copy/paste prompts for using Claude.ai to build the Field Service
  Scheduler. Organized by day/week with example prompts for architecture, backend,
  frontend, testing, and debugging. Includes tips for structuring prompts and iterating
  quickly.
---

## Quick-Start Claude.ai Prompts

Copy and paste these prompts directly into Claude.ai chat as you work through your sprint. Customize with your specific code/schema.

---

## WEEK 1 PROMPTS

### Day 1: Architecture & Tech Stack

**Prompt 1:**
```
I'm building a field service appointment scheduler MVP in 1-2 weeks.

Here's what I need:
- Agents set service zones (zip codes) and can see upcoming appointments
- Clients book appointments and see available times based on agent's location and travel time
- System automatically calculates availability accounting for travel time between jobs

Key constraints:
- MVP in 1-2 weeks
- Build with Claude.ai (so iterate fast)
- No admin panel, no in-app calendar, no recurring bookings

Should I use:
A) Next.js + Prisma + PostgreSQL
B) Python FastAPI + React + PostgreSQL
C) Something else?

For each option, tell me:
1. Time to MVP
2. Tradeoffs
3. Which is best for iteration with Claude?
```

**Prompt 2:**
```
Design a database schema for a location-aware field service scheduler.

Core data:
- Agents with service zones (multiple zip codes)
- Appointments with start/end times and locations (zip codes)
- Clients booking appointments

I need:
1. Table structure (normalized)
2. Key relationships
3. Indexes for performance
4. Example SQL (or Prisma schema)

The tricky part: I need to query available time slots based on:
- Agent's previous appointment end time
- Travel time from previous zip to new zip
- Service zones the agent covers
```

---

### Day 2: Availability Algorithm

**Prompt 3:**
```
I'm building the core availability calculation for a field service scheduler.

Here's my data model:
[paste your schema]

I need to calculate available time slots for an agent on a given day.

Given:
- agent_id
- client_zip_code
- date
- service_duration (default 1 hour)

Return:
- List of available 1-hour slots (e.g., [9:00 AM, 10:00 AM, 11:00 AM, ...])

Rules:
1. Agent only works 9 AM - 5 PM
2. Each appointment is 1 hour
3. Travel time between appointments depends on distance (I'll call Google Maps API)
4. Travel time is estimated: travel_time = distance / 60 km/h (rough estimate)
5. Example: If agent's last appointment ends at 10:00 AM and travel to next location is 30 min, earliest available slot is 10:30 AM

Write pseudocode or a detailed algorithm. Don't worry about the API call yet.
```

**Prompt 4:**
```
I'm calling Google Maps Distance Matrix API to calculate travel times.

Current approach: For each potential time slot, call the API.
Problem: This could be 50+ calls per booking request if there are 50 potential slots.

How should I:
1. Cache distance results? (same zip pairs recur often)
2. Optimize the availability calculation to minimize API calls?
3. Handle rate limiting?

Context: ~50 agents, ~1000 bookings/week, calling Google Maps API
```

---

### Day 3-4: Backend Implementation

**Prompt 5:**
```
I'm using Next.js with Prisma. Here's my schema:

[paste your Prisma schema]

I need to implement the availability calculation function in TypeScript/JavaScript.

Requirements:
- Input: agentId, clientZipCode, date
- Output: Array of available slots (times as ISO strings)
- Must factor in travel time between appointments
- Must validate zip code is in agent's service zones
- Must handle no previous appointment (first of day)

Write the function. You can assume Google Maps API call is already handled.
```

**Prompt 6:**
```
I need a Next.js API endpoint for booking.

Route: POST /api/bookings
Body:
{
  agentId: string,
  clientEmail: string,
  clientZip: string,
  slotTime: ISO datetime string
}

The endpoint should:
1. Validate the slot is available (call availability calculation)
2. Check for overlaps with existing appointments
3. Create the appointment in the database
4. Return the appointment details
5. Trigger email confirmation (SendGrid stub for now)

Write the handler.
```

---

### Day 5: Testing

**Prompt 7:**
```
Help me write test cases for the availability calculation function.

Function signature: getAvailableSlots(agentId, clientZip, date)

Test scenarios:
1. Agent with no appointments yet on that day
2. Agent with 1 appointment at 9-10 AM (zip A), booking request for zip B (30 min away)
3. Agent with 2 appointments: 9-10 AM (zip A), 10:45-11:45 AM (zip B), booking for zip C (15 min from B)
4. Booking request for zip outside agent's service zones
5. Booking request for date in past
6. Overlapping time slot (should be rejected)

Write Jest test cases with mock data.
```

---

## WEEK 2 PROMPTS

### Day 6: Agent Dashboard

**Prompt 8:**
```
I'm building a React component for the agent dashboard.

The page should show:
1. Agent name and service zones (read-only)
2. List of upcoming appointments (next 7 days)
3. For each appointment: client name, client zip, appointment time, duration, travel time to next appointment
4. Export to iCal button (just a link for MVP)

Keep it minimal—just a table or list. No calendar grid.

I'm using Next.js + React. The data comes from:
GET /api/agents/[id]/appointments

Write the component.
```

**Prompt 9:**
```
I need an agent onboarding form.

The form should:
1. Email (read-only, pre-filled)
2. Service zones: Add multiple zip codes (one per row, can add/remove)
3. Default service duration (in hours, default 1)
4. Submit button

On submit: POST /api/agents/setup

Keep it simple and mobile-friendly. I'm using Next.js + React Hook Form.

Write the component.
```

---

### Day 7: Client Booking

**Prompt 10:**
```
I'm building the public booking page.

Flow:
1. Select agent (I'll pass agentId in URL)
2. Enter zip code
3. Select date (radio buttons or date picker)
4. See available time slots (populated from backend, 1-hour slots)
5. Enter email and phone
6. Confirm booking
7. Show confirmation page with appointment details

The availability is dynamic: When user picks zip code and date, call:
GET /api/availability?agentId=X&clientZip=12345&date=2026-06-08

And display the slots.

Write the component (minimal, mobile-friendly, no fancy styling needed).
```

**Prompt 11:**
```
I need to generate an .ics (iCal) file in Node.js for a booking confirmation.

Input: Appointment object
{
  date: "2026-06-08",
  startTime: "10:00",
  endTime: "11:00",
  clientEmail: "client@example.com",
  clientName: "John Doe",
  agentName: "Jane Doe",
  notes: "Travel time from previous appointment: 15 minutes"
}

Output: .ics file content (as string)

The user should be able to download this and add it to Google Calendar.

Write the function.
```

---

### Day 8: Email & Integration

**Prompt 12:**
```
I'm using SendGrid in my Next.js app.

I need to send a booking confirmation email to the client.

Email should include:
- Appointment date/time
- Agent name
- Service address (or just the zip code)
- iCal file as attachment (or link to download)

I have the appointment details in my database.

Write the email send function using the SendGrid API (or use a library like Resend).
```

---

### Day 9: Testing & Polish

**Prompt 13:**
```
Help me write an end-to-end test scenario for the full booking flow.

Scenario:
1. Agent signs up and sets service zones (zip 12345, 12346)
2. Agent sets service duration to 1 hour
3. Agent gets 2 appointments booked: 9-10 AM (zip 12345), 11 AM - 12 PM (zip 12346)
4. Client visits booking page, selects zip 12347, sees available slots accounting for travel time
5. Client books 10:30 AM - 11:30 AM slot (travel from 12346 to 12347 is 30 min)
6. Client receives confirmation email with iCal attachment
7. Agent exports calendar to iCal and imports into Google Calendar

Write a Jest/Cypress test that validates this flow (or pseudocode).
```

---

## General Tips for Claude.ai Usage

### Structure Your Prompts Like This:
```
**Context:** [What you're building, tech stack, constraints]
**Given:** [What you have—code, schema, etc.]
**Need:** [What you want—function, component, etc.]
**Constraints:** [Time limit, performance requirements, etc.]
```

### When You Get Stuck:
1. "Explain why you chose this approach"
2. "How would you optimize this for performance?"
3. "Write test cases for this function"
4. "Refactor this to be more readable"

### Iterate Quickly:
1. Ask Claude to generate code
2. Test it (even a quick mental test)
3. Ask for improvements: "Make this simpler," "Add error handling," "Optimize the query"
4. Move on when it's "good enough" for MVP

### Copy/Paste Is Your Friend:
- Paste your actual schema, not a simplified version
- Paste error messages if you hit bugs
- Paste existing code you want to extend

---

## For Reference: Key Endpoints to Build

**Agent Setup:**
- `POST /api/agents` — Register agent
- `POST /api/agents/[id]/setup` — Set service zones

**Availability & Booking:**
- `GET /api/availability?agentId=X&clientZip=Y&date=Z` — Get available slots
- `POST /api/bookings` — Create booking
- `GET /api/agents/[id]/appointments` — List agent's appointments

**Calendar:**
- `GET /api/agents/[id]/calendar.ics` — Export to iCal

---

## Debugging Checklist

When something doesn't work:
- [ ] Is the API returning the right data? (Check browser DevTools Network tab)
- [ ] Is the distance API call working? (Test with a known zip pair first)
- [ ] Is the availability calculation logic correct? (Log intermediate values)
- [ ] Are time zones causing issues? (Use UTC everywhere)
- [ ] Is the email actually sending? (Check SendGrid dashboard)
- [ ] Are there database errors? (Check server logs)
