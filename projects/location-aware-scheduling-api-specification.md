---
title: Location Aware Scheduling - API Specification
created_at: '2026-06-08T19:34:21Z'
updated_at: '2026-06-08T19:35:27Z'
tags:
- api
- rest
- endpoints
- http
- specification
summary: Complete REST API specification with 21 endpoints covering agent scheduling,
  client booking, appointments, authentication, and admin functions, including detailed
  request/response models, error handling, rate limiting, and pagination.
---

## Location Aware Scheduling - API Specification

### Complete RESTful API Design with Endpoints, Request/Response Models

---

## Authentication & Authorization

All endpoints require JWT token in the Authorization header:

```
Authorization: Bearer <jwt_token>
```

**JWT Claims:**
```javascript
{
  sub: user_id,           // UUID of agent or client
  org_id: organization_id, // UUID
  role: "agent" | "client",
  iat: timestamp,
  exp: timestamp
}
```

---

## Base URL & Versioning

```
Base URL: https://api.schedulingservice.com/v1
```

---

## AGENT ENDPOINTS

### 1. Get Available Slots for Booking

**Endpoint:** `GET /agents/{agent_id}/available-slots`

**Purpose:** Client calls this to get available appointment slots for a specific agent on a specific date, considering location-based blocking.

**Request:**
```
GET /agents/{agent_id}/available-slots?date=2026-06-15&client_latitude=43.6532&client_longitude=-79.3832

Headers:
  Authorization: Bearer <jwt_token>
  Content-Type: application/json
```

**Query Parameters:**
```javascript
{
  date: string,               // YYYY-MM-DD, required
  client_latitude: number,    // decimal, required
  client_longitude: number,   // decimal, required
}
```

**Response (200 OK):**
```javascript
{
  agent_id: string,
  agent_name: string,
  date: string,
  timezone: string,
  available_slots: [
    {
      id: string,
      start_time: "2026-06-15T09:00:00Z",
      end_time: "2026-06-15T09:30:00Z",
      is_available: true,
      duration_minutes: number
    },
    {
      start_time: "2026-06-15T09:30:00Z",
      end_time: "2026-06-15T10:00:00Z",
      is_available: false,
      duration_minutes: 30,
      blocked_reason: "Travel time from previous appointment"
    }
  ],
  travel_time_cache_initialized: boolean,
  message: string (nullable)
}
```

---

### 2. Get Agent Profile & Scheduling Info

**Endpoint:** `GET /agents/{agent_id}/profile`

**Response (200 OK):**
```javascript
{
  id: string,
  email: string,
  name: string,
  phone: string,
  organization_id: string,
  start_time: string,           // HH:MM
  end_time: string,
  appointment_duration: number, // minutes
  timezone: string,
  profile_image_url: string (nullable),
  bio: string (nullable),
  is_active: boolean,
  created_at: string,
  updated_at: string
}
```

---

### 3. Get Agent's Appointments

**Endpoint:** `GET /agents/{agent_id}/appointments`

**Query Parameters:**
```javascript
{
  date: string (optional),     // YYYY-MM-DD
  status: string (optional),   // "booked" | "completed" | "cancelled" | "no_show"
  limit: number (optional),    // default 50, max 100
  offset: number (optional)    // default 0
}
```

**Response (200 OK):**
```javascript
{
  agent_id: string,
  total_count: number,
  appointments: [
    {
      id: string,
      client_id: string,
      client_name: string,
      client_phone: string,
      start_time: string,
      end_time: string,
      service_description: string,
      preparation_instructions: string,
      client_notes: string (nullable),
      agent_notes: string (nullable),
      status: string,
      client_location: {
        latitude: number,
        longitude: number,
        address: string,
        city: string
      },
      has_future_conflict: boolean,
      conflict_acknowledged_by_agent: boolean,
      created_at: string,
      updated_at: string
    }
  ]
}
```

---

### 4. Initialize Travel Time Cache for Day

**Endpoint:** `POST /agents/{agent_id}/travel-cache/initialize`

