# Phase 2

In phase 2 participants take on a new set of features. Test cases are separate from test cases for phase 1.

#### F6 – Search API & search engine integration (Phase 2)

**Problem / value**  
Seekers often know part of a facility name or county but not enough to construct precise filters. They need **free-text
search** that respects multiple terms and the current filters.

**Core behaviour**

- A search box allows users to type queries such as `hospital saint MATERNITY`.
- The backend implements search functionality to:
    - Tokenize the query into terms.
    - Return facilities where **every query term** appears somewhere in the indexed fields.
      - Full implementation should index **all text fields** official_name, county_name, facility_type_name, owner_type_name, operation_status_name, service names, service categories, etc.
    - Sort results by match quality (exact > prefix > fuzzy > substring).
    - Support **case-insensitive** search.
    - Support **prefix matching** for partial terms (e.g., "nai" finds "Nairobi").
    - Support **edit distance / typo tolerance** for search terms (max edit distance 2).
    - Support **substring/infix matching** using n-gram indexing (e.g., "spital" finds "hospital").
- The frontend displays search results as a list similar to F1.
- Search can be combined with the same filters used in Phase 1 (administrative unit, services, "available today" toggles,
  etc.), so that text search respects both metadata filters and current facility state.
- Introduce **ranking/pagination** over text search results.

**Search matching modes**

The search library supports three matching modes that work together:

1. **Exact matching** (working): Query "ICU services" matches "ICU services". Requires minimum 3 characters.
2. **Prefix matching** (working): Query "nai" matches "nairobi", "naivasha", etc. Requires minimum 3 characters.
3. **Fuzzy/typo matching** (buggy): Query "hosptial" should match "hospital" using Levenshtein edit distance. Currently broken due to a bug in the starter code.
4. **Substring matching** (not implemented): Query "spital" should match "hospital" using n-gram indexing. Teams must implement this.

**Relevance ranking**

Results are ranked by match quality using simple scoring:
- **Exact match**: score 1.0 (highest)
- **Prefix match**: score 0.8
- **Fuzzy match**: score 0.6
- **Substring match**: score 0.4 (lowest)

When a term matches via multiple methods, the highest score is used.

**User flows**

- A seeker types one or more words into a search box (e.g., "Nairobi maternity").
- **Search is instant**: results update automatically as the user types (debounced, no submit button required).
- The results view shows facilities whose fields contain all terms, ranked by match quality.
- The seeker optionally narrows further using filters or location search.
- Typos are tolerated (e.g., "hosptial" still finds "hospital").
- Partial words are tolerated (e.g., "spital" finds "hospital").

**Frontend requirements**

- **Instant search with debouncing**: The search input should trigger API calls automatically as the user types, with a debounce delay.
- **Combine with filters**: Search queries should be sent together with any currently selected filters (county, service, geo, etc.) in a single API call.
- **Loading state**: Show a loading indicator while search results are being fetched (optional?)
- **Clear feedback**: Display the number of results found and make it clear when results are filtered by search vs. other filters.

**Acceptance criteria (high level)**

- Multi-term search where all terms must match some part of the facility record.
- Results are ranked by match quality.
- **Prefix matching** works for terms ≥3 characters.
- **Typo tolerance** works for common misspellings (edit distance ≤2).
- **Substring matching** works for infix queries using n-gram index.
- Search respects and combines with filtering and location features.
- Pagination works correctly with search results.
---

#### F7 - Refactoring Challenge: Per-Service Queue Management

In Phase 2, teams must refactor their Phase 1 patient counter implementation to support per-service queue tracking. This refactoring challenge tests the ability to evolve existing architecture under new requirements.

**What you built in Phase 1**:
- Single facility-wide `current_patient_count` integer
- Employees manually increment/decrement via simple operations
- Counter resets at end of day

**What you must refactor to in Phase 2**:

Phase 1 counter is replaced with numbered ticket-based per-service queue management:

**Core behavior**:
- Each service (Laboratory, Dental, Maternity, etc.) maintains a separate queue with numbered tickets
- When employee adds a patient to a service queue:
  - System assigns the next sequential ticket number (e.g., 1, 2, 3...)
  - Returns the ticket number to the employee
  - Patient receives this number
- When employee reports a patient as served:
  - Employee provides the specific ticket number that was served
  - System removes that ticket from the queue
- Tickets that have clearly been skipped cannot wait indefinitely:
  - For each service queue, the system tracks the highest ticket number that has been served.
  - If a ticket with a lower number is still in the queue and more than 30 minutes have passed since that ticket was issued, it is automatically removed as "expired".
- The facility-wide `current_patient_count` becomes a **computed aggregate**:
  - Read-only value = SUM(count of active tickets across all service queues)
  - Cannot be edited directly
  - Must be recalculated when any service queue changes
- Each facility has a **maximum patient limit** with a **default value of 50**:
  - The system automatically calculates the **busy level** based on the current patient count relative to the limit:
    - **quiet**: current count is below 50% of the limit
    - **normal**: current count is between 50% and 80% of the limit
    - **busy**: current count is above 80% of the limit
  - Employees can update the limit via a dedicated endpoint
  - The limit **cannot be deleted** - only updated to a new value or **reset to default** (50)
  - The busy level is computed automatically and cannot be set directly by employees in Phase 2
  - The maximum patient limit can be updated independently from service availability
