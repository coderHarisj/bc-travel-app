<!--
 ============================================================================
  Project      : Travel Management — Itinerary Module
  Author       : YsquareTeam
  Organization : Ysquare Technologies Pvt. Ltd
  Owned On     : 2026-02-28
  Last Updated : 2026-02-28
  Version      : 1.0.0
  Copyright © From 2026 YsquareTeam. All rights reserved.
 ============================================================================
-->

# Technical Specification — API & Logic Layer
## Module: Itinerary (`itinenary`)
**Version:** 1.0.0 | **Date:** 2026-02-28 | **Status:** Awaiting Review

---

## 1. Problem Framing

### Restated Problem
Build a backend REST API for a Travel Management Application that enables authorized users (travel agents, coordinators, corporate planners) to create, manage, share, and collaborate on structured travel itineraries. Each itinerary contains ordered days, and each day contains typed events (Flight, Lodging, Activity, Ground Transport, Cruise, Tour, Booking, Info). The system must also support Smart Import (PDF/text → events), cover photo upload, real-time discussion (chat), a resource library, auto-save, preview, and PDF download of itineraries.

### Requirements
- CRUD for itineraries, days, and events
- Typed event creation (8 event types + Smart Import)
- Cover photo upload (PNG/JPEG ≤ 10 MB)
- Day reordering; event drag-and-drop reordering within a day
- Location management and library (hotels, city guides, country info)
- Discussion threads per itinerary
- Auto-save endpoint (invocable every 30 s from client)
- Preview data endpoint and PDF download endpoint
- Flight "Get Info" endpoint (delegates to flight data provider)
- Role-based access control (agents and coordinators write; travelers read)
- Pagination on list endpoints to support 100 k+ itineraries

### Non-Requirements
- Real-time WebSocket for discussion (v1 uses REST polling; chat upgrade deferred)
- AI itinerary generation (future enhancement)
- Calendar sync (future enhancement)
- Booking integrations with external PSPs (future enhancement)
- Offline mode (future enhancement)

### Assumptions
- A `users` table already exists in the auth/user microservice
- A JWT authentication gateway provides `userId` and `role` claims
- File/media storage is handled by a separate File Storage Service (FSS); this module calls FSS via HTTP and stores only the returned URL
- Flight data is fetched from a Flight Data Provider (FDP) via HTTP adapter; this module does not own flight data
- A separate Chat/Notification microservice will handle real-time discussion in v2; v1 stores messages in DB
- Requests originate from web, mobile, and tablet clients

### Non-Goals
- No UI rendering logic
- No payment processing
- No direct DB sharing with other microservices

---

## 2. Architectural Boundaries

```
HTTP Request
    │
    ▼
[Middleware Layer]          — Auth (JWT), rate-limit, Helmet, CORS, request-id injection
    │
    ▼
[Router Layer]              — Binds HTTP method + path + middleware + controller
    │
    ▼
[Controller Layer]          — Parse request, validate (Yup middleware), call service, map DTO
    │
    ▼
[Service Layer]             — Business rules, orchestration, calls repositories + adapters
    │
    ▼
[Repository Layer]          — Prisma queries; only layer touching DB
    │
    ▼
[External Adapter Layer]    — HTTP calls to: FSS, FDP, (future) Chat service
    │
    ▼
[Prisma / MySQL]
```

**Allowed dependencies per layer:**

| Layer | May depend on |
|---|---|
| Router | Middleware, Controller |
| Controller | Service interfaces, Yup DTOs |
| Service | Repository interfaces, Adapter interfaces |
| Repository | Prisma client only |
| External Adapter | HTTP client (axios), external API contracts |

**Forbidden cross-cuts:**
- Controllers MUST NOT import Prisma
- Services MUST NOT import Express types (`Request`, `Response`)
- Repositories MUST NOT contain business decisions
- Business logic MUST NOT appear in routers

---

## 3. Contracts (Interfaces & DTOs)

### 3.1 Domain Interfaces