**Request:**
```javascript
POST /agents/{agent_id}/travel-cache/initialize

Body:
{
  date: "2026-06-15"  // YYYY-MM-DD
}
```

**Response (200 OK):**
```javascript
{
  status: "success",
  agent_id: string,
  date: string,
  total_location_pairs: number,
  total_travel_times_cached: number,
  cache_expires_at: string,
  message: "Cached {total_travel_times_cached} travel time estimates for {date}"
}
```

---

### 5. Update Appointment (Agent)

**Endpoint:** `PATCH /appointments/{appointment_id}`

**Request:**
```javascript
Body:
{
  agent_notes: string (optional),
  actual_completion_time: string (optional),  // ISO 8601
  status: string (optional),  // "completed" | "no_show"
  new_start_time: string (optional)  // ISO 8601, for rescheduling
}
```

**Response (200 OK):**
```javascript
{
  id: string,
  status: string,
  agent_notes: string,
  actual_completion_time: string (nullable),
  updated_at: string
}
```

---

### 6. Cancel Appointment (Agent)

**Endpoint:** `DELETE /appointments/{appointment_id}`

**Request:**
```javascript
Body:
{
  cancellation_reason: string (optional)
}
```

**Response (200 OK):**
```javascript
{
  id: string,
  status: "cancelled",
  cancelled_by: "agent",
  cancelled_at: string,
  message: "Appointment cancelled successfully"
}
```

---

### 7. Acknowledge Conflict Warning

**Endpoint:** `POST /appointments/{appointment_id}/acknowledge-conflict`

**Response (200 OK):**
```javascript
{
  id: string,
  conflict_acknowledged_by_agent: true,
  acknowledged_at: string,
  message: "Conflict acknowledged. Appointment remains booked."
}
```

---

## CLIENT ENDPOINTS

### 8. Book Appointment

**Endpoint:** `POST /appointments`

**Request:**
```javascript
Body:
{
  agent_id: string,
  client_id: string,
  start_time: string,  // ISO 8601 datetime
  
  client_location: {
    latitude: number,
    longitude: number,
    address: string,
    city: string,
    state_province: string,
    postal_code: string,
    country: string,
    source: "manual" | "device_gps"
  },
  
  client_notes: string (optional),
  confirm: boolean  // must be true to proceed
}
```

**Response (201 Created):**
```javascript
{
  id: string,
  agent_id: string,
  client_id: string,
  start_time: string,
  end_time: string,
  service_description: string,
  preparation_instructions: string,
  client_notes: string (nullable),
  client_location: {
    latitude: number,
    longitude: number,
    address: string
  },
  status: "booked",
  created_at: string,
  
  warning: {
    type: "FUTURE_CONFLICT",
    message: string,
    delay_minutes: number
  } (nullable)
}
```

**Error - Slot Not Available (400):**
```javascript
{
  error: "SLOT_NOT_AVAILABLE",
  message: "This slot is not available due to travel time from previous appointment. Please select another time."
}
```

---

### 9. Get Client Profile

**Endpoint:** `GET /clients/{client_id}/profile`

**Response (200 OK):**
```javascript
{
  id: string,
  email: string,
  name: string,
  phone: string,
  company_name: string (nullable),
  location: {
    latitude: number (nullable),
    longitude: number (nullable),
    address: string (nullable),
    city: string (nullable),
    state_province: string (nullable),
    postal_code: string (nullable),
    country: string (nullable),
    source: "manual" | "device_gps" (nullable)
  },
  is_active: boolean,
  created_at: string,
  updated_at: string
}
```

---

### 10. Update Client Profile & Location

**Endpoint:** `PATCH /clients/{client_id}/profile`

**Request:**
```javascript
Body:
{
  name: string (optional),
  phone: string (optional),
  company_name: string (optional),
  location: {
    latitude: number (optional),
    longitude: number (optional),
    address: string (optional),
    city: string (optional),
    state_province: string (optional),
    postal_code: string (optional),
    country: string (optional),
    source: "manual" | "device_gps" (optional)
  }
}
```

