# Cloud Modeling Subsystem (CMS)
Deterministic Terrain‑Fixed Cloud Layers, Occupancy, Geometry, Patch Structure, and Cloud‑State Flags  
(Authoritative cloud source for CDS, thermals, turbulence, visibility, icing, and any other subsystems)

CMS is the **single source of truth** for cloud state.

It either:
- **Computes** cloud layers from METAR/HRRR/GFS/satellite/radar, or  
- **Copies** cloud layers from XP12 / external engines (ActiveSky, etc.)

All other subsystems read **CMS datarefs only**.

---------------------------------------------------------------------

# 0. Cloud Modeling Mode

CMS decides whether to compute or copy cloud layers:

```
cloud_modeling_mode_msc =
    0 → Compute from Atmos Dataset (METAR/HRRR/GFS/satellite/radar)
    1 → Copy from X‑Plane cloud layers
    2 → Copy from External Weather Engine (ActiveSky, etc.)
```

---------------------------------------------------------------------

# 1. Inputs

## 1.1 Atmospheric Inputs (Mode 0)

- METAR cloud base, coverage, type  
- HRRR cloud fraction CF(z), qc/qi, W(z), CAPE/CIN, LCL/LFC/EL  
- GFS cloud fraction, cloud water/ice, base/top  
- Satellite cloud‑top height, anvil extent, overshooting tops  
- Radar reflectivity, gust‑front boundaries, downdraft signatures  

## 1.2 X‑Plane Cloud Layer Inputs (Mode 1)

```
sim/weather/cloud_base_msl_m[i]
sim/weather/cloud_tops_msl_m[i]
sim/weather/cloud_type[i]
sim/weather/cloud_coverage[i]
```

## 1.3 External Weather Engine Inputs (Mode 2)

Example (ActiveSky):

```
as_cloud_base_msl_m[i]
as_cloud_top_msl_m[i]
as_cloud_type[i]
as_cloud_coverage[i]
```

## 1.4 Aircraft / Grid Inputs

- lat/lon/alt  
- AGL  
- Local terrain elevation  
- Horizontal position for occupancy sampling  

---------------------------------------------------------------------

# 2. CMS Cloud Layer Datarefs (Authoritative Layer Set)

CMS publishes its own **layer datarefs** for all subsystems to use:

### 2.1 Layer Geometry

| Dataref | Type | Description |
|---------|------|-------------|
| cloud/layer_base_msl_msc[i] | float | Layer base (m MSL) |
| cloud/layer_top_msl_msc[i] | float | Layer top (m MSL) |
| cloud/layer_thickness_msc[i] | float | Layer thickness (m) |



\[
\text{cloud/layer\_thickness\_msc[i]} = \text{cloud/layer\_top\_msl\_msc[i]} - \text{cloud/layer\_base\_msl\_msc[i]}
\]



### 2.2 Layer Type

| Dataref | Type | Description |
|---------|------|-------------|
| cloud/layer_type_msc[i] | int | Layer type (CU/TCU/CB/ST/CI/ANVIL) |

### 2.3 Layer Coverage

| Dataref | Type | Description |
|---------|------|-------------|
| cloud/layer_coverage_msc[i] | float | Layer coverage fraction [0–1] |

### 2.4 Layer Fraction Profile (Optional)

| Dataref | Type | Description |
|---------|------|-------------|
| cloud/layer_fraction_profile_msc[i] | float[Z] | Vertical cloud fraction profile |

---------------------------------------------------------------------

# 3. Compute vs Copy Logic

## 3.1 Mode 0 — Compute Layers (Atmos)

For each layer i:

- Derive base/top from METAR + HRRR + GFS + satellite  
- Derive type from METAR + microphysics + CAPE/CIN  
- Derive coverage from METAR + CF(z)  

Example:



\[
CB_i = \min(CB_{\text{METAR},i}, CB_{\text{HRRR},i}, CB_{\text{GFS},i})
\]





\[
CT_i = \max(CT_{\text{HRRR},i}, CT_{\text{GFS},i}, CT_{\text{satellite},i})
\]





\[
\text{cloud/layer\_base\_msl\_msc[i]} = CB_i
\]





\[
\text{cloud/layer\_top\_msl\_msc[i]} = CT_i
\]





