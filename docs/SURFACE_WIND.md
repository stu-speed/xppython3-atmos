# Surface Wind Subsystem  
Deterministic Surface‑Layer Physics + Blending & Gust Field Provider

Surface Wind computes all near‑surface wind physics and publishes:

- **ΔV_surface** — deterministic near‑surface wind modification  
- **vertical blending fields** — how strongly each subsystem should contribute near the ground  
- **gust envelope + σ driver** — atmospheric gust inputs for the Turbulence subsystem  

Waves and Turbulence consume Surface Wind as an input.

Surface Wind uses **Synoptic Wind subsystem datarefs** as its atmospheric driver.

---

# 1. Inputs

## 1.1 Observational & Model Inputs

### METAR (≤ 50 km)
- Surface wind speed \( U_{\text{METAR}} \)  
- Surface wind direction \( D_{\text{METAR}} \)  
- Surface gust speed \( G_{\text{METAR}} \)

Derived gust amplitude:


\[
A_{\text{gust, METAR}} = \max(0,\; G_{\text{METAR}} - U_{\text{METAR}})
\]



### External API (fallback)
Same fields as METAR.

### HRRR (CONUS)
- 10 m wind \( \vec{V}_{10m}^{HRRR} \)  
- Low‑level wind field \( \vec{V}_{LL}^{HRRR}(z) \)  
- Boundary layer height \( BLH \)  
- Shear diagnostics  
- HRRR gust potential  

### GFS 0.25°
- Pressure‑level winds  
- Temperature & humidity profiles  
- Jetstream structure  
- Synoptic shear  
- GFS gust potential  

### Background Turbulence Inputs
- Background RMS turbulence \( \sigma_{\text{bg}} \)  
- Background gust frequency scale \( f_{\text{bg}} \)

### Marine Inputs (WaveWatch III + SST)
- Significant wave height \( H_s \)  
- Peak wave period \( T_p \)  
- Swell direction \( \theta_{\text{swell}} \)  
- SST \( T_{sst} \)  
- Land–sea temperature contrast \( \Delta T_{ls} \)

---

## 1.2 Terrain & Landclass Inputs
- Landclass type  
- Roughness length \( z_0 \)  
- Vegetation density  
- Moisture retention  
- Distance to obstacles  
- Distance to coastline  

---

## 1.3 Atmospheric Inputs (From Synoptic Wind Subsystem)

### Synoptic Wind (baseline at aircraft altitude)
- wind_syn_x_msc  
- wind_syn_y_msc  
- wind_syn_z_msc  

Wind speed:


\[
U = \sqrt{wind\_syn\_x^2 + wind\_syn\_y^2}
\]



Wind direction:


\[
\theta_{wind} = \text{atan2}(wind\_syn\_y,\; wind\_syn\_x)
\]



### Synoptic Derived Fields
- Stability class  
- BLH  
- Temperature & humidity profiles  

### Additional Inputs
- AGL  
- Time of day  

---

## 1.4 Gust Inputs (Atmospheric Gust Envelope)

Surface Wind is the **only** subsystem that defines gust‑related wind information.

Inputs:
- gust_max_surface_msc  
- sigma_bg_surface_msc  
- gust_freq_surface_msc  

These define the **gust envelope** and **σ driver** for Turbulence.

Turbulence **never** generates gusts — it only consumes these fields.

---

# 2. Vertical Context Computation

Let \( h = AGL\_meters \).

### 2.1 Primary Surface Layer (0–300 m)


\[
w_{surf}(h) = 1 - \frac{h}{300}
\]



### 2.2 Fade‑Out Layer (300–700 m)


\[
w_{fade}(h) = 1 - \frac{h - 300}{400}
\]



### 2.3 Above 700 m


\[
w_{surf}(h) = 0
\]



Clamp all weights to \([0,1]\).

---

# 3. Surface‑Layer Physics (ΔV_surface)

## 3.1 Surface Friction & Vertical Shear  
Log‑wind profile:


\[
U(z) = U_{ref} \cdot \frac{\ln(z/z_0)}{\ln(z_{ref}/z_0)}
\]



Frictional ΔV:


\[
\Delta V_{friction} = (U(z) - U_{ref}) \cdot \hat{V}_{wind}
\]



Where:


\[
\hat{V}_{wind} = \frac{1}{U} \begin{bmatrix} wind\_syn\_x \\ wind\_syn\_y \end{bmatrix}
\]



---

## 3.2 Landclass Roughness Drag


\[
D_{lc} = k_{lc} \cdot w_{surf}(h)
\]




\[
\Delta \vec{V}_{lc} = -D_{lc} \cdot \hat{V}_{wind}
\]



---

## 3.3 Airport / Runway Smoothing


\[
S_{apt} = 1 - e^{-h/20}
\]




\[
\Delta \vec{V}_{apt} = -S_{apt} \cdot \hat{V}_{wind}
\]



---

## 3.4 Obstacle Shielding & Channeling


\[
C_{obs} = \sum_i \left( k_i \cdot e^{-d_i / R} \right)
\]




\[
\Delta \vec{V}_{obs} = -C_{obs} \cdot \hat{V}_{wind}
\]



---

## 3.5 Coastal Blending


\[
B_{coast} = e^{-d_c / 500}
\]




