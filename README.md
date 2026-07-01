# RE·ANALYZE — Technical Reference

**Scope of this document:** built exclusively from seven files — `app.py`, `mongo_supply.py`, `index.html`, `home.html`, `login.html`, `signup.html`, `forgot_password.html`. Every claim below is traceable to one of these. Nothing is carried over from prior documentation, chat history, or assumptions about files not listed here. Where app.py imports a module whose source wasn't provided (e.g. `scraper_pipeline`), that module is listed as a dependency with only what's observable from how `app.py` calls it — not its internals.

---

## 1. What the application is

RE·ANALYZE is a Flask + MongoDB web app for Hyderabad real estate market data, gated behind a login. From `home.html`'s own marketing copy: "Analyze supply, pricing, regulatory, and infrastructure data across 800+ Hyderabad localities." The landing page advertises 800+ localities, 31K+ projects, live RERA data, and 8 analysis tabs (the dashboard's actual nav has 11 panels — see §5).

---

## 2. Routes (from `app.py`)

Every `@app.route` in the file, with method and what it does:

| Method | Route | Behavior |
|---|---|---|
| GET | `/` | `render_template("home.html")` |
| GET | `/login` | `render_template("login.html")`; redirects to `/dashboard` if a session already exists |
| GET | `/signup` | `render_template("signup.html")`; redirects to `/dashboard` if a session already exists |
| GET | `/forgot-password` | `render_template("forgot_password.html")` |
| GET | `/dashboard` | `send_from_directory(..., "index.html")`; wrapped in `@_login_required` |
| POST | `/api/auth/login` | Looks up `real_estate.user_authentication` by email, `check_password_hash`, sets `session["user"]`, updates `last_login` |
| POST | `/api/auth/logout` | `session.clear()` |
| POST | `/api/auth/register` | Validates email format + 6-char min password, rejects duplicate email (409), inserts with `generate_password_hash` |
| POST | `/api/auth/forgot-password` | Generates `secrets.token_urlsafe(32)` + 2-hour expiry, stores on user doc, returns `reset_url` in the JSON response (always returns `ok:true` even for unknown emails) |
| POST | `/api/auth/reset-password` | Validates token + expiry, sets new `password_hash`, clears the token |
| GET | `/api/auth/me` | Returns the session user, or 401 |
| GET | `/api/health` | `{"status":"ok","time":...}` |
| POST | `/api/analyze` | Starts a background thread job (`_run`), returns a `job_id` |
| POST | `/api/supply` | Main query — `fetch_supply()` + (if `with_infra`) `get_infra_summary`/`get_nearby_pois`/`get_regulatory_summary` |
| POST | `/api/supply-radius` | `fetch_supply_by_radius()` + infra for that point |
| POST | `/api/pricing-intel` | `get_pricing_intel()` |
| GET | `/api/buyer-persona` | `get_buyer_persona_data()`; params `locality`, `radius_km` |
| GET | `/api/approval-stats` | `get_approval_stats()` |
| GET,POST | `/api/project-intel` | `get_project_intelligence()` |
| GET | `/api/pincode-boundaries` | Queries `insightforge.pincode_boundaries` directly (not via `mongo_supply.py`) |
| GET | `/api/news` | Live Google News RSS fetch with two fallback tiers (see §6.7) |
| GET | `/api/localities-by-city` | `get_localities_by_city()` |
| GET | `/api/locality-intel` | `get_locality_intelligence()` |
| GET | `/api/bp-localities` | Lists localities with direct vs. radius-only `buyer_persona` report coverage |
| GET | `/api/status/<job_id>` | Poll job progress |
| GET | `/api/results/<job_id>` | Completed job results |
| GET | `/api/report/<job_id>` | Download the job's Excel report file |
| GET | `/api/localities` | `list_localities_from_mongo()`, falling back to scanning cached analysis JSON files on disk |
| POST | `/api/load-cached` | Loads a cached analysis JSON file as a synthetic completed job |
| POST | `/api/infra` | `get_infra_summary()` + `get_nearby_pois()` for a raw lat/lng |
| GET | `/api/height-restrictions` | `get_height_restrictions()` |
| POST | `/api/regulatory` | `get_regulatory_summary()` |
| POST | `/api/report-html` | Returns a print-ready HTML report (`_render_report_html()`, an inline f-string in `app.py`) |
| POST | `/api/report-issue` | Inserts a user-submitted data-quality flag into `real_estate.report_issue` |

