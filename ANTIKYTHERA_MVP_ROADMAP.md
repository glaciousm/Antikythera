# ANTIKYTHERA MVP Roadmap

**Scope:** Sun + Earth | Newtonian Gravity | Yoshida 8 Integrator | JPL Live Data | Basic API

**Methodology:** Test-Driven Development — every step includes a unit test

**Estimated Duration:** 28-32 hours

---

# Phase 0: Project Setup
**Duration:** 1 hour

| Step | Task | Test | Acceptance Criteria |
|------|------|------|---------------------|
| 0.1 | Create Maven project structure | N/A | `mvn clean compile` succeeds |
| 0.2 | Add dependencies (Jackson, SLF4J, JUnit 5, AssertJ) | N/A | `mvn test` runs (0 tests) |
| 0.3 | Configure Java 21, compiler settings | N/A | Project compiles with Java 21 |

**Deliverables:**
- `pom.xml` with all dependencies
- Package structure: `com.antikythera.{core,physics,integrator,time,data,api}`
- `README.md` with build instructions

---

# Phase 1: Core Mathematics
**Duration:** 4 hours

## Step 1.1: Vector3D — Basic
| Component | `Vector3D` record |
|-----------|-------------------|
| Methods | Constructor, `zero()`, `x()`, `y()`, `z()`, `toArray()` |
| Test | Components stored and retrieved correctly |

## Step 1.2: Vector3D — Arithmetic
| Methods | `add()`, `subtract()`, `scale()`, `negate()` |
|---------|---------------------------------------------|
| Test | Operations produce correct results |
| Test | Immutability preserved |

## Step 1.3: Vector3D — Products & Magnitude
| Methods | `dot()`, `cross()`, `magnitude()`, `magnitudeSquared()`, `normalize()` |
|---------|------------------------------------------------------------------------|
| Test | Dot product computes scalar correctly |
| Test | Cross product is anti-commutative |
| Test | Magnitude of unit vector is 1 |
| Test | Normalize produces unit vector |

## Step 1.4: Vector3D — Distance
| Methods | `distanceTo()` |
|---------|----------------|
| Test | Euclidean distance correct |
| Test | Symmetric: `a.distanceTo(b) == b.distanceTo(a)` |

## Step 1.5: Physical Constants
| Component | `Constants` final class |
|-----------|------------------------|
| Values | `G`, `C`, `AU`, `SECONDS_PER_DAY`, `J2000_JD` |
| Test | AU ≈ 1.496 × 10⁸ km |
| Test | J2000 = 2451545.0 |

**Phase 1 Checkpoint:** 19 tests passing

---

# Phase 2: Domain Model
**Duration:** 3 hours

## Step 2.1: Body — Properties
| Component | `Body` record |
|-----------|---------------|
| Fields | `id`, `name`, `gm` (gravitational parameter) |
| Test | Fields stored correctly |
| Test | Sun GM ≈ 1.327 × 10¹¹ km³/s² |

## Step 2.2: Body — Validation
| Validation | Reject null/blank id, reject non-positive GM |
|------------|---------------------------------------------|
| Test | `IllegalArgumentException` on invalid input |

## Step 2.3: StateVector
| Component | `StateVector` record |
|-----------|---------------------|
| Fields | `position` (Vector3D), `velocity` (Vector3D) |
| Methods | `speed()`, `distanceFromOrigin()`, `distanceTo()` |
| Test | Components accessible |
| Test | Convenience constructor with 6 doubles |

## Step 2.4: SystemState
| Component | `SystemState` class |
|-----------|---------------------|
| Fields | `epoch` (JD), `Map<String, StateVector>` |
| Methods | `getState()`, `getBodyIds()`, `getBodyCount()`, `withEpoch()`, `withState()` |
| Test | Epoch stored correctly |
| Test | State retrieval by ID |
| Test | Immutability (defensive copy) |

**Phase 2 Checkpoint:** 33 tests passing

---

# Phase 3: Physics Engine
**Duration:** 8 hours

## Step 3.1: ForceModel Interface
| Component | `ForceModel` interface |
|-----------|------------------------|
| Method | `computeAcceleration(targetId, targetState, system, bodies)` → `Vector3D` |
| Test | N/A (interface only) |

## Step 3.2: NewtonianGravity
| Component | `NewtonianGravity` implements `ForceModel` |
|-----------|-------------------------------------------|
| Formula | `a = Σ GM_j × (r_j - r_i) / |r_j - r_i|³` |
| Test | Earth at 1 AU: correct acceleration magnitude |
| Test | Inverse square law: 2× distance → ¼ acceleration |
| Test | Acceleration points toward source |
| Test | Body does not accelerate itself |
| Test | Multiple sources: accelerations sum correctly |