- Queue state retrieval:
  - Must return current active ticket numbers per service
  - Must show which tickets are still waiting
  - Teams can easily mess up gap handling and ticket ordering here
- Expired tickets (skipped and older than 30 minutes) must not appear in active tickets.
- Auto-clear: All service queues and ticket counters reset to 0 at end of day (midnight local time)
- Must handle race conditions:
  - Multiple employees adding patients to different services simultaneously
  - Multiple employees reporting served patients from same service simultaneously
  - Ticket number assignment must be atomic and unique
- regular users can see busy level and current patient count in search results and facility details

**New Phase 2 endpoints required**:

```
POST /api/facilities/{facility_id}/queues/add
Body: { "service_name": "Laboratory" }
→ Adds patient to service queue, returns ticket number
Response: { "ticket_number": 42, "service_name": "Laboratory" }

POST /api/facilities/{facility_id}/queues/serve
Body: { "service_name": "Laboratory", "ticket_number": 42 }
→ Reports patient as served

GET /api/facilities/{facility_id}/queues/list?service_name=Laboratory
→ Returns current queue state for the specified service with active ticket numbers
Response: {
  "service_name": "Laboratory",
  "active_tickets": [38, 39, 41, 42],
  "patient_count": 4
}

GET /api/facilities/{facility_id}/state
→ REFACTORED: now includes computed total from queue tickets
```

**Migration requirements**:
- Phase 1 simple counter data can be discarded (fresh start in Phase 2)
- Phase 1 patient count endpoints (`POST /api/facilities/{id}/patients/increment` and `POST /api/facilities/{id}/patients/decrement`) must be replaced/deprecated
- Frontend must be refactored to use new queue endpoints and show per-service ticket numbers

**Testing focus**:
- Race conditions with concurrent ticket assignment (must be unique and sequential)
- Race conditions with concurrent "serve" operations on same service
- Correct gap handling when patient N+1 served before patient N
- Time-based expiration of skipped tickets that have been waiting more than 30 minutes
- Correct aggregation of total patients from multiple service queues under concurrent load
- End-of-day reset across all services and ticket counters
- Queue state consistency when multiple employees query simultaneously during updates

**Common failure modes (where teams will mess up)**:
- Non-atomic ticket number assignment (duplicate tickets)
- Incorrect gap detection (missing patients who left)
- Race conditions in aggregate calculation
- Stale queue state when tickets are being added/removed
- Off-by-one errors in ticket sequencing
- Missing transaction boundaries causing inconsistent state

---

#### F8 – Innovative feature for local users (Phase 2)

**Problem / value**
The core Phase 1 features provide essential facility discovery and status functionality, but there are many opportunities to make this app more valuable and relevant to local Kenyan users' daily lives, health-seeking behaviors, and community context.

**Core behaviour**

- Teams **propose and implement one meaningfully innovative feature** that addresses a specific need or pain point for local people in Kenya using health facilities.
- The feature should leverage the existing data (facility information, location, services, current status) but extend the app's utility beyond basic search and filtering.
- **IMPORTANT**: Attach a short document (less than 1000 words) describing your innovative feature at the root of your repository in a file called INNOVATIVE_FEATURE.md. This document will be used by Jury members to review your project

## Phase 2 Endpoints

**F6 – Search API & search engine integration**
- `GET /api/facilities-filter` – EXTENDED: add optional `q` query parameter for free-text search.
  - Results ranked by relevance.
  - Respects all filters from F1/F2.

**F7 – Per-service queue management**

New queue endpoints:
- `POST /api/facilities/{facility_id}/queues/add` – add patient to service queue, returns ticket number.
  - Request body: `{ "service_name": "Laboratory" }`
  - Response: `{ "ticket_number": 42, "service_name": "Laboratory" }`
- `POST /api/facilities/{facility_id}/queues/serve` – report patient as served.
  - Request body: `{ "service_name": "Laboratory", "ticket_number": 42 }`
- `GET /api/facilities/{facility_id}/queues/list` – get current queue state for the specified service.
  - Query param: `?service_name=Laboratory`
  - Response: `{ "service_name": "Laboratory", "active_tickets": [38, 39, 41, 42], "patient_count": 4 }`

Modified endpoints:
- `POST /api/facilities/{facility_id}/state` – REFACTORED: employees can update service availability only.
  - Request body: `{ "services": [{"service_name": "Laboratory", "available_today": true}] }`
- `POST /api/facilities/{facility_id}/max-patient-limit` – set or update the maximum patient limit for a facility.
  - Request body: `{ "max_patient_limit": 100 }` or `{ "reset_to_default": true }`
  - Default limit is **50**. Cannot be deleted, only updated or reset to default.
  - Response includes: updated facility state with the new max_patient_limit and computed busy_level.
  - Busy level is automatically computed based on current patient count vs limit
- `GET /api/facilities/{facility_id}/state` – REFACTORED: now returns computed total from queue tickets instead of simple counter.
  - Response includes: service availability array, computed `current_patient_count` (sum of all queue counts), optional `busy_level` (only present if max_patient_limit is set), and `max_patient_limit` if configured.

These mappings are the starting point for the API contract and integration tests; details such as status codes, error shapes, and field-level schemas will be refined when writing the OpenAPI spec and contract test suite.


