# Boreas Implementation Guide

**Companion to:** SPEC-007 v2.0
**Purpose:** Everything you need to go from `git clone` to a working dev environment and start building with consistent patterns.

---

## 1. Dev Environment Setup

### Prerequisites

| Tool | Version | Why |
|---|---|---|
| Node.js | 20 LTS | Next.js runtime |
| pnpm | 9.x | Workspace manager (faster than npm for monorepos) |
| Python | 3.12+ | FastAPI backend |
| Docker + Docker Compose | Latest | PostgreSQL + Redis for local dev |
| Rust | stable (1.77+) | Physics core (Phase 3, but install now) |
| wasm-pack | 0.13+ | WASM build (Phase 3) |

### First-time setup

```bash
# Clone
git clone https://github.com/sgupta604/boreas.git
cd boreas

# Install Node dependencies (all packages)
pnpm install

# Start infrastructure
docker compose up -d    # PostgreSQL + PostGIS + Redis

# Setup Python environment
cd packages/api
python -m venv .venv
source .venv/bin/activate   # or .venv\Scripts\activate on Windows
pip install -e ".[dev]"

# Run database migrations
alembic upgrade head

# Start API server (terminal 1)
uvicorn boreas_api.main:app --reload --port 8000

# Start web dev server (terminal 2)
cd packages/web
pnpm dev    # http://localhost:3000
```

### docker-compose.yml

```yaml
services:
  postgres:
    image: postgis/postgis:16-3.4
    environment:
      POSTGRES_DB: boreas
      POSTGRES_USER: boreas
      POSTGRES_PASSWORD: boreas_dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

### Environment files

**packages/web/.env.local:**
```env
NEXT_PUBLIC_MAPBOX_TOKEN=pk.ey...
NEXT_PUBLIC_API_BASE_URL=http://localhost:8000
```

**packages/api/.env:**
```env
SHYFT_API_KEY=owp_...
FOLKWEATHER_BASE_URL=https://folkweather.com/edr/collections
MAPBOX_ACCESS_TOKEN=pk.ey...
DATABASE_URL=postgresql+asyncpg://boreas:boreas_dev@localhost:5432/boreas
REDIS_URL=redis://localhost:6379
JWT_SECRET=dev-secret-change-in-prod
CORS_ORIGINS=http://localhost:3000
```

---

## 2. Monorepo Structure Rules

### Package manager: pnpm workspaces

Root `pnpm-workspace.yaml`:
```yaml
packages:
  - "packages/*"
