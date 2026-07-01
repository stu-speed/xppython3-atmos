# Wind Aggregator Subsystem  
Final Wind Composer for XP12 Injection  
(Deterministic Summation + Vertical Blending)

The Wind Aggregator is the **final stage** of the wind pipeline.  
It has three responsibilities:

1. **Compose the final current wind at aircraft altitude** (x, y, z)  
2. **Apply vertical blending to the Synoptic 37‑layer profile below a cutoff altitude**  
3. **Write ALL XP12 wind datarefs** (Aggregator is the ONLY subsystem allowed to do so)

All other subsystems write only to local plugin `_msc` refs.

---------------------------------------------------------------------

# 1. Inputs

## 1.1 Synoptic Wind (Baseline Atmosphere)

Local baseline at aircraft altitude:
- wind_syn_x_msc  
- wind_syn_y_msc  
- wind_syn_z_msc  

Smoothed 37‑layer baseline profile:
- syn_wind_speed_mps[0..36]  
- syn_wind_direction_deg[0..36]  
- syn_wind_vertical_mps[0..36]  

This is the **authoritative global atmosphere** before blending.

---------------------------------------------------------------------

## 1.2 Surface Wind (ΔV_surface + Blending Fields)

Deterministic near‑surface physics (including **marine/water effects**):
- wind_surface_x_msc  
- wind_surface_y_msc  
- wind_surface_z_msc  

Vertical blending fields (computed in Surface Wind):
- surface_atten_global_msc  
- surface_atten_terrain_msc  
- surface_atten_thermal_msc  
- surface_atten_turb_msc (for Turbulence subsystem only)

---------------------------------------------------------------------

## 1.3 Terrain Wind (Near‑Field Deterministic)

Deterministic ΔV:
- wind_terrain_x_msc  
- wind_terrain_y_msc  
- wind_terrain_z_msc  

Rotor envelope (for Turbulence subsystem only):
- wind_terrain_rotor_envelope_msc  

---------------------------------------------------------------------

## 1.4 Orographic Wind (Far‑Field Deterministic)

Deterministic envelopes:
- wind_orographic_wave_x_msc  
- wind_orographic_wave_y_msc  
- wind_orographic_wave_z_msc  

- wind_orographic_rotor_x_msc  
- wind_orographic_rotor_y_msc  
- wind_orographic_rotor_z_msc  

Divergence envelope (for Turbulence subsystem only):
- wind_orographic_divergence_msc  

---------------------------------------------------------------------

## 1.5 Cloud Dynamics Subsystem (CDS) — Deterministic Cloud‑Driven Wind

Vertical cloud‑driven wind:
- wind_cloud_updraft_z_msc  
- wind_cloud_downdraft_z_msc  

Horizontal cloud‑driven wind:
- wind_cloud_anvil_x_msc  
- wind_cloud_anvil_y_msc  
- wind_cloud_gustfront_x_msc  
- wind_cloud_gustfront_y_msc  
- wind_cloud_base_shear_x_msc  
- wind_cloud_base_shear_y_msc  
- wind_cloud_top_shear_x_msc  
- wind_cloud_top_shear_y_msc  

CDS turbulence‑driver envelopes are **not** used here.

---------------------------------------------------------------------

## 1.6 Precipitation Subsystem — Deterministic Microphysics‑Driven Wind

Vertical microphysics wind:
- wind_precip_downdraft_z_msc  

Horizontal microphysics wind:
- wind_precip_microburst_x_msc  
- wind_precip_microburst_y_msc  
- wind_precip_gustfront_x_msc  
- wind_precip_gustfront_y_msc  
- wind_precip_hail_drag_msc  

Precipitation turbulence‑driver envelopes are **not** used here.

---------------------------------------------------------------------

# 2. Vertical Blending Application  
(Why the Aggregator — and only the Aggregator — applies blending)

Surface Wind provides blending scalars at aircraft altitude:

- a_terrain = surface_atten_terrain_msc  
- a_turb    = surface_atten_turb_msc    (used by Turbulence subsystem only)  
- a_thermal = surface_atten_thermal_msc  
- a_global  = surface_atten_global_msc  

These are **vertical blending weights**, not suppression factors.

---------------------------------------------------------------------

## 2.1 Why blending exists  
Blending ensures:

- **near‑surface physics dominate near the ground**  
- **synoptic winds dominate aloft**  
- **smooth transitions** between subsystems  
- **no discontinuities** in the composed wind  

---------------------------------------------------------------------

## 2.2 Blended Terrain Wind


\[
\Delta \vec{V}_{terrain,blend} = a_{terrain} \cdot \Delta \vec{V}_{terrain}
\]



---------------------------------------------------------------------

## 2.3 Blended Thermals


\[
\Delta \vec{V}_{thermal,blend} = a_{thermal} \cdot \Delta \vec{V}_{thermal}
\]



---------------------------------------------------------------------

## 2.4 Blended Global Synoptic Wind


\[
\vec{V}_{syn,blend} = a_{global} \cdot \vec{V}_{syn}
\]



---------------------------------------------------------------------

# 3. Final Current Wind Composition (Aircraft Altitude Only)



\[
\vec{V}_{final} =
\vec{V}_{syn,blend}
+ \Delta \vec{V}_{surface}
+ \Delta \vec{V}_{terrain,blend}
+ \Delta \vec{V}_{orographic\_wave}
+ \Delta \vec{V}_{orographic\_rotor}
+ \Delta \vec{V}_{thermal,blend}
+ \Delta \vec{V}_{cloud}
+ \Delta \vec{V}_{precip}
\]



