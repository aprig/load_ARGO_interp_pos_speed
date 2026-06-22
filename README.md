# Argo Arctic Profile Processing for ASTE



## Overview

This script builds a quality-controlled, vertically-interpolated temperature
and salinity (T/S) profile dataset from Argo float data in the Arctic, for
use with the Arctic Subpolar gyre sTate Estimate (ASTE). It:

1. Reads raw Argo NetCDF profile files (`*_prof.nc`) and trajectory files
   (`*_Rtraj.nc`).
2. Applies quality control (QC) on time, position, pressure, and the T/S
   variables themselves.
3. For a subset of floats that drift under sea ice, replaces interpolated
   positions (`POSITION_QC = 8`) with terrain-following position estimates
   from an external trajectory-correction algorithm (Yamazaki et al., 2019),
   and computes drift speed.
4. Interpolates T/S profiles onto the standard ISAS17 vertical grid using an
   Akima 1D spline.
5. Writes per-float NetCDF files, merges them into combined PSAL/TEMP
   datasets, intersects them to keep only profiles present in both, converts
   in-situ temperature/practical salinity to potential temperature/absolute
   salinity (via the TEOS-10 `gsw` toolbox), and writes a final merged
   dataset.
6. Produces simple diagnostic maps of float positions and QC flags.

## Requirements

- Python 3 with: `xarray`, `numpy`, `pandas`, `matplotlib`, `cartopy`, `gsw`
- `ArcTools` — a local/custom module providing `plot_arctic_ax_map_new`
  (Arctic basemap plotting helper)
- Input data (see **Data layout** below)

## Data layout (paths are hard-coded in the script)

| Path | Description |
|---|---|
| `/data0/user/aprigent/ISAS/ISAS17_Mask.nc` | ISAS17 vertical grid (`depthCFD`), used as the target interpolation depth levels |
| `/data0/user/aprigent/ARGO/Arctic/*_prof.nc` | Raw Argo profile files (one per float, WMO number in filename) |
| `/data0/user/aprigent/ARGO/Arctic/*_Rtraj.nc` | Argo trajectory files (grounding flag, park pressure) |
| `/data0/user/aprigent/ARGO/estimate_profile_locations_<WMO>.csv` | Terrain-following corrected positions/speeds for floats that drift under bathymetry-following ice (see "special-handling floats" below) |
| `/data0/user/aprigent/texas/ARGO/bottom_interp_speed/` | Output directory for per-float interim NetCDF files |
| `/data0/user/aprigent/texas/ARGO_bottom_interp_speed_ASTE_PSAL.nc` / `..._TEMP.nc` | Merged PSAL/TEMP datasets across all floats |
| `/data0/user/aprigent/texas/ARGO_ASTE_common_v4.nc` | **Final output**: common T/S profiles with potential temperature, on the ISAS vertical grid |

Update these paths at the top of each section if running on a different
system.

## Processing steps in detail

### 1. Vertical grid
The target depth levels (`zi`) are read from the ISAS17 land/ocean mask file
and reused throughout for vertical interpolation.

### 2. `interp_prof` — Akima vertical interpolation
For each profile, valid (non-NaN) points common to both the depth and data
arrays are interpolated onto the ISAS depth grid using
`scipy.interpolate.Akima1DInterpolator`. Profiles with fewer than 2 valid
points, or that raise a `ValueError` during interpolation, are skipped.
Returns the interpolated array and the indices of the profiles that
succeeded, so downstream metadata can stay aligned.

### 3. Main loop — standard floats
Iterates over every WMO float found in the Argo directory **except**:
- Floats in `black_list_wmo` (known-bad floats, skipped entirely).
- Floats in `pos_bathy_follow_wmo` (handled separately in the next loop,
  because they need the terrain-following position correction).

For each remaining float:
- Keeps only ascending profiles (`DIRECTION == 'A'`).
- Adds empty `SPEED` / `SPEED_EST` variables.
- Loads the matching `_Rtraj.nc` file (if present) to map `GROUNDED` flag and
  `REPRESENTATIVE_PARK_PRESSURE` onto each profile cycle; missing trajectory
  files default to "unknown grounding" / NaN park pressure.