```

Root `package.json`:
```json
{
  "private": true,
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "lint": "turbo lint",
    "test": "turbo test",
    "typecheck": "turbo typecheck"
  },
  "devDependencies": {
    "turbo": "^2.0.0",
    "typescript": "^5.4.0"
  }
}
```

### Turborepo config

`turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "dev": { "persistent": true, "cache": false },
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] },
    "lint": {},
    "test": {},
    "typecheck": { "dependsOn": ["^build"] }
  }
}
```

### Top-level directory layout

```
boreas/
├── packages/
│   ├── web/           # Next.js frontend
│   ├── api/           # FastAPI backend
│   ├── shared/        # Shared TypeScript types
│   └── physics/       # Rust drift engine (Phase 3) — 3-crate workspace
│       ├── Cargo.toml           # [workspace] members = ["crates/*"]
│       └── crates/
│           ├── drift-core/      # Pure Rust physics (no platform deps)
│           ├── drift-python/    # PyO3 bindings → maturin build
│           └── drift-wasm/      # wasm-bindgen → wasm-pack build
├── docker-compose.yml
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
├── .github/
│   └── workflows/
│       └── ci.yml
├── .gitignore
└── README.md
```

---

## 3. File Structure: packages/web

```
packages/web/
├── app/                          # Next.js App Router (pages + layouts)
│   ├── layout.tsx                # Root: dark theme, Inter font, nav shell
│   ├── page.tsx                  # Landing / marketing
│   ├── plan/
│   │   ├── page.tsx              # Main planning interface
│   │   └── [operationId]/
│   │       └── page.tsx          # Saved operation view
│   ├── dome/
│   │   └── page.tsx              # 3D dome (Phase 4)
│   └── settings/
│       └── page.tsx
│
├── components/                   # React components (UI only, no business logic)
│   ├── map/
│   │   ├── MapView.tsx           # Mapbox GL wrapper
│   │   ├── PolygonTool.tsx       # Draw/edit field polygons
│   │   ├── FlightPathLayer.tsx   # Render flight paths on map
│   │   ├── DriftContour.tsx      # Drift zone polygon overlay
│   │   └── WindIndicator.tsx     # Wind arrow/barb at field centroid
│   ├── panels/
│   │   ├── OperationConfig.tsx   # Aircraft, nozzle, spray height, buffer
│   │   ├── ForecastTimeline.tsx  # 48-hour scrubber
│   │   ├── DriftAssessment.tsx   # Drift results + confidence label
│   │   ├── WindSummary.tsx       # Wind conditions display
│   │   ├── ComplianceCheck.tsx   # Pass/fail compliance indicators
│   │   ├── CoverageReport.tsx    # Coverage % and stats
│   │   └── ExportPanel.tsx       # Export format buttons
│   ├── dome/                     # Phase 4
│   │   └── DomeViewer.tsx
│   └── ui/                       # Generic primitives
│       ├── Button.tsx
│       ├── Slider.tsx
│       ├── Select.tsx
│       └── Panel.tsx
│
├── lib/                          # Business logic, API calls, computation
│   ├── api/
│   │   └── client.ts             # Typed fetch wrapper for FastAPI
│   ├── drift/
│   │   ├── ballistic.ts          # Phase 1 client-side ballistic drift
│   │   ├── terminal-velocity.ts  # ASABE class → v_t lookup
│   │   └── log-profile.ts       # Wind extrapolation 10m → spray height
│   ├── stores/
│   │   ├── operation-store.ts    # Zustand: current operation state
│   │   ├── weather-store.ts      # Zustand: cached wind data
│   │   └── ui-store.ts           # Zustand: panel open/close, selected time
│   └── utils/
│       ├── geo.ts                # Coordinate math, distance, bearing
│       ├── units.ts              # m/s ↔ mph ↔ knots, meters ↔ feet
│       └── format.ts             # Number/date formatting helpers
│
├── public/
│   └── wasm/                     # Phase 3: WASM module
│
├── next.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

### Rules for packages/web

1. **`app/` is routing only.** Pages import components. No business logic in page files beyond data fetching.
2. **`components/` is UI only.** Components receive data via props or Zustand stores. They don't call APIs directly.
3. **`lib/` is logic.** API calls, computation, state management. Components import from lib.
4. **One component per file.** Name matches: `OperationConfig.tsx` exports `OperationConfig`.
5. **No `index.ts` barrel files in packages/web.** Import directly: `import { MapView } from '@/components/map/MapView'`. Barrel files cause circular dependency issues and slow HMR. Exception: `packages/shared/src/index.ts` is fine — it's a types-only package with no HMR concerns.
6. **Zustand stores are the single source of truth** for client state. No prop drilling beyond 2 levels — if data needs to go deeper, put it in a store.
7. **`@/` alias** resolves to `packages/web/`. Configure in `tsconfig.json`.

---

## 4. File Structure: packages/api

