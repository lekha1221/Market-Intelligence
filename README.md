# RE·ANALYZE: Market Intelligence Dashboard
## Technical Reference (Current Production)

**Product:** RE·ANALYZE by ASBL  
**Purpose:** Real estate market intelligence for Hyderabad residential supply, infrastructure, pricing, regulation, and buyer insights  
**Audience:** Developers onboarding to the codebase; business stakeholders understanding how the live system works

---

## What This System Does

RE·ANALYZE is a web application that lets authenticated users explore Hyderabad’s residential real estate market. After signing in, a user selects a city and locality (or defines a geographic radius) and loads live data from MongoDB. The dashboard renders KPIs, charts, maps, and tables across ten analysis panels.

The application is intentionally small: **seven files** form the complete user-facing product. There is no separate frontend build step, no React application, and no microservice layer. A Flask server serves HTML pages and JSON APIs; a single data module queries MongoDB; one HTML file contains the entire dashboard UI.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Browser                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ home.html    │  │ login.html   │  │ index.html (dashboard)│ │
│  │ signup.html  │  │ forgot_pwd   │  │ ECharts + Leaflet     │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │
└─────────┼─────────────────┼─────────────────────┼─────────────┘
          │                 │                     │ fetch()
          └─────────────────┴─────────────────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │     app.py      │
                   │  Flask + Auth   │
                   │  REST API       │
                   └────────┬────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │ mongo_supply.py │
                   │ Query + Normalize│
                   └────────┬────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │    MongoDB      │
                   │ real_estate     │
                   │ insightforge    │
                   │ buyer_persona   │
                   └─────────────────┘