Every `/api/*` route except the five auth ones is wrapped in `@_login_required`: page routes redirect to `/login`, API routes return `401 {"error":"Authentication required","redirect":"/login"}`.

`app.secret_key` reads `os.environ.get("RE_SECRET_KEY", "re-analyze-secret-key-2024-change-in-prod")` — falls back to a hardcoded string if the env var isn't set.

The app runs via `app.run(debug=True, use_reloader=False, host="0.0.0.0", port=5000, threaded=True)`. A comment in the file explains `use_reloader=False`: it stops Flask restarting (and wiping the in-memory `JOBS` dict) when scraper files under `scrapers/` change on disk.

---

## 3. The `/api/analyze` background pipeline (`_run()` in `app.py`)

When a locality isn't already in the database, `/api/analyze` queues a background thread that runs, in order:

1. **Supply scraping** — `from scraper_pipeline import run_supply_pipeline` (max_pages=4). On exception, falls back to `from scrapermagicbricks import ScraperMagicBricks`.
2. **RERA** — `app.py`'s own `_fetch_rera()`: a BeautifulSoup scrape of `rera.telangana.gov.in/public/viewRegisteredProjects`, best-effort, returns `[]` on any failure.
3. **Infrastructure** — `from infrafinder import InfrastructureFinder`, `.get_infrastructure_data(locality, city, radius_km=...)`.
4. **Demand** — tries a local cache (`_load_cached_demand()`, reading `demand_raw_*.json` files) first; if empty, `from demand_scraper_noreddit import AlternativeDemandScraper`, `.scrape_all(...)`.
5. **Supply processing** — `from supplyprocessor import SupplyProcessor`: `.process()`, `.get_market_summary()`, `.export_csv()`.
6. **NLP / demand analysis** — `from demandanalyzer import DemandAnalyzer().analyze(...)`, only if ≥3 demand posts were collected. Output keys are remapped (`sentiment_distribution`, `overall_sentiment`, `buyer_requirements`, `builders`, etc.) before being stored.
7. **Buyer personas** — `from persona_builder import PersonaBuilder().build_personas(...)`. Each returned `segments[]` entry is reshaped into a `personas` dict keyed `segment_<id>` with fields including `is_nri`, `is_investor`, `wants_gated`, `price_elasticity`, `sentiment_label`.
8. **Excel report** — `from reportgenerator import ReportGenerator().generate(...)`.

Results are written to `data/<city>_<locality>_analysis_<timestamp>.json` and to the in-memory `JOBS` dict (persisted to `data/_jobs/job_<id>.json` after every status change, reloaded on restart for jobs already `done`/`error`).

**Modules imported here but not provided as source files, so undocumented beyond their call signature above:** `scraper_pipeline.py`, `scrapermagicbricks.py`, `infrafinder.py`, `demand_scraper_noreddit.py`, `supplyprocessor.py`, `demandanalyzer.py`, `persona_builder.py`, `reportgenerator.py`.

**Important distinction:** this clustering-based persona pipeline only runs for on-demand scrapes via `/api/analyze`. The live `GET /api/buyer-persona` endpoint used by the dashboard does **not** call `persona_builder.py` at all — it reads pre-existing MongoDB documents (§4, `buyer_persona` DB). These are two separate, non-unified persona systems that happen to share a name.

---

## 4. MongoDB — collections actually queried in `mongo_supply.py`

`mongo_supply.py` connects via three DB accessors: `_re()` → `real_estate`, `_ig()` → `insightforge`, `_bp()` → `buyer_persona`. Every collection below is referenced directly in the code (verified by grepping every `_re()[...]`, `_ig()[...]`, `_bp()[...]`, and `db[...]` access).

### `real_estate`

