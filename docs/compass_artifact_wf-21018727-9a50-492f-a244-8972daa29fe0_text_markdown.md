# Boreas: engineering the wind-aware spray planning stack

**The core challenge of Boreas is executing a shared Rust physics engine across browser and server, fed by real-time HRRR wind data, to produce drift-compensated agricultural flight plans in under a second.** This guide provides implementation-level detail for the four foundational systems: a dual-target Lagrangian drift engine, wind-aware coverage path algorithms, a weather data pipeline, and the end-to-end integration that stitches them together. Every formula, data structure, and architectural decision below is chosen for a solo-developer MVP that can scale to production.

---

## AREA 1: Rust drift engine internals

### The equations of motion for spray droplets

A Lagrangian particle tracker solves two coupled ODEs for each droplet. The position equation is **dX/dt = V_wind(z) + V_turbulence + V_droplet**, where V_wind is the local mean wind interpolated at the droplet's current height, V_turbulence is a stochastic perturbation, and V_droplet is the droplet's own velocity relative to the air. The momentum equation is **dV_droplet/dt = F_drag + F_gravity + F_buoyancy**, which for a spherical water droplet in air simplifies to:

```
dv/dt = -(3/4) * (ρ_air / ρ_water) * (C_D / d) * |v_rel| * v_rel + g * (1 - ρ_air/ρ_water)
```

where `v_rel = v_droplet - v_air`, `d` is droplet diameter, and `C_D` is the drag coefficient. For agricultural spray droplets (**50–600 μm diameter**), the particle Reynolds number ranges from **~0.01 for fine mist to ~20 for coarse spray**, placing most droplets in the Schiller-Naumann regime rather than pure Stokes flow.

**Integration scheme recommendation: RK4 with adaptive timestep.** Euler integration introduces ~5% error per step at Δt = 0.01s for this problem, while RK4 achieves the same accuracy at Δt = 0.05s — a 5× reduction in steps. For the WASM Tier 1 lookup table generation, use fixed-step RK4 with **Δt = 0.005s** (a 100 μm droplet at terminal velocity falls ~3m, so ~600 steps for a 3m release height). For the Tier 2 Monte Carlo server simulation, use RK4 with **Δt = 0.01s** and halve when Re > 10.

### Schiller-Naumann drag and terminal velocity

The Schiller-Naumann correlation covers **Re < 1000** (all agricultural spray conditions):

```
C_D = (24 / Re) * (1 + 0.15 * Re^0.687)
```

The Reynolds number for a falling droplet is **Re = ρ_air × |v_rel| × d / μ_air**, with typical values: ρ_air ≈ 1.2 kg/m³, μ_air ≈ 1.81 × 10⁻⁵ Pa·s. For Re < 1 (droplets below ~80 μm), Stokes drag C_D = 24/Re suffices and the terminal velocity has a closed-form solution: **V_t = ρ_water × g × d² / (18 × μ_air)**.

Terminal velocity for larger droplets requires iteration because C_D depends on v which depends on C_D. The standard algorithm converges in **3–5 iterations**:

```
ALGORITHM: TerminalVelocity(d, ρ_p, ρ_f, μ)
  1. Initial guess: V = ρ_p * g * d² / (18 * μ)        // Stokes estimate
  2. Loop (max 10 iterations):
     Re = ρ_f * V * d / μ
     C_D = (24/Re) * (1 + 0.15 * Re^0.687)
     V_new = sqrt(4 * d * (ρ_p - ρ_f) * g / (3 * C_D * ρ_f))
     if |V_new - V| / V < 1e-6: break
     V = V_new
  3. Return V
```

**Reference terminal velocities at 20°C, sea level:** 100 μm → **0.27 m/s** (Re ≈ 0.18, Stokes valid), 200 μm → **0.72 m/s** (Re ≈ 0.96), 400 μm → **1.67 m/s** (Re ≈ 4.5), 600 μm → **2.45 m/s** (Re ≈ 9.8). These should be pre-computed and stored in the lookup table.

### Turbulence parameterization via Pasquill-Gifford

Agricultural spraying occurs in the atmospheric surface layer (0–50m), where turbulence intensity depends on stability class. The **Langevin equation** approach is superior to discrete random walk for smooth trajectories:

```
v'(t + Δt) = v'(t) * exp(-Δt/T_L) + σ * sqrt(1 - exp(-2Δt/T_L)) * N(0,1)
```

