# Boreas platform: core output implementation guide

**The five core outputs that Boreas delivers to pilots all have well-established implementation patterns.** Flight path geometry relies on a rotate-clip-rotate-back algorithm in Shapely; coverage quality uses Coefficient of Variation on a rasterized deposition grid; spray windows score hourly forecasts with weighted composites and group favorable hours; MAVLink WPL and DJI WPML/KMZ are both fully specified formats with known command encodings; and forecast timeline data is small enough to pre-compute entirely in memory for instant scrubbing. What follows is the complete implementation blueprint for each.

---

## Area 1: Flight path geometry — the rotate-clip-rotate-back algorithm

The standard approach for generating parallel swath lines at an arbitrary angle across any polygon is a five-step pipeline: **rotate the polygon to align the desired sweep direction with the horizontal axis**, generate evenly-spaced horizontal lines across the rotated bounding box, clip each line to the rotated polygon via `intersection()`, rotate the clipped segments back, then connect them into a boustrophedon path. This avoids trigonometric line equation complexity by reducing everything to simple horizontal sweeps.

### Projection first: WGS84 to UTM

All metric operations (buffering, spacing, line generation) must happen in a projected coordinate system. Use pyproj to determine the UTM zone and create bidirectional transformers:

```python
from pyproj import Transformer
from shapely.ops import transform as shapely_transform

def get_utm_epsg(lon, lat):
    zone = int((lon + 180) / 6) + 1
    return 32600 + zone if lat >= 0 else 32700 + zone

def project_to_utm(polygon_wgs84, lon, lat):
    epsg = get_utm_epsg(lon, lat)
    to_utm = Transformer.from_crs("EPSG:4326", f"EPSG:{epsg}", always_xy=True)
    to_wgs = Transformer.from_crs(f"EPSG:{epsg}", "EPSG:4326", always_xy=True)
    poly_utm = shapely_transform(to_utm.transform, polygon_wgs84)
    return poly_utm, to_utm, to_wgs
```

The `always_xy=True` flag is critical — without it pyproj uses the official EPSG axis order (lat, lon for EPSG:4326), silently swapping coordinates.

### Core swath generation

```python
import numpy as np
from shapely.geometry import Polygon, LineString, MultiPolygon
from shapely import affinity, prepare

def generate_swath_lines(polygon, spacing, angle_deg=0, headland_width=0):
    work_poly = polygon
    if headland_width > 0:
        work_poly = polygon.buffer(-headland_width, join_style='mitre')
        if work_poly.is_empty:
            return []
        if isinstance(work_poly, MultiPolygon):
            work_poly = max(work_poly.geoms, key=lambda p: p.area)

    centroid = work_poly.centroid
    rotated = affinity.rotate(work_poly, -angle_deg, origin=centroid)
    prepare(rotated)  # GEOS spatial index for faster batch intersections

    minx, miny, maxx, maxy = rotated.bounds
    y_positions = np.arange(miny + spacing / 2.0, maxy, spacing)

    swath_lines = []
    for y in y_positions:
        line = LineString([(minx - 1, y), (maxx + 1, y)])
        clipped = rotated.intersection(line)
        for seg in _extract_linestrings(clipped):
            if seg.length > 0.01:
                swath_lines.append(affinity.rotate(seg, angle_deg, origin=centroid))
    return swath_lines

def _extract_linestrings(geometry):
    if geometry.is_empty: return []
    if geometry.geom_type == 'LineString': return [geometry]
    if geometry.geom_type == 'MultiLineString': return list(geometry.geoms)
    if geometry.geom_type == 'GeometryCollection':
        return [g for g in geometry.geoms if g.geom_type in ('LineString', 'MultiLineString')]
    return []
```

Calling `prepare(rotated)` before the intersection loop activates GEOS's spatial index, yielding **significant speedup** for fields with hundreds of swath lines. For a 1-hectare field with 5m spacing, expect ~200 lines processed in under a second.

### Why non-convex polygons create complications

When a horizontal line crosses a concave polygon (L-shaped, U-shaped, fields with indentations), `intersection()` returns a `MultiLineString` — multiple disconnected segments per scan line. The critical decisions are ordering and filtering. **Sort segments by x-position within each scan line** (left-to-right in the rotated frame), filter any segment shorter than a minimum swath length (e.g., 2m), and treat each remaining segment as an independent swath requiring its own spray-on/spray-off triggers.