```typescript
// ports/IItineraryRepository.ts
interface IItineraryRepository {
  createItinerary(payload: CreateItineraryPayload): Promise<ItineraryRecord>;
  findItineraryById(itineraryId: string): Promise<ItineraryRecord | null>;
  findItinerariesByOwner(ownerId: string, pagination: PaginationOptions): Promise<PaginatedResult<ItineraryRecord>>;
  updateItinerary(itineraryId: string, patch: UpdateItineraryPayload): Promise<ItineraryRecord>;
  softDeleteItinerary(itineraryId: string, deletedBy: string): Promise<void>;
}

// ports/IDayRepository.ts
interface IDayRepository {
  addDay(payload: CreateDayPayload): Promise<DayRecord>;
  findDaysByItinerary(itineraryId: string): Promise<DayRecord[]>;
  reorderDays(itineraryId: string, orderedDayIds: string[]): Promise<void>;
  updateDay(dayId: string, patch: UpdateDayPayload): Promise<DayRecord>;
  deleteDay(dayId: string, deletedBy: string): Promise<void>;
}

// ports/IEventRepository.ts
interface IEventRepository {
  addEvent(payload: CreateEventPayload): Promise<EventRecord>;
  findEventsByDay(dayId: string): Promise<EventRecord[]>;
  updateEvent(eventId: string, patch: UpdateEventPayload): Promise<EventRecord>;
  deleteEvent(eventId: string, deletedBy: string): Promise<void>;
  reorderEvents(dayId: string, orderedEventIds: string[]): Promise<void>;
}

// ports/ILocationRepository.ts
interface ILocationRepository {
  addLocation(payload: CreateLocationPayload): Promise<LocationRecord>;
  findLocationsByItinerary(itineraryId: string): Promise<LocationRecord[]>;
  removeLocation(locationId: string, deletedBy: string): Promise<void>;
}

// ports/IDiscussionRepository.ts
interface IDiscussionRepository {
  postMessage(payload: CreateMessagePayload): Promise<MessageRecord>;
  findMessagesByItinerary(itineraryId: string, pagination: PaginationOptions): Promise<PaginatedResult<MessageRecord>>;
}

// ports/IFlightDataAdapter.ts
interface IFlightDataAdapter {
  fetchFlightDetails(airlineCode: string, flightNumber: string): Promise<FlightDetails | null>;
}

// ports/IFileStorageAdapter.ts
interface IFileStorageAdapter {
  uploadFile(fileBuffer: Buffer, mimeType: string, originalName: string): Promise<UploadedFileReference>;
}

// ports/ISmartImportAdapter.ts
interface ISmartImportAdapter {
  parseDocumentToEvents(content: SmartImportContent): Promise<ParsedEventDraft[]>;
}
```

### 3.2 Value Objects / Types

```typescript
type EventType =
  | 'ACTIVITY'
  | 'FLIGHT'
  | 'LODGING'
  | 'GROUND_TRANSPORT'
  | 'CRUISE'
  | 'TOUR'
  | 'BOOKING'
  | 'INFO';

type UserRole = 'AGENT' | 'COORDINATOR' | 'PLANNER' | 'TRAVELER' | 'OPS_STAFF';

interface PaginationOptions {
  page: number;
  pageSize: number;
}

interface PaginatedResult<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
}

interface UploadedFileReference {
  fileUrl: string;
  fileKey: string;
}

interface SmartImportContent {
  sourceType: 'PDF_URL' | 'RAW_TEXT';
  content: string;
}

interface ParsedEventDraft {
  eventType: EventType;
  title: string;
  details: Record<string, unknown>;
}

interface FlightDetails {
  airlineName: string;
  departureAirport: string;
  arrivalAirport: string;
  scheduledDeparture: string;  // ISO-8601 UTC
  scheduledArrival: string;    // ISO-8601 UTC
  duration: number;            // minutes
}
```

### 3.3 Request & Response DTOs (Representative)

```typescript
// dto/CreateItineraryDto.ts
interface CreateItineraryRequestDto {
  title: string;                  // required, min 1 char
  tripStartDate?: string;         // ISO date
  tripEndDate?: string;           // ISO date; must be >= tripStartDate if both provided
}

interface ItineraryResponseDto {
  id: string;
  title: string;
  coverPhotoUrl: string | null;
  coverPhotoText: string | null;
  tripStartDate: string | null;
  tripEndDate: string | null;
  ownerId: string;
  status: 'DRAFT' | 'PUBLISHED';
  createdAt: string;
  updatedAt: string;
}

// dto/CreateDayDto.ts
interface CreateDayRequestDto {
  title: string;
  description: string;
  dayDate?: string;   // ISO date
  location?: string;
}

// dto/CreateEventDto.ts
interface CreateEventRequestDto {
  eventType: EventType;
  title: string;
  sortOrder: number;
  details: ActivityDetails | FlightDetails | LodgingDetails | GroundTransportDetails | CruiseDetails | TourDetails | BookingDetails | InfoDetails;
}

// Typed detail shapes (representative Flight)
interface FlightEventDetails {
  airlineCode: string;
  flightNumber: string;
  departureDatetime: string;   // ISO-8601 UTC
  arrivalDatetime: string;     // ISO-8601 UTC
  departureAirport?: string;
  arrivalAirport?: string;
  price?: number;
  currency?: string;
}

// Standard API response envelope
interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    requestId: string;
  };
}
```

