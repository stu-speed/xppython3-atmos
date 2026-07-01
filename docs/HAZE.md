# Haze Subsystem Specification

## 1. Overview

The Haze subsystem models **aerosol‑based atmospheric extinction** using real aerosol optical depth
(AOD), boundary layer height (BLH), DSF landclass, and DSF terrain. It computes:

- k_haze — aerosol extinction coefficient  
- V_ray — haze‑limited visibility  
- haze_density — shader extinction multiplier  
- upward and downward ray extinction  
- cloud‑blocked haze  
- valley haze pooling  

Haze is **not** fog. Fog is handled by the Fog subsystem.  
Haze is **not** precipitation. Precipitation is handled by the Precipitation subsystem.

The Haze subsystem outputs **V_ray** and **haze_density**, which are consumed by the Visibility subsystem.

---

## 2. Inputs (Free Public Data Sources + X‑Plane DSF)

The Haze subsystem consumes the following inputs:

### 2.1 CAMS (Copernicus Atmosphere Monitoring Service)
Provides:
- Aerosol Optical Depth (AOD)
- Regional aerosol loading
- Pollution / dust / smoke levels

Used for:
- Aerosol extinction (k = AOD / BLH)
- Regional haze realism
- Valley haze pooling

---

### 2.2 Open‑Meteo (Boundary Layer Height)
Provides:
- Boundary Layer Height (BLH)
- Temperature profile
- Humidity profile

Used for:
- Scaling haze extinction with altitude
- Determining haze layer thickness
- Distinguishing haze vs. fog

---

### 2.3 VisualCrossing / OpenWeatherMap (Surface Visibility)
Provides:
- Real surface visibility
- Relative humidity
- Temperature
- Dew point

Used for:
- Cross‑checking haze visibility against real surface conditions
- Preventing unrealistic haze when visibility is high

---

### 2.4 X‑Plane DSF Landclass
Provides:
- Landclass type (urban, forest, cropland, desert, water, wetlands, tundra)
- Surface roughness
- Vegetation type
- Aerosol source strength

Used for:
- Landclass‑dependent haze multipliers
- Valley haze pooling
- Urban pollution weighting

---

### 2.5 X‑Plane DSF Terrain Elevation (Mesh)
Provides:
- Terrain elevation
- Valley depth
- Basin geometry
- Slope and aspect

Used for:
- Valley haze pooling
- Terrain‑dependent extinction
- Downward ray geometry

---

### 2.6 Cloud Subsystem (Internal)
Provides:
- Cloud base
- Cloud top
- Cloud coverage

Used for:
- Cutting haze rays behind clouds
- Blocking haze beyond cloud layers

---

## 3. Aerosol Extinction (k_haze)

AOD and BLH define the base extinction:

    k_haze = AOD / BLH

Humidity correction:

    k_haze_eff = k_haze × (1 + 3 × RH²)

Landclass multipliers (examples):
- Urban: ×1.4  
- Cropland: ×1.2  
- Forest: ×1.1  
- Desert: ×1.3  
- Wetlands: ×1.0  
- Water: ×0.8  

Valley pooling:
- Additional multiplier based on valley depth and cold‑air drainage.

---

## 4. Ray‑Trace Extinction

Two rays are traced:

### 4.1 Upward Ray (+45°)
- Used to determine haze above the aircraft
- Ends at BLH top or cloud base
- Landclass influence decays with altitude
- Cloud layers cut the ray

### 4.2 Downward Ray (−25°)
Matches pilot’s dominant sightline.

- 40 km length
- 4 terrain‑intersecting samples
- Full landclass and humidity physics
- Valley haze pooling applied
- Cloud layers cut the ray

---

## 5. Pilot View Geometry (Critical)

The pilot primarily looks **downward at −25°**.

### Above 1000 ft AGL:
Visibility blends upward and downward rays:

    V_ray = 0.6 × V_down + 0.4 × V_up

### Below 1000 ft AGL:
Pilot view is dominated by the ground:

    V_ray = V_down

A smooth transition occurs between 800–1200 ft.

---

## 6. Cloud Interaction

Clouds block haze behind them.

Rules:
- If cloud coverage > 0.70, the ray is cut
- Haze behind the cloud is ignored
- Fog‑inserted cloud layers behave exactly like real clouds

This ensures:
- No haze above overcast
- No haze behind thick cloud decks
- Proper horizon fade under cloud layers

---

## 7. haze_density (Shader Extinction)

Raw haze density:

    haze_density_raw = 0.5 × sqrt(k_haze_eff / k_ref)

Clamp:

    haze_density = clamp(haze_density_raw, 0.0, 1.0)

Written to:

    sim/private/controls/skyc/haze_density

This controls:
- Horizon fade
- Terrain washout
- Sky/ground blending

---

## 8. Output Visibility (V_ray)

The Haze subsystem outputs:

    V_ray = haze‑limited visibility

This is **not** the final visibility.  
The Visibility subsystem selects the strictest limit.

---

## 9. Interaction With Other Subsystems

### 9.1 Visibility Subsystem
Consumes:
- V_ray
- haze_density

Visibility subsystem selects:

    V_final = min(V_fog, V_precip, V_VC, V_METAR_mod, V_ray)

### 9.2 Fog Subsystem
- Fog‑top cloud layers cut haze rays
- Fog visibility overrides haze when fog is present

### 9.3 Precipitation Subsystem
- Precipitation visibility overrides haze when heavy

### 9.4 Cloud Subsystem
- Cloud layers block haze below them

---

## 10. Outputs and Datarefs (Haze Subsystem — Local Debug)

| Output Name                    | Dataref Name                               | Type  | Description |
|--------------------------------|---------------------------------------------|-------|-------------|
| Haze Visibility (XP)           | sim/weather/region/haze_visibility_m        | float | Final meteorological visibility (m). Written to XP. |
| Haze Density (XP)              | sim/private/controls/skyc/haze_density      | float | Extinction coefficient for horizon fade. Written to XP. |
| Haze k‑Factor (local debug)    | msc/haze/k_factor_msc                       | float | Internal extinction multiplier before final haze_density. |
| Valley Haze Factor (local)     | msc/haze/valley_factor_msc                  | float | Valley aerosol enhancement when aircraft < BLH. |
| Landclass Haze Factor (local)  | msc/haze/landclass_factor_msc               | float | Aerosol weighting based on landclass (urban/forest/desert/water). |


---

## 11. Summary

- Haze subsystem models aerosol extinction using CAMS AOD, BLH, DSF landclass, and DSF terrain.
- Computes k_haze, V_ray, haze_density, and ray‑trace extinction.
- Pilot view geometry (−25° down) is fully modeled.
- Cloud layers cut haze rays, including fog‑inserted layers.
- Visibility subsystem selects the final visibility.
- Produces realistic haze, pollution, dust, smoke, and valley haze pooling.