**Response (200 OK):**
```javascript
{
  id: string,
  name: string,
  phone: string,
  location: {
    latitude: number,
    longitude: number,
    address: string,
    city: string
  },
  updated_at: string,
  message: "Profile updated successfully"
}
```

---

### 11. Get Client's Appointments

**Endpoint:** `GET /clients/{client_id}/appointments`

**Query Parameters:**
```javascript
{
  status: string (optional),  // "booked" | "completed" | "cancelled"
  limit: number (optional),   // default 50
  offset: number (optional)   // default 0
}
```

**Response (200 OK):**
```javascript
{
  client_id: string,
  total_count: number,
  appointments: [
    {
      id: string,
      agent_id: string,
      agent_name: string,
      agent_phone: string,
      start_time: string,
      end_time: string,
      service_description: string,
      preparation_instructions: string,
      client_notes: string,
      status: string,
      created_at: string
    }
  ]
}
```

---

### 12. Get Agent Details (Client View)

**Endpoint:** `GET /agents/{agent_id}/public-profile`

**Authorization:** Optional for MVP

**Response (200 OK):**
```javascript
{
  id: string,
  name: string,
  bio: string (nullable),
  profile_image_url: string (nullable),
  start_time: string,           // HH:MM
  end_time: string,
  appointment_duration: number,
  timezone: string,
  organization_name: string
}
```

---

## SHARED ENDPOINTS

### 13. Get Appointment Details

**Endpoint:** `GET /appointments/{appointment_id}`

**Response (200 OK):**
```javascript
{
  id: string,
  agent_id: string,
  agent_name: string,
  client_id: string,
  client_name: string,
  client_phone: string,
  start_time: string,
  end_time: string,
  duration_minutes: number,
  service_description: string,
  preparation_instructions: string,
  client_notes: string (nullable),
  agent_notes: string (nullable),
  status: string,
  client_location: {
    latitude: number,
    longitude: number,
    address: string,
    city: string
  },
  has_future_conflict: boolean,
  conflict_acknowledged_by_agent: boolean,
  created_at: string,
  updated_at: string
}
```

---

### 14. Search Agents

**Endpoint:** `GET /agents/search`

**Query Parameters:**
```javascript
{
  organization_id: string,        // UUID
  available_date: string (optional),  // YYYY-MM-DD
  limit: number (optional),       // default 20
  offset: number (optional)       // default 0
}
```

**Response (200 OK):**
```javascript
{
  total_count: number,
  agents: [
    {
      id: string,
      name: string,
      bio: string,
      profile_image_url: string (nullable),
      start_time: string,
      end_time: string,
      appointment_duration: number,
      timezone: string,
      is_active: boolean
    }
  ]
}
```

---

## ADMIN/ORGANIZATION ENDPOINTS

### 15. Register Agent

**Endpoint:** `POST /agents`

**Request:**
```javascript
Body:
{
  email: string,
  name: string,
  phone: string,
  password: string,
  organization_id: string,
  start_time: string,            // HH:MM
  end_time: string,
  appointment_duration: number,  // minutes
  timezone: string              // IANA timezone
}
```

**Response (201 Created):**
```javascript
{
  id: string,
  email: string,
  name: string,
  organization_id: string,
  start_time: string,
  end_time: string,
  appointment_duration: number,
  timezone: string,
  created_at: string,
  message: "Agent registered successfully"
}
```

---

### 16. Register Client

**Endpoint:** `POST /clients`

**Request:**
```javascript
Body:
{
  email: string,
  name: string,
  phone: string (optional),
  password: string (optional),
  organization_id: string,
  company_name: string (optional),
  location: {
    latitude: number (optional),
    longitude: number (optional),
    address: string (optional),
    city: string (optional),
    state_province: string (optional),
    postal_code: string (optional),
    country: string (optional),
    source: "manual" | "device_gps" (optional)
  }
}
```

