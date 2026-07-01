# Turbulence Subsystem  
Stochastic Wind‑Vector Generation  
(Low‑Frequency Rolling + High‑Frequency Sharp + Integrated Gust Algorithms + Aircraft Adjustment)

The Turbulence Subsystem converts **deterministic envelopes** from all other subsystems into  
**time‑evolving turbulence wind vectors**, including all gust and turbulence algorithms.

It produces:

1. **Low‑frequency rolling turbulence**  
2. **High‑frequency sharp turbulence**  
3. **Aircraft‑adjusted turbulence vectors**

The subsystem does **not** apply final attenuation.  
Attenuation and XP12 write‑control are handled by the Wind Aggregator.

The delta outputs may be fed into a haptic motion platform.

Full credit for the turbulence model goes to this project:
https://forums.x-plane.org/files/file/100195-x-proturb-professional-turbulence-engine/.  It is re-implemented here
to take advantage of all the subsystem inputs and allow for integration into a haptic motion platform.

---------------------------------------------------------------------

# 1. Unified Inputs (From All Other Subsystems)

The Turbulence Subsystem does **not** ingest raw atmospheric data.  
It consumes **deterministic envelopes** produced by upstream subsystems, plus GUI attenuators and aircraft properties.

## 1.1 From **Orographic Wind Subsystem** (Far‑Field)
- wind_orographic_divergence_msc  
- wind_orographic_wave_z_msc  
- wind_orographic_rotor_z_msc  

## 1.2 From **Terrain Wind Subsystem** (Near‑Field)
- wind_terrain_rotor_envelope_msc  

## 1.3 From **Synoptic Wind Subsystem**
- wind_syn_shear_msc[i]  

## 1.4 From **Surface Wind Subsystem**
- wind_surface_turbulence_msc  

## 1.5 From **Thermal / Cloud Dynamics Subsystems**  
(Updated to use refactored CDS turbulence drivers)

- wind_thermal_strength_msc  
- wind_cloud_convective_strength_msc  
- wind_cloud_top_CAT_envelope_msc  
- wind_cloud_microburst_turbulence_msc  
- wind_cloud_anvil_turbulence_msc  

## 1.6 From **CMS** (Cloud‑State Flags)  
(Used indirectly via CDS envelopes)

- cloud/occupancy_msc  
- cloud/under_cloud_flag_msc  
- cloud/coverage_msc  

## 1.7 From **Aircraft Datarefs**
- sim/aircraft/weight/acf_m_total  
- sim/aircraft/weight/acf_wing_area  
- sim/aircraft/weight/acf_wing_span  
- sim/aircraft/weight/acf_moi_x  
- sim/aircraft/weight/acf_moi_y  
- sim/aircraft/weight/acf_moi_z  
- sim/aircraft/controls/acf_has_autopilot  
- sim/aircraft/controls/acf_turb_cat (A/B/C/D)  

## 1.8 From **GUI / User Interface**
- turb_atten_lowfreq_msc  
- turb_atten_highfreq_msc  
- turb_disable_xplane_write_msc (0/1)  

---------------------------------------------------------------------

# 2. Turbulence Frequency Bands (With Integrated Algorithms)

All turbulence algorithms are mapped into two bands:

- **Low‑frequency band** → rolling turbulence + continuous gusts  
- **High‑frequency band** → sharp turbulence + discrete/chaotic gusts  

---------------------------------------------------------------------

## 2.1 Low‑Frequency Rolling Turbulence  
**Algorithms Included**
- Dryden (Low‑Frequency Tail)  
- Von Kármán (Low‑Frequency Tail)  
- Mountain Wave (Laminar Component)  
- Convective Gust Pulses  
- Solar‑Driven Thermal Cycle  
- FAA Continuous Turbulence (Low‑Frequency Component)  

Frequency: **0.01–0.10 Hz**  
Period: **10–100 s**

---------------------------------------------------------------------

## 2.2 High‑Frequency Sharp Turbulence  
**Algorithms Included**
- Mechanical / Boundary‑Layer Turbulence  
- Clear‑Air Turbulence (CAT)  
- Rotor / Wave Breaking / Hydraulic Jump  
- Severe‑Weather Turbulence (CB/Hail/Rain)  
- FAA Discrete Gusts (1‑Cosine)  
- Dryden (High‑Frequency Band)  
- Von Kármán (High‑Frequency Band)  
- Microburst Turbulence  
- Anvil Turbulence  
- Synoptic Shear Turbulence  
- Surface Mechanical Turbulence  

Frequency: **0.5–5.0 Hz**  
Period: **0.2–2.0 s**

---------------------------------------------------------------------

# 3. Envelope Combination

## 3.1 Rolling Turbulence Envelope (Low‑Frequency Band)  
**Algorithm: Envelope Synthesis (LF)**

```
E_roll =
    w_div   * wind_orographic_divergence_msc +
    w_wave  * wind_orographic_wave_z_msc +
    w_rotorL* wind_terrain_rotor_envelope_msc
```

Typical weights:
- w_div   = 1.0  
- w_wave  = 0.5  
- w_rotorL= 0.3  

Apply GUI attenuator:

```
E_roll_final = E_roll * turb_atten_lowfreq_msc
```