```
packages/api/
├── boreas_api/
│   ├── __init__.py
│   ├── main.py                   # FastAPI app: lifespan, CORS, router mounts
│   ├── config.py                 # Pydantic Settings (reads .env)
│   ├── dependencies.py           # Dependency injection (DB session, Redis, auth)
│   │
│   ├── routers/                  # HTTP route handlers (thin — validate, call domain, return)
│   │   ├── __init__.py
│   │   ├── wind.py               # GET /api/v1/wind
│   │   ├── drift.py              # POST /api/v1/drift
│   │   ├── flight_plan.py        # POST /api/v1/flight-plan/generate
│   │   ├── operations.py         # CRUD /api/v1/operations
│   │   └── auth.py               # POST /api/v1/auth/login, /register
│   │
│   ├── domain/                   # Business logic (framework-agnostic, testable)
│   │   ├── __init__.py
│   │   ├── weather/
│   │   │   ├── __init__.py
│   │   │   ├── ports.py          # WeatherDataPort protocol
│   │   │   ├── service.py        # WeatherService (router → adapters, caching)
│   │   │   ├── adapters/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── shyft.py      # Shyft EDR adapter
│   │   │   │   ├── folkweather.py # Folkweather EDR adapter
│   │   │   │   └── mock.py       # Mock adapter for testing
│   │   │   ├── normalize.py      # Raw EDR response → clean WindGrid
│   │   │   └── models.py         # WindGrid, WindVector, SurfaceConditions
│   │   │
│   │   ├── drift/
│   │   │   ├── __init__.py
│   │   │   ├── service.py        # DriftService (router calls this)
│   │   │   ├── ballistic.py      # Phase 1 server-side ballistic (matches client)
│   │   │   ├── lagrangian.py     # Phase 2 Lagrangian particle tracker (NumPy)
│   │   │   ├── droplet.py        # Terminal velocity, drag, ASABE classes
│   │   │   ├── atmosphere.py     # Log profile, stability class estimation
│   │   │   └── models.py         # DriftResult, DropletConfig, SimulationParams
│   │   │
│   │   ├── flight/
│   │   │   ├── __init__.py
│   │   │   ├── service.py        # FlightPlanService
│   │   │   ├── path_generator.py # Pure Python/Shapely coverage path planning
│   │   │   ├── wind_aware.py     # Swath angle optimization, drift offset
│   │   │   ├── export.py         # MAVLink WPL, KMZ, GeoJSON, CSV generators
│   │   │   └── models.py         # FlightPlan, Waypoint, CoverageStats
│   │   │
│   │   ├── compliance/
│   │   │   ├── __init__.py
│   │   │   ├── service.py        # ComplianceService
│   │   │   ├── wind_limits.py    # Label wind speed checks
│   │   │   ├── buffer_zones.py   # Buffer distance validation
│   │   │   └── models.py         # ComplianceResult, Violation
│   │   │
│   │   └── nozzle/
│   │       ├── __init__.py
│   │       ├── database.py       # Load + query nozzle JSON database
│   │       └── nozzles.json      # Curated nozzle data (30–50 entries)
│   │
│   ├── db/                       # Database layer
│   │   ├── __init__.py
│   │   ├── models.py             # SQLAlchemy + GeoAlchemy2 ORM models
│   │   ├── session.py            # Async session factory
│   │   └── migrations/           # Alembic
│   │       ├── env.py
│   │       └── versions/
│   │
│   └── tasks/                    # Background jobs (ARQ)
│       ├── __init__.py
│       └── monte_carlo.py        # Phase 3: async Tier 3 simulation
│
├── tests/
│   ├── conftest.py               # Fixtures: test client, mock weather, test DB
│   ├── test_wind.py
│   ├── test_drift.py
│   ├── test_flight_plan.py
│   ├── test_compliance.py
│   ├── test_droplet.py           # Terminal velocity unit tests
│   └── test_path_generator.py
│
├── pyproject.toml
├── Dockerfile
├── alembic.ini
└── railway.toml
```

### Rules for packages/api

1. **Routers are thin.** Validate request → call domain service → return response. No business logic in routers.
2. **Domain is framework-agnostic.** Nothing in `domain/` imports from FastAPI, SQLAlchemy, or Redis. It uses protocols (ports) for external dependencies, injected via `dependencies.py`.
3. **One service per bounded context.** `WeatherService`, `DriftService`, `FlightPlanService`, `ComplianceService`. Services can call each other (e.g., FlightPlanService calls DriftService and ComplianceService).
4. **Adapters implement ports.** `WeatherDataPort` is a Protocol. `ShyftAdapter` and `FolkweatherAdapter` implement it. Swap them without touching domain code.
5. **Models are Pydantic.** Domain models in `models.py` are Pydantic BaseModel. DB models in `db/models.py` are SQLAlchemy. Never mix them — convert explicitly.
6. **Tests mirror domain structure.** One test file per domain module. Use `pytest` + `pytest-asyncio`.
7. **Type everything.** Every function has full type annotations. Every Pydantic model has field types. `mypy --strict` should pass (configure in pyproject.toml).

