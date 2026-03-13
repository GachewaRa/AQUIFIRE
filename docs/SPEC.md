# AQUIFIRE — Project Specification

**Version:** 0.1 (MVP)
**Date:** 2026-03-13
**Status:** Draft

---

## 1. Overview

AQUIFIRE is a borehole records management system for drilling companies. It digitises the process of logging borehole data — from initial drilling through to water quality analysis — replacing paper-based records and fragmented files.

The MVP targets a single drilling company used internally by their staff. A multi-tenant SaaS upgrade path is noted in the Future Features section.

---

## 2. Goals

- Provide a central, searchable record of every borehole the company has drilled
- Capture all technically relevant data at each stage of a borehole's lifecycle
- Enable generation of borehole completion reports
- Visualise borehole locations on a map
- Be simple enough that field staff with basic digital literacy can use it

---

## 3. Tech Stack

| Layer | Technology |
|---|---|
| Framework | SvelteKit (full-stack — handles both frontend and backend) |
| Database | PostgreSQL (hosted on company VPS) |
| ORM | Drizzle ORM |
| Connection pooling | PgBouncer (on VPS, in front of PostgreSQL) |
| Auth | better-auth |
| Maps | Leaflet.js with OpenStreetMap tiles |
| PDF generation | pdfmake |
| Deployment | Vercel |
| Styling | TailwindCSS |

### Architecture notes

- All database queries run in SvelteKit server files (`+page.server.ts`, `+server.ts`) — never in the browser
- Vercel runs SvelteKit server code as serverless functions; PgBouncer on the VPS manages connection pooling to avoid exhausting PostgreSQL's connection limit
- The VPS PostgreSQL port must be reachable from Vercel (either open with SSL + strong credentials, or IP-allowlisted)
- Environment variables (DB connection string, auth secrets) are stored in Vercel project settings

---

## 4. User Roles

For the MVP, three roles are sufficient:

| Role | Capabilities |
|---|---|
| `admin` | Full access: manage users, clients, projects, boreholes, settings |
| `field_officer` | Create and edit borehole records, add logs and test data |
| `viewer` | Read-only access to all records (useful for management/clients) |

---

## 5. Data Models

### 5.1 User

Managed by better-auth. Extended with a `role` field.

| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| email | text | Unique |
| name | text | Display name |
| role | enum | `admin`, `field_officer`, `viewer` |
| created_at | timestamp | |

---

### 5.2 Client

The organisation or individual who commissioned the borehole.

| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| name | text | Company or person name |
| contact_person | text | Nullable |
| phone | text | Nullable |
| email | text | Nullable |
| address | text | Nullable |
| notes | text | Nullable |
| created_at | timestamp | |
| updated_at | timestamp | |

---

### 5.3 Project

A project groups one or more boreholes under a single commission. A client can have many projects.

| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| client_id | uuid | FK → Client |
| name | text | |
| description | text | Nullable |
| location_name | text | General area/region name |
| start_date | date | Nullable |
| end_date | date | Nullable |
| status | enum | `active`, `completed`, `on_hold` |
| created_by | uuid | FK → User |
| created_at | timestamp | |
| updated_at | timestamp | |

---

### 5.4 Borehole

The central entity. One project can have many boreholes.

**Identity & Location**

| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| project_id | uuid | FK → Project |
| reference_number | text | Unique identifier used in the field (e.g. `BH-2024-031`) |
| site_name | text | Human-readable location name |
| latitude | numeric(10,7) | GPS coordinate |
| longitude | numeric(10,7) | GPS coordinate |
| county | text | Nullable |
| district | text | Nullable |

**Drilling Details**

| Field | Type | Notes |
|---|---|---|
| drilling_start_date | date | Nullable |
| drilling_end_date | date | Nullable |
| drilling_method | enum | `rotary`, `percussion`, `down_the_hole_hammer`, `cable_tool`, `other` |
| rig_used | text | Rig name/ID, nullable |
| driller_name | text | Lead driller, nullable |
| planned_depth | numeric(8,2) | Metres, nullable |
| achieved_depth | numeric(8,2) | Metres |

**Completion & Casing**