| Collection | Used by | Notes |
|---|---|---|
| `projects_master` | `fetch_supply`, `fetch_supply_by_radius`, `get_pricing_intel`, `get_project_intelligence` | The only project data source. Module docstring states 28,869 docs. Schema confirmed field-by-field in `_norm_master()` — see §4.1. |
| `99a_locality_report` | `get_locality_intelligence` | 99acres locality-level report: `average_rate`, `ratings_summary`, `likes`, `dislikes`, `features_ratings`, `reviews_list`, `whats_great`, `whats_needs_attention`, `price_trends`, `sidebar_prices`. |
| `google_reviews` | `get_locality_intelligence` | Per-project Google reviews matched to a locality via a `neighborhood` field: `totalScore`, `reviewsCount`, `reviews[]`, `title`, `address`. |
| `report_issue` | `app.py`'s `/api/report-issue` (inserted directly, not via `mongo_supply.py`) | User-submitted data-quality flags. |
| `user_authentication` | `app.py`'s auth routes (via `_users_col()`, also direct, not via `mongo_supply.py`) | `email`, `name`, `password_hash`, `created_at`, `last_login`, `reset_token`, `reset_expires`. |
| `pincode_boundaries` | `app.py`'s `/api/pincode-boundaries` (queried directly in `app.py`, not through `mongo_supply.py`) | GeoJSON pincode polygons. **`index.html` never calls this endpoint** — see §7. |

### `insightforge`

| Collection | Used by | Notes |
|---|---|---|
| `points_of_interest` | `get_nearby_pois` | Coordinates in `geometry.coordinates` (GeoJSON `[lng, lat]`), not a `location` field. Filtered by `poi_type`, mapped to category keys via a hardcoded `_POI_KEY_MAP`. |
| `metro_stations`, `hospitals`, `schools`, `malls`, `it_companies`, `universities`, `junior_colleges`, `parks`, `bus_stops`, `industries`, `banks` | `get_infra_summary` | Iterated via a fixed `specs` list of `(key, collection_name, projection, extra_field)` tuples. |
| `lakes` | `get_infra_summary` | Returns nearby lakes with `buffer_m`, `is_protected`, and a computed `in_buffer_zone` flag. |
| `airport_height_restriction_zones` | `get_height_restrictions` | Queried with no spatial filter (function docstring: only 58 docs, all Hyderabad). |
| `approval_project_matches` | `get_approval_stats` | Aggregated via `$group` on `{stage_type, status}`; stage types named in the docstring: `aai_noc`, `environmental_clearance`, `fire_noc`, `municipal_permission`, `rera`. |
| `approval_projects` | `get_approval_stats` | A *different* collection from `approval_project_matches` — holds `district`/`mandal`/`village`/`id`, used to resolve which `project_id`s belong to a locality before aggregating matches. |
| `hmda_all_records` | `get_regulatory_summary` | Matched by `village` OR `mandal` (function comment: village field ~99% populated, used preferentially). |
| `fire_noc_r4` | `get_regulatory_summary` | Matched by `mandal` only (function comment: the `village` field here is "heavily corrupted"). |
| `customer_lifestyle_survey`, `customer_property_survey` | `get_buyer_insights` | Survey data. **`get_buyer_insights()` exists but is never called from any `app.py` route** — see §7. |
| `rera_scraped_data` | `get_rera_absorption` (called from `_norm_master`) | Per-RERA-number absorption: `booked_apartments`, `total_apartments`, `available_units`, `absorption_pct`, `construction_progress_pct`, `has_litigations`, `is_highrise`. |

### `buyer_persona`

| Collection | Used by | Notes |
|---|---|---|
| `localities` | `get_buyer_persona_full`, `get_locality_centroid` | Per-locality `market_tier`, `median_budget`, `centroid`, `rera_registered_units`. Also the fallback centroid source when a locality's `projects_master` docs lack coordinates. |
| `micromarkets` | `get_buyer_persona_full` | Looked up by exact-match locality name regex; structured claims doc (exact fields not enumerated in this pass). |
| `reports` | `get_buyer_persona_full` | Pre-generated reports queried by `{_locality, _radius_km}`. `_radius_km=0` (implied default) vs. an explicit 1–10 km value selects different stored documents; falls back to the closest available radius if the exact one is missing. |

`list_bp_localities()` and `get_buyer_persona_data()` (the function actually called by the `/api/buyer-persona` route) both sit on top of `get_buyer_persona_full()`.

### 4.1 `projects_master` field schema (from `_norm_master()`)

The document is nested under top-level keys `identity`, `location`, `pricing`, `rera`, `specifications`, `configurations`, `reviews`, `building`, `_meta`, `locality_overview`, `amenities`, `media`, `units[]`. Key derivations, exactly as coded:

