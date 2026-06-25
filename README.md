# AO 0235+164 — Multiwavelength Analysis Pipeline

**Author:** Joel Owusu Boateng Kwakye

Analysis of the intermediate synchrotron-peaked (ISP) BL Lac object **AO 0235+164**, covering Swift UVOT extinction-corrected light curves, multiwavelength temporal variability, cross-correlation, quasi-periodic oscillation analysis, and broadband SED modelling over an 8.5-year baseline (2008–2016).

---

## Repository Contents

| File | Description |
|------|-------------|
| `Obs_extractor.ipynb` | **Step 1 — SciServer** Query HEASARC and run HEASoft (`uvotimsum` + `uvotsource`) to produce raw `_mag.fits` files from archival Swift UVOT imaging |
| `Correction&LC.ipynb` | **Step 2 — local** Read the raw `_mag.fits` files, apply galactic extinction corrections, exclude non-detections, and plot corrected six-band light curves for AO 0235+164 |
| `UVOT_Pipeline.ipynb` | **Step 2 — universal** Same as above but handles any blazar/AGN source by editing a single config cell |
| `AO235+14_study.ipynb` | **Step 3** Full multiwavelength study: UVOT + XRT + Fermi-LAT light curves, DCF cross-correlation, period analysis, broadband SED modelling with MCMC |

---

## Obs_extractor.ipynb

SciServer notebook that queries HEASARC and uses HEASoft to photometrically reduce archival Swift UVOT imaging of any target blazar, producing per-epoch, per-filter `_mag.fits` files. This is **Step 1** — its output feeds directly into `Correction&LC.ipynb` and `UVOT_Pipeline.ipynb`.

> **Environment:** Must run inside a SciServer compute container with HEASoft installed and the `headata` FTP mirror mounted. Not intended for local execution.

### What it does

1. **Query** — uses `pyvo` to TAP-query HEASARC's `swiftmastr` catalog for all Swift observations pointed within 0.05° of the target (currently AO 0235+164: RA 39.66208°, Dec 16.61639°) within a chosen MJD window.
2. **Locate** — for each returned `obsid`, finds the UVOT sky image directory on the SciServer `headata` FTP mirror (`/home/idies/workspace/headata/FTP/swift/data/obs/`).
3. **Extract** — for each filter image (`*_sk.img.gz`):
   - decompresses the gzipped image,
   - runs `uvotimsum` to co-add snapshots into a single summed exposure,
   - runs `uvotsource` with a 5″ source aperture, 7″–40″ background annulus, and 5σ detection threshold.
4. **Save** — writes one `{target}_{obsid}_{filter}_mag.fits` file per observation per filter to persistent SciServer storage. Already-processed files are detected and skipped, so the cell is safe to re-run after an interruption.
5. **Package** (second cell) — zips the results folder into `UVOT_data.zip` for download off SciServer.

### Inputs

| Variable | Description |
|----------|-------------|
| `targets` | List of `{name, ra, dec}` dicts — add more blazars here to run the loop on multiple sources |
| `username` | Your SciServer username (used to locate persistent storage paths) |
| `output_dir` | Destination for extracted `_mag.fits` files |

Source and background region files (`source.reg`, `bkg.reg`) are generated automatically from each target's coordinates — no manual region-drawing needed.

### Outputs

| File | Description |
|------|-------------|
| `{target}_{obsid}_{filter}_mag.fits` | Per-epoch, per-filter photometry table (input to downstream notebooks) |
| `UVOT_data.zip` | Packaged results archive for download (created by Cell 2) |

### How to run

On SciServer, in a Jupyter session with HEASoft loaded:

1. Set `username` to your SciServer username in Cell 1.
2. Edit the `targets` list if processing additional sources.
3. Run Cell 1 — extracts all UVOT imaging and saves `_mag.fits` files.
4. Run Cell 2 — packages results into `UVOT_data.zip` for download.

Re-running after a partial or interrupted run is safe — existing output files are skipped automatically.

---

## AO235+14_study.ipynb

This notebook contains a unified computational pipeline for the multiwavelength temporal and spectral analysis of AO 0235+164. It processes quasi-simultaneous data from the *Neil Gehrels Swift Observatory* (XRT/UVOT) and the *Fermi Gamma-ray Space Telescope* (LAT).

