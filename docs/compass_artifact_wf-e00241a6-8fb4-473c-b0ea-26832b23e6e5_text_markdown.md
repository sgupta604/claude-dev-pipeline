# Boreas: Enterprise Architecture for Wind-Aware Agricultural Flight Planning

**Boreas should be built as a hybrid architecture — a Next.js frontend paired with a FastAPI Python computation backend and a shared Rust physics core compiled to both native and WebAssembly.** This design is driven by a fundamental insight: the platform's core value proposition depends on meteorological science that demands Python's unmatched scientific computing ecosystem (MetPy, xarray, NumPy), while its interactive drift visualization demands browser-native performance achievable only through WebAssembly. The HRRR weather model's 3km resolution, combined with bilinear interpolation and a tiered drift engine (sub-millisecond lookup tables for interactive preview, Lagrangian particle tracking for detailed analysis), can deliver professional-grade spray planning at interactive speeds. Critically, as of March 2026, the AGDISP drift model — EPA's gold standard — is undergoing active modernization with Bayer Crop Science joining that effort, creating a strategic window for Boreas to align its physics engine with an evolving regulatory framework.

---

## 1. Scientific foundation: drift physics and validated accuracy

### The AGDISP–AgDRIFT modeling hierarchy defines the accuracy standard

The U.S. spray drift modeling landscape centers on two related models developed over four decades. **AGDISP** (AGricultural DISPersal), created by Continuum Dynamics Inc. for the USDA Forest Service beginning in 1979, is a Lagrangian particle-tracking model that solves equations of motion for discrete droplet categories through the aircraft wake and atmosphere. It accounts for aircraft wake vortices, droplet evaporation, atmospheric turbulence (via mean relaxation time and turbulent travel time), gravity, and canopy penetration. AGDISP has been validated against extensive field studies conducted between 1972 and 1995, with the Spray Drift Task Force providing the definitive evaluation basis. **Validation accuracy extends to approximately 800m downwind**, with a Gaussian extension model (Teske & Thistle, 2004) stretching predictions to 20km at reduced confidence.

**AgDRIFT** embodies AGDISP's Lagrangian computational engine but wraps it in a three-tier regulatory framework. Tier I provides screening-level estimates from empirical curves derived from SDTF field data — requiring only droplet size class and application method as inputs. Tier II enables parameter adjustment (boom height, wind speed, temperature). Tier III offers the full AGDISP engine with detailed aircraft, nozzle, and meteorological inputs. EPA uses AgDRIFT for ecological and human health risk assessments in pesticide registration, while Australia's APVMA has recently switched to AGDISP directly for aerial assessments because it has continued evolving while AgDRIFT has not.

**Accuracy benchmarks from the literature** reveal that AGDISP tends to overpredict average volume deposition by roughly a factor of two compared to field measurements. The model shows limited sensitivity to wind direction variations within 10 degrees of the flight path. The Spray Drift Task Force database confirms that the **three dominant variables affecting off-target deposition are droplet size, spray release height, and wind speed** — in that order of importance. Droplet size classification dominates: field studies show that drift curves cluster by droplet size category regardless of wind speed, with Fine-class sprays producing dramatically more drift than Coarse-class sprays at any wind velocity.

### Droplet physics: terminal velocity and the 200-micron threshold

The ASABE S572.1 standard defines **six primary droplet categories** relevant to agriculture: Very Fine (VMD <145 μm), Fine (145–225 μm), Medium (226–325 μm), Coarse (326–400 μm), Very Coarse (401–500 μm), and Extremely Coarse (>500 μm). Ultra Coarse (>665 μm) is a seventh category seen with specialized drift-reduction nozzles.

Terminal velocity governs drift distance. Stokes' law (F_drag = 6πμrv) applies for Reynolds numbers below approximately 1, corresponding to **droplets smaller than about 80 μm in air**. Beyond this range, the Oseen correction and empirical drag correlations (Schiller-Naumann: C_D = 24/Re × (1 + 0.15Re^0.687)) become necessary. Practical terminal velocities for water droplets in air at standard conditions are approximately:

| Diameter (μm) | Terminal Velocity (m/s) | Fall Time from 3m (s) | Drift at 5 mph wind (m) |
|---|---|---|---|
| 100 | ~0.25 | 12.0 | 26.8 |
| 150 | ~0.55 | 5.5 | 12.3 |
| 200 | ~0.72 | 4.2 | 9.4 |
| 300 | ~1.2 | 2.5 | 5.6 |
| 500 | ~2.0 | 1.5 | 3.4 |

Field data from Ohio State University confirms that **drift distances for 200 μm and larger droplets do not exceed 0.3m** (about 1 foot) at wind velocities up to 10 mph when released from typical boom height (0.5m). The dramatic threshold occurs around 150 μm — below which drift increases exponentially. This is why the 200 μm VMD threshold is the practical dividing line between "manageable drift" and "high drift risk."

Evaporation significantly affects fine droplets. Droplets below 50 μm can completely evaporate before reaching the ground under typical field conditions (25°C, 60% RH), leaving only non-volatile residue cores that remain suspended indefinitely. Droplets between 50–200 μm experience significant evaporation affected by temperature and humidity. Above 200 μm, evaporation has negligible effect on drift distance during the short fall time.

### UAV-specific drift: recent field validation data

A rapidly growing body of UAV drift studies from 2020–2025 provides Boreas-specific validation data. A landmark study of **114 UASS drift trials** conducted in China (2021–2022) using the ISO 22866 protocol found that drift from UAV applications is highly concentrated near the field boundary, declining sharply within the first 5 meters. A 2025 vineyard study found that under optimized UAV settings (2.0m AGL, 1.0 m/s flight speed, buffer line applied), **drift became negligible beyond 10m downwind** — reducing drift at the closest sampling distance by 65–70% compared to conventional air-assisted sprayers.

Random forest feature importance analysis of the 114-trial dataset established that **wind speed and flight altitude are the dominant drift drivers for UASS**, followed by nozzle type and flight speed. The CropLife America Drones Working Group concluded that UASS drift lies between airplane and ground-based applications, comparable to airblast applications for orchard use. CHARM+AGDISP (coupling a helicopter aeromechanics model with AGDISP) has been adapted for UAV rotor wake modeling, but this combined model is proprietary to Mount Rose Scientific.

### EU regulatory drift: the Ganzelmeier-Rautmann framework

European drift regulation relies on the **Ganzelmeier-Rautmann empirical drift curves** — power-law regressions of the form D(x) = A × x^B, where D is drift deposition as a percentage of application rate and x is downwind distance. Originally developed from 119 drift studies conducted between 1989–1992 at Germany's BBA (now BVL), these curves were expanded to cover field crops, vineyards, fruit, and hops. The FOCUS surface water working group uses these values for EU-wide aquatic risk assessment. A 2025 EFSA analysis found that Ganzelmeier-Rautmann curves both overestimate and underestimate drift deposits, with overestimation (conservative from a regulatory standpoint) being the most common outcome. The Dutch SPEXUS model represents the next generation, accounting for weather conditions and crop development stage, but is not yet ready for regulatory use.

**Boreas should implement the Rautmann drift curves** as a simple, fast computation path that enables instant buffer zone estimation without full simulation, supplementing them with AGDISP-style Lagrangian modeling for detailed analysis.

### What Boreas needs: a tiered drift computation strategy

Based on the scientific literature, Boreas should implement three tiers of drift computation:

**Tier 1 — Lookup Table (interactive, <1ms):** Pre-computed drift envelopes parameterized by VMD class (6 categories), wind speed (5 bins), release height (3 bins), and temperature/humidity (2 bins). Approximately 180 entries, each storing drift deposition as polynomial coefficients. Query time is O(1) with interpolation. This powers the real-time map overlay as users adjust parameters.

**Tier 2 — Simplified Lagrangian (<500ms in WASM):** Track 1,000–5,000 representative particles across 5–10 size classes through a uniform wind field with basic turbulence parameterization (Gaussian diffusion). Uses the equations of motion dX/dt = V_drop, dV/dt = F_drag + F_gravity + F_turbulence. Sufficient for planning-grade accuracy. Powers the detailed drift visualization and buffer zone calculation.

