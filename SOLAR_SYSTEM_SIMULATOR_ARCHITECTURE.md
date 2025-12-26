# N-Body Solar System Simulator: Complete Architecture Specification

**Version:** 1.0.0  
**Status:** Implementation Ready  
**Last Updated:** December 2024

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Philosophy](#2-system-philosophy)
3. [Physical Model](#3-physical-model)
4. [Mathematical Foundation](#4-mathematical-foundation)
5. [Reference Frames & Coordinate Systems](#5-reference-frames--coordinate-systems)
6. [Time Systems](#6-time-systems)
7. [System Architecture](#7-system-architecture)
8. [Data Sources & Live Integration](#8-data-sources--live-integration)
9. [Synchronization & Calibration](#9-synchronization--calibration)
10. [Persistence Layer](#10-persistence-layer)
11. [Public API](#11-public-api)
12. [Validation & Metrics](#12-validation--metrics)
13. [Error Handling](#13-error-handling)
14. [Configuration System](#14-configuration-system)
15. [Extension Points](#15-extension-points)
16. [MVP Specification](#16-mvp-specification)
17. [Implementation Roadmap](#17-implementation-roadmap)
18. [Appendices](#18-appendices)

---

## 1. Executive Summary

### 1.1 Purpose

This document specifies a **complete, from-first-principles solar system simulator** that:

- Computes celestial body positions using **only fundamental physics**
- Obtains initial conditions from **live astronomical data** at startup
- Requires **no pre-computed ephemeris tables** during runtime
- Maintains accuracy through **periodic synchronization** with authoritative sources
- Is **mathematically consistent** and **theoretically scalable**

### 1.2 Core Principle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           FUNDAMENTAL TRUTH                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   After initialization, the system's ONLY source of truth is:           │
│                                                                          │
│                    F = G·m₁·m₂/r² + corrections                         │
│                                                                          │
│   Everything else (positions, velocities, orbital elements) is          │
│   DERIVED from integrating this equation forward in time.               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Key Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Physics-First** | All motion computed from fundamental laws |
| **Live-Initialized** | Initial state from JPL Horizons API |
| **Self-Correcting** | Periodic sync with optional drift correction |
| **Deterministic** | Identical inputs produce identical outputs |
| **Extensible** | Plugin architecture for forces, bodies, integrators |
| **Observable** | Comprehensive metrics and validation |

---

## 2. System Philosophy

### 2.1 What This System IS

1. A **numerical laboratory** for gravitational dynamics
2. A **pedagogically transparent** implementation of celestial mechanics
3. A **framework** for experimenting with integration methods and force models
4. A **validation testbed** comparing computed vs. observed positions

### 2.2 What This System IS NOT

1. A replacement for JPL Horizons (which uses decades of refined models)
2. A mission-critical navigation system
3. A black-box ephemeris generator

### 2.3 Accuracy Philosophy

The system pursues **maximum achievable accuracy** given:
- The physics models implemented
- The precision of initial conditions
- The numerical methods employed
- Available computational resources

Accuracy degrades gracefully over time due to:
- Truncation error in numerical integration
- Floating-point precision limits
- Unmodeled physical effects
- Initial condition uncertainties

The **sync mechanism** exists to reset accumulated drift when needed.

---

## 3. Physical Model

### 3.1 Fundamental Forces

#### 3.1.1 Newtonian Gravitation (Primary)

The gravitational acceleration on body *i* due to all other bodies:

```
a⃗ᵢ = Σⱼ≠ᵢ [ -G·mⱼ·(r⃗ᵢ - r⃗ⱼ) / |r⃗ᵢ - r⃗ⱼ|³ ]
```

Where:
- `G = 6.67430 × 10⁻¹¹ m³/(kg·s²)` — Gravitational constant
- `mⱼ` — Mass of body j
- `r⃗ᵢ, r⃗ⱼ` — Position vectors in barycentric frame

**Implementation Note:** Use `G·M` products (gravitational parameters μ) directly from JPL for higher precision, as these are measured more accurately than G and M separately.

#### 3.1.2 General Relativistic Corrections (Secondary)

For maximum accuracy, include first-order post-Newtonian corrections.

**Schwarzschild Precession** (dominant relativistic effect):

The additional acceleration due to spacetime curvature near the Sun:

```
a⃗_GR = (GM_☉/c²r²) · [ (4GM_☉/r - v²)·r̂ + 4(r⃗·v⃗)·v⃗/r ]
```

Where:
- `c = 299792458 m/s` — Speed of light
- `r = |r⃗|` — Distance from Sun
- `v⃗` — Velocity vector
- `r̂ = r⃗/r` — Unit radial vector

This produces Mercury's famous 43 arcseconds/century perihelion precession.

**Lense-Thirring Effect** (optional, for extreme precision):

Frame-dragging due to solar rotation. Negligible for most applications but included for completeness:

```
a⃗_LT = (2G/c²r³) · [ 3(J⃗_☉·r̂)·(v⃗×r̂) + v⃗×J⃗_☉ ]
```

Where `J⃗_☉` is the Sun's angular momentum vector.

#### 3.1.3 Oblateness (J₂) Corrections

For bodies with significant equatorial bulge:

```
a⃗_J₂ = -(3/2)·(J₂·μ·R²/r⁴) · [ (1 - 5cos²φ)·r̂ + 2cosφ·φ̂ ]
```

Where:
- `J₂` — Second zonal harmonic coefficient
- `R` — Body's equatorial radius
- `φ` — Latitude above equatorial plane
- `μ = G·M` — Gravitational parameter

**J₂ Values for Key Bodies:**

| Body | J₂ | Equatorial Radius (km) |
|------|----|-----------------------|
| Sun | 2.2 × 10⁻⁷ | 695,700 |
| Earth | 1.08263 × 10⁻³ | 6,378.137 |
| Jupiter | 1.4736 × 10⁻² | 71,492 |
| Saturn | 1.6298 × 10⁻² | 60,268 |

#### 3.1.4 Solar Radiation Pressure (Tertiary)

For small bodies and high-precision spacecraft modeling:

```
a⃗_SRP = -(L_☉·A·C_R) / (4π·c·m·r²) · r̂
```

Where:
- `L_☉ = 3.828 × 10²⁶ W` — Solar luminosity
- `A` — Cross-sectional area
- `C_R` — Radiation pressure coefficient (1.0 for black body, 2.0 for perfect reflector)
- `m` — Body mass

**Note:** Negligible for planets, significant for asteroids < 10 km diameter.

### 3.2 Force Priority and Computation Order

Forces are computed in order of magnitude for numerical stability:

```java
// Pseudocode for force accumulation
Vector3D totalAcceleration = Vector3D.ZERO;

// 1. Primary: N-body Newtonian gravity (always)
totalAcceleration = computeNewtonianGravity(body, allBodies);

// 2. Secondary: Relativistic corrections (if enabled)
if (config.enableRelativity) {
    totalAcceleration = totalAcceleration.add(
        computeSchwarzschildCorrection(body, sun)
    );
}

// 3. Tertiary: Oblateness (if enabled and significant)
if (config.enableOblateness) {
    for (Body oblateBody : oblateBodies) {
        totalAcceleration = totalAcceleration.add(
            computeJ2Correction(body, oblateBody)
        );
    }
}

// 4. Quaternary: Radiation pressure (if enabled and body is small)
if (config.enableSRP && body.isSmallBody()) {
    totalAcceleration = totalAcceleration.add(
        computeSolarRadiationPressure(body, sun)
    );
}
```

### 3.3 Body Catalog

#### 3.3.1 Complete Body List

| Category | Bodies | JPL Horizons ID |
|----------|--------|-----------------|
| **Star** | Sun | 10 |
| **Terrestrial Planets** | Mercury, Venus, Earth, Mars | 199, 299, 399, 499 |
| **Gas Giants** | Jupiter, Saturn | 599, 699 |
| **Ice Giants** | Uranus, Neptune | 799, 899 |
| **Dwarf Planets** | Pluto, Ceres, Eris, Makemake, Haumea | 999, 1, 136199, 136472, 136108 |
| **Major Moons** | Moon, Io, Europa, Ganymede, Callisto, Titan, Triton | 301, 501-504, 606, 801 |

#### 3.3.2 Physical Parameters

These values are **constants** compiled from IAU/JPL authoritative sources:

```java
// Gravitational Parameters (GM) in km³/s² - MORE PRECISE than G×M
public static final double GM_SUN     = 1.32712440041279419e+11;
public static final double GM_MERCURY = 2.2031868551e+04;
public static final double GM_VENUS   = 3.24858592000e+05;
public static final double GM_EARTH   = 3.98600435507e+05;
public static final double GM_MARS    = 4.2828375816e+04;
public static final double GM_JUPITER = 1.26712764100000e+08;
public static final double GM_SATURN  = 3.79405852000000e+07;
public static final double GM_URANUS  = 5.7945563000000e+06;
public static final double GM_NEPTUNE = 6.8365271005800e+06;
public static final double GM_MOON    = 4.9028001184e+03;
public static final double GM_PLUTO   = 8.71e+02;

// For N-body: we need GM values, not masses
// Mass can be derived: M = GM / G, but introduces error
```

### 3.4 Force Model Interface

```java
/**
 * Interface for all force computations.
 * Enables plugin architecture for custom force models.
 */
public interface ForceModel {
    
    /**
     * Compute acceleration on target body due to this force.
     * 
     * @param target The body experiencing the force
     * @param state  Current system state (all body positions/velocities)
     * @param time   Current simulation time (TDB)
     * @return Acceleration vector in km/s²
     */
    Vector3D computeAcceleration(Body target, SystemState state, double time);
    
    /**
     * @return Human-readable name for logging/debugging
     */
    String getName();
    
    /**
     * @return Typical magnitude of this force (for ordering)
     */
    double getTypicalMagnitude();
    
    /**
     * @return Whether this force conserves energy (affects integrator choice)
     */
    boolean isConservative();
}
```

---

## 4. Mathematical Foundation

### 4.1 The N-Body Problem

The system of differential equations we must solve:

```
d²r⃗ᵢ/dt² = Σⱼ≠ᵢ [ -G·mⱼ·(r⃗ᵢ - r⃗ⱼ) / |r⃗ᵢ - r⃗ⱼ|³ ] + a⃗_corrections
```

This is a system of `6N` first-order ODEs (position and velocity for N bodies in 3D).

### 4.2 Numerical Integration Strategy

#### 4.2.1 Why Symplectic Integrators?

Standard integrators (Runge-Kutta, Adams-Bashforth) do not preserve the **Hamiltonian structure** of the N-body problem. Over long integrations:

| Integrator Type | Energy Drift | Phase Drift | Suitable For |
|-----------------|--------------|-------------|--------------|
| RK4 | O(h⁴) secular | Unbounded | Short-term, non-orbital |
| Symplectic | Bounded oscillation | O(h^n) | Long-term orbital |
| Adaptive RK | Variable | Unbounded | Close encounters |

**Symplectic integrators preserve:**
- Bounded energy error (no secular drift)
- Phase-space volume (Liouville's theorem)
- Qualitative long-term behavior

**Trade-off:** Fixed timestep required for symplecticity.

#### 4.2.2 Primary Integrator: Yoshida 4th Order

The Yoshida 4th-order symplectic integrator:

```
Position update:  r⃗(t + c₁h) = r⃗(t) + c₁h·v⃗(t)
Velocity update:  v⃗(t + c₁h) = v⃗(t) + d₁h·a⃗(r⃗(t + c₁h))
Position update:  r⃗(t + (c₁+c₂)h) = r⃗(t + c₁h) + c₂h·v⃗(t + c₁h)
... (repeat pattern)
```

**Yoshida Coefficients:**

```java
// Yoshida 4th order coefficients
private static final double W0 = -Math.cbrt(2.0) / (2.0 - Math.cbrt(2.0));
private static final double W1 = 1.0 / (2.0 - Math.cbrt(2.0));

private static final double[] C = {
    W1 / 2.0,
    (W0 + W1) / 2.0,
    (W0 + W1) / 2.0,
    W1 / 2.0
};

private static final double[] D = {
    W1,
    W0,
    W1,
    0.0  // Last coefficient is zero
};
```

**Implementation:**

```java
public void step(SystemState state, double dt) {
    // Yoshida 4th order symplectic integration
    for (int i = 0; i < 4; i++) {
        // Drift: update positions
        if (C[i] != 0) {
            for (Body body : state.getBodies()) {
                body.position = body.position.add(body.velocity.scale(C[i] * dt));
            }
        }
        
        // Kick: update velocities
        if (D[i] != 0) {
            Vector3D[] accelerations = computeAllAccelerations(state);
            for (int j = 0; j < state.getBodyCount(); j++) {
                state.getBody(j).velocity = state.getBody(j).velocity
                    .add(accelerations[j].scale(D[i] * dt));
            }
        }
    }
}
```

#### 4.2.3 Secondary Integrator: Adaptive Störmer-Verlet with Close Encounter Detection

For handling close encounters (e.g., asteroid-planet), switch to adaptive integration:

```java
public class AdaptiveIntegrator implements Integrator {
    
    private final double tolerancePosition;  // km
    private final double toleranceVelocity;  // km/s
    private final double minTimestep;        // seconds
    private final double maxTimestep;        // seconds
    
    public IntegrationResult stepAdaptive(SystemState state, double dt) {
        // Two half-steps
        SystemState halfStep1 = state.copy();
        verletStep(halfStep1, dt / 2);
        SystemState halfStep2 = halfStep1.copy();
        verletStep(halfStep2, dt / 2);
        
        // One full step
        SystemState fullStep = state.copy();
        verletStep(fullStep, dt);
        
        // Error estimate (Richardson extrapolation)
        double error = computeMaxError(halfStep2, fullStep);
        
        // Adjust timestep
        double newDt = dt * Math.pow(tolerance / error, 0.2);
        newDt = Math.max(minTimestep, Math.min(maxTimestep, newDt));
        
        return new IntegrationResult(halfStep2, newDt, error);
    }
}
```

### 4.3 Error Analysis

#### 4.3.1 Truncation Error

For Yoshida 4th order, the local truncation error is O(h⁵), global error O(h⁴).

**Timestep Selection:**

For desired position error `ε` over time span `T`:

```
h ≈ (ε / T)^(1/4) × characteristic_timescale
```

**Recommended Timesteps:**

| Integration Scope | Timestep | Expected Error/Year |
|-------------------|----------|---------------------|
| Inner planets only | 0.5 days | < 100 km |
| All planets | 1 day | < 1000 km |
| Including moons | 0.1 days | < 100 km |
| High precision | 0.125 days (3 hours) | < 10 km |

#### 4.3.2 Floating-Point Error

Using IEEE 754 double precision (64-bit):

- **Mantissa:** 52 bits → ~15-16 significant decimal digits
- **Relative precision:** ε_machine ≈ 2.2 × 10⁻¹⁶

**Practical Limits:**

For positions in km with magnitude ~10⁹ (outer solar system):
```
Minimum distinguishable distance ≈ 10⁹ × 2.2 × 10⁻¹⁶ ≈ 0.2 mm
```

This is far below other error sources—floating-point is not the limiting factor.

**Numerical Hygiene:**

```java
// GOOD: Compute relative positions directly
Vector3D relativePos = body1.position.subtract(body2.position);

// BAD: Compute absolute then subtract (precision loss)
double dist = body1.position.magnitude() - body2.position.magnitude();
```

#### 4.3.3 Long-Term Drift Analysis

**Energy Conservation Test:**

For a well-tuned symplectic integrator, energy should oscillate but not drift:

```
ΔE/E₀ = O(h^n) oscillation, NOT secular growth
```

**Monitoring:**

```java
public class EnergyMonitor {
    private final double initialEnergy;
    private double maxDeviation = 0;
    
    public void check(SystemState state) {
        double currentEnergy = computeTotalEnergy(state);
        double relativeError = Math.abs(currentEnergy - initialEnergy) / Math.abs(initialEnergy);
        maxDeviation = Math.max(maxDeviation, relativeError);
        
        if (relativeError > ENERGY_ALARM_THRESHOLD) {
            logger.warn("Energy drift detected: {}", relativeError);
        }
    }
}
```

### 4.4 Compensated Summation

For summing many small accelerations without precision loss:

```java
/**
 * Kahan summation algorithm for high-precision accumulation
 */
public class CompensatedSum {
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

// Usage in acceleration computation
public Vector3D computeTotalAcceleration(Body target, List<Body> sources) {
    CompensatedSum ax = new CompensatedSum();
    CompensatedSum ay = new CompensatedSum();
    CompensatedSum az = new CompensatedSum();
    
    for (Body source : sources) {
        if (source == target) continue;
        Vector3D accel = computePairwiseAcceleration(target, source);
        ax.add(accel.x);
        ay.add(accel.y);
        az.add(accel.z);
    }
    
    return new Vector3D(ax.getSum(), ay.getSum(), az.getSum());
}
```

---

## 5. Reference Frames & Coordinate Systems

### 5.1 Primary Frame: International Celestial Reference Frame (ICRF)

**Definition:**
- Origin: Solar System Barycenter (SSB)
- X-axis: Toward vernal equinox at J2000.0 epoch
- Z-axis: Toward north celestial pole at J2000.0
- Y-axis: Completes right-handed system

**Why Barycentric?**
- Inertial (non-accelerating) to high precision
- Required for relativistic corrections
- Standard for interplanetary navigation

### 5.2 Secondary Frame: Heliocentric Ecliptic

**Definition:**
- Origin: Sun center
- X-axis: Toward vernal equinox at J2000.0
- Z-axis: Perpendicular to ecliptic plane (toward ecliptic north)
- Y-axis: Completes right-handed system

**Conversion to/from ICRF:**

The ecliptic is inclined 23.439291° to the equator at J2000.0.

```java
public class FrameTransformations {
    
    // Obliquity of ecliptic at J2000.0 (radians)
    private static final double EPSILON_J2000 = Math.toRadians(23.439291111);
    
    // Rotation matrix: Ecliptic → Equatorial (ICRF)
    private static final Matrix3D ECL_TO_EQ = Matrix3D.rotationX(EPSILON_J2000);
    
    // Rotation matrix: Equatorial (ICRF) → Ecliptic
    private static final Matrix3D EQ_TO_ECL = Matrix3D.rotationX(-EPSILON_J2000);
    
    /**
     * Convert position from Ecliptic to Equatorial (ICRF) coordinates.
     */
    public static Vector3D eclipticToEquatorial(Vector3D ecliptic) {
        return ECL_TO_EQ.multiply(ecliptic);
    }
    
    /**
     * Convert position from Equatorial (ICRF) to Ecliptic coordinates.
     */
    public static Vector3D equatorialToEcliptic(Vector3D equatorial) {
        return EQ_TO_ECL.multiply(equatorial);
    }
    
    /**
     * Convert from Heliocentric to Barycentric.
     * Requires Sun's barycentric position.
     */
    public static Vector3D heliocentricToBarycentric(Vector3D helio, Vector3D sunBary) {
        return helio.add(sunBary);
    }
    
    /**
     * Convert from Barycentric to Heliocentric.
     */
    public static Vector3D barycentricToHeliocentric(Vector3D bary, Vector3D sunBary) {
        return bary.subtract(sunBary);
    }
}
```

### 5.3 Observer Frames (for Output)

#### 5.3.1 Geocentric Equatorial (for Earth-based observers)

```java
/**
 * Convert barycentric position to geocentric apparent coordinates.
 * 
 * @param targetBary Target body barycentric position
 * @param earthBary  Earth barycentric position
 * @param time       TDB time for aberration correction
 * @return Right Ascension (hours), Declination (degrees), Distance (km)
 */
public static SphericalCoordinates toGeocentricApparent(
        Vector3D targetBary, Vector3D earthBary, double time) {
    
    // Geometric geocentric position
    Vector3D geocentric = targetBary.subtract(earthBary);
    
    // Light-time correction (iterative)
    double lightTime = geocentric.magnitude() / SPEED_OF_LIGHT_KM_S;
    // Would need target position at (time - lightTime)
    // Simplified: use geometric position for now
    
    // Convert to spherical
    double distance = geocentric.magnitude();
    double dec = Math.asin(geocentric.z / distance);
    double ra = Math.atan2(geocentric.y, geocentric.x);
    
    // Normalize RA to [0, 2π)
    if (ra < 0) ra += 2 * Math.PI;
    
    return new SphericalCoordinates(
        Math.toDegrees(ra) / 15.0,  // RA in hours
        Math.toDegrees(dec),         // Dec in degrees
        distance                     // Distance in km
    );
}
```

### 5.4 Coordinate System Summary

| Frame | Origin | X-Axis | Z-Axis | Use Case |
|-------|--------|--------|--------|----------|
| ICRF Barycentric | SSB | Vernal Equinox | North Celestial Pole | Internal computation |
| Heliocentric Ecliptic | Sun | Vernal Equinox | Ecliptic North | Orbital analysis |
| Geocentric Equatorial | Earth | Vernal Equinox | North Celestial Pole | Observer coordinates |

---

## 6. Time Systems

### 6.1 Internal Time: Barycentric Dynamical Time (TDB)

**Definition:** The time coordinate in the barycentric frame, used in the equations of motion.

**Properties:**
- Independent of observer location
- Relativistically consistent with barycentric coordinates
- Differs from Terrestrial Time (TT) by periodic terms (~1.7 ms amplitude)

**Representation:** Seconds since J2000.0 epoch (2000-01-01T12:00:00 TDB)

```java
public class TDBTime {
    // Seconds since J2000.0 TDB
    private final double secondsSinceJ2000;
    
    // J2000.0 in Unix seconds (approximately)
    public static final long J2000_UNIX = 946728000L;
    
    public static TDBTime fromJulianDate(double jd) {
        double daysSinceJ2000 = jd - 2451545.0;
        return new TDBTime(daysSinceJ2000 * 86400.0);
    }
    
    public double toJulianDate() {
        return 2451545.0 + secondsSinceJ2000 / 86400.0;
    }
}
```

### 6.2 External Interface: Unix Timestamp

**User-facing times** are Unix timestamps (seconds since 1970-01-01 00:00:00 UTC).

**Conversion Chain:**

```
Unix Timestamp → UTC → TAI → TT → TDB
```

```java
public class TimeConverter {
    
    // TAI - UTC offset (leap seconds as of 2024)
    private static final int LEAP_SECONDS = 37;
    
    // TT - TAI offset (constant)
    private static final double TT_TAI_OFFSET = 32.184;
    
    /**
     * Convert Unix timestamp to TDB seconds since J2000.
     */
    public static double unixToTDB(long unixSeconds) {
        // Unix → TAI (add leap seconds)
        double tai = unixSeconds + LEAP_SECONDS;
        
        // TAI → TT (constant offset)
        double tt = tai + TT_TAI_OFFSET;
        
        // TT → TDB (periodic terms, simplified)
        // Full expression has ~200 terms; this is the dominant term
        double ttCenturies = (tt - J2000_UNIX) / (36525.0 * 86400.0);
        double g = Math.toRadians(357.53 + 35999.05 * ttCenturies); // Mean anomaly of Earth
        double tdbMinusTT = 0.001657 * Math.sin(g) + 0.000022 * Math.sin(2 * g);
        double tdb = tt + tdbMinusTT;
        
        // Convert to seconds since J2000.0
        return tdb - J2000_UNIX;
    }
    
    /**
     * Convert TDB seconds since J2000 to Unix timestamp.
     */
    public static long tdbToUnix(double tdbSinceJ2000) {
        // Inverse transformation (iterative for full precision)
        double tdb = tdbSinceJ2000 + J2000_UNIX;
        
        // TDB → TT (approximate inverse)
        double ttCenturies = (tdb - J2000_UNIX) / (36525.0 * 86400.0);
        double g = Math.toRadians(357.53 + 35999.05 * ttCenturies);
        double tdbMinusTT = 0.001657 * Math.sin(g);
        double tt = tdb - tdbMinusTT;
        
        // TT → TAI
        double tai = tt - TT_TAI_OFFSET;
        
        // TAI → Unix
        return Math.round(tai - LEAP_SECONDS);
    }
}
```

### 6.3 Leap Second Handling

**Critical:** Leap seconds must be tracked for UTC conversion accuracy.

```java
public class LeapSecondTable {
    // Leap second insertion dates (Unix timestamps at 23:59:60)
    private static final long[] LEAP_SECOND_DATES = {
        78796800L,   // 1972-06-30
        94694400L,   // 1972-12-31
        // ... full table through 2016-12-31
        1483228800L, // 2016-12-31 (most recent as of knowledge cutoff)
    };
    
    /**
     * Get TAI-UTC offset for a given Unix time.
     */
    public static int getLeapSeconds(long unixTime) {
        int count = 10; // Pre-1972 offset
        for (long date : LEAP_SECOND_DATES) {
            if (unixTime >= date) count++;
        }
        return count;
    }
}
```

### 6.4 Julian Date Utilities

```java
public class JulianDate {
    
    /**
     * Convert calendar date to Julian Date.
     * Valid for dates after 4713 BC.
     */
    public static double fromCalendar(int year, int month, int day, 
                                       int hour, int minute, double second) {
        // Adjust for January/February
        int a = (14 - month) / 12;
        int y = year + 4800 - a;
        int m = month + 12 * a - 3;
        
        // Julian Day Number
        int jdn = day + (153 * m + 2) / 5 + 365 * y + y / 4 - y / 100 + y / 400 - 32045;
        
        // Add time fraction
        double fraction = (hour - 12) / 24.0 + minute / 1440.0 + second / 86400.0;
        
        return jdn + fraction;
    }
    
    /**
     * J2000.0 epoch in Julian Date.
     */
    public static final double J2000 = 2451545.0;
    
    /**
     * Julian centuries since J2000.0.
     */
    public static double centuriesSinceJ2000(double jd) {
        return (jd - J2000) / 36525.0;
    }
}
```

---

## 7. System Architecture

### 7.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            SOLAR SYSTEM SIMULATOR                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                          PUBLIC API LAYER                             │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────────┐ │   │
│  │  │PositionQuery│ │VelocityQuery│ │ElementQuery │ │AngularPosQuery  │ │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────────┘ │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────────┐ │   │
│  │  │ EventSearch │ │  Metrics    │ │ StateExport │ │  Configuration  │ │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         COORDINATION LAYER                            │   │
│  │  ┌─────────────────────┐              ┌─────────────────────┐        │   │
│  │  │    TimeEngine       │◄────────────►│   SimulationEngine  │        │   │
│  │  │  - Wall clock sync  │              │  - Step coordinator │        │   │
│  │  │  - Time scaling     │              │  - State transitions│        │   │
│  │  │  - TDB conversion   │              │  - Thread orchestr. │        │   │
│  │  └─────────────────────┘              └─────────────────────┘        │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│         ┌──────────────────────────┼──────────────────────────┐             │
│         ▼                          ▼                          ▼             │
│  ┌─────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐  │
│  │  PHYSICS ENGINE │    │    STATE MANAGER    │    │   SYNC SERVICE      │  │
│  │  ┌───────────┐  │    │  ┌───────────────┐  │    │  ┌───────────────┐  │  │
│  │  │ForceModels│  │    │  │ SystemState   │  │    │  │JPL Horizons   │  │  │
│  │  │ -Newton   │  │    │  │ - positions   │  │    │  │   Client      │  │  │
│  │  │ -GR       │  │    │  │ - velocities  │  │    │  └───────────────┘  │  │
│  │  │ -J2       │  │    │  │ - time        │  │    │  ┌───────────────┐  │  │
│  │  │ -SRP      │  │    │  └───────────────┘  │    │  │ Divergence    │  │  │
│  │  └───────────┘  │    │  ┌───────────────┐  │    │  │   Detector    │  │  │
│  │  ┌───────────┐  │    │  │ Snapshot      │  │    │  └───────────────┘  │  │
│  │  │Integrators│  │    │  │   Cache       │  │    │  ┌───────────────┐  │  │
│  │  │ -Yoshida  │  │    │  └───────────────┘  │    │  │ Correction    │  │  │
│  │  │ -Adaptive │  │    │                     │    │  │   Strategy    │  │  │
│  │  └───────────┘  │    │                     │    │  └───────────────┘  │  │
│  └─────────────────┘    └─────────────────────┘    └─────────────────────┘  │
│         │                          │                          │             │
│         └──────────────────────────┼──────────────────────────┘             │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                        PERSISTENCE LAYER                              │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐   │   │
│  │  │ SnapshotStore   │  │ ConfigStore     │  │ MetricsStore        │   │   │
│  │  │ (Time-indexed)  │  │ (Versioned)     │  │ (Time-series)       │   │   │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                         INFRASTRUCTURE                                │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │  Database   │  │   HTTP      │  │  Logging    │  │  Metrics    │  │   │
│  │  │  (H2/Postgres)│ │   Client    │  │  (SLF4J)    │  │  Export     │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Module Specifications

#### 7.2.1 Physics Engine

**Responsibility:** Compute accelerations and advance state.

```java
package com.solarsim.physics;

public class PhysicsEngine {
    private final List<ForceModel> forceModels;
    private final Integrator integrator;
    private final PhysicsConfig config;
    
    /**
     * Compute total acceleration for all bodies.
     * Thread-safe: can parallelize over bodies.
     */
    public Vector3D[] computeAccelerations(SystemState state) {
        int n = state.getBodyCount();
        Vector3D[] accelerations = new Vector3D[n];
        
        // Parallel computation over bodies
        IntStream.range(0, n).parallel().forEach(i -> {
            Body target = state.getBody(i);
            Vector3D totalAccel = Vector3D.ZERO;
            
            for (ForceModel model : forceModels) {
                totalAccel = totalAccel.add(
                    model.computeAcceleration(target, state, state.getTime())
                );
            }
            
            accelerations[i] = totalAccel;
        });
        
        return accelerations;
    }
    
    /**
     * Advance state by one timestep.
     */
    public void step(SystemState state, double dt) {
        integrator.step(state, dt, this::computeAccelerations);
    }
}
```

#### 7.2.2 Time Engine

**Responsibility:** Manage simulation time progression and wall-clock synchronization.

```java
package com.solarsim.time;

public class TimeEngine {
    
    public enum Mode {
        REALTIME,      // 1 sim-second = 1 real-second
        SCALED,        // 1 sim-second = scale * real-second
        FREE_RUNNING   // As fast as possible
    }
    
    private volatile Mode mode = Mode.SCALED;
    private volatile double timeScale = 86400.0;  // Default: 1 day per second
    private volatile double simulationTime;       // TDB seconds since J2000
    private volatile long lastWallTime;           // System.nanoTime()
    
    /**
     * Calculate how much simulation time should advance.
     */
    public double getNextTimestep(double maxDt) {
        long now = System.nanoTime();
        double wallElapsed = (now - lastWallTime) / 1_000_000_000.0;
        lastWallTime = now;
        
        switch (mode) {
            case REALTIME:
                return Math.min(wallElapsed, maxDt);
            case SCALED:
                return Math.min(wallElapsed * timeScale, maxDt);
            case FREE_RUNNING:
                return maxDt;
            default:
                throw new IllegalStateException();
        }
    }
    
    /**
     * Set time scale (simulation seconds per real second).
     */
    public void setTimeScale(double scale) {
        if (scale <= 0) throw new IllegalArgumentException("Scale must be positive");
        this.timeScale = scale;
    }
    
    /**
     * Jump to specific simulation time.
     * Requires state manager to load appropriate snapshot.
     */
    public void jumpToTime(double targetTDB) {
        this.simulationTime = targetTDB;
        // StateManager.loadNearestSnapshot() called externally
    }
}
```

#### 7.2.3 State Manager

**Responsibility:** Maintain current state and manage snapshots.

```java
package com.solarsim.state;

public class StateManager {
    
    private SystemState currentState;
    private final SnapshotStore snapshotStore;
    private final ReentrantReadWriteLock stateLock = new ReentrantReadWriteLock();
    
    /**
     * Get current state (read-only view).
     */
    public SystemState getCurrentState() {
        stateLock.readLock().lock();
        try {
            return currentState.immutableCopy();
        } finally {
            stateLock.readLock().unlock();
        }
    }
    
    /**
     * Update state after integration step.
     */
    public void updateState(SystemState newState) {
        stateLock.writeLock().lock();
        try {
            this.currentState = newState;
            
            // Auto-snapshot if interval elapsed
            if (shouldSnapshot()) {
                snapshotStore.save(newState);
            }
        } finally {
            stateLock.writeLock().unlock();
        }
    }
    
    /**
     * Load state from nearest snapshot to target time.
     * Used for time jumps.
     */
    public SystemState loadNearestSnapshot(double targetTime) {
        return snapshotStore.findNearest(targetTime)
            .orElseThrow(() -> new NoSnapshotException(targetTime));
    }
    
    /**
     * Get state at arbitrary time (may require integration).
     */
    public SystemState getStateAtTime(double targetTime) {
        SystemState nearest = snapshotStore.findNearest(targetTime).orElse(currentState);
        
        if (Math.abs(nearest.getTime() - targetTime) < EPSILON) {
            return nearest;
        }
        
        // Need to integrate from nearest snapshot
        return integrateToTime(nearest, targetTime);
    }
}
```

#### 7.2.4 Sync Service

**Responsibility:** Fetch live data and detect/correct drift.

```java
package com.solarsim.sync;

public class SyncService {
    
    private final JPLHorizonsClient horizonsClient;
    private final DivergenceDetector divergenceDetector;
    private final CorrectionStrategy correctionStrategy;
    private final SyncConfig config;
    
    /**
     * Perform synchronization check and optional correction.
     */
    public SyncResult synchronize(SystemState currentState) {
        // Fetch authoritative state from JPL
        Map<Body, StateVector> authoritativeState;
        try {
            authoritativeState = horizonsClient.fetchCurrentState(
                currentState.getBodies(),
                currentState.getTime()
            );
        } catch (IOException e) {
            return SyncResult.networkFailure(e);
        }
        
        // Compute divergence
        DivergenceReport divergence = divergenceDetector.analyze(
            currentState, authoritativeState
        );
        
        // Decide on action
        if (divergence.getMaxPositionError() < config.getAcceptableErrorKm()) {
            return SyncResult.withinTolerance(divergence);
        }
        
        if (divergence.getMaxPositionError() > config.getCriticalErrorKm()) {
            // Major divergence - need human decision or auto-reset
            if (config.isAutoResetEnabled()) {
                SystemState corrected = correctionStrategy.fullReset(authoritativeState);
                return SyncResult.resetPerformed(divergence, corrected);
            } else {
                return SyncResult.criticalDivergence(divergence);
            }
        }
        
        // Moderate divergence - apply weighted correction
        SystemState corrected = correctionStrategy.blend(
            currentState, authoritativeState, config.getCorrectionWeight()
        );
        return SyncResult.correctionApplied(divergence, corrected);
    }
}
```

### 7.3 Threading Model

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          THREAD ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                     MAIN SIMULATION THREAD                       │    │
│  │                                                                  │    │
│  │   while (running) {                                              │    │
│  │       dt = timeEngine.getNextTimestep();                         │    │
│  │       physicsEngine.step(state, dt);  // May use ForkJoinPool   │    │
│  │       stateManager.updateState(state);                           │    │
│  │       metricsCollector.record();                                 │    │
│  │   }                                                              │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────┐  ┌─────────────────────┐                       │
│  │  SYNC THREAD        │  │  API THREAD POOL    │                       │
│  │  (ScheduledExecutor)│  │  (Cached/Fixed)     │                       │
│  │                     │  │                     │                       │
│  │  - Periodic sync    │  │  - Query handling   │                       │
│  │  - Drift detection  │  │  - State access     │                       │
│  │  - Correction       │  │  - Metrics export   │                       │
│  └─────────────────────┘  └─────────────────────┘                       │
│                                                                          │
│  ┌─────────────────────┐  ┌─────────────────────┐                       │
│  │  SNAPSHOT THREAD    │  │  METRICS THREAD     │                       │
│  │  (Background)       │  │  (Scheduled)        │                       │
│  │                     │  │                     │                       │
│  │  - DB writes        │  │  - Aggregation      │                       │
│  │  - Compression      │  │  - Export           │                       │
│  └─────────────────────┘  └─────────────────────┘                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Synchronization Points:**

| Resource | Protection | Access Pattern |
|----------|------------|----------------|
| SystemState | ReadWriteLock | Many readers, one writer |
| SnapshotStore | Synchronized methods | Sequential writes |
| Configuration | Volatile + immutable | Read-mostly |
| Metrics | ConcurrentHashMap | Concurrent updates |

### 7.4 Package Structure

```
com.solarsim/
├── api/                          # Public API interfaces
│   ├── SolarSystemSimulator.java # Main facade
│   ├── BodyQuery.java            # Query interface
│   ├── SimulatorConfig.java      # Configuration builder
│   └── events/                   # Event search API
│       ├── ConjunctionFinder.java
│       └── EventQuery.java
├── physics/                      # Physics engine
│   ├── PhysicsEngine.java
│   ├── forces/                   # Force models
│   │   ├── ForceModel.java
│   │   ├── NewtonianGravity.java
│   │   ├── RelativisticCorrection.java
│   │   ├── OblatenessJ2.java
│   │   └── SolarRadiationPressure.java
│   └── integrators/              # Numerical integrators
│       ├── Integrator.java
│       ├── Yoshida4.java
│       ├── StormerVerlet.java
│       └── AdaptiveIntegrator.java
├── state/                        # State management
│   ├── SystemState.java
│   ├── Body.java
│   ├── StateVector.java
│   ├── StateManager.java
│   └── OrbitalElements.java
├── time/                         # Time handling
│   ├── TimeEngine.java
│   ├── TDBTime.java
│   ├── TimeConverter.java
│   ├── JulianDate.java
│   └── LeapSecondTable.java
├── coordinates/                  # Reference frames
│   ├── Vector3D.java
│   ├── Matrix3D.java
│   ├── ReferenceFrame.java
│   ├── FrameTransformations.java
│   └── SphericalCoordinates.java
├── sync/                         # Synchronization
│   ├── SyncService.java
│   ├── JPLHorizonsClient.java
│   ├── DivergenceDetector.java
│   └── CorrectionStrategy.java
├── persistence/                  # Data persistence
│   ├── SnapshotStore.java
│   ├── DatabaseConfig.java
│   └── schema/                   # Database schema
│       └── V1__initial_schema.sql
├── metrics/                      # Validation & metrics
│   ├── MetricsCollector.java
│   ├── EnergyMonitor.java
│   ├── MomentumMonitor.java
│   └── ValidationReport.java
├── config/                       # Configuration
│   ├── SimulatorConfiguration.java
│   ├── PhysicsConfiguration.java
│   └── SyncConfiguration.java
└── util/                         # Utilities
    ├── Constants.java
    ├── CompensatedSum.java
    └── MathUtils.java
```

---

## 8. Data Sources & Live Integration

### 8.1 JPL Horizons API

**Base URL:** `https://ssd.jpl.nasa.gov/api/horizons.api`

**Purpose:** Authoritative source for initial conditions and sync validation.

#### 8.1.1 State Vector Request

```java
public class JPLHorizonsClient {
    
    private static final String BASE_URL = "https://ssd.jpl.nasa.gov/api/horizons.api";
    private final HttpClient httpClient;
    private final ObjectMapper jsonMapper;
    
    /**
     * Fetch state vectors for multiple bodies at a specific time.
     */
    public Map<Integer, StateVector> fetchStateVectors(
            List<Integer> bodyIds, 
            double tdbTime) throws IOException {
        
        Map<Integer, StateVector> results = new HashMap<>();
        
        for (int bodyId : bodyIds) {
            String query = buildStateVectorQuery(bodyId, tdbTime);
            String response = executeQuery(query);
            StateVector sv = parseStateVector(response);
            results.put(bodyId, sv);
        }
        
        return results;
    }
    
    private String buildStateVectorQuery(int bodyId, double tdbTime) {
        // Convert TDB to calendar date for query
        double jd = tdbTime / 86400.0 + 2451545.0;
        String dateStr = julianToCalendar(jd);
        
        return String.format(
            "format=json" +
            "&COMMAND='%d'" +              // Body ID
            "&CENTER='500@0'" +            // Solar System Barycenter
            "&MAKE_EPHEM='YES'" +
            "&EPHEM_TYPE='VECTORS'" +
            "&VEC_TABLE='2'" +             // State vectors (pos + vel)
            "&REF_PLANE='FRAME'" +         // ICRF reference frame
            "&REF_SYSTEM='ICRF'" +
            "&OUT_UNITS='KM-S'" +          // km and km/s
            "&VEC_LABELS='YES'" +
            "&START_TIME='%s'" +
            "&STOP_TIME='%s'" +
            "&STEP_SIZE='1'",              // Single point
            bodyId, dateStr, dateStr
        );
    }
    
    private StateVector parseStateVector(String jsonResponse) {
        // Parse Horizons JSON response
        JsonNode root = jsonMapper.readTree(jsonResponse);
        String data = root.get("result").asText();
        
        // Extract state vector from formatted output
        // Format: X, Y, Z, VX, VY, VZ with scientific notation
        Pattern pattern = Pattern.compile(
            "X\\s*=\\s*([\\d.E+-]+).*" +
            "Y\\s*=\\s*([\\d.E+-]+).*" +
            "Z\\s*=\\s*([\\d.E+-]+).*" +
            "VX\\s*=\\s*([\\d.E+-]+).*" +
            "VY\\s*=\\s*([\\d.E+-]+).*" +
            "VZ\\s*=\\s*([\\d.E+-]+)",
            Pattern.DOTALL
        );
        
        Matcher m = pattern.matcher(data);
        if (!m.find()) {
            throw new ParseException("Failed to parse Horizons response");
        }
        
        return new StateVector(
            new Vector3D(
                Double.parseDouble(m.group(1)),
                Double.parseDouble(m.group(2)),
                Double.parseDouble(m.group(3))
            ),
            new Vector3D(
                Double.parseDouble(m.group(4)),
                Double.parseDouble(m.group(5)),
                Double.parseDouble(m.group(6))
            )
        );
    }
}
```

#### 8.1.2 Body ID Reference

| Body | Horizons ID | Notes |
|------|-------------|-------|
| Sun | 10 | |
| Mercury Barycenter | 1 | Use for outer planet perturbations |
| Mercury | 199 | Planet center |
| Venus Barycenter | 2 | |
| Venus | 299 | |
| Earth-Moon Barycenter | 3 | |
| Earth | 399 | |
| Moon | 301 | |
| Mars Barycenter | 4 | |
| Mars | 499 | |
| Jupiter Barycenter | 5 | |
| Jupiter | 599 | |
| Saturn Barycenter | 6 | |
| Saturn | 699 | |
| Uranus Barycenter | 7 | |
| Uranus | 799 | |
| Neptune Barycenter | 8 | |
| Neptune | 899 | |
| Pluto Barycenter | 9 | |
| Pluto | 999 | |

### 8.2 Initialization Sequence

```
┌─────────────────────────────────────────────────────────────────────┐
│                      STARTUP SEQUENCE                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Load Configuration                                               │
│     └─► Read config file / environment / defaults                    │
│                                                                      │
│  2. Initialize Infrastructure                                        │
│     ├─► Database connection                                          │
│     ├─► HTTP client                                                  │
│     └─► Logging                                                      │
│                                                                      │
│  3. Check for Cached State                                           │
│     ├─► If recent snapshot exists (< config.maxSnapshotAge):         │
│     │   └─► Load snapshot, integrate forward to current time         │
│     └─► Otherwise: proceed to live fetch                             │
│                                                                      │
│  4. Fetch Live Initial Conditions                                    │
│     ├─► Query JPL Horizons for all configured bodies                 │
│     ├─► Retry with backoff on failure                                │
│     └─► If persistent failure: use cached state or fail startup      │
│                                                                      │
│  5. Validate Initial State                                           │
│     ├─► Check all required bodies present                            │
│     ├─► Verify positions within expected bounds                      │
│     └─► Compute initial energy/momentum baselines                    │
│                                                                      │
│  6. Initialize Engines                                               │
│     ├─► Physics engine with configured force models                  │
│     ├─► Time engine with configured mode/scale                       │
│     └─► State manager with initial state                             │
│                                                                      │
│  7. Start Background Services                                        │
│     ├─► Sync scheduler                                               │
│     ├─► Snapshot scheduler                                           │
│     └─► Metrics collector                                            │
│                                                                      │
│  8. Begin Simulation Loop                                            │
│     └─► System is now operational                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.3 Fallback Strategy

```java
public class InitializationService {
    
    public SystemState initialize(SimulatorConfig config) {
        // Strategy 1: Live fetch
        try {
            return fetchLiveState(config);
        } catch (IOException e) {
            logger.warn("Live fetch failed, trying cached state", e);
        }
        
        // Strategy 2: Recent snapshot
        Optional<Snapshot> recent = snapshotStore.findMostRecent();
        if (recent.isPresent() && isAcceptablyFresh(recent.get(), config)) {
            logger.info("Using cached snapshot from {}", recent.get().getTime());
            return integrateToNow(recent.get().getState());
        }
        
        // Strategy 3: Fail explicitly (don't use stale data silently)
        throw new InitializationException(
            "Cannot initialize: network unavailable and no recent snapshot. " +
            "Please ensure network connectivity or provide a seed state file."
        );
    }
    
    private boolean isAcceptablyFresh(Snapshot snapshot, SimulatorConfig config) {
        double ageSeconds = currentTDBTime() - snapshot.getTime();
        return ageSeconds < config.getMaxSnapshotAgeSeconds();
    }
}
```

---

## 9. Synchronization & Calibration

### 9.1 Sync Schedule

```java
public class SyncScheduler {
    
    private final ScheduledExecutorService executor;
    private final SyncService syncService;
    private final SyncConfig config;
    
    public void start() {
        executor.scheduleAtFixedRate(
            this::performSync,
            config.getInitialDelaySeconds(),
            config.getIntervalSeconds(),
            TimeUnit.SECONDS
        );
    }
    
    private void performSync() {
        try {
            SyncResult result = syncService.synchronize(
                stateManager.getCurrentState()
            );
            
            handleResult(result);
            
        } catch (Exception e) {
            logger.error("Sync failed", e);
            metrics.recordSyncFailure();
        }
    }
    
    private void handleResult(SyncResult result) {
        switch (result.getStatus()) {
            case WITHIN_TOLERANCE:
                logger.debug("Sync: within tolerance, max error = {} km", 
                    result.getDivergence().getMaxPositionError());
                metrics.recordSyncSuccess(result.getDivergence());
                break;
                
            case CORRECTION_APPLIED:
                logger.info("Sync: correction applied, error reduced from {} to {} km",
                    result.getDivergence().getMaxPositionError(),
                    result.getCorrectedDivergence().getMaxPositionError());
                stateManager.updateState(result.getCorrectedState());
                metrics.recordSyncCorrection(result);
                break;
                
            case RESET_PERFORMED:
                logger.warn("Sync: full reset performed due to critical divergence");
                stateManager.updateState(result.getCorrectedState());
                metrics.recordSyncReset(result);
                alertService.sendAlert(AlertLevel.WARNING, 
                    "Simulation state reset due to divergence > {} km",
                    config.getCriticalErrorKm());
                break;
                
            case CRITICAL_DIVERGENCE:
                logger.error("Sync: critical divergence detected, manual intervention required");
                metrics.recordCriticalDivergence(result);
                alertService.sendAlert(AlertLevel.CRITICAL,
                    "Critical simulation divergence: {} km. Manual review required.",
                    result.getDivergence().getMaxPositionError());
                break;
                
            case NETWORK_FAILURE:
                logger.warn("Sync: network failure, will retry next interval");
                metrics.recordSyncNetworkFailure();
                break;
        }
    }
}
```

### 9.2 Divergence Detection

```java
public class DivergenceDetector {
    
    /**
     * Analyze divergence between computed and authoritative state.
     */
    public DivergenceReport analyze(
            SystemState computed, 
            Map<Body, StateVector> authoritative) {
        
        List<BodyDivergence> divergences = new ArrayList<>();
        
        for (Body body : computed.getBodies()) {
            StateVector authSV = authoritative.get(body);
            if (authSV == null) continue;
            
            StateVector compSV = computed.getStateVector(body);
            
            double positionError = compSV.position.subtract(authSV.position).magnitude();
            double velocityError = compSV.velocity.subtract(authSV.velocity).magnitude();
            
            divergences.add(new BodyDivergence(
                body,
                positionError,
                velocityError,
                positionError / body.getOrbitalRadius(),  // Relative error
                compSV,
                authSV
            ));
        }
        
        return new DivergenceReport(
            computed.getTime(),
            divergences,
            divergences.stream().mapToDouble(BodyDivergence::getPositionError).max().orElse(0),
            divergences.stream().mapToDouble(BodyDivergence::getVelocityError).max().orElse(0)
        );
    }
}
```

### 9.3 Correction Strategies

```java
public interface CorrectionStrategy {
    
    /**
     * Full reset to authoritative state.
     */
    SystemState fullReset(Map<Body, StateVector> authoritative);
    
    /**
     * Weighted blend between computed and authoritative.
     * 
     * @param weight 0.0 = keep computed, 1.0 = use authoritative
     */
    SystemState blend(SystemState computed, 
                      Map<Body, StateVector> authoritative, 
                      double weight);
}

public class DefaultCorrectionStrategy implements CorrectionStrategy {
    
    @Override
    public SystemState fullReset(Map<Body, StateVector> authoritative) {
        SystemState newState = new SystemState();
        for (Map.Entry<Body, StateVector> entry : authoritative.entrySet()) {
            newState.setStateVector(entry.getKey(), entry.getValue());
        }
        return newState;
    }
    
    @Override
    public SystemState blend(SystemState computed, 
                             Map<Body, StateVector> authoritative, 
                             double weight) {
        SystemState blended = computed.copy();
        
        for (Body body : computed.getBodies()) {
            StateVector authSV = authoritative.get(body);
            if (authSV == null) continue;
            
            StateVector compSV = computed.getStateVector(body);
            
            // Linear interpolation
            Vector3D newPos = compSV.position.scale(1 - weight)
                .add(authSV.position.scale(weight));
            Vector3D newVel = compSV.velocity.scale(1 - weight)
                .add(authSV.velocity.scale(weight));
            
            blended.setStateVector(body, new StateVector(newPos, newVel));
        }
        
        return blended;
    }
}
```

### 9.4 Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `sync.intervalSeconds` | 86400 (1 day) | Time between sync attempts |
| `sync.initialDelaySeconds` | 3600 (1 hour) | Delay after startup |
| `sync.acceptableErrorKm` | 1000 | No correction needed below this |
| `sync.warningErrorKm` | 10000 | Log warning above this |
| `sync.criticalErrorKm` | 100000 | Requires intervention above this |
| `sync.correctionWeight` | 0.5 | Blend weight for moderate errors |
| `sync.autoResetEnabled` | false | Auto-reset on critical divergence |
| `sync.retryAttempts` | 3 | Network retry count |
| `sync.retryDelayMs` | 5000 | Delay between retries |

---

## 10. Persistence Layer

### 10.1 Database Schema

```sql
-- V1__initial_schema.sql

-- System state snapshots
CREATE TABLE snapshots (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    tdb_time        DOUBLE NOT NULL,          -- TDB seconds since J2000
    unix_time       BIGINT NOT NULL,          -- Unix timestamp (for queries)
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Metadata
    integrator      VARCHAR(50),
    timestep_days   DOUBLE,
    force_models    VARCHAR(500),             -- JSON array of enabled models
    
    -- Validation metrics at snapshot time
    total_energy    DOUBLE,
    energy_drift    DOUBLE,                   -- Relative to initial
    
    -- Compressed state data
    state_data      BLOB NOT NULL,            -- Compressed JSON or binary
    
    INDEX idx_tdb_time (tdb_time),
    INDEX idx_unix_time (unix_time)
);

-- Individual body states (for efficient single-body queries)
CREATE TABLE body_states (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    snapshot_id     BIGINT NOT NULL,
    body_id         INT NOT NULL,             -- JPL Horizons ID
    body_name       VARCHAR(50),
    
    -- Position (km, ICRF barycentric)
    pos_x           DOUBLE NOT NULL,
    pos_y           DOUBLE NOT NULL,
    pos_z           DOUBLE NOT NULL,
    
    -- Velocity (km/s, ICRF barycentric)
    vel_x           DOUBLE NOT NULL,
    vel_y           DOUBLE NOT NULL,
    vel_z           DOUBLE NOT NULL,
    
    FOREIGN KEY (snapshot_id) REFERENCES snapshots(id) ON DELETE CASCADE,
    INDEX idx_snapshot_body (snapshot_id, body_id),
    INDEX idx_body_time (body_id, snapshot_id)
);

-- Sync history
CREATE TABLE sync_events (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    tdb_time        DOUBLE NOT NULL,
    unix_time       BIGINT NOT NULL,
    
    status          VARCHAR(30) NOT NULL,     -- WITHIN_TOLERANCE, CORRECTION_APPLIED, etc.
    max_pos_error   DOUBLE,                   -- km
    max_vel_error   DOUBLE,                   -- km/s
    
    correction_applied BOOLEAN DEFAULT FALSE,
    details         TEXT,                     -- JSON with per-body details
    
    INDEX idx_sync_time (unix_time)
);

-- Metrics time series
CREATE TABLE metrics (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    tdb_time        DOUBLE NOT NULL,
    unix_time       BIGINT NOT NULL,
    
    metric_name     VARCHAR(100) NOT NULL,
    metric_value    DOUBLE NOT NULL,
    
    INDEX idx_metric_time (metric_name, unix_time)
);

-- Configuration history
CREATE TABLE config_versions (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    version         INT NOT NULL,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    config_json     TEXT NOT NULL,
    
    UNIQUE INDEX idx_version (version)
);
```

### 10.2 Snapshot Store Implementation

```java
public class SnapshotStore {
    
    private final DataSource dataSource;
    private final Duration snapshotInterval;
    private volatile Instant lastSnapshotTime = Instant.EPOCH;
    
    /**
     * Save system state as snapshot.
     */
    public void save(SystemState state) {
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            
            // Insert snapshot record
            long snapshotId;
            try (PreparedStatement ps = conn.prepareStatement(
                    "INSERT INTO snapshots (tdb_time, unix_time, integrator, " +
                    "timestep_days, force_models, total_energy, energy_drift, state_data) " +
                    "VALUES (?, ?, ?, ?, ?, ?, ?, ?)",
                    Statement.RETURN_GENERATED_KEYS)) {
                
                ps.setDouble(1, state.getTime());
                ps.setLong(2, TimeConverter.tdbToUnix(state.getTime()));
                ps.setString(3, state.getIntegratorName());
                ps.setDouble(4, state.getTimestepDays());
                ps.setString(5, toJson(state.getForceModels()));
                ps.setDouble(6, state.getTotalEnergy());
                ps.setDouble(7, state.getEnergyDrift());
                ps.setBytes(8, compressState(state));
                
                ps.executeUpdate();
                
                try (ResultSet rs = ps.getGeneratedKeys()) {
                    rs.next();
                    snapshotId = rs.getLong(1);
                }
            }
            
            // Insert individual body states
            try (PreparedStatement ps = conn.prepareStatement(
                    "INSERT INTO body_states (snapshot_id, body_id, body_name, " +
                    "pos_x, pos_y, pos_z, vel_x, vel_y, vel_z) " +
                    "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)")) {
                
                for (Body body : state.getBodies()) {
                    StateVector sv = state.getStateVector(body);
                    ps.setLong(1, snapshotId);
                    ps.setInt(2, body.getHorizonsId());
                    ps.setString(3, body.getName());
                    ps.setDouble(4, sv.position.x);
                    ps.setDouble(5, sv.position.y);
                    ps.setDouble(6, sv.position.z);
                    ps.setDouble(7, sv.velocity.x);
                    ps.setDouble(8, sv.velocity.y);
                    ps.setDouble(9, sv.velocity.z);
                    ps.addBatch();
                }
                
                ps.executeBatch();
            }
            
            conn.commit();
            lastSnapshotTime = Instant.now();
            
        } catch (SQLException e) {
            throw new PersistenceException("Failed to save snapshot", e);
        }
    }
    
    /**
     * Find snapshot nearest to target time.
     */
    public Optional<SystemState> findNearest(double targetTDB) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(
                 "SELECT * FROM snapshots ORDER BY ABS(tdb_time - ?) LIMIT 1")) {
            
            ps.setDouble(1, targetTDB);
            
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(loadSnapshot(conn, rs));
                }
            }
            
        } catch (SQLException e) {
            throw new PersistenceException("Failed to find snapshot", e);
        }
        
        return Optional.empty();
    }
    
    /**
     * Find snapshots in time range (for time jumping).
     */
    public List<SnapshotMetadata> findInRange(double startTDB, double endTDB) {
        // Implementation similar to findNearest
    }
    
    private byte[] compressState(SystemState state) {
        // Serialize to JSON then compress with GZIP
        String json = objectMapper.writeValueAsString(state);
        return gzipCompress(json.getBytes(StandardCharsets.UTF_8));
    }
}
```

### 10.3 Snapshot Scheduling

```java
public class SnapshotScheduler {
    
    private final SnapshotStore store;
    private final StateManager stateManager;
    private final SnapshotConfig config;
    
    /**
     * Determine if snapshot should be taken.
     * Called after each integration step.
     */
    public boolean shouldSnapshot(SystemState state) {
        // Time-based: every N simulation-days
        double daysSinceLastSnapshot = 
            (state.getTime() - lastSnapshotTDB) / 86400.0;
        if (daysSinceLastSnapshot >= config.getIntervalDays()) {
            return true;
        }
        
        // Event-based: at specific orbital events (optional)
        if (config.isSnapshotAtPerihelion()) {
            // Check if any planet just passed perihelion
        }
        
        return false;
    }
}
```

### 10.4 Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `snapshot.intervalDays` | 30 | Simulation days between snapshots |
| `snapshot.maxCount` | 1000 | Maximum snapshots to retain |
| `snapshot.compressionEnabled` | true | GZIP compress state data |
| `snapshot.retentionDays` | 3650 | Delete snapshots older than this |

---

## 11. Public API

### 11.1 Main Interface

```java
package com.solarsim.api;

/**
 * Main entry point for the Solar System Simulator.
 * Thread-safe for concurrent access.
 */
public interface SolarSystemSimulator {
    
    // === Lifecycle ===
    
    /**
     * Initialize and start the simulator.
     * Fetches live initial conditions from JPL Horizons.
     */
    void start() throws InitializationException;
    
    /**
     * Stop the simulator gracefully.
     * Saves final snapshot and stops all background threads.
     */
    void stop();
    
    /**
     * Check if simulator is running.
     */
    boolean isRunning();
    
    // === Position Queries ===
    
    /**
     * Get position of a body at current simulation time.
     * 
     * @param body Target body
     * @param frame Reference frame for output
     * @return Position vector in specified frame (km)
     */
    Vector3D getPosition(Body body, ReferenceFrame frame);
    
    /**
     * Get position of a body at specified time.
     * May require integration from nearest snapshot.
     * 
     * @param body Target body
     * @param frame Reference frame for output
     * @param unixTime Target time as Unix timestamp
     * @return Position vector in specified frame (km)
     */
    Vector3D getPosition(Body body, ReferenceFrame frame, long unixTime);
    
    // === Velocity Queries ===
    
    /**
     * Get velocity of a body at current simulation time.
     */
    Vector3D getVelocity(Body body, ReferenceFrame frame);
    
    /**
     * Get velocity of a body at specified time.
     */
    Vector3D getVelocity(Body body, ReferenceFrame frame, long unixTime);
    
    // === Combined State ===
    
    /**
     * Get full state vector (position + velocity) at current time.
     */
    StateVector getStateVector(Body body, ReferenceFrame frame);
    
    /**
     * Get full state vector at specified time.
     */
    StateVector getStateVector(Body body, ReferenceFrame frame, long unixTime);
    
    // === Orbital Elements ===
    
    /**
     * Get osculating orbital elements at current time.
     */
    OrbitalElements getOrbitalElements(Body body);
    
    /**
     * Get osculating orbital elements at specified time.
     */
    OrbitalElements getOrbitalElements(Body body, long unixTime);
    
    // === Angular Position (for observers) ===
    
    /**
     * Get apparent angular position as seen from observer.
     * 
     * @param target Target body to observe
     * @param observer Observer body (e.g., Earth)
     * @return Right ascension (hours), Declination (degrees), Distance (km)
     */
    SphericalCoordinates getAngularPosition(Body target, Body observer);
    
    /**
     * Get angular position at specified time.
     */
    SphericalCoordinates getAngularPosition(Body target, Body observer, long unixTime);
    
    // === Event Search ===
    
    /**
     * Find conjunctions between two bodies.
     * 
     * @param body1 First body
     * @param body2 Second body
     * @param startUnix Start of search window
     * @param endUnix End of search window
     * @param maxSeparationDegrees Maximum angular separation to consider
     * @return List of conjunction events
     */
    List<ConjunctionEvent> findConjunctions(
        Body body1, Body body2, 
        long startUnix, long endUnix, 
        double maxSeparationDegrees);
    
    /**
     * Find oppositions (body opposite to Sun as seen from Earth).
     */
    List<OppositionEvent> findOppositions(
        Body body, long startUnix, long endUnix);
    
    // === Time Control ===
    
    /**
     * Get current simulation time as Unix timestamp.
     */
    long getCurrentTimeUnix();
    
    /**
     * Get current simulation time as TDB (internal).
     */
    double getCurrentTimeTDB();
    
    /**
     * Set time scaling factor.
     * 
     * @param scale Simulation seconds per real second
     */
    void setTimeScale(double scale);
    
    /**
     * Get current time scale.
     */
    double getTimeScale();
    
    /**
     * Jump to specific time.
     * Loads nearest snapshot and integrates to target.
     */
    void jumpToTime(long unixTime);
    
    // === Metrics & Validation ===
    
    /**
     * Get current validation metrics.
     */
    ValidationMetrics getMetrics();
    
    /**
     * Get sync status and history.
     */
    SyncStatus getSyncStatus();
    
    /**
     * Force immediate synchronization with JPL.
     */
    SyncResult forceSyncNow();
    
    // === Configuration ===
    
    /**
     * Get current configuration (read-only).
     */
    SimulatorConfig getConfig();
    
    /**
     * Update configuration (some changes require restart).
     */
    void updateConfig(SimulatorConfig newConfig);
    
    // === State Export ===
    
    /**
     * Export current state to JSON.
     */
    String exportStateJson();
    
    /**
     * Export state at specific time.
     */
    String exportStateJson(long unixTime);
}
```

### 11.2 Data Transfer Objects

```java
/**
 * Immutable 3D vector.
 */
public record Vector3D(double x, double y, double z) {
    
    public Vector3D add(Vector3D other) {
        return new Vector3D(x + other.x, y + other.y, z + other.z);
    }
    
    public Vector3D subtract(Vector3D other) {
        return new Vector3D(x - other.x, y - other.y, z - other.z);
    }
    
    public Vector3D scale(double factor) {
        return new Vector3D(x * factor, y * factor, z * factor);
    }
    
    public double magnitude() {
        return Math.sqrt(x * x + y * y + z * z);
    }
    
    public double dot(Vector3D other) {
        return x * other.x + y * other.y + z * other.z;
    }
    
    public Vector3D cross(Vector3D other) {
        return new Vector3D(
            y * other.z - z * other.y,
            z * other.x - x * other.z,
            x * other.y - y * other.x
        );
    }
    
    public Vector3D normalize() {
        double mag = magnitude();
        return new Vector3D(x / mag, y / mag, z / mag);
    }
}

/**
 * Position and velocity state.
 */
public record StateVector(Vector3D position, Vector3D velocity) {
    
    /**
     * Compute orbital energy (specific, i.e., per unit mass).
     * @param mu Gravitational parameter of central body
     */
    public double specificEnergy(double mu) {
        double v2 = velocity.dot(velocity);
        double r = position.magnitude();
        return v2 / 2 - mu / r;
    }
    
    /**
     * Compute specific angular momentum vector.
     */
    public Vector3D specificAngularMomentum() {
        return position.cross(velocity);
    }
}

/**
 * Classical orbital elements.
 */
public record OrbitalElements(
    double semiMajorAxis,       // km
    double eccentricity,        // dimensionless
    double inclination,         // radians
    double longitudeOfAscendingNode,  // radians (Ω)
    double argumentOfPeriapsis, // radians (ω)
    double trueAnomaly,         // radians (ν)
    double meanAnomaly,         // radians (M)
    double period               // seconds
) {
    /**
     * Convert state vector to orbital elements.
     */
    public static OrbitalElements fromStateVector(StateVector sv, double mu) {
        Vector3D r = sv.position();
        Vector3D v = sv.velocity();
        
        // Specific angular momentum
        Vector3D h = r.cross(v);
        double hMag = h.magnitude();
        
        // Node vector
        Vector3D n = new Vector3D(0, 0, 1).cross(h);
        double nMag = n.magnitude();
        
        // Eccentricity vector
        double rMag = r.magnitude();
        double vMag = v.magnitude();
        Vector3D e = r.scale(vMag * vMag - mu / rMag)
                      .subtract(v.scale(r.dot(v)))
                      .scale(1 / mu);
        double ecc = e.magnitude();
        
        // Semi-major axis
        double energy = vMag * vMag / 2 - mu / rMag;
        double a = -mu / (2 * energy);
        
        // Inclination
        double inc = Math.acos(h.z() / hMag);
        
        // Longitude of ascending node
        double lan = Math.atan2(n.y(), n.x());
        if (lan < 0) lan += 2 * Math.PI;
        
        // Argument of periapsis
        double aop = Math.acos(n.dot(e) / (nMag * ecc));
        if (e.z() < 0) aop = 2 * Math.PI - aop;
        
        // True anomaly
        double ta = Math.acos(e.dot(r) / (ecc * rMag));
        if (r.dot(v) < 0) ta = 2 * Math.PI - ta;
        
        // Mean anomaly (via eccentric anomaly)
        double ea = Math.atan2(
            Math.sqrt(1 - ecc * ecc) * Math.sin(ta),
            ecc + Math.cos(ta)
        );
        double ma = ea - ecc * Math.sin(ea);
        if (ma < 0) ma += 2 * Math.PI;
        
        // Period
        double period = 2 * Math.PI * Math.sqrt(a * a * a / mu);
        
        return new OrbitalElements(a, ecc, inc, lan, aop, ta, ma, period);
    }
}

/**
 * Spherical coordinates for angular position.
 */
public record SphericalCoordinates(
    double rightAscension,   // hours [0, 24)
    double declination,      // degrees [-90, 90]
    double distance          // km
) {}

/**
 * Conjunction event.
 */
public record ConjunctionEvent(
    long unixTime,
    Body body1,
    Body body2,
    double separationDegrees,
    SphericalCoordinates body1Position,
    SphericalCoordinates body2Position
) {}

/**
 * Reference frame enumeration.
 */
public enum ReferenceFrame {
    ICRF_BARYCENTRIC,       // Barycentric, equatorial (ICRF)
    ICRF_HELIOCENTRIC,      // Heliocentric, equatorial
    ECLIPTIC_BARYCENTRIC,   // Barycentric, ecliptic
    ECLIPTIC_HELIOCENTRIC   // Heliocentric, ecliptic
}
```

### 11.3 Body Enumeration

```java
/**
 * Celestial body identification.
 */
public enum Body {
    // Star
    SUN(10, "Sun", 1.32712440041279419e+11, 695700.0),
    
    // Planets
    MERCURY(199, "Mercury", 2.2031868551e+04, 2439.7),
    VENUS(299, "Venus", 3.24858592000e+05, 6051.8),
    EARTH(399, "Earth", 3.98600435507e+05, 6371.0),
    MARS(499, "Mars", 4.2828375816e+04, 3389.5),
    JUPITER(599, "Jupiter", 1.26712764100000e+08, 69911.0),
    SATURN(699, "Saturn", 3.79405852000000e+07, 58232.0),
    URANUS(799, "Uranus", 5.7945563000000e+06, 25362.0),
    NEPTUNE(899, "Neptune", 6.8365271005800e+06, 24622.0),
    
    // Dwarf planets
    PLUTO(999, "Pluto", 8.71e+02, 1188.3),
    CERES(1, "Ceres", 6.26325e+01, 473.0),
    ERIS(136199, "Eris", 1.108e+03, 1163.0),
    
    // Major moons
    MOON(301, "Moon", 4.9028001184e+03, 1737.4),
    IO(501, "Io", 5.959916e+03, 1821.6),
    EUROPA(502, "Europa", 3.202739e+03, 1560.8),
    GANYMEDE(503, "Ganymede", 9.887834e+03, 2634.1),
    CALLISTO(504, "Callisto", 7.179289e+03, 2410.3),
    TITAN(606, "Titan", 8.978138e+03, 2574.7),
    TRITON(801, "Triton", 1.427598e+03, 1353.4);
    
    private final int horizonsId;
    private final String name;
    private final double gm;      // km³/s²
    private final double radius;  // km
    
    Body(int horizonsId, String name, double gm, double radius) {
        this.horizonsId = horizonsId;
        this.name = name;
        this.gm = gm;
        this.radius = radius;
    }
    
    public int getHorizonsId() { return horizonsId; }
    public String getName() { return name; }
    public double getGM() { return gm; }
    public double getRadius() { return radius; }
    
    public static Body fromHorizonsId(int id) {
        for (Body b : values()) {
            if (b.horizonsId == id) return b;
        }
        throw new IllegalArgumentException("Unknown Horizons ID: " + id);
    }
}
```

### 11.4 Builder Pattern for Configuration

```java
public class SimulatorConfig {
    
    // Bodies
    private final Set<Body> bodies;
    
    // Physics
    private final boolean enableRelativity;
    private final boolean enableOblateness;
    private final boolean enableSRP;
    private final String integratorType;
    private final double timestepDays;
    
    // Time
    private final TimeEngine.Mode timeMode;
    private final double initialTimeScale;
    
    // Sync
    private final long syncIntervalSeconds;
    private final double acceptableErrorKm;
    private final boolean autoResetEnabled;
    
    // Persistence
    private final String databaseUrl;
    private final double snapshotIntervalDays;
    
    private SimulatorConfig(Builder builder) {
        this.bodies = Collections.unmodifiableSet(builder.bodies);
        // ... copy all fields
    }
    
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private Set<Body> bodies = EnumSet.of(Body.SUN, Body.EARTH);
        private boolean enableRelativity = true;
        private boolean enableOblateness = true;
        private boolean enableSRP = false;
        private String integratorType = "yoshida4";
        private double timestepDays = 0.125;  // 3 hours
        private TimeEngine.Mode timeMode = TimeEngine.Mode.SCALED;
        private double initialTimeScale = 86400.0;
        private long syncIntervalSeconds = 86400;
        private double acceptableErrorKm = 1000;
        private boolean autoResetEnabled = false;
        private String databaseUrl = "jdbc:h2:./solarsim";
        private double snapshotIntervalDays = 30;
        
        public Builder bodies(Body... bodies) {
            this.bodies = EnumSet.copyOf(Arrays.asList(bodies));
            return this;
        }
        
        public Builder allPlanets() {
            this.bodies = EnumSet.of(
                Body.SUN, Body.MERCURY, Body.VENUS, Body.EARTH, Body.MARS,
                Body.JUPITER, Body.SATURN, Body.URANUS, Body.NEPTUNE
            );
            return this;
        }
        
        public Builder enableRelativity(boolean enable) {
            this.enableRelativity = enable;
            return this;
        }
        
        public Builder timestepDays(double days) {
            this.timestepDays = days;
            return this;
        }
        
        // ... other builder methods
        
        public SimulatorConfig build() {
            validate();
            return new SimulatorConfig(this);
        }
        
        private void validate() {
            if (bodies.isEmpty()) {
                throw new IllegalStateException("At least one body required");
            }
            if (!bodies.contains(Body.SUN)) {
                throw new IllegalStateException("Sun must be included");
            }
            if (timestepDays <= 0 || timestepDays > 10) {
                throw new IllegalStateException("Timestep must be in (0, 10] days");
            }
        }
    }
}
```

---

## 12. Validation & Metrics

### 12.1 Conservation Laws

#### 12.1.1 Total Energy

```java
public class EnergyMonitor implements ConservationMonitor {
    
    private double initialEnergy;
    private double maxDeviation = 0;
    
    @Override
    public void initialize(SystemState state) {
        this.initialEnergy = computeTotalEnergy(state);
    }
    
    @Override
    public ConservationMetric check(SystemState state) {
        double currentEnergy = computeTotalEnergy(state);
        double relativeError = Math.abs(currentEnergy - initialEnergy) 
                             / Math.abs(initialEnergy);
        
        maxDeviation = Math.max(maxDeviation, relativeError);
        
        return new ConservationMetric(
            "TotalEnergy",
            initialEnergy,
            currentEnergy,
            relativeError,
            maxDeviation
        );
    }
    
    private double computeTotalEnergy(SystemState state) {
        double kinetic = 0;
        double potential = 0;
        
        List<Body> bodies = state.getBodies();
        
        for (Body body : bodies) {
            StateVector sv = state.getStateVector(body);
            
            // Kinetic: T = (1/2) * m * v²
            // Using GM instead of m: T = (1/2) * (GM/G) * v²
            // For relative comparison, we can use GM directly
            double v2 = sv.velocity().dot(sv.velocity());
            kinetic += 0.5 * body.getGM() * v2;
            
            // Potential: U = -G * m1 * m2 / r
            for (Body other : bodies) {
                if (other.ordinal() <= body.ordinal()) continue;
                
                Vector3D r = state.getStateVector(body).position()
                    .subtract(state.getStateVector(other).position());
                double dist = r.magnitude();
                
                // U = -GM1 * GM2 / (G * r) = -GM1 * GM2 / (G * r)
                // Using GM product divided by G
                potential -= body.getGM() * other.getGM() / (Constants.G * dist);
            }
        }
        
        return kinetic + potential;
    }
}
```

#### 12.1.2 Total Angular Momentum

```java
public class AngularMomentumMonitor implements ConservationMonitor {
    
    private Vector3D initialMomentum;
    private double maxDeviation = 0;
    
    @Override
    public void initialize(SystemState state) {
        this.initialMomentum = computeTotalAngularMomentum(state);
    }
    
    @Override
    public ConservationMetric check(SystemState state) {
        Vector3D current = computeTotalAngularMomentum(state);
        Vector3D diff = current.subtract(initialMomentum);
        double relativeError = diff.magnitude() / initialMomentum.magnitude();
        
        maxDeviation = Math.max(maxDeviation, relativeError);
        
        return new ConservationMetric(
            "TotalAngularMomentum",
            initialMomentum.magnitude(),
            current.magnitude(),
            relativeError,
            maxDeviation
        );
    }
    
    private Vector3D computeTotalAngularMomentum(SystemState state) {
        Vector3D total = Vector3D.ZERO;
        
        for (Body body : state.getBodies()) {
            StateVector sv = state.getStateVector(body);
            // L = r × p = r × (m * v)
            // Using GM/G as proxy for mass
            Vector3D l = sv.position().cross(sv.velocity()).scale(body.getGM() / Constants.G);
            total = total.add(l);
        }
        
        return total;
    }
}
```

#### 12.1.3 Linear Momentum

```java
public class LinearMomentumMonitor implements ConservationMonitor {
    
    private Vector3D initialMomentum;
    
    @Override
    public void initialize(SystemState state) {
        this.initialMomentum = computeTotalLinearMomentum(state);
    }
    
    @Override
    public ConservationMetric check(SystemState state) {
        Vector3D current = computeTotalLinearMomentum(state);
        Vector3D diff = current.subtract(initialMomentum);
        
        // For barycentric frame, total momentum should be ~zero
        double error = diff.magnitude();
        
        return new ConservationMetric(
            "TotalLinearMomentum",
            initialMomentum.magnitude(),
            current.magnitude(),
            error,
            error  // Absolute, not relative for this one
        );
    }
    
    private Vector3D computeTotalLinearMomentum(SystemState state) {
        Vector3D total = Vector3D.ZERO;
        
        for (Body body : state.getBodies()) {
            StateVector sv = state.getStateVector(body);
            Vector3D p = sv.velocity().scale(body.getGM() / Constants.G);
            total = total.add(p);
        }
        
        return total;
    }
}
```

### 12.2 Integration Quality Metrics

```java
public class IntegrationMetrics {
    
    private final AtomicLong stepCount = new AtomicLong(0);
    private final AtomicLong totalStepTimeNanos = new AtomicLong(0);
    private final DoubleAdder totalSimulatedTime = new DoubleAdder();
    
    // For adaptive integrators
    private final AtomicLong rejectedSteps = new AtomicLong(0);
    private final AtomicDouble minTimestep = new AtomicDouble(Double.MAX_VALUE);
    private final AtomicDouble maxTimestep = new AtomicDouble(0);
    
    public void recordStep(double dt, long wallTimeNanos, boolean accepted) {
        stepCount.incrementAndGet();
        totalStepTimeNanos.addAndGet(wallTimeNanos);
        
        if (accepted) {
            totalSimulatedTime.add(dt);
            minTimestep.updateAndGet(prev -> Math.min(prev, dt));
            maxTimestep.updateAndGet(prev -> Math.max(prev, dt));
        } else {
            rejectedSteps.incrementAndGet();
        }
    }
    
    public IntegrationReport getReport() {
        long steps = stepCount.get();
        return new IntegrationReport(
            steps,
            totalSimulatedTime.sum(),
            totalStepTimeNanos.get() / 1_000_000.0,  // ms
            steps > 0 ? (totalStepTimeNanos.get() / steps) / 1000.0 : 0,  // μs/step
            rejectedSteps.get(),
            minTimestep.get(),
            maxTimestep.get()
        );
    }
}
```

### 12.3 Sync Metrics

```java
public class SyncMetrics {
    
    private final List<SyncRecord> history = new CopyOnWriteArrayList<>();
    private final AtomicLong successCount = new AtomicLong(0);
    private final AtomicLong failureCount = new AtomicLong(0);
    private final AtomicLong correctionCount = new AtomicLong(0);
    
    public void recordSync(SyncResult result) {
        history.add(new SyncRecord(
            System.currentTimeMillis(),
            result.getStatus(),
            result.getDivergence()
        ));
        
        switch (result.getStatus()) {
            case WITHIN_TOLERANCE:
                successCount.incrementAndGet();
                break;
            case CORRECTION_APPLIED:
            case RESET_PERFORMED:
                correctionCount.incrementAndGet();
                break;
            case NETWORK_FAILURE:
            case CRITICAL_DIVERGENCE:
                failureCount.incrementAndGet();
                break;
        }
        
        // Trim history to last 1000 entries
        while (history.size() > 1000) {
            history.remove(0);
        }
    }
    
    public SyncReport getReport() {
        return new SyncReport(
            successCount.get(),
            failureCount.get(),
            correctionCount.get(),
            getAveragePositionError(),
            getMaxPositionError(),
            history.isEmpty() ? null : history.get(history.size() - 1)
        );
    }
}
```

### 12.4 Metrics Export

```java
public class MetricsExporter {
    
    /**
     * Export all metrics as JSON.
     */
    public String exportJson(ValidationMetrics metrics) {
        return objectMapper.writeValueAsString(metrics);
    }
    
    /**
     * Export metrics in Prometheus format.
     */
    public String exportPrometheus(ValidationMetrics metrics) {
        StringBuilder sb = new StringBuilder();
        
        // Energy
        sb.append("# HELP solarsim_energy_drift_ratio Relative energy drift from initial\n");
        sb.append("# TYPE solarsim_energy_drift_ratio gauge\n");
        sb.append(String.format("solarsim_energy_drift_ratio %e\n", 
            metrics.getEnergyDrift()));
        
        // Integration
        sb.append("# HELP solarsim_integration_steps_total Total integration steps\n");
        sb.append("# TYPE solarsim_integration_steps_total counter\n");
        sb.append(String.format("solarsim_integration_steps_total %d\n", 
            metrics.getStepCount()));
        
        // Sync
        sb.append("# HELP solarsim_sync_error_km Position error at last sync (km)\n");
        sb.append("# TYPE solarsim_sync_error_km gauge\n");
        sb.append(String.format("solarsim_sync_error_km %f\n", 
            metrics.getLastSyncError()));
        
        return sb.toString();
    }
}
```

### 12.5 Alerting Interface

```java
public interface AlertListener {
    void onAlert(Alert alert);
}

public record Alert(
    AlertLevel level,
    String category,
    String message,
    Map<String, Object> details,
    Instant timestamp
) {}

public enum AlertLevel {
    INFO,       // Informational
    WARNING,    // Attention recommended
    CRITICAL    // Immediate attention required
}

public class AlertService {
    
    private final List<AlertListener> listeners = new CopyOnWriteArrayList<>();
    
    public void addListener(AlertListener listener) {
        listeners.add(listener);
    }
    
    public void sendAlert(AlertLevel level, String category, String message) {
        Alert alert = new Alert(level, category, message, Map.of(), Instant.now());
        for (AlertListener listener : listeners) {
            try {
                listener.onAlert(alert);
            } catch (Exception e) {
                logger.error("Alert listener failed", e);
            }
        }
    }
}
```

---

## 13. Error Handling

### 13.1 Exception Hierarchy

```java
// Base exception
public class SolarSimException extends RuntimeException {
    public SolarSimException(String message) { super(message); }
    public SolarSimException(String message, Throwable cause) { super(message, cause); }
}

// Initialization failures
public class InitializationException extends SolarSimException {
    private final InitFailureReason reason;
    
    public InitializationException(InitFailureReason reason, String message) {
        super(message);
        this.reason = reason;
    }
    
    public enum InitFailureReason {
        NETWORK_UNAVAILABLE,
        API_ERROR,
        INVALID_RESPONSE,
        NO_CACHED_STATE,
        CONFIGURATION_ERROR
    }
}

// Integration failures
public class IntegrationException extends SolarSimException {
    private final double simulationTime;
    private final String integratorName;
    
    public IntegrationException(String message, double time, String integrator) {
        super(message);
        this.simulationTime = time;
        this.integratorName = integrator;
    }
}

// Numerical issues
public class NumericalException extends IntegrationException {
    public NumericalException(String message, double time, String integrator) {
        super(message, time, integrator);
    }
}

// State issues
public class InvalidStateException extends SolarSimException {
    public InvalidStateException(String message) { super(message); }
}

// Persistence issues
public class PersistenceException extends SolarSimException {
    public PersistenceException(String message, Throwable cause) { super(message, cause); }
}

// Sync issues
public class SyncException extends SolarSimException {
    private final SyncFailureReason reason;
    
    public SyncException(SyncFailureReason reason, String message) {
        super(message);
        this.reason = reason;
    }
    
    public enum SyncFailureReason {
        NETWORK_ERROR,
        API_ERROR,
        PARSE_ERROR,
        CRITICAL_DIVERGENCE
    }
}
```

### 13.2 Error Recovery Strategies

```java
public class ErrorRecoveryService {
    
    /**
     * Attempt recovery from integration error.
     */
    public RecoveryResult recoverFromIntegrationError(
            IntegrationException e, 
            SystemState lastGoodState) {
        
        // Strategy 1: Reduce timestep and retry
        if (e instanceof NumericalException) {
            double newTimestep = lastGoodState.getTimestep() / 2;
            if (newTimestep >= MIN_TIMESTEP) {
                return RecoveryResult.retryWithTimestep(newTimestep);
            }
        }
        
        // Strategy 2: Load previous snapshot
        Optional<SystemState> snapshot = snapshotStore.findBefore(lastGoodState.getTime());
        if (snapshot.isPresent()) {
            return RecoveryResult.restoreSnapshot(snapshot.get());
        }
        
        // Strategy 3: Re-initialize from live data
        return RecoveryResult.reinitialize();
    }
    
    /**
     * Attempt recovery from sync error.
     */
    public RecoveryResult recoverFromSyncError(SyncException e) {
        switch (e.getReason()) {
            case NETWORK_ERROR:
                // Schedule retry
                return RecoveryResult.scheduleRetry(Duration.ofMinutes(5));
                
            case CRITICAL_DIVERGENCE:
                if (config.isAutoRecoveryEnabled()) {
                    return RecoveryResult.reinitialize();
                } else {
                    return RecoveryResult.requireManualIntervention();
                }
                
            default:
                return RecoveryResult.logAndContinue();
        }
    }
}
```

### 13.3 Graceful Degradation

| Failure | Impact | Degraded Operation |
|---------|--------|-------------------|
| Network unavailable | No sync | Continue with physics only |
| Database unavailable | No snapshots | Continue in-memory only |
| High energy drift | Accuracy loss | Alert + continue |
| Critical divergence | Major inaccuracy | Stop or reinitialize |

---

## 14. Configuration System

### 14.1 Configuration File Format

```yaml
# solarsim-config.yaml

# Bodies to simulate
bodies:
  - SUN
  - MERCURY
  - VENUS
  - EARTH
  - MARS
  - JUPITER
  - SATURN
  - URANUS
  - NEPTUNE

# Physics configuration
physics:
  integrator: yoshida4        # yoshida4, stormer-verlet, adaptive
  timestep_days: 0.125        # 3 hours
  
  forces:
    newtonian: true           # Always on
    relativistic: true        # Schwarzschild correction
    oblateness: true          # J2 for Sun, Jupiter, Saturn
    radiation_pressure: false # Solar radiation pressure
  
  oblateness_bodies:
    - SUN
    - JUPITER
    - SATURN
    - EARTH

# Time configuration
time:
  mode: SCALED               # REALTIME, SCALED, FREE_RUNNING
  initial_scale: 86400       # 1 day per second
  internal_precision: TDB    # TDB (recommended) or TT

# Synchronization
sync:
  enabled: true
  interval_seconds: 86400    # 1 day
  initial_delay_seconds: 3600
  
  thresholds:
    acceptable_km: 1000
    warning_km: 10000
    critical_km: 100000
  
  correction:
    weight: 0.5              # Blend weight for corrections
    auto_reset: false        # Auto-reset on critical divergence
  
  retry:
    attempts: 3
    delay_ms: 5000

# Persistence
persistence:
  database:
    url: jdbc:postgresql://localhost:5432/solarsim
    username: solarsim
    password: ${DB_PASSWORD}  # Environment variable
    pool_size: 5
  
  snapshots:
    interval_days: 30
    max_count: 1000
    retention_days: 3650
    compression: true

# Metrics
metrics:
  conservation_check_interval: 1000  # Every N steps
  export_prometheus: true
  prometheus_port: 9090

# Logging
logging:
  level: INFO
  physics_debug: false       # Verbose physics logging
  sync_debug: false          # Verbose sync logging

# Memory
memory:
  max_heap_mb: 2048
  state_cache_size: 100      # Cached states for queries
```

### 14.2 Environment Variable Override

All configuration can be overridden via environment variables:

```bash
SOLARSIM_PHYSICS_TIMESTEP_DAYS=0.0625
SOLARSIM_SYNC_INTERVAL_SECONDS=43200
SOLARSIM_PERSISTENCE_DATABASE_URL=jdbc:h2:mem:solarsim
```

### 14.3 Runtime Configuration Updates

```java
public interface ConfigUpdateListener {
    void onConfigUpdate(ConfigUpdate update);
}

public class ConfigUpdate {
    private final String path;           // e.g., "sync.interval_seconds"
    private final Object oldValue;
    private final Object newValue;
    private final boolean requiresRestart;
}

// Configuration changes that can be applied at runtime
public enum HotReloadableConfig {
    TIME_SCALE,
    SYNC_INTERVAL,
    SNAPSHOT_INTERVAL,
    LOG_LEVEL
}

// Configuration changes that require restart
public enum ColdConfig {
    BODIES,
    INTEGRATOR,
    FORCE_MODELS,
    DATABASE_URL
}
```

---

## 15. Extension Points

### 15.1 Custom Force Models

```java
/**
 * Example: Custom gravitational perturbation from asteroid belt.
 */
public class AsteroidBeltPerturbation implements ForceModel {
    
    // Model asteroid belt as a ring of mass
    private static final double BELT_INNER_RADIUS = 2.2 * Constants.AU;  // km
    private static final double BELT_OUTER_RADIUS = 3.2 * Constants.AU;
    private static final double BELT_MASS = 3.0e21;  // kg (total asteroid belt mass)
    
    @Override
    public Vector3D computeAcceleration(Body target, SystemState state, double time) {
        Vector3D pos = state.getStateVector(target).position();
        
        // Simplified: treat as ring at 2.7 AU with mass distributed uniformly
        double ringRadius = 2.7 * Constants.AU;
        double ringGM = Constants.G * BELT_MASS;
        
        // Acceleration from ring (approximate)
        double r = pos.magnitude();
        double z = pos.z;  // Height above ecliptic
        
        // ... complex integral for ring potential
        // This is a placeholder for actual implementation
        
        return computeRingAcceleration(pos, ringRadius, ringGM);
    }
    
    @Override
    public String getName() {
        return "AsteroidBeltPerturbation";
    }
    
    @Override
    public double getTypicalMagnitude() {
        return 1e-15;  // Very small
    }
    
    @Override
    public boolean isConservative() {
        return true;
    }
}
```

### 15.2 Custom Integrators

```java
/**
 * Example: 8th order Yoshida integrator for extreme precision.
 */
public class Yoshida8 implements Integrator {
    
    // Coefficients from Yoshida (1990)
    private static final double[] C = { /* 15 coefficients */ };
    private static final double[] D = { /* 15 coefficients */ };
    
    @Override
    public void step(SystemState state, double dt, 
                     Function<SystemState, Vector3D[]> accelerationFunc) {
        for (int i = 0; i < 15; i++) {
            // Drift
            if (C[i] != 0) {
                for (int j = 0; j < state.getBodyCount(); j++) {
                    Body body = state.getBody(j);
                    Vector3D newPos = body.getPosition()
                        .add(body.getVelocity().scale(C[i] * dt));
                    state.setPosition(j, newPos);
                }
            }
            
            // Kick
            if (D[i] != 0) {
                Vector3D[] accels = accelerationFunc.apply(state);
                for (int j = 0; j < state.getBodyCount(); j++) {
                    Body body = state.getBody(j);
                    Vector3D newVel = body.getVelocity()
                        .add(accels[j].scale(D[i] * dt));
                    state.setVelocity(j, newVel);
                }
            }
        }
    }
    
    @Override
    public int getOrder() {
        return 8;
    }
    
    @Override
    public String getName() {
        return "Yoshida8";
    }
}
```

### 15.3 Runtime Body Addition

```java
public interface BodyRegistry {
    
    /**
     * Register a new body at runtime.
     * 
     * @param body Body definition
     * @param initialState Initial state vector
     */
    void registerBody(CustomBody body, StateVector initialState);
    
    /**
     * Remove a body (e.g., after it leaves the system).
     */
    void deregisterBody(String bodyId);
}

public class CustomBody {
    private final String id;
    private final String name;
    private final double gm;           // Gravitational parameter
    private final double radius;       // Physical radius
    private final boolean isActive;    // Participates in integration
    private final boolean isPassive;   // Affected by others but doesn't affect them
    
    // Builder pattern for construction
}
```

### 15.4 Plugin Architecture

```java
/**
 * Plugin interface for extending simulator capabilities.
 */
public interface SimulatorPlugin {
    
    /**
     * Called during simulator initialization.
     */
    void initialize(SimulatorContext context);
    
    /**
     * Called after each integration step.
     */
    default void onStep(SystemState state, double dt) {}
    
    /**
     * Called during shutdown.
     */
    default void shutdown() {}
    
    /**
     * Get plugin metadata.
     */
    PluginInfo getInfo();
}

public record PluginInfo(
    String id,
    String name,
    String version,
    String description,
    List<String> dependencies
) {}

/**
 * Context provided to plugins.
 */
public interface SimulatorContext {
    void registerForceModel(ForceModel model);
    void registerIntegrator(Integrator integrator);
    void addMetric(String name, Supplier<Double> valueSupplier);
    void addAlertListener(AlertListener listener);
    StateManager getStateManager();
    TimeEngine getTimeEngine();
}
```

---

## 16. MVP Specification

### 16.1 MVP Scope

The Minimum Viable Product includes:

| Component | MVP Scope | Full Scope |
|-----------|-----------|------------|
| Bodies | Sun + Earth | All planets + moons + dwarf planets |
| Forces | Newtonian only | + GR + J2 + SRP |
| Integrator | Yoshida 4th order | + Adaptive + Yoshida 8th |
| Sync | Startup only | + Periodic + Correction |
| Persistence | In-memory | + Database + Snapshots |
| API | Position queries | + Events + Export |
| Metrics | Energy conservation | + Full suite |

### 16.2 MVP Class Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          MVP CLASSES                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┐      ┌─────────────────┐                       │
│  │  Vector3D       │      │  StateVector    │                       │
│  │  - x, y, z      │      │  - position     │                       │
│  │  + add()        │      │  - velocity     │                       │
│  │  + scale()      │      └─────────────────┘                       │
│  │  + magnitude()  │                                                 │
│  └─────────────────┘                                                 │
│                                                                      │
│  ┌─────────────────┐      ┌─────────────────┐                       │
│  │  Body (enum)    │      │  SystemState    │                       │
│  │  - SUN          │      │  - bodies[]     │                       │
│  │  - EARTH        │      │  - time         │                       │
│  │  + getGM()      │      │  + getState()   │                       │
│  └─────────────────┘      │  + copy()       │                       │
│                           └─────────────────┘                       │
│                                                                      │
│  ┌─────────────────┐      ┌─────────────────┐                       │
│  │ NewtonianGravity│      │  Yoshida4       │                       │
│  │ + computeAccel()│      │  + step()       │                       │
│  └─────────────────┘      └─────────────────┘                       │
│                                                                      │
│  ┌─────────────────┐      ┌─────────────────┐                       │
│  │ JPLHorizonsClient      │  TimeConverter  │                       │
│  │ + fetchState()  │      │  + unixToTDB()  │                       │
│  └─────────────────┘      │  + tdbToUnix()  │                       │
│                           └─────────────────┘                       │
│                                                                      │
│  ┌─────────────────────────────────────────┐                        │
│  │          SolarSystemSimulatorMVP        │                        │
│  │  - state: SystemState                   │                        │
│  │  - physics: NewtonianGravity            │                        │
│  │  - integrator: Yoshida4                 │                        │
│  │  - timeEngine: TimeEngine               │                        │
│  │  + start()                              │                        │
│  │  + getPosition(body, frame)             │                        │
│  │  + getCurrentTimeUnix()                 │                        │
│  │  + setTimeScale(scale)                  │                        │
│  └─────────────────────────────────────────┘                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.3 MVP File Structure

```
src/main/java/com/solarsim/
├── SolarSystemSimulatorMVP.java    # Main entry point
├── core/
│   ├── Vector3D.java
│   ├── StateVector.java
│   ├── SystemState.java
│   └── Body.java
├── physics/
│   ├── NewtonianGravity.java
│   └── Yoshida4.java
├── time/
│   ├── TimeEngine.java
│   ├── TimeConverter.java
│   └── JulianDate.java
├── sync/
│   └── JPLHorizonsClient.java
└── util/
    └── Constants.java
```

### 16.4 MVP Implementation Checklist

```
[ ] Core Data Structures
    [ ] Vector3D with all operations
    [ ] StateVector
    [ ] SystemState with Sun + Earth
    [ ] Body enum (MVP: SUN, EARTH only)

[ ] Physics
    [ ] NewtonianGravity force computation
    [ ] Yoshida4 integrator
    [ ] Energy computation for validation

[ ] Time
    [ ] TimeEngine with SCALED mode
    [ ] Unix ↔ TDB conversion
    [ ] Julian date utilities

[ ] Sync
    [ ] JPL Horizons HTTP client
    [ ] State vector parsing
    [ ] Startup initialization

[ ] Main Loop
    [ ] Simulation thread
    [ ] Time-scaled stepping
    [ ] Basic logging

[ ] API
    [ ] getPosition(body, frame)
    [ ] getVelocity(body, frame)
    [ ] getCurrentTimeUnix()
    [ ] setTimeScale(scale)

[ ] Testing
    [ ] Unit tests for Vector3D
    [ ] Integration test: 1-year Earth orbit
    [ ] Energy conservation test
```

---

## 17. Implementation Roadmap

### Phase 1: MVP (Week 1-2)

**Goal:** Sun-Earth system running with live initialization.

| Task | Estimate | Dependencies |
|------|----------|--------------|
| Core data structures | 2 days | None |
| Newtonian gravity | 1 day | Core |
| Yoshida4 integrator | 2 days | Core |
| Time system | 1 day | None |
| JPL client | 2 days | Time |
| Main simulation loop | 1 day | All above |
| Basic API | 1 day | Main loop |
| Testing | 2 days | All |

**Deliverable:** JAR that starts, fetches Earth position, and simulates forward.

### Phase 2: Full Solar System (Week 3-4)

**Goal:** All 8 planets with relativistic corrections.

| Task | Estimate | Dependencies |
|------|----------|--------------|
| Add all planets | 1 day | MVP |
| Schwarzschild correction | 2 days | MVP |
| J2 oblateness | 2 days | MVP |
| Frame transformations | 1 day | MVP |
| Orbital elements | 2 days | MVP |
| Expanded API | 2 days | Above |
| Testing | 2 days | All |

**Deliverable:** Full planetary system with high-accuracy physics.

### Phase 3: Persistence & Sync (Week 5-6)

**Goal:** Database snapshots and periodic synchronization.

| Task | Estimate | Dependencies |
|------|----------|--------------|
| Database schema | 1 day | None |
| Snapshot store | 2 days | Schema |
| Time jump support | 2 days | Snapshots |
| Periodic sync | 2 days | MVP |
| Divergence detection | 1 day | Sync |
| Correction strategies | 2 days | Divergence |
| Testing | 2 days | All |

**Deliverable:** System that persists state and self-corrects.

### Phase 4: Extended Bodies & Events (Week 7-8)

**Goal:** Moons, dwarf planets, conjunction search.

| Task | Estimate | Dependencies |
|------|----------|--------------|
| Moon system | 2 days | Phase 2 |
| Galilean moons | 2 days | Phase 2 |
| Dwarf planets | 1 day | Phase 2 |
| Event search API | 3 days | Phase 2 |
| Angular position API | 2 days | Phase 2 |
| Testing | 2 days | All |

**Deliverable:** Complete solar system with astronomical event queries.

### Phase 5: Production Hardening (Week 9-10)

**Goal:** Production-ready with full observability.

| Task | Estimate | Dependencies |
|------|----------|--------------|
| Error recovery | 2 days | Phase 3 |
| Metrics export | 2 days | All phases |
| Alerting system | 1 day | Metrics |
| Configuration system | 2 days | All phases |
| Plugin architecture | 2 days | All phases |
| Documentation | 2 days | All |
| Performance tuning | 1 day | All |

**Deliverable:** Production-deployable system.

---

## 18. Appendices

### Appendix A: Physical Constants

```java
public final class Constants {
    
    private Constants() {}  // Prevent instantiation
    
    // Gravitational constant (m³ kg⁻¹ s⁻²)
    public static final double G = 6.67430e-11;
    
    // Gravitational constant (km³ kg⁻¹ s⁻²)
    public static final double G_KM = 6.67430e-20;
    
    // Speed of light (km/s)
    public static final double C = 299792.458;
    
    // Astronomical Unit (km)
    public static final double AU = 149597870.7;
    
    // Solar luminosity (W)
    public static final double L_SUN = 3.828e26;
    
    // Julian century (days)
    public static final double JULIAN_CENTURY = 36525.0;
    
    // Seconds per day
    public static final double SECONDS_PER_DAY = 86400.0;
    
    // J2000.0 epoch as Julian Date
    public static final double J2000_JD = 2451545.0;
    
    // J2000.0 epoch as Unix timestamp (approximately)
    public static final long J2000_UNIX = 946728000L;
    
    // Obliquity of ecliptic at J2000.0 (radians)
    public static final double OBLIQUITY_J2000 = Math.toRadians(23.439291111);
}
```

### Appendix B: Yoshida Coefficients

```java
public final class YoshidaCoefficients {
    
    // 4th order (3 stages)
    public static final class Order4 {
        private static final double CBRT_2 = Math.cbrt(2.0);
        private static final double W0 = -CBRT_2 / (2.0 - CBRT_2);
        private static final double W1 = 1.0 / (2.0 - CBRT_2);
        
        public static final double[] C = {
            W1 / 2.0,
            (W0 + W1) / 2.0,
            (W0 + W1) / 2.0,
            W1 / 2.0
        };
        
        public static final double[] D = { W1, W0, W1, 0.0 };
    }
    
    // 6th order (7 stages)
    public static final class Order6 {
        public static final double[] C = {
            0.78451361047755726382,
            0.23557321335935813368,
            -1.17767998417887100695,
            1.31518632068391121888,
            -1.17767998417887100695,
            0.23557321335935813368,
            0.78451361047755726382,
            0.0
        };
        
        public static final double[] D = {
            0.39225680523877863191 * 2,
            0.51004341191845769436 * 2,
            -0.47105338540975564327 * 2,
            0.06875316825252010574 * 2,
            -0.47105338540975564327 * 2,
            0.51004341191845769436 * 2,
            0.39225680523877863191 * 2,
            0.0
        };
    }
}
```

### Appendix C: JPL Horizons API Reference

**Base URL:** `https://ssd.jpl.nasa.gov/api/horizons.api`

**Common Parameters:**

| Parameter | Description | Example |
|-----------|-------------|---------|
| `COMMAND` | Target body ID | `'399'` (Earth) |
| `CENTER` | Reference point | `'500@0'` (SSB) |
| `MAKE_EPHEM` | Generate ephemeris | `'YES'` |
| `EPHEM_TYPE` | Output type | `'VECTORS'` |
| `VEC_TABLE` | Vector content | `'2'` (pos+vel) |
| `REF_PLANE` | Reference plane | `'FRAME'` (ICRF) |
| `OUT_UNITS` | Output units | `'KM-S'` |
| `START_TIME` | Query start | `'2024-01-01'` |
| `STOP_TIME` | Query end | `'2024-01-02'` |

**Response Format:**

```json
{
  "signature": {...},
  "result": "... formatted ephemeris data ..."
}
```

The `result` field contains formatted text with state vectors:

```
$$SOE
2460310.500000000 = A.D. 2024-Jan-01 00:00:00.0000 TDB 
 X =-2.627...E+07 Y = 1.445...E+08 Z = 3.031...E+04
 VX=-2.979...E+01 VY=-5.378...E+00 VZ= 6.166...E-04
$$EOE
```

### Appendix D: Database Migration Scripts

```sql
-- V2__add_indices.sql
CREATE INDEX idx_body_states_body_id ON body_states(body_id);
CREATE INDEX idx_snapshots_created_at ON snapshots(created_at);

-- V3__add_metrics_aggregation.sql
CREATE TABLE metrics_hourly (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    hour_start TIMESTAMP NOT NULL,
    metric_name VARCHAR(100) NOT NULL,
    min_value DOUBLE,
    max_value DOUBLE,
    avg_value DOUBLE,
    sample_count INT,
    UNIQUE INDEX idx_hourly_metric (hour_start, metric_name)
);

-- V4__add_audit_log.sql
CREATE TABLE audit_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    event_type VARCHAR(50) NOT NULL,
    details TEXT,
    INDEX idx_audit_time (timestamp)
);
```

### Appendix E: Testing Strategy

**Unit Tests:**
- Vector3D arithmetic
- Orbital elements conversion
- Time conversions
- Individual force models

**Integration Tests:**
- 1-year Earth orbit (compare to Kepler prediction)
- Mercury perihelion precession (with GR)
- Jupiter-Saturn perturbation effects
- Full system energy conservation

**Validation Tests:**
- Compare against JPL Horizons at multiple epochs
- Long-term integration stability (100+ years)
- Sync recovery scenarios

**Performance Tests:**
- Integration speed (steps/second)
- Memory usage under load
- Snapshot save/load times
- Concurrent API query throughput

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | December 2024 | Architecture Team | Initial specification |

---

*End of Document*