where `v'` is the turbulent velocity fluctuation, `T_L` is the Lagrangian integral timescale (**~10s for neutral conditions over crops**), σ is the turbulence standard deviation, and N(0,1) is a standard normal random variate.

Pasquill-Gifford stability classes map to turbulence sigmas at 10m height over agricultural terrain (z₀ ≈ 0.05–0.1m for low crops, 0.5–1.0m for corn/mature crops):

| Stability | Conditions | σ_u (m/s) | σ_v (m/s) | σ_w (m/s) | T_L (s) |
|-----------|-----------|-----------|-----------|-----------|---------|
| **A** (very unstable) | Strong solar, light wind | 0.35U | 0.35U | 0.35U | 5–15 |
| **B** (moderately unstable) | Moderate solar | 0.25U | 0.25U | 0.20U | 10–20 |
| **C** (slightly unstable) | Weak solar, moderate wind | 0.20U | 0.15U | 0.12U | 15–30 |
| **D** (neutral) | Overcast, moderate wind | 0.15U | 0.10U | 0.08U | 20–50 |
| **E** (slightly stable) | Night, moderate wind | 0.10U | 0.08U | 0.04U | 30–100 |
| **F** (stable) | Night, light wind | 0.05U | 0.04U | 0.02U | 50–200 |

Here U is the mean wind speed at 10m. **Spray application typically occurs in classes B-D** (moderate instability to neutral). For a 3 m/s wind in class C: σ_u ≈ 0.6, σ_v ≈ 0.45, σ_w ≈ 0.36 m/s.

### Droplet evaporation and when to skip it

The d²-law states **d²(t) = d₀² - K × t**, where K is the evaporation rate constant. For water droplets in air, K depends on temperature and humidity via:

```
K = (8 × D_v × M_w × Δp) / (ρ_water × R × T_mean)
```

where D_v ≈ 2.5 × 10⁻⁵ m²/s (water vapor diffusivity), M_w = 0.018 kg/mol, and Δp is the vapor pressure deficit. At **25°C and 50% RH**, K ≈ **1.1 × 10⁻⁹ m²/s**. A 200 μm droplet reaches the ground from 3m in ~4s, losing only ~5 μm of diameter. A 100 μm droplet falling the same distance loses ~20 μm — now 80 μm, with significantly altered drift. **Include evaporation for droplets < 200 μm; skip for ≥ 200 μm.** This is the threshold used by AGDISP/AgDRIFT.

### Monte Carlo envelope generation

Run **N = 1000 particles per droplet size class** with stochastic turbulence, recording each particle's ground-impact (x, y) position. For 5 size classes (ASABE Fine, Medium, Coarse bins) this gives 5000 total trajectories. Compute drift distance as the downwind displacement. Sort distances and extract **P50 (median), P90, P99** percentiles. The P90 envelope is the standard for regulatory buffer zones.

For statistical convergence of P99, **N = 5000 per class** is recommended — the 99th percentile of 5000 samples has a coefficient of variation of ~10%. Run as independent particles (embarrassingly parallel). In Rust, **rayon's par_iter** distributes across all cores:

```rust
use rayon::prelude::*;
let results: Vec<f64> = (0..5000)
    .into_par_iter()
    .map(|_| simulate_single_droplet(&params, &mut thread_rng()))
    .collect();
```

On a 4-core machine, 5000 particles × 5 classes completes in **~50ms native** (each particle trace is ~500 RK4 steps × ~20 FLOPs = ~50μs). In WASM (single-threaded), expect **~200ms**.

### Dual-target Rust architecture with PyO3 and wasm-bindgen

The workspace layout separates pure physics from platform bindings:

```
boreas-drift/
├── Cargo.toml                    # workspace
├── crates/
│   ├── drift-core/               # Pure Rust: equations, integration, lookup
│   │   ├── Cargo.toml            # depends on: ndarray, rand
│   │   └── src/lib.rs
│   ├── drift-python/             # PyO3 bindings
│   │   ├── Cargo.toml            # depends on: drift-core, pyo3, numpy
│   │   ├── pyproject.toml        # maturin config
│   │   └── src/lib.rs
│   └── drift-wasm/               # wasm-bindgen bindings
│       ├── Cargo.toml            # depends on: drift-core, wasm-bindgen, js-sys
│       └── src/lib.rs
```

**Critical Cargo.toml pattern** for drift-python — PyO3 must never be pulled into the WASM build:

```toml
[dependencies]
drift-core = { path = "../drift-core" }
pyo3 = { version = "0.23", features = ["extension-module"] }
numpy = "0.23"

[lib]
crate-type = ["cdylib"]
```