For complex field shapes, **boustrophedon cellular decomposition** (BCD) is the gold standard: decompose the non-convex polygon into convex cells, plan optimal coverage within each cell, then solve a Traveling Salesman variant to connect cells. The Fields2Cover library implements this approach and reports **14% lower path costs** than naive boustrophedon on complex polygons.

### Headland buffer edge cases

`polygon.buffer(-headland_width)` with `join_style='mitre'` preserves sharp agricultural field corners (unlike `'round'`, which erodes them). Three failure modes to handle:

- **Field too narrow**: buffer returns `POLYGON EMPTY` — fall back to covering the entire field without headlands
- **Narrow corridor pinches off**: buffer returns `MultiPolygon` — either process each sub-polygon separately or use only the largest piece
- **GEOS precision issues**: documented in Shapely Issue #1932 for very thin polygons; mitigate with `buffer(-width + 0.01)`

### Connecting swaths into a boustrophedon path

```python
def connect_boustrophedon(swath_lines):
    waypoints = []
    for i, line in enumerate(swath_lines):
        coords = list(line.coords)
        if i % 2 == 1:
            coords = coords[::-1]  # Reverse odd-numbered lines
        waypoints.extend(coords)
    return waypoints
```

For multirotor drones (holonomic — can stop and turn in place), simple straight-line connections between swath endpoints work perfectly. For fixed-wing or ground vehicles, use Dubins curves or the snake pattern (1, 3, 5, ..., 6, 4, 2) which provides wider turns.

### Spray on/off waypoint placement

Insert spray trigger waypoints **0.3–1.0 meters inside each swath endpoint** to compensate for nozzle response latency and drone deceleration. Use `line.interpolate(spray_lead_distance)` for precise metric placement:

```python
def insert_spray_triggers(line, lead_dist=0.5):
    if line.length > 2 * lead_dist:
        on_pt = line.interpolate(lead_dist)
        off_pt = line.interpolate(line.length - lead_dist)
        return (on_pt.x, on_pt.y), (off_pt.x, off_pt.y)
    return line.coords[0], line.coords[-1]  # Short swath: spray full length
```

### Polygon holes for obstacles

Shapely's `Polygon` natively supports interior rings. Use `field.difference(obstacle.buffer(safety_margin))` to subtract buffered obstacles — the resulting polygon's `intersection()` with swath lines automatically excludes hole regions, producing `MultiLineString` results where lines cross over obstacles.

### Reference implementations worth studying

- **Fields2Cover** (github.com/Fields2Cover/Fields2Cover): C++17 with Python SWIG bindings, BSD-3 license. Four-module pipeline (headland → swath → route → path). Published in IEEE RA-L 2023.
- **ethz-asl/polygon_coverage_planning**: BCD + E-GTSP optimization for general polygons with holes.
- **Greenzie/boustrophedon_planner**: ROS actionlib server handling convex and concave polygons.

---

## Area 2: Coverage heatmap — rasterize, accumulate, compute CV

### The standard metric is Coefficient of Variation

**CV = (σ / μ) × 100%** of deposition across the field. For agricultural drones, the thresholds established by ASABE Standard S386.2 (2018) are:

- **≤15% CV**: Excellent coverage (drone/aerial standard)
- **15–25% CV**: Acceptable for aerial application; ASABE defines Effective Swath Width at CV≤25% (ESW25)
- **25–30% CV**: Marginal; ESW30 threshold
- **>30% CV**: Poor — max:min deposit ratio exceeds **2.7:1**

Real-world UAAS studies (Byers et al., 2024, Frontiers in Agronomy) show drones routinely exceed 25% CV at manufacturer-recommended swath widths, making proper overlap computation essential.

### Cross-track distribution: Gaussian for drones

Three models apply depending on nozzle type. For agricultural drones, research consistently shows deposition concentrated directly below the flight path with rapid dropoff — a **Gaussian profile** fits best:

```python
def gaussian_profile(distance, half_width, sigma_factor=2.5):
    sigma = half_width / sigma_factor
    profile = np.exp(-0.5 * (distance / sigma) ** 2)
    profile[np.abs(distance) > half_width] = 0.0
    return profile
```

The sigma_factor of **2.5** means 95% of deposition falls within the swath width. For flat-fan nozzles on ground booms, a triangular profile is simpler and physically motivated — 50% overlap of two triangular profiles sums to perfectly uniform coverage.

