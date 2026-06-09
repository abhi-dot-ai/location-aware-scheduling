---
title: 'Development Roadmap: Field Service Scheduler (1-2 Weeks)'
created_at: '2026-06-08T18:16:26Z'
updated_at: '2026-06-08T18:18:01Z'
tags:
- project
- claude
- mvp
- roadmap
- development
- week1
- week2
summary: Week-by-week development roadmap for Field Service Appointment Scheduler
  MVP. Week 1 focuses on tech stack decision, architecture design, and backend core
  (distance API, availability engine). Week 2 focuses on frontend (agent setup, client
  booking), integration, and testing. Designed for Claude.ai-assisted development
  with clear handoff points.
---

## Development Roadmap: 1-2 Week MVP Build

**Timeline:** Sprint of 5-7 development days (1-2 weeks calendar time)  
**Approach:** Claude.ai-assisted development with clear iteration loops

---

## WEEK 1: Core Engine & Backend Foundation

### Day 1: Tech Stack Decision & Architecture Design
**Goal:** Finalize tech stack and design system architecture

**Activities:**
- [ ] Decide: Next.js (React + Node) OR Python FastAPI + React frontend
  - **Recommendation:** Next.js (faster iteration, single codebase, built-in auth)
- [ ] Map out data models:
  - `Agent` (id, email, service_zones, service_duration, created_at)
  - `ServiceZone` (agent_id, zip_code, available_from, available_to)
  - `Appointment` (id, agent_id, client_email, client_zip, scheduled_time, duration, status)
- [ ] Identify external APIs:
  - Distance/Travel Time: Google Maps API or MapBox
  - Email: SendGrid or Resend
- [ ] Database: PostgreSQL (or SQLite for MVP simplicity)
- [ ] Hosting: Vercel (Next.js), Railway (PostgreSQL), or similar

**Claude.ai Prompts to Use:**
1. "Design a database schema for a location-aware field service scheduler. I have agents with service zones (zip codes), and appointments. Help me normalize this and identify the key tables."
2. "Compare Next.js vs FastAPI + React for building a field service scheduler MVP in 1-2 weeks. What are the tradeoffs?"

**Deliverable:** Architectural diagram (can be ASCII or hand-drawn) + tech stack decision document

---

### Day 2: Core Availability Engine Logic
**Goal:** Build the brain of the system—calculating dynamic availability

**Activities:**
- [ ] Design the availability calculation algorithm:
  ```
  For each appointment slot (e.g., 9:00 AM, 10:00 AM, 11:00 AM):
    1. Get agent's last appointment end time
    2. Get travel time from last appointment zip to client's zip (API call)
    3. Calculate earliest available time = last_end + travel_time
    4. If earliest_available <= slot_time, slot is available
    5. Generate list of available slots for the day/week
  ```
- [ ] Decide: Fixed 1-hour slots vs flexible 30-min slots
  - **Recommendation:** Fixed 1-hour slots (simpler for MVP)
- [ ] Plan caching strategy for distance/travel time (avoid API hammering)
- [ ] Outline error handling: What if API fails? Fallback behavior?

**Claude.ai Prompts:**
1. "I'm building a scheduling system where availability depends on travel time between appointments. Here's my data model [paste schema]. Help me write pseudocode for calculating available time slots for an agent on a given day."
2. "How should I cache Google Maps distance API results to avoid hitting rate limits? I have ~50 agents and 1000+ bookings/week."

**Deliverable:** Pseudocode or algorithm flowchart + caching strategy document

---

### Day 3-4: Backend Implementation (Core Engine + API)
**Goal:** Build the backend with availability calculation and booking logic

**Activities:**
- [ ] Set up Next.js project (or FastAPI + Postgres)
  - [ ] Database setup and migrations
  - [ ] Models for Agent, ServiceZone, Appointment
- [ ] Implement availability calculation endpoint:
  - `POST /api/availability` → takes agent_id, client_zip, date → returns available slots
  - [ ] Calls distance API (Google Maps / MapBox)
  - [ ] Caches results
  - [ ] Returns list of 1-hour slots with realistic start times
- [ ] Implement booking endpoint:
  - `POST /api/bookings` → takes agent_id, client_email, client_zip, slot_time → creates appointment
  - [ ] Validates no overlap
  - [ ] Sends confirmation email (SendGrid stub for now)
