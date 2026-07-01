# XPPython3‑Atmos

### A Unified, Real World Data‑Driven, Cohesive Atmospheric Engine for X‑Plane

**XPPython3‑Atmos is a unified, data‑driven atmospheric engine where all subsystems share a normalized Atmos dataset,
feed each other to form a cohesive physical model, and expose their results through open local datarefs**

Subsystems operate from a **shared, normalized atmospheric dataset** built from available public
data sources.  Each subsystem contributes its own deterministic fields to a common pool of **local plugin datarefs**, 
forming a unified representation of the simulated atmosphere.  The final derived dataref values are then injected into
X-Plane to provide a realistic weather environment.

Subsystems are not independent scripts — they are **cooperating modules** inside one cohesive model. 
They feed each other where physically appropriate (e.g., Visually entering a Cumulus cloud layer -> Gusts → Turbulence)-
so the aircraft should yaw and give a kick in the pants at the same time.

All atmos and subsystem datarefs are available to **external plugins** for integration, visualization, or aircraft systems.

Python is an optimal code platform to manage subprocesses, external data access, and math libraries for accurate and
efficient Algo calcs.  In addition, this project takes advantage of the XPPython3 AI assisted code development platform.

---

## All Available Public Data, Normalized Into One Atmosphere  

The engine queries each configured data source and then outputs to standardized local datarefs