---

## 5. File Structure: packages/shared

```
packages/shared/
├── src/
│   ├── types/
│   │   ├── wind.ts           # WindVector, WindGrid, AltitudeLevel
│   │   ├── drift.ts          # DropletClass, DriftResult, DriftConfig
│   │   ├── flight.ts         # FlightPlan, Waypoint, CoverageStats
│   │   ├── operation.ts      # Operation, OperationConfig, OperationStatus
│   │   └── compliance.ts     # ComplianceResult, Violation
│   ├── constants.ts          # ASABE droplet classes, altitude levels, pressure mappings
│   ├── units.ts              # Conversion functions
│   └── index.ts              # Re-exports
├── tsconfig.json
└── package.json
```

### Rules for packages/shared

1. **Types only.** No runtime code, no dependencies. Just TypeScript interfaces, enums, and pure functions (unit conversions, constants).
2. **Source of truth for API contracts.** If the frontend sends `DropletClass.MEDIUM`, the backend expects the same string. Define it here once.
3. **No framework imports.** Nothing from React, Next.js, FastAPI. Pure TypeScript.

---

## 6. Coding Patterns

### API error handling (FastAPI)

All domain exceptions are caught and converted to HTTP responses in routers:

```python
# domain/weather/exceptions.py
class WeatherFetchError(Exception):
    """Upstream weather API failed."""
    def __init__(self, source: str, status: int, detail: str):
        self.source = source
        self.status = status
        self.detail = detail

# routers/wind.py
@router.get("/api/v1/wind")
async def get_wind(
    lat: float, lon: float, time: str,
    weather: WeatherService = Depends(get_weather_service),
):
    try:
        result = await weather.get_wind(lat, lon, time)
        return result
    except WeatherFetchError as e:
        raise HTTPException(502, detail=f"Weather source {e.source} failed: {e.detail}")
    except ValidationError as e:
        raise HTTPException(422, detail=str(e))
```

### API response format

All successful responses return data directly (Pydantic model serialized to JSON). Errors use standard HTTPException which produces `{"detail": "..."}`.

No response wrappers. No `{"status": "ok", "data": {...}}` pattern.

### Request validation (Pydantic)

```python
# routers/drift.py
class DriftRequest(BaseModel):
    polygon: list[tuple[float, float]]  # [[lon, lat], ...]
    spray_height_m: float = Field(ge=0.5, le=10.0)
    droplet_class: DropletClass = DropletClass.MEDIUM
    time: datetime
    crop_type: CropType = CropType.LOW_CROP  # bare_soil, short_grass, low_crop, high_crop, orchard

    @field_validator("polygon")
    @classmethod
    def validate_polygon(cls, v):
        if len(v) < 3:
            raise ValueError("Polygon needs at least 3 points")
        return v
```

### Client API calls (typed fetch wrapper)

```typescript
// lib/api/client.ts
const API_BASE = process.env.NEXT_PUBLIC_API_BASE_URL;

export async function fetchWind(lat: number, lon: number, time: string): Promise<WindGrid> {
  const res = await fetch(`${API_BASE}/api/v1/wind?lat=${lat}&lon=${lon}&time=${time}`);
  if (!res.ok) {
    const err = await res.json();
    throw new Error(err.detail || `Wind fetch failed: ${res.status}`);
  }
  return res.json();
}

export async function computeDrift(params: DriftRequest): Promise<DriftResult> {
  const res = await fetch(`${API_BASE}/api/v1/drift`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(params),
  });
  if (!res.ok) {
    const err = await res.json();
    throw new Error(err.detail || `Drift computation failed: ${res.status}`);
  }
  return res.json();
}
```

### Zustand store pattern