| Field | Type | Notes |
|---|---|---|
| completion_status | enum | `drilling_only`, `cased`, `developed`, `pump_tested`, `commissioned` |
| casing_type | enum | `pvc`, `steel`, `none`, nullable |
| casing_diameter_mm | integer | Nullable |
| casing_depth_m | numeric(8,2) | Nullable |
| gravel_pack | boolean | Was gravel pack installed? |
| cement_grouted | boolean | Was surface sealed with cement? |

**Water Level**

| Field | Type | Notes |
|---|---|---|
| static_water_level_m | numeric(8,2) | Depth to water at rest, nullable |
| swl_measured_date | date | Nullable |

**Meta**

| Field | Type | Notes |
|---|---|---|
| notes | text | General notes, nullable |
| created_by | uuid | FK → User |
| created_at | timestamp | |
| updated_at | timestamp | |

---

### 5.5 LithologyLayer

The geological log — what material was encountered at each depth interval. Multiple layers per borehole, ordered by depth.

| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| borehole_id | uuid | FK → Borehole |
| depth_from_m | numeric(8,2) | Top of layer |
| depth_to_m | numeric(8,2) | Bottom of layer |
| material | text | e.g. "Red clay", "Weathered granite", "Fractured quartzite" |
| color | text | Nullable |
| hardness | enum | `soft`, `medium`, `hard`, `very_hard`, nullable |
| is_water_bearing | boolean | Did this layer yield water? |
| notes | text | Nullable |

---

### 5.6 WaterStrike

Specific depth(s) at which water was encountered during drilling. A borehole can have multiple water strikes.

| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| borehole_id | uuid | FK → Borehole |
| depth_m | numeric(8,2) | |
| estimated_yield_lps | numeric(6,3) | Litres per second, nullable |
| notes | text | Nullable |

---

### 5.7 PumpTest

Yield testing carried out after drilling. A borehole can have multiple pump tests (e.g. step test followed by constant rate test).

| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| borehole_id | uuid | FK → Borehole |
| test_date | date | |
| test_type | enum | `step_drawdown`, `constant_rate`, `recovery` |
| duration_hours | numeric(5,2) | |
| pump_depth_m | numeric(8,2) | Nullable |
| pump_type | text | Nullable |
| pumping_rate_lps | numeric(6,3) | Litres per second |
| drawdown_m | numeric(8,2) | Depth to water level during pumping |
| rest_water_level_m | numeric(8,2) | Water level before test, nullable |
| recommended_yield_lps | numeric(6,3) | Safe sustainable yield, nullable |
| conducted_by | text | Person/firm who conducted the test |
| notes | text | Nullable |

---

### 5.8 WaterQualityAnalysis

Lab results from a water sample. A borehole can have multiple analyses over time.

**Meta**

| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| borehole_id | uuid | FK → Borehole |
| sample_date | date | When sample was collected |
| lab_name | text | Nullable |
| analysis_date | date | When results were issued, nullable |
| report_reference | text | Lab report number, nullable |

**Physical Parameters**

| Field | Type | Notes |
|---|---|---|
| ph | numeric(4,2) | Nullable |
| turbidity_ntu | numeric(8,3) | Nullable |
| conductivity_us_cm | numeric(8,2) | μS/cm, nullable |
| tds_mg_l | numeric(8,2) | Total dissolved solids, nullable |
| temperature_c | numeric(5,2) | Nullable |
| color_hazen | numeric(6,1) | Nullable |

**Chemical Parameters**

| Field | Type | Notes |
|---|---|---|
| hardness_mg_l | numeric(8,2) | Nullable |
| alkalinity_mg_l | numeric(8,2) | Nullable |
| chloride_mg_l | numeric(8,2) | Nullable |
| sulfate_mg_l | numeric(8,2) | Nullable |
| nitrate_mg_l | numeric(8,2) | Nullable |
| nitrite_mg_l | numeric(8,2) | Nullable |
| fluoride_mg_l | numeric(8,2) | Nullable |
| iron_mg_l | numeric(8,2) | Nullable |
| manganese_mg_l | numeric(8,2) | Nullable |
| arsenic_ug_l | numeric(8,3) | Micrograms/litre, nullable |

