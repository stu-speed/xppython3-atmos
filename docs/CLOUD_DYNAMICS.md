# Cloud Dynamics Subsystem (CDS)
Deterministic Cloud‑Driven Wind, Convective Structure, CU‑Lift, and Thermal Modulation  
(Reads authoritative cloud state from CMS)

CDS computes:

- Updrafts / downdrafts  
- Cloud‑base shear  
- Cloud‑top shear  
- Anvil outflow  
- Gust‑front winds  
- Convective turbulence  
- CU‑lift  
- Thermal modulation (sun angle × CMS shading × coverage)  

CDS **does not** determine cloud geometry or occupancy.  
CDS reads **CMS layer datarefs** and **CMS cloud‑state flags**.

---------------------------------------------------------------------

# 0. Cloud Source Selection

CDS reads cloud geometry from CMS:

```
cloud_source_mode_msc =
    0 → CMS computed layers (Atmos)
    1 → CMS copied XP12 layers
    2 → CMS copied external engine layers
```

CDS never reads XP12 or external layers directly.

---------------------------------------------------------------------

# 1. Inputs

## 1.1 CMS Layer Datarefs (Authoritative)

CDS reads:

```
cloud/layer_base_msl_msc[i]
cloud/layer_top_msl_msc[i]
cloud/layer_thickness_msc[i]
cloud/layer_type_msc[i]
cloud/layer_coverage_msc[i]
cloud/layer_fraction_profile_msc[i]   (optional)
```

## 1.2 CMS Effective Column

```
cloud/base_msl_msc
cloud/top_msl_msc
cloud/type_msc
cloud/coverage_msc
```

## 1.3 CMS Occupancy & Flags

```
cloud/occupancy_msc
cloud/in_cloud_flag_msc
cloud/under_cloud_flag_msc
cloud/clear_gap_flag_msc
cloud/convective_core_flag_msc
cloud/anvil_flag_msc
```

## 1.4 CMS Convective Structure

```
cloud/updraft_core_strength_msc
cloud/downdraft_core_strength_msc
cloud/anvil_divergence_msc
cloud/gustfront_strength_msc
cloud/vertical_shear_base_msc
cloud/vertical_shear_top_msc
```

## 1.5 Atmospheric Inputs (Optional Enhancements)

- HRRR W(z), CAPE, CIN  
- GFS W(z), cloud fraction  
- Radar reflectivity  
- Satellite anvil extent  

Used only when CMS is in compute mode.

## 1.6 Aircraft Inputs

- lat/lon/alt  
- AGL  
- Distance to cloud base/top  
- Horizontal position for sampling  

---------------------------------------------------------------------

# 2. Cloud Geometry (From CMS)

CDS does **not** reconstruct cloud geometry.

CDS uses:

```
CB = cloud/base_msl_msc
CT = cloud/top_msl_msc
cloud_fraction_msc = cloud/coverage_msc
cloud_type_msc = cloud/type_msc
```

Layer‑specific geometry is available via:

```
cloud/layer_base_msl_msc[i]
cloud/layer_top_msl_msc[i]
cloud/layer_coverage_msc[i]
cloud/layer_type_msc[i]
```

---------------------------------------------------------------------

# 3. Convective Structure Diagnostics

## 3.1 Updraft Strength



\[
U_{\text{up}} = cloud/updraft\_core\_strength\_msc
\]



## 3.2 Downdraft Strength



\[
D_{\text{down}} = cloud/downdraft\_core\_strength\_msc
\]



## 3.3 Cloud‑Base Shear



\[
S_{\text{base}} = cloud/vertical\_shear\_base\_msc
\]



## 3.4 Cloud‑Top Shear



\[
S_{\text{top}} = cloud/vertical\_shear\_top\_msc
\]



## 3.5 Anvil Divergence



\[
A_{\text{div}} = cloud/anvil\_divergence\_msc
\]



## 3.6 Gust‑Front Strength



\[
G_{\text{front}} = cloud/gustfront\_strength\_msc
\]



---------------------------------------------------------------------

# 4. Cloud‑Driven Wind Physics

## 4.1 Updraft Wind Envelope



\[
\Delta V_{\text{updraft,z}} = k_u \cdot U_{\text{up}}
\]



## 4.2 Downdraft Wind Envelope



\[
\Delta V_{\text{downdraft,z}} = -k_d \cdot D_{\text{down}}
\]



## 4.3 Anvil Outflow



\[
\Delta \vec{V}_{\text{anvil}} = k_a \cdot A_{\text{div}}
\]



## 4.4 Gust Front



\[
\Delta \vec{V}_{\text{gustfront}} = k_g \cdot G_{\text{front}}
\]



## 4.5 Cloud‑Base Shear



\[
\Delta \vec{V}_{\text{base\_shear}} = k_b \cdot S_{\text{base}}
\]



## 4.6 Cloud‑Top Shear



\[
\Delta \vec{V}_{\text{top\_shear}} = k_t \cdot S_{\text{top}}
\]