```

**Request flow for a typical analysis:**

1. User authenticates via `login.html` → `POST /api/auth/login` → session cookie set.
2. User opens `/dashboard` → `app.py` serves `index.html`.
3. User clicks **Analyze** → `POST /api/supply` → `mongo_supply.fetch_supply()` → JSON returned.
4. `index.html` renders KPIs, charts, maps, and tables from that JSON.
5. Additional panels call focused APIs (`/api/pricing-intel`, `/api/buyer-persona`, etc.) when opened or when data is needed.

---

## The Seven Core Files

These are the only files that constitute the Market Intelligence Dashboard application.

---

### 1. `app.py`

| | |
|---|---|
| **Purpose** | HTTP server: authentication, page routing, and REST API for the dashboard |
| **Responsibilities** | Serve HTML entry points; enforce login on `/dashboard` and all `/api/*` routes; translate HTTP requests into `mongo_supply.py` function calls; assemble and return JSON responses; generate print-ready HTML reports; persist user accounts and issue reports |

**Inputs**
- HTTP requests from the browser (JSON bodies, query parameters, Flask session cookies)
- Function return values from `mongo_supply.py`

**Outputs**
- HTML pages (via templates or `index.html`)
- JSON API responses
- Session state (`session["user"]`)

**Interactions**

| Interacts with | How |
|---|---|
| `mongo_supply.py` | Imports and calls all data functions (`fetch_supply`, `get_infra_summary`, `get_pricing_intel`, etc.) |
| `index.html` | Served at `GET /dashboard` via `send_from_directory` |
| `templates/*.html` | Rendered for `/`, `/login`, `/signup`, `/forgot-password` |
| MongoDB | Direct writes to `user_authentication` and `report_issue` collections for auth and feedback |

**Role in application flow**  
`app.py` is the single entry point for every browser and API interaction. It contains no business logic for supply normalization or infrastructure distance calculation—that lives in `mongo_supply.py`. Its job is routing, authentication, response assembly, and a small amount of inline report HTML generation (`_render_report_html`).

**API endpoints consumed by the dashboard**

| Endpoint | Method | Used for |
|---|---|---|
| `/api/auth/login` | POST | Sign in |
| `/api/auth/logout` | POST | Sign out |
| `/api/auth/register` | POST | Create account |
| `/api/auth/forgot-password` | POST | Password reset token |
| `/api/auth/reset-password` | POST | Complete password reset |
| `/api/supply` | POST | Primary data load (supply + infra + regulatory) |
| `/api/supply-radius` | POST | Radius-based project query |
| `/api/infra` | POST | Reload infrastructure at a new radius |
| `/api/pricing-intel` | POST | ASBL vs market pricing |
| `/api/project-intel` | GET | Developer rankings and classification |
| `/api/buyer-persona` | GET | Buyer persona and survey data |
| `/api/locality-intel` | GET | 99acres + Google review locality data |
| `/api/news` | GET | Google News RSS feed |
| `/api/localities-by-city` | GET | Locality dropdown population |
| `/api/bp-localities` | GET | Buyer persona locality list |
| `/api/localities` | GET | Recent analyses list |
| `/api/height-restrictions` | GET | AAI airport height zones |
| `/api/report-html` | POST | PDF-ready report export |
| `/api/report-issue` | POST | User-submitted data corrections |
| `/api/health` | GET | Connection status indicator |

**Run command:** `python app.py` → serves at `http://localhost:5000`

---

### 2. `mongo_supply.py`

| | |
|---|---|
| **Purpose** | Unified data access and normalization layer for all MongoDB queries |
| **Responsibilities** | Connect to MongoDB; canonicalize locality names; normalize raw project documents into a consistent frontend shape; compute market summaries; query infrastructure POIs within a radius; join RERA absorption data; serve pricing, persona, regulatory, and locality intelligence data |

**Inputs**
- Locality name, city name, latitude/longitude, radius (km), page number
- Raw MongoDB documents from `projects_master` and related collections

**Outputs**
- Python dictionaries consumed by `app.py` and serialized to JSON:
  - `supply_projects` — list of normalized project records
  - `supply_summary` — aggregated KPIs and distributions
  - `infra` — nearest verified POI per category
  - `pois` — full POI list for map rendering
  - `regulatory` — HMDA and Fire NOC summaries
  - Pricing, persona, and locality intelligence objects

**Interactions**

| Interacts with | How |
|---|---|
| `app.py` | Called exclusively by Flask route handlers; never imported by the frontend |
| MongoDB `real_estate` | `projects_master`, `99a_locality_report`, `google_reviews` |
| MongoDB `insightforge` | POI collections, RERA scraped data, surveys, HMDA/Fire NOC records, approval matches |
| MongoDB `buyer_persona` | `localities` collection |

**Role in application flow**  
Every number, chart, and table in the dashboard ultimately originates from a function in this file. The most critical path is:

```
fetch_supply(locality, city)
  → query projects_master
  → prefetch_rera_absorption_cache()
  → _norm_master() per document
  → _compute_summary()
  → return { supply_projects, supply_summary, meta }
```

**Key functions**

| Function | What it does |
|---|---|
| `fetch_supply()` | Main supply query; supports locality filter or city-wide Hyderabad view with pagination cache |
| `fetch_supply_by_radius()` | Geospatial project filter by lat/lng + radius |
| `_norm_master()` | Maps a `projects_master` document to the frontend project shape; validates BHK configurations |
| `_compute_summary()` | Builds KPIs: totals, segment/status/price/BHK distributions, absorption, developer counts |
| `canonicalize_locality()` | Resolves locality name variants via `LOCALITY_ALIASES` (400+ mappings) |
| `get_infra_summary()` | Nearest verified POI per category within radius |
| `get_nearby_pois()` | Full POI list for infrastructure map layers |
| `get_regulatory_summary()` | HMDA permits and Fire NOC via mandal/village mapping |
| `get_pricing_intel()` | ASBL vs market PSF comparison |
| `get_project_intelligence()` | Developer market share and project classification |
| `get_buyer_persona_data()` | Buyer persona tier, unit mix, survey aggregates |
| `get_locality_intelligence()` | 99acres ratings and Google review summaries |
| `prefetch_rera_absorption_cache()` | Batch RERA lookup to avoid per-project query overhead |

**Infrastructure defaults**
- `LOCALITY_INFRA_RADIUS_KM = 1.0` — default search radius
- Proposed or unverified POIs (e.g. proposed metro stations) are filtered out before results are returned
- BHK values are validated: unit-number noise like `101.3 BHK` is rejected; `configurations.bhk_list` in the database is authoritative

---

### 3. `index.html`

| | |
|---|---|
| **Purpose** | The complete dashboard application — UI, styling, client logic, charts, and maps |
| **Responsibilities** | Render all ten analysis panels; call Flask APIs; manage view state (locality vs radius mode, filters, theme); render ECharts visualizations and Leaflet maps; handle user interactions (search, sort, filter, export, detail panel) |

**Inputs**
- JSON responses from Flask API endpoints
- User interactions: locality selection, radius settings, panel navigation, table filters

**Outputs**
- Rendered DOM: KPI cards, charts, maps, tables, slide-out project detail panel
- CSV and JSON file downloads (client-side)
- Print-ready report window (from `/api/report-html`)

**Interactions**

| Interacts with | How |
|---|---|
| `app.py` | All data via `fetch()` to `/api/*` endpoints |
| ECharts (CDN) | `echarts.init()` on chart container divs |
| Leaflet (CDN) | Three maps: `map-main`, `map-infra`, `map-coverage` |
| `login.html` | Redirects to `/login` on 401 responses |

**Role in application flow**  
This file is both the UI and the client application. There is no separate JavaScript bundle. Global state is held in module-level variables:

| Variable | Holds |
|---|---|
| `_data` | Last full API response from `/api/supply` or `/api/supply-radius` |
| `_projects` | Current project list (may be filtered client-side) |
| `_msState` | Multi-select filter state: status, segment, BHK |
| `_viewMode` | `'locality'` or `'radius'` |
| `_infraRadius` | Infrastructure search radius (1, 3, 5, or 10 km) |
| `_activeRadius` | Supply search radius in radius view mode |
| `_charts` | ECharts instance registry |

**Key functions**

| Function | What it does |
|---|---|
| `loadData()` | Primary entry: calls `/api/supply` or delegates to `loadDataByRadius()` |
| `loadDataByRadius()` | Calls `/api/supply-radius` for geographic radius queries |
| `render(data)` | Distributes API response across all panels |
| `switchPanel(name)` | Shows one of ten panels; lazy-loads panel-specific data |
| `filterProjects()` | Client-side table filtering by status, segment, BHK, search text |
| `renderInfraTab()` | Infrastructure cards, POI map, radar chart |
| `reloadInfra()` | Calls `/api/infra` when user changes infra radius |
| `loadPricingIntel()` | Calls `/api/pricing-intel` |
| `loadBuyerPersonaDB()` | Calls `/api/buyer-persona` |
| `loadLocalityIntel()` | Calls `/api/locality-intel` |
| `loadLiveNews()` | Calls `/api/news` |
| `generateReport()` | Calls `/api/report-html` and opens print dialog |
| `renderCompareTable()` | Locality comparison (up to 3 localities) on Coverage Map |

**Dashboard panels**

| Panel ID | Name | Primary data source |
|---|---|---|
| `panel-overview` | Overview | `/api/supply` → `supply_summary`, `supply_projects`, `infra` |
| `panel-supply` | Supply Intel | `/api/supply` → `supply_projects` |
| `panel-project-intel` | Project Intel | `/api/project-intel` |
| `panel-pricing` | Pricing Intel | `/api/pricing-intel` |
| `panel-infra` | Infrastructure | `/api/supply` or `/api/infra` → `infra`, `pois` |
| `panel-regulatory` | Regulatory | `/api/supply` → `regulatory`, `supply_summary`; `/api/height-restrictions` |
| `panel-news` | Market News | `/api/news` |
| `panel-persona` | Buyer Persona | `/api/buyer-persona` |
| `panel-loc-intel` | Locality Intel | `/api/locality-intel` |
| `panel-coverage` | Coverage Map | `/api/bp-localities`, `/api/localities-by-city`, buyer persona tiers |
| `panel-reports` | Reports & Export | `/api/report-html`, `/api/localities` |

**View modes**
- **Locality View** — filter projects by locality name; infra centroid from project coordinates
- **Radius View** — filter projects within N km of a map center; configurable 1–20 km

**Libraries loaded from CDN**
- Leaflet 1.9.4 — maps
- ECharts 5.4.3 — all charts (SVG renderer)
- Google Fonts — Inter (UI), JetBrains Mono (numbers)

---

### 4. `templates/home.html`

| | |
|---|---|
| **Purpose** | Public landing page at `/` |
| **Responsibilities** | Introduce RE·ANALYZE; provide navigation to sign in or sign up |
| **Inputs** | None (static HTML) |
| **Outputs** | Rendered landing page |
| **Interactions** | Links to `/login` and `/signup`; served by `app.py` route `homepage()` |
| **Role in flow** | First page a new visitor sees; not part of the analysis workflow |

---

### 5. `templates/login.html`

| | |
|---|---|
| **Purpose** | Authentication page at `/login` |
| **Responsibilities** | Collect email and password; call `POST /api/auth/login`; redirect to `/dashboard` on success |
| **Inputs** | User email and password (form fields) |
| **Outputs** | Session cookie (set by Flask on successful login); redirect to dashboard |
| **Interactions** | `app.py` `auth_login()` validates credentials against `real_estate.user_authentication`; links to `/signup` and `/forgot-password` |
| **Role in flow** | Gateway to the dashboard; unauthenticated users hitting `/dashboard` are redirected here |

---

### 6. `templates/signup.html`

| | |
|---|---|
| **Purpose** | Account registration page at `/signup` |
| **Responsibilities** | Collect name, email, and password; call `POST /api/auth/register`; redirect to dashboard on success |
| **Inputs** | User registration form data |
| **Outputs** | New document in `user_authentication` collection; session cookie; redirect to `/dashboard` |
| **Interactions** | `app.py` `auth_register()` hashes password with Werkzeug and inserts user record |
| **Role in flow** | Creates new user accounts before first dashboard access |

---

### 7. `templates/forgot_password.html`

| | |
|---|---|
| **Purpose** | Password recovery page at `/forgot-password` |
| **Responsibilities** | Collect email; call `POST /api/auth/forgot-password` to issue a reset token; allow password reset via `POST /api/auth/reset-password` |
| **Inputs** | User email; reset token and new password (on reset step) |
| **Outputs** | Reset token stored on user document; confirmation message |
| **Interactions** | `app.py` forgot-password and reset-password handlers |
| **Role in flow** | Allows existing users to recover access without admin intervention |

---

## Data Sources (MongoDB)

All live dashboard data comes from MongoDB. The dashboard does not scrape property portals at query time.

### `real_estate` database

| Collection | Used for |
|---|---|
| `projects_master` | All supply data: projects, prices, BHK, units, coordinates, RERA, segment, status |
| `99a_locality_report` | Locality Intelligence: ratings, likes, dislikes, price trends |
| `google_reviews` | Locality Intelligence: project review scores by neighborhood |
| `user_authentication` | Login and registration |
| `report_issue` | User-submitted data correction reports |

### `insightforge` database

| Collection | Used for |
|---|---|
| `rera_scraped_data` | Absorption rate (booked vs total units) |
| `metro_stations`, `hospitals`, `schools`, `malls`, `it_companies` | Infrastructure cards and map |
| `universities`, `parks`, `bus_stops`, `industries`, `banks`, `lakes` | Extended POI data and radar chart |
| `points_of_interest` | General POI proximity queries |
| `hmda_all_records` | HMDA permit data (via mandal/village join) |
| `fire_noc_r4` | Fire NOC records |
| `approval_project_matches` | Project-level approval stages |
| `customer_lifestyle_survey` | Buyer Persona survey insights |
| `customer_property_survey` | Buyer property preference surveys |
| `airport_height_restriction_zones` | AAI height restriction data |

### `buyer_persona` database

| Collection | Used for |
|---|---|
| `localities` | Market tier, RERA unit mix, persona bootstrap per locality |

### External source (at request time)

| Source | Used for |
|---|---|
| Google News RSS | Market News panel (`/api/news`) |

---

## Dashboard Features (Current)

### Authentication
- Email/password login, registration, and password reset
- Session-based access control on dashboard and all APIs

### Overview
- AI-generated market summary narrative from loaded supply data
- KPI cards: total projects, units, PSF, gated communities, absorption, developers, RERA/HMDA/GHMC counts
- Charts: segment split, construction status, price buckets, BHK configuration
- Geo Intelligence Map with Supply, Price, and Infra overlay modes
- Top developers list
- Locality View and Radius View with configurable km radius

### Supply Intelligence
- Paginated project table with sort and multi-select filters (status, segment, BHK)
- Inventory and absorption mini-charts
- Project detail slide-out panel
- CSV export

### Project Intelligence
- Developer leaderboard and market share chart
- RERA approval status chart
- Project classification chart (Gated / Villa / Standalone / Plotted)
- Litigation and risk flags

### Pricing Intelligence
- ASBL vs market average PSF with premium/discount percentage
- Competitor PSF comparison chart
- PSF by segment chart
- PSF by BHK configuration table

### Infrastructure
- Category cards: Metro, Hospital, School, Mall, IT Company, University
- Selectable infra radius: 1, 3, 5, or 10 km
- POI map with toggleable category layers
- Infrastructure radar chart and connectivity score
- Nearest amenities lists; lakes and buffer zone warnings
- Verified infrastructure only (proposed facilities excluded)

### Regulatory
- GHMC / HMDA authority distribution
- RERA registration and approval counts
- AAI airport height restriction zones
- Litigation flags from project data

### Market News
- Live feed from Google News RSS with locality-specific queries
- Category filters

### Buyer Persona
- Persona profile cards with budget, dominant BHK, employer context
- Survey insights from lifestyle and property surveys
- BHK demand charts and supply gap analysis
- Configurable persona radius aggregation

### Locality Intelligence
- 99acres locality ratings, likes, dislikes, and price trends
- Google review aggregation for projects in the locality
- Resident review excerpts

### Coverage Map
- City-wide locality markers with tier coloring from buyer persona data
- Locality comparison (select up to 3 localities, side-by-side metrics)
- View modes: projects, price tier, buyer persona tier

### Reports & Export
- Print-ready HTML report (browser Print → Save as PDF)
- CSV export of project table
- JSON export of full analysis data
- Recent localities list for quick reload

### Global UI
- Light/dark theme toggle (persisted in `localStorage`)
- Sidebar locality search with fuzzy filter
- API health status indicator
- Toast notifications
- Data issue reporting form

---

## End-to-End Workflow

### Standard locality analysis

```
1. User visits / → home.html
2. User signs in → login.html → POST /api/auth/login → session created
3. User redirected to /dashboard → index.html loaded
4. index.html calls GET /api/localities-by-city and GET /api/bp-localities
5. User selects city "Hyderabad" and locality "Gachibowli"
6. User clicks Analyze → POST /api/supply
7. app.py calls mongo_supply.fetch_supply("Gachibowli", "Hyderabad")
8. mongo_supply queries projects_master, normalizes documents, computes summary
9. app.py calculates centroid, calls get_infra_summary() and get_regulatory_summary()
10. JSON returned to browser → render() populates all panels
11. User opens Pricing Intel → POST /api/pricing-intel → renderPricingPanel()
12. User opens Buyer Persona → GET /api/buyer-persona → renderPersonaDBData()
13. User changes infra radius to 3 km → POST /api/infra → renderInfraTab()
14. User exports CSV → client-side Blob download from _projects array
```

### City-wide analysis (no locality selected)

```
1. User selects Hyderabad with no locality
2. POST /api/supply { city: "Hyderabad", locality: "" }
3. mongo_supply.fetch_supply() uses city-wide query with pagination
4. First page: full normalization + RERA batch prefetch
5. Subsequent pages served from in-memory cache
6. KPIs reflect entire Hyderabad dataset
```

### Radius analysis

```
1. User switches to Radius View
2. User sets center on map and radius (e.g. 3 km)
3. POST /api/supply-radius { lat, lng, radius_km: 3, city: "Hyderabad" }
4. mongo_supply.fetch_supply_by_radius() filters projects by Haversine distance
5. Infra loaded from radius center coordinates
```

---

## Technology Stack

| Layer | Technology | Role |
|---|---|---|
| Backend | Python 3, Flask 3 | HTTP server and API |
| Auth | Flask sessions, Werkzeug password hashing | Login and access control |
| Database | MongoDB via PyMongo | All project and intelligence data |
| Frontend | HTML, CSS, Vanilla JavaScript | Dashboard UI (no framework) |
| Charts | ECharts 5.4.3 (CDN) | All visualizations |
| Maps | Leaflet 1.9.4 (CDN) | Project pins, POI layers, coverage map |
| Typography | Google Fonts (Inter, JetBrains Mono) | UI and numeric display |
| News | Google News RSS | Market News panel |

---

## Business Value

| Team | How the dashboard helps |
|---|---|
| **Leadership** | Instant view of supply density, price positioning, and regulatory exposure in any Hyderabad locality |
| **Sales** | ASBL vs market PSF, competitor project details, and infra talking points for client meetings |
| **Strategy** | Developer concentration, BHK mix, absorption rates, and buyer persona supply gaps |
| **Marketing** | Locality sentiment (99acres/Google), buyer budget ranges, and market news |
| **Product** | Configuration distribution, amenity benchmarks, and segment-wise pricing bands |

---

## Quick Reference

| Item | Value |
|---|---|
| Start server | `python app.py` |
| Landing page | `http://localhost:5000/` |
| Dashboard | `http://localhost:5000/dashboard` |
| Health check | `http://localhost:5000/api/health` |
| MongoDB | `mongodb://localhost:27017` |
| Default city | Hyderabad |
| Default infra radius | 1 km |
| Core files | `app.py`, `mongo_supply.py`, `index.html`, `templates/home.html`, `templates/login.html`, `templates/signup.html`, `templates/forgot_password.html` |

---

*This document describes the Market Intelligence Dashboard as implemented in the seven core application files. It is the primary reference for onboarding and future development.*