For the WASM crate, rayon falls back to sequential. Use conditional compilation in drift-core:

```rust
#[cfg(not(target_arch = "wasm32"))]
use rayon::prelude::*;

pub fn simulate_monte_carlo(params: &SimParams, n: usize) -> Vec<DriftResult> {
    #[cfg(not(target_arch = "wasm32"))]
    let iter = (0..n).into_par_iter();
    #[cfg(target_arch = "wasm32")]
    let iter = (0..n).into_iter();
    
    iter.map(|_| simulate_single(params)).collect()
}
```

Build commands: `maturin build --release` for Python wheels; `wasm-pack build --target web crates/drift-wasm/` for the npm package. The WASM module runs in a **Web Worker** to avoid blocking the UI thread, communicating via `postMessage` with Transferable `ArrayBuffer` objects.

### Floating-point determinism between targets

WASM is **more deterministic** than native — the WebAssembly spec prohibits FMA contraction and requires IEEE 754 round-to-nearest. Native Rust/LLVM may fuse multiply-add operations, changing results by 1-2 ULP. To ensure identical results:

1. **Disable FMA on native** via `.cargo/config.toml`: `rustflags = ["-C", "llvm-args=-fp-contract=off"]`
2. **Avoid transcendental functions** (sin, cos, exp) that use platform-specific implementations — use polynomial approximations from a shared Rust crate instead
3. **Use deterministic RNG** (e.g., `rand_chacha::ChaCha8Rng`) seeded identically on both targets
4. Accept that **NaN bit patterns may differ** (the only WASM nondeterminism) — but NaN should never appear in valid simulation

### Lookup table design for O(1) drift queries

Pre-compute a 5D table parameterized by: **droplet_class** (5 ASABE bins: VF, F, M, C, VC), **wind_speed** (0.5–10 m/s, 20 steps), **release_height** (1–5m, 9 steps), **temperature** (10–40°C, 7 steps), **humidity** (20–90%, 8 steps). Total entries: 5 × 20 × 9 × 7 × 8 = **50,400**. Each entry stores: drift_distance_p50, drift_distance_p90, drift_distance_p99, fall_time. At 4 floats × 4 bytes = 16 bytes per entry, total size is **~800 KB** — easily fits in WASM memory.

For real-time query, use **multilinear interpolation** across the continuous dimensions (wind, height, temp, humidity), with nearest-neighbor for the discrete droplet class. The interpolation formula for 4 continuous dimensions requires 2⁴ = 16 lookups and 15 lerp operations — still **< 1 μs** per query.

---

## AREA 2: Wind-aware flight path algorithms

### Fields2Cover as the algorithmic foundation

Fields2Cover (BSD-3, C++17 with Python/SWIG bindings) implements the full coverage path planning pipeline in four modular stages. **Headland Generator** buffers the field polygon inward by `n × robot_width` (typically 3× for turning room). **Swath Generator** creates parallel lines at the configured coverage width and brute-force searches all angles in [0°, 180°) at a configurable step (default 1°) to minimize total swath length or number of swaths. **Route Planner** sequences the swaths — boustrophedon (sequential), snake (skip-one alternating), or spiral patterns. **Path Planner** connects swath endpoints with Dubins curves or Reeds-Shepp curves respecting minimum turn radius.

For Boreas, **wrap Fields2Cover's Python bindings on the server** for authoritative path generation, and reimplement the geometric core (swath generation + wind offset) in Rust/WASM for instant client-side preview. The four-module architecture maps cleanly:

```python
import fields2cover as f2c

robot = f2c.Robot(2.0, 6.0)          # width=2m, coverage_width=6m (DJI T40)
robot.setMinTurningRadius(1.5)        # multirotor can turn tight

field = f2c.Cell()                    # from GeoJSON polygon
const_hl = f2c.HG_Const_gen()
inner = const_hl.generateHeadlands(field, 3.0 * robot.getWidth())

bf = f2c.SG_BruteForce()
swaths = bf.generateBestSwaths(robot.getCovWidth(), inner.getGeometry(0))

boustro = f2c.RP_Boustrophedon()
sorted_swaths = boustro.genSortedSwaths(swaths)

dubins = f2c.PP_DubinsCurves()
path = f2c.PP_PathPlanning().planPath(robot, sorted_swaths, dubins)
```

### Wind-aware swath angle optimization

The optimal swath angle is **perpendicular to wind direction**, ensuring spray drifts along (not across) the flight line. Given wind direction α (compass bearing the wind blows FROM):