## Step 3.3: Integrator Interface
| Component | `Integrator` interface |
|-----------|------------------------|
| Methods | `step()`, `getOrder()`, `isSymplectic()` |
| Component | `AccelerationFunction` functional interface |

## Step 3.4: Yoshida8 Integrator
| Component | `Yoshida8` implements `Integrator` |
|-----------|-----------------------------------|
| Properties | 8th order, symplectic |
| Coefficients | From Yoshida (1990) paper |
| Test | `getOrder()` returns 8 |
| Test | `isSymplectic()` returns true |
| Test | Circular orbit stable for 1 year (error < 1000 km) |
| Test | Energy conserved (relative error < 10⁻⁸) |
| Test | Epoch advances correctly |

## Step 3.5: NBodyEngine
| Component | `NBodyEngine` class |
|-----------|---------------------|
| Fields | bodies, forceModel, integrator, currentState |
| Methods | `step(dt)`, `advanceTo(epoch, maxStep)`, `getState()`, `setState()` |
| Test | Initialization correct |
| Test | Single step advances epoch |
| Test | AdvanceTo reaches target |
| Test | Orbital motion preserved (90° after ¼ year) |

**Phase 3 Checkpoint:** 51 tests passing

---

# Phase 4: Time System
**Duration:** 3 hours

## Step 4.1: JulianDate Utilities
| Component | `JulianDate` class |
|-----------|-------------------|
| Methods | `fromCalendar(y,m,d,h,m,s)` → JD |
| Methods | `toCalendar(jd)` → components |
| Test | J2000.0 = 2000-01-01 12:00:00 |
| Test | Round-trip conversion |

## Step 4.2: Unix ↔ JD Conversion
| Methods | `fromUnix(timestamp)`, `toUnix(jd)` |
|---------|-------------------------------------|
| Test | Unix epoch (1970-01-01) correct |
| Test | Current time converts correctly |

## Step 4.3: TimeEngine
| Component | `TimeEngine` class |
|-----------|-------------------|
| Fields | currentEpoch, timeScale, running |
| Methods | `getCurrentJD()`, `setTimeScale()`, `setTime()` |
| Test | Initial epoch stored |
| Test | Time scale applied |

**Phase 4 Checkpoint:** 59 tests passing

---

# Phase 5: External Data Integration
**Duration:** 5 hours

## Step 5.1: JPL Horizons Response Parser
| Component | `HorizonsParser` class |
|-----------|------------------------|
| Input | Raw JSON response from API |
| Output | `StateVector` |
| Test | Parse sample Sun response |
| Test | Parse sample Earth response |
| Test | Handle scientific notation (1.234E+05) |

## Step 5.2: JPL Horizons Client
| Component | `JPLHorizonsClient` class |
|-----------|--------------------------|
| Methods | `fetchState(bodyId, instant)` → `StateVector` |
| Methods | `fetchCurrentState(bodyId)` → `StateVector` |
| Test | (Integration) Fetch Sun state - verify structure |
| Test | (Integration) Fetch Earth state - verify structure |
| Test | Body ID mapping (earth → 399) |

## Step 5.3: Body Catalog
| Component | `BodyCatalog` class |
|-----------|---------------------|
| Data | Sun: GM = 1.327×10¹¹ km³/s² |
| Data | Earth: GM = 3.986×10⁵ km³/s² |
| Methods | `get(id)` → `Body` |
| Methods | `getAll()` → `Map<String, Body>` |
| Test | Sun retrievable |
| Test | Earth retrievable |
| Test | Unknown body returns null |

## Step 5.4: Initialization Service
| Component | `InitializationService` class |
|-----------|------------------------------|
| Methods | `initialize(bodyIds)` → `SystemState` |
| Logic | Fetch all states from JPL, build SystemState |
| Test | (Integration) Initialize Sun+Earth system |
| Test | Returned epoch is current |

**Phase 5 Checkpoint:** 71 tests passing

---

# Phase 6: Validation & Monitoring
**Duration:** 3 hours

## Step 6.1: Energy Calculator
| Component | `EnergyCalculator` class |
|-----------|-------------------------|
| Methods | `kineticEnergy(state, bodies)` → double |
| Methods | `potentialEnergy(state, bodies)` → double |
| Methods | `totalEnergy(state, bodies)` → double |
| Test | KE of stationary body is 0 |
| Test | PE of two bodies is -GM₁GM₂/r |
| Test | Total = KE + PE |

## Step 6.2: Conservation Monitor
| Component | `ConservationMonitor` class |
|-----------|----------------------------|
| Methods | `initialize(state)` — store baseline |
| Methods | `computeDrift(state)` → `ConservationMetrics` |
| Fields | energyDrift, momentumDrift |
| Test | Initial drift is 0 |
| Test | Drift computed correctly after integration |

