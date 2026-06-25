# AO 0235+164 — Multiwavelength Analysis Pipeline

**Author:** Joel Owusu Boateng Kwakye

Analysis of the intermediate synchrotron-peaked (ISP) BL Lac object **AO 0235+164**, covering Swift UVOT extinction-corrected light curves, multiwavelength temporal variability, cross-correlation, quasi-periodic oscillation analysis, and broadband SED modelling over an 8.5-year baseline (2008–2016).

---

## Repository Contents

| File | Description |
|------|-------------|
| `AO235+14_study.ipynb` | Full multiwavelength study: UVOT + XRT + Fermi-LAT light curves, DCF cross-correlation, period analysis, broadband SED modelling with MCMC |
| `Correction&LC.ipynb` | Swift UVOT extraction pipeline: reads raw FITS files, applies galactic extinction corrections, excludes non-detections, and plots corrected six-band light curves |

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