---------------------------------------------------------------------

# 5. Turbulence‑Driver Envelopes

## 5.1 Convective Turbulence



\[
E_{\text{conv}} = k_c \cdot (U_{\text{up}} + D_{\text{down}})
\]



## 5.2 Cloud‑Top CAT



\[
E_{\text{CAT}} = k_{cat} \cdot S_{\text{top}}
\]



## 5.3 Microburst Turbulence



\[
E_{\text{micro}} = k_m \cdot G_{\text{front}}
\]



## 5.4 Anvil Turbulence



\[
E_{\text{anvil}} = k_{anv} \cdot cloud/coverage\_msc
\]



---------------------------------------------------------------------

# 6. Cumulus Thermal Lift (CU‑Lift)

## 6.1 Activation Logic

```
isCU = (cloud_type_msc == CU or cloud_type_msc == TCU)
belowBase = aircraft_alt < CB
lclMatch = abs(LCL - CB) < 150
thermalActive = wind_thermal_strength_msc > 0.5
updraftPresent = U_up > 0.5
notOvercast = cloud/coverage_msc <= 0.7

thermal_cu_lift_flag_msc =
    isCU AND belowBase AND lclMatch AND
    thermalActive AND updraftPresent AND notOvercast
```

## 6.2 FBM Thermal Core Field

```
N(x,y,z) = FBM(x*sxy, y*sxy, z*sz)
core_mask = smoothstep(0.6, 0.8, normalize(N))
```

Typical:

```
sxy = 1/300
sz  = 1/500
```

## 6.3 Wind‑Tilted Thermal Column

```
tilt_offset = wind_dir_unit * tilt_factor * (CB - surface_alt)
(x',y') = (x - tilt_offset_x, y - tilt_offset_y)
```

## 6.4 Lift + Sink Structure

```
L = core_mask * wind_thermal_strength_msc
S = -sink_factor * wind_thermal_strength_msc * (1 - core_mask)
W_CU = L + S
```

---------------------------------------------------------------------

# 7. Thermal Modulation (Sun Angle × CMS Shading × Coverage)

CDS computes sun elevation:

```
sun_elevation_deg = compute_solar_elevation(lat, lon, time)
sun_factor = max(0, sin(radians(sun_elevation_deg)))
```

Coverage penalty:

```
if cloud/coverage_msc <= 0.4: coverage_penalty = 0
elif cloud/coverage_msc >= 0.7: coverage_penalty = 1
else: coverage_penalty = (cloud/coverage_msc - 0.4) / 0.3
```

Ground heating factor:

```
f_heat = sun_factor * (1 - cloud/occupancy_msc) * (1 - coverage_penalty)
```

Final thermal strength:

```
thermal_strength_msc = thermal_base_strength_msc * f_heat
```

Hard kill:

```
if cloud/coverage_msc > 0.7:
    thermal_strength_msc = 0
```

---------------------------------------------------------------------

# 8. Final Deterministic Wind Output



\[
\Delta \vec{V}_{cloud} =
\Delta V_{\text{updraft}} +
\Delta V_{\text{downdraft}} +
\Delta \vec{V}_{\text{anvil}} +
\Delta \vec{V}_{\text{gustfront}} +
\Delta \vec{V}_{\text{base\_shear}} +
\Delta \vec{V}_{\text{top\_shear}}
\]



CU‑lift vertical component:



\[
\Delta V_{CU,z} = W_{CU}
\]



---------------------------------------------------------------------

# 9. Output (Registered Datarefs)

## 9.1 Deterministic Wind Envelopes

```
wind_cloud_updraft_z_msc
wind_cloud_downdraft_z_msc
wind_cloud_anvil_x_msc
wind_cloud_anvil_y_msc
wind_cloud_gustfront_x_msc
wind_cloud_gustfront_y_msc
wind_cloud_base_shear_x_msc
wind_cloud_base_shear_y_msc
wind_cloud_top_shear_x_msc
wind_cloud_top_shear_y_msc
wind_cloud_cu_lift_z_msc
```

## 9.2 Turbulence‑Driver Envelopes

```
wind_cloud_convective_strength_msc
wind_cloud_top_CAT_envelope_msc
wind_cloud_microburst_turbulence_msc
wind_cloud_anvil_turbulence_msc
```

## 9.3 Thermal Modulation Outputs

```
wind_thermal_strength_msc
wind_thermal_sun_factor_msc
wind_thermal_ground_heating_msc
```

## 9.4 Cloud Source Mode

```
cloud_source_mode_msc
```

---------------------------------------------------------------------

# 10. Notes

- CDS reads **CMS cloud layers only**.  
- CDS never reconstructs cloud geometry unless CMS is in compute mode.  
- CU‑lift is blotchy, tilted, and core‑based.  
- Overcast (coverage > 0.7) disables thermals.  
- Sun angle modulates thermal strength.  
- CMS provides shading; CDS computes heating.  
- Wind Aggregator blends CDS output with all other wind subsystems.
