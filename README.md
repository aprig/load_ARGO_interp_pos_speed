# Argo Under-Ice Position Correction Pipeline

Processing pipeline to correct and estimate positions of Argo profiling floats operating under sea ice, using a terrain-following bathymetry algorithm. Output files are formatted for assimilation into the [ISAS](https://www.seanoe.org/data/00412/52367/) ocean analysis system.

---

## Overview

Argo floats under sea ice cannot obtain GPS fixes and are assigned interpolated positions (POSITION_QC = 8) or missing positions (POSITION_QC = 9). This pipeline replaces those positions with estimates from a terrain-following algorithm constrained by GEBCO bathymetry and the float's parking depth, then interpolates the corrected profiles onto the standard ISAS vertical grid.

The position estimation is based on the method of [Yamazaki et al. (2019)](https://doi.org/10.1029/2019JC015406), as improved and implemented by the Coriolis team:
[https://github.com/cabanesc/Coriolis-under-ice-positioning](https://github.com/cabanesc/Coriolis-under-ice-positioning/blob/develop/estimate_profile_locations.m)

The pipeline produces one NetCDF file per float per variable (temperature, salinity), containing profiles interpolated on the ISAS vertical grid along with corrected positions, drift speeds, grounding flags, and parking depths.

---

## Repository Structure

```
.
├── process_argo_arctic.ipynb     # Main processing notebook (both loops)
├── README.md
```

The notebook contains two processing loops that share the same output format and variable attributes but differ in how positions are assigned:

- **Loop 1**: processes floats whose positions are all valid (POSITION_QC = 1) or linearly/geodesically interpolated (QC = 8). No external correction file is needed.
- **Loop 2**: loads pre-computed CSV files from the Coriolis terrain-following tool and replaces interpolated positions with bathymetry-constrained estimates. Floats are listed in `pos_bathy_follow_wmo`.

---

## Dependencies

| Package | Purpose |
|---|---|
| `xarray` | NetCDF I/O and dataset construction |
| `numpy` | Array operations |
| `pandas` | Reading terrain-following CSV output |
| `gsw` | Pressure-to-depth conversion (TEOS-10) |
| `scipy` | Akima 1D interpolation onto ISAS grid |

The terrain-following position estimates must be pre-computed using the MATLAB tool at [cabanesc/Coriolis-under-ice-positioning](https://github.com/cabanesc/Coriolis-under-ice-positioning), which produces one CSV per float named `estimate_profile_locations_{WMO}.csv`.

---

## Inputs

| File | Description |
|---|---|
| `{WMO}_prof.nc` | Argo mono-profile NetCDF (format version 3.1) |
| `{WMO}_Rtraj.nc` | Argo trajectory NetCDF — provides grounding flag and representative park pressure |
| `estimate_profile_locations_{WMO}.csv` | Terrain-following position estimates (Loop 2 only) |

Input Argo files are expected at `/data0/user/aprigent/ARGO/Arctic/`.
CSV correction files are expected at `/data0/user/aprigent/ARGO/`.

---

## Processing Steps

1. **Load profiles** — ascending profiles only (DIRECTION = 'A')
2. **Apply terrain-following corrections** (Loop 2 only) — replace POSITION_QC=8 positions with bathymetry-constrained estimates from the CSV; set POSITION_QC to 7 for corrected cycles
3. **Load trajectory file** — extract grounding flag (GROUNDED) and representative park pressure per cycle
4. **Time QC** — keep profiles with JULD_QC = 1 only
5. **Position QC** — accept POSITION_QC ∈ {1, 8} (Loop 1) or {1, 7, 8} (Loop 2)
6. **Pressure QC** — set levels with PRES_QC ≠ 1 to NaN before depth conversion
7. **Depth conversion** — pressure to depth via `gsw.z_from_p` using profile latitude
8. **Variable QC** — mask TEMP and PSAL where variable QC ≠ 1
9. **Vertical interpolation** — Akima 1D interpolation onto the ISAS standard depth grid `zi`
10. **Range checks** — TEMP ∈ [−2, 30] °C; PSAL ∈ [0, 40] PSU
11. **Save** — one NetCDF per float per variable (TEMP, PSAL)

---

## Output Files

Output files are written to `/data0/user/aprigent/texas/ARGO/bottom_interp_speed/` and named:

```
{WMO}_ASTE_TEMP.nc
{WMO}_ASTE_PSAL.nc
```

### Key Output Variables

| Variable | Description |
|---|---|
| `prof_lat` / `prof_lon` | Profile position (observed or interpolated; corrected for under-ice floats in Loop 2) |
| `prof_lat_traj` / `prof_lon_traj` | Terrain-following estimated position (NaN where algorithm did not converge) |
| `prof_speed` | Drift speed from initial interpolated trajectory (cm/s) |
| `prof_speed_traj` | Drift speed from terrain-following estimated trajectory (cm/s) |
| `prof_T` / `prof_S` | Temperature / salinity on ISAS vertical grid |
| `prof_date` | Profile date (days since 0001-01-01, Python ordinal) |
| `prof_pos` | POSITION_QC flag (1 = GPS fix, 7 = terrain-following corrected, 8 = interpolated) |
| `prof_grnd` | Grounding flag (Y/B/N/S/U — see below) |
| `prof_pdep` | Parking depth (m) |
| `prof_gdep` | Maximum depth reached when grounded (m) |
| `prof_descr` | Platform identifier (WMO number, 30-char string) |
| `prof_depth` | ISAS depth levels (m, positive down) |

### Grounding Flag Values (`prof_grnd`)

| Flag | Meaning |
|---|---|
| `Y` | Float touched the seafloor |
| `B` | Bathymetry verified — float constrained by depth |
| `N` | No seafloor contact |
| `S` | Shallow drift — float stuck near surface |
| `U` | Unknown |

### Speed Variables — Formula

Both speed variables are computed as:

```
speed = 100 × geodesic_distance(pos[i−1], pos[i]) / ((time[i] − time[i−1]) × 86400)
```

where the result is in **cm/s**, distance is in metres, and time is in days.

- `prof_speed` uses the **initial interpolated positions** and `JULD_LOCATION` as the time reference.
- `prof_speed_traj` uses the **terrain-following estimated positions** (`TrajLon/TrajLat`) and `JULD` (profile date) as the time reference.

---

## Float Lists

**Black list** (excluded entirely):
```python
black_list_wmo = ['4900501', '6900648', '6900195']
```

**Terrain-following floats** (processed by Loop 2):
```python
pos_bathy_follow_wmo = [
    '6903119', '6903143', '6903144', '6903145', '6903146',
    '6903147', '7901038', '3902112', '3902118', '4903655',
    '6903587', '6903588', '6903589', '6904087', '7900549', '7902188'
]
```

All other floats in `list_wmo` are processed by Loop 1.

---

## References

- Yamazaki, K., Aoki, S., Shimada, K., Kobayashi, T., & Kitade, Y. (2020). Structure of the subpolar gyre in the Australian-Antarctic Basin derived from Argo floats. *Journal of Geophysical Research: Oceans*, 125, e2019JC015406. https://doi.org/10.1029/2019JC015406
- Cabanes, C. et al. — Coriolis under-ice positioning tool: https://github.com/cabanesc/Coriolis-under-ice-positioning
- ISAS — In Situ Analysis System: https://www.seanoe.org/data/00412/52367/
- GEBCO Bathymetric Chart of the Oceans: https://www.gebco.net