**Tier 3 — Full Monte Carlo (1–5s async):** 100 stochastic scenarios with turbulence variation for robust drift envelopes. Accounts for droplet evaporation, atmospheric stability, and variable wind fields. Used for final plan validation and compliance reporting.

---

## 2. Atmospheric science: wind data and boundary layer physics

### HRRR is the optimal weather data source for US agricultural operations

The **High-Resolution Rapid Refresh (HRRR)** model operates at 3km horizontal resolution with hourly update cycles (24 runs per day), producing forecasts out to 18 hours (48 hours for the 00, 06, 12, 18 UTC runs). It is the highest-resolution operational forecast model run by NCEP. HRRR surface output includes 10m u- and v-wind components, 2m temperature, 2m dewpoint, and surface pressure — precisely the variables needed for spray drift calculation.

Data access is straightforward and free. HRRR GRIB2 files are archived on **AWS S3** (s3://noaa-hrrr-bdp-pds/), **Google Cloud Storage**, and **Microsoft Azure**. The University of Utah maintains a parallel **Zarr-format archive** (s3://hrrrzarr/) that provides ~40× faster access for time-series queries. Surface files are approximately 100 MB per forecast hour in GRIB2 format, but Zarr chunks are ~1 MB per variable, enabling efficient point queries. Data availability latency is approximately 45–60 minutes after model initialization for GRIB2 and ~3 hours for Zarr.

**For Boreas specifically, the 10m wind product is directly applicable** to drone operations at 2–10m AGL. The HRRR's Lambert Conformal Conic projection requires coordinate transformation for point queries, but the **Herbie** Python library provides a standard interface: `H = Herbie('2026-03-16 12:00', model='hrrr'); ds = H.xarray("UGRD:10 m")`.

For a typical agricultural field (40–640 acres, spanning 0.16–2.6 km²), the field encompasses **1–3 HRRR grid cells at most**. Wind variation across this scale is minimal under normal conditions. Bilinear interpolation from the 4 surrounding grid points is computationally trivial (O(1) per query) and meteorologically sufficient.

### Near-surface wind profile extrapolation

The logarithmic wind profile, u(z) = (u*/κ) × ln((z−d)/z₀), describes wind speed variation with height in the surface layer (lowest ~100m). The von Kármán constant κ ≈ 0.41. Critical parameters for agricultural terrain include:

**Surface roughness length (z₀):** Bare soil: 0.001–0.01 m. Short grass/pasture: 0.01–0.03 m. Short crops (wheat, soybeans): 0.05–0.1 m. Tall crops (corn): 0.1–0.25 m. Orchards: ~0.5 m. z₀ approximates 10% of the average height of roughness elements.

**Zero-plane displacement (d):** For dense, uniform crop canopy, d ≈ 2/3 to 3/4 of canopy height. For a 1m corn crop, d ≈ 0.7m.

Extrapolating HRRR's 10m wind to drone spray height (3–5m AGL) using the log profile introduces modest uncertainty. For typical agricultural terrain (z₀ = 0.05m, neutral stability), wind speed at 3m is approximately **75–85% of the 10m value**. Uncertainty in z₀ by a factor of 2 corresponds to approximately **6% uncertainty in surface wind speed** — acceptable for spray planning.

The Monin-Obukhov stability correction ψ(z/L) becomes important under non-neutral conditions. During **stable conditions** (nighttime, temperature inversions), wind speed decreases more rapidly toward the surface and turbulent mixing is suppressed — exactly when spray drift is most dangerous because suspended fine droplets can travel long distances without vertical dispersal. During **unstable conditions** (sunny afternoon, heated surface), enhanced mixing disperses droplets vertically but also brings upper-level wind momentum down to the surface. For Boreas, incorporating Pasquill-Gifford stability class estimation (derivable from wind speed and solar radiation) provides the most practical accuracy improvement beyond the neutral log profile.

### Spray timing windows and atmospheric stability

Professional ag aviators typically operate during specific atmospheric windows. **Early morning (1–3 hours after sunrise)** and **late afternoon** are preferred because winds are moderate and steady, the boundary layer is transitioning from stable to neutral or vice versa, and thermal convection is minimal. The **worst conditions for spraying** are temperature inversions (stable stratification trapping fine droplets near the surface for long-distance transport) and gusty convective afternoons. Most pesticide labels restrict application to **wind speeds between 3–10 mph** — below 3 mph indicates potential inversion conditions, above 10 mph creates excessive mechanical drift.

---

## 3. Recommended architecture: hybrid with shared physics core

### Why Next.js alone is insufficient — and what to use instead

The current specification's Next.js monolith fails for three specific reasons. First, Vercel's serverless functions have a **4.5 MB response body limit**, too small for weather data payloads. Second, JavaScript lacks any equivalent to Python's meteorological computing ecosystem (MetPy, xarray, cfgrib, Herbie). Third, stateless serverless invocations cannot maintain in-memory weather grid caches, forcing expensive re-fetches. Similar platforms confirm this pattern: Windy.com uses heavy server-side pre-processing with client-side WebGL rendering; ForeFlight integrates multiple weather sources through a native backend; CesiumJS offloads tile processing to Web Workers.

**The recommended architecture:**

```
┌──────────────────────────────────────────────┐
│          Next.js 14+ Frontend (Vercel)        │
│  React + Mapbox GL JS + Three.js              │
│  WASM drift preview in Web Worker (Comlink)   │
│  SSE for weather updates, WebSocket for sims  │
└──────────────┬───────────────────────────────┘
               │ REST API + SSE/WebSocket
┌──────────────┴───────────────────────────────┐
│       FastAPI Computation Service             │
│  Python 3.12+: MetPy, xarray, Herbie, NumPy  │
│  Rust physics via PyO3 bindings               │
│  ARQ task queue for batch simulations         │
│  Deployed: Railway / Fly.io / AWS ECS         │
├──────────────────────────────────────────────┤
│  PostgreSQL + PostGIS  │  Redis (cache+pubsub)│
└──────────────────────────────────────────────┘
```

The **shared Rust physics core** is the architectural centerpiece. A single Rust library implementing drift simulation (Lagrangian particle tracking, lookup table generation, droplet physics) compiles to:
- **Native binary** via PyO3 for server-side batch computation and compliance reporting
- **WebAssembly** via wasm-bindgen for browser-based interactive preview in Web Workers
- **Test binary** for CI/CD with synthetic wind fields and zero infrastructure dependencies

This eliminates the risk of physics divergence between client preview and server computation — a critical property for a tool where regulatory accuracy matters.

### Domain-driven design with five bounded contexts

The system naturally decomposes into five bounded contexts with clear communication patterns:

**Weather Context** owns forecast ingestion, caching, and interpolation. It publishes `NewForecastAvailable` and `WindFieldUpdated` domain events when new HRRR cycles arrive. Its anti-corruption layer translates OGC EDR CoverageJSON or raw GRIB2 into domain `WindField` value objects.

**Drift Simulation Context** subscribes to weather events and owns all drift physics. Its aggregate root `SimulationRun` is immutable once computed — new parameters require a new run. It produces `DriftResult` objects containing deposition grids, buffer distances, and maximum drift distances.

**Flight Planning Context** orchestrates the user-facing workflow. Its `FlightPlan` aggregate enforces invariants: plans cannot be approved without passing compliance checks and drift assessment within threshold. It consumes drift results and generates flight paths using wind-aware coverage path planning.

**Field Management Context** owns field boundaries, crop zones, and terrain data. It shares a kernel of geometry types (GeoJSON polygons) with Flight Planning.

**Compliance Context** acts as a validation service, encoding EPA buffer zone rules, label-specific wind speed limits, state regulations, and Part 137 operational requirements. It is queried synchronously before flight plan approval.

### Hexagonal architecture for the physics core

The drift simulation engine follows strict dependency inversion. Ports (interfaces) define `WeatherDataPort`, `DriftSimulationPort`, and `SimulationResultPort`. Adapters implement these for specific infrastructure:

```python
class WeatherDataPort(Protocol):
    async def get_wind_field(
        self, bounds: BoundingBox, altitude: Altitude,
        time: datetime, model: str = "HRRR"
    ) -> WindField: ...

# Adapters: OgcEdrWeatherAdapter, HerbieWeatherAdapter,
#           OpenMeteoWeatherAdapter, MockWeatherAdapter
```

This makes weather sources swappable without modifying physics code — essential as the OGC EDR ecosystem matures and providers like Shyft Weather and Folkweather evolve.

---

## 4. Computational engine: algorithms and performance

### Wind field interpolation: bilinear is enough

For field-scale operations, the research is clear: **bilinear interpolation from the 4 nearest HRRR grid points** provides sufficient spatial accuracy. The 3km grid means a typical 160-acre field sits within a single grid cell. Ordinary Kriging outperforms IDW for irregular station networks, but for regular model grids the additional complexity yields negligible improvement. Temporal interpolation between hourly forecasts should use linear interpolation of u/v wind components independently. Performance: O(1) per query, effectively free.

### Coverage path planning: Fields2Cover as reference

**Fields2Cover** (BSD-3 license, C++ with Python bindings, published in IEEE RA-L 2023) provides the most complete open-source coverage path planning pipeline: headland generation → swath generation → route planning → path planning. It computes complete coverage paths for 1 hectare in 0.5–3.5 seconds with brute-force angle search.

For wind-aware spray operations, the key finding from Coombes et al. (2019, ICRA) is that **swaths should be oriented perpendicular to wind direction** to minimize off-target drift (drift goes along the swath rather than across it). The optimal flight direction is therefore **parallel to wind** with spray rows perpendicular. Additionally, the spray sequence should proceed **upwind to downwind** to prevent flying through one's own spray cloud.

Boreas should port Fields2Cover's core algorithms to Rust (for WASM compilation) and extend them with wind-awareness: dynamic swath angle optimization as a function of real-time wind direction, overlap adjustment based on predicted crosswind drift, and battery-optimized swath ordering via nearest-neighbor TSP heuristic.

### Client-side computation: WASM and WebGPU

WebAssembly benchmarks (Jangda et al., USENIX ATC 2019) show WASM runs **1.45–1.55× slower than native** for compute-heavy tasks. For Boreas's drift calculations, this means:

| Computation | WASM Performance | Interactive? |
|---|---|---|
| Lookup table drift query | <0.01ms | Yes, instant |
| 1,000-particle Lagrangian sim | 5–50ms | Yes |
| 10,000-particle simulation | 50–500ms | Yes, with progressive rendering |
| Path planning (160-acre field) | 1–5s | Async with progress updates |
| Full Monte Carlo (100 scenarios) | 500ms–5s | Async |

**WebGPU compute shaders** can handle 1M+ particles at 60 FPS, making the drift visualization pipeline (10,000 droplet trajectories rendered as particle trails) trivially interactive. Browser support as of early 2026 covers Chrome 113+, Edge 113+, with Firefox nearing stable release — approximately 70–80% of desktop users. Three.js's experimental WebGPU renderer enables unified rendering of the drift visualization and field map.

The recommended client architecture uses two dedicated Web Workers (drift engine and path planner) communicating with the main thread via Comlink proxies, with Transferable ArrayBuffer objects for zero-copy data transfer.

### Caching: four-layer temporal-spatial strategy

Weather data caching aligns with HRRR's known update cycle:

**Layer 1 (CDN/Edge):** Pre-processed weather visualization tiles, TTL 1 hour aligned to HRRR cycles. `Cache-Control: public, max-age=3600, stale-while-revalidate=1800`.

**Layer 2 (Redis):** Decoded wind field data for API regions, TTL 2 hours. Tag-based invalidation keyed by model run (`hrrr:2026031612`). Redis Pub/Sub broadcasts `weather:updated` events to connected clients.

**Layer 3 (Service Worker):** Pre-fetched weather data for planned operations, supporting offline field use with up to 50 MB cached data and stale-while-revalidate pattern.

**Layer 4 (In-memory):** Decoded Float32Array wind grids and interpolation results, LRU eviction at 20 MB, invalidated when new forecasts load.

---

## 5. Data architecture and storage patterns

### PostgreSQL + PostGIS for spatial domain data

Field boundaries, flight plans, waypoints, and compliance records live in PostgreSQL with PostGIS extensions for spatial queries. Key tables include:

**fields** (id, owner_id, boundary GEOMETRY(Polygon, 4326), crop_type, crop_height_m, surface_roughness_z0, created_at)

**flight_plans** (id, field_id, status, route GEOMETRY(LineString, 4326), altitude_m, scheduled_start, weather_snapshot_id, drift_assessment_id, compliance_result JSONB)

**simulation_runs** (id, flight_plan_id, wind_field_hash, parameters JSONB, result JSONB containing deposition_grid, buffer_distances, max_drift_m, status, computed_at)

**spray_records** (id, flight_plan_id, executed_at, actual_weather JSONB, equipment JSONB, product_name, epa_reg_number, total_area_treated_ha, total_amount_applied_l) — fulfilling federal record-keeping requirements under 7 CFR Part 110.

### Time-series weather data in Redis + Zarr

Weather forecast data should not be stored in PostgreSQL. Instead, the FastAPI service fetches HRRR data via Herbie into xarray datasets, extracts relevant point/area data, and caches the decoded arrays in Redis as serialized NumPy/binary blobs. For historical weather auditing (proving what forecast was available when a plan was approved), store weather snapshot references with hash verification.

### Audit trail architecture

Every flight plan state transition, drift simulation result, and compliance check should be logged to an append-only `audit_events` table with timestamps, actor_id, event_type, and JSONB payload. This supports both regulatory record-keeping (2-year retention for RUP applications) and potential legal defense of spray decisions.

---

## 6. Compliance framework and regulatory alignment

### What is required vs. recommended

**Required compliance:** FAA Part 107 + Part 137 operational constraints (Boreas should encode congested-area restrictions, operational boundaries). EPA pesticide label compliance — the label is the law under FIFRA. Buffer zone calculations must respect label-specific requirements. FAA Remote ID awareness (mandatory since March 2024). Federal/state spray record-keeping facilitation.

**Highly recommended:** Ag Data Transparent certification for farmer trust and market differentiation. ASTM F3201-24 (Ensuring Dependability of Software Used in UAS) — the most directly applicable software standard for Boreas, lighter-weight than DO-178C. Professional liability (E&O) insurance for the software provider.

**Not required:** DO-178C certification — it applies to airborne flight control software, not ground-based planning tools. Even if applied, Boreas would be DAL D or DAL E (minimal to no objectives). No drone planning platform (DJI, DroneDeploy) claims DO-178C compliance.

### Flight plan export formats for maximum compatibility

Boreas should export in four formats: **MAVLink WPL** (QGroundControl format) for ArduPilot/PX4 compatibility. **KMZ** for DJI enterprise drones via DJI Pilot 2. **GeoJSON** for web visualization and UTM integration. **CSV/JSON** for human-readable compliance records and farm management system integration.

### OGC EDR as the weather interface standard

The OGC API – Environmental Data Retrieval standard (v1.1, with v1.2 in development) provides RESTful point, area, and trajectory queries returning CoverageJSON — a web-friendly JSON format for multidimensional spatiotemporal data. NOAA's MDL has a production prototype EDR server. OGC EDR Part 2 (Pub/Sub) was adopted in 2024, enabling push-based weather update notifications. Boreas should implement the WeatherDataPort adapter pattern to support both OGC EDR providers and direct HRRR access via Herbie, ensuring resilience as the EDR ecosystem matures.

---

## 7. Technology decision matrix

| Decision | Choice | Primary Justification | Alternatives Considered |
|---|---|---|---|
| Frontend framework | Next.js 14+ (App Router) | SSR, React ecosystem, ISR for static content | SvelteKit (smaller but weaker ecosystem) |
| Map engine | Mapbox GL JS | Best 2D geospatial rendering, custom data layers | MapLibre GL JS (open source fork, viable alternative) |
| 3D visualization | Three.js with WebGPU renderer | Lighter than CesiumJS for focused drift visualization | CesiumJS (overkill for this use case) |
| Computation backend | FastAPI (Python 3.12+) | MetPy, xarray, Herbie, NumPy ecosystem; async I/O | Rust Axum (faster but lacks met libraries) |
| Physics core | Rust (dual-target: PyO3 + WASM) | One codebase for server and browser; near-native performance | Pure Python (too slow for client); Pure JS (no type safety) |
| Database | PostgreSQL + PostGIS | Spatial queries, mature, well-supported | MongoDB (weaker spatial, no ACID) |
| Cache/messaging | Redis (cache + Streams + Pub/Sub) | Already needed for weather cache; eliminates extra infra | NATS (consider if scaling beyond 3 services) |
| Task queue | ARQ (async Redis queue) | Lightweight, uses existing Redis; Python-native | Celery (heavier, more features than needed initially) |
| Weather data | HRRR via Herbie + OGC EDR adapters | 3km resolution, hourly, free, cloud-optimized Zarr | ECMWF (higher cost, better global but US-focused product doesn't need it) |
| Client computation | Rust→WASM in Web Workers via Comlink | 1.5× native performance, zero-copy transfers | Pure JS Web Workers (2-3× slower) |
| Drift model (fast) | Pre-computed lookup tables | O(1) query, <1ms, interactive | Always-on Lagrangian (too slow for scrubbing) |
| Drift model (detailed) | Simplified Lagrangian (1K–5K particles) | Planning-grade accuracy in <500ms | Full AGDISP (proprietary, not embeddable) |
| Path planning | Rust port of Fields2Cover algorithms | BSD-3, modular, benchmarked, wind-aware extensions | Custom from scratch (unnecessary risk) |
| Deployment | Vercel (frontend) + Railway/Fly.io (backend) | Separation of concerns; each optimized | All-AWS (more complex, premature optimization) |

---

## 8. Implementation roadmap

### Phase 1: Foundation (weeks 1–8)

**Objective:** Core platform with basic drift estimation and field management.

Set up the monorepo (Turborepo recommended) with three packages: `@boreas/web` (Next.js), `@boreas/api` (FastAPI), `@boreas/physics` (Rust library). Implement field boundary management with Mapbox GL JS drawing tools and PostGIS storage. Build the weather data pipeline: Herbie-based HRRR fetching → Redis caching → REST API endpoint returning wind speed/direction for a given lat/lon/time. Implement Tier 1 drift estimation (lookup tables) in TypeScript for immediate interactivity. Basic flight plan creation with boustrophedon coverage paths (simple parallel swaths, user-defined swath angle). Export to MAVLink WPL and KMZ formats.

### Phase 2: Physics engine (weeks 9–16)

**Objective:** Scientifically validated drift computation with browser-native performance.

Build the Rust physics core: Lagrangian particle tracking with droplet drag (Schiller-Naumann), gravity, and basic evaporation model. Compile to WASM and integrate into Web Worker with Comlink. Implement log wind profile extrapolation with surface roughness lookup by crop type. Add Tier 2 drift simulation (1,000-particle Lagrangian) with real-time visualization on Mapbox custom layer. Build wind-aware path optimization: swath angle as function of wind direction, upwind-to-downwind sequencing, overlap adjustment for crosswind drift. Implement compliance validation service with EPA buffer zone rules.

### Phase 3: Enterprise features (weeks 17–24)

**Objective:** Production-ready platform with full meteorological integration and compliance.

Add OGC EDR weather adapter (alongside direct Herbie access). Implement SSE-based weather update notifications and cache invalidation. Build Tier 3 Monte Carlo simulation with async execution and progressive result streaming via WebSocket. Add Three.js 3D drift plume visualization with WebGPU compute shaders for particle rendering. Implement spray record generation meeting federal/state requirements. Build multi-field planning with TSP-based route optimization. Add Pasquill-Gifford atmospheric stability estimation and Monin-Obukhov stability corrections.

### Phase 4: Scale and certify (weeks 25–32)

**Objective:** Market readiness with data trust and performance at scale.

Pursue Ag Data Transparent certification. Implement ASTM F3201-24 software dependability practices. Add service worker offline support for field operations. Performance optimization: CDN weather tile distribution, WebGPU drift rendering, worker pool tuning. Build multi-user collaboration features and organization management. Integrate with DJI AGRAS SDK for direct drone upload. Add historical weather replay for post-incident analysis. Begin ASTM F3548 UTM integration preparation.

---

## 9. Risk assessment and mitigation

### Technical risks

**WASM complexity delays physics engine delivery.** Mitigation: Phase 1 uses pure TypeScript lookup tables. Server-side Python drift simulation (via PyO3-bound Rust) provides full accuracy without WASM. Client WASM is a Phase 2 optimization, not a blocker.

**OGC EDR provider instability.** Mitigation: Port/adapter pattern enables fallback to direct HRRR access via Herbie (proven, stable) or Open-Meteo's free GFS+HRRR API. Never depend on a single weather data path.

**HRRR 3km resolution insufficient for microclimate effects.** Mitigation: HRRR resolution is adequate for the open agricultural fields where drone spraying occurs. Terrain-induced microclimate effects (shelterbelts, buildings) operate at scales below any operational weather model. Acknowledge this limitation transparently — even professional ag aviators rely on on-site wind observations for final go/no-go decisions. Add optional integration with field-deployed weather stations.

### Scientific risks

**Drift model accuracy questioned in legal disputes.** Mitigation: Implement models traceable to AGDISP/AgDRIFT peer-reviewed literature. Include clear disclaimers that predictions are decision-support tools, not guarantees. Maintain complete audit trails of inputs and outputs. The March 2026 AGDISP modernization effort (with Bayer Crop Science) validates that even the regulatory community recognizes current models need updating — Boreas's approach of transparent, physics-based modeling with documented assumptions is defensible.

**UAV-specific wake effects not captured.** Mitigation: For multi-rotor UAVs at typical spray altitudes (2–5m AGL), rotor downwash actually pushes droplets downward, reducing drift compared to fixed-wing aircraft. The simplified Lagrangian model that ignores wake effects will be slightly conservative (overpredicting drift) for UAV applications — a safe default. CHARM+AGDISP provides the scientific basis for future UAV-specific refinement.

### Market risks

**Regulatory framework for drone spraying still evolving.** Mitigation: The UAPASTF (consortium of 9 major crop protection companies) is actively generating drone-specific drift data for EPA. Boreas's modular compliance context can adapt to new regulations as they emerge. The platform's ability to incorporate drone-specific parameters positions it ahead of competitors still using fixed-wing assumptions.

**Seasonal usage patterns create infrastructure cost spikes.** Mitigation: Railway/Fly.io scale-to-zero capabilities minimize costs during off-season. Redis and PostgreSQL can use managed services with automatic scaling. The compute-heavy client-side architecture (WASM/WebGPU) offloads peak computational load to user devices rather than servers.

---

## Conclusion

Boreas occupies a genuine gap in the agricultural technology landscape. No existing drone flight planning platform — neither DJI's built-in tools nor DroneDeploy's more sophisticated platform — incorporates **real-time wind-aware drift computation into the flight path optimization loop**. The scientific foundation is mature (AGDISP has 40+ years of development; HRRR provides 3km hourly wind data for free), the computational tools exist (Rust→WASM achieves 65% of native speed in the browser; Fields2Cover provides BSD-licensed coverage planning), and the regulatory environment is actively evolving toward drone-specific drift assessment.

The critical architectural insight is that **the physics must be shared between server and client** through the dual-target Rust core. This isn't merely an engineering preference — it's a product requirement. Interactive drift preview demands sub-100ms client computation; compliance reporting demands server-side auditability; and both must produce identical results for user trust. The hybrid FastAPI+Next.js architecture, with Python's scientific ecosystem handling meteorological data and Rust handling physics, resolves the fundamental tension between scientific computing capability and browser performance.

The greatest near-term opportunity is the AGDISP modernization initiative. With Bayer Crop Science joining this effort in March 2026, the regulatory community is signaling openness to updated, technology-aware drift models. Boreas should engage with this process — not just consuming its outputs, but potentially contributing UAV-specific validation data that strengthens both the platform's credibility and the broader regulatory framework.