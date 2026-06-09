---
title: Location Aware Scheduling - Database Structure
created_at: '2026-06-08T19:33:51Z'
updated_at: '2026-06-08T19:35:27Z'
tags:
- database
- schema
- sql
- tables
- postgres
summary: Complete SQL database schema including 6 tables (Organizations, Agents, Clients,
  Appointments, Travel Time Cache, API Credentials) with indexes, constraints, entity
  relationships, and critical query patterns.
---

## Location Aware Scheduling - Database Structure

### Complete Database Schema with Tables and Indexes

---

## Table 1: Organizations

```sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  industry VARCHAR(100),
  
  email VARCHAR(255) NOT NULL,
  phone VARCHAR(20),
  
  address VARCHAR(500),
  city VARCHAR(100),
  state_province VARCHAR(100),
  postal_code VARCHAR(20),
  country VARCHAR(100),
  
  timezone VARCHAR(50) DEFAULT 'UTC',
  
  plan VARCHAR(20) DEFAULT 'free' CHECK (plan IN ('free', 'pro', 'enterprise')),
  
  is_deleted BOOLEAN DEFAULT false,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_organizations_name ON organizations(name);
CREATE INDEX idx_organizations_is_deleted ON organizations(is_deleted);
```

---

## Table 2: Agents

```sql
CREATE TABLE agents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  phone VARCHAR(20),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  
  start_time TIME NOT NULL,
  end_time TIME NOT NULL,
  appointment_duration INTEGER NOT NULL CHECK (appointment_duration > 0),
  
  password_hash VARCHAR(255) NOT NULL,
  is_active BOOLEAN DEFAULT true,
  is_deleted BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  timezone VARCHAR(50) DEFAULT 'UTC',
  profile_image_url VARCHAR(500),
  bio TEXT,
  
  CONSTRAINT valid_times CHECK (end_time > start_time)
);

CREATE INDEX idx_agents_organization_id ON agents(organization_id);
CREATE INDEX idx_agents_email ON agents(email);
CREATE INDEX idx_agents_is_deleted ON agents(is_deleted);
```

**Purpose:** Stores field service agent profiles and scheduling configuration. Each agent has their own start/end times and configurable appointment duration.

---

## Table 3: Clients

```sql
CREATE TABLE clients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  phone VARCHAR(20),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  
  company_name VARCHAR(255),
  
  -- Single Location (denormalized)
  location_latitude DECIMAL(10, 8),
  location_longitude DECIMAL(11, 8),
  location_address VARCHAR(500),
  location_city VARCHAR(100),
  location_state_province VARCHAR(100),
  location_postal_code VARCHAR(20),
  location_country VARCHAR(100),
  location_source VARCHAR(20) CHECK (location_source IN ('manual', 'device_gps')),
  
  password_hash VARCHAR(255),
  is_active BOOLEAN DEFAULT true,
  is_deleted BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  profile_image_url VARCHAR(500)
);

CREATE INDEX idx_clients_organization_id ON clients(organization_id);
CREATE INDEX idx_clients_email ON clients(email);
CREATE INDEX idx_clients_is_deleted ON clients(is_deleted);
```

**Purpose:** Stores client profiles. Each client has a single location (latitude/longitude coordinates), which can be entered manually or obtained via device GPS.

---

## Table 4: Appointments

```sql
CREATE TABLE appointments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  agent_id UUID NOT NULL REFERENCES agents(id),
  client_id UUID NOT NULL REFERENCES clients(id),
  
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP NOT NULL,
  
  service_description VARCHAR(500) NOT NULL,
  preparation_instructions TEXT,
  
  client_notes TEXT,
  agent_notes TEXT,
  
  status VARCHAR(20) NOT NULL DEFAULT 'booked' 
    CHECK (status IN ('booked', 'cancelled', 'completed', 'no_show')),
  
  cancelled_by VARCHAR(20) CHECK (cancelled_by IN ('agent', 'system')),
  cancellation_reason TEXT,
  cancelled_at TIMESTAMP,
  
  actual_completion_time TIMESTAMP,
  
  has_future_conflict BOOLEAN DEFAULT false,
  conflict_acknowledged_by_agent BOOLEAN DEFAULT false,
  
  is_deleted BOOLEAN DEFAULT false,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  CONSTRAINT valid_times CHECK (end_time > start_time),
  CONSTRAINT valid_cancellation CHECK (
    (status = 'cancelled' AND cancelled_at IS NOT NULL) OR 
    (status != 'cancelled' AND cancelled_at IS NULL)
  )
);

CREATE INDEX idx_appointments_agent_id ON appointments(agent_id);
CREATE INDEX idx_appointments_client_id ON appointments(client_id);
CREATE INDEX idx_appointments_organization_id ON appointments(organization_id);
CREATE INDEX idx_appointments_status ON appointments(status);
CREATE INDEX idx_appointments_start_time ON appointments(start_time);
CREATE INDEX idx_appointments_agent_date ON appointments(agent_id, start_time);
CREATE INDEX idx_appointments_is_deleted ON appointments(is_deleted);
```

**Purpose:** Stores appointment bookings. Includes service descriptions, preparation instructions visible to clients, notes from both client and agent, status tracking, and conflict flags.

---

## Table 5: Travel Time Cache