---

## 4. Domain Logic

### 4.1 Itinerary Status Machine
```
DRAFT → PUBLISHED → ARCHIVED (soft-delete)
```
- New itineraries are created in `DRAFT` status.
- Publishing is a deliberate action (validated: must have title + at least one day with at least one event).
- Archived itineraries are soft-deleted (`deleted_at IS NOT NULL`).

### 4.2 Day Sequencing Rule
- Days carry a `day_number` (1-based) and a `sort_order` (integer).
- When a user reorders days, `sort_order` values are recalculated for all days of the itinerary.
- `day_number` is recalculated server-side in `sort_order` ascending sequence after every reorder.
- **No client-supplied `day_number` is trusted.**

### 4.3 Event Sort Order Rule
- Events within a day carry a `sort_order` integer.
- Gaps are allowed. Client sends an ordered array of event IDs; the service assigns sort_order = index * 10 for sparse gaps.

### 4.4 Cover Photo Validation Rule (Domain, not Infrastructure)
```
ALLOWED MIME TYPES: image/png, image/jpeg
MAX FILE SIZE: 10,485,760 bytes (10 MB)
```
Violation → `ItineraryDomainError(INVALID_COVER_PHOTO, ...)`

### 4.5 Date Consistency Rule
- If both `tripStartDate` and `tripEndDate` are provided, `tripEndDate >= tripStartDate` is mandatory.
- Days added manually outside this range are permitted in DRAFT (flexibility for agents).

### 4.6 Smart Import Orchestration
```
1. Validate input (sourceType + content present)
2. Call ISmartImportAdapter.parseDocumentToEvents()
3. If adapter throws ParseFailureError, propagate as ItineraryDomainError(SMART_IMPORT_PARSE_FAILURE)
4. For each ParsedEventDraft: call IEventRepository.addEvent()
5. Return list of created EventRecords
```

### 4.7 Auto-Save Rule
- Auto-save is a `PATCH /api/v1/itineraries/:id` with only changed fields.
- Idempotent: if nothing changed, returns 200 with current state without a DB write.
- The service compares incoming patch fields against current values before persisting.

### 4.8 Access Control Rule (Pure Domain)
```
WRITE (create/update/delete): AGENT | COORDINATOR | PLANNER
READ:  all roles including TRAVELER and OPS_STAFF
OWN ACCESS: only owner may update/delete own itinerary
ADMIN OVERRIDE: future (not in v1 scope)
```

---

## 5. Application Use Cases (Service Layer)

### `ItineraryService` — use cases

| Method | Responsibility |
|---|---|
| `createItinerary(requestorId, dto)` | Creates draft, auto-generates days from date range if provided |
| `getItineraryById(requestorId, itineraryId)` | Fetches with access check |
| `listItineraries(requestorId, pagination)` | Returns paginated list scoped to owner |
| `updateItinerary(requestorId, itineraryId, patch)` | Patches allowed fields; enforces ownership |
| `deleteItinerary(requestorId, itineraryId)` | Soft deletes; enforces ownership |
| `uploadCoverPhoto(requestorId, itineraryId, fileBuffer, mimeType, filename)` | Validates file, calls FSS, stores URL |
| `publishItinerary(requestorId, itineraryId)` | Validates publish preconditions, transitions status |

### `DayService` — use cases

| Method | Responsibility |
|---|---|
| `addDay(requestorId, itineraryId, dto)` | Creates next sequential day |
| `getDaysForItinerary(requestorId, itineraryId)` | Returns ordered day list |
| `reorderDays(requestorId, itineraryId, orderedDayIds)` | Validates all IDs belong to itinerary, recalculates sort_order and day_number |
| `updateDayDetails(requestorId, dayId, patch)` | Updates day metadata |
| `removeDay(requestorId, dayId)` | Soft deletes day and its events |

### `EventService` — use cases