```
optimal_swath_angle = (α + 90°) mod 360°
```

When the wind shifts during a multi-hour operation, recompute the angle for each time block and accept a compromise angle that minimizes worst-case cross-track drift across the planning window. The cross-track drift for a given flight angle θ is:

```
crosswind_drift = drift_distance × |sin(θ - α)|
```

Override Fields2Cover's brute-force angle search by passing the wind-optimal angle directly.

### Drift offset and effective swath width

Given spray fall time `t_fall ≈ h / V_terminal` and wind speed `U_spray_height` (extrapolated from 10m via log profile), the **drift vector** in local ENU coordinates is:

```
drift_x = U_spray × sin(α_rad) × t_fall    # East component
drift_y = U_spray × cos(α_rad) × t_fall    # North component
```

Each swath line is **shifted upwind** by negating this vector: `shifted_line = translate(original_line, -drift_x, -drift_y)`. After shifting, clip to the field polygon boundary using `Shapely.intersection()` on the server or `turf.lineIntersect()` on the client.

**Effective swath width** determines line spacing: `effective = nozzle_swath - |crosswind_drift|`. When flying perpendicular to wind, crosswind_drift ≈ 0 and effective ≈ nozzle_swath. **If effective ≤ 0, emit a NO-FLY warning** — the wind is too strong for coverage. Swath spacing is then: `spacing = effective × (1 - overlap)`, where overlap is typically **0.1–0.3** (10–30%).

### Upwind-to-downwind pass sequencing

Spray passes must proceed from upwind to downwind to avoid flying through your own spray cloud. Sort swath midpoints by their projection onto the **downwind** direction vector:

```
downwind = (sin(α + 180°), cos(α + 180°))
score(swath) = midpoint.x × downwind.x + midpoint.y × downwind.y
sort swaths by score ascending  // first = most upwind
```

### Dubins paths for multirotor turns

Multirotor drones have a practical minimum turn radius of **1–3 meters** (much tighter than fixed-wing). Dubins paths connect two oriented waypoints (x, y, θ) with the shortest path composed of **arcs and line segments**. Six candidate paths exist (LSL, LSR, RSL, RSR, RLR, LRL); compute all six and select the shortest valid one. Fields2Cover implements this natively. For headland efficiency, the boustrophedon pattern alternates direction on adjacent swaths, producing simple U-turns that Dubins handles optimally.

### Polygon decomposition for non-convex fields

Fields2Cover v2.0 supports **boustrophedon cell decomposition** (BCD), which sweep-decomposes non-convex polygons into convex or y-monotone cells. Each cell gets independent boustrophedon coverage. An adjacency graph determines cell traversal order (minimize deadhead travel via TSP heuristic). Fields with holes (ponds, buildings) are handled by subtracting hole polygons from the field before decomposition — `Shapely.difference()` handles this cleanly.

### Coordinate systems and projection

All geometric operations (buffering, line spacing, drift offset) must use **metric coordinates**. Convert WGS84 lat/lon to **UTM** for computation, then convert results back:

```python
from pyproj import Transformer
utm_zone = int((lon + 180) / 6) + 1
to_utm = Transformer.from_crs("EPSG:4326", f"EPSG:326{utm_zone:02d}", always_xy=True)
x, y = to_utm.transform(lon, lat)
```

On the client, **Turf.js** handles buffer, intersection, and centroid operations in WGS84 with internal projection, but for sub-meter precision (required for swath placement), project explicitly.

---

## AREA 3: Weather data pipeline

### HRRR data fundamentals

The High-Resolution Rapid Refresh model produces **3 km resolution** forecasts over CONUS on a **1059 × 1799 grid** using Lambert Conformal Conic projection (lat₁ = lat₂ = lat₀ = 38.5°, lon₀ = -97.5°, **spherical earth R = 6,371,229m**). It runs every hour with 0–18h forecasts (0–48h for 00/06/12/18Z cycles). The variables needed for Boreas are **UGRD and VGRD at 10m above ground**, plus **TMP at 2m** and **RH at 2m** for evaporation modeling.

**Critical detail:** HRRR wind components are **grid-relative**, not earth-relative. To get true meteorological U/V, rotate by the convergence angle:

```python
ROTCON = math.sin(math.radians(38.5))  # ≈ 0.6225
angle = ROTCON * (lon - (-97.5)) * math.pi / 180
u_true = math.cos(angle) * u_grid + math.sin(angle) * v_grid
v_true = -math.sin(angle) * u_grid + math.cos(angle) * v_grid
```