```typescript
// lib/stores/operation-store.ts
import { create } from 'zustand';
import type { OperationConfig, DriftResult } from '@boreas/shared';

interface OperationState {
  polygon: GeoJSON.Feature | null;
  config: OperationConfig;
  driftResult: DriftResult | null;
  isComputing: boolean;

  setPolygon: (polygon: GeoJSON.Feature) => void;
  updateConfig: (partial: Partial<OperationConfig>) => void;
  setDriftResult: (result: DriftResult) => void;
  setComputing: (v: boolean) => void;
  reset: () => void;
}

const defaultConfig: OperationConfig = {
  sprayHeightM: 3,
  dropletClass: 'medium',
  cropType: 'low_crop',
  bufferM: 30,
  overlapPct: 0.15,
  nozzleId: 'teejet-xr-11002',
};

export const useOperationStore = create<OperationState>((set) => ({
  polygon: null,
  config: defaultConfig,
  driftResult: null,
  isComputing: false,

  setPolygon: (polygon) => set({ polygon }),
  updateConfig: (partial) => set((s) => ({ config: { ...s.config, ...partial } })),
  setDriftResult: (result) => set({ driftResult: result }),
  setComputing: (v) => set({ isComputing: v }),
  reset: () => set({ polygon: null, config: defaultConfig, driftResult: null }),
}));
```

### Component pattern

```tsx
// components/panels/DriftAssessment.tsx
'use client';

import { useOperationStore } from '@/lib/stores/operation-store';

export function DriftAssessment() {
  const { driftResult, isComputing } = useOperationStore();

  if (!driftResult) return null;

  return (
    <div className="space-y-2">
      <h3 className="text-sm font-medium text-zinc-400">Drift Assessment</h3>
      {isComputing ? (
        <p className="text-zinc-500">Computing...</p>
      ) : (
        <>
          <p className="text-lg font-mono text-white">
            {driftResult.drift_distance_m.toFixed(1)}m {bearingToCardinal(driftResult.drift_bearing_deg)}
          </p>
          <p className="text-sm text-zinc-400">
            Buffer required: {driftResult.buffer_required_m.toFixed(1)}m
          </p>
          <p className="text-xs text-zinc-500">
            Confidence: {driftResult.confidence}
          </p>
        </>
      )}
    </div>
  );
}
```

---

## 7. Testing Strategy

### Frontend (packages/web)

**Framework:** Vitest + React Testing Library
**What to test:**
- `lib/drift/ballistic.ts` — unit tests for drift computation (pure math, no DOM)
- `lib/drift/terminal-velocity.ts` — verify ASABE class lookups
- `lib/drift/log-profile.ts` — verify wind extrapolation
- `lib/utils/*.ts` — unit conversion, geo math

**What NOT to test:** Individual component rendering (fragile, low value for a solo dev). Test behavior through the stores and lib functions.

### Backend (packages/api)

**Framework:** pytest + pytest-asyncio + httpx (for TestClient)
**What to test:**

```python
# tests/conftest.py
import pytest
from httpx import AsyncClient, ASGITransport
from boreas_api.main import app
from boreas_api.domain.weather.adapters.mock import MockWeatherAdapter

@pytest.fixture
def mock_weather():
    return MockWeatherAdapter(
        wind_u=-2.3, wind_v=1.8, temp_c=20.0
    )

@pytest.fixture
async def client():
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as c:
        yield c
```

**Test categories:**

| Category | What | Example |
|---|---|---|
| Unit: droplet physics | Terminal velocity convergence | `test_terminal_velocity_100um_matches_stokes()` |
| Unit: drift model | Ballistic drift against analytical solution | `test_ballistic_drift_uniform_wind()` |
| Unit: path generator | Swath count for known polygon + spacing | `test_swath_count_square_field()` |
| Unit: compliance | Wind limit check pass/fail | `test_wind_exceeds_label_limit()` |
| Integration: wind API | EDR adapter → normalized response | `test_shyft_adapter_returns_wind_grid()` |
| Integration: drift API | POST /drift → valid response | `test_drift_endpoint_returns_result()` |
| Integration: flight plan | POST /flight-plan → valid waypoints | `test_flight_plan_generates_path()` |

**Run:** `cd packages/api && pytest -v`

### Rust (packages/physics) — Phase 3

