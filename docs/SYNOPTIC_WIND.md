# Synoptic Wind Subsystem  
Global / Regional Synoptic Wind Profile Builder  

Synoptic Wind builds the **baseline synoptic wind profile** from METAR, external APIs,
HRRR, GFS, and real‑world cruise‑level winds.

Synoptic Wind is a **pure atmosphere builder**:
- It constructs the global wind field from surface → top of atmosphere.
- It evaluates the profile at XP12’s altitude levels.
- It outputs only plugin-local datarefs.

The **Wind Aggregator** performs all blending and XP12 injection.

---

# 1. Purpose

- Construct a smooth, real‑world synoptic vertical wind profile from surface → FL450+.  
- Replace XP12’s native winds at *all* altitudes (full atmospheric ownership).  
- Apply smoothing across all data sources.  
- Evaluate the continuous profile **exactly at XP12’s regional altitude levels**.  
- Provide the baseline atmosphere to the Wind Aggregator.

---

# 2. Inputs

## 2.1 Observational & Model Inputs

### METAR (≤ 50 km)
- Surface wind speed \( U_{\text{METAR}} \)
- Surface wind direction \( D_{\text{METAR}} \)

### External API (fallback)
- Surface wind speed \( U_{\text{API}} \)
- Surface wind direction \( D_{\text{API}} \)

### HRRR (CONUS)
- 10 m wind \( \vec{V}_{10m}^{HRRR} \)
- Low‑level wind field \( \vec{V}_{LL}^{HRRR}(z) \)
- Boundary layer height \( BLH \)
- Shear diagnostics

### GFS 0.25°
- Pressure‑level winds \( \vec{V}^{GFS}(p) \)
- Temperature profile \( T(p) \)
- Humidity profile \( q(p) \)
- Jetstream structure
- Synoptic shear

---

## 2.2 XP12 Regional Atmosphere (Altitude Grid Only)

XP12 defines **fixed regional altitude levels**:

- `sim/weather/region/atmosphere_alt_levels_m[i]`

These levels are:
- Non‑uniform  
- Hand‑tuned  
- Used by XP12 FMS, ATC, and cruise wind interpolation  

### Why Synoptic Wind must use XP12’s levels

Even with full atmospheric ownership:

- FMS still queries winds at these exact altitudes.  
- ATC still uses these levels for climb/descent planning.  
- Using XP12’s altitude grid prevents interpolation artifacts.

### Therefore:

**Synoptic Wind must evaluate its continuous profile *exactly* at XP12’s altitude levels**,  
but it may **replace XP12’s winds at all altitudes**.

---

# 3. Synoptic Baseline Construction

We construct a continuous baseline wind profile \( \vec{V}_{syn}(z) \) from surface → top of atmosphere.

---

## 3.1 Surface Layer (0–100 m AGL)

### 3.1.1 Surface Anchor Selection
If METAR within 50 km:


\[
U_{surf} = U_{\text{METAR}}, \quad D_{surf} = D_{\text{METAR}}
\]


Else:


\[
U_{surf} = U_{\text{API}}, \quad D_{surf} = D_{\text{API}}
\]



Vector:


\[
\vec{V}_{surf} =
\begin{bmatrix}
U_{surf} \cos D_{surf} \\
U_{surf} \sin D_{surf}
\end{bmatrix}
\]



### 3.1.2 HRRR Blend (CONUS)
If HRRR available:


\[
\vec{V}_{10m}^{blend} = (1 - \beta)\vec{V}_{surf} + \beta\vec{V}_{10m}^{HRRR}
\]


Typical \( \beta = 0.3 \).

Else:


\[
\vec{V}_{10m}^{blend} = \vec{V}_{surf}
\]



Set:


\[
\vec{V}_{syn}(z \le 100\,\text{m}) = \vec{V}_{10m}^{blend}
\]



---

## 3.2 Boundary Layer (100 m–BLH)

If HRRR available:


\[
\vec{V}_{syn}(z) = \vec{V}_{LL}^{HRRR}(z)
\]



Else (GFS interpolation):


\[
\vec{V}_{syn}(z) =
\vec{V}^{GFS}(z_{GFS,1}) +
\left(
\frac{z - 100}{BLH - 100}
\right)
\left[
\vec{V}^{GFS}(z_{GFS,2}) - \vec{V}^{GFS}(z_{GFS,1})
\right]
\]



---

## 3.3 Mid‑Level (BLH–FL140)

Pressure‑level interpolation:


\[
\vec{V}_{syn}(z) =
\vec{V}^{GFS}(p_1) +
\left(
\frac{p(z) - p_1}{p_2 - p_1}
\right)
\left[
\vec{V}^{GFS}(p_2) - \vec{V}^{GFS}(p_1)
\right]
\]



Apply vertical smoothing.

---

## 3.4 Transition Band (FL140–FL180)

Originally this blended into XP12 winds.  
With **full atmospheric ownership**, this becomes:

### Smooth continuation of the GFS/HRRR profile


\[
\vec{V}_{syn}(z) = \text{smoothed}( \vec{V}_{syn}(z) )
\]



No XP12 blending is performed.

---

## 3.5 Cruise Region (≥ FL180)

With full control:



\[
\vec{V}_{syn}(z) = \vec{V}^{GFS}(p(z))
\]



or any real‑world source (ECMWF, NOAA, etc.).

XP12 cruise winds are **ignored**.

---

# 4. Evaluation at XP12 Altitude Levels  
(Required for FMS/ATC/Flight‑Level Compatibility)

For each XP12 altitude level:


\[
z_i = \text{atmosphere\_alt\_levels\_m}[i]
\]



Compute:


\[
\vec{V}_{syn}(z_i)
\]



Convert to components:


\[
V_{syn,x}(z_i) = U_{syn}(z_i)\cos D_{syn}(z_i)
\]




\[
V_{syn,y}(z_i) = U_{syn}(z_i)\sin D_{syn}(z_i)
\]



These values are passed to the **Wind Aggregator**, not written to XP12.

---

# 5. Output Datarefs (Synoptic‑Only — Plugin Local)

| Dataref Name         | Type  | Description                                      |
|----------------------|-------|--------------------------------------------------|
| `wind_syn_x_msc`     | float | Synoptic baseline X component at aircraft altitude |
| `wind_syn_y_msc`     | float | Synoptic baseline Y component at aircraft altitude |
| `wind_syn_z_msc`     | float | Synoptic baseline Z component (if modeled)        |

These are consumed by the **Wind Aggregator**, which performs:

- All subsystem blending  
- Final wind composition  
- **All XP12 wind dataref writes**  

Synoptic Wind **never** writes to XP12.

---

# 6. Architectural Notes

- Synoptic Wind is **not** the final composer.  
- It produces a **clean, continuous synoptic profile** from surface → top of atmosphere.  
- It now **fully replaces XP12 winds at all altitudes**.  
- It evaluates winds **exactly at XP12’s altitude levels** for FMS/ATC compatibility.  
- It does **not** add Terrain, Surface, Orographic, or Turbulence effects.  
- It does **not** write to XP12 regional arrays.  
- The **Wind Aggregator** performs all blending and XP12 injection.  