---------------------------------------------------------------------

## 3.2 Sharp Turbulence Envelope (High‑Frequency Band)  
**Algorithm: Envelope Synthesis (HF)**

Updated to use refactored CDS turbulence drivers:

```
E_sharp =
    w_shear  * wind_syn_shear_msc[i] +
    w_surf   * wind_surface_turbulence_msc +
    w_therm  * wind_thermal_strength_msc +
    w_conv   * wind_cloud_convective_strength_msc +
    w_CAT    * wind_cloud_top_CAT_envelope_msc +
    w_micro  * wind_cloud_microburst_turbulence_msc +
    w_anvil  * wind_cloud_anvil_turbulence_msc +
    w_rotorS * wind_orographic_rotor_z_msc
```

Typical weights:
- w_shear  = 1.0  
- w_surf   = 0.7  
- w_therm  = 0.5  
- w_conv   = 0.6  
- w_CAT    = 0.8  
- w_micro  = 0.9  
- w_anvil  = 0.4  
- w_rotorS = 0.8  

Apply GUI attenuator:

```
E_sharp_final = E_sharp * turb_atten_highfreq_msc
```

---------------------------------------------------------------------

# 4. Turbulence Time Evolution

## 4.1 Rolling Turbulence (Low‑Frequency Band)  
**Algorithm: Harmonic Oscillator (LF Dryden/VK Base)**

Phase evolution:

```
φ_low(t) = φ_low(t−Δt) + 2π f_low Δt
```

Rolling turbulence vector:

```
ΔV_turb_roll(t) =
    E_roll_final *
    [ sin(φ_low + δ_x),
      sin(φ_low + δ_y),
      sin(φ_low + δ_z) ]
```

Phase offsets:
- δ_x = 0  
- δ_y = 2.1  
- δ_z = 4.2  

---------------------------------------------------------------------

## 4.2 Sharp Turbulence (High‑Frequency Band)  
**Algorithm: FBM + High‑Pass Filter (HF Dryden/VK Base)**

Base noise:

```
n(t) = FBM(t, f_high)
```

Filtered:

```
n_f(t) = HPF(n(t), f_cut = 0.3 Hz)
```

Sharp turbulence vector:

```
ΔV_turb_sharp(t) =
    E_sharp_final *
    [ n_f(t + τ_x),
      n_f(t + τ_y),
      n_f(t + τ_z) ]
```

Offsets:
- τ_x = 0.00  
- τ_y = 0.37  
- τ_z = 0.91  

---------------------------------------------------------------------

# 5. Aircraft Adjustment Algorithm  
### (Applied ONLY to Turbulence Output)

This step ensures turbulence affects each aircraft according to its  
mass, inertia, wing loading, and certification category.

## 5.1 Derived Aircraft Properties

```
wing_loading = acf_m_total / acf_wing_area
span_loading = acf_m_total / acf_wing_span
inertia_factor = (acf_moi_x + acf_moi_y + acf_moi_z) / 3
```

## 5.2 Aircraft Response Coefficient

```
C_resp =
    k_mass        * (1 / acf_m_total) +
    k_wingload    * (1 / wing_loading) +
    k_spanload    * (1 / span_loading) +
    k_inertia     * (1 / inertia_factor)
```

Typical values:
- k_mass      = 0.40  
- k_wingload  = 0.30  
- k_spanload  = 0.20  
- k_inertia   = 0.10  

## 5.3 Certification Category Scaling

```
C_cat =
    1.00 for CAT A
    0.75 for CAT B
    0.50 for CAT C
    0.35 for CAT D
```

## 5.4 Autopilot Damping

```
C_ap =
    1.00 if autopilot off
    0.65 if autopilot on
    0.45 if autopilot in turbulence mode
```

## 5.5 Final Aircraft Adjustment Coefficient

```
C_aircraft = C_resp * C_cat * C_ap
```

## 5.6 Adjusted Turbulence Vector  
**Algorithm: Vector Superposition + Aircraft Scaling**

```
ΔV_turb_adj =
    C_aircraft *
    (ΔV_turb_roll + ΔV_turb_sharp)
```

---------------------------------------------------------------------

# 6. Final Turbulence Output

The subsystem outputs **both** raw and aircraft‑adjusted turbulence vectors.

Raw:
- wind_turb_rolling_x_msc  
- wind_turb_rolling_y_msc  
- wind_turb_rolling_z_msc  
- wind_turb_sharp_x_msc  
- wind_turb_sharp_y_msc  
- wind_turb_sharp_z_msc  

Adjusted:
- wind_turb_adj_x_msc  
- wind_turb_adj_y_msc  
- wind_turb_adj_z_msc  

Wind Aggregator applies final attenuation and adds turbulence to deterministic wind,  
**unless `turb_disable_xplane_write_msc == 1`, in which case turbulence is local‑only.**

---------------------------------------------------------------------

# 7. Notes

- Aircraft Adjustment affects **only turbulence**, never deterministic wind.  
- All turbulence algorithms (Dryden, VK, CAT, mechanical, convective, rotor, mountain wave, severe weather, FAA gusts) are represented.  
- All turbulence outputs remain available via local datarefs even when XP12 writes are disabled.  
- This subsystem is fully Level‑D compliant.
