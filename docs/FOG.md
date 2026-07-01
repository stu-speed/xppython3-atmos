# Fog Subsystem Specification

## 1. Overview

The Fog subsystem is responsible for detecting, modeling, and rendering fog as a distinct
meteorological phenomenon. It computes fog formation, fog density, fog depth, fog‑top height,
and fog‑limited visibility (V_fog).

Fog is not part of the Haze subsystem and not derived from AOD/BLH. Fog is driven by humidity,
temperature, dewpoint spread, wind, and terrain.

The Fog subsystem outputs:

- V_fog — fog‑limited visibility
- fog_base_height — fog base (usually terrain or valley floor)
- fog_top_height — physical fog‑top height
- fog_density — normalized fog extinction strength
- synthetic fog‑top cloud layer (optional)
- virtual METAR visibility (optional)

These outputs feed into the Visibility subsystem, which selects the final visibility.

Fog‑inserted cloud layers are treated as normal clouds by the Haze subsystem for ray‑cutting.

---

## 2. Note on “Volumetric Fog” in X‑Plane 12

X‑Plane 12 advertises “volumetric fog,” but this refers to the internal lighting and scattering
pipeline, not to true 3D fog volumes. XP12 does not support fog banks, fog patches, valley fog,
fog layers at altitude, or fog that exists in one location but not another.

XP12’s fog is a global scattering effect driven by visibility and humidity. It cannot represent:

- terrain‑dependent fog
- valley fog
- fog with physical depth
- fog‑top cloud layers
- fog that forms below ridgelines
- fog that exists away from METAR stations

For this reason, the Fog subsystem provides a full physical fog model, including fog depth,
fog‑top height, fog‑limited visibility (V_fog), and optional fog‑top cloud insertion.

---

## 3. Inputs (Free Public Data Sources + X‑Plane DSF)

The Fog subsystem consumes the following **free public data sources** and **X‑Plane DSF terrain data**.

### 3.1 NOAA METAR / SYNOP
Provides:
- Surface visibility
- Fog/mist/haze codes
- Dewpoint spread
- Cloud layers (base, coverage, type)
- Precipitation type/intensity
- Surface wind + gusts

Used for:
Fog detection, dewpoint spread collapse, cloud‑layer consistency, virtual METAR visibility.

---

### 3.2 VisualCrossing / OpenWeatherMap (Surface Conditions)
Provides:
- Surface visibility
- Relative humidity
- Temperature
- Dew point
- Surface wind

Used for:
Regional fog context between METAR stations, saturation detection, correcting METAR AUTO 10SM errors.

---

### 3.3 Open‑Meteo (Boundary Layer Height)
Provides:
- Boundary Layer Height (BLH)
- Temperature profile
- Humidity profile

Used for:
Determining fog depth, fog‑top height, inversion detection, distinguishing fog vs. haze vs. mist.

---

### 3.4 CAMS (Copernicus Atmosphere Monitoring Service)
Provides:
- Aerosol Optical Depth (AOD)
- Regional aerosol loading
- Pollution / dust / smoke levels

Used for:
Distinguishing haze from fog, preventing false fog triggers, valley haze pooling logic.

---

### 3.5 X‑Plane DSF Landclass
Provides:
- Local landclass type (urban, forest, cropland, desert, water, wetlands, tundra, etc.)
- Surface roughness
- Vegetation type
- Moisture‑retention potential
- Evapotranspiration weighting

Used for:
Fog formation likelihood (wetlands, forests, croplands), valley moisture pooling, upslope fog enhancement.

---

### 3.6 X‑Plane DSF Terrain Elevation (Mesh)
Provides:
- Terrain elevation at any lat/lon
- Valley depth
- Basin geometry
- Slope and aspect
- Terrain‑induced cold‑air drainage

Used for:
Valley fog detection, fog pooling depth, fog‑top height relative to terrain, determining when METAR stations miss valley fog.

---

### 3.7 Time of Day / Solar Flux (Internal)
Provides:
- Local time
- Solar elevation
- Solar heating rate

Used for:
Radiation fog formation (nighttime cooling), fog burn‑off timing.

---

### 3.8 Precipitation Subsystem (Internal)
Provides:
- Precipitation rate
- Precipitation type

Used for:
Avoiding double‑counting fog vs. precip visibility.

---

## 4. Fog Formation Conditions

Fog forms when near‑surface air becomes saturated and cooling or moisture transport maintains
saturation.

Fog triggers when any of the following are true:

### 4.1 Radiation Fog
- Dewpoint spread < 2°C
- RH > 95%
- Light winds (< 5 kt)
- Surface inversion present
- Nighttime or early morning cooling

### 4.2 Advection Fog
- Warm, moist air moves over a colder surface
- Dewpoint spread collapses
- Wind 5–15 kt