| Method | Responsibility |
|---|---|
| `addEvent(requestorId, dayId, dto)` | Validates event type, persists event with typed details |
| `getEventsByDay(requestorId, dayId)` | Returns ordered events |
| `updateEvent(requestorId, eventId, patch)` | Updates event fields |
| `removeEvent(requestorId, eventId)` | Soft deletes event |
| `reorderEvents(requestorId, dayId, orderedEventIds)` | Recalculates sort_order |
| `triggerSmartImport(requestorId, dayId, content)` | Orchestrates smart import flow |
| `fetchAndPopulateFlightDetails(requestorId, eventId, airlineCode, flightNumber)` | Calls FDP adapter, patches event |

### `LocationService` & `DiscussionService`
- `LocationService.addLocation / listLocations / removeLocation`
- `DiscussionService.postMessage / listMessages(paginated)`

---

## 6. Infrastructure Adapters

### 6.1 Module Folder Structure
```
src/modules/itinerary/
├── routes/
│   ├── itinerary.routes.ts
│   ├── day.routes.ts
│   ├── event.routes.ts
│   ├── location.routes.ts
│   ├── discussion.routes.ts
│   └── smartimport.routes.ts
├── controllers/
│   ├── itinerary.controller.ts
│   ├── day.controller.ts
│   ├── event.controller.ts
│   ├── location.controller.ts
│   ├── discussion.controller.ts
│   └── smartimport.controller.ts
├── services/
│   ├── itinerary.service.ts
│   ├── day.service.ts
│   ├── event.service.ts
│   ├── location.service.ts
│   └── discussion.service.ts
├── repositories/
│   ├── itinerary.repository.ts
│   ├── day.repository.ts
│   ├── event.repository.ts
│   ├── location.repository.ts
│   └── discussion.repository.ts
├── adapters/
│   ├── flight-data.adapter.ts
│   ├── file-storage.adapter.ts
│   └── smart-import.adapter.ts
├── dto/
│   ├── itinerary.dto.ts
│   ├── day.dto.ts
│   ├── event.dto.ts
│   ├── location.dto.ts
│   └── discussion.dto.ts
├── validators/
│   ├── itinerary.validator.ts
│   ├── day.validator.ts
│   ├── event.validator.ts
│   └── smartimport.validator.ts
├── errors/
│   └── itinerary.errors.ts
└── index.ts
```

### 6.2 REST API Contracts

**Base path:** `/api/v1`

#### Itineraries

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/itineraries` | Required | List itineraries (paginated, scoped to caller) |
| POST | `/itineraries` | Required | Create itinerary |
| GET | `/itineraries/:itineraryId` | Required | Get itinerary by ID |
| PATCH | `/itineraries/:itineraryId` | Required | Update / auto-save itinerary |
| DELETE | `/itineraries/:itineraryId` | Required | Soft-delete itinerary |
| POST | `/itineraries/:itineraryId/cover-photo` | Required | Upload cover photo (multipart/form-data) |
| POST | `/itineraries/:itineraryId/publish` | Required | Publish itinerary |

#### Days

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/itineraries/:itineraryId/days` | Required | Add day |
| GET | `/itineraries/:itineraryId/days` | Required | Get all days |
| PATCH | `/itineraries/:itineraryId/days/:dayId` | Required | Update day metadata |
| DELETE | `/itineraries/:itineraryId/days/:dayId` | Required | Remove day |
| PUT | `/itineraries/:itineraryId/days/reorder` | Required | Reorder days |

#### Events

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/itineraries/:itineraryId/days/:dayId/events` | Required | Add event |
| GET | `/itineraries/:itineraryId/days/:dayId/events` | Required | Get events for day |
| PATCH | `/itineraries/:itineraryId/days/:dayId/events/:eventId` | Required | Update event |
| DELETE | `/itineraries/:itineraryId/days/:dayId/events/:eventId` | Required | Remove event |
| PUT | `/itineraries/:itineraryId/days/:dayId/events/reorder` | Required | Reorder events |
| POST | `/itineraries/:itineraryId/days/:dayId/smart-import` | Required | Smart Import |
| POST | `/itineraries/:itineraryId/days/:dayId/events/:eventId/flight-info` | Required | Fetch & populate flight info |

#### Locations

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/itineraries/:itineraryId/locations` | Required | Add location |
| GET | `/itineraries/:itineraryId/locations` | Required | List locations |
| DELETE | `/itineraries/:itineraryId/locations/:locationId` | Required | Remove location |

#### Discussion

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/itineraries/:itineraryId/discussions` | Required | Post message |
| GET | `/itineraries/:itineraryId/discussions` | Required | List messages (paginated) |

#### Export

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/itineraries/:itineraryId/preview` | Required | Get preview data |
| GET | `/itineraries/:itineraryId/download` | Required | Download as PDF (delegates to PDF adapter) |