\[
\text{cloud/layer\_coverage\_msc[i]} = C_i \in [0,1]
\]



Type classification:

- ST, CU, TCU, CB, CI, ANVIL based on METAR + qc/qi + CAPE/CIN + satellite.

## 3.2 Mode 1 — Copy Layers (XP12)

For each XP layer i:

```
cloud/layer_base_msl_msc[i]     = sim/weather/cloud_base_msl_m[i]
cloud/layer_top_msl_msc[i]      = sim/weather/cloud_tops_msl_m[i]
cloud/layer_type_msc[i]         = sim/weather/cloud_type[i]
cloud/layer_coverage_msc[i]     = sim/weather/cloud_coverage[i]
cloud/layer_thickness_msc[i]    = cloud/layer_top_msl_msc[i] - cloud/layer_base_msl_msc[i]
```

## 3.3 Mode 2 — Copy Layers (External Engine)

For each external layer i:

```
cloud/layer_base_msl_msc[i]     = as_cloud_base_msl_m[i]
cloud/layer_top_msl_msc[i]      = as_cloud_top_msl_m[i]
cloud/layer_type_msc[i]         = as_cloud_type[i]
cloud/layer_coverage_msc[i]     = as_cloud_coverage[i]
cloud/layer_thickness_msc[i]    = cloud/layer_top_msl_msc[i] - cloud/layer_base_msl_msc[i]
```

---------------------------------------------------------------------

# 4. Single‑Column Effective Cloud Geometry

CMS also publishes a **single effective column** for subsystems that don’t care about per‑layer detail:

| Dataref | Type | Description |
|---------|------|-------------|
| cloud/base_msl_msc | float | Effective cloud base (lowest layer base) |
| cloud/top_msl_msc | float | Effective cloud top (highest layer top) |
| cloud/type_msc | int | Dominant cloud type |
| cloud/coverage_msc | float | Total coverage fraction |

Computed as:

```
cloud/base_msl_msc = min(cloud/layer_base_msl_msc[i])
cloud/top_msl_msc  = max(cloud/layer_top_msl_msc[i])
cloud/coverage_msc = max(cloud/layer_coverage_msc[i])  // or weighted
cloud/type_msc     = dominant cloud/layer_type_msc[i]
```

---------------------------------------------------------------------

# 5. Terrain‑Fixed Cloud Occupancy Field

CMS maintains a **2D terrain‑fixed occupancy grid**:



\[
O_{\text{phys}}(x,y) \in [0,1]
\]



Grid resolution: ~1 km × 1 km (configurable).

## 5.1 Microphysics Sampling

At each grid cell (x,y), sample at mid‑cloud height:



\[
z_{\text{mid}} = \frac{\text{cloud/base\_msl\_msc} + \text{cloud/top\_msl\_msc}}{2}
\]



Sample:

- qc(x,y,z_mid)  
- qi(x,y,z_mid)  
- W(x,y,z_mid)  
- Z(x,y,z_mid)  

## 5.2 Cloud Mass Term



\[
M(x,y) = w_c \cdot qc(x,y,z_{\text{mid}}) + w_i \cdot qi(x,y,z_{\text{mid}})
\]



## 5.3 Convective Term



\[
C(x,y) = w_W \cdot \max(W(x,y,z_{\text{mid}}), 0)
\]



## 5.4 Reflectivity Term



\[
R(x,y) = w_Z \cdot \frac{Z(x,y,z_{\text{mid}})}{Z_{\max}}
\]



## 5.5 Raw Occupancy



\[
O_{\text{raw}}(x,y) = M(x,y) + C(x,y) + R(x,y)
\]



## 5.6 Normalization



\[
O_{\text{phys}}(x,y) = \frac{O_{\text{raw}}(x,y)}{O_{\text{raw,max}} + \epsilon}
\]



Clamped to [0,1].

---------------------------------------------------------------------

# 6. Coverage‑Driven Patch Sizing & Noise

Let total coverage:



\[
C = \text{cloud/coverage\_msc} \in [0,1]
\]



## 6.1 Patch Size



\[
S = S_{\min} + (S_{\max} - S_{\min}) \cdot C
\]



Typical:

```
S_min = 0.5 km
S_max = 8.0 km
```

## 6.2 Noise Coordinates