### Rasterization algorithm

The core computation rasterizes the field at **0.5m resolution** (rule of thumb: grid cell ≤ W/10 for swath width W), then accumulates deposition contributions from every flight line:

```python
def compute_coverage(flight_lines, field_bounds, swath_width, resolution=0.5):
    xmin, ymin, xmax, ymax = field_bounds
    half_w = swath_width / 2.0
    x = np.arange(xmin + resolution/2, xmax, resolution)
    y = np.arange(ymin + resolution/2, ymax, resolution)
    xx, yy = np.meshgrid(x, y)
    deposition = np.zeros_like(xx)

    for line in flight_lines:
        dist = perpendicular_distance(xx, yy, line)  # vectorized
        mask = dist < half_w * 1.2
        contribution = np.zeros_like(dist)
        contribution[mask] = gaussian_profile(dist[mask], half_w)
        deposition += contribution

    field_vals = deposition[deposition > 0]  # within-field pixels
    cv = (np.std(field_vals) / np.mean(field_vals)) * 100
    coverage_pct = np.sum(field_vals >= 0.5 * np.mean(field_vals)) / field_vals.size * 100
    return deposition, cv, coverage_pct
```

The perpendicular distance calculation uses the standard point-to-line-segment formula, fully vectorized across the grid with numpy. For a 10-hectare field at 0.5m resolution, the grid is ~400K cells — manageable for real-time recomputation.

### Overlap math

**Overlap = (W − S) / W × 100%** where W is effective swath width and S is line spacing. Recommended overlaps for drones are **20–30%**, compared to 30–50% for ground boom sprayers. At 30% overlap with a Gaussian profile, expect **CV < 15%**. Coverage "adequacy" threshold is typically **≥50% of mean target deposition** — pixels below this are flagged as underserved.

Gap detection uses `scipy.ndimage.label()` to identify connected regions of zero or below-threshold deposition, filtering out regions smaller than 4 pixels (1 m²) to avoid noise.

---

## Area 3: Spray window scoring — weighted composites with window grouping

### Five parameters, one composite score

The algorithm evaluates each forecast hour on five parameters, each scored 0–100, then combines them with empirically-derived weights:

| Parameter | Weight | Optimal range | Key threshold |
|-----------|--------|--------------|---------------|
| Wind speed | **30%** | 3–8 mph | <3 mph = inversion risk; >10 mph = drift |
| Precipitation | **25%** | 0% probability | >30% = disqualifying |
| Temperature | **15%** | 40–85°F | <32°F or >95°F = zero score |
| Humidity / Delta-T | **15%** | 40–80% RH; Delta-T 2–8°F | <20% RH = rapid evaporation |
| Inversion risk | **15%** | No inversion | Composite of wind, cloud, time-of-day |

Wind speed carries the highest weight because it is simultaneously the most variable parameter hour-to-hour, the primary drift determinant, and — when calm — the strongest inversion indicator. Each parameter uses a **trapezoidal scoring function** that assigns 100 in the optimal range and tapers linearly outside it.

### Delta-T is superior to humidity alone

**Delta-T (dry bulb minus wet bulb temperature)** is linearly proportional to evaporation rate, making it a better predictor of droplet survival than relative humidity alone. The traffic-light system used by Australian and Canadian spray advisory services: **2–8°F = green**, 8–10°F = yellow, >10°F or <2°F = red. Compute it from temperature and RH using Stull's 2011 wet-bulb approximation.

### Inversion detection from HRRR surface data

Temperature inversions trap fine droplets near the surface, causing unpredictable long-range drift. From HRRR GRIB2 surface files, use these proxy indicators since direct vertical profiling requires pressure-level data:

- **10m wind speed < 3 mph** (+40 risk points)
- **Cloud cover < 30%** (+20 — clear skies enable radiational cooling)
- **Nighttime/twilight** (+25 — inversions form 1–3h before sunset through ~2h after sunrise)
- **Delta-T < 2°F** (+15 — high moisture, stable conditions)

For enhanced detection, compare TMP at 2m AGL with TMP at 925 mb (~750m AGL). If the 925 mb temperature exceeds the 2m temperature, a strong inversion is confirmed.

### Window aggregation with bridging