- **BHK list**: primary source is `configurations.bhk_list`; supplemented by parsing `units[].apartment_type` (regex `(\d+(?:\.\d+)?)\s*BHK`) and, if still empty, by parsing `configurations.cards[].label`.
- **Price**: `pricing.min_price`/`max_price`, with a ₹50 Cr sanity cap (values above are zeroed — comment notes this filters a "₹60,000,000,000 sentinel" garbage value seen in the data) and a card-price fallback when the top-level fields are missing.
- **PSF**: three-tier fallback — (1) `pricing.price_per_sqft` if present; (2) parsed from `pricing.sqft_range_str` text; (3) derived as `price_min / avg(min_size_sqft, max_size_sqft)`. Tiers 2 and 3 set `psf_is_estimated: true`.
- **Status**: normalized from `identity.construction_status` (or `rera.status` as fallback) into exactly one of `Ready to Move`, `New Launch`, `Under Construction`, `Pre Launch`, or `Unknown`.
- **Segment**: `identity.project_segment` directly (comment: "set by classify_segments.py" — that script wasn't provided). `is_gated` is `True` iff `segment == "Gated Community"`.
- **Amenities**: `amenities.all` (a code comment explicitly flags that `facilities.items` is the *wrong*, previously-buggy path).
- **Absorption**: only populated when the doc has a `rera.number` — then joined against `insightforge.rera_scraped_data` via `get_rera_absorption()`, with a per-request `rera_cache` dict to avoid repeat lookups.
- **Coordinates**: flat `location.lat` / `location.lng` — not GeoJSON.

### 4.2 Locality name handling

`canonicalize_locality()` runs a 400+ entry `LOCALITY_ALIASES` dict (visible in full in `mongo_supply.py`, e.g. `"himayath nagar"` → `"Himayat Nagar"`, `"kphb phase 3"` → `"KPHB"`, junk entries like `"road"`/`"hyderabad"`/`"na"` map to `None`). `get_locality_areas()` is a function (not a static dict) that resolves a canonical locality to its HMDA mandal(s)/village(s) for the regulatory joins above.

### 4.3 Geospatial implementation

`fetch_supply_by_radius()` does **not** use MongoDB `$geoWithin`/`2dsphere`. It computes a lat/lng bounding box (`_bbox()`) as a pre-filter, then an in-Python Haversine distance check (`_haversine()`) against `location.lat`/`location.lng`. `get_infra_summary`/`get_nearby_pois` do the same bbox+Haversine pattern, but against `geometry.coordinates` (GeoJSON `[lng, lat]`) instead — i.e. the two systems store coordinates in different shapes and are queried with different field paths.

---

## 5. Frontend — `index.html` (the dashboard, served at `/dashboard`)

### 5.1 Confirmed structure

The sidebar nav (`<div class="sb-item" data-panel="...">`) defines exactly 11 panels in 3 labelled groups:

```
Analysis:      Overview · Supply Intel · Project Intel · Pricing Intel · Infrastructure · Regulatory
Intelligence:  Market News [LIVE] · Buyer Persona · Locality Intel
Platform:      Coverage Map · Reports & Export
```

Tech confirmed from the `<head>`: Google Fonts Inter + JetBrains Mono, Leaflet 1.9.4, ECharts 5.4.3 (all loaded via CDN `<script>`/`<link>` tags). A `data-theme` attribute on `<html>` is set from `localStorage('ra-theme')` or `prefers-color-scheme` before paint; `[data-theme="light"]`/`[data-theme="dark"]` selectors define two full CSS variable sets (`--bg`, `--text`, `--amber`, etc.), plus a shared status palette (`--warn`, `--teal`, `--red`, `--blue`, `--purple`, `--green`, `--coral`) used regardless of theme. A topbar `☀`/`☽` pair calls `setTheme('light'|'dark')`.

### 5.2 Overview panel

KPI labels present in the HTML: Total Projects, Total Units, Under Construction, Ready to Move In, New Launches, Avg Price/Sqft, Gated Communities, Absorption Rate, RERA Registered, RERA Approved, HMDA Projects, GHMC Projects. Card titles: Segment Split, Status Distribution, Price Buckets, Geo Intelligence Map, BHK Configuration, Top Developers. A `mode-toggle` switches between `setMode('locality')` and `setMode('radius')`; radius mode shows preset buttons (`setRadius(1|3|5, el)`) plus a custom km input, and the supply call switches to `POST /api/supply-radius`.

### 5.3 Supply Intel panel

Card titles: Inventory Status, Price Distribution, Supply Mix, All Projects. The "All Projects" table is server-paginated — the fetch body includes `page` and `page_size` (`_supplyPageSize`, default `200`), and a `#supply-pagination` control (initially hidden) is shown once results return.

### 5.4 Project Intel panel

Card titles: Top 15 Developers, Developer Market Share, RERA Approval Status, Project Distribution by Units, Litigation & Risk Flags.

### 5.5 Pricing Intel panel

Card titles: ASBL vs Market PSF, Market Avg PSF, PSF Band, ASBL Projects, Competitor PSF Comparison, Price by Segment, Price by Configuration, ASBL Projects Detail. Rendered by `renderPricingPanel(d)`, fed from `POST /api/pricing-intel` (`get_pricing_intel()` — see §4). ASBL identification is a simple substring match: `"asbl" in developer.lower() or "asbl" in project_name.lower()`.

### 5.6 Regulatory panel

Card titles: Authority, Approved Projects, Litigations, Projects with Litigation Flag. **Confirmed from `renderRegulatory(reg, projects)`: the `reg` argument — the object returned by the backend's `get_regulatory_summary()`, containing the actual HMDA/Fire NOC/approval-stage data — is never read.** Every number on this panel is instead computed from `_data.supply_summary` (the `ghmc_projects`/`hmda_projects`/`rera_count`/`rera_approved_count` fields from `_compute_summary()` in `mongo_supply.py`) and from per-project `rera_authority`/`rera_approved`/`has_litigations` fields already present in `supply_projects`. In other words: `insightforge.hmda_all_records` and `fire_noc_r4` are fetched server-side but not rendered anywhere in this panel.

### 5.7 Infrastructure panel

Card titles: Nearby Amenities Map, Infrastructure Radar, Connectivity Score, Nearest Amenities, Lakes & Buffer Zones, Nearby Places.

### 5.8 Market News panel

A scrolling ticker (`#ticker-inner`) ships with 5 hardcoded items in the HTML, then gets overwritten by `loadLiveNews()`. Below it, one card titled "Market News" with filter buttons (`filterNews('all'|'RERA'|'INFRA'|'LAUNCH'|'POLICY', el)`) and a refresh button (`loadLiveNews(true)`), feeding into `#news-feed`. No separate timeline or policy-tracker widget exists in the markup beyond this single filterable card.

### 5.9 Buyer Persona panel

This panel is built entirely around the `buyer_persona` DB report (§4), not a persona-card grid. Structure: a radius selector bar (`Exact Locality` / 1 / 3 / 5 / 10 km, `setBpRadius()`), a fallback notice (shown when the served data is from a nearby locality rather than an exact match, with an "included localities" chip list), then cards titled Buyer Profile (with a confidence badge, median budget, avg income, and a coverage-quality bar), Market Signals (market tier, median budget, dominant BHK, a second coverage indicator), Income & Budget, BHK Breakdown, Buyer Pain Points, Decision Signals (with a "View matching projects →" button that jumps to the Supply tab), Employer Cluster, Age & Income, Area Range by BHK, Designation Split. Several chart containers (`chart-persona-mix`, `chart-buying-intent`, `chart-bhk-demand`, `persona-tier-dist`, `persona-budget-dist`) exist in the DOM but are explicitly wrapped in `style="display:none"` with an HTML comment: "Row 2 (removed): Buyer Mix, Buying Intent, BHK Demand — hidden per spec" and "Row 6: Market-wide context — removed per spec".

### 5.10 Locality Intel panel

Card titles: Locality Overview (avg property rate, "what's great" tags), Locality Ratings (99acres overall rating, star breakdown), Needs Attention (dislikes/pain points), What People Like, Google Reviews (aggregate rating + most-reviewed projects), Price Trend (chart), Resident Reviews (verified 99acres review list). Fed by `GET /api/locality-intel`, fired from `switchPanel()` when `name === 'loc-intel'`.

### 5.11 Coverage Map panel

**Not a pincode-boundary map.** Four summary stats (one live: "Localities in System"; three hardcoded in the HTML: `31K+` Total Projects, `100` Buyer Persona Data, `94%` Hyderabad Metro). Below that: a "Locality Coverage Map" card with a `By Projects`/`By Price` view toggle (`setCoverageView()`) and a segment filter (All/Premium/Mid-Segment/Affordable, `setCoverageFilter()`) rendered on a Leaflet map (`#map-coverage`, centered on `[17.385, 78.487]`, zoom 11) — populated from `/api/localities-by-city` and `/api/localities`, **not** `/api/pincode-boundaries`. Alongside it: "Explore More Locality" (a scrollable list, `#top-localities`) and "Locality Comparison" — click up to 3 localities on the map or list to select them, then `runComparison()`.

### 5.12 Reports & Export panel

Card titles: PDF Report (`generateReport()` → `/api/report-html`, opens print dialog), Data Export (`exportCSV()`, client-side CSV of the current project set), Recent Analyses (`loadRecent()` → `GET /api/localities`).

### 5.13 Other confirmed global UI

- A right-side detail slide-out (`#detail-panel`) with name/developer/badges and a ratings section, closed via `closeDetail()`.
- Topbar: theme toggle, a `READY`/status badge, hidden-until-relevant PDF/CSV buttons, an "Analyze" button (`loadData()`), and a "⎋ Sign Out" button (`doLogout()` → `POST /api/auth/logout`).
- `#api-dot` / `#api-status` connection indicator in the sidebar.

---

## 6. Auth pages (`home.html`, `login.html`, `signup.html`, `forgot_password.html`)

All four share one design system: Inter + JetBrains Mono fonts, dark `#0f1117`/`#161b27`/`#1e2535` background scale, teal `#0d9488` accent, `border-radius: 8px` inputs/buttons. (This is the same Inter/JetBrains Mono system `index.html` itself uses — there is one unified design language across the whole app, not two.)

- **`home.html`** (`/`): nav with Sign In / Get Started buttons (both → `/login`); hero with a badge ("Live RERA & Market Data · Hyderabad"), headline, a stats strip (800+ Localities, 31K+ Projects, "Live" RERA Data, 8 Analysis Tabs); six feature cards (Supply Intelligence, Pricing Intelligence, Regulatory & RERA, Infrastructure Map, Buyer Persona, Coverage Map); a 4-step "how it works" list; a closing CTA box; footer text "Hyderabad Real Estate Intelligence Platform · Data sourced from RERA, HMDA, GHMC."
- **`login.html`** (`/login`): email + password form, `doLogin()` posts JSON to `/api/auth/login`, on `{ok:true}` redirects to `/dashboard`, otherwise shows `data.error` in an inline banner. Footer links to `/forgot-password` and `/signup`.
- **`signup.html`** (`/signup`): name, email, password (min 6 chars, `minlength` attribute), confirm-password. `doSignup()` checks password===confirm client-side before posting to `/api/auth/register`; on success shows a message and redirects to `/login` after 1.8s.
- **`forgot_password.html`** (`/forgot-password`): two-state page. Default state posts an email to `/api/auth/forgot-password` and, if the response includes a `reset_url`, renders it as a clickable link directly in the page (a `.reset-link-box` element) — it is not emailed. If the page loads with a `?token=` query param, JS swaps the UI to a "set new password" form that posts `{token, password}` to `/api/auth/reset-password`, then redirects to `/login` after 2s.

All four forms use `fetch()` with `Content-Type: application/json`, show a CSS spinner on the submit button while in flight, and display either `data.error` or a generic "Connection error" message on `catch`.

---

## 7. Things observed that look like dead code or unused capacity

Listed because they're directly verifiable from these files, not because they're necessarily bugs:

- `GET /api/pincode-boundaries` exists in `app.py` and queries `insightforge.pincode_boundaries`, but `index.html`'s Coverage panel never calls it — that panel uses `/api/localities-by-city` and `/api/localities` instead.
- `get_buyer_insights()` exists in `mongo_supply.py` (reads `customer_lifestyle_survey` + `customer_property_survey`) but is not wired to any route in `app.py`.
- The Regulatory panel in `index.html` ignores the `regulatory` object returned by `/api/supply` / `/api/regulatory` (i.e. `get_regulatory_summary()`'s HMDA/Fire NOC output) and computes everything from fields already in `supply_summary`/`supply_projects` instead (§5.6).
- The on-demand `/api/analyze` pipeline's `persona_builder.py` clustering output and the live `/api/buyer-persona` endpoint's `buyer_persona`-DB output are two unconnected systems that both populate something called "personas."
- `app.secret_key` has a hardcoded fallback value if `RE_SECRET_KEY` is unset in the environment.
- Password reset delivers the link in-page rather than by email — there's no email-sending code in `app.py`.