\[
n_x = \frac{x}{S}, \quad n_y = \frac{y}{S}
\]



Noise:

```
N(x,y) = Perlin(n_x, n_y)
```

## 6.3 Coverage‑Dependent Threshold



\[
T = 0.5 + (1 - C) \cdot 0.25
\]



## 6.4 Patch Mask

```
P(x,y) = 1 if N(x,y) > T
         0 otherwise
```

## 6.5 Combined Occupancy



\[
O_{\text{combined}}(x,y) = O_{\text{phys}}(x,y) \cdot P(x,y)
\]



---------------------------------------------------------------------

# 7. Vertical Masking & Aircraft Occupancy

For aircraft altitude \(z_{\text{ac}}\):

## 7.1 Vertical Mask

```
if z_ac < cloud/base_msl_msc: vertical_mask = 0.5
elif cloud/base_msl_msc <= z_ac <= cloud/top_msl_msc: vertical_mask = 1.0
else: vertical_mask = 0.0
```

## 7.2 Final Aircraft Occupancy



\[
O_{\text{ac}} = O_{\text{combined}}(x_{\text{ac}}, y_{\text{ac}}) \cdot vertical\_mask
\]
1G


CMS publishes:

| Dataref | Type | Description |
|---------|------|-------------|
| cloud/occupancy_msc | float | Final aircraft occupancy [0–1] |

---------------------------------------------------------------------

# 8. Cloud‑State Flags

## 8.1 In‑Cloud Flag

```
cloud/in_cloud_flag_msc = (O_ac > 0.6)
```

## 8.2 Under‑Cloud Flag (Thermal Shading)

```
cloud/under_cloud_flag_msc = (z_ac < cloud/base_msl_msc) AND (O_ac > 0.3)
```

## 8.3 Clear Gap Flag

```
cloud/clear_gap_flag_msc = (O_ac < 0.2)
```

## 8.4 Convective Core Flag

Convective core:



\[
core(x,y) = (W > W_{\text{thr}}) \land (qc+qi > q_{\text{thr}}) \land (Z > Z_{\text{thr}})
\]



```
cloud/convective_core_flag_msc = core(x_ac, y_ac)
```

## 8.5 Anvil Flag

```
cloud/anvil_flag_msc =
    (cloud/type_msc == ANVIL) OR
    (z_ac < cloud/top_msl_msc AND O_ac > 0.4 AND CF(CT) > 0.6)
```

---------------------------------------------------------------------

# 9. Convective Structure Outputs

CMS publishes convective structure for CDS and turbulence:

| Dataref | Type | Description |
|---------|------|-------------|
| cloud/updraft_core_strength_msc | float | Updraft strength |
| cloud/downdraft_core_strength_msc | float | Downdraft strength |
| cloud/anvil_divergence_msc | float | Divergence at cloud top |
| cloud/gustfront_strength_msc | float | Gust‑front strength |
| cloud/vertical_shear_base_msc | float | Shear at cloud base |
| cloud/vertical_shear_top_msc | float | Shear at cloud top |

---------------------------------------------------------------------

# 10. Cloud Layer Writer (Optional)

CMS **does not need** to write to XP datarefs for internal subsystems,  
but may optionally mirror its layer set to XP12 for visual consistency:

If `cloud_modeling_mode_msc == 0` (compute):

```
sim/weather/cloud_base_msl_m[i]   = cloud/layer_base_msl_msc[i]
sim/weather/cloud_tops_msl_m[i]   = cloud/layer_top_msl_msc[i]
sim/weather/cloud_type[i]         = cloud/layer_type_msc[i]
sim/weather/cloud_coverage[i]     = cloud/layer_coverage_msc[i]
```

If mode = 1 or 2, CMS **reads** from XP/external and **does not overwrite** them.

---------------------------------------------------------------------

# 11. Summary

- CMS owns **all cloud‑layer datarefs** (`cloud/layer_*_msc`).  
- CMS decides whether to **compute** or **copy** cloud layers.  
- All subsystems (CDS, thermals, turbulence, visibility, icing) read **CMS datarefs only**.  
- CMS provides occupancy, shading, gaps, convective cores, anvils, and structure.  
- CDS adds sun angle and physics on top of CMS cloud state.