### Pipeline Architecture & Core Modules

**Module 1 — Multiwavelength Temporal Alignment & Visualization**
- Standardizes disparate time formats from three instruments into MJD using mission-specific reference offsets (Swift $T_0 = 242031772.07373$)
- Segregates statistically significant detections from non-detection upper limits (TS ≥ 4 for Fermi-LAT; σ ≥ 3 for Swift XRT/UVOT)
- Highlights transient flaring structures (F1–F4) using `axvspan` and `annotate` for co-spatial jet emission analysis

**Module 2 — Irregularly Sampled Time-Series Analysis (Lomb-Scargle)**
- Implements a normalized Lomb-Scargle Periodogram over a 10,000-step frequency mesh (0.5–3.0 year range)
- Resolves a dominant Quasi-Periodic Oscillation (QPO) at approximately **1.24 years** during the quiescent state (MJD 55151–56731)

**Module 3 — Bayesian Parameter Estimation (MCMC)**
- 6-parameter dual log-parabolic model for the synchrotron and Inverse Compton continuum peaks
- Affine-invariant ensemble sampler (`emcee`): 5,000 steps, 64 walkers, burn-in discarded
- Corner plots mapping 1D/2D parameter covariances with asymmetric 1σ errors (16th/50th/84th percentiles)

**Module 4 — State-Dependent SED Evolution**
- Tracks broadband SED changes across the 2008 Major Flare, 2009–2014 Quiescent State, and 2014–2015 Flares
- Analytically back-calculates peak flux scaling factors from MCMC shape parameters to anchor model curves through median epoch fluxes

---

## Correction&LC.ipynb

Swift UVOT extraction and extinction correction pipeline producing a clean, analysis-ready CSV and multiband light curve plot.

### What it does

**Cell 1 — Extraction & Correction**

Scans all Swift UVOT `_mag.fits` files in `AO_0235+164_data/` and produces `AO_0235+164_UVOT_Corrected.csv`:

| Column | Description |
|--------|-------------|
| `MJD` | Modified Julian Date of observation |
| `Filter` | UVOT filter (V, B, U, W1, M2, W2) |
| `Intrinsic_Flux` | Extinction-corrected flux in erg s⁻¹ cm⁻² Å⁻¹ |
| `Flux_Error` | Propagated flux uncertainty |

Key steps:
- Filter identification from filename (`uvv`→V, `ubb`→B, `uuu`→U, `uw1`→W1, `um2`→M2, `uw2`→W2)
- MJD from FITS header (`MJD-OBS`, `DATE-OBS`), with fallback to table column `TSTART` via Swift MET reference (`MJD = 51910.0007428 + TSTART/86400`)
- Non-detections excluded via UVOT's `MAG = 99` sentinel flag
- Extinction correction: `F_intrinsic = F_raw × 10^(A_λ / 2.5)`

**Cell 2 — Light Curve Plot**

Plots all six filters as stacked subpanels with shared MJD x-axis. Saved to `AO_0235+164_Multiband.jpg`.

### Extinction Values

Galactic extinction A_λ (mag) for the line of sight to AO 0235+164:

| Filter | A_λ (mag) | Source |
|--------|-----------|--------|
| V (5468 Å) | 0.2188 | Schlafly & Finkbeiner (2011) — Landolt V |
| B (4392 Å) | 0.2893 | Schlafly & Finkbeiner (2011) — Landolt B |
| U (3465 Å) | 0.3458 | Schlafly & Finkbeiner (2011) — Landolt U |
| W1 (2600 Å) | 0.4514 | Derived: E(B-V)=0.0705 × R_W1=6.40 |
| M2 (2246 Å) | 0.6132 | Derived: E(B-V)=0.0705 × R_M2=8.70 |
| W2 (1928 Å) | 0.5716 | Derived: E(B-V)=0.0705 × R_W2=8.10 |

E(B-V) = A_B − A_V = 0.0705 mag. UV R_λ coefficients follow the Cardelli et al. (1989) extinction law (R_V = 3.1).

### Usage

1. Place all Swift UVOT `_mag.fits` files in `AO_0235+164_data/`
2. Ensure `AO_0235+164.csv` (NED extinction table) is present
3. Run Cell 1 → generates `AO_0235+164_UVOT_Corrected.csv`
4. Run Cell 2 → plots and saves `AO_0235+164_Multiband.jpg`

