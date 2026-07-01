# Precipitation Subsystem  
Deterministic Microphysics, Downdrafts, Microbursts, Gust Fronts  
(Rain/Snow/Hail/Graupel, Reflectivity, Cold Pools, Storm Severity)

The Precipitation Subsystem computes **microphysics‑driven wind envelopes** and  
**storm‑structure diagnostics** using METAR, HRRR, GFS, radar, and satellite data.

Outputs feed:
- Cloud Dynamics Subsystem (CDS)
- Turbulence Subsystem (turbulence‑driver envelopes)
- Wind Aggregator (deterministic wind envelopes)
- X‑Plane precipitation renderer (rain/snow/hail/puddles)

The subsystem does **not** apply attenuation.  
The subsystem does **not** generate turbulence.  
The subsystem does **not** modify visibility (provides V_precip only).

---------------------------------------------------------------------

# 1. Inputs

## 1.1 Observational & Model Inputs

### METAR (≤ 50 km)
- Precipitation type (RA, SN, PL, GR)
- Precipitation intensity (−RA, RA, +RA)
- Visibility (proxy for precip rate)
- Weather codes (TS, FZRA, SHRA, VCTS)

Derived precipitation rate:



\[
P_{\text{rate, METAR}} = f(\text{intensity code})
\]



---------------------------------------------------------------------

### External API (fallback)
Same fields as METAR.

---------------------------------------------------------------------

### HRRR (CONUS)
High‑resolution microphysics:

- Rain mixing ratio \( q_r(z) \)
- Snow mixing ratio \( q_s(z) \)
- Graupel mixing ratio \( q_g(z) \)
- Hail/graupel probability
- Reflectivity \( Z_{HRRR}(z) \)
- Vertical velocity \( W^{HRRR}(z) \)
- Cold pool strength (θe deficit)
- Downdraft CAPE (DCAPE)
- Updraft helicity (storm severity)
- Precipitation rate \( P_{HRRR} \)

---------------------------------------------------------------------

### GFS 0.25°
Global microphysics:

- Cloud water/ice content
- Rain/snow/graupel mixing ratios
- Precipitation rate
- Vertical velocity (W)
- Temperature & humidity profiles
- DCAPE (downdraft potential)
- Storm‑scale vertical shear

---------------------------------------------------------------------

### Radar (NEXRAD, where available)
- Reflectivity (dBZ)
- Hail probability
- Storm intensity class
- Gust‑front boundaries
- Microburst signatures

Derived reflectivity:



\[
Z_{\text{radar}} = \max(Z_{\text{NEXRAD}},\; Z_{\text{HRRR}})
\]



---------------------------------------------------------------------

### Satellite (GOES / Himawari / EUMETSAT)
- Cloud‑top cooling rate (convective initiation)
- Overshooting top detection
- Anvil expansion rate
- IR brightness temperature (storm severity proxy)

---------------------------------------------------------------------

## 1.2 Aircraft Inputs
- Aircraft position (lat/lon/alt)
- AGL
- Distance to storm core
- Distance to gust front boundary
- Distance to downdraft core

---------------------------------------------------------------------

# 2. Microphysics Reconstruction

## 2.1 Precipitation Type Classification
Based on:
- METAR type
- HRRR/GFS mixing ratios
- Radar reflectivity

Decision logic:
- Rain if \( q_r > q_s, q_g \)
- Snow if \( q_s > q_r, q_g \)
- Graupel/hail if \( q_g \) large and \( Z > 50 \) dBZ

---------------------------------------------------------------------

## 2.2 Precipitation Rate



\[
P_{\text{rate}} = \max(P_{\text{METAR}},\; P_{\text{HRRR}},\; P_{\text{GFS}})
\]



---------------------------------------------------------------------

## 2.3 Reflectivity Field



\[
Z = \max(Z_{\text{radar}},\; Z_{\text{HRRR}})
\]



Reflectivity class:
- Light: 5–20 dBZ  
- Moderate: 20–40 dBZ  
- Heavy: 40–50 dBZ  
- Severe: > 50 dBZ  

---------------------------------------------------------------------

## 2.4 Microphysics‑Driven Downdraft Potential
Downdraft CAPE:



\[
DCAPE = \int_{z_{LFS}}^{z_{sfc}} g \left( \frac{T_v' - T_v}{T_v} \right) dz
\]



Downdraft strength:



\[
D_{\text{micro}} = k_d \cdot \sqrt{2 \cdot DCAPE}
\]



---------------------------------------------------------------------

## 2.5 Cold Pool Strength
Cold pool (θe deficit):



\[
CP = \theta_{e,\text{env}} - \theta_{e,\text{coldpool}}
\]



Gust front potential:



\[
GF = k_{gf} \cdot CP
\]



---------------------------------------------------------------------

# 3. Storm Structure Diagnostics

## 3.1 Storm Severity Class
Based on:
- Reflectivity Z
- Updraft helicity
- Overshooting top detection
- Lightning rate (from satellite)
- DCAPE

Classes:
- Weak
- Moderate
- Strong
- Severe
- Extreme (supercell / microburst)

---------------------------------------------------------------------

## 3.2 Microburst Probability



\[
P_{\text{micro}} = f(DCAPE,\; Z,\; q_g,\; W_{\text{down}})
\]



---------------------------------------------------------------------

## 3.3 Gust Front Boundary Detection
From radar velocity divergence + cold pool gradient.

Binary flag:
- 0 = none  
- 1 = gust front present  

---------------------------------------------------------------------

# 4. Core Calculations (Wind + Rain / Phase + Wetness)

## 4.1 Precipitation‑Driven Wind Physics  
(All wind envelopes expressed in **m/s**)

### 4.1.1 Downdraft Wind Envelope



\[
\Delta V_{\text{downdraft,z}} = -k_{dz} \cdot D_{\text{micro}}
\]



### 4.1.2 Microburst Core Wind



\[
\Delta \vec{V}_{\text{microburst}} = k_{mb} \cdot P_{\text{micro}} \cdot \hat{r}
\]



Direction \( \hat{r} \) = radial outward from downdraft core.

### 4.1.3 Gust Front Outflow



\[
\Delta \vec{V}_{\text{gustfront}} = GF \cdot \hat{n}
\]



Direction \( \hat{n} \) = outward normal to cold pool boundary.

### 4.1.4 Hail/Graupel Drag Wind



\[
\Delta V_{\text{hail}} = k_h \cdot q_g
\]



### 4.1.5 Final Deterministic Wind Vector (Local)



\[
\Delta \vec{V}_{\text{precip}} =
\Delta V_{\text{downdraft}} +
\Delta \vec{V}_{\text{microburst}} +
\Delta \vec{V}_{\text{gustfront}} +
\Delta V_{\text{hail}}
\]



Precipitation Subsystem applies **no blending**.  
Wind Aggregator performs all blending, clamping, and XP12 injection.

---------------------------------------------------------------------

## 4.2 Rain / Snow / Hail Phase and Rate (XP12)

### 4.2.1 Total Precipitation Rate



\[
P_{\text{rate}} = \max(P_{\text{METAR}},\; P_{\text{HRRR}},\; P_{\text{GFS}})
\]



This is written to XP as **mm/hr**.

### 4.2.2 Phase Fractions from Mixing Ratios

Let:



\[
Q_{\text{tot}} = q_r + q_s + q_g + \epsilon
\]



with \(\epsilon\) a small number to avoid division by zero.

Then:



\[
\text{rain\_percent} = \frac{q_r}{Q_{\text{tot}}}
\]




\[
\text{snow\_percent} = \frac{q_s}{Q_{\text{tot}}}
\]




\[
\text{hail\_percent} = \frac{q_g}{Q_{\text{tot}}}
\]



These fractions are clamped to \([0, 1]\) and may be renormalized so:



\[
\text{rain\_percent} + \text{snow\_percent} + \text{hail\_percent} \approx 1
\]



---------------------------------------------------------------------

## 4.3 Surface Wetness and Puddles (XP12)

Let:

- \( P_{\text{rate}} \) = precip rate (mm/hr)  
- \( t_{\text{wet}} \) = time since onset of precip (hr)  
- \( T_{\text{sfc}} \) = surface temperature (K)  
- \( E(T_{\text{sfc}}) \in [0,1] \) = evaporation factor (higher when warm/dry)

### 4.3.1 Surface Wetness

Raw wetness accumulation:



\[
W_{\text{raw}} = k_w \cdot P_{\text{rate}} \cdot t_{\text{wet}} \cdot (1 - E(T_{\text{sfc}}))
\]



Clamped to [0, 1]:



\[
\text{surface\_wetness} = \min(1,\; W_{\text{raw}})
\]



### 4.3.2 Puddle Depth

Raw puddle depth:



\[
D_{\text{puddle, raw}} = k_p \cdot P_{\text{rate}} \cdot t_{\text{wet}} - k_{evap} \cdot E(T_{\text{sfc}})
\]



Clamped to \(\ge 0\):



\[
\text{puddle\_depth} = \max(0,\; D_{\text{puddle, raw}})
\]



---------------------------------------------------------------------

## 4.4 Turbulence‑Driver Envelopes (Scalar, Dimensionless)

### 4.4.1 Severe‑Weather Turbulence Envelope



\[
E_{\text{severe}} = k_s \cdot Z
\]



### 4.4.2 Microburst Turbulence Envelope



\[
E_{\text{micro}} = k_m \cdot P_{\text{micro}}
\]



### 4.4.3 Hail/Graupel Turbulence Envelope



\[
E_{\text{hail}} = k_{hg} \cdot q_g
\]



---------------------------------------------------------------------

# 5. Outputs (Local Plugin + X‑Plane)

## 5.1 Deterministic Wind Envelopes (Local)

| Dataref Name                                 | Type  | Description |
|----------------------------------------------|-------|-------------|
| msc/wind/precip_downdraft_z_msc              | float | Downdraft vertical wind envelope (m/s). |
| msc/wind/precip_microburst_x_msc             | float | Microburst X component (m/s). |
| msc/wind/precip_microburst_y_msc             | float | Microburst Y component (m/s). |
| msc/wind/precip_gustfront_x_msc              | float | Gust front X component (m/s). |
| msc/wind/precip_gustfront_y_msc              | float | Gust front Y component (m/s). |
| msc/wind/precip_hail_drag_msc                | float | Hail/graupel drag wind (m/s). |

## 5.2 Turbulence‑Driver Envelopes (Local)

| Dataref Name                                 | Type  | Description |
|----------------------------------------------|-------|-------------|
| msc/turb/precip_severe_msc                   | float | Severe‑weather turbulence driver (scalar). |
| msc/turb/precip_microburst_msc               | float | Microburst turbulence driver (scalar). |
| msc/turb/precip_hail_msc                     | float | Hail/graupel turbulence driver (scalar). |

## 5.3 X‑Plane Precipitation Outputs (XP12 Weather Injection)

### 5.3.1 Precipitation Rate & Phase

| Dataref Name                           | Type  | Description |
|----------------------------------------|-------|-------------|
| sim/weather/region/precipitation_rate  | float | Total precip rate (mm/hr), \(P_{\text{rate}}\). |
| sim/weather/region/rain_percent        | float | Fraction of precip that is rain (0–1). |
| sim/weather/region/snow_percent        | float | Fraction of precip that is snow (0–1). |
| sim/weather/region/hail_percent        | float | Fraction of precip that is hail/graupel (0–1). |

### 5.3.2 Surface Wetness / Puddles

| Dataref Name                           | Type  | Description |
|----------------------------------------|-------|-------------|
| sim/weather/region/surface_wetness     | float | Surface wetness (0–1), from \(W_{\text{raw}}\). |
| sim/weather/region/puddle_depth_m      | float | Puddle depth (m), from \(D_{\text{puddle}}\). |

### 5.3.3 Precipitation Type (Local)

| Dataref Name           | Type | Description                          |
|------------------------|------|--------------------------------------|
| msc/precip/type_msc    | int  | 0=none, 1=rain, 2=snow, 3=graupel/hail |

Classification consistent with §2.1:
- 0 = none if \( P_{\text{rate}} \approx 0 \)  
- 1 = rain if \( q_r > q_s, q_g \)  
- 2 = snow if \( q_s > q_r, q_g \)  
- 3 = graupel/hail if \( q_g \) large and \( Z > 50 \) dBZ  

---------------------------------------------------------------------

# 6. Notes

- Precipitation Subsystem is **deterministic**.  
- It produces **wind envelopes**, not forces.  
- It outputs **local plugin datarefs** for wind and turbulence.  
- It outputs **XP12 precipitation datarefs** for rate, phase, wetness, and puddles.  
- It does **not** modify visibility (provides V_precip only to Visibility Subsystem).  
- Wind Aggregator blends precipitation wind with all other wind subsystems.  
- Turbulence Subsystem consumes precipitation turbulence‑driver envelopes.  
- Precipitation Subsystem is the authoritative source of:  
  - **microphysics‑driven wind**  
  - **storm severity**  
  - **precipitation type and rate**  
  - **surface wetness / puddles**