---

### 6.3 HTTP Status Code Contracts

| Scenario | Status |
|---|---|
| Successful GET / PATCH | 200 |
| Successful POST (creation) | 201 |
| Successful DELETE | 200 (body: `{ success: true }`) |
| Validation failure | 400 |
| Unauthenticated | 401 |
| Forbidden (wrong role or ownership) | 403 |
| Resource not found | 404 |
| External adapter failure (FDP, FSS) | 502 |
| Internal error | 500 |

---

### 6.4 Request Validation Schemas (Yup — Representative)

```typescript
// validators/itinerary.validator.ts
import * as yup from 'yup';

export const createItinerarySchema = yup.object({
  title: yup.string().trim().min(1).max(255).required(),
  tripStartDate: yup.string().matches(/^\d{4}-\d{2}-\d{2}$/).optional(),
  tripEndDate: yup
    .string()
    .matches(/^\d{4}-\d{2}-\d{2}$/)
    .optional()
    .when('tripStartDate', (tripStartDate, schema) =>
      tripStartDate
        ? schema.test(
            'end-after-start',
            'tripEndDate must be >= tripStartDate',
            (tripEndDate) => !tripEndDate || tripEndDate >= (tripStartDate as unknown as string)
          )
        : schema
    ),
});

// validators/event.validator.ts — flight event
export const flightEventDetailsSchema = yup.object({
  airlineCode: yup.string().uppercase().length(2).required(),
  flightNumber: yup.string().matches(/^[A-Z]{2}\d{1,4}$/).required(),
  departureDatetime: yup.string().required(),
  arrivalDatetime: yup.string().required(),
});
```

---

### 6.5 Error Catalogue

```typescript
// errors/itinerary.errors.ts
export type ItineraryErrorCode =
  | 'ITINERARY_NOT_FOUND'
  | 'DAY_NOT_FOUND'
  | 'EVENT_NOT_FOUND'
  | 'LOCATION_NOT_FOUND'
  | 'FORBIDDEN_ACCESS'
  | 'INVALID_COVER_PHOTO'
  | 'INVALID_DATE_RANGE'
  | 'INVALID_EVENT_TYPE'
  | 'SMART_IMPORT_PARSE_FAILURE'
  | 'FLIGHT_DATA_PROVIDER_UNAVAILABLE'
  | 'FILE_STORAGE_SERVICE_UNAVAILABLE'
  | 'PUBLISH_PRECONDITION_FAILED'
  | 'REORDER_IDS_MISMATCH';
```

Error response shape (envelope):
```json
{
  "success": false,
  "error": {
    "code": "ITINERARY_NOT_FOUND",
    "message": "Itinerary with ID [abc123] does not exist or has been deleted.",
    "requestId": "req_abc123"
  }
}
```

---

## 7. Failure-First Design

| Dependency | Failure Mode | Detection Signal | Mitigation | Residual Risk |
|---|---|---|---|---|
| MySQL / Prisma | Connection drop | Prisma `P1008`, `P1001` errors | Connection pool retry (3 attempts, exponential back-off); circuit-breaker pattern at adapter | Data loss if all retries fail — alert triggered |
| File Storage Service (FSS) | HTTP 5xx / timeout | Axios error `ECONNREFUSED`, HTTP ≥ 500 | Retry once with 1 s delay; surface `FILE_STORAGE_SERVICE_UNAVAILABLE` with 502 | Cover photo not uploaded; itinerary remains without photo |
| Flight Data Provider (FDP) | API rate limit, timeout, 5xx | HTTP ≥ 500, `ECONNRESET` | Return partial event without pre-filled flight data; UI allows manual override | User must manually fill flight details |
| Smart Import Adapter | Parse failure | Adapter throws `ParseFailureError` | Surface `SMART_IMPORT_PARSE_FAILURE` with actionable message (provide manual entry fallback) | Itinerary not auto-populated; no data loss |
| JWT Auth Gateway | Token expiry / invalid signature | 401 from middleware | Client must refresh token; no retry at API level | Request rejected; no resource leak |

---

## 8. Observability Contract

### Log Format
All logs MUST follow:  
`ITINERARY_<SERVICE>_<ACTION> - {entityId} : {valueIfApplicable}`