### Recommended pipeline: Herbie + Redis for MVP

For a solo developer, **Herbie (v2026.3.0)** is the pragmatic choice. It downloads only the needed GRIB2 messages via HTTP range requests (~2–5 MB per variable per timestep from AWS), caches locally, and returns xarray datasets:

```python
from herbie import Herbie

H = Herbie('2026-03-17 12:00', model='hrrr', product='sfc', fxx=6)
ds_u = H.xarray(":UGRD:10 m above ground:")
ds_v = H.xarray(":VGRD:10 m above ground:")
# Each is shape (1059, 1799) with 2D lat/lon coordinates
```

Wrap Herbie in a FastAPI service with Redis caching:

```python
class WindService:
    def __init__(self):
        self.cache = redis.Redis()
        self.TTL = 7200  # 2 hours
    
    async def get_wind(self, lat: float, lon: float, 
                       run: str, fxx: int) -> WindData:
        key = f"hrrr:{run}:f{fxx:02d}:wind:{lat:.3f}:{lon:.3f}"
        cached = self.cache.get(key)
        if cached:
            return WindData.from_json(cached)
        
        # Fetch from Herbie, interpolate, cache
        u_grid, v_grid = self._fetch_wind_field(run, fxx)
        u, v = self._interpolate_to_point(u_grid, v_grid, lat, lon)
        result = WindData(u=u, v=v, speed=hypot(u,v), 
                         direction=atan2(-u, -v) * 180/pi % 360)
        self.cache.setex(key, self.TTL, result.to_json())
        return result
```

### Bilinear interpolation from HRRR grid

Because the HRRR Lambert Conformal grid has **uniform 3 km spacing in projected coordinates**, interpolation is trivial once you transform the target lat/lon to projected (x, y):

```python
from pyproj import Transformer

hrrr_proj = '+proj=lcc +lat_1=38.5 +lat_2=38.5 +lat_0=38.5 +lon_0=-97.5 +a=6371229 +b=6371229'
to_hrrr = Transformer.from_crs("EPSG:4326", hrrr_proj, always_xy=True)
x, y = to_hrrr.transform(lon, lat)

# Fractional grid indices
x_min, y_min = -2697520.142522, -1587306.152557
fi = (x - x_min) / 3000.0   # x-index
fj = (y - y_min) / 3000.0   # y-index

# Standard bilinear interpolation on the 4 surrounding points
i0, j0 = int(floor(fi)), int(floor(fj))
dx, dy = fi - i0, fj - j0
value = (data[j0,i0]*(1-dx)*(1-dy) + data[j0,i0+1]*dx*(1-dy) +
         data[j0+1,i0]*(1-dx)*dy + data[j0+1,i0+1]*dx*dy)
```

**Pitfall:** You must use the **spherical earth** (R = 6,371,229m) in the projection. Using the default WGS84 ellipsoid causes positioning errors up to **7 km at domain corners**.

### Log wind profile extrapolation to spray height

HRRR provides 10m wind. Drones spray at 2–5m AGL. The logarithmic wind profile extrapolates:

```
U(z) = U(z_ref) × ln(z / z₀) / ln(z_ref / z₀)
```

Surface roughness z₀ for agricultural land: bare soil **0.005m**, short grass/stubble **0.01–0.03m**, low crops (wheat) **0.04–0.1m**, tall crops (corn/mature) **0.5–1.0m**. For a 3 m/s wind at 10m over short crops (z₀ = 0.03m): U(3m) ≈ 3.0 × ln(3/0.03) / ln(10/0.03) ≈ 3.0 × 4.61/5.81 ≈ **2.38 m/s**. This 20% reduction matters for drift calculations.

### Alternative: Open-Meteo for zero-infrastructure MVP

Open-Meteo ingests HRRR and serves pre-interpolated point forecasts via a free REST API. One call returns all needed variables:

```
GET https://api.open-meteo.com/v1/gfs?
  latitude=39.74&longitude=-104.99
  &hourly=wind_speed_10m,wind_direction_10m,temperature_2m,relative_humidity_2m
  &models=ncep_hrrr_conus&forecast_days=2
```

This eliminates GRIB2 decoding, projection math, and grid interpolation entirely. **Trade-off:** no spatial fields (needed for mapping wind across the field), no custom interpolation, **10,000 requests/day rate limit**, and you depend on a third-party service. Use Open-Meteo for MVP; migrate to Herbie + Redis when you need spatial wind visualization or higher throughput.