Where:

### Cloud Dynamics contribution:


\[
\Delta \vec{V}_{cloud} =
\begin{bmatrix}
wind\_cloud\_anvil\_x\_msc +
wind\_cloud\_gustfront\_x\_msc +
wind\_cloud\_base\_shear\_x\_msc +
wind\_cloud\_top\_shear\_x\_msc \\
wind\_cloud\_anvil\_y\_msc +
wind\_cloud\_gustfront\_y\_msc +
wind\_cloud\_base\_shear\_y\_msc +
wind\_cloud\_top\_shear\_y\_msc \\
wind\_cloud\_updraft\_z\_msc +
wind\_cloud\_downdraft\_z\_msc
\end{bmatrix}
\]



### Precipitation contribution:


\[
\Delta \vec{V}_{precip} =
\begin{bmatrix}
wind\_precip\_microburst\_x\_msc +
wind\_precip\_gustfront\_x\_msc +
wind\_precip\_hail\_drag\_msc \\
wind\_precip\_microburst\_y\_msc +
wind\_precip\_gustfront\_y\_msc \\
wind\_precip\_downdraft\_z\_msc
\end{bmatrix}
\]



Notes:

- **No turbulence** (turbulence outputs forces only)  
- **No Local Wind** subsystem  
- **Water/marine effects** are already inside ΔV_surface  

---------------------------------------------------------------------

# 4. Write Final Current Wind to XP12  
(**Aggregator is the ONLY subsystem allowed to write these**)

- `sim/weather/wind_now_x_mps = wind_final_x_msc`  
- `sim/weather/wind_now_y_mps = wind_final_y_msc`  
- `sim/weather/wind_now_z_mps = wind_final_z_msc`  

This is the wind the **flight model actually feels** each frame.

---------------------------------------------------------------------

# 5. Blended 37‑Layer Synoptic Profile  
(Blending applied **only below a cutoff altitude**)

Let:

- z_i = XP12 layer altitude  
- z_att_max = maximum altitude where blending applies  
  - Typically 300–700 m AGL equivalent  

Reconstruct Synoptic baseline vector:


\[
\vec{V}_{syn}(z_i)
\]



Apply blending:

If \( z_i \le z_{\text{att,max}} \):


\[
\vec{V}_{syn,blend}(z_i) = a_{global}(z_i) \cdot \vec{V}_{syn}(z_i)
\]



If \( z_i > z_{\text{att,max}} \):


\[
\vec{V}_{syn,blend}(z_i) = \vec{V}_{syn}(z_i)
\]



Write to XP12:

- `sim/weather/region/wind_speed_mps[i]`  
- `sim/weather/region/wind_direction_deg[i]`  
- `sim/weather/region/wind_vertical_mps[i]`  

**Important:**

- Terrain, Orographic, Surface, Cloud, and Precip ΔV **never** enter the 37 layers  
- Only **global blending** modifies the Synoptic profile  
- Above the cutoff altitude, the Synoptic atmosphere is **untouched**  

---------------------------------------------------------------------

# 6. Outputs (Standard Spec Table — ALL Datarefs)

| Output Name | Type | Description | Elements |
|-------------|------|-------------|----------|
| wind_final_x_msc | float | Final composed wind X component at aircraft altitude (m/s). | scalar |
| wind_final_y_msc | float | Final composed wind Y component at aircraft altitude (m/s). | scalar |
| wind_final_z_msc | float | Final composed wind Z component at aircraft altitude (m/s). | scalar |
| `sim/weather/wind_now_x_mps` | float | XP12 current wind X component (Aggregator ONLY). | scalar |
| `sim/weather/wind_now_y_mps` | float | XP12 current wind Y component (Aggregator ONLY). | scalar |
| `sim/weather/wind_now_z_mps` | float | XP12 current wind Z component (Aggregator ONLY). | scalar |
| `sim/weather/region/wind_speed_mps[i]` | float[37] | 37‑layer Synoptic wind speed after global blending below cutoff altitude. | 0..36 |
| `sim/weather/region/wind_direction_deg[i]` | float[37] | 37‑layer Synoptic wind direction after global blending below cutoff altitude. | 0..36 |
| `sim/weather/region/wind_vertical_mps[i]` | float[37] | 37‑layer Synoptic vertical wind after global blending below cutoff altitude. | 0..36 |
| wind_agg_att_global_msc[i] | float[37] | (Optional debug) Global blend factor applied to each layer. | 0..36 |
| wind_agg_cutoff_alt_msc | float | (Optional debug) Altitude above which no blending is applied. | scalar |

---------------------------------------------------------------------

# 7. Architectural Notes

- Synoptic Wind owns the **smoothed global atmosphere** (37 layers + local baseline).  
- Wind Aggregator owns the **final current wind at aircraft altitude**.  
- Aggregator **only modifies the 37 layers via global blending**, and only **below a cutoff altitude**.  
- **Aggregator is the ONLY subsystem that writes XP12 wind datarefs**.  
- Turbulence subsystem outputs **forces/moments only**, not wind.  
- Surface Wind is the **only subsystem** that provides blending scalars.  
- Cloud Dynamics and Precipitation subsystems provide **deterministic wind envelopes** that are added directly at aircraft altitude.  