**Microbiological**

| Field | Type | Notes |
|---|---|---|
| total_coliforms_cfu | integer | CFU/100ml, nullable |
| e_coli_cfu | integer | CFU/100ml, nullable |

**Summary**

| Field | Type | Notes |
|---|---|---|
| overall_status | enum | `acceptable`, `requires_treatment`, `unsafe`, nullable |
| notes | text | Nullable |

---

### 5.9 BoreholeAttachment

Photos or documents (lab reports, site surveys) attached to a borehole.

| Field | Type | Notes |
|---|---|---|
| id | uuid | PK |
| borehole_id | uuid | FK → Borehole |
| file_url | text | Storage URL |
| file_name | text | Original filename |
| file_type | text | MIME type |
| caption | text | Nullable |
| uploaded_by | uuid | FK → User |
| uploaded_at | timestamp | |

---

## 6. Application Routes

```
/                          → Dashboard
/login                     → Login page

/map                       → Map view of all boreholes

/clients                   → Client list
/clients/new               → Create client
/clients/[id]              → Client detail (projects + boreholes)
/clients/[id]/edit         → Edit client

/projects                  → Project list
/projects/new              → Create project
/projects/[id]             → Project detail (borehole list)
/projects/[id]/edit        → Edit project

/boreholes                 → Borehole list (searchable, filterable)
/boreholes/new             → Create borehole
/boreholes/[id]            → Borehole detail (overview + all sub-records)
/boreholes/[id]/edit       → Edit borehole core details
/boreholes/[id]/lithology  → Manage lithology log
/boreholes/[id]/pump-test  → Manage pump tests
/boreholes/[id]/water-quality → Manage water quality analyses
/boreholes/[id]/report     → View and export PDF completion report

/guide                     → User guide
/settings                  → App settings (admin only)
/settings/users            → User management (admin only)
```

---

## 7. Key Pages — Detail

### 7.1 Dashboard (`/`)

- Summary cards: total boreholes, boreholes this month, boreholes by completion status
- Recent activity feed (last 10 borehole records updated)
- Quick-access link to create a new borehole
- Small embedded map preview showing recent borehole locations

### 7.2 Borehole Detail (`/boreholes/[id]`)

A single-page view with tabbed sections:
- **Overview** — location, drilling summary, completion status, static water level
- **Lithology** — layered depth chart of geological materials
- **Water Strikes** — list of strike depths and estimated yields
- **Pump Test** — test results and recommended yield
- **Water Quality** — latest analysis with colour-coded parameter indicators
- **Attachments** — photo gallery and document list
- **Report** — button to generate PDF

### 7.3 Map (`/map`)

- Full-screen interactive map (Leaflet + OpenStreetMap)
- All boreholes plotted as markers
- Marker colour reflects completion status
- Clicking a marker shows a popup with key info and a link to the full borehole record
- Filter sidebar: by project, by status, by date range, by yield range

### 7.4 Borehole Report (`/boreholes/[id]/report`)

Generates a printable/downloadable PDF containing:
- Borehole reference and location details
- Drilling summary (dates, method, depths)
- Lithology log (tabular)
- Water strike summary
- Pump test results
- Water quality results (if available)
- Completion status
- Company branding (name, logo — configurable in settings)

### 7.5 User Guide (`/guide`)

A structured in-app guide that introduces users to the platform. Written in plain language, targeted at field officers with basic digital literacy. Sections:

1. **Getting started** — logging in, navigating the app
2. **Creating a borehole record** — step-by-step walkthrough
3. **Recording the geological log** — how to enter lithology layers
4. **Logging water strikes** — when and how to record them
5. **Entering pump test results** — what each field means
6. **Adding water quality data** — understanding the parameters
7. **Uploading photos and documents** — file attachment workflow
8. **Generating a completion report** — exporting to PDF
9. **Using the map** — filtering and navigating borehole locations
10. **Managing clients and projects** — for admin users
11. **Managing users** — for admin users only

The guide is rendered from markdown content stored in the codebase. This makes it easy to update without touching database records.

---

## 8. API Structure

SvelteKit server routes handle all data operations. No separate API server.