\[
\Delta \vec{V}_{coast} = -B_{coast} \cdot \hat{V}_{wind}
\]



---

## 3.6 Marine Wind Modifiers

### 3.6.1 Wave‑Dependent Roughness


\[
z_{0,sea} = \alpha \cdot \frac{H_s^2}{g T_p^2}
\]




\[
\Delta \vec{V}_{sea\_rough} = -z_{0,sea} \cdot \hat{V}_{wind}
\]



### 3.6.2 Swell‑Aligned Flow


\[
\Delta \theta = \theta_{wind} - \theta_{swell}
\]




\[
F_{swell} = \cos(\Delta \theta)
\]




\[
\Delta \vec{V}_{swell} = F_{swell} \cdot \hat{V}_{wind}
\]



### 3.6.3 SST‑Driven Stability


\[
S_{sst} = \tanh\left(\frac{T_{air} - T_{sst}}{4}\right)
\]




\[
\Delta \vec{V}_{sst} = S_{sst} \cdot \hat{V}_{wind}
\]



### 3.6.4 Sea‑Breeze Enhancement


\[
V_{sb} = k_{sb} \cdot \Delta T_{ls} \cdot d
\]




\[
\Delta \vec{V}_{sb} = V_{sb} \cdot \hat{V}_{wind}
\]



---

# 3.7 Deterministic Gust Envelope (Atmospheric Gust Source)



\[
G(h) = G_{max} \cdot w_{surf}(h)
\]




\[
\Delta \vec{V}_{gust} = G(h) \cdot \hat{V}_{wind}
\]



### 3.7.1 σ Driver


\[
\sigma_{surface}(h) = \sigma_{bg} \cdot w_{surf}(h)
\]



### 3.7.2 Gust Frequency Driver


\[
f_{gust}(h) = gust\_freq\_surface\_msc \cdot w_{surf}(h)
\]



---

# 3.8 Final ΔV_surface


\[
\Delta \vec{V}_{surface} =
\Delta \vec{V}_{friction}
+ \Delta \vec{V}_{lc}
+ \Delta \vec{V}_{apt}
+ \Delta \vec{V}_{obs}
+ \Delta \vec{V}_{coast}
+ \Delta \vec{V}_{sea\_rough}
+ \Delta \vec{V}_{swell}
+ \Delta \vec{V}_{sst}
+ \Delta \vec{V}_{sb}
+ \Delta \vec{V}_{gust}
\]



---

# 4. Blending Field Computation  
(Used by Wind Aggregator — Surface Wind does NOT write XP12 datarefs)

Blending factors are dimensionless scalars:

- **0.0 → subsystem suppressed**  
- **1.0 → subsystem full strength**

These match the Wind Aggregator’s blending model.

---

## 4.1 Terrain Wind Blending


\[
a_{terrain} = w_{surf}(h)
\]



---

## 4.2 Turbulence Blending


\[
a_{turb} = 1 - S_{sst} \cdot w_{surf}(h)
\]



---

## 4.3 Thermal Blending


\[
a_{thermal} = 1 - e^{-|\Delta T_{ls}|}
\]



---

## 4.4 Global Synoptic Blending  
(Small, smooth, METAR‑compatible)



\[
a_{global} = w_{surf}(h) \cdot (1 - D_{lc})
\]



This ensures:

- Synoptic is **slightly reduced** near the ground  
- Synoptic is **fully restored** aloft  
- METAR + ΔV_surface dominate the surface layer  

---

# 5. Output (Registered Datarefs Only)

| Dataref Name                | Type  | Description                                                                 | Elements |
|-----------------------------|-------|-----------------------------------------------------------------------------|----------|
| `wind_surface_x_msc`        | float | ΔV_surface contribution in X direction                                      | scalar   |
| `wind_surface_y_msc`        | float | ΔV_surface contribution in Y direction                                      | scalar   |
| `wind_surface_z_msc`        | float | ΔV_surface contribution in Z direction                                      | scalar   |
| `surface_atten_terrain_msc` | float | Blending factor for Terrain Wind (0.0–1.0)                                  | scalar   |
| `surface_atten_turb_msc`    | float | Blending factor for Turbulence subsystem (0.0–1.0)                          | scalar   |
| `surface_atten_thermal_msc` | float | Blending factor for Thermals subsystem (0.0–1.0)                            | scalar   |
| `surface_atten_global_msc`  | float | Blending factor for Global Synoptic Wind (0.0–1.0)                          | scalar   |
| `surface_gust_env_msc`      | float | Deterministic gust envelope \(G(h)\), consumed by Turbulence                | scalar   |
| `surface_sigma_driver_msc`  | float | Near‑surface σ driver (m/s RMS), consumed by Turbulence                     | scalar   |
| `surface_gust_freq_msc`     | float | Effective near‑surface gust frequency scale (1/s), consumed by Turbulence   | scalar   |

---

# Architectural Note

Surface Wind is the **only** subsystem that produces gust‑related atmospheric descriptors.  
Turbulence **never** generates wind — it only consumes:

- `surface_gust_env_msc`  
- `surface_sigma_driver_msc`  
- `surface_gust_freq_msc`  

Wind Aggregator is the **only** subsystem that writes XP12 wind datarefs.