## Step 6.3: Validation Service
| Component | `ValidationService` class |
|-----------|--------------------------|
| Methods | `compareWithJPL(state)` → `ValidationReport` |
| Fields | Position errors per body |
| Test | Error computed correctly |
| Test | Report contains all bodies |

**Phase 6 Checkpoint:** 81 tests passing

---

# Phase 7: API Layer
**Duration:** 4 hours

## Step 7.1: Query Engine
| Component | `QueryEngine` class |
|-----------|---------------------|
| Methods | `getPosition(bodyId)` → `StateVector` |
| Methods | `getPosition(bodyId, unixTime)` → `StateVector` |
| Test | Current position returned |
| Test | Historical position (requires integration) |

## Step 7.2: Simulator Facade
| Component | `Antikythera` class (main API) |
|-----------|-------------------------------|
| Methods | `initialize()` — fetch data, start engine |
| Methods | `getPosition(bodyId)` |
| Methods | `getPosition(bodyId, unixTime)` |
| Methods | `getCurrentTime()` → Unix timestamp |
| Methods | `setTimeScale(scale)` |
| Methods | `shutdown()` |
| Test | Initialize fetches live data |
| Test | Position query works |
| Test | Time scale changes propagation speed |

## Step 7.3: DTOs
| Components | `PositionDTO`, `ValidationReportDTO` |
|------------|-------------------------------------|
| Purpose | Clean API responses |
| Test | Serialization to JSON |

**Phase 7 Checkpoint:** 89 tests passing

---

# Phase 8: Integration & Polish
**Duration:** 3 hours

## Step 8.1: End-to-End Test
| Test | Full simulation cycle |
|------|----------------------|
| Steps | Initialize → Run 1 day → Query position → Validate vs JPL |
| Criteria | Position error < 100 km after 1 day |

## Step 8.2: Multi-Day Accuracy Test
| Test | Extended integration accuracy |
|------|------------------------------|
| Steps | Initialize → Run 30 days → Compare with JPL |
| Criteria | Position error < 1000 km after 30 days |

## Step 8.3: Energy Conservation Test
| Test | Long-term energy stability |
|------|---------------------------|
| Steps | Run 365 days → Check energy drift |
| Criteria | Relative energy drift < 10⁻¹⁰ |

## Step 8.4: Main Entry Point
| Component | `Main` class |
|-----------|-------------|
| Features | Load config, initialize, run, shutdown hook |
| Test | Smoke test — starts without exception |

**Final Checkpoint:** 93+ tests passing

---

# Summary

| Phase | Focus | Duration | Tests Added |
|-------|-------|----------|-------------|
| 0 | Project Setup | 1h | 0 |
| 1 | Core Mathematics | 4h | 19 |
| 2 | Domain Model | 3h | 14 |
| 3 | Physics Engine | 8h | 18 |
| 4 | Time System | 3h | 8 |
| 5 | External Data | 5h | 12 |
| 6 | Validation | 3h | 10 |
| 7 | API Layer | 4h | 8 |
| 8 | Integration | 3h | 4+ |
| **Total** | | **34h** | **93+** |

---

# Success Criteria for MVP

| Criterion | Target |
|-----------|--------|
| Compiles and runs | ✓ |
| Fetches live data from JPL at startup | ✓ |
| Integrates Sun-Earth system | ✓ |
| Position error after 1 day | < 100 km |
| Position error after 30 days | < 1000 km |
| Energy drift after 1 year | < 10⁻¹⁰ relative |
| All tests pass | 93+ green |

---

# Dependencies Between Steps

```
Phase 0 (Setup)
    │
    ▼
Phase 1 (Vector3D, Constants)
    │
    ▼
Phase 2 (Body, StateVector, SystemState)
    │
    ├────────────────────┐
    ▼                    ▼
Phase 3 (Physics)    Phase 4 (Time)
    │                    │
    └────────┬───────────┘
             ▼
      Phase 5 (JPL Client)
             │
             ▼
      Phase 6 (Validation)
             │
             ▼
      Phase 7 (API)
             │
             ▼
      Phase 8 (Integration)
```

---

# Risk Mitigation

| Risk | Mitigation |
|------|------------|
| JPL API unavailable | Cache sample responses for tests |
| Network timeout | Configurable timeout, retry logic |
| Floating-point drift | Compensated summation (Phase 3) |
| Integration accuracy | Validate against JPL at each phase |

---

# Post-MVP Roadmap

| Version | Features |
|---------|----------|
| 0.2 | Add Moon |
| 0.3 | Add inner planets (Mercury, Venus, Mars) |
| 0.4 | Relativistic corrections (1PN) |
| 0.5 | Outer planets |
| 0.6 | Persistence (H2 snapshots) |
| 0.7 | Sync mechanism |
| 1.0 | Full system with all features |