- Applies time QC (`JULD_QC == '1'`) and skips floats with no valid times or
  whose data entirely predates 2000.
- Applies pressure QC, then converts pressure to depth with `gsw.z_from_p`.
- Computes the maximum depth reached while "grounded" (`GROUNDED == 'Y'`).
- Builds a combined validity mask from position QC (accepts flags `1` and
  `8`), variable QC flags, pressure QC, and data mode (`D`/`A`/`R` all
  accepted), and masks out everything that fails.
- Vertically interpolates T and S onto the ISAS grid via `interp_prof`, then
  clips to physically realistic ranges (T: −2–30 °C, S: 0–40).
- Assembles a `prof_descr` platform-ID string, and writes one salinity
  NetCDF (`<WMO>_ASTE_PSAL.nc`) and one temperature NetCDF
  (`<WMO>_ASTE_TEMP.nc`) per float, each with full CF-style variable
  attributes (units, long names, descriptions of the speed variables, etc.).

### 4. Second loop — bathymetry-following floats
Repeats the same QC/interpolation/output pipeline for the floats listed in
`pos_bathy_follow_wmo` (floats that follow bathymetry while parked/drifting
under ice and therefore need corrected positions), with one addition:
- Reads a companion CSV (`estimate_profile_locations_<WMO>.csv`) containing
  terrain-following corrected longitude/latitude/speed per cycle.
- For cycles where the original `POSITION_QC == 8`, overwrites
  `LONGITUDE`/`LATITUDE` with the corrected trajectory position, fills in
  `SPEED`/`SPEED_EST`, and re-flags `POSITION_QC` as `'7'` ("corrected").
- Position QC acceptance is extended to include the new flag `7`.
- Calls `plot_trajectories_new` to visualize raw vs. QC1 vs. grounded
  positions on an Arctic map before writing the per-float NetCDFs.

### 5. Merge all floats
All per-float `*_ASTE_PSAL.nc` and `*_ASTE_TEMP.nc` files are concatenated
along `iPROF` into two combined datasets and saved.

### 6. Match common T/S profiles
Because temperature and salinity QC can drop different profiles, a unique
key (`time_lat_lon_platform`) is built for each profile in both datasets,
and only profiles present in **both** TEMP and PSAL (`np.intersect1d`) are
kept, after verifying the two subsets are correctly aligned.

### 7. Build the final dataset
- Recomputes pressure from depth and latitude, converts practical salinity
  to absolute salinity and in-situ temperature to potential temperature
  using TEOS-10 (`gsw.conversions`).
- Assembles `ds_final` with lat/lon, speeds, dates, T/S (and their
  instrumental errors), grounding/park depth, position QC, and the ISAS
  depth grid, with full attribute metadata (including the terrain-following
  algorithm description and reference for QC=7 positions).
- Replaces `prof_T` with potential temperature and writes the final file:
  `ARGO_ASTE_common_v4.nc`.

### 8. Quick QC visualization
Reloads the final file and plots four Arctic maps comparing profile
locations for QC flags 1 (good), 8 (interpolated), 7 (terrain-following
corrected), and the union of 1/7/8, saved as
`plot_spatial_distribution_v4.png`.

## Key QC / flag conventions

| Flag | Meaning |
|---|---|
| `POSITION_QC = 1` | Good GPS fix |
| `POSITION_QC = 8` | Interpolated/estimated position (e.g., under ice) |
| `POSITION_QC = 7` | Position corrected via terrain-following algorithm (assigned by this script) |
| `prof_grnd = Y/B/N/S/U` | Ground contact / bathymetry-verified / no contact / shallow drift / unknown |

## Notes & caveats

- Several blocks of trajectory-attribute metadata (`prof_lon_traj`,
  `prof_lat_traj`, etc.) are commented out in the source — those variables
  are not currently written, even though their full attribute dictionaries
  remain in the code for reference/future use.
- File paths, the float blacklist, and the bathymetry-following float list
  are all hard-coded and should be reviewed before reuse on a new dataset.
- The script is structured as a converted Jupyter notebook (`# In[ ]:` cell
  markers); it should still run top-to-bottom as a plain `.py` file.