### Trade-off summary

| Approach | Complexity | Latency | Cost | Best for |
|----------|-----------|---------|------|----------|
| **Open-Meteo API** | Trivial (HTTP GET) | < 100ms | Free (non-commercial) | MVP, prototyping |
| **Herbie + Redis** | Moderate (projection, interpolation) | 1-5s cold, < 10ms cached | Free (AWS public data) | Production with caching |
| **Direct Zarr (s3://hrrrzarr/)** | High (S3, xarray, chunking) | 200ms–2s per variable | AWS egress | Spatial analysis, ML |
| **Self-hosted pipeline** | Very high (MinIO, scheduler, EDR API) | < 10ms (local) | Server costs | Enterprise, air-gapped |

---

## AREA 4: End-to-end data flow

### Step 1 → Polygon drawing (client, instant)

User draws a field boundary on Mapbox GL JS using `@mapbox/mapbox-gl-draw`. The `draw.create` event fires with a GeoJSON Feature:

```javascript
map.on('draw.create', (e) => {
  const polygon = e.features[0];  // GeoJSON Polygon
  validatePolygon(polygon);        // check self-intersection, min area
  store.dispatch(setFieldPolygon(polygon));
  triggerWindFetch(turf.centroid(polygon).geometry.coordinates);
});
```

**Validation:** Use `turf.kinks()` to detect self-intersections and reject polygons < 0.1 hectares. Data format: GeoJSON Feature with `geometry.type = "Polygon"`. Runs entirely client-side, **< 1ms**.

### Step 2 → Wind data fetch (client → server, ~1s)

Client sends the polygon centroid + selected forecast datetime to `POST /api/wind`:

```json
{
  "lat": 39.74, "lon": -104.99,
  "forecast_time": "2026-03-17T18:00:00Z",
  "spray_height_m": 3.0,
  "surface_roughness": "low_crop"
}
```

FastAPI endpoint queries the WindService (Herbie + Redis or Open-Meteo), extrapolates to spray height via log profile, and returns:

```json
{
  "wind_speed_10m": 3.2,
  "wind_direction_10m": 225,
  "wind_speed_spray_height": 2.54,
  "wind_direction_spray_height": 225,
  "temperature_2m": 22.5,
  "humidity_2m": 45,
  "gust_speed_10m": 5.1,
  "stability_class": "C",
  "hrrr_run": "2026031712",
  "hrrr_fxx": 6
}
```

**Computation time:** ~10ms if cached, ~2–5s on cold Herbie fetch. **Async:** use FastAPI `async def` with `httpx` for non-blocking Herbie calls.

### Step 3 → Tier 1 drift preview (client WASM, ~5ms)

The WASM drift module loads a pre-computed **800 KB lookup table** into linear memory at initialization. When wind data arrives, the client-side Web Worker queries the table:

```javascript
// worker.js
import init, { lookup_drift } from './pkg/drift_wasm.js';
await init();

self.onmessage = (e) => {
  const { droplet_class, wind_speed, height, temp, rh } = e.data;
  const result = lookup_drift(droplet_class, wind_speed, height, temp, rh);
  // result: { p50_m: 2.1, p90_m: 5.4, p99_m: 12.7, fall_time_s: 1.8 }
  self.postMessage(result);
};
```

The result immediately updates the drift visualization on the map — a gradient polygon extending downwind from the field boundary, colored by percentile. **Total latency: < 5ms** including Worker message overhead. This runs every time the user changes any parameter (wind, height, nozzle), providing real-time interactive feedback.

### Step 4 → Operation configuration (client, sync)

User selects aircraft (DJI Agras T40: 16m swath), nozzle type (XR 11002 flat fan), spray height (3m AGL), buffer distance (30m from sensitive areas), overlap (15%). These are stored in React state and passed to the path generator.

### Step 5 → Flight path generation (server, ~200ms)

Client sends full configuration to `POST /api/flight-plan/generate`:

```json
{
  "field_polygon": { /* GeoJSON */ },
  "wind": { "speed": 2.54, "direction": 225 },
  "drift": { "p90_distance": 5.4, "dx": -3.8, "dy": -3.8 },
  "aircraft": { "swath_width": 16, "min_turn_radius": 1.5, "speed": 6 },
  "overlap": 0.15,
  "buffer_distance": 30,
  "spray_height": 3
}
```

Server-side algorithm sequence:

1. **Project** field polygon to UTM (~1ms)
2. **Buffer** sensitive areas and subtract from field (~5ms, Shapely)
3. **Compute headlands** via Fields2Cover HG_Const_gen (~2ms)
4. **Compute effective swath** = 16 - |crosswind_drift| = 16 - 0 = 16m (perpendicular to wind), spacing = 16 × 0.85 = 13.6m (~1ms)
5. **Generate swaths** at wind-perpendicular angle via Fields2Cover SG_BruteForce (~20ms)
6. **Offset swaths** upwind by drift vector, clip to field (~10ms, Shapely translate + intersection)
7. **Sort swaths** upwind-to-downwind (~1ms)
8. **Generate Dubins turns** via Fields2Cover PP_DubinsCurves (~10ms)
9. **Reproject** waypoints back to WGS84 (~1ms)
10. **Add spray on/off** commands at swath start/end (~1ms)

**Total: ~50–200ms.** Return as GeoJSON FeatureCollection with properties: waypoint type (transit/spray/turn), altitude, speed, spray_state.

### Step 6 → Tier 2 Lagrangian simulation (server async, ~2s)

Triggered by `POST /api/drift/simulate` as a **background task** (FastAPI BackgroundTasks or Redis Queue for production):

```python
@app.post("/api/drift/simulate")
async def simulate_drift(request: DriftSimRequest, background_tasks: BackgroundTasks):
    job_id = str(uuid4())
    background_tasks.add_task(run_lagrangian_sim, job_id, request)
    return {"job_id": job_id, "status": "running"}
```

The Rust drift engine (via PyO3) runs 5000 particles across 5 droplet classes with the full turbulence model. Inputs: wind profile, temperature, humidity, stability class, spray height, droplet size distribution. Outputs: per-particle (x, y) ground impact points → aggregated into P50/P90/P99 contour polygons.

Client polls `GET /api/drift/simulate/{job_id}` or receives updates via **Server-Sent Events** (SSE):

```python
@app.get("/api/drift/simulate/{job_id}/stream")
async def stream_progress(job_id: str):
    async def event_generator():
        while True:
            status = redis.get(f"sim:{job_id}:status")
            yield f"data: {status}\n\n"
            if status == "complete": break
            await asyncio.sleep(0.5)
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

**Computation time:** ~500ms native with rayon parallelism on 4 cores. Including API overhead: ~1–2s end-to-end.

### Step 7 → Compliance check (server, ~50ms)

After path + drift generation, check against rules:

- **Wind speed limits:** Warning at > 3 m/s (7 mph), NO FLY at > 4.5 m/s (10 mph) per EPA label guidance
- **Temperature inversion check:** If temp_2m > temp_surface (from HRRR surface layer), warn of stable conditions trapping spray
- **Buffer zones:** PostGIS `ST_DWithin(flight_path, sensitive_area, buffer_m)` to verify all spray waypoints are ≥ buffer distance from water bodies, habitations, organic fields
- **Effective swath > 0:** Already checked in path generation
- **Battery feasibility:** total_distance / speed × power_consumption ≤ battery_capacity

Return pass/fail with specific violation details. **Format:** array of `{ rule, status, message, severity }`.

### Step 8 → Coverage heatmap (server, ~100ms)

Rasterize the spray deposition pattern onto a grid (1m resolution). For each swath line, project the nozzle spray pattern (Gaussian cross-track distribution with σ = swath_width/4) accounting for wind-induced displacement. Sum overlapping contributions. Compute **coefficient of variation (CV)** = σ_deposition / mean_deposition — CV < 15% is good uniformity.

Return as a GeoTIFF or as a vector grid of colored cells for Mapbox rendering.

### Step 9 → Best spray window (server, ~500ms)

Score each hour in the 48-hour HRRR forecast:

```python
def score_hour(wind, temp, rh, rain_prob):
    score = 100
    if wind > 4.5: return 0               # NO FLY
    score -= max(0, (wind - 2.0)) * 20    # Penalize wind > 2 m/s
    score -= max(0, (30 - temp)) * 1      # Penalize cold
    score -= max(0, (temp - 35)) * 3      # Penalize hot (evaporation)
    score -= max(0, (70 - rh)) * 0.5      # Penalize low humidity
    score -= rain_prob * 50               # Penalize rain chance
    if temp < 10 and wind < 1: score -= 30  # Inversion risk
    return max(0, score)
```

Return a 48-element array rendered as a **timeline chart** with color-coded spray suitability (green/yellow/red).

### Step 10 → Map visualization (client)

Render three layers on Mapbox GL JS:
- **Flight path:** GeoJSON LineString source → line layer with directional arrows, colored by spray_state (green = spraying, gray = transit)
- **Drift plume:** GeoJSON Polygon source → fill layer with opacity gradient from P50 (30% opacity) to P99 (10% opacity), colored orange-to-red
- **Coverage heatmap:** Raster source or fill-extrusion layer showing deposition uniformity

For 3D drift visualization, use a **Mapbox custom layer** with Three.js rendering particle traces in 3D space above the map.

### Step 11 → Database persistence

When user approves the plan, store to PostgreSQL + PostGIS:

```sql
CREATE TABLE flight_plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  field_polygon GEOMETRY(Polygon, 4326),
  flight_path GEOMETRY(LineString, 4326),
  drift_envelope_p90 GEOMETRY(Polygon, 4326),
  weather_snapshot JSONB,          -- wind, temp, humidity at plan time
  operation_config JSONB,          -- aircraft, nozzle, height, overlap
  drift_assessment JSONB,          -- p50/p90/p99, compliance results
  coverage_cv FLOAT,
  best_window_scores FLOAT[],
  status TEXT DEFAULT 'draft',     -- draft, approved, exported, completed
  created_at TIMESTAMPTZ DEFAULT now(),
  approved_at TIMESTAMPTZ,
  approved_by UUID
);