### Form Actions (SvelteKit `+page.server.ts`)
Used for create/update/delete operations submitted via HTML forms with progressive enhancement:

- `POST /clients` — create client
- `POST /clients/[id]` — update client
- `POST /projects` — create project
- `POST /boreholes` — create borehole
- `POST /boreholes/[id]` — update borehole
- `POST /boreholes/[id]/lithology` — add/update/delete lithology layer
- `POST /boreholes/[id]/pump-test` — add/update pump test
- `POST /boreholes/[id]/water-quality` — add/update water quality analysis
- `POST /boreholes/[id]/attachments` — upload file

### API Endpoints (`+server.ts`)
Used for client-side interactions (map data, search):

- `GET /api/boreholes/geo` — returns GeoJSON of all boreholes for map rendering
- `GET /api/boreholes/search?q=` — typeahead search
- `GET /api/boreholes/[id]/report` — streams PDF file

---

## 9. Auth

Handled by **better-auth**.

- Email + password login (no social auth for MVP)
- Session-based (cookies), managed server-side
- Route protection via SvelteKit `hooks.server.ts`
- Role-based access control applied at the server load/action level
- Password reset flow via email (SMTP configurable in settings)
- Admin can create user accounts; no self-registration (internal app)

---

## 10. File Storage

For the MVP, attachments are stored on the same VPS via a simple file storage directory served over HTTPS, or alternatively on a Vercel-compatible object store (e.g. Cloudflare R2 or AWS S3). The storage backend is abstracted behind a single `storage` module so it can be swapped without changing application code.

---

## 11. Environment Variables

```
DATABASE_URL           # PostgreSQL connection string (via PgBouncer)
BETTER_AUTH_SECRET     # Auth signing secret
STORAGE_BUCKET         # File storage bucket name
STORAGE_ENDPOINT       # Storage service endpoint
STORAGE_ACCESS_KEY     # Storage credentials
STORAGE_SECRET_KEY     #
SMTP_HOST              # Email (for password reset)
SMTP_PORT              #
SMTP_USER              #
SMTP_PASS              #
PUBLIC_APP_NAME        # Displayed in UI ("AQUIFIRE" or company name)
```

---

## 12. Implementation Phases

### Phase 1 — Foundation
- Project scaffold (SvelteKit + Drizzle + TailwindCSS)
- Database schema creation
- Auth setup (better-auth, login/logout, route protection)
- User management (admin creates accounts)

### Phase 2 — Core Records
- Client CRUD
- Project CRUD
- Borehole CRUD (core fields)
- Borehole list and detail pages

### Phase 3 — Technical Logs
- Lithology log entry UI (table with add/edit/delete rows)
- Water strike logging
- Pump test entry
- Water quality analysis entry

### Phase 4 — Map & Search
- Map view with Leaflet
- Borehole search and filtering

### Phase 5 — Reports & Attachments
- PDF report generation
- File upload and attachment gallery

### Phase 6 — User Guide & Polish
- User guide page (markdown-rendered)
- Dashboard metrics
- UI polish, responsive design for tablet/mobile

---

## 13. Future Features (Post-MVP Backlog)

These are explicitly out of scope for the MVP but should be kept in mind during implementation so that the architecture does not block them.

| Feature | Notes |
|---|---|
| Multi-tenant SaaS | Each company has isolated data; billing, subdomain routing |
| Regulatory report formats | Submission-ready formats for water boards or geological surveys |
| Offline-first / mobile sync | PWA with local storage and sync for field use without internet |
| Borehole monitoring | Periodic water level and yield measurements over time |
| Equipment & fleet management | Track drilling rigs: maintenance schedules, deployment history |
| Crew/HR logging | Assign crew to jobs, log hours |
| Cost & invoicing | Track drilling costs per borehole, generate client invoices |
| Groundwater trend analysis | Charts showing yield or water level trends across a region |
| Import/export | Bulk import from CSV/Excel; export to standard hydrogeology formats |
| API for third-party integrations | Expose data to GIS platforms or government databases |
| Automated water quality flagging | Alert when parameters exceed WHO or local regulatory limits |
| Role expansion | More granular permissions (e.g. per-project access) |