**Framework:** Built-in `#[cfg(test)]` + `cargo test`
**What to test:**
- Terminal velocity convergence against known values
- Drift distance against analytical solutions (still air, uniform wind)
- Monte Carlo convergence (P99 stable after N samples)
- WASM vs native result identity (cross-compile test)

---

## 8. CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  web-lint-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm --filter @boreas/web lint
      - run: pnpm --filter @boreas/web typecheck

  api-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgis/postgis:16-3.4
        env:
          POSTGRES_DB: boreas_test
          POSTGRES_USER: boreas
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: |
          cd packages/api
          pip install -e ".[dev]"
          pytest -v --tb=short
        env:
          DATABASE_URL: postgresql+asyncpg://boreas:test@localhost:5432/boreas_test
          REDIS_URL: redis://localhost:6379

  # Phase 3: uncomment when Rust crate exists
  # physics-test:
  #   runs-on: ubuntu-latest
  #   steps:
  #     - uses: actions/checkout@v4
  #     - uses: dtolnay/rust-toolchain@stable
  #     - run: cd packages/physics && cargo test
```

### Deployment

**Web (Vercel):** Auto-deploys on push to `main`. Vercel detects Next.js in `packages/web/`. Set root directory to `packages/web` in Vercel project settings.

**API (Railway):** Auto-deploys on push to `main`. Uses Dockerfile in `packages/api/`. Environment variables set in Railway dashboard.

**Database + Redis:** Railway managed services. Connection URLs in Railway environment variables.

---

## 9. Git Workflow

### Branch naming

```
main                    # Production. Always deployable.
feat/wind-api           # New feature
fix/folkweather-lon     # Bug fix
refactor/drift-models   # Code reorganization
```

### Commit messages

```
feat(api): add Shyft EDR wind adapter
fix(web): correct drift contour polygon rendering
refactor(api): extract WeatherService from router
test(api): add terminal velocity unit tests
docs: update SPEC-007 phase 2 checklist
```

Prefix with package when relevant: `feat(api):`, `fix(web):`, `feat(shared):`.

### PR workflow

1. Branch from `main`
2. Work, commit, push
3. Open PR → CI runs automatically
4. Merge to `main` → auto-deploy

No staging environment for MVP. `main` = production. Use feature flags or route-level guards for incomplete features.

---

## 10. Deployment Configuration

### packages/api/Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# System deps for Shapely, pyproj, psycopg2
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgeos-dev \
    libproj-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml .
RUN pip install --no-cache-dir .

COPY boreas_api/ boreas_api/

EXPOSE 8000
CMD ["uvicorn", "boreas_api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Target image size:** ~250–350 MB (python:slim + Shapely + pyproj + NumPy). No GDAL. No Fields2Cover.

### packages/api/pyproject.toml

```toml
[project]
name = "boreas-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.111",
    "uvicorn[standard]>=0.30",
    "httpx>=0.27",
    "redis>=5.0",
    "pydantic>=2.7",
    "pydantic-settings>=2.3",
    "shapely>=2.0",
    "pyproj>=3.6",
    "numpy>=1.26",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "mypy>=1.10",
    "ruff>=0.4",
]
db = [
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.29",
    "geoalchemy2>=0.14",
    "alembic>=1.13",
]
auth = [
    "pyjwt>=2.8",
    "passlib[bcrypt]>=1.7",
]
tasks = [
    "arq>=0.26",
]

[tool.pytest.ini_options]
asyncio_mode = "auto"

[tool.ruff]
target-version = "py312"
line-length = 100

[tool.mypy]
python_version = "3.12"
strict = true
```

Phase 1 installs only the base dependencies. Phase 3 adds `[db]`, `[auth]`, `[tasks]`. Note: `shapely` and `pyproj` are in base deps even though they're first used in Phase 2 path planning — the ~10MB overhead is worth avoiding optional dependency gymnastics.

### packages/web/next.config.ts

```typescript
import type { NextConfig } from 'next';

const config: NextConfig = {
  // Required for mapbox-gl
  transpilePackages: ['mapbox-gl'],

  // WASM support (Phase 3)
  webpack: (config) => {
    config.experiments = {
      ...config.experiments,
      asyncWebAssembly: true,
    };
    return config;
  },
};