CREATE TABLE audit_events (
  id BIGSERIAL PRIMARY KEY,
  flight_plan_id UUID REFERENCES flight_plans(id),
  event_type TEXT,                  -- created, modified, approved, exported, weather_updated
  event_data JSONB,
  actor_id UUID,
  timestamp TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_audit_plan ON audit_events(flight_plan_id);
```

### Step 12 → Export formats

**MAVLink WPL** (QGroundControl / ArduPilot):
```
QGC WPL 110
0	1	3	22	0	0	0	0	39.7392	-104.9903	10	1
1	0	3	16	0	5	0	0	39.7393	-104.9895	3	1
2	0	3	183	1	1500	0	0	0	0	0	1     // DO_SET_SERVO: spray ON
...
```

Key MAV_CMD values: **16** = NAV_WAYPOINT, **22** = NAV_TAKEOFF, **21** = NAV_LAND, **20** = NAV_RETURN_TO_LAUNCH, **183** = DO_SET_SERVO (spray on/off), **178** = DO_CHANGE_SPEED.

**KMZ:** Wrap a KML document with path LineString, drift polygon, and field boundary inside a ZIP container. **GeoJSON:** direct export of the FeatureCollection used for map rendering. **CSV:** waypoint table with columns `index, lat, lon, alt, type, spray_state`.

### Step 13 → Audit trail

Every plan creation, modification, approval, export, and weather-condition-change is logged to `audit_events` with full JSONB snapshots. This supports regulatory traceability. Events are **append-only** (no UPDATE/DELETE on audit table). Include the HRRR model run identifier so the exact forecast used can be reconstructed.

---

## Key architectural decisions and recommendations

**Use Fields2Cover's Python bindings server-side** rather than reimplementing the full CPP pipeline in Rust. The coverage path planning problem has many edge cases (non-convex decomposition, headland corner handling) that Fields2Cover has already solved and validated in field experiments. Reserve Rust for the physics engine where performance matters.

**Start with Open-Meteo, graduate to Herbie.** The weather pipeline is the highest-ops-burden component. Open-Meteo eliminates it entirely for MVP. When you need spatial wind fields or hit rate limits, switch to Herbie + Redis — the interpolation code is ~100 lines of Python.

**The WASM drift lookup table is the key UX innovation.** It makes drift assessment feel instant (~5ms), turning what would be a server round-trip into a real-time interactive preview. Pre-compute the table offline (takes ~30 minutes for the full 50K entries) and ship it as a static asset alongside the WASM module.

**Floating-point determinism is achievable but requires discipline.** Disable FMA on native (`-fp-contract=off`), use deterministic RNG, and avoid platform-dependent transcendentals. If exact bit-identity proves too costly to maintain, accept < 0.1% relative difference between Tier 1 (WASM lookup) and Tier 2 (native Monte Carlo) — the stochastic nature of drift simulation already introduces larger variance than FMA rounding differences.

**The dual-target Rust architecture pays for itself.** A single physics core compiled to both PyO3 and WASM eliminates the "two implementations drift apart" problem that plagues similar systems. The workspace layout (core + python-bindings + wasm-bindings) is well-proven by projects like Hugging Face tokenizers and Polars.