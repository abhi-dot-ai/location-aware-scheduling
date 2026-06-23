---
title: Zip Code to Client Location Bridge Logic
created_at: '2026-06-23T00:00:00Z'
tags:
- booking
- location-aware
- zip-code
- client-flow
summary: Complete algorithm connecting agent serviceable cities, client zip code entry, geocoding, address collection, and travel-time calculations for slot visibility.
---

## Zip Code to Client Location Bridge Logic

### Overview

This document describes the complete flow from when a client enters their zip code in Step 1 through to how their location is used in the booking-logic algorithm's travel-time calculations.

The bridge has **three distinct phases:**
1. **Agent Setup:** Agent configures serviceable cities
2. **Client Discovery:** Client enters zip code; system validates service area
3. **Slot Calculation:** Zip code is geocoded; final address collected; travel times calculated

---

## Phase 1: Agent Setup — Serviceable Cities Configuration

### When Agent Registers/Updates Profile

**Agent Input:**
- Agent selects a list of **city names** they're willing to service (e.g., "Toronto", "Mississauga", "Brampton")
- Agent does NOT manually enter zip codes

**System Action:**
- System automatically looks up all zip codes that belong to each city
- Store the mapping: `agent_id → [city_names] → [zip_codes]`
- This mapping is cached/refreshed whenever agent updates their serviceable cities

**Data Structure (Conceptual):**
```javascript
agents.serviceable_cities = [
  {
    city_name: "Toronto",
    zip_codes: ["M1A", "M1B", "M2A", "M2B", ... ]  // All Toronto postal codes
  },
  {
    city_name: "Mississauga", 
    zip_codes: ["L4T", "L4U", "L4V", "L4W", ... ]  // All Mississauga postal codes
  }
]
```

