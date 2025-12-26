# N-Body Solar System Simulator
## Complete Technical Specification v1.0

---

# Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Overview](#2-system-overview)
3. [Physical Model](#3-physical-model)
4. [Mathematical Framework](#4-mathematical-framework)
5. [Numerical Integration](#5-numerical-integration)
6. [Reference Frames & Coordinate Systems](#6-reference-frames--coordinate-systems)
7. [Time Systems](#7-time-systems)
8. [Architecture](#8-architecture)
9. [Data Model](#9-data-model)
10. [External Data Integration](#10-external-data-integration)
11. [State Management & Persistence](#11-state-management--persistence)
12. [API Specification](#12-api-specification)
13. [Validation & Monitoring](#13-validation--monitoring)
14. [Configuration](#14-configuration)
15. [Build & Deployment](#15-build--deployment)
16. [Implementation Roadmap](#16-implementation-roadmap)
17. [Appendices](#17-appendices)

---

# 1. Executive Summary

## 1.1 Purpose

This document specifies a real-time computational model of the Solar System that:

- Computes celestial body positions from **first principles** (fundamental physics)
- Does **not** rely on pre-computed ephemerides, position tables, or astronomy libraries
- Acquires initial state vectors from live data sources at startup
- Propagates the system state using pure N-body gravitational mechanics
- Periodically synchronizes with external observations for drift correction

## 1.2 Core Philosophy

```
TRUTH SOURCE AT t=0:     Live observational data (JPL Horizons API)
TRUTH SOURCE AT t>0:     Fundamental physics (Newton + corrections)
VALIDATION:              Periodic comparison with observations
```

## 1.3 Scope

| Phase | Bodies | Features |
|-------|--------|----------|
| MVP | Sun, Earth | Core physics engine, basic integration |
| Phase 2 | + Moon, inner planets | Lunar perturbations, Mercury precession |
| Phase 3 | + Outer planets | Full N-body interactions |
| Phase 4 | + Major moons | Satellite systems |
| Phase 5 | + Dwarf planets, asteroids | Extensible body registry |

## 1.4 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Reference frame | Barycentric (ICRS) | Physically correct for N-body |
| Internal time | TDB | Proper time for solar system dynamics |
| Integrator | Symplectic (Yoshida 8th order) | Long-term energy conservation |
| Relativistic corrections | Post-Newtonian (1PN) | Required for Mercury, high accuracy |
| State persistence | Embedded database (H2) | Snapshots for time travel |

---

# 2. System Overview

## 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SOLAR SYSTEM SIMULATOR                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────────┐  │
│  │   DATA LAYER    │      │  PHYSICS LAYER  │      │   SERVICE LAYER     │  │
│  │                 │      │                 │      │                     │  │
│  │ ┌─────────────┐ │      │ ┌─────────────┐ │      │ ┌─────────────────┐ │  │
│  │ │ JPL Client  │ │      │ │  N-Body     │ │      │ │  Query Engine   │ │  │
│  │ └─────────────┘ │      │ │  Engine     │ │      │ └─────────────────┘ │  │
│  │ ┌─────────────┐ │      │ └─────────────┘ │      │ ┌─────────────────┐ │  │
│  │ │ State Store │ │      │ ┌─────────────┐ │      │ │  Event Finder   │ │  │
│  │ └─────────────┘ │      │ │  Force      │ │      │ └─────────────────┘ │  │
│  │ ┌─────────────┐ │      │ │  Models     │ │      │ ┌─────────────────┐ │  │
│  │ │ Snapshot DB │ │      │ └─────────────┘ │      │ │  Validation     │ │  │
│  │ └─────────────┘ │      │ ┌─────────────┐ │      │ └─────────────────┘ │  │
│  │                 │      │ │  Integrator │ │      │                     │  │
│  │                 │      │ └─────────────┘ │      │                     │  │
│  └────────┬────────┘      └────────┬────────┘      └──────────┬──────────┘  │
│           │                        │                          │             │
│           └────────────────────────┼──────────────────────────┘             │
│                                    │                                        │
│                          ┌─────────┴─────────┐                              │
│                          │   TIME ENGINE     │                              │
│                          │  (TDB/UTC/Unix)   │                              │
│                          └───────────────────┘                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2.2 Component Responsibilities

| Component | Responsibility | Dependencies |
|-----------|---------------|--------------|
| **JPL Client** | Fetch initial state vectors from Horizons API | HTTP client |
| **State Store** | In-memory current state of all bodies | None |
| **Snapshot DB** | Historical state persistence | H2 database |
| **N-Body Engine** | Orchestrate force computation + integration | Force Models, Integrator |
| **Force Models** | Compute accelerations (gravity, relativity, etc.) | None |
| **Integrator** | Advance state by timestep | None |
| **Time Engine** | Time conversions, scaling, epoch management | None |
| **Query Engine** | Answer position/velocity queries | State Store, Snapshot DB |
| **Event Finder** | Find conjunctions, eclipses, etc. | Query Engine |
| **Validation** | Compare model vs observations, compute metrics | JPL Client |

## 2.3 Data Flow

```
STARTUP:
  JPL API ──────► JPL Client ──────► State Store ──────► Snapshot DB
                                          │
                                          ▼
RUNTIME:                            ┌───────────┐
                                    │ Time      │
                                    │ Engine    │
                                    └─────┬─────┘
                                          │ dt
                                          ▼
  State Store ◄────── Integrator ◄────── N-Body Engine
       │                                      ▲
       │                                      │
       ▼                                      │
  Snapshot DB                          Force Models
       │
       ▼
  Query Engine ──────► API Response

SYNC:
  JPL API ──────► Validation ──────► Correction ──────► State Store
```

---

# 3. Physical Model

## 3.1 Fundamental Equations of Motion

The system evolves according to Newton's second law with gravitational and relativistic forces:

```
d²rᵢ/dt² = aᵢ,Newton + aᵢ,1PN + aᵢ,J₂ + aᵢ,other
```

Where:
- `rᵢ` = position vector of body i in barycentric coordinates
- `aᵢ,Newton` = Newtonian gravitational acceleration
- `aᵢ,1PN` = First post-Newtonian (relativistic) correction
- `aᵢ,J₂` = Oblateness correction (for Sun, planets)
- `aᵢ,other` = Other perturbations (radiation pressure, etc.)

## 3.2 Newtonian Gravitation

For N bodies, the Newtonian acceleration on body i is:

```
         N
aᵢ,Newton = Σ  G·mⱼ·(rⱼ - rᵢ) / |rⱼ - rᵢ|³     (j ≠ i)
        j=1
```

Where:
- `G` = 6.67430 × 10⁻¹¹ m³/(kg·s²)  (gravitational constant)
- `mⱼ` = mass of body j
- `rᵢ`, `rⱼ` = position vectors of bodies i and j

**Implementation note**: Use `GM` products (gravitational parameters) directly rather than `G` and `m` separately, as `GM` is known to much higher precision than `G` or `m` individually.

### 3.2.1 Gravitational Parameters (GM values)

| Body | GM (km³/s²) | Source |
|------|-------------|--------|
| Sun | 1.32712440041279419 × 10¹¹ | DE440 |
| Mercury | 2.2031868551 × 10⁴ | DE440 |
| Venus | 3.24858592000 × 10⁵ | DE440 |
| Earth | 3.98600435507 × 10⁵ | DE440 |
| Moon | 4.902800118 × 10³ | DE440 |
| Mars | 4.2828375816 × 10⁴ | DE440 |
| Jupiter | 1.26712764100000 × 10⁸ | DE440 |
| Saturn | 3.79405852000000 × 10⁷ | DE440 |
| Uranus | 5.7945490070 × 10⁶ | DE440 |
| Neptune | 6.8365271005 × 10⁶ | DE440 |
| Pluto | 8.699633 × 10² | DE440 |

## 3.3 Post-Newtonian Correction (1PN)

The first post-Newtonian correction accounts for general relativistic effects. For body i accelerated by body j:

```
aᵢ,1PN = (GM_j / c²r²ᵢⱼ) × {
    n̂ᵢⱼ × [
        (4GM_j/rᵢⱼ - v²ᵢ + 2v²ⱼ - 4(vᵢ·vⱼ) + 3/2(n̂ᵢⱼ·vⱼ)²)
    ]
    + (vᵢ - vⱼ) × [4(n̂ᵢⱼ·vᵢ) - 3(n̂ᵢⱼ·vⱼ)]
}
```

Where:
- `c` = 299792.458 km/s (speed of light)
- `rᵢⱼ` = |rⱼ - rᵢ| (distance between bodies)
- `n̂ᵢⱼ` = (rⱼ - rᵢ)/rᵢⱼ (unit vector from i to j)
- `vᵢ`, `vⱼ` = velocity vectors

**Simplification for Solar System**: Since the Sun dominates, the primary 1PN contribution comes from Sun-planet interactions. The full expression for the Sun's influence on planet i:

```
aᵢ,1PN,Sun = (GM_Sun / c²r²) × {
    r̂ × [(4GM_Sun/r) - v²ᵢ]
    + 4(r̂·vᵢ)vᵢ
}
```

Where `r` is the heliocentric distance and `r̂` is the unit vector toward the Sun.

## 3.4 Oblateness (J₂) Correction

For an oblate body (like the Sun or Jupiter), the gravitational potential includes a J₂ term:

```
aᵢ,J₂ = -(3/2) × (GM_j × J₂,j × R²_j / r⁵) × {
    r × [5(r·ẑ)²/r² - 1]
    - 2(r·ẑ)ẑ
}
```

Where:
- `J₂,j` = second zonal harmonic coefficient
- `R_j` = equatorial radius of body j
- `ẑ` = unit vector along body's rotation axis
- `r` = position vector from body j to body i

### 3.4.1 J₂ Values

| Body | J₂ | Equatorial Radius (km) |
|------|-----|----------------------|
| Sun | 2.0 × 10⁻⁷ | 696,000 |
| Earth | 1.08263 × 10⁻³ | 6,378.137 |
| Jupiter | 1.4736 × 10⁻² | 71,492 |
| Saturn | 1.6298 × 10⁻² | 60,268 |

## 3.5 Solar Radiation Pressure (Optional, for small bodies)

For small bodies with significant area-to-mass ratio:

```
aᵢ,SRP = -(L_Sun × Cᵣ × A / (4π × c × m × r²)) × r̂
```

Where:
- `L_Sun` = 3.828 × 10²⁶ W (solar luminosity)
- `Cᵣ` = radiation pressure coefficient (≈1.0-2.0)
- `A` = cross-sectional area
- `m` = mass of body
- `r` = heliocentric distance
- `c` = speed of light

**Note**: Only relevant for asteroids, comets, spacecraft. Negligible for planets.

## 3.6 Force Model Configuration

```java
public enum ForceModel {
    NEWTONIAN,          // Basic N-body gravity only
    NEWTONIAN_J2,       // + oblateness
    RELATIVISTIC,       // + 1PN corrections
    RELATIVISTIC_J2,    // + both
    FULL                // + all available corrections
}
```

Default: `RELATIVISTIC_J2` for maximum accuracy.

---

# 4. Mathematical Framework

## 4.1 State Vector Representation

The state of body i at time t is represented as:

```
Xᵢ(t) = [rᵢ(t), vᵢ(t)] = [x, y, z, vₓ, vᵧ, vᵤ]ᵢ
```

The complete system state for N bodies:

```
X(t) = [X₁(t), X₂(t), ..., Xₙ(t)]
```

Total state dimension: 6N scalars.

## 4.2 Equations of Motion in First-Order Form

Converting the second-order system to first-order:

```
drᵢ/dt = vᵢ
dvᵢ/dt = aᵢ(r₁, r₂, ..., rₙ, v₁, v₂, ..., vₙ, t)
```

Or in matrix form:

```
dX/dt = F(X, t)
```

Where F contains:
- Position derivatives = velocities (trivial)
- Velocity derivatives = accelerations (computed from force models)

## 4.3 Conservation Laws

### 4.3.1 Total Energy

```
E = T + U

T = (1/2) Σᵢ mᵢvᵢ²                              (kinetic energy)

U = -Σᵢ Σⱼ₍ⱼ₎ᵢ₎ (Gmᵢmⱼ / rᵢⱼ)                    (potential energy)
```

**Conservation criterion**: |ΔE/E₀| should remain < 10⁻¹⁰ per century with symplectic integration.

### 4.3.2 Total Linear Momentum

```
P = Σᵢ mᵢvᵢ
```

In barycentric coordinates, P should remain constant (and small, ideally zero).

### 4.3.3 Total Angular Momentum

```
L = Σᵢ mᵢ(rᵢ × vᵢ)
```

Should remain constant in direction and magnitude.

## 4.4 Orbital Elements (for output/analysis)

Given state vector (r, v), compute Keplerian elements:

```
a   = semi-major axis        = -μ / (2ε)
e   = eccentricity           = |e⃗|
i   = inclination            = arccos(ĥ·ẑ)
Ω   = longitude of ascending node  = atan2(n̂·ŷ, n̂·x̂)
ω   = argument of perihelion = atan2(e⃗·ẑ/sin(i), e⃗·n̂)
ν   = true anomaly           = atan2(h(r·v), h²-μr)
M   = mean anomaly           (from ν via eccentric anomaly E)
```

Where:
- `μ` = GM of central body
- `ε` = v²/2 - μ/r (specific orbital energy)
- `h⃗` = r × v (specific angular momentum)
- `ĥ` = h⃗/|h⃗|
- `n⃗` = ẑ × h⃗ (node vector)
- `e⃗` = (v × h⃗)/μ - r̂ (eccentricity vector)

### 4.4.1 State Vector from Orbital Elements

Given (a, e, i, Ω, ω, ν), compute (r, v):

```
p = a(1 - e²)                           // semi-latus rectum
r_mag = p / (1 + e·cos(ν))              // orbital radius

// Position in perifocal frame
r_pqw = r_mag × [cos(ν), sin(ν), 0]ᵀ

// Velocity in perifocal frame
v_pqw = √(μ/p) × [-sin(ν), e + cos(ν), 0]ᵀ

// Rotation matrix: perifocal → inertial
R = R₃(-Ω) × R₁(-i) × R₃(-ω)

// Transform to inertial frame
r = R × r_pqw
v = R × v_pqw
```

Where R₁, R₃ are rotation matrices about x and z axes.

---

# 5. Numerical Integration

## 5.1 Integrator Selection

### 5.1.1 Requirements

| Requirement | Priority | Rationale |
|-------------|----------|-----------|
| Long-term stability | Critical | Multi-century integrations |
| Energy conservation | Critical | Physical accuracy |
| Momentum conservation | Critical | Barycenter stability |
| Efficiency | High | Real-time operation |
| Accuracy | High | Observation matching |

### 5.1.2 Comparison of Methods

| Method | Order | Energy Drift | Symplectic | Cost/step |
|--------|-------|--------------|------------|-----------|
| Euler | 1 | O(h) | No | 1 eval |
| RK4 | 4 | O(h⁴) | No | 4 evals |
| Leapfrog | 2 | Bounded | Yes | 1 eval |
| Yoshida 4 | 4 | Bounded | Yes | 3 evals |
| Yoshida 6 | 6 | Bounded | Yes | 7 evals |
| **Yoshida 8** | **8** | **Bounded** | **Yes** | **15 evals** |

**Selection: Yoshida 8th order symplectic integrator**

Rationale:
- Symplectic → bounded energy error over arbitrary time
- 8th order → high accuracy per step, allows larger timesteps
- Well-suited for Hamiltonian systems (planetary dynamics)

## 5.2 Yoshida 8th Order Integrator

### 5.2.1 Theory

Symplectic integrators exactly preserve the symplectic structure of Hamiltonian systems, leading to bounded energy error rather than secular drift.

The Yoshida method composes leapfrog steps with carefully chosen coefficients.

### 5.2.2 Coefficients

For the 8th order integrator (Solution A from Yoshida 1990):

```java
// Position update coefficients (d)
d[0] =  0.311790812418427e0
d[1] = -0.155946803821447e1
d[2] = -0.167896928259640e1
d[3] =  0.166335809963315e1
d[4] = -0.106458714789183e1
d[5] =  0.136934946416871e1
d[6] =  0.629030650210433e0
d[7] =  0.311790812418427e0

// Velocity update coefficients (c)
c[0] =  0.623581625817660e0
c[1] = -0.100984744834850e1
c[2] = -0.698690738326960e0
c[3] =  0.169235252729290e1
c[4] = -0.203339175067820e1
c[5] =  0.200687459421740e1
c[6] = -0.129932513636650e0
c[7] =  0.0e0
```

### 5.2.3 Algorithm

```
function yoshida8_step(bodies, dt):
    for i = 0 to 7:
        // Position drift
        for each body b:
            b.position += c[i] * dt * b.velocity
        
        // Velocity kick (if d[i] ≠ 0)
        if d[i] ≠ 0:
            accelerations = compute_accelerations(bodies)
            for each body b:
                b.velocity += d[i] * dt * accelerations[b]
    
    return bodies
```

### 5.2.4 Alternative: SABA Integrators

For cases requiring even better energy conservation, SABA (Symmetric, Adaptive, Bounded, Accurate) integrators can be used:

```
SABA2: 2nd order, 2 stages
SABA4: 4th order, 5 stages  
SABA6: 6th order, 9 stages
```

These are particularly good when perturbations (1PN, J₂) are small compared to Keplerian motion.

## 5.3 Timestep Selection

### 5.3.1 Fixed Timestep Approach

For maximum reproducibility and symplectic structure preservation:

```
dt_base = 0.125 days = 10800 seconds
```

Rationale:
- Small enough to resolve Mercury's 88-day orbit (≈700 steps)
- Large enough for efficiency
- Power of 2 for clean arithmetic

### 5.3.2 Adaptive Considerations

If adaptive stepping is needed, use reversible step size selection:

```
dt = dt_base × min(1, (r_min / r_critical)^0.5)
```

Where `r_min` is the minimum pairwise distance and `r_critical` is a threshold (e.g., 0.1 AU for close encounters).

**Warning**: Adaptive stepping breaks exact symplecticity. Use only when necessary.

## 5.4 Error Analysis

### 5.4.1 Truncation Error

For Yoshida 8, the local truncation error is O(h⁹) and global error is O(h⁸).

Expected position error per step:
```
ε_step ≈ C × h⁸ × |a_max|
```

For h = 0.125 days and |a_max| ≈ 0.01 km/s²:
```
ε_step ≈ 10⁻¹⁸ km per step
```

Accumulated over 1 year (≈3000 steps):
```
ε_year ≈ 10⁻¹⁵ km ≈ 10⁻⁹ AU
```

This is far below observational precision.

### 5.4.2 Floating-Point Error

Using IEEE 754 double precision (64-bit):
- Significand: 53 bits ≈ 15.9 decimal digits
- Machine epsilon: 2.22 × 10⁻¹⁶

For positions in km (AU scale ≈ 10⁸ km):
```
Float precision ≈ 10⁸ × 10⁻¹⁶ = 10⁻⁸ km = 10 meters
```

Over many operations, this accumulates. Mitigation:
- Use compensated summation (Kahan)
- Regularize coordinates near singularities

### 5.4.3 Long-Term Drift

Symplectic integrators bound energy error but not position error. Over centuries:
- Energy drift: Bounded, oscillatory
- Position drift: Grows as √t (random walk)
- Angle drift: Linear in t

Expected angular drift after 100 years: < 1 arcsecond for outer planets, < 10 arcseconds for Mercury.

## 5.5 Compensated Summation (Kahan Algorithm)

To reduce floating-point accumulation errors:

```java
class CompensatedAccumulator {
    private double sum = 0.0;
    private double compensation = 0.0;
    
    public void add(double value) {
        double y = value - compensation;
        double t = sum + y;
        compensation = (t - sum) - y;
        sum = t;
    }
    
    public double getSum() {
        return sum;
    }
}
```

Use for:
- Position updates
- Energy calculations
- Any long-running summation

---

# 6. Reference Frames & Coordinate Systems

## 6.1 Primary Frame: ICRS (Barycentric)

The International Celestial Reference System (ICRS):
- Origin: Solar System Barycenter
- Axes: Fixed relative to distant quasars
- Practical realization: ICRF3 (International Celestial Reference Frame 3)

### 6.1.1 Axis Orientation (J2000.0 aligned)

```
X-axis: Toward vernal equinox of J2000.0
Y-axis: In equatorial plane, 90° east of X
Z-axis: Toward celestial north pole of J2000.0
```

### 6.1.2 Why Barycentric?

In heliocentric coordinates, the Sun accelerates due to planetary gravity, introducing pseudo-forces. Barycentric coordinates are inertial (to first approximation), making N-body equations simpler.

## 6.2 Ecliptic Coordinates

### 6.2.1 Definition

```
X_ecl = X_eq
Y_ecl = Y_eq × cos(ε) + Z_eq × sin(ε)
Z_ecl = -Y_eq × sin(ε) + Z_eq × cos(ε)
```

Where ε = obliquity of the ecliptic = 23.439281° (J2000.0)

### 6.2.2 Transformation Matrix (Equatorial → Ecliptic)

```
        ┌ 1      0        0     ┐
R_eq→ecl = │ 0    cos(ε)   sin(ε) │
        └ 0   -sin(ε)   cos(ε) ┘
```

Inverse (Ecliptic → Equatorial): transpose of above.

## 6.3 Spherical Coordinates

### 6.3.1 From Cartesian (x, y, z) to Spherical (r, λ, β)

```
r = √(x² + y² + z²)           // Distance
λ = atan2(y, x)               // Longitude (0 to 2π)
β = asin(z / r)               // Latitude (-π/2 to π/2)
```

### 6.3.2 Right Ascension / Declination (Equatorial)

```
RA = atan2(y_eq, x_eq)        // Right Ascension (0 to 24h)
Dec = asin(z_eq / r)          // Declination (-90° to +90°)
```

### 6.3.3 Ecliptic Longitude / Latitude

```
λ_ecl = atan2(y_ecl, x_ecl)   // Ecliptic longitude
β_ecl = asin(z_ecl / r)       // Ecliptic latitude
```

## 6.4 Heliocentric Coordinates

For queries relative to Sun:

```
r_helio = r_bary - r_Sun
v_helio = v_bary - v_Sun
```

Where r_Sun, v_Sun are the Sun's barycentric state.

## 6.5 Topocentric Coordinates

For Earth-based observer at (lon, lat, alt):

```
r_topo = r_body - r_Earth - r_observer
```

Where r_observer is computed from Earth's rotation.

### 6.5.1 Observer Position in Earth-Fixed Frame

```
// WGS84 ellipsoid
a_E = 6378.137 km              // Equatorial radius
f = 1/298.257223563            // Flattening
b_E = a_E(1-f)                 // Polar radius

// Observer in Earth-fixed (ITRF)
N = a_E / √(1 - e²sin²(lat))   // Radius of curvature
x_ITRF = (N + alt) × cos(lat) × cos(lon)
y_ITRF = (N + alt) × cos(lat) × sin(lon)
z_ITRF = (N(1-e²) + alt) × sin(lat)
```

### 6.5.2 ITRF to ICRS Transformation

```
r_ICRS = R_precession × R_nutation × R_rotation × r_ITRF
```

This requires:
- Earth Orientation Parameters (EOP)
- Precession-nutation model (IAU 2006)

**Simplification for MVP**: Use a basic rotation for Earth's axial tilt and current sidereal time.

## 6.6 Reference Frame API

```java
public enum ReferenceFrame {
    ICRS_BARYCENTRIC,    // Primary internal frame
    ICRS_HELIOCENTRIC,   // Sun-centered, ICRS axes
    ECLIPTIC_BARYCENTRIC,
    ECLIPTIC_HELIOCENTRIC,
    EQUATORIAL_GEOCENTRIC,
    TOPOCENTRIC          // Requires observer location
}
```

---

# 7. Time Systems

## 7.1 Overview of Astronomical Time Scales

| Scale | Full Name | Description |
|-------|-----------|-------------|
| TAI | International Atomic Time | Continuous atomic time |
| UTC | Coordinated Universal Time | Civil time, has leap seconds |
| TT | Terrestrial Time | Time on Earth's geoid |
| TDB | Barycentric Dynamical Time | Time for solar system dynamics |
| UT1 | Universal Time | Based on Earth rotation |

### 7.1.1 Relationships

```
TT = TAI + 32.184 seconds
TDB ≈ TT + periodic terms (max ±1.7 ms)
UTC = TAI - (leap seconds)        // As of 2024: 37 leap seconds
```

## 7.2 Internal Time: TDB

The physics engine operates in TDB (Barycentric Dynamical Time):
- Independent of Earth's rotation
- Proper coordinate time for barycentric calculations
- Free from leap seconds

### 7.2.1 TDB-TT Difference

```
TDB - TT = 0.001657 sin(g) + 0.000022 sin(L_J) + ...

Where:
g = 357.53° + 0.9856003°(JD - 2451545.0)    // Mean anomaly of Earth
L_J = 246.11° + 0.9025179°(JD - 2451545.0)  // Mean longitude of Jupiter
```

For most applications, |TDB - TT| < 2 ms, which can be ignored for planetary positions. For sub-millisecond timing, use the full expression.

## 7.3 Julian Date

### 7.3.1 Definition

Julian Date (JD): Continuous count of days since noon, January 1, 4713 BCE (Julian calendar).

```
JD 2451545.0 = January 1, 2000, 12:00:00 TT (J2000.0 epoch)
```

### 7.3.2 Modified Julian Date

```
MJD = JD - 2400000.5
```

Starts at midnight, smaller numbers.

### 7.3.3 Julian Date from Calendar Date

```java
public static double toJulianDate(int year, int month, int day, 
                                  int hour, int minute, double second) {
    // Algorithm from Meeus, "Astronomical Algorithms"
    if (month <= 2) {
        year -= 1;
        month += 12;
    }
    
    int A = year / 100;
    int B = 2 - A + A / 4;  // Gregorian correction
    
    double JD = Math.floor(365.25 * (year + 4716))
              + Math.floor(30.6001 * (month + 1))
              + day + B - 1524.5
              + (hour + minute/60.0 + second/3600.0) / 24.0;
    
    return JD;
}
```

### 7.3.4 Calendar Date from Julian Date

```java
public static int[] toCalendarDate(double JD) {
    // Returns [year, month, day, hour, minute, second]
    double Z = Math.floor(JD + 0.5);
    double F = (JD + 0.5) - Z;
    
    double A;
    if (Z < 2299161) {
        A = Z;
    } else {
        double alpha = Math.floor((Z - 1867216.25) / 36524.25);
        A = Z + 1 + alpha - Math.floor(alpha / 4);
    }
    
    double B = A + 1524;
    int C = (int) Math.floor((B - 122.1) / 365.25);
    int D = (int) Math.floor(365.25 * C);
    int E = (int) Math.floor((B - D) / 30.6001);
    
    int day = (int) (B - D - Math.floor(30.6001 * E));
    int month = (E < 14) ? E - 1 : E - 13;
    int year = (month > 2) ? C - 4716 : C - 4715;
    
    double dayFraction = F * 24.0;
    int hour = (int) dayFraction;
    int minute = (int) ((dayFraction - hour) * 60);
    double second = ((dayFraction - hour) * 60 - minute) * 60;
    
    return new int[] {year, month, day, hour, minute, (int) second};
}
```

## 7.4 Unix Timestamp Conversion

### 7.4.1 Unix to JD (TT)

```java
public static double unixToJD_TT(long unixSeconds) {
    // Unix epoch: 1970-01-01 00:00:00 UTC
    // JD of Unix epoch: 2440587.5
    // Need to account for leap seconds (UTC to TAI) then TAI to TT
    
    double JD_UTC = 2440587.5 + unixSeconds / 86400.0;
    
    // UTC to TAI: add leap seconds
    int leapSeconds = getLeapSeconds(JD_UTC);
    double JD_TAI = JD_UTC + leapSeconds / 86400.0;
    
    // TAI to TT: add 32.184 seconds
    double JD_TT = JD_TAI + 32.184 / 86400.0;
    
    return JD_TT;
}
```

### 7.4.2 JD (TT) to Unix

```java
public static long jdTTToUnix(double JD_TT) {
    // TT to TAI
    double JD_TAI = JD_TT - 32.184 / 86400.0;
    
    // TAI to UTC: subtract leap seconds
    int leapSeconds = getLeapSeconds(JD_TT);  // Approximate
    double JD_UTC = JD_TAI - leapSeconds / 86400.0;
    
    return (long) ((JD_UTC - 2440587.5) * 86400.0);
}
```

### 7.4.3 Leap Second Table

```java
// Leap seconds introduced after 1972
private static final long[][] LEAP_SECONDS = {
    // {Unix timestamp when leap second added, cumulative leap seconds}
    {63072000L, 10},      // 1972-01-01
    {78796800L, 11},      // 1972-07-01
    // ... continue through current
    {1483228800L, 37},    // 2017-01-01 (most recent as of 2024)
};
```

## 7.5 Time Engine API

```java
public class TimeEngine {
    // Current simulation time (TDB)
    private double currentJD_TDB;
    
    // Time scaling
    private double timeScale = 1.0;  // 1.0 = real-time
    private boolean running = false;
    
    // Convert external time to internal
    public double unixToTDB(long unixTimestamp);
    public long tdbToUnix(double jd_TDB);
    
    // Advance simulation
    public void step(double dt_TDB);
    
    // Real-time operation
    public void setTimeScale(double scale);  // e.g., 86400 = 1 day/sec
    public double getTimeScale();
    
    // Jump to specific time
    public void setTime(double jd_TDB);
    public double getTime();
}
```

---

# 8. Architecture

## 8.1 Package Structure

```
com.solarsim
├── core
│   ├── Body.java                 // Celestial body representation
│   ├── StateVector.java          // Position + velocity
│   ├── SystemState.java          // Complete N-body state
│   └── PhysicalConstants.java    // G, c, AU, etc.
│
├── physics
│   ├── ForceModel.java           // Interface for force computation
│   ├── NewtonianGravity.java     // F = Gmm/r²
│   ├── PostNewtonian.java        // 1PN relativistic correction
│   ├── Oblateness.java           // J₂ perturbation
│   ├── CompositeForce.java       // Combines multiple force models
│   └── NBodyEngine.java          // Orchestrates physics
│
├── integrator
│   ├── Integrator.java           // Interface
│   ├── Yoshida8.java             // 8th order symplectic
│   ├── LeapFrog.java             // 2nd order (for testing)
│   └── RungeKutta4.java          // Classical RK4 (for comparison)
│
├── time
│   ├── TimeEngine.java           // Time management
│   ├── TimeScale.java            // TDB, TT, UTC conversions
│   ├── JulianDate.java           // JD utilities
│   └── LeapSecondTable.java      // UTC leap seconds
│
├── coordinates
│   ├── ReferenceFrame.java       // Frame enumeration
│   ├── CoordinateTransform.java  // Frame transformations
│   ├── SphericalCoords.java      // (r, λ, β) ↔ (x, y, z)
│   └── OrbitalElements.java      // Keplerian elements
│
├── data
│   ├── JPLHorizonsClient.java    // Fetch initial state from JPL
│   ├── BodyCatalog.java          // Physical parameters database
│   ├── SnapshotStore.java        // H2 database interface
│   └── StateSnapshot.java        // Serializable state
│
├── validation
│   ├── ConservationMonitor.java  // Energy, momentum tracking
│   ├── ResidualCalculator.java   // Model vs observation
│   ├── ValidationReport.java     // Metrics container
│   └── AlertSystem.java          // Threshold notifications
│
├── api
│   ├── QueryEngine.java          // Position/velocity queries
│   ├── EventFinder.java          // Conjunctions, eclipses
│   ├── SimulatorFacade.java      // Main entry point
│   └── dto/                      // Data transfer objects
│
└── config
    ├── SimulatorConfig.java      // Configuration POJO
    └── ConfigLoader.java         // YAML/properties loader
```

## 8.2 Core Classes

### 8.2.1 Body

```java
public final class Body {
    private final String id;              // e.g., "earth", "sun"
    private final String name;            // e.g., "Earth", "Sun"
    private final double gm;              // Gravitational parameter (km³/s²)
    private final double radius;          // Mean radius (km)
    private final double j2;              // Oblateness coefficient (0 if point mass)
    private final double equatorialRadius; // For J₂ calculation
    private final Vector3D poleDirection;  // Rotation axis in ICRS
    
    // Physical properties (immutable)
    public String getId();
    public String getName();
    public double getGM();
    public double getRadius();
    public double getJ2();
    public double getEquatorialRadius();
    public Vector3D getPoleDirection();
}
```

### 8.2.2 StateVector

```java
public final class StateVector {
    private final Vector3D position;    // km
    private final Vector3D velocity;    // km/s
    
    public StateVector(Vector3D position, Vector3D velocity);
    public StateVector(double x, double y, double z, 
                       double vx, double vy, double vz);
    
    public Vector3D getPosition();
    public Vector3D getVelocity();
    
    // Arithmetic
    public StateVector add(StateVector other);
    public StateVector subtract(StateVector other);
    public StateVector scale(double factor);
    
    // Derived quantities
    public double distanceTo(StateVector other);
    public double speed();
    public OrbitalElements toOrbitalElements(double centralGM);
}
```

### 8.2.3 Vector3D

```java
public final class Vector3D {
    private final double x, y, z;
    
    // Constructors
    public Vector3D(double x, double y, double z);
    public static Vector3D zero();
    
    // Basic operations
    public Vector3D add(Vector3D v);
    public Vector3D subtract(Vector3D v);
    public Vector3D scale(double s);
    public Vector3D negate();
    
    // Products
    public double dot(Vector3D v);
    public Vector3D cross(Vector3D v);
    
    // Magnitude
    public double magnitude();
    public double magnitudeSquared();
    public Vector3D normalize();
    
    // Components
    public double getX();
    public double getY();
    public double getZ();
    public double[] toArray();
}
```

### 8.2.4 SystemState

```java
public final class SystemState {
    private final double epoch;  // JD TDB
    private final Map<String, StateVector> states;  // body ID -> state
    
    public SystemState(double epoch, Map<String, StateVector> states);
    
    public double getEpoch();
    public StateVector getState(String bodyId);
    public Set<String> getBodyIds();
    public int getBodyCount();
    
    // Create modified copy
    public SystemState withState(String bodyId, StateVector state);
    public SystemState withEpoch(double newEpoch);
    
    // Bulk access for integrator
    public double[] toFlatArray();  // [x1,y1,z1,vx1,vy1,vz1, x2,y2,z2,...]
    public static SystemState fromFlatArray(double epoch, 
                                            List<String> bodyIds, 
                                            double[] flat);
}
```

## 8.3 Physics Layer

### 8.3.1 ForceModel Interface

```java
public interface ForceModel {
    /**
     * Compute acceleration on target body due to this force.
     * 
     * @param target Body experiencing the force
     * @param targetState Current state of target
     * @param system Complete system state
     * @param bodies All body definitions
     * @return Acceleration vector (km/s²)
     */
    Vector3D computeAcceleration(Body target, 
                                 StateVector targetState,
                                 SystemState system,
                                 Map<String, Body> bodies);
    
    /**
     * @return Human-readable name for logging
     */
    String getName();
}
```

### 8.3.2 NewtonianGravity

```java
public class NewtonianGravity implements ForceModel {
    @Override
    public Vector3D computeAcceleration(Body target, 
                                        StateVector targetState,
                                        SystemState system,
                                        Map<String, Body> bodies) {
        Vector3D totalAccel = Vector3D.zero();
        
        for (String otherId : system.getBodyIds()) {
            if (otherId.equals(target.getId())) continue;
            
            Body other = bodies.get(otherId);
            StateVector otherState = system.getState(otherId);
            
            Vector3D r = otherState.getPosition()
                                   .subtract(targetState.getPosition());
            double rMag = r.magnitude();
            double rMag3 = rMag * rMag * rMag;
            
            // a = GM * r / |r|³
            Vector3D accel = r.scale(other.getGM() / rMag3);
            totalAccel = totalAccel.add(accel);
        }
        
        return totalAccel;
    }
    
    @Override
    public String getName() {
        return "Newtonian Gravity";
    }
}
```

### 8.3.3 PostNewtonian

```java
public class PostNewtonian implements ForceModel {
    private static final double C = 299792.458;  // km/s
    private static final double C2 = C * C;
    
    @Override
    public Vector3D computeAcceleration(Body target, 
                                        StateVector targetState,
                                        SystemState system,
                                        Map<String, Body> bodies) {
        Vector3D totalAccel = Vector3D.zero();
        
        for (String otherId : system.getBodyIds()) {
            if (otherId.equals(target.getId())) continue;
            
            Body other = bodies.get(otherId);
            StateVector otherState = system.getState(otherId);
            
            Vector3D r = targetState.getPosition()
                                    .subtract(otherState.getPosition());
            double rMag = r.magnitude();
            Vector3D rHat = r.normalize();
            
            Vector3D vi = targetState.getVelocity();
            Vector3D vj = otherState.getVelocity();
            
            double vi2 = vi.magnitudeSquared();
            double vj2 = vj.magnitudeSquared();
            double viDotVj = vi.dot(vj);
            double rHatDotVj = rHat.dot(vj);
            
            double GM = other.getGM();
            double coeff = GM / (C2 * rMag * rMag);
            
            // 1PN terms
            double term1 = 4 * GM / rMag - vi2 + 2 * vj2 - 4 * viDotVj 
                         + 1.5 * rHatDotVj * rHatDotVj;
            double term2 = 4 * rHat.dot(vi) - 3 * rHatDotVj;
            
            Vector3D accel = rHat.scale(coeff * term1)
                                 .add(vi.subtract(vj).scale(coeff * term2));
            
            totalAccel = totalAccel.add(accel);
        }
        
        return totalAccel;
    }
    
    @Override
    public String getName() {
        return "Post-Newtonian (1PN)";
    }
}
```

### 8.3.4 CompositeForce

```java
public class CompositeForce implements ForceModel {
    private final List<ForceModel> models;
    
    public CompositeForce(ForceModel... models) {
        this.models = List.of(models);
    }
    
    @Override
    public Vector3D computeAcceleration(Body target, 
                                        StateVector targetState,
                                        SystemState system,
                                        Map<String, Body> bodies) {
        Vector3D total = Vector3D.zero();
        for (ForceModel model : models) {
            total = total.add(
                model.computeAcceleration(target, targetState, system, bodies)
            );
        }
        return total;
    }
    
    @Override
    public String getName() {
        return models.stream()
                     .map(ForceModel::getName)
                     .collect(Collectors.joining(" + "));
    }
}
```

### 8.3.5 NBodyEngine

```java
public class NBodyEngine {
    private final Map<String, Body> bodies;
    private final ForceModel forceModel;
    private final Integrator integrator;
    private SystemState currentState;
    
    public NBodyEngine(Map<String, Body> bodies,
                       ForceModel forceModel,
                       Integrator integrator,
                       SystemState initialState) {
        this.bodies = Map.copyOf(bodies);
        this.forceModel = forceModel;
        this.integrator = integrator;
        this.currentState = initialState;
    }
    
    /**
     * Advance the simulation by one timestep.
     * @param dt Time step in days
     */
    public void step(double dt) {
        currentState = integrator.step(currentState, dt, 
                                       this::computeAccelerations);
    }
    
    /**
     * Advance the simulation to a target time.
     * @param targetJD Target Julian Date (TDB)
     * @param maxStepDays Maximum step size
     */
    public void advanceTo(double targetJD, double maxStepDays) {
        while (currentState.getEpoch() < targetJD) {
            double remaining = targetJD - currentState.getEpoch();
            double dt = Math.min(remaining, maxStepDays);
            step(dt);
        }
    }
    
    /**
     * Compute accelerations for all bodies.
     */
    public Map<String, Vector3D> computeAccelerations(SystemState state) {
        Map<String, Vector3D> accels = new HashMap<>();
        
        for (String bodyId : state.getBodyIds()) {
            Body body = bodies.get(bodyId);
            StateVector bodyState = state.getState(bodyId);
            Vector3D accel = forceModel.computeAcceleration(
                body, bodyState, state, bodies
            );
            accels.put(bodyId, accel);
        }
        
        return accels;
    }
    
    public SystemState getState() {
        return currentState;
    }
    
    public void setState(SystemState state) {
        this.currentState = state;
    }
}
```

## 8.4 Integrator Layer

### 8.4.1 Integrator Interface

```java
@FunctionalInterface
public interface AccelerationFunction {
    Map<String, Vector3D> compute(SystemState state);
}

public interface Integrator {
    /**
     * Advance state by one timestep.
     * @param state Current state
     * @param dt Time step in days
     * @param accelFunc Function to compute accelerations
     * @return New state at t + dt
     */
    SystemState step(SystemState state, double dt, AccelerationFunction accelFunc);
    
    /**
     * @return Order of the method
     */
    int getOrder();
    
    /**
     * @return True if the integrator preserves symplectic structure
     */
    boolean isSymplectic();
}
```

### 8.4.2 Yoshida8 Implementation

```java
public class Yoshida8 implements Integrator {
    // Coefficients from Yoshida (1990), Solution A
    private static final double[] C = {
         0.311790812418427e0,
        -0.155946803821447e1,
        -0.167896928259640e1,
         0.166335809963315e1,
        -0.106458714789183e1,
         0.136934946416871e1,
         0.629030650210433e0,
         0.311790812418427e0
    };
    
    private static final double[] D = {
         0.623581625817660e0,
        -0.100984744834850e1,
        -0.698690738326960e0,
         0.169235252729290e1,
        -0.203339175067820e1,
         0.200687459421740e1,
        -0.129932513636650e0,
         0.0e0
    };
    
    // Conversion: days to seconds
    private static final double DAYS_TO_SECONDS = 86400.0;
    
    @Override
    public SystemState step(SystemState state, double dt, 
                            AccelerationFunction accelFunc) {
        double dtSeconds = dt * DAYS_TO_SECONDS;
        
        // Working copies of positions and velocities
        Map<String, Vector3D> positions = new HashMap<>();
        Map<String, Vector3D> velocities = new HashMap<>();
        
        for (String id : state.getBodyIds()) {
            StateVector sv = state.getState(id);
            positions.put(id, sv.getPosition());
            velocities.put(id, sv.getVelocity());
        }
        
        // Yoshida 8th order integration
        for (int i = 0; i < 8; i++) {
            // Velocity kick (if coefficient non-zero)
            if (Math.abs(D[i]) > 1e-20) {
                // Build temporary state for acceleration computation
                SystemState tempState = buildState(state.getEpoch(), 
                                                   positions, velocities);
                Map<String, Vector3D> accels = accelFunc.compute(tempState);
                
                for (String id : state.getBodyIds()) {
                    Vector3D v = velocities.get(id);
                    Vector3D a = accels.get(id);
                    velocities.put(id, v.add(a.scale(D[i] * dtSeconds)));
                }
            }
            
            // Position drift
            for (String id : state.getBodyIds()) {
                Vector3D r = positions.get(id);
                Vector3D v = velocities.get(id);
                positions.put(id, r.add(v.scale(C[i] * dtSeconds)));
            }
        }
        
        // Build final state
        double newEpoch = state.getEpoch() + dt;
        return buildState(newEpoch, positions, velocities);
    }
    
    private SystemState buildState(double epoch,
                                   Map<String, Vector3D> positions,
                                   Map<String, Vector3D> velocities) {
        Map<String, StateVector> states = new HashMap<>();
        for (String id : positions.keySet()) {
            states.put(id, new StateVector(positions.get(id), velocities.get(id)));
        }
        return new SystemState(epoch, states);
    }
    
    @Override
    public int getOrder() {
        return 8;
    }
    
    @Override
    public boolean isSymplectic() {
        return true;
    }
}
```

## 8.5 Time Engine

```java
public class TimeEngine {
    private double currentJD_TDB;
    private double timeScale;           // Ratio: simulation time / wall time
    private boolean running;
    private long lastWallTimeNanos;
    
    private final ScheduledExecutorService scheduler;
    private final List<Consumer<Double>> timeListeners;
    
    public TimeEngine(double initialJD_TDB) {
        this.currentJD_TDB = initialJD_TDB;
        this.timeScale = 1.0;
        this.running = false;
        this.scheduler = Executors.newSingleThreadScheduledExecutor();
        this.timeListeners = new CopyOnWriteArrayList<>();
    }
    
    // Time queries
    public double getCurrentJD_TDB() {
        return currentJD_TDB;
    }
    
    public long getCurrentUnixTime() {
        return TimeScale.tdbToUnix(currentJD_TDB);
    }
    
    // Time control
    public void setTime(double jd_TDB) {
        this.currentJD_TDB = jd_TDB;
        notifyListeners();
    }
    
    public void setTimeScale(double scale) {
        this.timeScale = scale;
    }
    
    public double getTimeScale() {
        return timeScale;
    }
    
    // Real-time operation
    public void start() {
        if (running) return;
        running = true;
        lastWallTimeNanos = System.nanoTime();
        
        scheduler.scheduleAtFixedRate(this::tick, 0, 10, TimeUnit.MILLISECONDS);
    }
    
    public void stop() {
        running = false;
    }
    
    private void tick() {
        if (!running) return;
        
        long now = System.nanoTime();
        double wallElapsedDays = (now - lastWallTimeNanos) / 1e9 / 86400.0;
        lastWallTimeNanos = now;
        
        double simElapsedDays = wallElapsedDays * timeScale;
        currentJD_TDB += simElapsedDays;
        
        notifyListeners();
    }
    
    // Listeners
    public void addTimeListener(Consumer<Double> listener) {
        timeListeners.add(listener);
    }
    
    private void notifyListeners() {
        for (Consumer<Double> listener : timeListeners) {
            listener.accept(currentJD_TDB);
        }
    }
    
    public void shutdown() {
        scheduler.shutdown();
    }
}
```

---

# 9. Data Model

## 9.1 Database Schema (H2)

```sql
-- State snapshots for time travel
CREATE TABLE state_snapshots (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    epoch_jd DOUBLE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    body_count INT NOT NULL,
    total_energy DOUBLE,
    CONSTRAINT uk_epoch UNIQUE (epoch_jd)
);

CREATE INDEX idx_snapshot_epoch ON state_snapshots(epoch_jd);

-- Individual body states within a snapshot
CREATE TABLE body_states (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    snapshot_id BIGINT NOT NULL,
    body_id VARCHAR(32) NOT NULL,
    pos_x DOUBLE NOT NULL,
    pos_y DOUBLE NOT NULL,
    pos_z DOUBLE NOT NULL,
    vel_x DOUBLE NOT NULL,
    vel_y DOUBLE NOT NULL,
    vel_z DOUBLE NOT NULL,
    CONSTRAINT fk_snapshot FOREIGN KEY (snapshot_id) 
        REFERENCES state_snapshots(id) ON DELETE CASCADE
);

CREATE INDEX idx_body_snapshot ON body_states(snapshot_id);

-- Validation records
CREATE TABLE validation_records (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    epoch_jd DOUBLE NOT NULL,
    body_id VARCHAR(32) NOT NULL,
    position_error_km DOUBLE,
    velocity_error_kms DOUBLE,
    angular_error_arcsec DOUBLE,
    source VARCHAR(64)
);

-- Conservation metrics
CREATE TABLE conservation_metrics (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    epoch_jd DOUBLE NOT NULL,
    total_energy DOUBLE NOT NULL,
    energy_drift_relative DOUBLE,
    momentum_x DOUBLE,
    momentum_y DOUBLE,
    momentum_z DOUBLE,
    angular_momentum_x DOUBLE,
    angular_momentum_y DOUBLE,
    angular_momentum_z DOUBLE
);
```

## 9.2 Snapshot Store Implementation

```java
public class SnapshotStore {
    private final DataSource dataSource;
    private final double snapshotIntervalDays;
    private double lastSnapshotEpoch = Double.NEGATIVE_INFINITY;
    
    public SnapshotStore(String jdbcUrl, double snapshotIntervalDays) {
        this.dataSource = createDataSource(jdbcUrl);
        this.snapshotIntervalDays = snapshotIntervalDays;
        initializeSchema();
    }
    
    /**
     * Save state if enough time has passed since last snapshot.
     */
    public void saveIfDue(SystemState state, double totalEnergy) {
        if (state.getEpoch() - lastSnapshotEpoch >= snapshotIntervalDays) {
            save(state, totalEnergy);
            lastSnapshotEpoch = state.getEpoch();
        }
    }
    
    /**
     * Force save a snapshot.
     */
    public void save(SystemState state, double totalEnergy) {
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            
            // Insert snapshot header
            long snapshotId;
            try (PreparedStatement ps = conn.prepareStatement(
                    "INSERT INTO state_snapshots (epoch_jd, body_count, total_energy) " +
                    "VALUES (?, ?, ?)",
                    Statement.RETURN_GENERATED_KEYS)) {
                ps.setDouble(1, state.getEpoch());
                ps.setInt(2, state.getBodyCount());
                ps.setDouble(3, totalEnergy);
                ps.executeUpdate();
                
                try (ResultSet rs = ps.getGeneratedKeys()) {
                    rs.next();
                    snapshotId = rs.getLong(1);
                }
            }
            
            // Insert body states
            try (PreparedStatement ps = conn.prepareStatement(
                    "INSERT INTO body_states " +
                    "(snapshot_id, body_id, pos_x, pos_y, pos_z, vel_x, vel_y, vel_z) " +
                    "VALUES (?, ?, ?, ?, ?, ?, ?, ?)")) {
                
                for (String bodyId : state.getBodyIds()) {
                    StateVector sv = state.getState(bodyId);
                    Vector3D pos = sv.getPosition();
                    Vector3D vel = sv.getVelocity();
                    
                    ps.setLong(1, snapshotId);
                    ps.setString(2, bodyId);
                    ps.setDouble(3, pos.getX());
                    ps.setDouble(4, pos.getY());
                    ps.setDouble(5, pos.getZ());
                    ps.setDouble(6, vel.getX());
                    ps.setDouble(7, vel.getY());
                    ps.setDouble(8, vel.getZ());
                    ps.addBatch();
                }
                ps.executeBatch();
            }
            
            conn.commit();
        } catch (SQLException e) {
            throw new RuntimeException("Failed to save snapshot", e);
        }
    }
    
    /**
     * Load the nearest snapshot to target epoch.
     * @return Snapshot, or empty if none found
     */
    public Optional<SystemState> loadNearest(double targetEpoch) {
        try (Connection conn = dataSource.getConnection()) {
            // Find nearest snapshot
            Long snapshotId = null;
            double foundEpoch = 0;
            
            try (PreparedStatement ps = conn.prepareStatement(
                    "SELECT id, epoch_jd FROM state_snapshots " +
                    "ORDER BY ABS(epoch_jd - ?) LIMIT 1")) {
                ps.setDouble(1, targetEpoch);
                try (ResultSet rs = ps.executeQuery()) {
                    if (rs.next()) {
                        snapshotId = rs.getLong("id");
                        foundEpoch = rs.getDouble("epoch_jd");
                    }
                }
            }
            
            if (snapshotId == null) {
                return Optional.empty();
            }
            
            // Load body states
            Map<String, StateVector> states = new HashMap<>();
            try (PreparedStatement ps = conn.prepareStatement(
                    "SELECT body_id, pos_x, pos_y, pos_z, vel_x, vel_y, vel_z " +
                    "FROM body_states WHERE snapshot_id = ?")) {
                ps.setLong(1, snapshotId);
                try (ResultSet rs = ps.executeQuery()) {
                    while (rs.next()) {
                        String bodyId = rs.getString("body_id");
                        Vector3D pos = new Vector3D(
                            rs.getDouble("pos_x"),
                            rs.getDouble("pos_y"),
                            rs.getDouble("pos_z")
                        );
                        Vector3D vel = new Vector3D(
                            rs.getDouble("vel_x"),
                            rs.getDouble("vel_y"),
                            rs.getDouble("vel_z")
                        );
                        states.put(bodyId, new StateVector(pos, vel));
                    }
                }
            }
            
            return Optional.of(new SystemState(foundEpoch, states));
        } catch (SQLException e) {
            throw new RuntimeException("Failed to load snapshot", e);
        }
    }
    
    /**
     * Find snapshots bracketing a target epoch (for interpolation).
     */
    public BracketingSnapshots findBracketing(double targetEpoch) {
        // Implementation returns before/after snapshots
        // ...
    }
}
```

---

# 10. External Data Integration

## 10.1 JPL Horizons API

### 10.1.1 API Endpoint

```
Base URL: https://ssd.jpl.nasa.gov/api/horizons.api
```

### 10.1.2 Request Parameters for State Vectors

| Parameter | Value | Description |
|-----------|-------|-------------|
| `format` | `json` | Response format |
| `COMMAND` | `'<body_id>'` | Target body (see below) |
| `CENTER` | `'500@0'` | Solar System Barycenter |
| `MAKE_EPHEM` | `'YES'` | Generate ephemeris |
| `EPHEM_TYPE` | `'VECTORS'` | State vectors output |
| `VEC_TABLE` | `'2'` | Position and velocity |
| `REF_PLANE` | `'FRAME'` | ICRF reference frame |
| `REF_SYSTEM` | `'ICRF'` | ICRS coordinates |
| `OUT_UNITS` | `'KM-S'` | km and km/s |
| `VEC_LABELS` | `'YES'` | Include labels |
| `START_TIME` | `'<datetime>'` | Start time (UTC) |
| `STOP_TIME` | `'<datetime>'` | End time (UTC) |
| `STEP_SIZE` | `'1'` | One output |

### 10.1.3 Body IDs

| Body | Horizons ID |
|------|-------------|
| Sun | `10` |
| Mercury | `199` |
| Venus | `299` |
| Earth | `399` |
| Moon | `301` |
| Mars | `499` |
| Jupiter | `599` |
| Saturn | `699` |
| Uranus | `799` |
| Neptune | `899` |
| Pluto | `999` |
| Earth-Moon Barycenter | `3` |
| Mars Barycenter | `4` |

### 10.1.4 Example Request

```
https://ssd.jpl.nasa.gov/api/horizons.api?format=json&COMMAND='399'&CENTER='500@0'&MAKE_EPHEM='YES'&EPHEM_TYPE='VECTORS'&VEC_TABLE='2'&REF_PLANE='FRAME'&REF_SYSTEM='ICRF'&OUT_UNITS='KM-S'&VEC_LABELS='YES'&START_TIME='2024-01-01 00:00'&STOP_TIME='2024-01-01 00:01'&STEP_SIZE='1'
```

### 10.1.5 Response Parsing

```json
{
  "result": "...\n$$SOE\n2460310.500000000 = A.D. 2024-Jan-01 00:00:00.0000 TDB \n X =-2.627..E+07 Y = 1.445..E+08 Z = 1.932..E+04\n VX=-2.97..E+01 VY=-5.37..E+00 VZ= 4.52..E-04\n$$EOE\n..."
}
```

Key markers:
- `$$SOE` = Start of Ephemeris
- `$$EOE` = End of Ephemeris
- Values in scientific notation

## 10.2 JPL Horizons Client Implementation

```java
public class JPLHorizonsClient {
    private static final String BASE_URL = 
        "https://ssd.jpl.nasa.gov/api/horizons.api";
    
    private final HttpClient httpClient;
    private final ObjectMapper jsonMapper;
    
    // Body ID mapping
    private static final Map<String, String> BODY_IDS = Map.ofEntries(
        Map.entry("sun", "10"),
        Map.entry("mercury", "199"),
        Map.entry("venus", "299"),
        Map.entry("earth", "399"),
        Map.entry("moon", "301"),
        Map.entry("mars", "499"),
        Map.entry("jupiter", "599"),
        Map.entry("saturn", "699"),
        Map.entry("uranus", "799"),
        Map.entry("neptune", "899"),
        Map.entry("pluto", "999")
    );
    
    public JPLHorizonsClient() {
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(30))
            .build();
        this.jsonMapper = new ObjectMapper();
    }
    
    /**
     * Fetch current state vector for a body.
     * @param bodyId Internal body identifier (e.g., "earth")
     * @return State vector in ICRS barycentric coordinates
     */
    public StateVector fetchCurrentState(String bodyId) throws IOException {
        Instant now = Instant.now();
        return fetchState(bodyId, now);
    }
    
    /**
     * Fetch state vector at a specific time.
     */
    public StateVector fetchState(String bodyId, Instant time) throws IOException {
        String horizonsId = BODY_IDS.get(bodyId.toLowerCase());
        if (horizonsId == null) {
            throw new IllegalArgumentException("Unknown body: " + bodyId);
        }
        
        DateTimeFormatter formatter = DateTimeFormatter
            .ofPattern("yyyy-MM-dd HH:mm")
            .withZone(ZoneOffset.UTC);
        
        String startTime = formatter.format(time);
        String stopTime = formatter.format(time.plusSeconds(60));
        
        String url = buildUrl(horizonsId, startTime, stopTime);
        
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(url))
            .timeout(Duration.ofSeconds(30))
            .GET()
            .build();
        
        HttpResponse<String> response = httpClient.send(
            request, HttpResponse.BodyHandlers.ofString()
        );
        
        if (response.statusCode() != 200) {
            throw new IOException("JPL API returned status " + response.statusCode());
        }
        
        return parseResponse(response.body());
    }
    
    /**
     * Fetch initial states for all configured bodies.
     */
    public Map<String, StateVector> fetchAllCurrentStates(Collection<String> bodyIds) 
            throws IOException {
        Map<String, StateVector> states = new HashMap<>();
        
        for (String bodyId : bodyIds) {
            StateVector state = fetchCurrentState(bodyId);
            states.put(bodyId, state);
            
            // Rate limiting: JPL recommends < 1 request/second
            try {
                Thread.sleep(1100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new IOException("Interrupted during fetch", e);
            }
        }
        
        return states;
    }
    
    private String buildUrl(String horizonsId, String startTime, String stopTime) {
        return BASE_URL + "?" + String.join("&",
            "format=json",
            "COMMAND='" + horizonsId + "'",
            "CENTER='500@0'",
            "MAKE_EPHEM='YES'",
            "EPHEM_TYPE='VECTORS'",
            "VEC_TABLE='2'",
            "REF_PLANE='FRAME'",
            "REF_SYSTEM='ICRF'",
            "OUT_UNITS='KM-S'",
            "VEC_LABELS='YES'",
            "START_TIME='" + URLEncoder.encode(startTime, StandardCharsets.UTF_8) + "'",
            "STOP_TIME='" + URLEncoder.encode(stopTime, StandardCharsets.UTF_8) + "'",
            "STEP_SIZE='1'"
        );
    }
    
    private StateVector parseResponse(String jsonResponse) throws IOException {
        JsonNode root = jsonMapper.readTree(jsonResponse);
        String result = root.get("result").asText();
        
        // Find data between $$SOE and $$EOE
        int soeIndex = result.indexOf("$$SOE");
        int eoeIndex = result.indexOf("$$EOE");
        
        if (soeIndex < 0 || eoeIndex < 0) {
            throw new IOException("Could not find ephemeris markers in response");
        }
        
        String ephemData = result.substring(soeIndex + 5, eoeIndex);
        
        // Parse position and velocity
        // Format: X = value Y = value Z = value VX = value VY = value VZ = value
        double x = parseValue(ephemData, "X");
        double y = parseValue(ephemData, "Y");
        double z = parseValue(ephemData, "Z");
        double vx = parseValue(ephemData, "VX");
        double vy = parseValue(ephemData, "VY");
        double vz = parseValue(ephemData, "VZ");
        
        return new StateVector(x, y, z, vx, vy, vz);
    }
    
    private double parseValue(String data, String label) throws IOException {
        // Pattern: "X = 1.234E+05" or "X =-1.234E+05"
        Pattern pattern = Pattern.compile(
            label + "\\s*=\\s*([+-]?\\d+\\.\\d+E[+-]?\\d+)"
        );
        Matcher matcher = pattern.matcher(data);
        
        if (!matcher.find()) {
            throw new IOException("Could not parse " + label + " from response");
        }
        
        return Double.parseDouble(matcher.group(1));
    }
}
```

## 10.3 Sync Manager

```java
public class SyncManager {
    private final JPLHorizonsClient jplClient;
    private final NBodyEngine engine;
    private final ValidationService validator;
    private final AlertSystem alertSystem;
    
    private final Duration syncInterval;
    private final double maxAllowedDivergenceKm;
    
    private final ScheduledExecutorService scheduler;
    
    public SyncManager(JPLHorizonsClient jplClient,
                       NBodyEngine engine,
                       ValidationService validator,
                       AlertSystem alertSystem,
                       Duration syncInterval,
                       double maxAllowedDivergenceKm) {
        this.jplClient = jplClient;
        this.engine = engine;
        this.validator = validator;
        this.alertSystem = alertSystem;
        this.syncInterval = syncInterval;
        this.maxAllowedDivergenceKm = maxAllowedDivergenceKm;
        this.scheduler = Executors.newSingleThreadScheduledExecutor();
    }
    
    public void startPeriodicSync() {
        scheduler.scheduleAtFixedRate(
            this::performSync,
            syncInterval.toMillis(),
            syncInterval.toMillis(),
            TimeUnit.MILLISECONDS
        );
    }
    
    public SyncResult performSync() {
        try {
            SystemState modelState = engine.getState();
            
            // Fetch current observed states
            Map<String, StateVector> observedStates = 
                jplClient.fetchAllCurrentStates(modelState.getBodyIds());
            
            // Calculate residuals
            Map<String, Double> positionErrors = new HashMap<>();
            double maxError = 0;
            String worstBody = null;
            
            for (String bodyId : modelState.getBodyIds()) {
                StateVector model = modelState.getState(bodyId);
                StateVector observed = observedStates.get(bodyId);
                
                double error = model.getPosition()
                                    .subtract(observed.getPosition())
                                    .magnitude();
                positionErrors.put(bodyId, error);
                
                if (error > maxError) {
                    maxError = error;
                    worstBody = bodyId;
                }
            }
            
            // Log validation record
            validator.recordValidation(modelState.getEpoch(), positionErrors);
            
            // Check divergence threshold
            if (maxError > maxAllowedDivergenceKm) {
                alertSystem.alert(AlertLevel.WARNING,
                    String.format("Model divergence %.2f km for %s exceeds threshold %.2f km",
                        maxError, worstBody, maxAllowedDivergenceKm));
                
                // Return result indicating correction needed
                return new SyncResult(SyncStatus.DIVERGENCE_DETECTED, 
                                     positionErrors, observedStates);
            }
            
            return new SyncResult(SyncStatus.OK, positionErrors, null);
            
        } catch (IOException e) {
            alertSystem.alert(AlertLevel.WARNING, 
                "Sync failed: " + e.getMessage());
            return new SyncResult(SyncStatus.FETCH_FAILED, null, null);
        }
    }
    
    /**
     * Apply correction from observed data.
     * Strategy: Reset to observed state.
     */
    public void applyCorrection(Map<String, StateVector> observedStates) {
        SystemState currentState = engine.getState();
        
        Map<String, StateVector> correctedStates = new HashMap<>();
        for (String bodyId : currentState.getBodyIds()) {
            correctedStates.put(bodyId, observedStates.get(bodyId));
        }
        
        SystemState correctedState = new SystemState(
            currentState.getEpoch(), 
            correctedStates
        );
        
        engine.setState(correctedState);
        
        alertSystem.alert(AlertLevel.INFO, "State corrected from observations");
    }
    
    public void shutdown() {
        scheduler.shutdown();
    }
}

public enum SyncStatus {
    OK,
    DIVERGENCE_DETECTED,
    FETCH_FAILED
}

public record SyncResult(
    SyncStatus status,
    Map<String, Double> positionErrors,
    Map<String, StateVector> observedStates
) {}
```

---

# 11. State Management & Persistence

## 11.1 State Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         STATE LIFECYCLE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STARTUP                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │  Check   │───▶│  Load    │───▶│ Validate │───▶│  Ready   │          │
│  │ Snapshot │    │ Snapshot │    │  State   │    │          │          │
│  └────┬─────┘    └──────────┘    └──────────┘    └──────────┘          │
│       │ not found                                                        │
│       ▼                                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                           │
│  │  Fetch   │───▶│Initialize│───▶│   Save   │                           │
│  │ JPL Data │    │  State   │    │ Snapshot │                           │
│  └──────────┘    └──────────┘    └──────────┘                           │
│                                                                          │
│  RUNTIME                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ Current  │───▶│Integrate │───▶│  Update  │───▶│ Snapshot │          │
│  │  State   │    │          │    │  State   │    │ (if due) │          │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│                                        │                                 │
│                                        ▼                                 │
│                                  ┌──────────┐                            │
│                                  │  Metrics │                            │
│                                  │ & Alerts │                            │
│                                  └──────────┘                            │
│                                                                          │
│  TIME TRAVEL                                                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │  Query   │───▶│   Find   │───▶│ Integrate│───▶│  Return  │          │
│  │ (t < now)│    │ Snapshot │    │  to t    │    │  State   │          │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## 11.2 Snapshot Strategy

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Snapshot interval | 1 day | Balance storage vs. re-integration time |
| Retention policy | 10 years | Reasonable historical range |
| Compression | GZIP | Reduce storage footprint |

Storage estimate:
- Per snapshot: ~10 bodies × 6 doubles × 8 bytes = 480 bytes + overhead ≈ 1 KB
- Per year: 365 KB
- 10 years: ~4 MB

## 11.3 Time Travel Implementation

```java
public class TimeTravelService {
    private final SnapshotStore snapshotStore;
    private final NBodyEngine engine;
    private final Map<String, Body> bodies;
    private final Integrator integrator;
    private final ForceModel forceModel;
    
    /**
     * Get system state at any historical time.
     */
    public SystemState getStateAt(double targetEpoch) {
        // Check if target is in the future
        SystemState current = engine.getState();
        if (targetEpoch >= current.getEpoch()) {
            // Need to integrate forward
            return integrateForward(current, targetEpoch);
        }
        
        // Find nearest snapshot before target
        Optional<SystemState> snapshot = snapshotStore.loadNearest(targetEpoch);
        
        if (snapshot.isEmpty()) {
            throw new IllegalStateException(
                "No snapshot available near epoch " + targetEpoch);
        }
        
        SystemState startState = snapshot.get();
        
        // Integrate from snapshot to target
        if (startState.getEpoch() <= targetEpoch) {
            return integrateForward(startState, targetEpoch);
        } else {
            return integrateBackward(startState, targetEpoch);
        }
    }
    
    private SystemState integrateForward(SystemState start, double targetEpoch) {
        // Create temporary engine for this calculation
        NBodyEngine tempEngine = new NBodyEngine(
            bodies, forceModel, integrator, start
        );
        
        double dt = 0.125;  // days
        while (tempEngine.getState().getEpoch() < targetEpoch) {
            double remaining = targetEpoch - tempEngine.getState().getEpoch();
            tempEngine.step(Math.min(dt, remaining));
        }
        
        return tempEngine.getState();
    }
    
    private SystemState integrateBackward(SystemState start, double targetEpoch) {
        // Integrate with negative timestep
        NBodyEngine tempEngine = new NBodyEngine(
            bodies, forceModel, integrator, start
        );
        
        double dt = -0.125;  // days (negative for backward)
        while (tempEngine.getState().getEpoch() > targetEpoch) {
            double remaining = targetEpoch - tempEngine.getState().getEpoch();
            tempEngine.step(Math.max(dt, remaining));
        }
        
        return tempEngine.getState();
    }
}
```

---

# 12. API Specification

## 12.1 Main Facade

```java
public interface SolarSystemSimulator {
    // Initialization
    void initialize() throws IOException;
    void shutdown();
    
    // Time control
    void setTimeScale(double scale);
    double getTimeScale();
    void jumpToTime(long unixTimestamp);
    long getCurrentTime();  // Unix timestamp
    
    // Queries
    StateVector getPosition(String bodyId);
    StateVector getPosition(String bodyId, long unixTimestamp);
    StateVector getPosition(String bodyId, long unixTimestamp, ReferenceFrame frame);
    
    OrbitalElements getOrbitalElements(String bodyId);
    OrbitalElements getOrbitalElements(String bodyId, String centralBodyId);
    
    AngularPosition getAngularPosition(String bodyId, ObserverLocation observer);
    
    // Events
    List<Conjunction> findConjunctions(String body1, String body2, 
                                       long startTime, long endTime);
    List<Opposition> findOppositions(String bodyId, 
                                     long startTime, long endTime);
    
    // Validation
    ValidationReport getValidationReport();
    ConservationMetrics getConservationMetrics();
    
    // Sync
    SyncResult performSync();
    void setAutoSync(boolean enabled);
    
    // Bodies
    Set<String> getAvailableBodies();
    Body getBodyInfo(String bodyId);
    void addBody(Body body, StateVector initialState);  // Extensibility
}
```

## 12.2 Data Transfer Objects

```java
// Position/velocity query result
public record StateVector(
    double x,    // km
    double y,    // km
    double z,    // km
    double vx,   // km/s
    double vy,   // km/s
    double vz    // km/s
) {
    public double distanceFromOrigin() {
        return Math.sqrt(x*x + y*y + z*z);
    }
    
    public double speed() {
        return Math.sqrt(vx*vx + vy*vy + vz*vz);
    }
}

// Orbital elements
public record OrbitalElements(
    double semiMajorAxis,        // km
    double eccentricity,         // dimensionless
    double inclination,          // radians
    double longitudeOfAscendingNode,  // radians
    double argumentOfPerihelion, // radians
    double meanAnomaly,          // radians
    double trueAnomaly,          // radians
    double period                // days
) {}

// Angular position (as seen from Earth or observer)
public record AngularPosition(
    double rightAscension,   // radians (0 to 2π)
    double declination,      // radians (-π/2 to π/2)
    double distance,         // km
    double eclipticLongitude,  // radians
    double eclipticLatitude,   // radians
    double elongation,       // radians (angle from Sun)
    double phaseAngle        // radians
) {}

// Observer location
public record ObserverLocation(
    double longitude,   // degrees (-180 to 180)
    double latitude,    // degrees (-90 to 90)
    double altitude     // meters above WGS84 ellipsoid
) {}

// Conjunction event
public record Conjunction(
    String body1,
    String body2,
    long timestamp,             // Unix
    double separation,          // radians
    double body1RightAscension,
    double body1Declination,
    double body2RightAscension,
    double body2Declination
) {}

// Opposition event
public record Opposition(
    String bodyId,
    long timestamp,
    double distance,            // km from Earth
    double angularDiameter,     // arcseconds
    double magnitude            // apparent magnitude (optional)
) {}

// Validation report
public record ValidationReport(
    long timestamp,
    double simulationEpoch,
    Map<String, BodyValidation> bodyValidations,
    double overallRmsError
) {}

public record BodyValidation(
    String bodyId,
    double positionErrorKm,
    double velocityErrorKmS,
    double angularErrorArcsec
) {}

// Conservation metrics
public record ConservationMetrics(
    double totalEnergy,
    double energyDriftRelative,
    Vector3D totalMomentum,
    Vector3D totalAngularMomentum,
    double momentumDriftRelative,
    double angularMomentumDriftRelative
) {}
```

## 12.3 Factory and Builder

```java
public class SolarSystemSimulatorBuilder {
    private Set<String> bodies = Set.of("sun", "earth");
    private ForceModel forceModel = ForceModel.RELATIVISTIC_J2;
    private IntegratorType integrator = IntegratorType.YOSHIDA_8;
    private double timestepDays = 0.125;
    private String databasePath = "./solarsim.h2";
    private Duration syncInterval = Duration.ofHours(24);
    private double divergenceThresholdKm = 1000.0;
    private boolean autoSync = true;
    
    public SolarSystemSimulatorBuilder withBodies(String... bodies) {
        this.bodies = Set.of(bodies);
        return this;
    }
    
    public SolarSystemSimulatorBuilder withForceModel(ForceModel model) {
        this.forceModel = model;
        return this;
    }
    
    public SolarSystemSimulatorBuilder withIntegrator(IntegratorType type) {
        this.integrator = type;
        return this;
    }
    
    public SolarSystemSimulatorBuilder withTimestep(double days) {
        this.timestepDays = days;
        return this;
    }
    
    public SolarSystemSimulatorBuilder withDatabase(String path) {
        this.databasePath = path;
        return this;
    }
    
    public SolarSystemSimulatorBuilder withSyncInterval(Duration interval) {
        this.syncInterval = interval;
        return this;
    }
    
    public SolarSystemSimulatorBuilder withDivergenceThreshold(double km) {
        this.divergenceThresholdKm = km;
        return this;
    }
    
    public SolarSystemSimulatorBuilder withAutoSync(boolean enabled) {
        this.autoSync = enabled;
        return this;
    }
    
    public SolarSystemSimulator build() {
        return new SolarSystemSimulatorImpl(this);
    }
}

// Usage example
SolarSystemSimulator sim = new SolarSystemSimulatorBuilder()
    .withBodies("sun", "earth", "moon", "mars", "jupiter")
    .withForceModel(ForceModel.RELATIVISTIC_J2)
    .withTimestep(0.125)
    .withSyncInterval(Duration.ofHours(6))
    .build();

sim.initialize();
```

---

# 13. Validation & Monitoring

## 13.1 Conservation Monitor

```java
public class ConservationMonitor {
    private double initialEnergy;
    private Vector3D initialMomentum;
    private Vector3D initialAngularMomentum;
    private boolean initialized = false;
    
    private final Map<String, Body> bodies;
    
    public ConservationMonitor(Map<String, Body> bodies) {
        this.bodies = bodies;
    }
    
    /**
     * Calculate total mechanical energy.
     */
    public double computeTotalEnergy(SystemState state) {
        double kinetic = 0.0;
        double potential = 0.0;
        
        // Kinetic energy: sum of (1/2)mv²
        for (String id : state.getBodyIds()) {
            Body body = bodies.get(id);
            StateVector sv = state.getState(id);
            double v2 = sv.getVelocity().magnitudeSquared();
            // Use GM/G as proxy for mass (we don't need actual mass)
            kinetic += 0.5 * body.getGM() * v2;
        }
        
        // Potential energy: sum of -GMiMj/rij
        List<String> bodyIds = new ArrayList<>(state.getBodyIds());
        for (int i = 0; i < bodyIds.size(); i++) {
            for (int j = i + 1; j < bodyIds.size(); j++) {
                Body bi = bodies.get(bodyIds.get(i));
                Body bj = bodies.get(bodyIds.get(j));
                StateVector si = state.getState(bodyIds.get(i));
                StateVector sj = state.getState(bodyIds.get(j));
                
                double r = si.getPosition().subtract(sj.getPosition()).magnitude();
                potential -= bi.getGM() * bj.getGM() / r;
            }
        }
        
        return kinetic + potential;
    }
    
    /**
     * Calculate total linear momentum.
     */
    public Vector3D computeTotalMomentum(SystemState state) {
        Vector3D total = Vector3D.zero();
        
        for (String id : state.getBodyIds()) {
            Body body = bodies.get(id);
            StateVector sv = state.getState(id);
            // p = mv ∝ GM * v
            total = total.add(sv.getVelocity().scale(body.getGM()));
        }
        
        return total;
    }
    
    /**
     * Calculate total angular momentum.
     */
    public Vector3D computeTotalAngularMomentum(SystemState state) {
        Vector3D total = Vector3D.zero();
        
        for (String id : state.getBodyIds()) {
            Body body = bodies.get(id);
            StateVector sv = state.getState(id);
            // L = r × p = r × (mv) ∝ r × (GM * v)
            Vector3D L = sv.getPosition().cross(sv.getVelocity()).scale(body.getGM());
            total = total.add(L);
        }
        
        return total;
    }
    
    /**
     * Initialize baseline values.
     */
    public void initialize(SystemState state) {
        initialEnergy = computeTotalEnergy(state);
        initialMomentum = computeTotalMomentum(state);
        initialAngularMomentum = computeTotalAngularMomentum(state);
        initialized = true;
    }
    
    /**
     * Compute current conservation metrics.
     */
    public ConservationMetrics computeMetrics(SystemState state) {
        if (!initialized) {
            throw new IllegalStateException("Monitor not initialized");
        }
        
        double currentEnergy = computeTotalEnergy(state);
        Vector3D currentMomentum = computeTotalMomentum(state);
        Vector3D currentAngularMomentum = computeTotalAngularMomentum(state);
        
        double energyDrift = (currentEnergy - initialEnergy) / Math.abs(initialEnergy);
        
        double momentumDrift = currentMomentum.subtract(initialMomentum).magnitude()
                             / initialMomentum.magnitude();
        
        double angMomDrift = currentAngularMomentum.subtract(initialAngularMomentum).magnitude()
                           / initialAngularMomentum.magnitude();
        
        return new ConservationMetrics(
            currentEnergy,
            energyDrift,
            currentMomentum,
            currentAngularMomentum,
            momentumDrift,
            angMomDrift
        );
    }
}
```

## 13.2 Alert System

```java
public enum AlertLevel {
    DEBUG,
    INFO,
    WARNING,
    ERROR,
    CRITICAL
}

public interface AlertListener {
    void onAlert(AlertLevel level, String message, Instant timestamp);
}

public class AlertSystem {
    private final List<AlertListener> listeners = new CopyOnWriteArrayList<>();
    private final Logger logger = LoggerFactory.getLogger(AlertSystem.class);
    
    // Thresholds
    private double energyDriftThreshold = 1e-10;
    private double momentumDriftThreshold = 1e-10;
    private double positionDivergenceKm = 1000.0;
    
    public void addListener(AlertListener listener) {
        listeners.add(listener);
    }
    
    public void alert(AlertLevel level, String message) {
        Instant now = Instant.now();
        
        // Log
        switch (level) {
            case DEBUG -> logger.debug(message);
            case INFO -> logger.info(message);
            case WARNING -> logger.warn(message);
            case ERROR -> logger.error(message);
            case CRITICAL -> logger.error("CRITICAL: " + message);
        }
        
        // Notify listeners
        for (AlertListener listener : listeners) {
            listener.onAlert(level, message, now);
        }
    }
    
    public void checkConservation(ConservationMetrics metrics) {
        if (Math.abs(metrics.energyDriftRelative()) > energyDriftThreshold) {
            alert(AlertLevel.WARNING, String.format(
                "Energy drift %.2e exceeds threshold %.2e",
                metrics.energyDriftRelative(), energyDriftThreshold));
        }
        
        if (metrics.momentumDriftRelative() > momentumDriftThreshold) {
            alert(AlertLevel.WARNING, String.format(
                "Momentum drift %.2e exceeds threshold %.2e",
                metrics.momentumDriftRelative(), momentumDriftThreshold));
        }
    }
    
    public void setEnergyDriftThreshold(double threshold) {
        this.energyDriftThreshold = threshold;
    }
    
    public void setMomentumDriftThreshold(double threshold) {
        this.momentumDriftThreshold = threshold;
    }
    
    public void setPositionDivergenceThreshold(double km) {
        this.positionDivergenceKm = km;
    }
}
```

## 13.3 Validation Metrics

| Metric | Description | Target | Alert Threshold |
|--------|-------------|--------|-----------------|
| Energy drift | ΔE/E₀ | < 10⁻¹² | > 10⁻¹⁰ |
| Momentum drift | ΔP/P₀ | < 10⁻¹² | > 10⁻¹⁰ |
| Angular momentum drift | ΔL/L₀ | < 10⁻¹² | > 10⁻¹⁰ |
| Position residual (Earth) | vs JPL | < 100 km | > 1000 km |
| Position residual (Mars) | vs JPL | < 500 km | > 5000 km |
| Angular error | apparent position | < 1 arcsec | > 10 arcsec |

---

# 14. Configuration

## 14.1 Configuration File (YAML)

```yaml
# solarsim-config.yaml

simulation:
  # Bodies to simulate
  bodies:
    - sun
    - earth
    # MVP: sun + earth only
    # Full system: uncomment below
    # - moon
    # - mercury
    # - venus
    # - mars
    # - jupiter
    # - saturn
    # - uranus
    # - neptune
    # - pluto
  
  # Physics model
  physics:
    forceModel: RELATIVISTIC_J2  # NEWTONIAN | NEWTONIAN_J2 | RELATIVISTIC | RELATIVISTIC_J2 | FULL
    includeRelativity: true
    includeOblateness: true
    includeSolarPressure: false
  
  # Integration
  integration:
    integrator: YOSHIDA_8  # LEAPFROG | YOSHIDA_4 | YOSHIDA_8 | RK4
    timestepDays: 0.125
    useCompensatedSummation: true

time:
  # Real-time settings
  initialTimeScale: 1.0  # 1.0 = real-time, 86400 = 1 day/second
  
sync:
  # External data synchronization
  enabled: true
  intervalHours: 24
  onFailure: CONTINUE  # CONTINUE | RETRY | HALT
  maxRetries: 3
  retryDelaySeconds: 60
  
  # Divergence handling
  divergenceThresholdKm: 1000.0
  onDivergence: ALERT  # ALERT | CORRECT | ALERT_AND_CORRECT

persistence:
  # Database
  databasePath: ./data/solarsim.h2
  
  # Snapshots
  snapshotIntervalDays: 1.0
  snapshotRetentionDays: 3650  # 10 years
  compressSnapshots: true

validation:
  # Conservation thresholds
  energyDriftThreshold: 1.0e-10
  momentumDriftThreshold: 1.0e-10
  
  # Alert levels
  alertOnDrift: true
  alertOnDivergence: true

logging:
  level: INFO  # DEBUG | INFO | WARN | ERROR
  file: ./logs/solarsim.log
  maxFileSizeMb: 100
  maxFiles: 10
```

## 14.2 Configuration Loader

```java
public class ConfigLoader {
    public SimulatorConfig load(String path) throws IOException {
        ObjectMapper mapper = new ObjectMapper(new YAMLFactory());
        return mapper.readValue(new File(path), SimulatorConfig.class);
    }
    
    public SimulatorConfig loadDefault() {
        return SimulatorConfig.builder()
            .bodies(Set.of("sun", "earth"))
            .forceModel(ForceModel.RELATIVISTIC_J2)
            .integrator(IntegratorType.YOSHIDA_8)
            .timestepDays(0.125)
            .syncIntervalHours(24)
            .divergenceThresholdKm(1000.0)
            .build();
    }
}
```

---

# 15. Build & Deployment

## 15.1 Maven POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.solarsim</groupId>
    <artifactId>solar-system-simulator</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    
    <name>Solar System Simulator</name>
    <description>N-body solar system simulation from first principles</description>
    
    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
    
    <dependencies>
        <!-- HTTP Client (built into Java 11+, but adding for clarity) -->
        <!-- No external dependency needed -->
        
        <!-- JSON Processing -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.16.1</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-yaml</artifactId>
            <version>2.16.1</version>
        </dependency>
        
        <!-- Database -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <version>2.2.224</version>
        </dependency>
        
        <!-- Logging -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>2.0.11</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.4.14</version>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>5.10.1</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>3.25.1</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.12.1</version>
                <configuration>
                    <release>21</release>
                    <compilerArgs>
                        <arg>--enable-preview</arg>
                    </compilerArgs>
                </configuration>
            </plugin>
            
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.3</version>
                <configuration>
                    <argLine>--enable-preview</argLine>
                </configuration>
            </plugin>
            
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.3.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>com.solarsim.Main</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
            
            <!-- Fat JAR for standalone deployment -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.5.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>com.solarsim.Main</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

## 15.2 Directory Structure

```
solar-system-simulator/
├── pom.xml
├── README.md
├── LICENSE
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── solarsim/
│   │   │           ├── Main.java
│   │   │           ├── core/
│   │   │           ├── physics/
│   │   │           ├── integrator/
│   │   │           ├── time/
│   │   │           ├── coordinates/
│   │   │           ├── data/
│   │   │           ├── validation/
│   │   │           ├── api/
│   │   │           └── config/
│   │   └── resources/
│   │       ├── logback.xml
│   │       ├── solarsim-config.yaml
│   │       └── body-catalog.yaml
│   └── test/
│       └── java/
│           └── com/
│               └── solarsim/
│                   ├── physics/
│                   ├── integrator/
│                   └── validation/
├── data/                    # Runtime data (gitignored)
│   └── solarsim.h2
└── logs/                    # Log files (gitignored)
```

## 15.3 Entry Point

```java
package com.solarsim;

import com.solarsim.api.SolarSystemSimulator;
import com.solarsim.api.SolarSystemSimulatorBuilder;
import com.solarsim.config.ConfigLoader;
import com.solarsim.config.SimulatorConfig;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.nio.file.Files;
import java.nio.file.Path;

public class Main {
    private static final Logger log = LoggerFactory.getLogger(Main.class);
    
    public static void main(String[] args) {
        try {
            // Load configuration
            SimulatorConfig config = loadConfig(args);
            
            // Build simulator
            SolarSystemSimulator simulator = new SolarSystemSimulatorBuilder()
                .withConfig(config)
                .build();
            
            // Initialize (fetches live data)
            log.info("Initializing solar system simulator...");
            simulator.initialize();
            log.info("Initialization complete. Simulation epoch: {}", 
                     simulator.getCurrentTime());
            
            // Start real-time simulation
            simulator.setTimeScale(config.getInitialTimeScale());
            
            // Keep running until shutdown
            Runtime.getRuntime().addShutdownHook(new Thread(() -> {
                log.info("Shutting down...");
                simulator.shutdown();
            }));
            
            // Main loop (or just wait if running as service)
            Thread.currentThread().join();
            
        } catch (Exception e) {
            log.error("Fatal error", e);
            System.exit(1);
        }
    }
    
    private static SimulatorConfig loadConfig(String[] args) {
        String configPath = args.length > 0 ? args[0] : "solarsim-config.yaml";
        
        if (Files.exists(Path.of(configPath))) {
            return new ConfigLoader().load(configPath);
        } else {
            log.warn("Config file not found, using defaults");
            return new ConfigLoader().loadDefault();
        }
    }
}
```

---

# 16. Implementation Roadmap

## 16.1 Phase 1: MVP (Sun + Earth)

**Goal**: Minimal working system demonstrating core architecture.

| Task | Priority | Estimate |
|------|----------|----------|
| Vector3D, StateVector classes | P0 | 2h |
| Body, SystemState classes | P0 | 2h |
| NewtonianGravity force model | P0 | 2h |
| Yoshida8 integrator | P0 | 4h |
| JPLHorizonsClient | P0 | 4h |
| TimeEngine (basic) | P0 | 2h |
| NBodyEngine | P0 | 4h |
| Basic API facade | P0 | 2h |
| Unit tests | P0 | 4h |
| Integration test | P0 | 2h |

**Total MVP**: ~28 hours

**Acceptance criteria**:
- Fetch Sun + Earth state from JPL
- Integrate forward 1 year
- Position error < 1000 km vs JPL
- Energy drift < 10⁻⁸

## 16.2 Phase 2: Relativistic Corrections

| Task | Priority | Estimate |
|------|----------|----------|
| PostNewtonian force model | P1 | 4h |
| Oblateness force model | P1 | 3h |
| CompositeForce | P1 | 1h |
| Mercury precession test | P1 | 2h |

**Acceptance criteria**:
- Mercury perihelion precession ≈ 43"/century (relativistic)
- Position error reduced

## 16.3 Phase 3: Full Solar System

| Task | Priority | Estimate |
|------|----------|----------|
| Add all planets | P1 | 2h |
| Add Moon | P1 | 2h |
| Multi-body validation | P1 | 4h |
| Performance optimization | P2 | 4h |

## 16.4 Phase 4: Persistence & Time Travel

| Task | Priority | Estimate |
|------|----------|----------|
| H2 database setup | P1 | 2h |
| SnapshotStore | P1 | 4h |
| TimeTravelService | P1 | 4h |
| Backward integration | P2 | 2h |

## 16.5 Phase 5: Sync & Validation

| Task | Priority | Estimate |
|------|----------|----------|
| SyncManager | P1 | 4h |
| ConservationMonitor | P1 | 3h |
| AlertSystem | P2 | 2h |
| ValidationReport | P2 | 2h |

## 16.6 Phase 6: Advanced Features

| Task | Priority | Estimate |
|------|----------|----------|
| Coordinate transformations | P2 | 4h |
| Event finder (conjunctions) | P2 | 4h |
| Topocentric coordinates | P3 | 4h |
| Runtime body addition | P3 | 4h |

---

# 17. Appendices

## 17.1 Physical Constants

```java
public final class PhysicalConstants {
    private PhysicalConstants() {}
    
    // Fundamental constants
    public static final double G = 6.67430e-20;           // km³/(kg·s²)
    public static final double C = 299792.458;            // km/s
    public static final double AU = 149597870.7;          // km
    
    // Time constants
    public static final double SECONDS_PER_DAY = 86400.0;
    public static final double DAYS_PER_YEAR = 365.25;
    public static final double J2000_JD = 2451545.0;      // Jan 1, 2000, 12:00 TT
    
    // Earth constants
    public static final double EARTH_EQUATORIAL_RADIUS = 6378.137;  // km
    public static final double EARTH_FLATTENING = 1.0 / 298.257223563;
    public static final double OBLIQUITY_J2000 = Math.toRadians(23.439281);
    
    // Solar constants
    public static final double SOLAR_LUMINOSITY = 3.828e26;  // W
    public static final double SOLAR_RADIUS = 696000.0;      // km
}
```

## 17.2 Body Catalog

```yaml
# body-catalog.yaml
bodies:
  sun:
    name: "Sun"
    gm: 1.32712440041279419e11  # km³/s²
    radius: 696000.0            # km
    j2: 2.0e-7
    equatorialRadius: 696000.0
    poleRA: 286.13              # degrees
    poleDec: 63.87              # degrees
    
  mercury:
    name: "Mercury"
    gm: 2.2031868551e4
    radius: 2439.7
    j2: 0.0
    
  venus:
    name: "Venus"
    gm: 3.24858592e5
    radius: 6051.8
    j2: 0.0
    
  earth:
    name: "Earth"
    gm: 3.98600435507e5
    radius: 6371.0
    j2: 1.08263e-3
    equatorialRadius: 6378.137
    poleRA: 0.0
    poleDec: 90.0
    
  moon:
    name: "Moon"
    gm: 4.902800118e3
    radius: 1737.4
    j2: 2.027e-4
    
  mars:
    name: "Mars"
    gm: 4.2828375816e4
    radius: 3389.5
    j2: 1.964e-3
    
  jupiter:
    name: "Jupiter"
    gm: 1.267127641e8
    radius: 69911.0
    j2: 1.4736e-2
    equatorialRadius: 71492.0
    
  saturn:
    name: "Saturn"
    gm: 3.79405852e7
    radius: 58232.0
    j2: 1.6298e-2
    equatorialRadius: 60268.0
    
  uranus:
    name: "Uranus"
    gm: 5.794549007e6
    radius: 25362.0
    j2: 3.343e-3
    
  neptune:
    name: "Neptune"
    gm: 6.8365271005e6
    radius: 24622.0
    j2: 3.411e-3
    
  pluto:
    name: "Pluto"
    gm: 8.699633e2
    radius: 1188.3
    j2: 0.0
```

## 17.3 Test Cases

### 17.3.1 Two-Body Kepler Orbit

```java
@Test
void twoBodyKeplerOrbit_shouldConserveOrbitalElements() {
    // Setup: Earth orbiting Sun only
    Body sun = BodyCatalog.get("sun");
    Body earth = BodyCatalog.get("earth");
    
    // Initial state: circular orbit at 1 AU
    double r = PhysicalConstants.AU;
    double v = Math.sqrt(sun.getGM() / r);
    
    StateVector earthState = new StateVector(r, 0, 0, 0, v, 0);
    StateVector sunState = new StateVector(0, 0, 0, 0, 0, 0);
    
    SystemState initial = new SystemState(
        PhysicalConstants.J2000_JD,
        Map.of("sun", sunState, "earth", earthState)
    );
    
    // Integrate 1 year
    NBodyEngine engine = new NBodyEngine(
        Map.of("sun", sun, "earth", earth),
        new NewtonianGravity(),
        new Yoshida8(),
        initial
    );
    
    engine.advanceTo(PhysicalConstants.J2000_JD + 365.25, 0.125);
    
    // Verify: position should return near start
    StateVector finalEarth = engine.getState().getState("earth");
    assertThat(finalEarth.getPosition().subtract(earthState.getPosition()).magnitude())
        .isLessThan(1000);  // < 1000 km after 1 year
}
```

### 17.3.2 Mercury Precession

```java
@Test
void mercuryOrbit_withRelativity_shouldShowPrecession() {
    // Integrate Mercury with and without relativity
    // Compare argument of perihelion after 100 years
    // Expected: ~43 arcseconds/century from 1PN
}
```

### 17.3.3 Energy Conservation

```java
@Test
void fullSolarSystem_shouldConserveEnergy() {
    // Setup full solar system
    // Integrate 10 years
    // Verify |ΔE/E₀| < 10⁻¹⁰
}
```

## 17.4 Glossary

| Term | Definition |
|------|------------|
| **Barycenter** | Center of mass of a system of bodies |
| **Ephemeris** | Table of positions at specific times |
| **ICRS** | International Celestial Reference System |
| **J2** | Second zonal harmonic (oblateness coefficient) |
| **JD** | Julian Date |
| **N-body** | Gravitational system of N mutually attracting masses |
| **Post-Newtonian** | Approximation to General Relativity |
| **Symplectic** | Structure-preserving (for Hamiltonian systems) |
| **TDB** | Barycentric Dynamical Time |
| **TT** | Terrestrial Time |

## 17.5 References

1. Montenbruck, O., & Gill, E. (2000). *Satellite Orbits: Models, Methods and Applications*. Springer.

2. Urban, S. E., & Seidelmann, P. K. (2012). *Explanatory Supplement to the Astronomical Almanac* (3rd ed.). University Science Books.

3. Yoshida, H. (1990). "Construction of higher order symplectic integrators." *Physics Letters A*, 150(5-7), 262-268.

4. JPL Horizons System: https://ssd.jpl.nasa.gov/horizons/

5. IERS Conventions (2010): https://www.iers.org/IERS/EN/Publications/TechnicalNotes/tn36.html

---

# Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2024-XX-XX | [Author] | Initial specification |

---

*End of Specification*