export default config;
```

---

## 11. Key Conventions

### Naming

| Thing | Convention | Example |
|---|---|---|
| TypeScript files | camelCase | `operationStore.ts` |
| React components | PascalCase | `DriftAssessment.tsx` |
| Python files | snake_case | `path_generator.py` |
| Python classes | PascalCase | `WeatherService` |
| API endpoints | kebab-case | `/api/v1/flight-plan/generate` |
| Database tables | snake_case plural | `flight_plans` |
| Database columns | snake_case | `spray_height_m` |
| Env variables | SCREAMING_SNAKE | `SHYFT_API_KEY` |
| Git branches | kebab-case with prefix | `feat/wind-api` |
| Rust files | snake_case | `terminal_velocity.rs` |

### Units

**All internal computation uses SI units.** Conversion to display units (mph, feet, etc.) happens only at the UI boundary.

| Quantity | Internal unit | Display options |
|---|---|---|
| Wind speed | m/s | mph, knots, km/h |
| Distance | meters | feet, miles |
| Altitude | meters | feet |
| Temperature | °C | °F |
| Area | m² | acres, hectares |
| Volume | liters | gallons |
| Pressure | hPa | mb (same), inHg |

### Error messages

User-facing errors are plain English: "Wind data unavailable. Try again in a few seconds."
Developer-facing errors include context: `WeatherFetchError(source="shyft", status=500, detail="Internal Server Error for GFS_isobaric position query")`

Never expose stack traces or internal details to the client.

---

## 12. Phase 1 Checklist (Copy This to Your Issue Tracker)

```
## Phase 1: Foundation

### Infrastructure
- [ ] Create GitHub repo: sgupta604/boreas
- [ ] pnpm workspace + turbo.json setup
- [ ] packages/web: npx create-next-app@latest --typescript --tailwind --app
- [ ] packages/api: FastAPI skeleton with health check endpoint
- [ ] packages/shared: TypeScript types for WindGrid, DriftResult
- [ ] docker-compose.yml: PostgreSQL + Redis
- [ ] .github/workflows/ci.yml: lint + typecheck + pytest
- [ ] Deploy web to Vercel (even if empty)
- [ ] Deploy API to Railway (even if just health check)
- [ ] Verify end-to-end: web → API → response works in production

### Wind Data
- [ ] Shyft EDR adapter (port logic from Flutter mobile app)
- [ ] Folkweather EDR adapter (port logic from Flutter mobile app)
- [ ] Wind router: Shyft → Folkweather fallback
- [ ] Redis caching (1-hour TTL)
- [ ] GET /api/v1/wind endpoint returning normalized WindGrid
- [ ] Test: mock adapter, Shyft adapter, Folkweather adapter

### Map
- [ ] Mapbox GL JS in Next.js (satellite layer, dark theme)
- [ ] Polygon drawing tool (@mapbox/mapbox-gl-draw)
- [ ] Field polygon stored in Zustand
- [ ] Wind arrow/barb rendered at field centroid
- [ ] Drift contour polygon overlay

### Forecast Timeline
- [ ] 72-hour timeline scrubber component (GFS via Shyft; narrows to 48h when HRRR added)
- [ ] Scrubbing fetches wind for selected time
- [ ] Drift recomputes on scrub (client-side ballistic)

### Client Drift
- [ ] ballistic.ts: terminal velocity lookup + fall_time + drift_distance
- [ ] log-profile.ts: extrapolate 10m wind to spray height
- [ ] Drift result displayed in panel
- [ ] Drift contour updates on parameter change

### Done Criteria
- [ ] Live URL where you can draw a polygon and see wind + drift
```

---

## Summary

You now have two documents:

**SPEC-007 v2** answers: *What are we building, what architecture, what physics, what data, in what order?*

**This implementation guide** answers: *How do I set up my machine, where do files go, what patterns do I follow, how do I test, how do I deploy?*

Start with the Phase 1 checklist. First commit should be the monorepo skeleton with a health check endpoint deployed to production. Everything else builds on that.