---

## UVOT_Pipeline.ipynb

Universal Swift UVOT extraction and light curve pipeline. Supports any number of sources — only the config cell needs editing.

### What it does

**Cell 1 — Configuration (edit here only)**

Define all sources in the `SOURCES` list. Each entry takes:

| Key | Description |
|-----|-------------|
| `name` | Source name — used for output filenames |
| `data_dir` | Folder containing raw `_mag.fits` files |
| `ext_csv` | NED galactic extinction table (must have `Bandpass` and `Galactic Extinction` columns) |
| `out_dir` | Where to save the corrected CSV and multiband plot |
| `mjd_min` | Start of MJD window for the plot (`None` = no limit) |
| `mjd_max` | End of MJD window for the plot (`None` = no limit) |

Example entry:
```python
{
    'name':     'AO_0235+164',
    'data_dir': '/path/to/AO_0235+164_data/',
    'ext_csv':  '/path/to/AO_0235+164.csv',
    'out_dir':  '/path/to/AO_0235+164_data/',
    'mjd_min':  54400,   # zoom into a flare window
    'mjd_max':  54800,
}
```

**Cell 2 — Pipeline (do not edit)**

Runs automatically for every source in `SOURCES`. For each source it:

1. Reads Landolt V/B/U extinctions from the NED CSV
2. Derives UV extinctions (W1, M2, W2) automatically via E(B-V) = A_B − A_V and the Cardelli et al. (1989) law
3. Scans all `_mag.fits` files, identifies the UVOT filter from the filename
4. Extracts MJD from the FITS header (`MJD-OBS`, `DATE-OBS`) or falls back to the table `TSTART` column (Swift MET → MJD conversion)
5. Excludes non-detections (`MAG = 99` sentinel) and skipped files
6. Applies extinction correction: `F_intrinsic = F_raw × 10^(A_λ / 2.5)`
7. Saves `{name}_UVOT_Corrected.csv` to `out_dir`
8. Plots six stacked filter panels (V, B, U, W1, M2, W2) with shared MJD axis, restricted to `mjd_min`–`mjd_max` if set
9. Saves `{name}_Multiband.jpg` to `out_dir`

### Extinction derivation

UV extinctions (W1/M2/W2) are not in the Schlafly & Finkbeiner (2011) NED tables. The pipeline derives them from the optical values:

```
E(B-V) = A_B − A_V
A_W1   = E(B-V) × 6.40
A_M2   = E(B-V) × 8.70
A_W2   = E(B-V) × 8.10
```

R_λ coefficients follow Cardelli, Clayton & Mathis (1989), R_V = 3.1.

Example derived values for the four pre-configured sources:

| Source | E(B-V) | A_W1 | A_M2 | A_W2 |
|--------|--------|------|------|------|
| AO 0235+164 | 0.0705 | 0.451 | 0.613 | 0.572 |
| 3C 66A | 0.0745 | 0.477 | 0.648 | 0.604 |
| PKS 1749+096 | 0.1570 | 1.005 | 1.366 | 1.272 |
| PKS 1730−130 | 0.4593 | 2.939 | 3.996 | 3.720 |

### Output files

For each source the pipeline writes two files to `out_dir`:

| File | Description |
|------|-------------|
| `{name}_UVOT_Corrected.csv` | Extinction-corrected detections (columns: MJD, Filter, Intrinsic_Flux, Flux_Error) |
| `{name}_Multiband.jpg` | Six-panel stacked light curve plot |

### Adding a new source

1. Download Swift UVOT `_mag.fits` files into a folder
2. Export the NED extinction table for the source as a CSV
3. Add a new dict to `SOURCES` in Cell 1 with the paths and optional MJD limits
4. Run Cell 2 — no other changes needed

---

## Dependencies

```
astropy
numpy
pandas
matplotlib
emcee
corner
```

---

## References

- Schlafly & Finkbeiner (2011), ApJ, 737, 103
- Cardelli, Clayton & Mathis (1989), ApJ, 345, 245
- Poole et al. (2008), MNRAS, 383, 627 — Swift UVOT photometric system
- Breeveld et al. (2011), AIP Conf. Proc., 1358, 373 — UVOT calibration