- [ ] Implement agent endpoints:
  - `POST /api/agents/setup` → agent registers, sets service zones
  - `GET /api/agents/{id}/appointments` → list upcoming appointments

**Claude.ai Prompts:**
1. "I'm using Next.js with Prisma. Here's my schema [paste]. Build me the availability calculation function. It should take agent_id, client_zip_code, and date, then return available 1-hour slots accounting for travel time from the agent's previous appointment."
2. "Write the API endpoint for booking an appointment. It should check for overlaps, create the booking, and trigger an email confirmation."

**Deliverable:** Working backend with /api/availability, /api/bookings, /api/agents endpoints

---

### Day 5: Distance API Integration & Testing
**Goal:** Wire up real distance/travel time data and test core logic

**Activities:**
- [ ] Sign up for Google Maps API (or MapBox)
- [ ] Integrate distance matrix API into availability calculation
- [ ] Test with real zip code pairs (e.g., Vancouver to Abbotsford = 60 km, ~45 min)
- [ ] Write unit tests for availability logic:
  - Test case: Agent with 2 appointments (9-10 AM in zip A, 10:30-11:30 AM in zip B) + new booking request in zip C
  - Verify correct travel time is factored in
- [ ] Test edge cases:
  - First appointment of the day (no previous job)
  - Day with no appointments yet
  - Zip code outside service zones (should fail)

**Claude.ai Prompts:**
1. "Help me test the availability calculation logic. Write test cases for these scenarios: [list scenarios]"
2. "I'm getting timeouts calling Google Maps API. How should I optimize the availability endpoint for performance?"

**Deliverable:** Working distance integration + test suite

---

## WEEK 2: Frontend & Integration

### Day 6: Agent Setup & Dashboard
**Goal:** Build minimal agent self-service interface

**Activities:**
- [ ] Create agent authentication (simple email/password with next-auth)
- [ ] Build agent setup form:
  - [ ] Enter service zones (add multiple zip codes, one per row)
  - [ ] Set default service duration (e.g., 1 hour)
  - [ ] One-time setup, no editing for MVP
- [ ] Build agent dashboard:
  - [ ] List of upcoming appointments (read-only)
  - [ ] Show booking date, client email, client zip, appointment time, travel time to next
  - [ ] Button to export to iCal (can be a simple link for MVP)
- [ ] Keep UI minimal: no calendar grid, no drag-and-drop

**Claude.ai Prompts:**
1. "Build me a React component for agent onboarding. It should have a form to enter service zones (multiple zip codes) and default job duration. Make it simple and mobile-friendly."
2. "Create a simple React component to display upcoming appointments with travel time to the next appointment."

**Deliverable:** Agent signup/login + setup form + dashboard with appointment list

---

### Day 7: Client Booking Interface
**Goal:** Build the public booking flow

**Activities:**
- [ ] Create booking page (no auth required):
  - [ ] Select agent (dropdown or link via agent-specific URL)
  - [ ] Enter zip code
  - [ ] Select date
  - [ ] See available time slots (dynamically generated)
  - [ ] Enter email, phone
  - [ ] Confirm booking
- [ ] Call `/api/availability` to populate time slots in real-time
- [ ] Call `/api/bookings` to create appointment
- [ ] Show confirmation page with details + add to calendar link (iCal file)
- [ ] Keep UI minimal and mobile-friendly

**Claude.ai Prompts:**
1. "Build me a React booking form. User selects their zip code and a date, and I show them available time slots from my backend. Make it simple and responsive."
2. "How do I generate an .ics (iCal) file in Node.js so users can download it and add the appointment to their calendar?"

**Deliverable:** Public booking page + confirmation flow

---

### Day 8: Email & iCal Export
**Goal:** Wire up communication and calendar integration

**Activities:**
- [ ] Set up SendGrid (or Resend) for transactional email
- [ ] Send confirmation email to client after booking:
  - [ ] Appointment details (date, time, agent, location)
  - [ ] Confirmation link (to view appointment)
  - [ ] iCal file attachment (or link to download)
- [ ] Implement iCal export for agents:
  - [ ] Endpoint `/api/agents/{id}/calendar.ics` → returns iCal file
  - [ ] Includes all upcoming appointments with travel time notes
  - [ ] Agents can import into Google Calendar
- [ ] Test email delivery and formatting