The scoring function classifies each hour as green (≥70), yellow (40–69), or red (<40). The aggregation algorithm scans hourly scores sequentially, grouping consecutive green hours into windows while allowing **up to one yellow hour as a "bridge"** between green hours. Two or more consecutive yellow hours break the window. Windows shorter than **2 hours** (the minimum practical operation duration) are discarded. Windows are ranked by `avg_score × log₂(duration + 1)`, which rewards both quality and length.

### Presentation: color-coded timeline with parameter breakdown

The recommended UI pattern is a horizontal scrollable timeline with hour-cells colored green/yellow/red, bracketed highlights around recommended windows, and a **parameter breakdown strip** below showing individual scores as mini heat-map rows (wind, temp, humidity, precip, inversion). This lets operators instantly see which factor is limiting any given hour — the single most requested feature in spray advisory UIs.

---

## Area 4: MAVLink WPL is tab-delimited text, DJI WPML is XML in a ZIP

### MAVLink WPL format specification

The file starts with header `QGC WPL 110` followed by tab-delimited lines with 12 columns: index, current_wp, coord_frame, command, param1–4, latitude, longitude, altitude, autocontinue. A complete spray mission file:

```
QGC WPL 110
0	1	0	16	0	0	0	0	35.362000	-118.329000	0	1
1	0	3	22	0	0	0	0	35.362000	-118.329000	10	1
2	0	3	178	1	5.0	-1	0	0	0	0	1
3	0	3	183	9	1900	0	0	0	0	0	1
4	0	3	16	0	0	0	0	35.362100	-118.328500	5	1
5	0	3	16	0	0	0	0	35.362200	-118.328000	5	1
6	0	3	183	9	1100	0	0	0	0	0	1
7	0	3	20	0	0	0	0	0	0	0	1
```

The critical commands: **MAV_CMD_NAV_WAYPOINT (16)** for navigation, **MAV_CMD_NAV_TAKEOFF (22)** as the first command, **MAV_CMD_DO_CHANGE_SPEED (178)** with param1=1 (ground speed) and param2=speed in m/s, **MAV_CMD_DO_SET_SERVO (183)** with param1=servo channel and param2=PWM value for spray control, and **MAV_CMD_NAV_RETURN_TO_LAUNCH (20)** to end. Use **coord_frame=3** (MAV_FRAME_GLOBAL_RELATIVE_ALT) for all spray waypoints — altitude relative to takeoff point, the standard for copter missions. Typical spray altitude is **2–5 meters AGL**.

### Spray servo encoding

For ArduPilot-based spray drones: **servo channel 9** (AUX1 on Pixhawk) with SERVO9_FUNCTION=22 (SprayerPump). PWM **1900 µs = spray on**, **1100 µs = spray off**. DJI AGRAS drones do NOT use MAVLink servo commands — they use DJI's proprietary mission system with spray rate parameters (L/mu) embedded in the mission.

### DJI WPML/KMZ structure

DJI Pilot 2 imports KMZ files (ZIP archives) containing a specific directory structure:

```
mission.kmz (ZIP)
├── wpmz/
│   ├── template.kml      (for DJI Pilot 2)
│   └── waylines.wpml     (for MSDK/firmware)
└── res/                   (optional resources)
```

The template.kml uses two XML namespaces: standard KML (`http://www.opengis.net/kml/2.2`) and DJI's WPML extension (`http://www.dji.com/wpmz/1.0.2`). Coordinates inside `<coordinates>` tags are **longitude,latitude** (comma-separated, no space). Key WPML elements include `<wpml:autoFlightSpeed>` for m/s speed, `<wpml:heightMode>` set to `relativeToStartPoint` for spray operations, and `<wpml:actionGroup>` blocks for waypoint actions. DJI AGRAS drones use the separate DJI AGRAS app, not Pilot 2, with its own spray parameter encoding.

### GeoJSON for web map rendering

Structure the FeatureCollection with Point features for each waypoint (carrying properties: `waypointIndex`, `waypointType`, `command`, `commandId`, `sprayState`, `altitude_m`, `speed_ms`), a LineString feature for the complete flight path visualization, and a Polygon feature for the spray boundary. Use **[longitude, latitude, altitude]** coordinate order per RFC 7946, with **6 decimal places** (~0.11m precision). DO_SET_SERVO and DO_CHANGE_SPEED commands are properties on the preceding waypoint Feature since they execute at that location.

---

## Area 5: Forecast data — progressive load, pre-compute everything, scrub instantly