```sql
CREATE TABLE travel_time_cache (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
  
  origin_client_id UUID NOT NULL REFERENCES clients(id),
  destination_client_id UUID NOT NULL REFERENCES clients(id),
  
  travel_time_minutes INTEGER NOT NULL CHECK (travel_time_minutes >= 0),
  route_distance_km DECIMAL(10, 2),
  
  cached_date DATE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP NOT NULL,
  
  CONSTRAINT different_clients CHECK (origin_client_id != destination_client_id)
);

CREATE INDEX idx_travel_time_cache_agent_date ON travel_time_cache(agent_id, cached_date);
CREATE INDEX idx_travel_time_cache_client_pair ON travel_time_cache(origin_client_id, destination_client_id);
CREATE INDEX idx_travel_time_cache_expires_at ON travel_time_cache(expires_at);
```

**Purpose:** Caches Google Maps travel time estimates between client locations for a specific agent on a specific date. Loaded once per day at session start to avoid repeated API calls.

---

## Table 6: API Credentials

```sql
CREATE TABLE api_credentials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  
  service_type VARCHAR(50) NOT NULL,
  
  api_key VARCHAR(500) NOT NULL,     -- encrypted in application
  api_secret VARCHAR(500),           -- encrypted in application
  
  is_active BOOLEAN DEFAULT true,
  rate_limit_per_minute INTEGER,
  
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_by UUID,
  
  is_deleted BOOLEAN DEFAULT false,
  
  CONSTRAINT unique_active_service_per_org UNIQUE (organization_id, service_type) 
    WHERE is_active = true AND is_deleted = false
);

CREATE INDEX idx_api_credentials_organization_id ON api_credentials(organization_id);
CREATE INDEX idx_api_credentials_service_type ON api_credentials(service_type);
```

**Purpose:** Stores encrypted API credentials for third-party services (Google Maps, Twilio, etc.). Only one active credential per service per organization.

---

## Entity Relationship Diagram

```
Organizations (1:N)
  ├── Agents (soft delete: is_deleted)
  ├── Clients (soft delete: is_deleted)
  ├── Appointments (soft delete: is_deleted)
  └── API_Credentials (soft delete: is_deleted)

Agents (1:N) Appointments
Clients (1:N) Appointments

Agents (1:N) TravelTimeCache
Clients (1:N) TravelTimeCache
  - as origin_client_id
  - as destination_client_id
```

---

## Key Database Design Decisions

✅ **Single Client Location:** Denormalized in clients table (no separate table)  
✅ **Soft Deletes:** All main tables include is_deleted flag  
✅ **Travel Time Cache:** Keyed by agent + client pair + date  
✅ **Appointment Duration:** Stored in agents table (fixed per agent)  
✅ **Location Coordinates:** Stored as DECIMAL(10,8) and DECIMAL(11,8) for latitude/longitude  
✅ **Timestamps:** All in UTC, converted to agent's timezone at application layer  
✅ **Constraints:** CHECK constraints for valid times, status values, and business logic  
✅ **Indexes:** Optimized for common queries (agent_date lookups, status filtering)  

---

## Critical Queries

```sql
-- Get all appointments for agent on specific date (active only)
SELECT * FROM appointments
WHERE agent_id = $1
  AND DATE(start_time) = $2
  AND status != 'cancelled'
  AND is_deleted = false
ORDER BY start_time ASC;

-- Find most recent appointment before a time
SELECT * FROM appointments
WHERE agent_id = $1
  AND start_time < $2
  AND status != 'cancelled'
  AND is_deleted = false
ORDER BY start_time DESC
LIMIT 1;

-- Find next appointment after a time (for conflict detection)
SELECT * FROM appointments
WHERE agent_id = $1
  AND start_time > $2
  AND status != 'cancelled'
  AND is_deleted = false
ORDER BY start_time ASC
LIMIT 1;

-- Get travel time from cache
SELECT travel_time_minutes FROM travel_time_cache
WHERE agent_id = $1
  AND origin_client_id = $2
  AND destination_client_id = $3
  AND cached_date = $4
  AND expires_at > NOW();

-- Find unique client pairs for cache initialization
SELECT DISTINCT 
  a1.client_id as origin_client_id,
  a2.client_id as destination_client_id
FROM appointments a1
JOIN appointments a2 
  ON a1.agent_id = a2.agent_id 
  AND a2.start_time > a1.start_time
  AND DATE(a1.start_time) = DATE(a2.start_time)
WHERE a1.agent_id = $1
  AND DATE(a1.start_time) = $2
  AND a1.status != 'cancelled'
  AND a2.status != 'cancelled'
  AND a1.is_deleted = false
  AND a2.is_deleted = false
ORDER BY a1.start_time ASC;

-- Get API credentials for organization
SELECT * FROM api_credentials
WHERE organization_id = $1
  AND service_type = $2
  AND is_active = true
  AND is_deleted = false
LIMIT 1;
```

---

## Constraints & Validations

- **Agent:** start_time < end_time, appointment_duration > 0
- **Appointment:** end_time > start_time, valid status values
- **Client Location:** latitude -90 to 90, longitude -180 to 180
- **Travel Time Cache:** origin ≠ destination, travel_time >= 0
- **API Credentials:** Only one active service per org per type
- **Soft Deletes:** All queries must filter is_deleted = false