**Claude.ai Prompts:**
1. "Write a function to generate an iCal (.ics) file for a set of appointments. Include travel time between appointments as notes."
2. "Set up SendGrid transactional email in my Next.js app. I need to send booking confirmations with appointment details."

**Deliverable:** Email confirmations + iCal export working end-to-end

---

### Day 9: Integration Testing & Polish
**Goal:** End-to-end testing and MVP readiness

**Activities:**
- [ ] Full user flow test:
  - [ ] Agent signs up, sets service zones
  - [ ] Client books appointment (sees location-aware availability)
  - [ ] Confirmation email arrives
  - [ ] Agent can export to iCal and import into Google Calendar
  - [ ] Availability updates correctly when previous appointment is added
- [ ] Edge case testing:
  - [ ] Booking outside service zones (should fail gracefully)
  - [ ] Overlapping appointments (should prevent)
  - [ ] API failures (distance API down, email service down)
- [ ] Performance testing:
  - [ ] Availability calculation responds in < 500ms
  - [ ] Booking endpoint responds in < 1s
- [ ] UI polish:
  - [ ] Mobile responsiveness
  - [ ] Error messages are clear
  - [ ] Loading states where needed

**Claude.ai Prompts:**
1. "Help me write end-to-end test cases for this booking flow: [describe flow]"
2. "I'm getting slow response times on the availability endpoint. Here's my code [paste]. How can I optimize it?"

**Deliverable:** Tested, polished MVP ready for users

---

## Technical Checklist

### Backend
- [ ] Agent authentication (email/password)
- [ ] Agent setup endpoint (service zones, duration)
- [ ] Availability calculation engine with distance API
- [ ] Booking creation with overlap validation
- [ ] Email service integration (SendGrid)
- [ ] iCal export endpoint

### Frontend
- [ ] Agent onboarding form
- [ ] Agent dashboard (appointment list)
- [ ] Public booking page (no auth)
- [ ] Confirmation page + iCal download
- [ ] Error handling and loading states

### External Integrations
- [ ] Google Maps API (distance + travel time)
- [ ] SendGrid (email confirmations)
- [ ] Database (PostgreSQL or SQLite)
- [ ] Hosting (Vercel + Railway or equivalent)

### Testing
- [ ] Unit tests (availability logic, overlap validation)
- [ ] Integration tests (end-to-end booking flow)
- [ ] Manual E2E testing (full user journey)

---

## How to Use Claude.ai for Development

**Approach:** Break down each task into specific, code-focused prompts

**Good Prompt Structure:**
```
I'm building [brief context].
Here's my [data model / component / function].
I need to [specific task].
[Paste relevant code or pseudo-code]
Constraints: [time limit, tech stack, etc.]
```

**Example:**
```
I'm building a field service scheduler in Next.js.
Here's my Prisma schema for appointments.
I need to write an API endpoint that calculates available time slots
accounting for travel time between appointments.
The endpoint should accept agent_id, client_zip, and date.
Constraints: Must respond in < 500ms.
```

**Tips:**
1. **Provide context:** Paste your schema, existing code, or data structure
2. **Be specific:** "Write a function" vs "Help me optimize" produces different results
3. **Ask for explanation:** "Explain why you chose this approach"
4. **Iterate:** Use feedback loops ("Can you make this more performant?")
5. **Test as you go:** Have Claude.ai write test cases alongside code

---

## Post-MVP Roadmap (Not building in Week 1-2)

**Phase 2:** Real-time updates, recurring bookings, admin panel  
**Phase 3:** Payment processing, GPS tracking, team management  
**Phase 4:** Mobile app, advanced routing, integrations (Stripe, Slack, etc.)

---

## Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| API rate limits (distance API) | Implement caching; test with realistic volume |
| Slow availability calculations | Use background jobs or memoization for repeated queries |
| Timezone confusion | Store all times in UTC; convert on frontend |
| Email delivery failures | Log failures; provide manual resend option (post-MVP) |
| Database performance | Index agent_id, appointment_date columns |

---

## Success Criteria for MVP

✓ Agent can sign up and set service zones in < 5 minutes  
✓ Client can book an appointment and see location-aware availability  
✓ Travel time is correctly factored into availability  
✓ Confirmation email arrives within 1 minute of booking  
✓ iCal export works with Google Calendar  
✓ No overlapping appointments for same agent  
✓ System handles edge cases gracefully (API failures, invalid zip codes)
