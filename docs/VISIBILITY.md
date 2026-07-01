# Visibility Subsystem Specification (Refactored to Standard Input/Output Format)

## 1. Overview

The Visibility Subsystem selects the **final meteorological visibility** used by X‑Plane.  
It does not compute haze, fog, or precipitation physics; instead, it consumes visibility
candidates from upstream subsystems and chooses the strictest real‑world limit.

Final visibility is written to:

```
sim/weather/region/visibility_reported_m
```

This value directly drives X‑Plane’s volumetric fog shader, cloud extinction, and runway
visual range logic.

---

## 2. Inputs (Upstream Subsystems)

The Visibility Subsystem consumes visibility limits from **five** upstream subsystems.  
All inputs are **local plugin datarefs** created by design.

### 2.1 Haze Subsystem (Aerosol Extinction)
Provides:
- `msc/visibility/visibility_haze_msc` — V_ray (AOD/BLH extinction)
- haze_density (used by renderer, not by Visibility)

Represents:
- Aerosol haze  
- Pollution  
- Dust  
- Smoke  
- Valley haze pooling  

---

### 2.2 Fog Subsystem (Saturation Fog Physics)
Provides:
- `msc/visibility/visibility_fog_msc` — V_fog  
- fog_base_height  
- fog_top_height  
- fog_density  
- synthetic fog‑top cloud layer  
- virtual METAR visibility (local)

Represents:
- Radiation fog  
- Valley fog  
- Advection fog  
- Upslope fog  
- Mist  

---

### 2.3 Precipitation Subsystem
Provides:
- `msc/visibility/visibility_precip_msc` — V_precip

Represents:
- Rain  
- Snow  
- Drizzle  
- Ice pellets  
- Mixed precip  

---

### 2.4 VisualCrossing / OpenWeatherMap (Surface Visibility)
Provides:
- `msc/visibility/visibility_vc_msc` — V_VC

Used for:
- Filling gaps between METAR stations  
- Correcting METAR AUTO 10SM issues  
- Providing realistic regional visibility gradients  

---

### 2.5 METAR (Corrected)
Provides:
- `msc/visibility/visibility_metar_msc` — V_METAR_mod

Corrections include:
- 10SM → “10 miles or more” (not a hard cap)  
- 9999 → “10 km or more”  
- CAVOK → unlimited visibility  
- Missing visibility fields  
- AUTO station normalization  

---

## 3. Volumetric Fog Activation (X‑Plane Renderer)

X‑Plane 12 activates volumetric fog automatically when:

- `visibility_reported_m < 10,000 m`  
- fog‑top cloud layers exist  
- humidity approaches saturation  

The Visibility Subsystem therefore controls fog **indirectly** by setting the final visibility.

Fog‑inserted cloud layers are treated as normal clouds by the renderer.

---

## 4. Final Visibility Selection

The Visibility Subsystem selects the strictest (minimum) visibility:

```
V_final = min(V_fog, V_precip, V_VC, V_METAR_mod, V_ray)
```

This ensures:

- Fog always limits visibility when present  
- Precipitation overrides haze and fog when heavy  
- Haze cannot exceed real meteorological limits  
- METAR cannot artificially cap visibility at 10SM  
- VisualCrossing fills regional gaps  
- The strictest real‑world constraint always wins  

---

## 5. Output Datarefs

### 5.1 Final Visibility (written to X‑Plane)

| Output Name        | Dataref Name                           | Type  | Description |
|--------------------|-----------------------------------------|-------|-------------|
| Final Visibility   | sim/weather/region/visibility_reported_m | float | Final meteorological visibility written to X‑Plane. Drives volumetric fog shader, cloud extinction, and runway visual range logic. |

No other visibility datarefs are written to X‑Plane.

---

## 7. Summary

- Visibility Subsystem selects the **final meteorological visibility**.  
- Consumes visibility candidates from **haze, fog, precipitation, METAR, and VisualCrossing**.  
- Final visibility is the **minimum of all upstream inputs**.  
- Drives X‑Plane’s volumetric fog shader indirectly.  
- Ensures realistic transitions between haze, fog, mist, precipitation, and clear conditions.  
- Prevents METAR 10SM from capping visibility.  
- Produces regionally accurate visibility using free public data sources and DSF terrain/landclass.