**Response (201 Created):**
```javascript
{
  id: string,
  email: string,
  name: string,
  organization_id: string,
  location: {
    latitude: number (nullable),
    longitude: number (nullable),
    address: string (nullable)
  },
  created_at: string,
  message: "Client registered successfully"
}
```

---

### 17. Configure API Credentials

**Endpoint:** `POST /organizations/{organization_id}/api-credentials`

**Request:**
```javascript
Body:
{
  service_type: "google_maps",
  api_key: string,
  api_secret: string (optional),
  is_active: boolean (optional, default true)
}
```

**Response (201 Created):**
```javascript
{
  id: string,
  service_type: string,
  is_active: boolean,
  created_at: string,
  message: "API credentials configured successfully"
}
```

**Note:** api_key and api_secret are never returned in responses.

---

### 18. Get Organization Settings

**Endpoint:** `GET /organizations/{organization_id}/settings`

**Response (200 OK):**
```javascript
{
  id: string,
  name: string,
  industry: string (nullable),
  email: string,
  phone: string (nullable),
  address: string,
  city: string,
  state_province: string,
  postal_code: string,
  country: string,
  timezone: string,
  plan: string,  // "free" | "pro" | "enterprise"
  total_agents: number,
  total_clients: number,
  total_appointments_this_month: number,
  api_services_configured: ["google_maps"],
  created_at: string,
  updated_at: string
}
```

---

## AUTH ENDPOINTS

### 19. Login

**Endpoint:** `POST /auth/login`

**Request:**
```javascript
Body:
{
  email: string,
  password: string
}
```

**Response (200 OK):**
```javascript
{
  token: string,              // JWT
  expires_in: number,         // seconds
  user: {
    id: string,
    email: string,
    name: string,
    role: "agent" | "client",
    organization_id: string
  }
}
```

---

### 20. Logout

**Endpoint:** `POST /auth/logout`

**Response (200 OK):**
```javascript
{
  message: "Logged out successfully"
}
```

---

### 21. Refresh Token

**Endpoint:** `POST /auth/refresh`

**Request:**
```javascript
Body:
{
  refresh_token: string
}
```

**Response (200 OK):**
```javascript
{
  token: string,
  expires_in: number
}
```

---

## Error Handling

All error responses follow this format:

```javascript
{
  error: string,              // Machine-readable error code
  message: string,            // Human-readable message
  details: object (optional), // Additional context
  timestamp: string,          // ISO 8601
  path: string               // Request path
}
```

**Common HTTP Status Codes:**
- 400 - BAD_REQUEST
- 401 - UNAUTHORIZED
- 403 - FORBIDDEN
- 404 - NOT_FOUND
- 409 - CONFLICT
- 422 - UNPROCESSABLE_ENTITY
- 500 - INTERNAL_SERVER_ERROR
- 503 - SERVICE_UNAVAILABLE

---

## Rate Limiting

Response headers include:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1623456789  // Unix timestamp
```

---

## Pagination

For list endpoints:

```javascript
{
  data: [...],
  pagination: {
    total_count: number,
    limit: number,
    offset: number,
    has_more: boolean
  }
}
```

---

## Timezone Handling

All timestamps in API are ISO 8601 format (UTC). Clients convert to agent's timezone:

```javascript
{
  agent_timezone: "America/Toronto",
  start_time: "2026-06-15T13:00:00Z",  // UTC
  // Client converts to "2026-06-15T09:00:00" (EDT = UTC-4)
}
```

---

## Full Flow Example: Client Books Appointment

**Step 1: Search for agents**
```
GET /agents/search?organization_id=org-123
```

**Step 2: Get available slots**
```
GET /agents/agent-456/available-slots?date=2026-06-15&client_latitude=43.6532&client_longitude=-79.3832
```

**Step 3: Book appointment**
```
POST /appointments
Body: { agent_id, client_id, start_time, client_location, client_notes, confirm: true }
Response: { id, status: "booked", warning?: {...} }
```

**Step 4: Agent acknowledges conflict (if warning)**
```
POST /appointments/apt-123/acknowledge-conflict
Response: { conflict_acknowledged_by_agent: true }
```