**External Data Source:**
- Maintain a master lookup table: `city_name → [zip_codes]`
- This can be built from:
  - Google Maps data (reverse geocode a sample of each city's coordinates)
  - Canada Post postal code database (if Canadian market)
  - Other geolocation APIs (if expanding to other regions)
- Update this mapping quarterly or as cities change boundaries

---

## Phase 2: Client Discovery — Zip Code Validation & Geocoding

### Step 1: Client Enters Zip Code (Web/App)

**What Client Does:**
- Opens agent's shareable link
- Prompted: "What's your zip code?" (or "Enter your postal code")
- Client types zip code (e.g., "M5V 3A8" in Canada or "10001" in US)

**System Action:**
```
INPUT: client_zip_code = "M5V 3A8"

NORMALIZE:
  - Remove spaces, convert to uppercase
  - Result: "M5V3A8"

LOOKUP CITY:
  - Query master city lookup table: "M5V3A8" → "Toronto"
  - If found, city_name = "Toronto"
  - If not found, city_name = NULL (not a valid zip code in database)
```

### Step 2: Validate Against Agent's Serviceable Cities

**System Logic:**
```
IF city_name == NULL:
  SHOW ERROR: "Invalid zip code. Please try again."
  RETURN to zip code input

IF city_name NOT IN agent.serviceable_cities:
  SHOW ERROR: "We don't serve {city_name} yet. Please check back soon!"
  RETURN to zip code input

IF city_name IN agent.serviceable_cities:
  PROCEED to Step 2.3
```

**Example:**
- Client enters "M5V3A8" (Toronto)
- Agent serves Toronto
- ✅ Proceed to calendar

---

- Client enters "L5N2X8" (Oakville)
- Agent serves Toronto and Mississauga only
- ❌ Show error: "We don't serve Oakville yet."
- Client must re-enter zip code

---

### Step 2.3: Geocode Zip Code to Lat/Long

**System Action:**
```
INPUT: client_zip_code = "M5V3A8"

GEOCODE:
  - Call Google Maps Geocoding API: geocode(client_zip_code)
  - Return: {latitude, longitude}
  - Example result: {latitude: 43.6426, longitude: -79.3957}

STORE TEMPORARILY:
  - session.zip_code_location = {
      zip_code: "M5V3A8",
      city: "Toronto",
      latitude: 43.6426,
      longitude: -79.3957,
      source: "zip_code"
    }
```

**Important:** This lat/long is derived from the zip code centroid (center of the postal code area), NOT the client's exact address yet.

---

### Step 3: Retrieve Available Slots

**System Action:**
```
CALL: GET /agents/{agent_id}/available-slots
  PARAMS:
    - date: {selected_date}
    - client_latitude: 43.6426    ← From zip code centroid
    - client_longitude: -79.3957
    
RESPONSE:
  - available_slots = [
      {start_time: "09:00", end_time: "09:30", is_available: true},
      {start_time: "09:30", end_time: "10:00", is_available: false, blocked_reason: "Travel time from previous appointment"},
      {start_time: "10:00", end_time: "10:30", is_available: true},
      ...
    ]
```

**Slot Calculation Logic:** Uses the booking-logic algorithm with `client_location = zip_code_centroid`

**Result Shown to Client:**
- Calendar with selected date
- Available slots displayed with blocked slots hidden/disabled
- No "blocked_reason" details shown to client (just unavailable slots grayed out)

---

## Phase 3: Address Collection & Final Location Lock

### Step 4: Client Provides Full Address (Details Form)

**What Client Enters:**
- Name (required)
- Street Address (required) — e.g., "123 Main St"
- City (auto-filled or editable) — e.g., "Toronto"
- Postal Code (optional, pre-filled from Step 1)
- Breed of dog (required)
- Disclaimers (required: checkbox)

**System Action:**
```
INPUT: 
  - street_address = "123 Main St"
  - city = "Toronto"
  - postal_code = "M5V3A8"

GEOCODE FULL ADDRESS:
  - Call Google Maps Geocoding API: geocode("123 Main St, Toronto, M5V3A8")
  - Return: {latitude, longitude, formatted_address}
  - Example: {latitude: 43.6431, longitude: -79.3951, formatted_address: "123 Main St, Toronto, ON M5V 3A8, Canada"}

STORE IN SESSION:
  - session.client_location = {
      latitude: 43.6431,
      longitude: -79.3951,
      address: "123 Main St",
      city: "Toronto",
      postal_code: "M5V3A8",
      source: "client_provided_address"
    }

VALIDATE AGAINST ZIP CODE:
  - IF geocoded_city != zip_code_city:
    SHOW WARNING: "You entered postal code {zip_code} ({zip_code_city}), but address is in {geocoded_city}. Is this correct?"
    - Client can confirm or correct
  - IF confirmed:
    UPDATE session.client_location with the geocoded address
```

**Key Point:** The full address is now the official `client_location` used for travel-time calculations going forward. The zip-code centroid is replaced.

---

### Step 5: Phone Number & Confirmation

**What Client Enters:**
- Phone number (required) — for SMS confirmation after agent approval

**System Action:**
```
STORE APPOINTMENT DATA:
  - appointment.client_location = session.client_location
    {
      latitude: 43.6431,
      longitude: -79.3951,
      address: "123 Main St, Toronto, ON M5V 3A8, Canada",
      source: "client_provided_address"
    }
  - appointment.client_phone = {phone_number}
  - appointment.client_name = {name}
  - appointment.breed_of_dog = {breed}
  - appointment.status = "booked"  ← Per JTBD: awaiting agent approval

SEND TO API:
  POST /appointments
    {
      agent_id,
      client_id,
      start_time,
      client_location: {
        latitude: 43.6431,
        longitude: -79.3951,
        address: "123 Main St, Toronto, ON M5V 3A8, Canada",
        source: "client_provided_address"
      },
      client_notes: {notes},
      confirm: true
    }

RESPONSE:
  - Appointment created with status "booked"
  - If future conflict detected, API returns warning (agent will acknowledge later)
  - Client sees: "Processing... We'll text you once [Agent Name] confirms your appointment."
```

---

## Complete Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     CLIENT BOOKING JOURNEY                   │
└─────────────────────────────────────────────────────────────┘

STEP 1: ZIP CODE ENTRY
  ├─ Client enters: "M5V3A8"
  ├─ System normalizes: "M5V3A8"
  ├─ System looks up city: "Toronto"
  ├─ System validates against agent's serviceable_cities: ✓ YES
  └─ System geocodes: {lat: 43.6426, lng: -79.3957} ← ZIP CENTROID
        └─ Stored as: session.zip_code_location
        └─ Used for: available-slots API call

STEP 2: CALENDAR & SLOT SELECTION
  ├─ Client selects date
  ├─ System calls: GET /agents/{id}/available-slots
  │  └─ PARAMS: date, client_latitude: 43.6426, client_longitude: -79.3957
  │  └─ Uses booking-logic algorithm with ZIP CENTROID
  ├─ System shows available slots
  └─ Client selects time slot (e.g., 09:00 AM)

STEP 3: DETAILS FORM (Address Collection)
  ├─ Client enters:
  │  ├─ Name
  │  ├─ Street Address: "123 Main St"
  │  ├─ City: "Toronto"
  │  ├─ Postal Code: "M5V3A8" (pre-filled)
  │  └─ Breed of Dog
  ├─ System geocodes full address: {lat: 43.6431, lng: -79.3951} ← PRECISE ADDRESS
  ├─ System validates address city matches zip code city: ✓ YES
  └─ Stored as: session.client_location ← REPLACES ZIP CENTROID
        └─ Used for: appointment booking & future travel-time calculations

STEP 4: PHONE NUMBER & CONFIRMATION
  ├─ Client enters phone number
  ├─ System books appointment via API:
  │  └─ Uses session.client_location (PRECISE ADDRESS)
  │  └─ Calculates future conflicts using booking-logic
  ├─ Status: "booked" (awaiting agent approval)
  └─ Client sees: "Processing... confirmation after agent approves"

STEP 5: AGENT APPROVAL (Backend)
  ├─ Agent reviews appointment in their dashboard
  ├─ If future conflict exists:
  │  └─ Agent sees warning: "May be delayed 30 min for next appointment"
  │  └─ Agent can acknowledge & approve OR cancel
  ├─ Agent approves
  └─ System sends SMS to client: "Confirmed! [Agent] will be here at 9:00 AM"
```

---

## Location Data Priority & Fallback Logic

**Priority order for travel-time calculations:**

```
1. Client-provided street address (Step 4 geocoded) ← HIGHEST PRIORITY
   └─ If available, use this for all travel-time calculations
   
2. Zip code centroid (Step 1 geocoded)
   └─ If no street address yet (shouldn't happen for booked appointments)
   
3. Device GPS (Step 5, optional)
   └─ NOT used for travel-time calculations
   └─ Used only for map display/agent routing after approval

Current Implementation: Always use #1 (street address) for booked appointments
```

---

## Database Schema Updates Needed

To support this logic, ensure the database has:

### 1. Master City-to-Zip-Code Lookup Table

```sql
CREATE TABLE zip_code_master (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  country VARCHAR(100) NOT NULL,
  city_name VARCHAR(100) NOT NULL,
  zip_code VARCHAR(20) NOT NULL,
  latitude DECIMAL(10, 8),
  longitude DECIMAL(11, 8),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(country, city_name, zip_code),
  INDEX idx_zip_code_lookup (country, zip_code),
  INDEX idx_city_lookup (country, city_name)
);
```

### 2. Agent Serviceable Cities

```sql
-- Add to agents table or create separate table:

CREATE TABLE agent_serviceable_cities (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID NOT NULL REFERENCES agents(id),
  city_name VARCHAR(100) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(agent_id, city_name),
  INDEX idx_agent_cities (agent_id)
);

-- When agent updates serviceable cities:
-- 1. DELETE all rows for that agent_id
-- 2. INSERT new rows for each selected city
-- 3. Query zip_code_master to hydrate all zip codes for those cities
```

### 3. Appointments Table (Already Exists)

Ensure `client_location` fields store the final geocoded address:

```sql
-- Already in schema:
location_latitude DECIMAL(10, 8),        ← Street address lat
location_longitude DECIMAL(11, 8),       ← Street address lng
location_address VARCHAR(500),           ← Street address
location_source VARCHAR(20),             ← "manual" or "device_gps"

-- Also store zip code for reference (optional):
location_postal_code VARCHAR(20)         ← From Step 1 or Step 4
```

---

## API Endpoint Changes Needed

### New Endpoint: Validate Zip Code (Optional, for real-time validation in UI)

```
GET /agents/{agent_id}/validate-zip-code?zip_code=M5V3A8

Response:
{
  is_valid: true,
  city: "Toronto",
  is_serviceable_by_agent: true,
  message: "We serve Toronto! Select a date below."
}

OR

{
  is_valid: true,
  city: "Oakville",
  is_serviceable_by_agent: false,
  message: "We don't serve Oakville yet."
}

OR

{
  is_valid: false,
  message: "Invalid zip code. Please try again."
}
```

### Existing Endpoint: GET /agents/{agent_id}/available-slots

**Already supports this flow — no changes needed:**
- Client passes `client_latitude` and `client_longitude` (from zip code centroid)
- System calculates slots using booking-logic algorithm
- Returns available slots

---

## Key Design Decisions

✅ **Zip code is a coarse filter**, not the definitive location  
✅ **Street address is the authoritative location** for travel-time calculations  
✅ **System auto-looks-up zip codes from cities**, reducing agent data entry  
✅ **Zip code centroid used for slot visibility**, then replaced by precise address  
✅ **Address validation** checks that provided street address matches zip code city  
✅ **GPS is optional** and used for map display only, not travel calculations  
✅ **Serviceable cities are agent-configurable**, not hard-coded  
✅ **Master zip-code table maintained centrally**, updated quarterly  

---

## Error Handling & Edge Cases

### Case 1: Client Enters Invalid Zip Code
```
User types: "12345" (not a valid postal code)
System: "Invalid zip code. Please try again."
Resolution: Client re-enters valid zip code
```

### Case 2: Client Enters Zip Code Outside Service Area
```
User types: "L5N2X8" (Oakville)
Agent serves: Toronto, Mississauga
System: "We don't serve Oakville yet. Please check back soon!"
Resolution: Client re-enters different zip code or leaves
```

### Case 3: Street Address City Doesn't Match Zip Code City
```
Step 1: Client enters zip "M5V3A8" (Toronto)
Step 4: Client enters address "456 Oak Ave, Mississauga"
System: "You entered postal code M5V (Toronto), but address is in Mississauga. Is this correct?"
Client: Confirms → uses Mississauga address as client_location
Client: Corrects → updates address to Toronto
```

### Case 4: Street Address Geocoding Fails
```
Step 4: Client enters "123 Fake Nonexistent St, Toronto"
Geocoding API: Returns no results
System: "We couldn't find that address. Please check the spelling or enter a different address."
Resolution: Client corrects address
```

### Case 5: Client Enters Zip Code for One City, Books in Another
```
Step 1: Client enters "M5V3A8" (Toronto), slots calculated with Toronto centroid
Step 4: Client provides address in Mississauga
Step 5: Address submitted for booking

RISK: Slots shown were based on Toronto centroid, but appointment is actually in Mississauga — travel times may be different!

MITIGATION: When address is provided in Step 4 and city != zip_code_city, system should:
- Warn client: "Your appointment location is in Mississauga, which is different from the zip code area. Availability may change."
- Option A: Recalculate slots for the new address and show updated availability
- Option B: Allow booking to proceed (simpler MVP) but flag for agent review
```

---

## Testing Checklist

- [ ] Zip code normalization (spaces, case) works correctly
- [ ] Master zip-code table has complete coverage for all serviceable cities
- [ ] Zip code lookup returns correct city
- [ ] Agent serviceable cities can be updated; zip codes hydrated correctly
- [ ] Invalid zip code shows error message
- [ ] Out-of-service-area zip code shows error message
- [ ] Zip code geocoding returns correct centroid
- [ ] Available slots calculated with zip centroid (not street address)
- [ ] Street address geocoding works correctly
- [ ] Address-city-mismatch warning shown when needed
- [ ] Final appointment stored with street address location
- [ ] Travel times use street address, not zip centroid
- [ ] Edge case: zip code from one city, address from different city — handled gracefully

---

## Summary

The **zip code → client location → slot visibility** bridge works as follows:

1. **Zip Code Entry (Step 1):** Client enters zip code → normalized → looked up in master table → validated against agent's serviceable cities → geocoded to centroid → used for available-slots calculation
2. **Slot Visibility (Step 2-3):** Slots calculated using zip-code centroid via booking-logic algorithm
3. **Address Collection (Step 4):** Client provides street address → geocoded to precise lat/long → replaces zip centroid in session
4. **Appointment Booking (Step 5):** Street address is the definitive client_location stored and used for all future travel-time calculations
5. **Agent Approval:** Agent sees appointment details with street address; future conflicts calculated using that precise location

**Key principle:** Zip code is for discovery and filtering; street address is for accuracy.