Each subsystem consumes only the fields it needs.  
If a required field is missing, the subsystem simply **does not create its local `_msc` dataref`**,
and downstream modules automatically adapt.

This makes the engine:

- **data‑opportunistic**  
- **fault‑tolerant**  
- **deterministic**  
- **physically consistent**

---

## Subsystems Feed Each Other to Form a Cohesive Model  
Subsystems are designed to interoperate:

- **Precipitation** provides microphysics → used by **Cloud Dynamics**  
- **Cloud Dynamics** provides convective envelopes → used by **Turbulence**  
- **Surface Wind** provides blending scalars → used by **Wind Aggregator**  
- **Synoptic Wind** provides the global baseline → used by all wind subsystems  

Each module contributes to the shared atmospheric state, ensuring the entire system behaves as a **single, coherent model**.

---

## All Subsystem Outputs Are Exposed as Local Datarefs  
Every subsystem publishes its results as structured local plugin datarefs:

- wind fields  
- cloud‑driven envelopes  
- microphysics‑driven envelopes  
- turbulence drivers  
- visibility & haze fields  
- fog fields  
- water state fields  

These form the **Atmos Dataset** — a complete plugin‑side representation of the atmosphere.

Any plugin can read these datarefs for instumentation.

# Subsystems Overview

Follow all the subsystem hyper-links for detailed specs.

| Subsystem                                              | Purpose | Inputs (Physics / Data) | Outputs (Local / XP12 Controls Modified) |
|--------------------------------------------------------|----------|--------------------------|-------------------------------------------|
| **[Visibility (non‑haze limits)](docs/VISIBILITY.md)** | Non‑haze visibility limits (fog, precip, low‑vis events) | METAR vis, fog physics, precip rate, dewpoint spread | `sim/weather/region/visibility_reported_m` |
| **[Haze (AOD + humidity + BLH)](docs/HAZE.md)**        | Aerosol extinction, horizon washout, contrast loss | AOD, humidity profile, BLH, landclass, valley pooling | `sim/weather/region/visibility_reported_m`, `sim/private/controls/skyc/haze_density` |
| **[Fog](docs/FOG.md)**                                 | True fog formation, dissipation, pooling | Dewpoint spread, humidity, BLH, landclass, time of day | `sim/weather/region/visibility_reported_m` (primary), may bias `haze_density` |
| **[Synoptic Wind](docs/SYNOPTIC_WIND.md)**             | Large‑scale wind field construction | GFS/HRRR wind, jet level, shear, stability | `sim/weather/region/wind_speed_mps[]`, `sim/weather/region/wind_direction_deg[]`, `sim/weather/region/wind_vertical_mps[]` |
| **[Surface Wind](docs/SURFACE_WIND.md)**               | Near‑surface shaping, gustiness, marine/land transitions | Landclass, roughness length, SST, BLH, surface wind | Local: `wind_surface_x_msc`, `wind_surface_y_msc`, `wind_surface_z_msc`, plus blending scalars |
| **[Terrain Wind](docs/TERRAIN_WIND.md)**               | Mechanical turbulence, roughness drag, obstacle effects | Landclass, terrain roughness, buildings, forests | Local: `wind_terrain_x_msc`, `wind_terrain_y_msc`, `wind_terrain_z_msc`, rotor envelope |
| **[Orographic Wind](docs/OROGRAPHIC_WIND.md)**         | Terrain‑induced flow (ridge lift, valley flow, channeling) | Terrain slope/aspect, elevation, wind direction, BLH | Local: `wind_orographic_wave_x_msc`, `wind_orographic_wave_y_msc`, `wind_orographic_wave_z_msc`, rotor envelopes |
| **[Cloud Dynamics](docs/CLOUD_DYNAMICS.md)**           | Updrafts, downdrafts, anvil outflow, gust fronts, cloud shear | CAPE, CIN, W, cloud fraction, microphysics, storm structure | Local: `wind_cloud_updraft_z_msc`, `wind_cloud_downdraft_z_msc`, `wind_cloud_anvil_x_msc`, `wind_cloud_anvil_y_msc`, `wind_cloud_gustfront_x_msc`, `wind_cloud_gustfront_y_msc`, `wind_cloud_base_shear_x_msc`, `wind_cloud_base_shear_y_msc`, `wind_cloud_top_shear_x_msc`, `wind_cloud_top_shear_y_msc` |
| **[Precipitation](docs/PRECIPITATION.md)**             | Microphysics‑driven wind (microburst, gust front, hail drag) + precip visuals | LWC/IWC, vertical velocity, DCAPE, reflectivity, storm type | Local: `wind_precip_downdraft_z_msc`, `wind_precip_microburst_x_msc`, `wind_precip_microburst_y_msc`, `wind_precip_gustfront_x_msc`, `wind_precip_gustfront_y_msc`, `wind_precip_hail_drag_msc`; XP12 visuals: `sim/weather/rain_percent`, `sim/weather/snow_percent` |
| **[Wind Aggregation](docs/WIND_AGG.md)**               | Merge synoptic + orographic + terrain + surface + cloud + precip winds into unified local wind | Synoptic wind profile, terrain deflection, surface roughness, BLH, cloud ΔV, precip ΔV | Local: `wind_final_x_msc`, `wind_final_y_msc`, `wind_final_z_msc`; XP12: `sim/weather/wind_now_x_mps`, `sim/weather/wind_now_y_mps`, `sim/weather/wind_now_z_mps`, `sim/weather/region/wind_*` |
| **[Turbulence](docs/TURBULENCE.md)**                   | Mechanical + convective + wave/rotor turbulence | Shear, BLH, CAPE, stability, terrain roughness, landclass | Local stochastic deltas: `wind_turbulence_x_msc`, `wind_turbulence_y_msc`, `wind_turbulence_z_msc` (forces only, NOT XP12 wind) |
| **[Water](docs/WATER.md)**                             | Ocean/lake waves, swell, chop, water drag, floatplane interaction | Surface wind, WaveWatch III swell/wind‑sea, SST, depth, currents | `sim/private/controls/waves/*`, `sim/private/controls/water/*` |

---

# Public Data Sources and What They Provide

These are the free, publicly accessible data sources used across the weather engine, combined with
X‑Plane’s own DSF terrain system for landclass and elevation. All subsystems (Haze, Fog,
Visibility, Precipitation, Wind, Clouds) draw from these inputs.

---

## NOAA METAR / SYNOP
Provides:
- Surface visibility
- Fog/mist/haze codes
- Dewpoint spread
- Cloud layers (base, coverage, type)
- Precipitation type/intensity
- Surface wind + gusts

Used for:
Visibility limits, fog detection, cloud‑layer consistency, precipitation type, surface wind initialization.

---

## CAMS (Copernicus Atmosphere Monitoring Service)
Provides:
- Satellite‑measured aerosol optical depth (AOD)
- Regional aerosol loading
- Pollution / dust / smoke levels
- Valley haze pooling indicators

Used for:
Haze subsystem, aerosol extinction (k = AOD / BLH), haze‑limited visibility, haze_density.

---

## VisualCrossing & OpenWeatherMap (Surface Conditions)
Provides:
- Real surface visibility
- Relative humidity
- Temperature
- Dew point
- Surface wind

Used for:
Regional visibility between METAR stations, fog/mist detection, correcting METAR AUTO 10SM errors,
near‑surface saturation physics.

---

## Open‑Meteo (Boundary Layer Height)
Provides:
- Real atmospheric boundary layer height (BLH)
- Temperature profile
- Humidity profile

Used for:
Scaling haze extinction with altitude, determining fog depth, fog‑top height, inversion detection,
distinguishing haze vs. fog vs. mist.

---

## WaveWatch III (Global Wave Model)
Provides:
- Significant wave height
- Peak wave period
- Swell direction
- Wind‑wave vs swell separation

Used for:
Surface Wind subsystem wave‑dependent roughness, marine drag, coastal wind transitions, swell‑aligned flow.

---

## Global SST (OISST / GHRSST)
Provides:
- Sea surface temperature
- Marine boundary‑layer stability indicators
- Coastal temperature gradients

Used for:
Sea‑breeze modeling, marine stability, coastal fog detection, wave–wind coupling, humidity flux.

---

## Satellite Cloud Products (GOES / Himawari / EUMETSAT)
Provides:
- Cloud top height
- Cloud thickness
- Overshooting tops
- Anvil spread
- Convective initiation

Used for:
Cloud microphysics shaping, storm structure, convective realism, precipitation onset.

---

## X‑Plane DSF Landclass
Provides:
- Local landclass type (urban, forest, cropland, desert, water, wetlands, tundra, etc.)
- Surface roughness
- Vegetation type
- Moisture‑retention potential
- Evapotranspiration weighting

Used for:
Haze extinction scaling, fog formation likelihood (wetlands, forests, croplands), valley moisture pooling,
surface‑layer physics, turbulence roughness length.

---

## X‑Plane DSF Terrain Elevation (Mesh)
Provides:
- Terrain elevation at any lat/lon
- Valley depth
- Basin geometry
- Slope and aspect
- Terrain‑induced cold‑air drainage
- Ridge lift geometry
- Wave and rotor geometry

Used for:
Valley fog detection, fog pooling depth, fog‑top height relative to terrain, haze pooling, ridge lift,
waves, rotors, convergence.

## 🧮 Required Python Libraries for All Atmos Engine Modules  
(Cloud Modeling, Cloud Dynamics, Turbulence, Thermals, Wind, Terrain)

The Atmos Engine uses a set of **high‑precision scientific libraries** to ensure  
accurate wind‑field synthesis, turbulence generation, cloud modeling, thermal  
modulation, and terrain‑aware interpolation.

These libraries are **required** for development and offline computation.

---

# ✅ Required Libraries (Core Math, Physics, Interpolation, Noise)

| Library | Purpose | Used In |
|--------|---------|---------|
| **NumPy** | Vectorized math, trig, FFT, stable floating‑point ops | Turbulence, CDS, CMS, thermals, wind shear, CU‑lift |
| **SciPy (signal)** | HPF/LPF filters, spectral shaping, FFT | Dryden/VK, CAT, rotor/wave‑breaking, severe weather |
| **SciPy (interpolate)** | Terrain interpolation, bilinear/bicubic, RBF | Terrain Wind, Orographic Wind, CMS occupancy grid |
| **SciPy (spatial)** | KD‑trees, nearest‑neighbor, Delaunay | Terrain sampling, wind‑grid lookup, cloud‑grid lookup |
| **NumPy Random (PCG64)** | High‑quality turbulence noise | FBM, mechanical BL turbulence, CAT, microburst noise |
| **noise / opensimplex** | Perlin/Simplex/OpenSimplex noise | CU‑lift cores, HF turbulence FBM, rotor turbulence |
| **PySolar / Astral** | Accurate solar elevation/azimuth | Thermal modulation, solar‑driven turbulence cycle |
| **xarray** | Multi‑dimensional atmospheric datasets | HRRR/GFS ingestion for CMS/CDS |
| **netCDF4** | Reading HRRR/GFS/WRF files | CMS cloud fraction, W(z), qc/qi, CAPE/CIN |
| **pyproj** | Coordinate transforms | Terrain sampling, wind‑grid alignment |
| **Shapely** | Geometry ops | Terrain polygons, ridge detection, gap flows |

---

# 📘 Module‑by‑Module Breakdown

### 🌤 Cloud Modeling Subsystem (CMS)
**Required:**
- NumPy  
- SciPy (interpolate, spatial)  
- xarray  
- netCDF4  
- noise / opensimplex  

**Used for:**
- Cloud occupancy grid  
- Cloud fraction interpolation  
- Patch noise fields  
- Convective core detection  
- Terrain‑fixed cloud sampling  

---

### ⛈ Cloud Dynamics Subsystem (CDS)
**Required:**
- NumPy  
- SciPy (signal)  
- noise / opensimplex  
- PySolar / Astral  

**Used for:**
- Updraft/downdraft envelopes  
- Anvil divergence  
- Gust‑front strength  
- CU‑lift FBM fields  
- Thermal modulation (sun angle × shading)  

---

### 🌡 Thermal Subsystem
**Required:**
- NumPy  
- PySolar / Astral  
- noise / opensimplex  

**Used for:**
- Solar elevation  
- Ground‑heating factor  
- Thermal strength modulation  
- FBM thermal cores  

---

### 🌬 Turbulence Subsystem
**Required:**
- NumPy  
- SciPy (signal)  
- NumPy Random (PCG64)  
- noise / opensimplex  

**Used for:**
- Dryden LF/HF  
- Von Kármán LF/HF  
- HPF/LPF shaping  
- CAT turbulence  
- Rotor/wave‑breaking  
- Microburst turbulence  
- FAA 1‑Cosine gusts  
- FBM sharp turbulence  

---

### 🏔 Terrain Wind Subsystem
**Required:**
- SciPy (interpolate)  
- SciPy (spatial)  
- NumPy  
- pyproj  
- Shapely  

**Used for:**
- Terrain slope sampling  
- Ridge/valley detection  
- Gap‑flow identification  
- Terrain‑aligned wind interpolation  

---

### 🌍 Orographic Wind Subsystem
**Required:**
- SciPy (interpolate)  
- SciPy (spatial)  
- NumPy  
- Shapely  

**Used for:**
- Mountain wave laminar component  
- Rotor zone detection  
- Divergence field computation  

---

### 🌐 Synoptic Wind Subsystem
**Required:**
- NumPy  
- xarray  
- netCDF4  

**Used for:**
- Shear profiles  
- Jetstream sampling  
- W(z) vertical velocity fields  

---

### 🌫 Surface Wind Subsystem
**Required:**
- NumPy  
- SciPy (signal)  

**Used for:**
- Mechanical turbulence  
- Surface gust envelopes  

---

# 📊 Optional Visualization & Debugging Libraries

These are **not required** for runtime, but extremely useful for  
debugging turbulence spectra, FBM fields, filter responses, and wind envelopes.

| Library | Purpose |
|--------|---------|
| **Matplotlib** | Plot Dryden/VK PSD, HPF/LPF curves, rotor envelopes |
| **Pandas** | Analyze turbulence logs, envelope tuning |
| **Seaborn** | Statistical visualization of turbulence distributions |
| **Plotly** | Interactive 3D visualization of FBM, CU‑lift, rotor fields |
| **Mayavi / PyVista** | 3D volumetric visualization of cloud occupancy & turbulence volumes |

---

## 🧠 Summary

These libraries provide the numerical accuracy required for:

- Spectrally correct turbulence  
- Accurate cloud modeling  
- Terrain‑aware wind interpolation  
- Physically correct thermal modulation  
- Deterministic, reproducible atmospheric behavior  

Optional visualization tools help validate and tune the system.

If you want, I can generate a **pip install section** or a **developer environment setup** for your README.