**Examples:**
```
ITINERARY_SERVICE_CREATING_ITINERARY - usr_9182 : {title: "Paris Trip 2026"}
ITINERARY_SERVICE_CREATED_ITINERARY - itn_7731 : {ownerId: "usr_9182"}
ITINERARY_SERVICE_COVER_PHOTO_UPLOAD_FAILED - itn_7731 : {reason: "FILE_TOO_LARGE", sizeBytes: 15728640}
ITINERARY_DAY_SERVICE_ADDING_DAY - itn_7731 : {dayNumber: 3}
ITINERARY_EVENT_SERVICE_SMART_IMPORT_FAILED - day_4421 : {reason: "ParseFailureError"}
ITINERARY_FLIGHT_ADAPTER_FETCH_FAILED - evt_8811 : {airline: "AI", flight: "AI202"}
ITINERARY_DISCUSSION_SERVICE_MSG_POSTED - itn_7731 : {userId: "usr_9182"}
```

Every log entry must carry:
- `requestId` (from `x-request-id` header, injected by middleware)
- `userId` (from JWT claims)
- UTC timestamp

### Metrics (to be collected at infrastructure layer)
- `itinerary_create_total` — counter
- `itinerary_event_create_total{type}` — counter by event type
- `smart_import_success_rate` — gauge
- `flight_data_adapter_latency_ms` — histogram
- `cover_photo_upload_latency_ms` — histogram
- `itinerary_api_error_total{code}` — counter by error code

### Alerts
- Smart import success rate < 80% for 15-min window → PagerDuty alert
- Cover photo upload 502 rate > 5% → PagerDuty alert
- `itinerary_create_total` drops to 0 for 10 min during business hours → PagerDuty alert

---

## 9. Economic & Scale Analysis

### Cost per Request (v1 baseline — self-hosted MySQL + Node.js)
| Operation | Estimated Latency | DB Queries | External Calls |
|---|---|---|---|
| Create Itinerary | < 50 ms | 1 INSERT | 0 |
| Add Event | < 60 ms | 1 INSERT | 0 |
| Fetch Itinerary (with days + events) | < 100 ms | 3 SELECTs | 0 |
| Flight Get Info | < 500 ms | 1 INSERT/UPDATE | 1 (FDP) |
| Cover Photo Upload | < 800 ms | 1 UPDATE | 1 (FSS) |
| Smart Import | < 2000 ms | N INSERTs | 1 (AI/Parser adapter) |

### Bottlenecks at 10× (10 k concurrent users)
- MySQL becomes the primary bottleneck on concurrent itinerary + event inserts
- **Mitigation:** Read replicas for GET endpoints; connection pooling via `pgBouncer` equivalent
- File uploads bypass main service; FSS handles concurrency independently

### Bottlenecks at 100× (100 k concurrent users)
- Horizontal scaling of Node.js service (stateless — no session state)
- MySQL sharding by `owner_id` (hash-based); consider read replica per shard
- Smart Import heavy lifting offloaded to async job queue (Bull + Redis) — single synchronous call is not viable at 100× without queueing

### Cheapest valid alternative considered
- Used a self-hosted NLP parser over a paid SaaS AI for Smart Import → reduces per-import cost to $0 vs $0.01–0.05/call at 100× scale = $4k–20k/month savings at 100k imports/day

---

## 10. Self-Audit

| Rule | Status | Notes |
|---|---|---|
| Layer separation enforced | ✅ | No Prisma in controllers/services |
| No business logic in routes | ✅ | Routes are wiring only |
| Yup validation at boundary | ✅ | Middleware validates before controller |
| ORM entities do not escape service | ✅ | DTOs mapped at controller boundary |
| Controllers have no try/catch | ✅ | Global error middleware handles all errors |
| No free-text logging | ✅ | Structured format enforced |
| SOLID + DI enforced | ✅ | Constructor injection for all services/repos |
| Error messages state what/why/action | ✅ | Error catalogue with human messages |
| Feature deletable without side effects | ✅ | Module is isolated; no shared state |
| Failure paths designed | ✅ | Section 7 covers all external deps |
| All public APIs pass self-descriptiveness test | ✅ | No abbreviations; intent clear from names |

**Trade-offs made:**
1. Discussion v1 uses REST + DB (not WebSocket) — avoids Chat microservice dependency; deferred to v2
2. Smart Import is synchronous in v1 — simpler implementation; async job queue designed for v2 migration path
3. PDF generation delegates to a PDF adapter (not inline) — keeps service boundary clean; swap-able with any PDF library or external service

**Intentional Deviations:** NONE

---

*All public APIs pass the self-descriptiveness test.*