### The data is tiny — pre-compute aggressively

A single-point 72-hour forecast (wind U/V, gusts, temperature, humidity per hour) is roughly **5–10 KB** as JSON. Even a 10×10 spatial grid over 72 hours is ~500 KB. This is small enough to hold entirely in memory and pre-compute all derived values (wind speed, direction, drift distance, spray condition scores) into a flat array indexed by time offset.

The optimal loading strategy is **progressive**: fetch hours 0–24 immediately on location set (covers the critical near-term window), then load hours 24–72 in the background via `requestIdleCallback()`. The user can start interacting after ~200ms.

### OGC EDR supports full time-range queries

A single EDR API call can return all 72 hours using ISO 8601 interval syntax in the `datetime` parameter:

```
GET /collections/hrrr/position?
  coords=POINT(-97.5 38.0)&
  datetime=2026-03-25T12:00Z/2026-03-28T12:00Z&
  parameter-name=wind_u_component,wind_v_component,temperature,relative_humidity&
  f=CoverageJSON
```

This eliminates the need for 72 separate API calls. For spatial data around a field, use the `area` query type with a POLYGON WKT geometry.

### Interpolation: always decompose into U/V components

**Never interpolate wind speed and direction directly.** Interpolating between 350° and 10° yields 180° — completely wrong. Instead, interpolate U (east-west) and V (north-south) wind components independently using linear interpolation, then recompute speed and direction from the interpolated vector:

```typescript
function interpolateWind(w0: WindVector, w1: WindVector, t: number) {
  const u = w0.u * (1 - t) + w1.u * t;
  const v = w0.v * (1 - t) + w1.v * t;
  const speed = Math.sqrt(u * u + v * v);
  const dir = (Math.atan2(-u, -v) * 180 / Math.PI + 360) % 360;
  return { u, v, speed, dir };
}
```

Pre-compute interpolated values at **15-minute intervals** (288 entries for 72 hours, ~17 KB) on data load. Slider scrubbing then becomes an O(1) array index lookup — zero computation at interaction time.

### Two-tier caching and stale data handling

Use **in-memory Map** for the active session (instant access for slider) and **IndexedDB** for cross-session persistence (avoids re-fetching on page reload). Key cache entries by model run timestamp: `HRRR_2026032512_-97.5_38.0`.

HRRR runs hourly, with data available **~45–90 minutes** after initialization (18h runs) or ~110 minutes (extended 48h runs at 00z/06z/12z/18z). Poll every 5 minutes for new model runs. When detected, show a non-intrusive toast ("New forecast available") with an "Update Now" action, pre-fetch the new data in the background without replacing the current dataset, and display a freshness indicator: **green** (<1.5h old), **yellow** (<3h), **orange** (<6h), **red** (>6h).

### Debouncing: throttle at 16ms, debounce API calls at 300ms

For the timeline slider, use `requestAnimationFrame` for visual updates (effectively throttled to ~60fps / 16ms). Since all derived values are pre-computed, each frame is just an array lookup. If heavy grid-based computations are needed (e.g., full-field drift recomputation), offload to a **Web Worker** with `Transferable` buffers for zero-copy data transfer back to the main thread. API calls triggered by location changes should be debounced at **300–500ms**.

---

## Conclusion

The five outputs share a common implementation philosophy: **do expensive work once, then serve results instantly**. Flight paths are generated server-side in Shapely using the rotate-clip-rotate-back pattern with `prepare()` for batch performance. Coverage heatmaps rasterize at 0.5m resolution using vectorized numpy distance calculations against a Gaussian cross-track profile, producing CV and coverage percentage in real time. Spray window scoring reduces to a stateless function over hourly forecast tuples, and window aggregation is a single linear scan. Export formats (WPL, KMZ, GeoJSON) are templated string/XML generation with no computation. And forecast data is small enough — 5KB for a point time series — that the entire 72-hour scrubbing experience can be pre-computed into a flat array at load time.

The most critical architectural insight across all five areas: **non-convex polygon handling is the primary source of bugs**. Every Shapely intersection can return LineString, MultiLineString, GeometryCollection, Point, or empty geometry. Every negative buffer can return Polygon, MultiPolygon, or empty. The Fields2Cover library's cellular decomposition approach — splitting complex polygons into convex cells before path planning — eliminates an entire class of edge cases and should be the target architecture for production use.