### 4.3 Upslope Fog
- Moist air forced upslope
- Cooling by expansion
- RH > 95%

### 4.4 Valley Fog
- Cold air drainage into valleys
- Moisture pooling
- Dewpoint spread collapses
- Often forms even when METAR stations on ridges do not report fog

---

## 5. Fog Depth and Fog‑Top Height

Fog depth is computed using:
- Surface RH
- Dewpoint spread
- Temperature lapse rate
- Terrain elevation (DSF)
- Moisture flux
- Inversion strength

Fog‑top height is the altitude where RH falls below saturation or where temperature increases
enough to break the inversion.

Outputs:

    fog_base_height (AGL)
    fog_top_height  (AGL)

---

## 6. Fog Visibility (V_fog)

Fog visibility is computed using an extinction model:

    k_fog = f(RH, dewpoint_spread, droplet_density, wind)

Visibility:

    V_fog = 3 / k_fog

Typical ranges:
- Dense fog: 50–200 m
- Moderate fog: 200–600 m
- Mist: 600–2000 m

V_fog is passed directly to the Visibility subsystem.

---

## 7. Fog‑Top Cloud Layer Injection

Fog often forms in valleys or basins where METAR stations do not see the fog layer due to terrain
shielding or distance.

When fog is present, the Fog subsystem may:

1. Compute fog depth
2. Compute fog‑top height
3. Insert a synthetic low cloud layer at fog_top_height if:
   - No METAR cloud layer exists at that height
   - Existing METAR layers are too high
   - Valley fog is present but METAR is from a ridge or distant airport

This synthetic layer:
- Does not replace METAR clouds
- Is added alongside them
- Is treated as a normal cloud layer by the Haze subsystem

---

## 8. Virtual METAR Visibility

If fog is present but the METAR does not report low visibility, the Fog subsystem generates:

    V_fog_METAR = V_fog

This acts as a virtual METAR visibility and is passed to the Visibility subsystem.

The Visibility subsystem then selects:

    V_final = min(V_fog, V_precip, V_VC, V_METAR_mod, V_ray)

---

## 9. Interaction With Other Subsystems

### 9.1 Visibility Subsystem
Consumes:
- V_fog
- fog_top_height
- fog_base_height
- virtual METAR visibility
- synthetic fog‑top cloud layer

### 9.2 Haze Subsystem
Fog‑inserted cloud layers:
- Cut haze rays
- Block haze behind them
- Do not modify haze extinction

### 9.3 Precipitation Subsystem
Fog and precipitation can coexist. Visibility subsystem resolves:

    V_weather = min(V_fog, V_precip)

---

## 10. Outputs and Datarefs (Fog Subsystem)

| Output Name                   | Dataref Name                                 | Type  | Description |
|------------------------------|-----------------------------------------------|-------|-------------|
| Fog Visibility               | sim/weather/region/fog_visibility_m           | float | Fog‑limited visibility (meters). Computed from fog extinction model and consumed by the Visibility subsystem. |
| Fog Base Height              | sim/weather/region/fog_base_m                 | float | Height of the fog base (MSL). Defines lower boundary of fog layer. |
| Fog Top Height               | sim/weather/region/fog_top_m                  | float | Height of the fog top (MSL). Defines fog depth and rendering geometry. |
| Fog Density                  | sim/weather/region/fog_density                | float | Normalized fog extinction coefficient (0–1). Controls fog opacity and distance fade. |
| Virtual METAR Visibility     | sim/weather/region/fog_metar_visibility_m     | float | METAR‑equivalent visibility synthesized when fog exists but METAR does not report it. |
| Synthetic Fog‑Cloud Base     | msc/fog/cloud_base_msc                        | float | Synthetic fog‑top cloud base used when fog layer requires a cloud cap (e.g., valley fog, radiation fog). |
| Synthetic Fog‑Cloud Top      | msc/fog/cloud_top_msc                         | float | Synthetic fog‑top cloud top height. Used to generate a cloud layer above fog. |
| Synthetic Fog‑Cloud Coverage | msc/fog/cloud_coverage_msc                    | float | Coverage (0–1) of synthetic fog‑top cloud layer. Treated as a normal cloud layer by CDS/Haze subsystems. |

---

## 11. Summary

- Fog is its own subsystem, independent of haze and precipitation.
- Uses free public data sources (METAR, VisualCrossing/OWM, Open‑Meteo BLH, CAMS) plus X‑Plane DSF landclass and terrain.
- Computes fog formation, fog depth, fog‑top height, and V_fog.
- Can insert fog‑top cloud layers even when METAR already reports clouds.
- Can generate virtual METAR visibility when fog exists but METAR does not report it.
- Provides fog visibility and fog geometry to the Visibility subsystem.
- Fog‑inserted clouds cut haze rays in the Haze subsystem.
- Produces realistic valley fog, radiation fog, advection fog, and upslope fog.
