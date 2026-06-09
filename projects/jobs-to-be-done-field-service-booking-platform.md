---
title: Jobs to Be Done – Field Service Booking Platform
created_at: '2026-06-08T18:44:30Z'
updated_at: '2026-06-08T18:44:51Z'
tags:
- mvp
- field-service
- jobs-to-be-done
- booking-platform
- project-requirements
summary: Defines the core feature set for an MVP field service booking platform where
  clients book location-based appointments with agents, agents manage availability
  calendars, and bookings require agent approval before confirmation.
---

## 1. Client: Availability Discovery Based on Location
When booking an appointment, clients must enter their zip code or address first. This location data determines which field service agents are available to service them and what appointment slots are realistically open. The system calculates probable booking times based on:
- The field service agent's previous appointment wrap-up time
- Distance between the agent's current location (previous appointment) and the client's location
- Commute time to the client's location

This gives clients accurate, real-world availability windows before they commit to booking.

## 2. Field Service Agent: Calendar Setup & Publishing
Field service agents configure their availability by:
- Setting appointment duration (30 minutes for MVP)
- Specifying the number of daily appointments
- Defining start and end times for their service window
- Selecting which cities/locations they want to service
- Choosing to apply this schedule across the entire week or to individual days
- Publishing the calendar when ready

Once published, the agent's calendar generates a shareable link (similar to Calendly) that clients use to book appointments.

## 3. Client: Booking Flow & Approval Workflow
The client booking experience follows this sequence:
1. **Location Entry** (mandatory): Client provides zip code or address
2. **Availability Display**: System shows available time slots based on location and agent proximity
3. **Booking Selection**: Client selects their preferred appointment time
4. **Details Collection**: Client provides name, contact information, and exact service address to reduce friction
5. **Pending Approval**: Booking enters a "Processing" state—no confirmation is sent yet
6. **Agent Approval**: Field service agent reviews and approves the booking
7. **Final Confirmation**: Once approved, client receives a confirmation of the appointment

Until the agent approves, clients see only a "processing" status.

## 4. Scope Constraints for MVP
- **No cancellations**: Clients cannot cancel booked appointments
- **No rescheduling**: Clients cannot move or modify booked time slots
- **No login required**: Clients access and book without creating an account; the shareable link is the only entry point

---

This keeps the MVP lean while delivering the core value: location-aware availability matching clients to nearby agents with realistic time windows.